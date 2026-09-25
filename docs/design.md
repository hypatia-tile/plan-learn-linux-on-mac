# Design

Why the plan for learning Linux has the shape it has. The steps themselves live
in [`learn-linux-on-mac`](https://github.com/hypatia-tile/learn-linux-on-mac);
this file is the reasoning behind them, and it is the only thing this
repository holds.

## 1. The subject is the delta, not Linux

macOS is already Unix. `ls`, `grep`, pipes, redirection, file permissions and
processes are the same on both sides, and time spent on them here buys nothing.

What macOS does not teach is the part that makes a machine a machine: an init
system that owns every service, a package manager that owns every file under
`/usr`, a journal that every service logs into, a network stack configured by
text files, and block devices you partition yourself. That delta is the
subject.

The goal is **operator-level competence**: being able to run a service on a
Linux machine and, when it stops working, find out why.

## 2. The real skill is diagnosis, not construction

An operator's competence is not "can build a working system". It is "can find
the cause when a working system stops working". These are different skills and
only one of them is hard.

This matters because it disqualifies the obvious way to structure a curriculum.
A step whose completion criterion is "the service is running" trains
construction and nothing else. Every step here is instead completed by a
diagnosis: the service is broken deliberately, and the step is done when the
cause has been found **and correctly named**, not merely when the service runs
again. A lucky fix that restores service without explaining it is a failure.

## 3. Faults are injected by the AI, not by the learner

A fault you inflicted on yourself cannot be diagnosed — you already know the
answer. You can rehearse the commands, but `systemctl status` tells you nothing
you did not already know, which is precisely the experience that real
troubleshooting does not resemble.

So the AI breaks the machine and does not say what it broke. This is the one
mechanism in the plan that cannot be replaced by discipline, because the
blindness has to be structural: a "don't read the script you are about to run"
rule fails the moment curiosity wins.

### The blindness ramp

Full blindness is training for someone who already knows the tools. Before
that, it is just a stopped hand. So blindness is graded in three phases:

| Phase | Steps | What the learner is told |
|---|---|---|
| 1 | 0–4 | Nothing, or the exact surface ("one line in the unit file was changed") |
| 2 | 5–8 | The area only ("either the user or the permissions") |
| 3 | 9–16 | Nothing at all |

### The hint timebox

Hints are graded and clock-driven, not mood-driven: after 30 minutes stuck, a
first-level hint (where to look, nothing more); after another 30, a second
(narrow the area); only then the answer. The clock is external on purpose —
without it, "have I tried hard enough?" is decided by how tired you are.

## 4. The machine is a disposable VM, not a container

A Docker container is a bad teacher for everything in §1. PID 1 is your own
process, so there is no init system to learn; there is no journal, no boot, no
network configuration and no mount management. What a container teaches is how
to build an image, which is a different subject.

Colima is already installed and is, underneath, a real Linux VM. So the machine
is a **second colima profile**, `linuxlab`, kept separate from the default
profile so that breaking it does not take the working Docker setup with it.
`colima delete -p linuxlab` is a supported outcome, not an accident.

Rejected alternatives:

- **Lima directly** would allow choosing the distribution and writing the
  cloud-init by hand, which is more honest, but it requires changing
  `dotfiles-mac` first. It stays available as a later move.
- **A VPS** is the only way to get real DNS, real certificates and real
  attackers in the log. It also costs money and makes "break it" expensive.
  Same: a later move, not the starting point.
- **`--runtime incus`** would make throwaway system containers cheap, but adds
  Incus itself to the list of things being learned.

### Containers still have two jobs

Not as the subject, but as tools:

1. **A throwaway comparison bench.** `docker run -it debian` beside
   `alpine` and `fedora` shows apt against apk against dnf, and GNU coreutils
   against busybox, in seconds and without booting anything. This runs on the
   *default* profile, on the host, and never touches `linuxlab`.
2. **A dissection subject, once, at the end.** Step 16 installs a runtime
   inside `linuxlab` by hand and looks at what a container actually is —
   `/proc/<pid>/ns/*` and the cgroup tree — from the machine's side. cgroups
   are systemd's own mechanism, so this lands back on the main subject rather
   than wandering off it.

`linuxlab` therefore starts with **no container runtime at all**
(`--runtime none`). Installing one by hand at Step 16 is itself a package
management exercise, and until then the VM stays free of a service nobody is
studying.

## 5. The isolation boundary

The AI needs `sudo` inside `linuxlab` to inject faults. That is only acceptable
if `linuxlab` cannot reach anything that matters.

Colima's own embedded configuration states the default:

```
# Colima mounts user's home directory by default to provide a familiar
# user experience.
# Colima default behaviour: $HOME is mounted as writable.
```

Writable, not read-only. Left alone, a VM the AI has root on would have write
access to the entire host home directory. So `linuxlab` is started with
`--mount none`, and confirming that the mount is genuinely absent is part of
Step 0 rather than something taken on trust.

The boundary is then:

- **Inside `linuxlab`** — the AI may do anything, including `sudo` and
  including destroying the VM.
- **On the host** — unchanged from every other repository: read-only commands
  only.

The cost of `--mount none` is that files cannot be handed to the VM through a
shared directory. They arrive by `git clone` over the network instead, which is
how software reaches a real server anyway. The inconvenience is pointing in the
right direction.

One more colima default worth knowing before it surprises someone:
`sshConfig: true` means colima edits `~/.ssh/config` on start.

## 6. The spine is one service, grown

The steps are not independent topics. They are one service — a hand-written,
deliberately tiny HTTP daemon in C — acquiring what a real service needs: a
unit file, its own user, a log, a port, a firewall, a data volume, a backup, a
reverse proxy in front of it.

**The daemon is written by hand, and not installed with `apt`, for one reason.**
Choosing `User=`, `Restart=`, `WorkingDirectory=` and the dependency ordering
*is* the systemd lesson. `apt install nginx` skips all of it: the unit arrives
pre-written and correct, and the decisions that would have taught something
were made by the packager. nginx does appear, at Step 12, but as the second
service — the one that makes unit-to-unit dependencies real.

C rather than Go or a shell script, because C is already familiar ground and
because dynamic linking is not optional: `ldd`, `ldconfig`, rpath and "the
library is missing on the target" are among the most common real deployment
failures. A static Go binary makes deployment easy, and easy deployment teaches
nothing. A shell script skips the build entirely.

## 7. Ordering is by difficulty, and two things follow from that

**`journalctl` comes before the first fault.** It is the instrument, not the
subject. A curriculum that breaks a unit file at Step 2 and teaches log reading
at Step 4 has the order backwards: the first diagnosis stalls on not knowing
how to look, which teaches nothing about systemd.

**The filesystem hierarchy is a placement decision, not a table.** There is no
step that recites what `/var/lib` is for. Instead, Step 3 requires deciding
where the binary, the configuration and the daemon's mutable state each go, and
the distinctions between `/usr/local/bin`, `/etc`, `/var/lib`, `/var/log` and
`/var/cache` are learned by having to choose. (The `/etc` reached through Nix
is a symlink farm; an ordinary Linux `/etc`, written by packages and edited by
the administrator, is a different thing, and Step 1 looks at both.)

Dynamic linking lands late, at Step 14, for the same difficulty reason, and
because by then the daemon has a reason to grow a library dependency.

## 8. The VM is grown, then rebuilt on purpose

Growing one machine and rebuilding it from scratch pull in opposite directions,
and both are needed.

Colima keeps its instance state and generated `colima.yaml` under `~/.colima`,
which is outside Nix's management — so nothing about `linuxlab` is reproducible
except what `lab.md` records. Meanwhile a VM that has absorbed a dozen injected
faults accumulates residue that muddies the next diagnosis.

So: grow it, and at two checkpoints (Steps 7 and 15) delete it and restore the
current state from `lab.md` and the repository alone. Failing to restore is the
point of the exercise — it means the procedure was incomplete, and that is a
finding, not a setback.

## 9. Two repositories

| | Holds | Why |
|---|---|---|
| `plan-learn-linux-on-mac` | this file | The reasoning is stable; the steps are not |
| `learn-linux-on-mac` | `docs/roadmap.md`, `docs/lab.md`, `src/`, `etc/`, the issues, the reviews | Reviews are pinned to commit hashes, so the issues must live where the commits live |

Splitting the issues from the commits would break exactly that pin, which is
why the roadmap moved to the code repository rather than staying beside this
file.

## 10. Out of scope, deliberately

- **Boot loader and initramfs.** Lima boots a cloud image; breaking GRUB
  produces `colima delete`, not a diagnosis. Low return for the risk.
- **Kernel building and modules.** Possible, heavy, and a different subject
  from §1.
- **Dockerfiles, image layers, compose, registries.** Useful, but they would
  eat the step budget that the operator subject needs.
