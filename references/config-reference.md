# Configuration Reference

Loaded on demand from `SKILL.md`. Assemble final configs, explain trade-offs, or
walk the user through architecture here.

## Complete VM XML (key sections)

```xml
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
  <name>win11</name>
  <memory unit='GiB'>32</memory>
  <vcpu placement='static'>18</vcpu>

  <!-- CPU mode: host-model WITHOUT migratable (critical for VFIO + shmem) -->
  <cpu mode='host-model' check='none'>
    <topology sockets='1' dies='1' cores='18' threads='1'/>
  </cpu>

  <!-- GPU passthrough: BOTH functions (GPU + audio) -->
  <hostdev mode='subsystem' type='pci' managed='yes'>
    <source>
      <address domain='0x0000' bus='0x08' slot='0x00' function='0x0'/>
    </source>
  </hostdev>
  <hostdev mode='subsystem' type='pci' managed='yes'>
    <source>
      <address domain='0x0000' bus='0x08' slot='0x00' function='0x1'/>
    </source>
  </hostdev>

  <!-- SPICE for input only (video comes from Looking Glass) -->
  <graphics type="spice" autoport="yes">
    <listen type="address" address="127.0.0.1"/>
  </graphics>

  <!-- No memory ballooning -->
  <memballoon model="none"/>

  <!-- NO USB tablet device (incompatible with Looking Glass) -->

  <!-- KVMFR shared memory (JSON syntax for QEMU 6.2+; size in bytes) -->
  <qemu:commandline>
    <qemu:arg value="-device"/>
    <qemu:arg value="{'driver':'ivshmem-plain','id':'shmem0','memdev':'looking-glass'}"/>
    <qemu:arg value="-object"/>
    <qemu:arg value="{'qom-type':'memory-backend-file','id':'looking-glass','mem-path':'/dev/kvmfr0','size':134217728,'share':true}"/>
  </qemu:commandline>
</domain>
```

For the **Alder Lake iGPU (GVT-d)**, replace the two `<hostdev>` PCI addresses
with the iGPU's GPU + audio functions, and (if Code 43 / VBIOS issues bite)
consider adding a `<rom file='/path/to/extracted.vbios'/>` inside the GPU
`<hostdev>`.

## Host configuration files

| File | Contents |
|---|---|
| `/etc/sdboot-manage.conf` | `LINUX_OPTIONS="intel_iommu=on rd.driver.pre=vfio-pci video=efifb:off"` (CachyOS; run `sudo sdboot-manage gen`) |
| `/etc/modprobe.d/vfio.conf` | `options vfio-pci ids=<gpu_id>,<audio_id>` + `softdep snd_hda_intel pre: vfio-pci` |
| `/etc/modprobe.d/kvmfr.conf` | `options kvmfr static_size_mb=128` |
| `/etc/modules-load.d/kvmfr.conf` | `kvmfr` |
| `/etc/udev/rules.d/99-kvmfr.rules` | `SUBSYSTEM=="kvmfr", OWNER="your_username", GROUP="kvm", MODE="0660"` |
| `/etc/libvirt/qemu.conf` | Add `/dev/kvmfr0` to `cgroup_device_acl`; restart libvirtd |
| `/etc/mkinitcpio.conf` | `MODULES=(vfio_pci vfio vfio_iommu_type1)`; rebuild with `sudo mkinitcpio -P` |
| `~/.looking-glass-client.ini` | `[app] renderer=opengl` (NVIDIA+Wayland **flicker** fix); `[input] rawMouse=yes` + `mouseSmoothing=no` (input latency); `[win] showFPS=yes` (diagnose). The **crash** fix (`__NV_DISABLE_EXPLICIT_SYNC=1`) is an *env var*, not this file — apply via a `~/.local/share/applications/` `.desktop` override. See `field-lessons.md` §5 |

GVT-d-only additions: blacklist `i915` (e.g. `/etc/modprobe.d/i915.conf` →
`blacklist i915`) so the host framebuffer driver doesn't grab the iGPU first.

## Windows guest components

| Component | Location / value |
|---|---|
| Looking Glass host | `C:\Program Files\Looking Glass (host)\looking-glass-host.exe` |
| Host log | `%ProgramData%\Looking Glass (host)\looking-glass-host.txt` |
| Capture method | D12 (default; fastest on Windows 10+) |
| Virtual display driver | **VDD** (`github.com/VirtualDrivers/Virtual-Display-Driver`; `winget install --id=VirtualDrivers.Virtual-Display-Driver -e`) — required for a headless SR-IOV VF (gives D12 something to capture); set secondary until LG is capturing, then primary |
| SPICE guest tools | Installed for clipboard sync / input integration |
| GPU driver | Standalone installer from the vendor (auto-detect tools often fail in VMs) |

## Architecture: how Looking Glass works

Looking Glass shares GPU framebuffer data between guest and host through shared
memory, **bypassing the normal display path entirely**.

- The **host app** runs inside the Windows VM and uses the Desktop Duplication API
  (**D12** on Windows 10+) to capture the frames the GPU is rendering — the same
  mechanism screen recording uses. Instead of encoding (like RDP/VNC), it writes
  **raw, uncompressed** frame data into shared memory.
- That shared memory is the **IVSHMEM device** — a virtual PCI device both sides
  access at the same physical addresses. The **KVMFR** kernel module manages it on
  Linux; the IVSHMEM PCI driver handles it on Windows.
- The **client** on the Linux desktop maps that same region and reads frames
  directly. No encode/decode/network → latency is essentially one DMA copy plus
  the GPU→shared-memory copy.
- **SPICE** runs alongside purely for input (keyboard/mouse/clipboard/audio). The
  video frames never touch SPICE.

## Why not Moonlight/Sunshine (for this workload)

Tested for CAD at 3440×1440 and rejected:

- **165 FPS** (native refresh) → massive compression artifacts; GTX 1080 NVENC
  can't keep up.
- **60 FPS / 150 Mbps** → image OK, but mouse input felt sluggish — unacceptable
  for precision CAD (Fusion 360).
- HEVC encode unavailable (client-side decode issues on CachyOS).
- The encode/decode pipeline adds perceptible latency vs. raw framebuffer sharing.

Moonlight/Sunshine is excellent for gaming/media; it's the wrong tool for
pixel-perfect, zero-lag productivity/CAD work. (The **GL.iNet Comet Pro** IP-KVM
was also tried: useful as a fallback/BIOS-access device, but adds its own
compression and latency — not a primary productivity display.)

## iGPU GVT-d: the real story (corrected)

Earlier framing called this "abandoned / impossible." That was wrong. What's true:

- **It works.** UHD 770 full passthrough (GVT-d) is documented working in
  community guides (e.g. `github.com/LongQT-sea/intel-igpu-passthru`, Derek
  Seaman's Proxmox + Alder Lake VT-d write-up). The device enumerates in the
  guest as "Intel UHD Graphics 770."
- **The hard part is Code 43, a Windows *driver* error** (driver detects the
  hypervisor and won't start) — NOT "the guest can't POST it." Don't confuse
  Code 43 with a firmware/POST failure: the device is present, the driver just
  refuses. (Some tools/agents mislabel this as "Code 10" or blame the
  in-firmware GOP; the GOP detail is real but only affects pre-boot display,
  which Looking Glass/SPICE bypass anyway.)
- **What the original doc actually said:** iGPU GVT-d was "not worth pursuing"
  because the author *had a GTX 1080* that was "vastly more capable" for CAD — an
  effort-vs-reward call, **not** a claim of impossibility.

So when a user asks "can I passthrough my Alder Lake iGPU?": **yes**, via GVT-d
(or SR-IOV). Recommend discrete only when they need real GPU horsepower for the
workload.

## Sourcing the iGPU VBIOS / GOP

The iGPU's GOP/VBT lives in the motherboard UEFI (no discrete OpROM chip), so
OVMF has nothing to run by default. The device still *enumerates* without one,
but a romfile can clear stubborn init failures. Three ways to get one:

1. **Use a pre-extracted community ROM** (easiest). `github.com/LongQT-sea/intel-igpu-passthru`
   ships GOP ROMs (e.g. `ARL_MTL_GOPv22_igd.rom`) that multiple users confirm
   clear Code 43 on Intel iGPU passthrough. Match the ROM to your gen/SKU.
2. **Extract `IntelGopDriver.efi` from your BIOS update.** Open the vendor BIOS
   in [UEFITool](https://github.com/LongSoft/UEFITool) and search for the GOP
   driver (its Unicode GUID/name string). For Dell packs, `BIOSUtilities`
   (`DellPfsExtract`) unpacks `BIOS_IMG.rcv` into the individual firmware blobs,
   one of which is the GOP driver; then convert/concatenate to a `.rom` QEMU can
   load. Place it at e.g. `/usr/share/kvm/igpu.rom`.
3. **Dump the running image** with `intel-gpu-tools` / by reading the UEFI
   firmware region (fiddly; least reliable).

Attach it on the GPU `<hostdev>`:

```xml
<hostdev mode='subsystem' type='pci' managed='yes'>
  <rom file='/usr/share/kvm/igpu.rom'/>
  <source>...</source>
</hostdev>
```

Re-test Code 43. If a romfile makes things *worse*, remove it — a wrong ROM only
hurts, and Alder Lake often needs no romfile at all once the Code 43 fixes below
are applied.

## Intel iGPU Code 43: root causes & fixes

**Intel iGPU Code 43 is not the NVIDIA Code 43** (which is fixed by hiding the
hypervisor). For Intel, the driver fails to initialize for a handful of distinct
reasons. Triage in this order. (Evidence: `strongtz/i915-sriov-dkms` issues #313
and #387, William Lam's NUC write-ups, Derek Seaman's guide.)

### 1. Windows nested virtualization (the #1 cause)

The Intel graphics driver / i915 SR-IOV **cannot run under Windows' own
virtualization stack**. Windows 11 enables these by default on many images, and
each one triggers Code 43 — reproduced exactly: enabling **Virtual Machine
Platform** → instant Code 43; disabling → fixed. In the **guest**, turn OFF:

- **Memory Integrity** (Core Isolation / HVCI) — Windows Security → Device
  security → Core isolation details → Memory integrity = **Off**.
- **Virtual Machine Platform** — Settings → System → Optional Features → More
  Windows Features → uncheck **Virtual Machine Platform**.
- **Hyper-V** (uncheck, if listed).
- **WSL2 / Windows Sandbox** — both turn on Virtual Machine Platform; avoid them
  in the guest.

Canonical disable checklist: `github.com/strongtz/i915-sriov-dkms/issues/335`.

### 2. libvirt `<hyperv>` over-configuration (libvirt-specific)

A `<hyperv mode='custom'>` block "with everything on" (and/or a spoofed
`vendor_id`) **causes** Code 43 on Intel iGPU. Use passthrough instead:

```xml
<features>
  <acpi/><apic/>
  <hyperv mode='passthrough'/>
</features>
```

Confirmed by multiple users: switching to `mode='passthrough'` cleared Code 43
**without** any `vendor_id`, even after *removing* the GOP romfile (60 FPS WebGL,
Parsec working in H.265). **Do NOT apply the NVIDIA `<kvm><hidden state='on'/>`
+ hyperv `vendor_id` trick to an Intel iGPU — it's counterproductive here.**

### 3. Driver version + Windows Update downgrade

Code 43 is driver-version-sensitive — "if I updated, Code 43 was guaranteed" is
a common report. Use Intel's **"Recommended"** build for your gen (e.g.
`32.0.101.7085` for Alder Lake) from the standalone .exe. Then stop Windows
Update silently swapping it for a broken 2022 build:

- Group Policy → Computer Configuration → Administrative Templates → Windows
  Components → Windows Update → **Do not include drivers with Windows Update** =
  Enabled (Pro/Enterprise). (Home: `wushowhide` / InControl.)
- After install, if still Code 43: **Disable → Enable** the device in Device
  Manager (or reboot) — clears it for some (William Lam).

### 4. Make the iGPU/VF the primary display

If Windows POSTed with a different (emulated) adapter, the iGPU driver can fail.
Remove the `<video>` element, or use `<video><model type='none'/></video>` (**not**
`<video model='none'/>` — that's invalid schema), so the iGPU/VF is the
guest's only/primary adapter. (Proxmox equivalents for cross-reference:
`vga: none`, `Primary GPU` / `x-vga=1`.)

### 5. Missing/wrong GOP VBIOS (stubborn cases; common on Arrow Lake 14th/15th gen)

Supply a GOP romfile (see "Sourcing the iGPU VBIOS / GOP" above). For **Alder
Lake (12th gen)** this is usually *not* needed once (1)–(4) are done — but it's
the fallback for a device that still won't start.

### 6. Full GVT-d still broken → use SR-IOV VFs

If whole-device passthrough refuses to clear Code 43, switch to the **SR-IOV
virtual-function** path (`references/sr-iov-vgpu.md`), which the community
consistently gets working on Alder Lake where full GVT-d fails.

## SR-IOV vGPU (mediated, GVT-g-like)

Skip full passthrough and hand the VM a **virtual function** while the host keeps
the iGPU — the closest thing to the old GVT-g, and the **most reliable path on
Alder Lake**. Tradeoff: no physical monitor output (RDP / SPICE / Looking Glass
only). Full Arch/CachyOS + libvirt procedure in `references/sr-iov-vgpu.md`
(ported from Derek Seaman's Proxmox guide). Prefer it when you want the host to
keep using the iGPU (Quick Sync, a host display) too, or when full GVT-d won't
clear Code 43.
