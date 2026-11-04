# Session 2: Xen guest core for PVH (rows A, B, E) — `aster-xen`

Read `docs/qubes-port/PLAN.md` first (§2 facts, §4 interface, §5 conventions), then `CLAUDE.md`,
`AGENTS.md`, `ostd/src/arch/x86/boot/pvh/`, `ostd/src/arch/x86/boot/bsp_boot.S`,
`ostd/src/arch/x86/kernel/tsc.rs`, `kernel/core/comps/time/src/rtc/kvmclock.rs`,
`ostd/src/arch/x86/cpu/cpuid.rs`, `kernel/core/comps/console/`, `kernel/core/comps/virtio/`
(as the model for a bus + device component), and `ostd`'s IRQ and frame APIs. Branch:
`qubes-port/s2-xen-core`. You have no Xen host; QEMU/TCG is your only runtime, and it is not Xen.

## Goal

When Asterinas boots as a Xen PVH guest (detected by CPUID `0x40000000` = `XenVMMXenVMM`), it:

1. sets up the hypercall page (leaf base+2: MSR index and page count; `XENPF`-free, x86-64
   calling convention per `arch-x86/xen-x86_64.h`), with typed wrappers for `xen_version`,
   `memory_op`, `hvm_op`, `event_channel_op`, `grant_table_op`, `console_io`, `sched_op`,
   `vcpu_op`, `set_callback` is not needed (PVH uses `HVM_PARAM_CALLBACK_IRQ` vector mode:
   `(2 << 56) | vector`);
2. maps `shared_info` (`XENMEM_add_to_physmap`, space `XENMAPSPACE_shared_info`) at a reserved
   gfn, registers `vcpu_info` for every vCPU (`VCPUOP_register_vcpu_info`) so vCPUs ≥ 32 work;
3. takes the TSC frequency and wall clock from the pvclock in `shared_info` (reuse the KVM
   pvclock code; both are `pvclock_vcpu_time_info` layout) and makes the `time` component work
   with no PIT/HPET/RTC present (PVH on Xen has none; LAPIC timer + TSC deadline are available);
4. installs the event-channel upcall vector, with the 2-level pending/mask scan, and the
   `Evtchn` trait from PLAN §4 (`alloc_unbound`, `bind_interdomain`, `bind_virq` for `VIRQ_TIMER`
   is optional, `close`, `notify`, `mask/unmask`, handlers);
5. sets up the grant table v1 (`XENMEM_add_to_physmap` space `XENMAPSPACE_grant_table`, frames
   as `GNTTABOP_query_size` reports; start with 32 frames) and the `Grants` trait:
   `grant_access`/`end_access` over the entry array, and `map_foreign` through
   `GNTTABOP_map_grant_ref` with `GNTMAP_host_map` into a reserved, unpopulated gfn window above
   RAM (pick the window from the start_info memory map; document the choice), unmapped on drop;
6. brings up the PV console (`HVM_PARAM_CONSOLE_PFN/EVTCHN`, `xencons_interface` ring) as a
   console backend so that kernel logs reach `xl console`, with `HYPERVISOR_console_io`
   (`CONSOLEIO_write`) as the earliest fallback before the ring exists;
7. brings up xenstore (`HVM_PARAM_STORE_PFN/EVTCHN`, `xenstore_domain_interface` ring, the
   `xs_wire.h` protocol with request ids, watches delivered on a channel, transactions) and the
   `Xenstore` trait; logs `domid`, `name` and `vm` at boot.

Everything is gated on `is_xen()`; the non-Xen boot paths must be byte-for-byte unaffected in
behaviour (both regression gates in PLAN §5 must pass).

## How to test without Xen

- Struct layout tests (`#[ktest]`/unit tests): sizes and offsets of every vendored public
  struct you use versus constants derived from the headers (e.g. `shared_info` is 4096 bytes on
  x86-64; `grant_entry_v1` is 8; `xencons_interface` is 2056; `xenstore_domain_interface` has
  `req`/`rsp` rings of 1024 each; `evtchn_pending` has 64 words). Use `core::mem::offset_of!`.
- The xenstore client and the PV console ring must be written against an abstract ring (two
  `&mut [u8]` + atomic indices + a notify closure), so a host-side `cargo test` (or `ktest`) can
  drive them against a mock xenstored that speaks `xs_wire` (requests, replies, watch events,
  errors, transactions, a reply that arrives interleaved with a watch event).
- The event-channel pending/mask scan tested on an in-memory `shared_info` with injected bits.
- Grant entry encode/decode tested on an in-memory entry array.
- Boot regression: both `AUTO_TEST=boot` runs from PLAN §5.
- If QEMU in the image happens to support `-accel kvm,xen-version=0x40013` (it needs KVM, which
  the cloud VM lacks), do not spend time on it; note it in the report as the next test rig.

## Deliverables

- `kernel/comps/xen/` (`aster-xen`) with `lib.rs` exposing PLAN §4, vendored headers under
  `xen-headers/` with their licence notice, the hypercall and shared_info code in `ostd` where
  OSTD's safety model requires it (follow how `tdx_guest.rs` is placed), a `README.md` in the
  component describing the design and the gfn choices.
- `docs/qubes-port/REPORT-s2.md` per PLAN §5, plus a section "First Xen boot checklist" for
  Ivan: what `xl console` should print, what `xenstore-ls` in dom0 should show under the
  domain, and the `qvm-prefs`/kernel placement needed (kernel from dom0 as an ELF with the PVH
  note; `initramfs` as the first module).
