# Engineering Notes

The container-runtime and Docker networking problems that shaped LabGen, and the design
decisions that came out of them. Almost all of it was learned by a lab failing to come
up in a way that blamed the wrong component.

---

## Docker networking problems

### Static IPs contend across runs

The defining problem of the project. Isolated containers want specific addresses —
`172.21.0.2`, `172.21.0.3` — because the config declares them and because a lab whose
addresses move between runs is not reproducible.

Docker hands out addresses on a network to whatever attaches first. A pivot host
attached to the internal network from a previous run holds one, plausibly exactly the
one an isolated container is declared to have. Docker then refuses to start that
container, and the error names the container that *failed*, not the one holding its
address.

The fix is ordering, and it has to be explicit:

```python
# Disconnect pivot hosts from the internal network before starting isolated
# containers. If a previous run failed mid-way, pivots may hold IPs that
# isolated containers need (e.g. .2/.3), blocking their creation.
for pivot in pivot_hosts:
    if INTERNAL_NET in current_nets(pivot):
        run(f"docker network disconnect {INTERNAL_NET} {pivot}", check=False)
```

Disconnect pivots, start isolated containers so they claim their declared addresses
unopposed, then reattach pivots to whatever is free. Pivots do not need deterministic
addresses — nothing in the config references them by their internal-segment IP — so
they are the right side of the contention to yield.

**Lesson:** when some addresses must be deterministic and others need not be, make the
flexible ones yield explicitly. Relying on attach order to work out is relying on the
previous run having exited cleanly, which is exactly the case where it did not.

### Exited containers hold addresses

```python
# Not running — remove and recreate. docker start on exited containers
# leaves stale network endpoints that cause "Address already in use" on
# the next start, especially on isolated networks after a daemon restart.
run(f"docker rm -f {name}", check=False)
```

`docker start` on an exited container looks correct and is faster. It fails after a
Docker daemon restart, because the endpoint records survive while the containers do not.
The address appears taken, nothing visible is using it, and `docker ps` shows an empty
network.

Remove and recreate costs a couple of seconds and is unconditionally correct. Running
containers are skipped, so `up` stays idempotent.

**Lesson:** in Docker, a container's network endpoint is not the same object as the
container. "Nothing is running" does not mean "nothing holds the address."

### `down` has to clean up configs that no longer exist

Teardown removes containers on the isolated network *first*, and removes orphans from
previous configs rather than only the containers the current config names.

Without that, editing a config to rename or remove a target leaves the old container
running. It still holds its address and still sits on the network, so the next `up`
fails against a container the current config has never heard of. Tearing down by "what
is actually on these networks" rather than by "what my config lists" is the only
version that survives config edits — which is the entire intended workflow.

**Lesson:** teardown should be driven by observed state, not by declared state. The
declaration is what you *want*; cleanup has to handle what is *there*.

---

## Container runtime problems

### PID 1 does not reap, and `pgrep` does not care

Found while building reactive ALTRON scenarios on these labs, and the best bug across
either project.

Lab targets run `systemd`, but minimal helper containers commonly run
`tail -f /dev/null` as PID 1. That process never reaps children. A killed process
becomes a zombie and stays one for the container's lifetime.

And `pgrep -f <pattern>` still matches a zombie, because the pattern is matched against
`/proc/<pid>/stat`'s comm field, which a zombie retains.

So a liveness check of the form `pgrep -f implant.sh` returns success forever, including
long after the process was killed. A trainee successfully evicting a persistence
mechanism was told it was still running.

This is correct Linux behaviour in every component. It is only wrong in combination, and
the combination only exists in containers with a non-reaping PID 1 — so the check works
perfectly on a developer's host and fails silently in the lab.

**Lesson:** process-liveness checks that are correct on a normal Linux host are wrong
inside a container whose PID 1 is not an init. Check for a live process state explicitly
rather than for the existence of a PID, and be suspicious of any check that has never
been tested against a *killed* process rather than an absent one.

### `systemd` as PID 1, deliberately

Running `systemd` in a container is widely discouraged, and for most workloads the
advice is right. For this one it is wrong, for a specific reason: a trainee practising
persistence needs `systemctl` to work.

If a container runs a single service process, then a systemd unit is not a viable
persistence mechanism, `systemctl list-unit-files` returns nothing useful, and an entire
category of technique — the one that matters most on modern Linux — cannot be practised
or detected. The lab would teach that persistence means cron, which is wrong.

The cost is `--privileged` or close to it on some images, slower startup, and a less
idiomatic container. Accepted, because the alternative removes content from the
curriculum.

**Lesson:** container best practices optimise for production workloads. A training
target's requirements are different, and "this is not how you would run it in
production" is not an argument when the thing being simulated *is* a production host.

### Services need init to be up before they are enabled

Configuration happens in `lab-init.sh`, a systemd unit, rather than in the entrypoint.
The entrypoint does almost nothing:

```bash
{
    for var in SSH_USERS SERVICES ISOLATED CALLBACK_HOST CALLBACK_PORT; do
        val="${!var}"
        [ -n "$val" ] && printf '%s=%s\n' "$var" "$val"
    done
} > /etc/lab.env

exec /sbin/init
```

Enabling and starting services from the entrypoint means doing it before systemd exists,
so `systemctl start` has nothing to talk to. Running configuration as a unit means
systemd is already up, services genuinely start, and the container's own init manages
them from then on — which is also what makes them restartable by a trainee.

**Lesson:** `exec`ing init from an entrypoint means the entrypoint's work happens in the
wrong world. Move anything that needs init into a unit.

---

## Design decisions

### Defence in depth for isolation, because one layer is a typo away

Isolation could have been routing alone. Isolated containers join only the internal
network; the host has no interface there. That is genuinely strong.

It is also one accidental `docker network connect` from being nothing — including one
typed by an instructor debugging something unrelated at 2am before a competition.

So there is a second layer, applied inside the container against its own discovered
gateway:

```bash
if [ "${ISOLATED:-}" = "1" ]; then
    GW=$(ip route show 2>/dev/null | awk '/default/ {print $3; exit}')
    if [ -n "$GW" ]; then
        echo "ALL: $GW" >> /etc/hosts.deny
    fi
fi
```

Discovering the gateway at runtime rather than hardcoding it is what makes this
maintainable — the block stays correct whatever address Docker assigns, and survives a
change to the internal subnet.

Neither layer is sufficient. Routing fails to an accidental attachment; TCP wrappers
fail for any service that ignores them. Together, two independent things must be wrong
before a trainee reaches the isolated segment directly.

**Lesson:** the value of a second enforcement layer is not that it is stronger. It is
that it fails for different reasons than the first one.

### Provision AD at boot rather than shipping an image

Provisioning a Samba domain with `samba-tool` at first boot costs about 60 seconds. A
pre-provisioned image would start instantly.

Provisioning won on two grounds. The domain is **configurable** — realm, NetBIOS name,
OUs, users, groups, and SPNs all come from JSON, so a different domain is a config edit
rather than a new image build. And the domain is **reviewable** — there is no opaque
multi-gigabyte artifact whose contents nobody can inspect. What the domain contains is
text in a file under version control.

For a lab whose purpose is reproducibility, 60 seconds against "the domain is defined by
reviewable text" is not a close call.

**Lesson:** prefer a slower path that keeps the artifact declarative. Build-time
baking trades configurability and reviewability for startup time, and training
infrastructure values the first two more.

### Packed environment strings over mounted config files

Infrastructure nodes receive their configuration as packed strings:

```python
# Ifaces string: ip:subnet:zone|...
ifaces_parts.append(f"{iface.get('ip','')}:{subnet}:{zone}")

# Routes: prefix:via,...
routes = ",".join(f"{r['prefix']}:{r['via']}" for r in inf.get("routes", []))
```

A mounted JSON file would be cleaner to parse. Environment strings won because LabGen
runs on native Linux, WSL2, Docker Desktop on Windows, and Codespaces — and bind mounts
behave differently across all four, particularly around path translation and file
permissions. An environment variable behaves identically everywhere.

A side benefit: `docker inspect` shows a container's complete configuration. Debugging a
firewall node means reading one command's output, not finding a mounted file and
checking whether it is the file you think it is.

The cost is a shell parsing layer and a schema that breaks if a value contains `:` or
`|`. For IPs, subnets, zone names, ports, and actions, that constraint never binds — but
it is a real ceiling, and a config shape with free text in it would need the mount.

**Lesson:** cross-platform portability is worth accepting a worse data format for.
Choose the serialisation your weakest deployment target handles identically.

### Derive, do not store

Recurring, and it eliminated a class of bug each time.

**Pivot hosts** are computed from which targets declare `isolated` with a `via`, not
maintained as a list. **Zone membership** is implied by a target's address falling in a
segment's subnet, not declared separately. **Segment gateways** default to `.1` of the
declared subnet. The **engagement report** is generated from the config rather than
written alongside it.

Each of these is a field that does not exist and therefore cannot disagree with its
source. The report case is the most valuable: an answer key maintained by hand drifts
from the lab the first time someone changes a password, and a drifted answer key is
worse than none, because it is trusted.

**Lesson:** every stored copy of derivable information is a future inconsistency.

### A lite config, not a scaled-down product

The Codespaces lite config is four containers covering HTTP basic auth, FTP, MySQL,
Telnet with weak credentials, and an SSH pivot to an isolated segment. It fits a free
2-core/4 GB Codespace.

The deliberate choice is that it keeps the **pivot exercise**, which is the one thing
that most distinguishes the lab. The cuts are Active Directory and multi-zone routing —
real losses, but losses of breadth rather than of the core lesson. Isolation is a
property of the network topology, not of scale, so a two-network lite lab enforces it
exactly as the full one does.

**Lesson:** when building a reduced tier, cut breadth and keep the thing that makes the
product distinctive. A lite version that drops the differentiator demonstrates nothing.

### Narrow sudo for WireGuard

Remote lab access needs `wg-quick` and `wg`, which need root. The install asks for
passwordless sudo on an explicitly enumerated command set — `wg-quick`, `wg`, writing
exactly one config path, and `chmod` on that path — rather than blanket sudo.

It is more documentation and a `visudo` step an instructor has to perform. A training
tool that asks for general root is the weakest thing on the machine it is installed on,
and this tool is installed on machines that deliberately host vulnerable services.

**Lesson:** ask for the capability, not the privilege level that includes it.

---

## Operational notes

**`PARTIAL` is a real state and needs surfacing.** The topology graph shows UP, PARTIAL,
or OFFLINE, and PARTIAL is the common case during debugging — the AD DC takes 60 seconds
to provision while everything else is already up. Without a distinct state, an
instructor watching a lab start cannot tell "still coming up" from "something failed."

**First build needs network access to upstream repos.** Images build from upstream bases
including an older Debian release for one vulnerable component, which pulls from archive
repositories. A lab that must come up in an air-gapped classroom needs its images built
and cached first, and that is worth knowing before the morning of a class.

**Nothing persists across `down` → `up`.** Deliberate. A full teardown and rebuild gives
a provably fresh lab, which is the reset path for both a broken scenario and the start of
a new exercise. Persistence between runs would make the lab's state a function of its
history, and the entire value proposition is that it is a function of its config.

---

## Known gaps

- **Packed-string config schema is brittle to free text.** Any value containing `:` or
  `|` breaks parsing. Fine for IPs, ports, and zone names; a blocker for a config shape
  with descriptions in it.
- **No per-container state reset.** The only clean reset is a full `down`/`up`. Resetting
  one host to its initial state mid-exercise is not supported, and would be genuinely
  useful during a scenario.
- **FreeBSD is simulated, not real.** The BSD node approximates FreeBSD conventions on a
  Linux container. Adequate for practising against unfamiliar service layouts and
  credential conventions; not adequate for anything that touches kernel behaviour.
- **AD provisioning is first-boot only.** Changing `ad_config` requires a teardown of
  that container rather than a reconfiguration in place.
- **No automated validation of the config.** A malformed target surfaces as a Docker
  error rather than a schema message, with the exception of AD password complexity. A
  preflight schema check would turn several confusing failures into clear ones.

---

## See also

- **[ARCHITECTURE.md](ARCHITECTURE.md)** — the config model, orchestration, and image library
- **[ALTRON](https://github.com/JohnBullDog/altron-docs)** — the agent platform these labs test, whose own [engineering notes](https://github.com/JohnBullDog/altron-docs/blob/main/docs/ENGINEERING_NOTES.md) cover bugs found by running scenarios here
