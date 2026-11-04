# Qubes port of Asterinas — working notes

- `PLAN.md`: goal, verified facts, gap table, the shared Xen interface, conventions.
- `session-1-chromium.md`: ungoogled-chromium on Asterinas NixOS, SysV shm, uinput, seccomp.
- `session-2-xen-core.md`: hypercalls, shared_info, event channels, grants, PV console, xenstore.
- `session-3-xen-devices.md`: `/dev/xen/*` with the Linux ioctl ABI.
- `session-4-frontends.md`: blkfront, netfront.
- `REPORT-s<N>.md`: written by each session as it goes.

Sessions 1–4 are independent branches `qubes-port/s<N>-*` off `qubes-port`; 2, 3 and 4 share
the interface in `PLAN.md` §4 and are merged by hand afterwards. Hardware tests (a real Qubes
PVH domU) happen after the merge, on Ivan's machines.

## Launching a session

Fork: `github.com/generalsophontics/asterinas-xen`; cloud environment `asterinas-xen`.

From the browser (claude.ai/code): repository `generalsophontics/asterinas-xen`, environment
`asterinas-xen`, one session per brief with the prompt

```
git checkout qubes-port first. Then do docs/qubes-port/session-2-xen-core.md;
start by reading docs/qubes-port/PLAN.md.
```

From milky: `cd ~/asterinas-qubes && claude --cloud "<same prompt>"`.

Cloud environment prerequisites: network access must allow `cachix.org` and `*.cachix.org`
(the Asterinas binary caches) in addition to the default list (Docker Hub, `*.nixos.org`,
GitHub, rustup are already allowed); a setup script of
`docker pull asterinas/kernel-dev:0.19.0-20261012` is worth adding even if it exceeds the
five-minute cache window. Pick a permission mode that lets the session run Docker and QEMU
without prompts.
