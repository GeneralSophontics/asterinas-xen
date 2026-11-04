# Qubes port of Asterinas — working notes

- `PLAN.md`: goal, verified facts, gap table, the shared Xen interface, conventions.
- `session-1-chromium.md`: ungoogled-chromium on Asterinas NixOS, SysV shm, uinput, seccomp.
- `session-2-xen-core.md`: hypercalls, shared_info, event channels, grants, PV console, xenstore.
- `session-3-xen-devices.md`: `/dev/xen/*` with the Linux ioctl ABI.
- `session-4-frontends.md`: blkfront, netfront.
- `REPORT-s<N>.md`: written by each session as it goes.

Sessions 1–4 are independent branches `qubes-port-s<N>-<name>` off `qubes-port` (not
`qubes-port/s<N>`: git refuses a ref `qubes-port/x` while the ref `qubes-port` exists, locally
and on GitHub); 2, 3 and 4 share the interface in `PLAN.md` §4 and are merged by hand
afterwards. Hardware tests (a real Qubes PVH domU) happen after the merge, on Ivan's machines.

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
GitHub, rustup are already allowed). The cloud VM has the Docker client and `dockerd` installed
but **no daemon running**: start it first (`setsid nohup dockerd >/tmp/dockerd.log 2>&1 &`; it
is ready in a few seconds, overlayfs, cgroup v1, 4 vCPUs, 15 GiB RAM, no `/dev/kvm`), and put
that line plus `docker pull asterinas/kernel-dev:0.19.0-20261012` in the environment's setup
script even if the pull exceeds the five-minute cache window. Pick a permission mode that lets
the session run Docker and QEMU without prompts.

Inside the container, HTTPS is re-terminated by the cloud environment's egress proxy, and the
image's `nightly-2026-07-21` ships only the host target, so the first `make` has rustup download
the `*-unknown-none*` targets from `static.rust-lang.org`. That works only if the container
trusts the proxy CA: add
`-v /root/.ccr/ca-bundle.crt:/etc/ssl/certs/ccr-ca.crt:ro -e SSL_CERT_FILE=/etc/ssl/certs/ccr-ca.crt -e CURL_CA_BUNDLE=/etc/ssl/certs/ccr-ca.crt -e CARGO_HTTP_CAINFO=/etc/ssl/certs/ccr-ca.crt -e GIT_SSL_CAINFO=/etc/ssl/certs/ccr-ca.crt -e NIX_SSL_CERT_FILE=/etc/ssl/certs/ccr-ca.crt`
to every `docker run` (without it, `make` dies in `install_osdk` with
`invalid peer certificate: UnknownIssuer` three seconds in). The image is 16.5 GB on disk and
pulls in about four minutes.

Measured in this environment on 2026-11-04 at `2ea21cd` (4 vCPUs, TCG, both PLAN §5 gates
green):

| gate | wall clock | notes |
|---|---|---|
| `make run_kernel ENABLE_KVM=0 AUTO_TEST=boot` | 315 s | includes the rustup target download, the Nix initramfs from the binary cache and the first full kernel build |
| `make run_kernel ENABLE_KVM=0 BOOT_PROTOCOL=pvh AUTO_TEST=boot` | 189 s | incremental build plus boot |

TCG does not expose TSC-deadline, so under QEMU the APIC timer takes the PIT-calibrated path;
a real Xen host exercises the deadline path instead. Neither gate says anything about Xen.
