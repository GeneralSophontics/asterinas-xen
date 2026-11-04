# Session 3: the userspace Xen ABI — `/dev/xen/{evtchn,gntalloc,gntdev,xenbus}` (row D)

Read `docs/qubes-port/PLAN.md` first (§2, §4, §5), then `CLAUDE.md`, `AGENTS.md`, the
`device` component and `kernel/core/src/device/`, how `memfd` and `/dev/shm`-style VMOs are
backed (`kernel/core/src/fs/fs_impls/ramfs/memfd.rs`, `kernel/core/src/vm/`), how `ioctl`s are
dispatched (`kernel/core/src/syscall/ioctl.rs` and the ioctl infrastructure from 0.17), and
how `/dev/dri` maps device memory (`kernel/comps/drm`, GEM dumb buffers) — that is the closest
existing example of mmap over kernel-owned frames. Branch: `qubes-port-s3-xen-devices`.

Session 2 is implementing the `aster-xen` crate (PLAN §4) in parallel on another branch. You
code against the §4 traits and provide `kernel/comps/xen/src/mock.rs` behind a feature `mock`:
an in-memory grant table and event-channel table where `grant_access` hands out a gref that
`map_foreign` of the *same* domain resolves to the same frames, `alloc_unbound`/`bind_interdomain`
pair ports inside one VM, `notify` fires the peer's handler, and `Xenstore` is a mutable tree
with watches. The mock must be good enough that a single Asterinas VM under QEMU/TCG can run
two processes that share pages and signal each other through your devices. Keep `lib.rs`
minimal (the §4 signatures and `mock` wiring) so the merge with Session 2 is mechanical.

## Goal

Linux-ABI-compatible device nodes, so that Xen's own `libxenevtchn`, `libxengnttab`,
`libxengntshr`, `libxenstore` and Qubes' `libvchan` work unmodified:

1. `/dev/xen/evtchn` (`include/uapi/xen/evtchn.h`): `IOCTL_EVTCHN_BIND_VIRQ`,
   `BIND_INTERDOMAIN`, `BIND_UNBOUND_PORT`, `UNBIND`, `NOTIFY`, `RESET`, `RESTRICT_DOMID`;
   `read` returns fired ports as `u32`s (and masks them), `write` of ports unmasks; `poll`/
   `epoll` support; ports are closed when the fd closes.
2. `/dev/xen/gntalloc` (`gntalloc.h`): `IOCTL_GNTALLOC_ALLOC_GREF` (allocates `count` frames,
   grants them to `domid`, `GNTALLOC_FLAG_WRITABLE`, returns `index` for `mmap` and the grefs
   array), `DEALLOC_GREF`, `SET_UNMAP_NOTIFY` (clear a byte and/or send an event on unmap);
   `mmap` at `index` maps those frames; the grants end when the mapping and the fd are gone
   (Linux semantics: a grant the remote still maps cannot be ended; Linux keeps the page and
   retries once a second — implement that simply with a timer and a retry list).
3. `/dev/xen/gntdev` (`gntdev.h`): `IOCTL_GNTDEV_MAP_GRANT_REF` (`count` of `(domid, ref)`,
   returns `index`), `UNMAP_GRANT_REF`, `GET_OFFSET_FOR_VADDR`, `SET_MAX_GRANTS`,
   `SET_UNMAP_NOTIFY`; `mmap` at `index` maps the foreign frames (read-only if `PROT_WRITE` is
   absent; the mapping is a VMO backed by a `ForeignMapping`); the dma-buf ioctls return
   `ENOSYS` (`DMABUF_EXP_FROM_REFS`, `IMP_TO_REFS`: document that udmabuf/dma-buf are out of
   scope for the Alice side).
4. `/dev/xen/xenbus` (Linux `xenbus_dev_frontend.c`): the `xs_wire` protocol over `read`/
   `write` with per-fd transactions and watches (`XS_WATCH` replies and watch events
   interleaved like Linux), `poll`; plus `/dev/xen/xenbus_backend` is **not** needed.
5. A `xenstore-read`-compatible path: `libxenstore` must find `/dev/xen/xenbus`; also expose
   `/proc/xen/capabilities` (empty for a domU) because some tools probe it.

Reference: the Linux files named in PLAN §4; fetch each raw from GitHub, do not clone Linux.

## How to test

- A NixOS test crate `test/nixos/tests/xen-devices/` (TCG, `mock` feature on) with
  `extra_config.nix` adding `xen` userspace tools if nixpkgs has them packaged for a guest
  (`pkgs.xen` is heavy; prefer small C programs compiled with the Linux uapi headers that you
  vendor into the test, or Python with `fcntl.ioctl`). Tests:
  `evtchn_unbound_bind_notify_roundtrip` (two processes), `gntalloc_gntdev_share_pages`
  (process A allocs + writes, process B maps via gntdev and reads; then unmap notify),
  `xenbus_read_write_watch` (write a key, a watcher gets the event; transaction abort/commit),
  `gntdev_readonly_segv` (writing a read-only map faults, process dies with SIGSEGV).
- If a NixOS test is too slow to iterate under TCG, do the same with the regression-test
  initramfs (`test/`), but ship the NixOS test in the end.
- Both boot regression gates from PLAN §5.

## Deliverables

- `kernel/comps/xen/src/{mock.rs,devices/...}` (or a separate `aster-xen-dev` crate if that
  separates the merge better; say which and why), registered device nodes, the uapi constants
  with a comment giving the Linux header and version they came from.
- `docs/qubes-port/REPORT-s3.md` per PLAN §5, with a table "ioctl → implemented/stubbed/
  missing" for all four devices, and a note on how `ForeignMapping` was turned into a VMO
  (the one place where Asterinas's memory model and Xen's foreign frames meet).
