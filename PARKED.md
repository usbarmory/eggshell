# Parked — live-session gaps, causes, and what to add first

Status: parked. Eggshell is a nice toy, not functional for daily use; the
author states early-days experimental. Revisit in a few months. This note
records what a live root session actually hit, why, and the order to fix it.

Session transcript (genuine Saphira bootstrap userspace on eggshell `4c2b03c`):

```
root@yolk:~# ping google.com
ping: prctl: Invalid argument
root@yolk:~# sudo ping google.com
sudo: unable to allocate pty: No such file or directory
root@yolk:~# df
df: cannot read table of mounted file systems: No such file or directory
```

## 1. `ping` → `prctl: Invalid argument`

Two stacked causes, either alone is fatal:

- prctl(2) coverage is 4 ops (`PR_SET_NAME`, `PR_SET_PDEATHSIG`,
  `PR_GET/SET_DUMPABLE`, `kernel/sys_proc.go:806-837`). Whatever op the
  ping binary issues next returns `EINVAL`.
- Even past prctl, ICMP needs `SOCK_RAW`. Eggshell is IPv4 TCP/UDP only
  (`kernel/net_gvisor.go`), no raw sockets — documented, by design.
  DNS would be the third hurdle (`/etc/resolv.conf` is present, stack is not).

Why missing: minimal-stub strategy for a lab unikernel; raw sockets are a
deliberate gVisor-scope cut. Not a bug, but ping-style tools can never work
until the stance changes.

## 2. `sudo` → `unable to allocate pty`

- `/dev` has no `ptmx` (`kernel/devices.go` creates null/zero/full/random/
  urandom/tty/console/ttyS0/stdin/out/err/fd/shm/pts; `mknod` rejects
  anything but REG/FIFO/SOCK, `kernel/sys_fs.go:677-685`).
- The kernel already HAS ptys (`kernel.OpenPTY`, used by sshd); only the
  `ptmx` device node + unlockpt path (`TIOCSPTLCK`, `ptsname`) is missing.

Why missing: simply not wired up. This is the cheapest high-value fix:
it unblocks `sudo`, `ssh` clients, `tmux`/`screen`, anything needing a pty.

## 3. `df` → `cannot read table of mounted file systems`

- The bootstrap ships `/etc/mtab -> /proc/self/mounts`. The kernel would
  create `/etc/mtab -> /proc/mounts` (which it serves,
  `kernel/procfs.go:133-137`) but only `if nil` (`procfs.go:85-90`), so the
  bootstrap's link wins.
- `/proc/self` resolves to a per-pid dir that has cmdline/environ/comm/exe/
  cwd/root/stat/statm/status/maps/fd — but no `mounts`/`mountinfo`
  (`procfs.go:221-283`). So the link dangles → `ENOENT` → df gives up.

Why missing: per-pid `mounts` was never added (one `synthText` with the same
static content would do). Second-cheapest fix after ptmx.

## What to add first (in order)

1. `/dev/ptmx` + unlockpt/`ptsname` — unlocks sudo/ssh-client/tmux.
2. Per-pid `mounts` (or make the kernel's mtab win) — unlocks df and
   everything reading `/proc/self/mounts`.
3. Wider `prctl` + an explicit raw-socket verdict (support or documented
   wontfix) — decides whether ping-like tools can ever work.
4. `SYS_SYSLOG` ring buffer — `dmesg` already runs, kernel returns `ENOSYS`.
5. `inotify`/`timerfd`/`signalfd` — the ecosystem unlock (`tail -f`,
   event loops, Go tooling).
6. Robustness before features: COW `fork`, no-panic trap paths
   (`arch_amd64.go:110-112`, `mem.go:98-99`), then SMP.

## Verdict

**PARKED.** Everything in `EGGSHELL-AUDIT.md` and `SAPHIRA-COMPARISON.md`
stands; the above is the live-session footnote. Nothing here contradicts the
author's own framing — early days, experimental. Revisit when 1–3 land
upstream.
