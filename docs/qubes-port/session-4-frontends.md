# Session 4: blkfront and netfront (row C)

Read `docs/qubes-port/PLAN.md` first (§2, §4, §5), then `CLAUDE.md`, `AGENTS.md`,
`kernel/core/comps/virtio/` (`virtio-blk`, `virtio-net` and how they register with the `block`
and `network` components), `kernel/core/comps/block/`, `kernel/core/comps/network/`. Branch:
`qubes-port-s4-frontends`. Sessions 2 and 3 are building `aster-xen` (PLAN §4) and the mock in
parallel; code against the §4 traits and, if Session 3's mock is not on your branch, write the
minimal mock you need (same shape as Session 3's description) behind feature `mock`.

## Goal

1. **xenbus device enumeration**: watch `device/vbd` and `device/vif` in xenstore, read
   `backend`, `backend-id`, drive the `XenbusState` machine (`Initialising → Initialised →
   Connected`, and `Closing/Closed` on hot-unplug) exactly as `io/xenbus.h` and Linux's
   `xenbus_switch_state` do; write `feature-*` keys the frontends advertise.
2. **blkfront** (`io/blkif.h`, `xen-blkfront.c`): one ring page (`ring-ref`, `event-channel`,
   `protocol` = `x86_64-abi`), `BLKIF_OP_READ/WRITE/FLUSH_DISKCACHE/WRITE_BARRIER` per
   `feature-flush-cache`/`feature-barrier`, segments granted per request (`BLKIF_MAX_SEGMENTS_PER_REQUEST`
   = 11; optional `feature-max-indirect-segments` later), device size from `sectors` ×
   `sector-size`, `info` flags (read-only, cdrom, removable); registers as a block device
   named `xvda`, `xvdb`, … from `virtual-device` like Linux. Qubes gives a domU `xvda`
   (root), `xvdb` (private), `xvdc` (volatile), `xvdd` (kernel modules, ro).
3. **netfront** (`io/netif.h`, `xen-netfront.c`): one tx and one rx ring (`tx-ring-ref`,
   `rx-ring-ref`, `event-channel`), `request-rx-copy = 1`, `feature-rx-notify = 1`, no SG/TSO
   in the first version (`feature-sg = 0`), MAC from `mac`; rx buffers granted ahead of time,
   tx data copied into granted frames; registers as a network device so the existing
   TCP/IP stack (smoltcp) uses it; link state from `backend`'s `state`. Qubes sets the IP
   configuration through qubesdb, not DHCP; static IP from `xenstore` keys is enough for
   testing (`qubes-ip`, `qubes-netmask`, `qubes-gateway` under `/qubes-*` in Qubes 4.x are in
   **qubesdb**, not xenstore — just expose the device; addressing is userland's job).

## How to test without Xen

- Ring protocol unit tests: request/response encode/decode against the header layouts,
  producer/consumer index arithmetic (`RING_PUSH_REQUESTS_AND_CHECK_NOTIFY`,
  `RING_HAS_UNCONSUMED_RESPONSES`) on in-memory rings; property tests for wrap-around.
- A mock **backend** inside the same VM (behind `mock`): a kernel thread that serves blkif
  requests from an in-memory disk image and echoes netif tx frames back to rx, driven by the
  mock event channels. The NixOS or regression test then mounts an ext2 image through
  `xvda` and pings itself through the vif.
- The xenbus state machine tested on the mock xenstore with an injected backend that moves
  through the states and once closes unexpectedly.
- Both boot regression gates from PLAN §5.

## Deliverables

- `kernel/comps/xen/src/{xenbus.rs,blkfront.rs,netfront.rs}` (or sibling crates; say why).
- `docs/qubes-port/REPORT-s4.md` per PLAN §5, including which `feature-*` keys are advertised,
  the request size limits, and a "Qubes first disk boot" note: `root=/dev/xvda`, the Qubes
  volumes list above, and what a `qvm-volume` of 2 GiB looks like from the frontend.
