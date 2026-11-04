# Eggshell vs Linux — Saphira comparison benchmarks

Independent measurement on Homer. Not a reproduction of `BENCHMARKS.md`
(different host, different userland); same binaries, same QEMU machine,
KVM on both sides, 3 rounds, medians with min–max ranges.

## Code under test

- Eggshell rev `4c2b03c` ("reduce duplicated call paths", upstream
  `usbarmory/eggshell` main as of 2026-11-03). Local akadata checkout remains
  at `44ed4d1`; this file records the newer rev actually measured.
- Cross-check via https://syscall.sh/x64/ (371-entry x86-64 table):
  eggshell defines 363 numbers, serves 207, returns `ENOSYS` for 156
  (list verified by script; identical coverage before/after `4c2b03c` —
  the refactor only centralized stub registration). Missing from sysnum:
  the newest kernel additions (335, 336, 443, 453+). `SYS_SYSLOG` (103)
  is defined but has no handler in either rev.

## Userland (proper Saphira proof)

- Rootfs built from the official bootstrap:
  `saphira-linux-v0.bootstrap.tar.xz` (sha256
  `765b2833cbaa3ddebd71bbf01d4d24da290cc15c7ee20020047444f5c3c5b37f2`,
  verified before extraction), trimmed only of `/boot`, `/usr/src`,
  `/usr/share`, `/usr/include`, the compiler backend, volatile dirs and the
  firstboot flag (an installed system has completed firstboot; the OpenRC
  binaries stay in place, unused — eggshell runs `/sbin/init` script).
- Test binaries built with Saphira's own GCC 16.1.0 from that rootfs
  (executed on Homer via the bootstrap loader + wrapper shims, no host
  changes): musl static `posixtest`, `bench`, `ctxtest`, `pingpong`,
  `probe`, `hello`, plus musl dynamic `hello`
  (`PT_INTERP /lib/ld-musl-x86_64.so.1`, verified with `readelf`).
- `/bin/dmesg` is the genuine util-linux binary from the bootstrap. Under
  eggshell it prints `dmesg: read kernel buffer failed: Function not
  implemented` (exit 1): the binary runs, `SYS_SYSLOG` returns `ENOSYS`.
  The `dmesg` facility is unimplemented (both revs), not a rootfs gap.

## Functional re-test on `4c2b03c` (same rootfs)

- `posixtest-musl`: **62 passed, 0 failed**
- `ctxtest-musl`: **5 passed, 0 failed**
- Unsupported-syscall probe (`ptrace`, `inotify_init`, `timerfd_create`,
  `signalfd`, `setns`, `unshare`, `syscall_9999` → `ENOSYS`;
  `mount`, `chroot` → `EPERM`): 9/9 predictable, no crash.
- The I/O-vector refactor in `4c2b03c` (`readv`/`writev` chunking,
  shared `t.send`/`t.recv`) holds for the exercised paths.

## Benchmark setup (identical both sides)

- QEMU `microvm`, 1 vCPU, 2 GiB, `-enable-kvm`,
  `-cpu host,invtsc=on,kvmclock=on`, virtio-net user mode.
- Eggshell: `build/kernel-amd64.elf` (`4c2b03c` + bootstrap rootfs
  embedded), `-append 'quiet ssh=0'`.
- Linux: `/boot/vmlinuz-linux` (Arch `7.2.2-arch1-1`) + same rootfs as
  initramfs (newc cpio via `bsdtar`; a hand-rolled python cpio writer was
  found to hang SeaBIOS and was discarded), `rdinit=/sbin/init`,
  `ip link set lo up` before the run (TCP needs loopback).
- Same `bench-musl` binary (Saphira musl static), `/tmp`-backed files.
- Host: i9-13900K, shared with production VMs (loaded host — absolute
  numbers move with load; the head-to-head ratio is the signal).

## Results (median [min–max], 3 rounds)

| benchmark | unit | eggshell | Linux | eggshell vs Linux |
|---|---|---:|---:|---|
| cpu_xorshift_50M | ms | 77.99 [76.13-93.57] | 89.79 [82.42-95.32] | 1.15× faster |
| clock_gettime | ns | 11.69 [11.69-11.72] | 14.54 [14.48-14.69] | 1.24× faster |
| thread_create_join | us | 8.69 [8.42-10.30] | 9.36 [9.21-9.41] | 1.08× faster |
| mmap_touch_munmap | us | 2.01 [2.01-2.01] | 2.21 [2.21-2.75] | 1.10× faster |
| file_write | MB/s | 8,896.81 [8,392.71-9,740.62] | 7,903.79 [6,516.41-8,261.74] | 1.13× faster |
| file_read | MB/s | 12,831.13 [12,447.88-13,276.40] | 12,736.46 [12,703.04-13,470.59] | 1.01× faster |
| null_syscall | ns | 332.60 [322.06-333.49] | 54.51 [54.06-56.51] | 6.10× slower |
| fork_exit_wait | us | 27.28 [26.91-28.10] | 26.35 [25.86-26.50] | 1.04× slower |
| fork_exec_wait | us | 299.26 [291.63-303.06] | 205.45 [204.18-208.80] | 1.46× slower |
| condvar_pingpong | us | 2.35 [2.02-2.46] | 1.73 [1.67-1.94] | 1.36× slower |
| page_fault | ns/page | 2,058.78 [1,978.74-2,092.53] | 625.88 [328.87-917.82] | 3.29× slower |
| memset_bandwidth | MB/s | 17,253.06 [16,707.22-18,113.98] | 18,582.26 [18,066.17-19,199.66] | 1.08× slower |
| open_close | us | 0.88 [0.87-0.96] | 0.29 [0.29-0.31] | 3.03× slower |
| stat | us | 0.43 [0.43-0.57] | 0.30 [0.30-0.31] | 1.43× slower |
| pipe_latency | us | 2.98 [2.96-3.02] | 1.32 [1.31-1.38] | 2.26× slower |
| unix_latency | us | 2.95 [2.94-2.96] | 1.87 [1.87-1.98] | 1.58× slower |
| tcp_latency | us | 13.66 [13.60-15.73] | 3.37 [3.34-3.44] | 4.05× slower |
| pipe_bandwidth | MB/s | 10,549.81 [10,136.01-10,587.78] | 15,906.92 [13,349.56-16,030.89] | 1.51× slower |
| unix_bandwidth | MB/s | 10,329.40 [10,263.97-10,403.19] | 16,954.93 [15,966.41-17,116.01] | 1.64× slower |
| tcp_bandwidth | MB/s | 4,731.21 [4,221.74-4,850.59] | 10,414.10 [10,112.34-10,944.43] | 2.20× slower |

Direction matches the published table (eggshell ahead on vDSO clock,
user compute, RAM-FS writes; behind on trap-heavy paths and networking).
Two deltas to note: `page_fault` here is 3.29× slower for eggshell while
the published table has it slightly faster — Linux shows wide spread
across rounds (328–917 ns/page) on this shared host, so treat the ratio
as load-sensitive; and absolute MB/s are lower on both sides than
published (loaded host, different CPU).

## Security note (unchanged, re-confirmed code path)

`sshd.go` still grants `NoClientAuth` when `/root/.ssh/authorized_keys`
is absent (the bootstrap rootfs ships none), and the earlier
end-to-end unauthenticated root login stands. Default-open SSH on a
unikernel with no user separation is the highest-severity finding;
see `EGGSHELL-AUDIT.md` §6.

## Reproduce

1. Verify + extract the bootstrap tarball (sha256 above), trim per the
   exclusion list, overlay `/sbin/init` + tests, pack ustar sorted tar.
2. Build `4c2b03c` kernel with the tar as `rootfs/amd64.tar`.
3. Boot per §Benchmark setup; run `/usr/libexec/tests/bench-musl`;
   Linux side needs `ip link set lo up` first. Three rounds, medians.
