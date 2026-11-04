# Asterinas as a Qubes OS PVH guest, running ungoogled-chromium with Vulkan outsourced over xgpu

Status: plan, 2026-11-04. Owner: Ivan Kozlov. Drafted by Claude (Fable 5.1) on milky from the
Asterinas tree at `2ea21cd` (v0.19.0) and the Qubes guest agents (`qubes-gui-agent-linux`,
`qubes-core-vchan-xen`, `qubes-core-qrexec`, heads of 2026-11-04).

## 1. Goal

A Qubes OS AppVM (PVH, no PCI devices) whose kernel is Asterinas instead of Linux, running:

- the Qubes guest agents unmodified: `qubesdb-daemon`, `qrexec-agent`, `qubes-gui-agent`,
  Xorg with the `dummyqbs` video driver and the `qubes` input driver;
- ungoogled-chromium with ANGLE/Skia on Vulkan, where Vulkan is Mesa's Venus driver talking to
  the xgpu shim, which forwards to a GPU qube over qrexec (Ivan's `~/xgpu` project). The GPU qube
  side is untouched by this plan.

Security motive: the qube's kernel becomes ~240 syscalls of Rust; a compromised backend domain
(sys-net, the GPU qube) attacks safe ring code instead of C; the qube holds no GPU driver.
Stated precisely: the kernel's backend-facing surface (blkfront, netfront, grants, event
channels, xenstore) becomes small, memory-safe outside OSTD, and written to distrust the backend
(§5). The hypercall and grant plumbing itself lives in OSTD (unsafe Rust), and the GPU qube's
peer on the Alice side is Mesa Venus inside Chromium's GPU process (C/C++), which this plan does
not change.

## 2. Verified facts about the trees (do not re-derive)

Asterinas (`2ea21cd`):

- PVH boot already exists: `ostd/src/arch/x86/boot/pvh/` (entry `__pvh_boot` in `bsp_boot.S`,
  `HvmStartInfoV0/V1` parsing, ELF note in `pvh/note.S`, feature `pvh_boot`,
  `make ... BOOT_PROTOCOL=pvh`). It was written for Firecracker / Cloud Hypervisor. Xen uses
  the same `hvm_start_info` (magic `xEn3`) and fills `rsdp_paddr`, so `find_acpi_root_table_address`
  must prefer `rsdp_paddr` over the BIOS scan (it already does when nonzero).
- There is no Xen code anywhere else: no hypercalls, event channels, grant tables, xenstore.
- `ostd/src/arch/x86/kernel/kvm_clock.rs` (the pvclock reader, signature-gated on
  `KVMKVMKVM`, so it stays off under Xen), `ostd/src/arch/x86/kernel/tsc.rs` and
  `kernel/core/comps/time/src/rtc/kvmclock.rs` know the KVM pvclock. Xen's `vcpu_time_info`
  in `shared_info` has the same layout.
- Boot path under Xen PVH, as the tree stands: `late_init_on_bsp` (`ostd/src/arch/x86/mod.rs`)
  runs `irq::chip::init` three lines before `tsc::init_tsc_freq`. libxl builds a PVH domU's
  ACPI tables with `table_flags = 0` (`tools/libs/light/libxl_x86_acpi.c`), so `construct_madt`
  in `tools/libacpi/build.c` emits LAPIC entries and no I/O APIC entry (and no interrupt source
  overrides); the MADT exists while `apic` is on (the libxl default) or `vcpus > 1`, and
  `irq::chip::init` `unwrap()`s its absence. With that MADT, `irq::chip::init`
  (`ostd/src/arch/x86/irq/chip/mod.rs`) ends in `assert!(!io_apics.is_empty(), "No I/O APICs
  found")`. So the first Xen boot **panics in `irq::chip::init`**, silently: the COM1 probe
  fails (unhandled port I/O reads `0xff`), `early_print` has nowhere to write, and `abort()`
  halts the domain, which from dom0 looks like a hang. Once that assert is relaxed,
  `tsc::init_tsc_freq` is the next panic: kvmclock is signature-gated on `KVMKVMKVM` (Xen
  presents `XenVMMXenVMM` at leaf 0x40000000), Xen's guest CPUID policy models basic leaves
  0x0–0xd only (`CPUID_GUEST_NR_BASIC` in `xen/include/xen/lib/x86/cpu-policy.h`, so OSTD's
  `cpuid()` returns `None` for leaf 0x15 without executing it), and the PIT path maps ISA IRQ 0
  through `pit::enable_interrupt` → `map_isa_pin_to` → `IrqChip::map_gsi_pin_to`, which
  `unwrap()`s on the empty I/O APIC list; a PVH domU emulates only the LAPIC
  (`XEN_X86_EMU_LAPIC`: no I/O APIC, no PIT), so a non-panicking mapping would hang in the
  calibration loop instead. The TSC frequency is available from Xen's CPUID leaf 0x40000003
  subleaf 0 (`ECX` = tsc_khz) as well as from the pvclock. Those are the Gate 0 blockers; the
  rest of the boot is safe on that machine: the serial probe (`ostd/src/arch/x86/serial.rs`)
  fails on `0xff` reads (and is off anyway without the `earlycon` token); the i8042 driver
  returns `NotPresent` because libxl's FADT leaves the 8042 boot-arch flag clear (do not pass
  `i8042.exist`); the HPET driver in `timer/hpet.rs` is dead code on every platform; PCI probing
  over `0xCF8/0xCFC` reads `0xffff` and finds nothing; the CMOS RTC is skipped on the FADT's
  `NO_CMOS_RTC` flag and the time component falls back to the epoch; `iommu::init` finds no
  DMAR and warns; `vm::init` is gated on the VMX CPUID bit, which Qubes' libvirt template
  disables; the MADT's `PCAT_COMPAT` flag makes `pic::init_and_disable` poke ports of a PIC
  that is not emulated, harmlessly. The APIC timer takes the TSC-deadline path
  (`ostd/src/arch/x86/timer/apic.rs`) only if the host CPU has CPUID.1:ECX[24] and Xen passes
  it through (`TSC_DEADLINE` is `S` in `cpufeatureset.h`, but `calculate_hvm_max_policy` starts
  from the host policy and does not emulate the feature: many AMD hosts and Intel hosts with old
  microcode lack it; check for `tsc_deadline_timer` in dom0's `/proc/cpuinfo`); otherwise
  `init_periodic_mode_config` calibrates against the PIT, which is a third panic of the same
  class, so Gate 0 needs a Xen-gated timer fallback too (a periodic LAPIC timer calibrated from
  the TSC, or a `VIRQ_TIMER` tick).
- Guest-physical layout a PVH domU sees (from `tools/libs/light/libxl_x86.c` and
  `libxl_x86_acpi.c`): RAM from 0 up to the domain's `maxmem` (see the Qubes memory note below);
  the kernel at `KERNEL_LMA` 0x8000000, so `memory` must comfortably exceed 128 MiB plus the
  image plus the initramfs (the unstripped debug ELF spans [0x8000000, 0x943f858) and the NixOS
  initramfs is 14 MB; about 170 MiB is the floor and Qubes' 400 MiB default is fine; libxc
  rejects a kernel segment beyond `total_pages` with "segment kernel too large"); the ACPI
  tables at 0xfc000000 as E820_ACPI, which Asterinas maps to `Reclaimable` and counts as
  physical, so `mm::frame::meta::init` sizes its metadata up to there, about 63 MiB taken from
  the domain's RAM and invisible under QEMU, whose tables sit inside RAM; the LAPIC at
  0xfee00000; and the eight HVM special pages at 0xfeff8000–0xff000000 as E820_RESERVED
  (console page = `special_pfn(7)` = 0xfefff000, xenstore page = `special_pfn(4)` =
  0xfeffc000). Reserved pages are outside the linear mapping. `IoMem::acquire` succeeds only
  inside [about 0xfc100000, 4 GiB) and above 4 GiB (`ostd/src/arch/x86/io/mod.rs`,
  `ostd/src/io/io_mem/allocator.rs`), never in the gap between the top of RAM and 0xfc000000,
  and it defaults to `Uncacheable` (`acquire_with_cache_policy(.., Writeback)` exists for
  RAM-backed pages such as rings, `shared_info` and grant frames). The IoMem allocator is
  initialised by `crate::io::init` at the *end* of `late_init_on_bsp`, after `irq::chip::init`
  and TSC calibration; the only earlier mapping precedent is `io_mem_builder.reserve(..)` as
  the xAPIC code does. `early_print` (`ostd/src/console/mod.rs`), the panic handler and
  pre-logger log records write only to `SERIAL_PORT`, so a Xen console that only registers an
  `aster_console` device shows nothing from `late_init_on_bsp`, where all the panics above
  happen.
- The kernel ELF is linked at `KERNEL_LMA = 0x8000000` with physical `p_paddr`s and a
  `.note.Xen` section carrying `XEN_ELFNOTE_PHYS32_ENTRY`, placed in a `PT_NOTE` program header
  by the `note` PHDR of `osdk/src/base_crate/x86_64.ld.template` (libelf scans `PT_NOTE` first
  and falls back to `SHT_NOTE` sections). The file OSDK emits for `vmm-direct`
  (`target/osdk/asterinas-osdk-bin.elf`) is ELFCLASS64 with `e_machine` patched to `EM_386` for
  QEMU's multiboot loader (`osdk/src/commands/build/bin.rs`); libelf keys on the class and, for
  an HVM container, never consults the machine. Xen 4.19's libelf accepts an image whose only
  Xen note is `PHYS32_ENTRY` ("Found PVH image") and, for an HVM container, substitutes
  `phys_entry` for the ELF entry before its bounds check (`xen/common/libelf/libelf-dominfo.c`,
  `elf_xen_note_check` / `elf_xen_addr_calc_check`); libxl dispatches `type = "pvh"` to
  `libxl__build_hvm`, which sets `XC_DOM_HVM_CONTAINER`. So the existing `BOOT_PROTOCOL=pvh`
  ELF is loadable by `xl`/libxl as is (a default multiboot2 build has no note and is rejected).
  It is 170 MB unstripped; the initramfs OSDK stages next to it is the `ramdisk`, and it is
  mandatory: with no module the kernel panics in `init.rs` for lack of a root filesystem.
- `ostd/src/arch/x86/cpu/cpuid.rs` reads leaf `0x40000000` for the hypervisor signature.
- Components: `kernel/core/comps/{block,console,device,framebuffer,input,network,nvme,pci,
  virtio,time,...}`, `kernel/comps/{drm,simpledrm,i8042,uart}`. Virtio block/net are the models
  for new frontends.
- Syscalls present (`kernel/core/src/syscall/`): namespaces of type `CLONE_NEWIPC/NEWNS/NEWUTS/
  NEWCGROUP` only (`CLONE_NEWUSER/NEWPID/NEWNET/NEWTIME` return `EINVAL` from `clone`, `clone3`
  and `unshare`: `kernel/core/src/process/clone.rs`, `kernel/core/src/namespace/`), `memfd_create`, `eventfd`, `epoll`, `timerfd`, `signalfd`, `inotify`,
  `socketpair`, `sendmsg`/`recvmsg` with `SCM_RIGHTS`, `futex`, `mremap`, `madvise`, `mprotect`,
  `prctl` (incl. `PR_SET_NO_NEW_PRIVS`), `ptrace`, `statx`, `fallocate`, `pidfd_*`,
  `sched_affinity`, `semget/semop/semctl`, `getrandom`, pty (`devpts`, `ptmx`).
- Syscalls missing that matter here: `seccomp`, `shmget/shmat/shmdt/shmctl`, `rseq`,
  `pkey_*`, `io_uring_*`, `copy_file_range`, `splice`, `process_vm_readv`, `userfaultfd`
  (checked against `kernel/core/src/syscall/arch/x86.rs`). `clone3` **is** implemented
  (`sys_clone3` in `kernel/core/src/syscall/clone.rs`; it insists on `size == 88` and silently
  ignores `CLONE_CLEAR_SIGHAND`, `CLONE_INTO_CGROUP`, `set_tid` and `cgroup`). glibc and V8
  fall back on `ENOSYS` for `rseq` and `pkey_*`. The missing user, PID and network namespaces
  break Chromium's layer-1 namespace sandbox and the setuid sandbox on their own, independently
  of seccomp (row F): expect `--no-sandbox` until they exist.
- No `/dev/uinput` (the input subsystem exists). The Qubes gui-agent uses `/dev/uinput` only
  when the `gui-agent-virtual-input-device` qubes-service is enabled (it is not in the default
  service set); otherwise, and on any uinput failure, it drives the `qubes` X input driver
  (`xf86-input-mfndev`) over the AF_UNIX socket `/var/run/xf86-qubes-socket`. Row H is
  therefore optional for Qubes input.
- Graphics: DRM core, SimpleDRM, GEM dumb buffers, KMS; Xorg `modesetting` runs on Asterinas
  NixOS (`test/nixos/tests/drm`). Firefox is verified (`test/nixos/tests/web-browsers`).
  A patched systemd (`pkgs.aster_systemd`) is the init of Asterinas NixOS.
- Dev environment: Docker image `asterinas/kernel-dev:0.19.0-20261012` (Ubuntu 24.04, nightly
  `nightly-2026-07-21`, patched QEMU 10.2.1, grub, OVMF, Nix with the Asterinas binary caches
  `aster-nixos-release.cachix.org` and `aster-nixos-dev.cachix.org`). CI runs this image on
  `ubuntu-latest`. `ENABLE_KVM=0` selects TCG. `make run_kernel AUTO_TEST=boot` checks
  `Successfully booted.` in `qemu.log`. NixOS tests: `make nixos NIXOS_TEST_SUITE=<name>` then
  `make run_nixos NIXOS_TEST_SUITE=<name>`; a test is a crate under `test/nixos/tests/<name>/`
  with `extra_config.nix` (see `test/nixos/README.md`).

Qubes guest side (what the agents need from the kernel):

- `libvchan` (`qubes-core-vchan-xen`): libxenstore (`xs_open`, `xs_read/write/rm/directory`,
  `xs_watch`, transactions) → `/dev/xen/xenbus`; libxenevtchn → `/dev/xen/evtchn`;
  libxengnttab / libxengntshr → `/dev/xen/gntdev`, `/dev/xen/gntalloc`.
- `qrexec-agent` and `qubes-gui-agent`: libvchan and libqubesdb; the GUI agent additionally
  needs `/dev/tty7` (systemd `TTYPath`, Xorg on `vt07`, `VT_ACTIVATE`), a PAM session with
  D-Bus/logind, Xorg with the `dummyqbs` driver (`/dev/xen/gntalloc`, optionally
  `/dev/dri/renderD128`), `RLIMIT_MEMLOCK` of 128 MiB for group `qubes`, the
  `vm.compact_unevictable_allowed` sysctl, non-fatal writes to
  `/sys/module/xen_gntalloc/parameters/limit` and `/sys/bus/pci/drivers/bochs-drm`, udev rules
  (the `platform-pcspkr` seat tag), and either `/dev/uinput` or the `qubes` X input driver's
  socket.
- Xorg `dummyqbs` driver: `xengntshr_open/share_pages/unshare` (gntalloc) for the framebuffer;
  `GUIDomID` from the config. `xorg-qubes.conf.template` loads `fb` and `glamoregl`.
- xgpu shim (`~/xgpu/shim`, `~/xgpu/sys/src/xen.rs`): `IOCTL_GNTDEV_MAP_GRANT_REF` /
  `UNMAP_GRANT_REF` + `mmap` of the gntdev fd, `IOCTL_GNTALLOC_ALLOC_GREF`, `xenstore-read domid`,
  eventfd, memfd with seals, SEQPACKET sockets with fd passing. Mesa Venus needs no DRM node.
- Qubes' Xen is 4.19 on R4.3 (4.17 on R4.2; same guest ABI for everything used here). PVH
  guests get ACPI tables (RSDP in `hvm_start_info`), a local APIC, no I/O APIC, no PCI bus, no
  PIT/HPET/RTC/serial. Unhandled port I/O reads as `0xff` and writes are dropped (harmless).
  Console, disk, network, xenstore are PV only.
- `HYPERVISOR_console_io(CONSOLEIO_write)` from an unprivileged domU is **denied** by Xen's
  built-in permission check (`xen/include/xsm/dummy.h`, `xsm_console_io`; Qubes builds without
  XSM) unless Xen was built with `CONFIG_VERBOSE_DEBUG` or the domain is the console domain
  (`is_console`, set only by Arm dom0less code in 4.19). Qubes' hypervisor config carries
  `# CONFIG_VERBOSE_DEBUG is not set`, and its separate `xen-hypervisor-debug` package is built
  from the same config with only `CONFIG_DEBUG`, `FRAME_POINTER` and `TRACEBUFFER` appended, so
  `console_io` is refused under it too; seeing those writes needs a locally rebuilt Xen with
  `CONFIG_VERBOSE_DEBUG=y` (plus `guest_loglvl=all` on a non-debug build, because domU output is
  logged at `XENLOG_G_DEBUG`). Forget `console_io`. The PV console ring at
  `HVM_PARAM_CONSOLE_PFN` with `HVM_PARAM_CONSOLE_EVTCHN` is provisioned by libxl/libxenguest
  during domain build, before unpause, so it is the earliest console a domU reliably has. First
  output needs three hypercalls after the hypercall page (an MSR write): `HVMOP_get_param` for
  the pfn and for the port, then `EVTCHNOP_send` after writing the 2048-byte `out` ring and
  bumping `out_prod` with a write barrier. The page is at gfn 0xfefff (E820_RESERVED: map it
  explicitly, never treat it as RAM). xenconsoled attaches via the `console/{ring-ref,port,type}`
  keys libxl writes, maps the page through grant ref 0 (refs 0–7 are reserved: never hand them
  out), drains only on an event and rate-limits itself to 30 events per 200 ms, so a full ring
  must be spun on (`out_cons`) and re-signalled; on Qubes its output lands in
  `/var/log/xen/console/guest-<name>.log` even with no `xl console` attached.
- Qubes boots PVH kernels from dom0 (`/var/lib/qubes/vm-kernels/<name>/vmlinuz` + `initramfs`,
  as the PVH module list) or via pvgrub2-pvh from the guest disk. qubesd requires only
  `vmlinuz` in that directory (`initramfs` and `modules.img` are optional to qubesd; Asterinas
  needs the initramfs). The libvirt template (`templates/libvirt/xen.xml`) passes `<kernel>`,
  `<initrd>` and `<cmdline>` straight to libxl and prepends Linux defaults
  (`root=/dev/mapper/dmroot ro nomodeset console=hvc0 ... swiotlb=2048`) unless the qube has
  the `no-default-kernelopts` feature; Asterinas forwards unknown words to init's argv and
  environment, so set `qvm-features <vm> no-default-kernelopts 1` and a deliberate `kernelopts`.
- Qubes memory: libxl builds the PVH e820 from `maxmem` and populates only `memory` worth of
  pages (populate-on-demand); a guest without a balloon driver that touches more distinct pages
  than `memory` is killed by Xen with nothing on the console. qubesd treats every PVH qube as
  balloon-capable and defaults `maxmem` to 4000 MiB for a non-Linux `os` feature, so run
  `qvm-prefs <vm> maxmem 0` (the template then emits `<memory>` = `memory`) before the first
  `qvm-start`; for `xl create` simply omit `maxmem`. `qvm-start` kills a qube whose
  qrexec-agent has not connected within `qrexec_timeout` (60 s) only when the `qrexec` feature
  is set on the qube or its template; a fresh StandaloneVM without it is started with
  `QREXEC_STARTUP_NOWAIT=1` and left running.

## 3. Gap table

| # | Piece | Where | Size (Linux C yardstick) | Session |
|---|---|---|---|---|
| A | Xen guest core: CPUID detection, hypercall page, `shared_info`, `vcpu_info`, evtchn upcall vector, event-channel ops, grant table v1 + foreign mapping, Xen pvclock | OSTD (unsafe TCB) | ~2k | S2 |
| B | xenstore ring + wire-protocol client with watches; PV console ring | kernel comps | ~1k | S2 |
| C | blkfront, netfront | kernel comps | ~3k | S4 |
| D | `/dev/xen/{evtchn,gntalloc,gntdev,xenbus}` with Linux ioctl ABI; mmap of foreign frames | kernel | ~2.5k (740+600+1200+700) | S3 |
| E | Xen-PVH quirks: `IrqChip` constructible with zero I/O APICs and `map_gsi_pin_to`/`disable_gsi` returning errors instead of `unwrap()` (today's first Gate 0 panic, §2); TSC frequency from the Xen pvclock or CPUID leaf 0x40000003 ahead of the PIT fallback (the second); a Xen-gated LAPIC timer fallback for hosts without TSC-deadline (the third); the ring console wired into OSTD's `early_print`; `IoMem` windows chosen per the layout in §2; no PCI, no legacy timers | OSTD | small | S2 |
| F | seccomp-bpf (`seccomp`, `prctl(PR_SET_SECCOMP)`, `SIGSYS`, `/proc/self/status` Seccomp), and user/PID/network namespaces (`EINVAL` today, §2), without which Chromium needs `--no-sandbox` | kernel | ~2k + namespaces | S1 stretch |
| G | SysV shared memory (`shmget/shmat/shmdt/shmctl`, `/proc/sysvipc/shm`, `ipcs`) | kernel | ~0.5k | S1 |
| H | `/dev/uinput` (optional for Qubes input, §2) | kernel | ~0.3k | S1 |
| I | Chromium's remaining ENOSYS/EINVAL list, found with `sctrace` | kernel | handful | S1 |
| J | Userland: init, qubesdb, qrexec-agent, gui-agent, Xorg, Mesa Venus, shim, chromium | packaging | — | later, on Qubes |

Milestone order after the sessions merge: Gate 0 (A+E+console boots as a Qubes PVH and prints on
`xl console`) → B (xenstore) → D (qrexec, GUI agent, xgpu shim connect; everything from an
initramfs, no disk, no network) → C (root disk, network) → J.

## 4. The shared Xen interface (S2 implements, S3 and S4 code against it with a mock)

Crate `kernel/comps/xen` (`aster-xen`). S2 owns `lib.rs`; S3/S4 add `mock.rs` behind a Cargo
feature `mock` if S2's implementation is not available to them, and keep the signatures below.
Deviations are allowed when the OSTD types force them; record every deviation in the report.

```rust
pub type DomId = u16;
pub type GrantRef = u32;
pub type EvtchnPort = u32;

/// Grant table of this domain.
pub trait Grants: Send + Sync {
    /// Grants `frame` to `dom`. The frame stays owned by the caller.
    fn grant_access(&self, dom: DomId, frame: &Frame<()>, readonly: bool) -> Result<GrantRef>;
    /// Ends the grant. Fails with EBUSY while the remote still maps it (Linux keeps retrying).
    fn end_access(&self, gref: GrantRef) -> Result<()>;
    /// Maps `grefs` of `dom` into this domain's physical space, contiguously.
    fn map_foreign(&self, dom: DomId, grefs: &[GrantRef], readonly: bool) -> Result<ForeignMapping>;
}

/// A contiguous run of foreign frames; unmapped on drop. `paddr()` is a guest-physical
/// address range usable like any device memory (ioremap / VMO backing).
pub struct ForeignMapping { /* paddr: Paddr, nframes: usize, handles: Vec<GrantHandle> */ }

pub trait Evtchn: Send + Sync {
    fn alloc_unbound(&self, remote: DomId) -> Result<EvtchnPort>;
    fn bind_interdomain(&self, remote: DomId, remote_port: EvtchnPort) -> Result<EvtchnPort>;
    fn close(&self, port: EvtchnPort) -> Result<()>;
    fn notify(&self, port: EvtchnPort) -> Result<()>;
    fn mask(&self, port: EvtchnPort);
    fn unmask(&self, port: EvtchnPort);
    /// Handler runs in IRQ/softirq context when the port fires; replaces any previous handler.
    fn set_handler(&self, port: EvtchnPort, handler: Option<Box<dyn Fn() + Send + Sync>>);
}

pub trait Xenstore: Send + Sync {
    fn domid(&self) -> DomId;
    fn read(&self, path: &str) -> Result<Vec<u8>>;
    fn write(&self, path: &str, value: &[u8]) -> Result<()>;
    fn mkdir(&self, path: &str) -> Result<()>;
    fn rm(&self, path: &str) -> Result<()>;
    fn directory(&self, path: &str) -> Result<Vec<String>>;
    fn get_perms(&self, path: &str) -> Result<Vec<u8>>;   // raw xs wire perms
    fn set_perms(&self, path: &str, perms: &[u8]) -> Result<()>;
    /// Watch events arrive as (path, token) on the returned channel.
    fn watch(&self, path: &str, token: &str) -> Result<WatchHandle>;
    fn unwatch(&self, path: &str, token: &str) -> Result<()>;
    fn transaction_start(&self) -> Result<u32>;
    fn transaction_end(&self, id: u32, commit: bool) -> Result<()>;
}

pub fn grants() -> &'static dyn Grants;
pub fn evtchn() -> &'static dyn Evtchn;
pub fn xenstore() -> &'static dyn Xenstore;
pub fn is_xen() -> bool;   // CPUID 0x40000000 signature "XenVMMXenVMM"
```

Spec sources (vendor the headers into `kernel/comps/xen/xen-headers/` from
`https://github.com/xen-project/xen`, tag `RELEASE-4.19.0` or the newest 4.19.x, directory
`xen/include/public/`): `xen.h`, `arch-x86/xen.h`, `arch-x86/xen-x86_64.h`, `arch-x86/cpuid.h`,
`hvm/start_info.h`, `hvm/params.h`, `hvm/hvm_op.h`, `event_channel.h`, `grant_table.h`,
`memory.h`, `vcpu.h`, `io/xs_wire.h`, `io/console.h`, `io/ring.h`, `io/blkif.h`, `io/netif.h`,
`io/xenbus.h`. Linux is the reference implementation: `arch/x86/xen/enlighten_hvm.c`,
`enlighten_pvh.c`, `drivers/xen/{events/,grant-table.c,xenbus/,evtchn.c,gntalloc.c,gntdev.c}`,
`drivers/block/xen-blkfront.c`, `drivers/net/xen-netfront.c`, `drivers/tty/hvc/hvc_xen.c`,
`include/uapi/xen/{evtchn.h,gntalloc.h,gntdev.h}`. Fetch Linux files one at a time from
`https://raw.githubusercontent.com/torvalds/linux/v6.12/...`; do not clone Linux.

## 5. Conventions for every session

- Read `CLAUDE.md` and `AGENTS.md` at the repo root first; follow the project's style,
  licence headers (`SPDX-License-Identifier: MPL-2.0`) and `make check` / `make format`.
- Work on your own branch `qubes-port-s<N>-<name>`, branched from `qubes-port` (a slash after
  `qubes-port` is impossible while the `qubes-port` ref exists). Commit small
  and often; push after every green test run to Ivan's fork
  `https://github.com/generalsophontics/asterinas-xen.git` (if `origin` is not that URL, add it
  as remote `fork` and push there). Never force-push.
- Build and test inside the Docker image `asterinas/kernel-dev:0.19.0-20261012`
  (`docker run --rm -it -v "$PWD":/root/asterinas -w /root/asterinas --privileged --network=host
  -v /root/.ccr/ca-bundle.crt:/etc/ssl/certs/ccr-ca.crt:ro -e SSL_CERT_FILE=/etc/ssl/certs/ccr-ca.crt
  -e CURL_CA_BUNDLE=/etc/ssl/certs/ccr-ca.crt -e CARGO_HTTP_CAINFO=/etc/ssl/certs/ccr-ca.crt
  -e GIT_SSL_CAINFO=/etc/ssl/certs/ccr-ca.crt -e NIX_SSL_CERT_FILE=/etc/ssl/certs/ccr-ca.crt
  asterinas/kernel-dev:0.19.0-20261012 bash -lc '...'`; the CA mount lets rustup, cargo, git and
  nix inside the container pass the cloud environment's TLS-intercepting proxy, see
  `README.md`). If `docker ps` fails, the daemon is not
  running: `setsid nohup dockerd >/tmp/dockerd.log 2>&1 &` and retry after a few seconds.
  There is no `/dev/kvm` in the cloud VM:
  always pass `ENABLE_KVM=0`. TCG is 10–50× slower than KVM: use `SMP=2`, be patient, and
  raise the Bash tool timeout to 10 minutes for QEMU runs; run long builds in the background
  with the log in a file.
- The regression gate before every push: `make run_kernel ENABLE_KVM=0 AUTO_TEST=boot` for the
  default boot protocol **and** `make run_kernel ENABLE_KVM=0 BOOT_PROTOCOL=pvh AUTO_TEST=boot`
  must both still pass (QEMU is not Xen, so all Xen paths must be gated on `is_xen()` and the
  non-Xen boot must be unchanged).
- No Xen host is available to you. Say so plainly in the report; never claim a Xen-side test
  you did not run. Ivan and Claude test on Qubes afterwards.
- Every page shared with another domain (rings, granted frames, `shared_info`, the xenstore and
  console pages) is memory the other side can rewrite at any instant. Read each field once into
  a local (`read_once`/volatile), validate it against the ring size and the request you issued,
  and never dereference backend-supplied indices, lengths or grant handles without a bounds
  check; a grant whose unmap fails stays unmapped and its frame is leaked, not reused (the Linux
  XSA-155/XSA-396 class of bugs, and the reason the frontends are worth rewriting at all).
  Safe Rust checks none of this for you.
- Write `docs/qubes-port/REPORT-s<N>.md` as you go (not at the end): what works, the exact
  commands that verify it, what is missing, every deviation from §4, open questions, and the
  wall-clock you spent. Commit and push the report with the code.
- Stop when the scope is done, or when you have been blocked on the environment for more than
  one hour (then write the report with the blocker and push), or after 10 hours of wall-clock.
- Do not touch files outside your scope except for the minimum glue (e.g. registering a
  component). Do not rebase onto upstream `main`.
