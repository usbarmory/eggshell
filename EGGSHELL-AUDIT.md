# EGGSHELL INDEPENDENT ENGINEERING AUDIT

Operator tasking: source audit, build, boot, functionality, independent benchmarks,
security assessment. Diagnostic changes kept separate; upstream source untouched.

## 1. Exact revision and test environment

- Git revision: `44ed4d19aa35d8614c72ec7c8b1c698a4f1957dd` (`44ed4d1`, 2026-11-03, `tidying`)
- Remote: `git@github.com:akadata/eggshell.git` (fetch/push) — note: mission cites
  `github.com/usbarmory/eggshell`; local clone points at an `akadata` fork. No upstream
  diff was performed; treat as the tested tree.
- Workdir: `/scratch/eggshell`, branch `main`, `git status` clean except ignored
  artifacts (`build/*`, `rootfs/*.tar`, `*.so`, `bench/build`, per `.gitignore`).
- Host (Homer): Arch Linux, kernel `7.2.2-arch1-1`, x86-64, i9-13900K, 32 threads,
  125 GiB RAM. QEMU `11.1.2` (published table claims `11.1.3` — minor delta).
  Go `go1.27.3-X:nodwarf5`, GCC `16.2.1`, `cc`/`gcc` present, `musl-gcc` absent,
  `nix-build` absent, `/nix` absent, `/dev/kvm` present, no `tap0` (user-net used).
- QEMU machine (all runs): `microvm,x-option-roms=on,pit=off,pic=off,rtc=on`,
  `-smp 1`, `-m 2G`, `-enable-kvm`, `-cpu host,invtsc=on,kvmclock=on`,
  `virtio-net-device + user net 10.0.0.0/24`, `isa-debug-exit`. Linux control runs
  used the identical machine except where noted (one early Linux run omitted
  `-enable-kvm` and is discarded as TCG).
- Rootfs: NO nix rootfs available. Built minimal Saphira-musl rootfs from
  `packages.saphira.vm2.uk/hatched/x86_64` APKs (musl 1.2.6-r6, bash 5.3-r3,
  coreutils 9.7-r6, ncurses 6.5-r11, readline 8.3-r7, saphira-baselayout 0.1-r65),
  overlaid with `userland/skel` (`/sbin/init`, `/etc`, `/root/.bashrc`), plus
  host-compiled static tests (`cc -static`). `rootfs/amd64.tar`: 28.5 MiB, ustar,
  sorted, `--mtime=@1`, `tar -C /tmp/eggshell-root ... -cf rootfs/amd64.tar .`.
  Kernel `build/kernel-amd64.elf`: 35.4 MiB. Non-usr-merged FHS preserved
  (`/bin`, `/sbin`, `/lib/ld-musl-x86_64.so.1 -> /usr/lib/libc.so`, `/usr/lib/libc.so`).
  No `busybox`, no `redis`, no Go toolchain, no `hello-dynamic` (nix `withTests`/
  `withGo` equivalents absent) — recorded as Saphira-rootfs limits, not Eggshell faults.

## 2. Architecture findings (source refs)

- Bare-metal, no Linux guest — VERIFIED. `main.go:6-9,49-60` builds `GOOS=tamago`;
  `board/microvm/board.go:65-89` sets pool/console/clock; `kernel/kernel.go:113-154`
  `Start()` unpacks tar, spawns init, blocks on `init.done`. User ELFs loaded by
  `kernel/elf.go:48-170`. No VMEXIT to a guest kernel.
- Ring 0 Go / ring 3 apps — VERIFIED mechanism with hardening gaps. GDT/TSS/CS/DS
  `monitor/monitor_amd64.go:62-69,199-225`; `EFER.SCE|NXE`, `STAR/LSTAR/SFMASK`
  `:241-246`; IDT all-vectors-trap `:227-239`; entry `exec/enter` saves Go state,
  loads user FS/GS, `LIDTR uIDT`, `CR3=root`, `IRETQ` (`monitor_amd64.s:33-165`);
  exit via `syscallEntry/intrCommon/kreturn` (`:167-267`). Leaf PTEs `pteUS`,
  kernel halves are supervisor-only TamaGo identity copies (`kernel/mmu_amd64.go:69-77,
  124-159`); no `CR0.WP`/`CR4.SMEP/SMAP` use found; `NXE` set. `CS&3!=3` panics the
  whole Go runtime (`kernel/arch_amd64.go:110-112`, no recover) — one bad user CR3/RIP
  kills all processes; `putPage` refcount-0 panics (`kernel/mem.go:98-99`).
- Syscall ABI — PARTIALLY (broad but not complete). 363 numbers defined
  (`kernel/sysnum_amd64.go:12-375`); 207 table entries, of which 3 explicit ENOSYS
  stubs (`CLONE3`, `GET_ROBUST_LIST`, `RSEQ`) and ~204 functional
  (`kernel/syscall.go:16-61`, `kernel/sys_proc.go`, `sys_fs.go`, `sys_mm.go`,
  `poll.go`, `net.go`, `arch_amd64.go`). ~156 numbers fall through to ENOSYS.
  Correctly returns ENOSYS/EPERM for `ptrace`, `inotify`, `timerfd`, `signalfd`,
  `setns`, `unshare`, `mount`, `chroot` (tested §4). Known shortcuts: `GETDENTS`
  aliased to `GETDENTS64` (`arch_amd64.go:512`); `ACCESS` only checks `X_OK`
  (`sys_fs.go:1179-1198`); `UID/GID/CAP/SETGROUPS` nop/fake (`sys_proc.go:239-258`);
  `MLOCK*` nop, `MINCORE=1`, `MADV_*` mostly silent (`sys_mm.go:89-150`);
  `FUTEX` subset only (`sys_proc.go:918-971`); `EPOLLET` stored but level-triggered
  (`poll.go:21`, README admits); `FALLOCATE` extend-only; `SENDFILE` single 1M round.
- ELF — VERIFIED for tested cases, single-level only. `loadELF` maps `PT_LOAD`,
  records `PT_INTERP`, ignores `PT_DYNAMIC/PT_TLS/PT_GNU_*` (`elf.go:93-162`);
  `execve` handles `#!` (depth 4), loads PIE at `0x2800000000`, interp at
  `0x2000000000`, rejects nested interp (`task.go:498-601`); `setupStack` emits
  standard auxv incl. `AT_SYSINFO_EHDR` when vDSO mapped (`elf.go:172-263`).
  Dynamic Saphira musl binaries (`/bin/bash`, `/bin/ls`, `/bin/cat`, interp
  `/lib/ld-musl-x86_64.so.1`) run. No `LD_*` search, no multi-level loader.
- MM/pages/faults — VERIFIED for tested paths. Demand-zero/file-fill only, no COW
  (fork is eager copy `mm.go:815-877`), no swap, no `GROWSDOWN` (`mm.go:630-672`);
  `mmap/munmap/mprotect/mremap/brk` (`mm.go:367-626`, `sys_mm.go:9-87`);
  `MAP_SHARED` of content-backed files silently becomes private (`sys_mm.go:51-60,427-428`).
- Proc/threads/futex/signals/sched — VERIFIED functionally, single-core.
  `clone/fork/vfork/execve/exit/wait` (`task.go:394-496,498-658,660-853`);
  each thread is a goroutine, blocking syscalls map to Go scheduler
  (`kernel.go:6-12,66-80`); quantum 10 ms, LAPIC one-shot, `goos.Idle` HLT
  (`monitor_amd64.go:284-331`, `task.go:245-263`); `README:160-161` single-core only,
  `SCHED_AFFINITY=cpu0`, `GETCPU=0` stubs. Signals: full disposition/mask/frame/
  `sigreturn`/`sigsuspend`/`itimer` path (`signal.go`, `arch_amd64.go:165-332`).
  Futex keyed by phys addr (`futex.go:32-38`) — wrong for file-backed SHARED (pages
  are private copies until writeback), and global linear scan wakes broadly.
- VFS/net/TTY — PARTIALLY (as documented plus verified). Volatile tmpfs single mount,
  tar unpack (`filedata.go`, `tar.go`, `vfs.go`); no block/persist/sync (all sync ops
  nop). `/proc`/`/dev` curated subsets (`procfs.go`, `devices.go`); no `/proc/net/*`,
  no `ptmx`, no disk nodes; `mknod` rejects non-REG/FIFO/SOCK. Pipes/FIFO (64 KiB,
  `PIPE_BUF` atomic, `SIGPIPE`), `poll/select/epoll/eventfd` level-triggered,
  PTY with line discipline/job-control subset (`TOSTOP`/job signals missing).
  virtio-net + gVisor (`network/*`, `net_gvisor.go`): IPv4-only, TCP+UDP only, no
  raw/seqpacket, no `SCM_RIGHTS`/cmsg, no unix-datagram semantics — matches README
  limits. Loopback TCP works (pingpong/tcp tests pass).
- SSH — PRESENT and functional; default is unauthenticated (see §6).

## 3. Build results

- `kernel/vdso/build.sh`: PASS with host gcc (`vdso_amd64.so: 2384 bytes`).
- TamaGo `go1.27.3` toolchain bootstrap via `go tool tamago`: PASS (clones
  `tamago-go1.27.3`, builds with host Go). No host config change.
- `go mod tidy` / module cache: present (`tamago`, `go-net`, `gvisor` in cache).
- `make rootfs` (documented nix path): BLOCKED — `nix-build: command not found`,
  `/nix` absent. Not installed per operator order (no Nix).
- Saphira-APK rootfs path (operator-approved): PASS as described in §1.
  First boot failed on missing `libncursesw.so.6` (`Error loading shared library
  ... needed by /bin/sh`, plus `tgoto/tgetflag/...` reloc errors) — fixed by adding
  `ncurses` + `readline` APKs. Rebuilt tar + `make kernel`: PASS, no warnings shown.
- Artifacts: `build/kernel-amd64.elf` 35.4 MiB, `rootfs/amd64.tar` 28.5 MiB,
  both gitignored (diagnostic, not upstream).

## 4. Functional tests (Saphira rootfs, `quiet ssh=0`, KVM; all isolated user-net)

- Boot to userspace: PASS. Banner + motd + `root@yolk:~#`; `uname -a` →
  `eggshell yolk tamago go1.27.3 x86_64 GNU/Linux`.
- Bash + coreutils (dynamic musl): PASS (`/bin/bash --login`, `/bin/ls`, `/bin/cat`,
  `/bin/echo`, `head`, `hostname` content `yolk`).
- Static executables (host `cc -static` glibc): PASS (`hello` prints pid/argv).
- Dynamic musl: PASS (bash/ls/cat are `interpreter /lib/ld-musl-x86_64.so.1`).
- `posixtest` (project suite, static build): **62 passed, 0 failed** — fork, waitpid
  (exit/signaled/ECHILD), pipe, fork+exec+pipe, popen, system, signals (handler,
  mask/pending, alarm+pause, SIGSEGV recovery, EINTR, EPIPE), pthreads + TLS,
  mmap/demand/munmap/malloc/MAP_SHARED-fork/CoW/mprotect-fault, files
  (chdir/getcwd/O_CREAT/rw/lseek/fstat/pread/ftruncate/mmap/rename/link/symlink/
  lstat/mkdir/readdir/ENOTEMPTY/unlink/ENOENT/stdio/dup), poll/select/epoll(+timeout)/
  NONBLOCK/socketpair-io, TCP reuse-after-peer-close, usleep/clocks/FP/vDSO/agreement,
  FPU-across-syscalls. Matches project's claimed POSIX scope for these areas.
- `ctxtest`: **5 passed, 0 failed** (TLS/FS, GS/WRGSBASE, per-thread TLS, signal XMM).
- `pingpong`: pipe 3.0, unix 2.8, tcp-loopback 13.9 µs/roundtrip (same order as
  published magnitudes; single run, not a claim).
- `probe`: user 1ns, trap_asm 284ns, monitor_go 279ns, getppid 283ns, getpid 294ns.
- `bench` (project `userland/tests/bench.c`, static glibc build, single run):
  cpu 77.99ms, null 305.93ns, clock 11.55ns, fork_exit 45.10µs, fork_exec 381.85µs,
  thread 9.13µs, condvar 2.80µs, page_fault 2050.74ns/page, memset 49927MB/s,
  mmap_touch 2.02µs, open_close 0.83µs, stat 0.75µs, file_write 9367MB/s,
  file_read 12987MB/s, pipe_lat 2.87µs, unix_lat 2.88µs, tcp_lat 13.69µs,
  pipe_bw 10420MB/s, unix_bw 10326MB/s, tcp_bw 4896MB/s, `BENCH done`.
- Negative tests (own `negtest`, static): **9/9 predictably fail without crash** —
  `ptrace/inotify_init/timerfd_create/signalfd/setns/unshare/syscall_9999` →
  `ENOSYS (38)`; `mount/chroot` → `EPERM (1)`.
- NOT tested (blocked, not invented): BusyBox applets (no busybox in Saphira rootfs;
  coreutils used instead), `hello-dynamic` (nix `withTests` artifact absent),
  `redis*`/`go-build*`/`tar-go-src`/`gzip-go-binary` (no redis/Go/gzip in minimal
  rootfs), `u-root`, `sftp`, `SCM_RIGHTS`/unix-datagram/raw sockets (documented absent).

## 5. Benchmarks — published reproduction NOT claimed; Saphira-compat comparison only

- Published methodology (`BENCHMARKS.md` + `bench/run.py`): same `microvm` 1vCPU/2G
  KVM `-cpu host`, same nix userland as initramfs, `rounds=3` medians, `quiet ssh=0`
  vs `quiet`, helpers `tm`/`fio` + `guest.sh` + `linux-init`. Requires nix rootfs
  (unavailable — no Nix per order), Ubuntu `7.0.0-38-generic` kernel (unavailable;
  we have Arch `vmlinuz-linux` / `7.2.2-arch1-1`), redis + Go toolchain in guest
  (absent from minimal Saphira rootfs). **Reproduction BLOCKED; no published numbers
  are quoted here as our measurements.**
- What was done instead (clearly separate): same Saphira rootfs tar converted to
  cpio initramfs, same QEMU `microvm` 1vCPU/2G KVM `host,invtsc=on,kvmclock=on`,
  same static `bench` binary, single round each, Eggshell (`quiet ssh=0`,
  `build/kernel-amd64.elf`) vs Linux (`/boot/vmlinuz-linux`, `rdinit=/sbin/init`).
  Host: i9-13900K (faster than published i7-1165G7), so absolute numbers are not
  comparable to the published table.
- Single-round Saphira-compat results (Eggshell vs Linux-KVM, higher MB/s or lower
  time wins): cpu 77.99 vs 69.83ms (Linux faster); null 305.93 vs 52.94ns (Linux
  ~5.8×); clock 11.55 vs 14.33ns (Eggshell faster, vDSO); fork_exit 45.10 vs 32.14µs;
  fork_exec 381.85 vs 240.37µs; thread 9.13 vs 6.34µs; condvar 2.80 vs 1.35µs;
  page_fault 2050.74 vs 1546.12ns/page (both far above published 762/887 — glibc-static
  binary + different host; single round); memset 49927 vs 51220MB/s; mmap_touch 2.02
  vs 2.28µs; open_close 0.83 vs 0.29µs; stat 0.75 vs 0.18µs; file_write 9367 vs
  6948MB/s (Eggshell faster, RAM-FS); file_read 12987 vs 12855MB/s; pipe_lat 2.87 vs
  1.34µs; unix_lat 2.88 vs 1.90µs. Linux TCP tests hung because minimal Linux init
  leaves `lo` down (no `ip`/`ifconfig` in rootfs; project's `linux-init` does
  `ip link set lo up`) — Eggshell TCP works via gVisor loopback (see §4). No medians,
  no variance, no redis — NOT a reproduction.
- `bench/run.py` gaps noted for any future rerun: nixpkgs unpinned (`nixos-unstable`
  tarball), Go release live-fetched, host/governor/pinning/mitigations unspecified,
  `rounds=3` median-only, fixed eggshell→Linux order, `pit`/`cpu` diverge KVM vs TCG,
  `tm` silences children and `PERF FAIL` lines vanish from medians, `fio` best-of-3
  reads vs single-shot writes, `REDISP16` 10×N pipelined over gVisor vs Linux TCP.

## 6. Security findings (evidence + severity)

- **S1 — Unauthenticated root SSH by default (HIGH, verified end-to-end).**
  `sshd/sshd.go:15-16,102-136`: if `/root/.ssh/authorized_keys` is absent or parses
  to zero keys, `cfg.NoClientAuth=true`. `nix/default.nix:20-21,94-96,200-205`:
  `sshAuthorizedKeys=[]` default and the image omits the file unless non-empty —
  so default builds are open. Our Saphira rootfs likewise has no authorized_keys.
  Observed: `sshd: listening on port 22`, banner `SSH-2.0-Go`, then from host
  `ssh -p 2222 root@127.0.0.1 'echo SSH_OK'` with `BatchMode=yes` (no key, no
  password) → `rc 0`, `STDOUT SSH_OK`; guest logs `sshd: 10.0.0.2:...: connected
  (SSH-2.0-OpenSSH_10.6)`. Test used loopback-only user-net forward
  (`hostfwd=tcp:127.0.0.1:2222-:22`), no public exposure. With `tap0`/bridged net
  the same listener would be network-reachable. Fix direction: default `ssh=0` or
  require a key; at minimum warn loudly on console when `NoClientAuth` is active.
- **S2 — Single-user, no privilege separation (MEDIUM, by design, documented).**
  README admits: only single user, identity ops are nop (`sys_proc.go:239-258`,
  `CAPGET` fake). All processes run as uid 0 with one shared root FS. Verified by
  code; no bypass needed — there is no boundary to bypass between users.
- **S3 — Wide syscall stubs that silently succeed (MEDIUM).** `MLOCK*`, `SCHED_*`,
  `FLOCK`, `SYNC*`, `MADV_*` (non-DONTNEED), `MINCORE=1`, `EPOLLET`-as-level,
  `F_GETLK`-unlocked etc. return success without doing the thing. Software that
  depends on the semantics (memory locking, affinity, file locks, durability) will
  believe it is protected. Prefer explicit `ENOSYS`/documented deviation for
  security-relevant stubs.
- **S4 — Kernel-wide DoS on malformed user state (LOW-MEDIUM).** `CS&3!=3` panic
  (`arch_amd64.go:110-112`), `putPage` refcount panic (`mem.go:98-99`), no recover
  around `enterUser` except per-syscall `safeSyscall`. A single malicious/buggy
  thread can halt the whole unikernel (all processes). Acceptable for a lab
  unikernel, not for multi-tenant use.
- Scope note: source review + black-box boot/SSH tests only; not a formal proof.
  No public-network exposure was performed; VM stayed on isolated user-net/loopback.

## 7. Documented vs observed limitations

Documented (README:160-175) and confirmed: single-core supervision; single user /
identity nop; IPv4-only, no raw, no `SCM_RIGHTS`, no unix datagrams; `EPOLLET`≈level;
no `ptrace`/SysV/`inotify`/`timerfd`/`signalfd`/timers/mounts/namespaces; volatile
in-memory FS. All confirmed by code (`syscall.go:55-62` ENOSYS default; `mount/
umount2/chroot` EPERM; `clone3/rseq` ENOSYS) and by `negtest` (ENOSYS/EPERM, no crash).
Observed beyond docs: `MAP_SHARED`-file silently private; futex keying wrong for
shared-file pages; `GETDENTS` ABI alias; `ACCESS` R/W always allow; `ARCH_SET_GS`
refused while `WRGSBASE` from ring 3 remains possible; `CLOSE_RANGE` ignores
`UNSHARE`; `MEMFD_CREATE` ignores seals; `SENDFILE` 1 MiB cap; `LSEEK` on chr-dev
returns 0; `STATX` fixed mask; `EXECVEAT` ignores flags; page-fault path single-round
~2 µs/page on this host/binary (vs published sub-µs — different stack, not a
contradiction). SSH default-open (§6) is documented in code (`sshd.go:15-16`) and
`nix/README.md:17` but deserves a README warning next to the `ssh=` option.

## 8. Outstanding questions + how to reproduce

- Obtain/verify the `usbarmory/eggshell` vs `akadata/eggshell` relationship; re-pin to
  the exact upstream commit before any comparative claim.
- Full published reproduction needs: Nix (or a byte-identical nix rootfs with
  BusyBox+redis+Go+tests), Ubuntu `vmlinuz-7.0.0-38-generic` + config, i7-1165G7-class
  host (or renormalized analysis), `bench/run.py --linux <vmlinuz> --rounds 3`,
  multiple rounds with variance, `tun/tap` vs user-net explicitly recorded.
- Saphira path: add `iproute2` (bring `lo` up under Linux for TCP parity), `gzip`,
  `redis`, Go toolchain, and a musl `hello-dynamic` built with the Saphira toolchain
  to complete `guest.sh`; rerun Eggshell-vs-Linux with `rdinit=/root/bench/linux-init`.
- Reproduce: `cp <rootfs>.tar rootfs/amd64.tar; make kernel` (needs `vdso` gcc);
  boot isolated: `qemu ... -kernel build/kernel-amd64.elf -append 'quiet ssh=0'`;
  run `/usr/libexec/tests/posixtest`, `/ctxtest`, `/bench`, `/pingpong`, `/negtest`;
  SSH test only on loopback with `ssh=22` + `hostfwd=tcp:127.0.0.1:2222-:22`.

## Verdicts (per significant project claim)

- TamaGo bare-metal execution, no Linux guest kernel: **VERIFIED**.
- Ring 0 Go runtime / ring 3 app isolation (GoTEE-style switch, paging, faults):
  **VERIFIED** as implemented; hardening gaps noted (§2/S4) but no isolation break found.
- Linux syscall ABI + "run unmodified POSIX C on musl": **PARTIALLY VERIFIED** —
  broad coverage (~204 handlers, static+dynamic run, 62/62 posixtest, 5/5 ctxtest),
  but ~156 numbers ENOSYS plus documented exclusions; not complete Linux.
- Static and dynamic musl linking: **VERIFIED** for tested Saphira musl binaries
  (bash/ls/cat dynamic, static tests); nix `hello-dynamic` artifact itself not tested.
- Memory management / page tables / protection / faults: **VERIFIED** for exercised
  paths (mmap/mprotect/munmap/brk/faults all pass).
- Processes/fork/exec/threads/futexes/signals: **VERIFIED** (posixtest signal/thread/
  futex-driven condvar/pingpong all pass, single-core).
- VFS/networking/SSH functional: **PARTIALLY VERIFIED** — in-memory FS, pipes,
  PTY, poll/epoll, IPv4 TCP/UDP + loopback work; SSH works but defaults to
  unauthenticated root (S1); raw/SCM_RIGHTS/datagram/timerfd/signalfd/inotify/
  mounts/namespaces absent as documented.
- Scheduling/preemption/concurrency: **VERIFIED** single-core (10 ms quantum,
  LAPIC preempt, Go-scheduler mapping); SMP supervision remains future work.
- Published benchmarks: **NOT VERIFIED** — methodology not matched (different host,
  different rootfs/binaries, single rounds, missing redis/Go/TCP-parity). Separate
  Saphira-compat single-round numbers are provided in §5 with blockers; trend
  (Eggshell strong on vDSO/clock/mem/file-write, slower on trap-heavy fork/syscall/
  IPC/TCP) is consistent with the published direction but is NOT a reproduction.
- Security posture ("experimental", SSH present): **PARTIALLY VERIFIED** — isolation
  mechanism holds for tested cases, but unauthenticated-root-by-default (S1),
  no-op privilege separation (S2), silent-success stubs (S3), and panic-on-bad-state
  DoS (S4) mean it must not be exposed or treated as hardened.
