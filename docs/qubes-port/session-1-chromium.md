# Session 1: ungoogled-chromium on Asterinas NixOS (rows G, H, I; F as a stretch)

Read `docs/qubes-port/PLAN.md` first, then `CLAUDE.md`, `AGENTS.md`, `test/nixos/README.md`,
`book/src/distro/popular-applications/web-browsers/README.md`, `test/nixos/tests/web-browsers/`,
`test/nixos/tests/drm/`, and the `sctrace` docs (`tools/sctrace`, `book/`). Branch:
`qubes-port-s1-chromium`. No Xen is involved in this session; everything is testable here.

## Goal

ungoogled-chromium (`pkgs.ungoogled-chromium` in nixpkgs) runs on Asterinas NixOS under QEMU/TCG:
(1) headless `--screenshot` of a local HTML file, (2) windowed on Xorg (the `drm` test's X setup),
with a screenshot taken from inside the VM proving a rendered page. Along the way, add the
kernel features it needs, in this order of value:

1. **SysV shared memory** (`shmget`, `shmat`, `shmdt`, `shmctl` incl. `IPC_STAT`, `IPC_RMID`,
   `SHM_STAT`; `/proc/sysvipc/shm`; respect the IPC namespace like `sem*` already does).
   Verify with a C or Python test in the NixOS test, and with `ipcs -m`. Xorg's MIT-SHM and
   Chromium's `XShmImagePool` are the consumers; MIT-SHM must work (`xdpyinfo | grep MIT-SHM`,
   and a client actually using it, e.g. `xwd` or chromium itself).
2. **`/dev/uinput`** (optional for Qubes: the gui-agent uses it only when the
   `gui-agent-virtual-input-device` service is enabled, PLAN §2; do it after 1 and 3):
   `UI_DEV_SETUP`/`UI_SET_EVBIT`/`UI_SET_KEYBIT`/`UI_SET_RELBIT`/
   `UI_SET_ABSBIT`/`UI_ABS_SETUP`/`UI_DEV_CREATE`/`UI_DEV_DESTROY`, writes of `input_event`
   delivered to `/dev/input/eventN` and to Xorg (`libinput`/`evdev`). Verify with `evtest` or
   `python-evdev`. Keep the older `UI_DEV_SETUP`-less path (`write` of `uinput_user_dev`) too:
   the Qubes gui-agent may use either.
3. **Chromium's syscall gaps**: run chromium under `sctrace` (and `strace` if it works there)
   with `--no-sandbox --disable-gpu` first, then `--no-sandbox` with software GL, then without
   `--no-sandbox` (neither sandbox layer can work today: user, PID and network namespaces
   return `EINVAL` (PLAN §2) and seccomp is missing; record what Chromium's sandbox probes do
   and whether `--no-sandbox` is the only outcome).
   For every `ENOSYS`/`EINVAL`/`EOPNOTSUPP`/crash, decide: implement (small, clean), stub
   (document why it is safe), or record as "needs row F". `rseq` and `pkey_*` are
   expected to return `ENOSYS` and be tolerated (`clone3` is implemented, PLAN §2); confirm that.
4. **Stretch, only if 1–3 are done and pushed**: seccomp-bpf filter mode
   (`seccomp(SECCOMP_SET_MODE_FILTER)`, `prctl(PR_SET_SECCOMP)`, `SECCOMP_RET_KILL/TRAP/ERRNO/
   ALLOW/LOG`, `SIGSYS` with `si_syscall`, `TSYNC`, `Seccomp:` in `/proc/self/status`). cBPF
   interpreter only; no eBPF. Then Chromium's full sandbox should start.

## Constraints

- Keep `programs.firefox` working; the Firefox test is the regression for this area.
- Chromium under TCG is slow: first start can take minutes. Give it `MEM=8G`, `SMP=4`,
  `NIXOS_TEST_TIMEOUT=60min`, and use `--disable-gpu --no-sandbox --headless=new` for the first
  success before adding difficulty.
- Add a NixOS test crate `test/nixos/tests/web-browsers-chromium/` with `extra_config.nix`
  adding `ungoogled-chromium`, `xorg.xwd`, `xorg.xdpyinfo`, `evtest`, `util-linux` (`ipcs`),
  and the `drm` test's Xorg/XFCE config. Tests: `chromium_headless_screenshot`,
  `shm_sysv_roundtrip`, `uinput_event_roundtrip`, `chromium_x11_window_screenshot`.
- Update `book/src/distro/popular-applications/web-browsers/README.md` per the test README's
  documentation rule.

## Report (`docs/qubes-port/REPORT-s1.md`)

Must contain: the exact sctrace/strace findings table (syscall, args, errno, what you did),
the list of flags Chromium still needs, timings under TCG, and what remains for Qubes: fonts,
`/dev/shm` size, `/proc/sys/kernel/*` reads, anything about `memfd` seals or `SCM_RIGHTS`.
