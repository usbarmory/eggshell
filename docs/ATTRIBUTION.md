# Third-party data

## `syscalls.json`

- Source: https://syscall.sh/x64/syscalls.json (the x86-64 table)
- Retrieved: 2026-11-03
- Author: TMZ — https://syscall.sh/about/ — https://github.com/guitmz
- Rights: © TMZ (no explicit licence published on the site as of retrieval).

We used this table as an independent cross-check of eggshell's syscall
coverage (`kernel/sysnum_amd64.go` defines 363 numbers; 207 have handlers,
156 fall through to `ENOSYS`; the reference lists 371 entries for current
kernels). Thank you — it saved us from hand-counting, and it is linked
from `SAPHIRA-COMPARISON.md`.

If you are the author and would rather we fetched this at build time
instead of vendoring it, say the word (tmz@syscall.sh) and we will switch
to a download link.
