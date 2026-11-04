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

## 2. Verified facts about the trees (do not re-derive)

Asterinas (`2ea21cd`):

- PVH boot already exists: `ostd/src/arch/x86/boot/pvh/` (entry `__pvh_boot` in `bsp_boot.S`,
  `HvmStartInfoV0/V1` parsing, ELF note in `pvh/note.S`, feature `pvh_boot`,
  `make ... BOOT_PROTOCOL=pvh`). It was written for Firecracker / Cloud Hypervisor. Xen uses
  the same `hvm_start_info` (magic `xEn3`) and fills `rsdp_paddr`, so `find_acpi_root_table_address`
  must prefer `rsdp_paddr` over the BIOS scan (it already does when nonzero).
- There is no Xen code anywhere else: no hypercalls, event channels, grant tables, xenstore.
- `ostd/src/arch/x86/kernel/tsc.rs` and `kernel/core/comps/time/src/rtc/kvmclock.rs` know the
  KVM pvclock. Xen's `vcpu_time_info` in `shared_info` has the same layout.
- `ostd/src/arch/x86/cpu/cpuid.rs` reads leaf `0x40000000` for the hypervisor signature.
- Components: `kernel/core/comps/{block,console,device,framebuffer,input,network,nvme,pci,
  virtio,time,...}`, `kernel/comps/{drm,simpledrm,i8042,uart}`. Virtio block/net are the models
  for new frontends.
- Syscalls present (`kernel/core/src/syscall/`): all 8 namespace types (`CLONE_NEWUSER/PID/NET/
  IPC/NS/UTS/CGROUP/TIME`), `memfd_create`, `eventfd`, `epoll`, `timerfd`, `signalfd`, `inotify`,
  `socketpair`, `sendmsg`/`recvmsg` with `SCM_RIGHTS`, `futex`, `mremap`, `madvise`, `mprotect`,
  `prctl` (incl. `PR_SET_NO_NEW_PRIVS`), `ptrace`, `statx`, `fallocate`, `pidfd_*`,
  `sched_affinity`, `semget/semop/semctl`, `getrandom`, pty (`devpts`, `ptmx`).
- Syscalls missing that matter here: `seccomp`, `shmget/shmat/shmdt/shmctl`, `clone3`, `rseq`,
  `pkey_*`, `io_uring_*`, `copy_file_range`, `splice`, `process_vm_readv`, `userfaultfd`.
  glibc and V8 fall back on `ENOSYS` for `clone3`, `rseq`, `pkey_*`.
- No `/dev/uinput` (the input subsystem exists). The Qubes gui-agent prefers `/dev/uinput` and
  falls back to the `qubes` X input driver (`xf86-input-mfndev`).
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
- `qrexec-agent` and `qubes-gui-agent`: libvchan only, plus `/dev/uinput` or the X FIFO driver,
  `/dev/tty`.
- Xorg `dummyqbs` driver: `xengntshr_open/share_pages/unshare` (gntalloc) for the framebuffer;
  `GUIDomID` from the config. `xorg-qubes.conf.template` loads `fb` and `glamoregl`.
- xgpu shim (`~/xgpu/shim`, `~/xgpu/sys/src/xen.rs`): `IOCTL_GNTDEV_MAP_GRANT_REF` /
  `UNMAP_GRANT_REF` + `mmap` of the gntdev fd, `IOCTL_GNTALLOC_ALLOC_GREF`, `xenstore-read domid`,
  eventfd, memfd with seals, SEQPACKET sockets with fd passing. Mesa Venus needs no DRM node.
- Qubes' Xen is 4.19; PVH guests get ACPI tables (RSDP in `hvm_start_info`), a local APIC,
  no PCI bus, no PIT/HPET/RTC/serial. Console, disk, network, xenstore are PV only.
  Qubes boots PVH kernels from dom0 (`/var/lib/qubes/vm-kernels/<name>/vmlinuz` + `initramfs`,
  as the PVH module list) or via pvgrub2-pvh from the guest disk.

## 3. Gap table

| # | Piece | Where | Size (Linux C yardstick) | Session |
|---|---|---|---|---|
| A | Xen guest core: CPUID detection, hypercall page, `shared_info`, `vcpu_info`, evtchn upcall vector, event-channel ops, grant table v1 + foreign mapping, Xen pvclock | OSTD (unsafe TCB) | ~2k | S2 |
| B | xenstore ring + wire-protocol client with watches; PV console ring | kernel comps | ~1k | S2 |
| C | blkfront, netfront | kernel comps | ~3k | S4 |
| D | `/dev/xen/{evtchn,gntalloc,gntdev,xenbus}` with Linux ioctl ABI; mmap of foreign frames | kernel | ~2.5k (740+600+1200+700) | S3 |
| E | Xen-PVH quirks: no PCI, no legacy timers, early console blind until B | cfg | small | S2 |
| F | seccomp-bpf (`seccomp`, `prctl(PR_SET_SECCOMP)`, `SIGSYS`, `/proc/self/status` Seccomp) | kernel | ~2k | S1 stretch |
| G | SysV shared memory (`shmget/shmat/shmdt/shmctl`, `/proc/sysvipc/shm`, `ipcs`) | kernel | ~0.5k | S1 |
| H | `/dev/uinput` | kernel | ~0.3k | S1 |
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
- Work on your own branch `qubes-port/s<N>-<name>`, branched from `qubes-port`. Commit small
  and often; push after every green test run to Ivan's fork
  `https://github.com/generalsophontics/asterinas-xen.git` (if `origin` is not that URL, add it
  as remote `fork` and push there). Never force-push.
- Build and test inside the Docker image `asterinas/kernel-dev:0.19.0-20261012`
  (`docker run --rm -it -v "$PWD":/root/asterinas -w /root/asterinas --privileged --network=host
  asterinas/kernel-dev:0.19.0-20261012 bash -lc '...'`). There is no `/dev/kvm` in the cloud VM:
  always pass `ENABLE_KVM=0`. TCG is 10–50× slower than KVM: use `SMP=2`, be patient, and
  raise the Bash tool timeout to 10 minutes for QEMU runs; run long builds in the background
  with the log in a file.
- The regression gate before every push: `make run_kernel ENABLE_KVM=0 AUTO_TEST=boot` for the
  default boot protocol **and** `make run_kernel ENABLE_KVM=0 BOOT_PROTOCOL=pvh AUTO_TEST=boot`
  must both still pass (QEMU is not Xen, so all Xen paths must be gated on `is_xen()` and the
  non-Xen boot must be unchanged).
- No Xen host is available to you. Say so plainly in the report; never claim a Xen-side test
  you did not run. Ivan and Claude test on Qubes afterwards.
- Write `docs/qubes-port/REPORT-s<N>.md` as you go (not at the end): what works, the exact
  commands that verify it, what is missing, every deviation from §4, open questions, and the
  wall-clock you spent. Commit and push the report with the code.
- Stop when the scope is done, or when you have been blocked on the environment for more than
  one hour (then write the report with the blocker and push), or after 10 hours of wall-clock.
- Do not touch files outside your scope except for the minimum glue (e.g. registering a
  component). Do not rebase onto upstream `main`.
