---
name: intel-gpu-passthrough-skill
description: Configure KVM/QEMU GPU passthrough (VFIO) on Intel Alder Lake (12th-gen) systems and bring the guest display to the host with Looking Glass via IVSHMEM shared memory. Use when the user wants to pass through an Intel UHD iGPU (GVT-d) or a discrete GPU to a Windows/Linux VM, set up VFIO binding and IOMMU on Arch/CachyOS, build the KVMFR shared-memory module, configure libvirt/QEMU XML for ivshmem-plain, build the Looking Glass client, or troubleshoot VFIO/IVSHMEM/Looking Glass problems (Code 43, blank displays, /dev/kvmfr0 permission or device-node issues, "PCI slot not available", flickering black boxes on NVIDIA+Wayland). Triggers include "gpu passthrough", "VFIO", "looking glass", "ivshmem", "kvmfr", "pass through my intel igpu", "GVT-d", "SR-IOV", "i915-sriov", "vGPU", "alder lake virtualization", or running Windows/CAD/GPU workloads in a VM with near-zero-latency display.
---

# Alder Lake iGPU / GPU Passthrough + Looking Glass

Complete pipeline for passing a GPU through to a KVM VM and displaying it with
**Looking Glass** on an Arch/CachyOS host. Written from a real install on an
**Intel i9-12900K (Alder Lake, UHD 770 iGPU)** with NVIDIA discrete cards; every
issue below was actually hit and resolved.

## ⚠️ Read this first: the Alder Lake iGPU reality (corrected)

Passing the **Alder Lake (12th-gen) Intel iGPU (UHD 770)** to a VM **works**.
Earlier drafts of this skill hedged it as "not worth it / abandon"; that was too
pessimistic. The honest picture:

- **No GVT-g** (Intel's old *shared*-GPU mediation was dropped before Alder
  Lake). The device is **not** shareable host+guest via the official path.
- **Two real options instead:**
  1. **GVT-d — full passthrough.** Give the entire iGPU to the VM. Host loses it
     (`i915` blacklisted), so the host display must come from another GPU. On a
     typical dual-GPU box (iGPU idle, NVIDIA driving the host) that's a
     non-issue. *Detailed below in Part 0 → "GVT-d specifics."*
  2. **SR-IOV virtual functions** via the community `i915-sriov-dkms` module — a
     modern GVT-g-like split: the VM gets a virtual function while the host keeps
     the iGPU. See `references/config-reference.md` → "SR-IOV alternative."
- **The device WILL enumerate in the guest** — Windows lists it as "Intel(R) UHD
  Graphics 770." So if it shows up, the passthrough itself *succeeded*.
- **Code 43 is the real boss, and it's now well understood.** For an Intel iGPU
  the classic Code 43 is a *driver-level* failure, and the **#1 cause is Windows
  nested virtualization** (Memory Integrity / Virtual Machine Platform / WSL2) —
  fixable in the guest. It is **not** the NVIDIA-style hypervisor-sniffing Code 43
  (don't reach for the NVIDIA `kvm-hidden`/`vendor_id` trick). Full triage:
  `references/config-reference.md` → "Intel iGPU Code 43"; SR-IOV VF path:
  `references/sr-iov-vgpu.md`.
  Dismiss claims that "the GOP/VBIOS is baked into the motherboard so the guest
  can't POST it": that GOP detail is real but only governs *pre-boot display in
  the guest*, which you don't need when displaying via Looking Glass/SPICE.
- **Honest caveat:** for heavy CAD (e.g. Fusion 360), the UHD 770 is genuinely
  weak vs. a discrete card — a *performance* reason to prefer discrete, **not** a
  "can't do it" reason.

So: if a spare discrete card is available AND the workload needs GPU muscle,
discrete is the easier/better path (Part A). If you specifically want the iGPU in
the VM (the discrete card must stay 100% on the host, or the guest only needs
desktop/video/light-compute), GVT-d is a real, working choice.

## How it fits together (30-second version)

Looking Glass shares raw, uncompressed GPU framebuffer data between guest and
host through the **IVSHMEM** device — a region of RAM mapped at the same physical
addresses on both sides. No encoding, no network, so latency is essentially one
DMA copy. SPICE runs alongside purely for input (keyboard/mouse/clipboard/audio);
video frames never touch SPICE.

- **Guest** (Windows): Looking Glass *host* service captures the desktop via the
  **D12** Desktop Duplication API and writes frames into IVSHMEM.
- **Shared memory**: the **KVMFR** kernel module exposes `/dev/kvmfr0`.
- **Host** (Linux): Looking Glass *client* maps `/dev/kvmfr0` and renders frames.

## Prerequisites

- **VT-d enabled in BIOS/UEFI** (look for "VT-d" / "Intel Virtualization
  Technology for Directed I/O" / "IOMMU"). `intel_iommu=on` does nothing without
  it. Also confirm VT-x.
- QEMU 6.2+ and libvirt 7.9+ (this guide uses **JSON-style** QEMU args).
- A second GPU to drive the host display (mandatory for GVT-d, since the iGPU is
  given away; discretionary for discrete passthrough).
- Root/sudo, and `dkms` for the KVMFR module.

Reference setup this guide was validated against: CachyOS + Hyprland, host GPU
NVIDIA RTX 3080 Ti, guest GPU NVIDIA GTX 1080, 32 GB of 64 GB RAM to VM, 18 vCPUs,
Looking Glass **B7**, 3440×1440 target.

---

## Part A: Pick the GPU to pass through

Identify candidates and their PCI IDs:

```bash
lspci -nn | grep -E "VGA|Audio" | grep -iE "nvidia|intel"
```

Example dual-GPU output:

```
01:00.0 VGA ... NVIDIA ... RTX 3080 Ti [10de:2208] (rev a1)
01:00.1 Audio ... NVIDIA ... [10de:1aef]
08:00.0 VGA ... NVIDIA ... GTX 1080 [10de:1b80] (rev a1)
08:00.1 Audio ... NVIDIA ... [10de:10f0]
```

Note **both** the GPU function (`xxxx.y.0`) and its **audio function**
(`xxxx.y.1`) — both must be passed through together, and both PCI vendor:device
IDs go into the VFIO bind list.

**Recommendation:** pass through a discrete card. If the user insists on the
Alder Lake iGPU, see **GVT-d specifics** in Part 0 — the device will look like an
Intel `VGA ... Intel ... UHD [8086:xxxx]` with an audio function (`8086:xxxx`).

---

## Part 0: VFIO passthrough setup

> If the user already has a working passthrough, skip to Part 1.

### 0.1 Persistent kernel parameters (CachyOS / systemd-boot)

CachyOS regenerates systemd-boot entries on kernel updates and **overwrites**
manual edits to `/boot/loader/entries/*.conf`. Set parameters persistently:

```bash
sudo vim /etc/sdboot-manage.conf
# set: LINUX_OPTIONS="intel_iommu=on rd.driver.pre=vfio-pci video=efifb:off"
sudo sdboot-manage gen
```

Then reboot and verify with `cat /proc/cmdline`.

**Do not add `iommu=pt`.** Despite many guides recommending it, passthrough mode
prevents IOMMU groups from being populated (you'll see `Default domain type:
Passthrough` in dmesg and an empty `/sys/kernel/iommu_groups/`). Remove it
entirely, or use `iommu=1`. The parameter that actually matters for early VFIO
binding is `rd.driver.pre=vfio-pci`.

If parameters vanish after an update, the user edited boot entries directly —
fix it via `/etc/sdboot-manage.conf` + `sudo sdboot-manage gen`.

### 0.2 Bind the guest GPU to vfio-pci

Create `/etc/modprobe.d/vfio.conf` (substitute the real IDs for both functions):

```
options vfio-pci ids=10de:1b80,10de:10f0
softdep snd_hda_intel pre: vfio-pci
```

The `softdep` prevents the audio driver from racing to claim the GPU's audio
function before VFIO.

Add VFIO modules to the initramfs. Edit `/etc/mkinitcpio.conf`:

```
MODULES=(vfio_pci vfio vfio_iommu_type1)
```

Rebuild and reboot:

```bash
sudo mkinitcpio -P
```

Verify both functions show `Kernel driver in use: vfio-pci`:

```bash
lspci -nnk -s 08:00
```

If `lspci -nnk` shows **no driver**, the modules aren't in the initramfs.
Confirm CachyOS's image name and contents:

```bash
sudo lsinitcpio /boot/initramfs-linux-cachyos.img | grep vfio
# (image may be linux-cachyos or linux-cachyos-lts)
```

### GVT-d specifics (Alder Lake iGPU passthrough)

Full passthrough of the UHD 770. The device *will* show up in the guest as
"Intel UHD Graphics 770"; the fight is clearing **Code 43** so the driver
starts. Work through these in order:

1. **Identify the iGPU functions** — capture the GPU and its audio PCI IDs:
   `lspci -nn | grep -i vga` → `... Intel ... UHD Graphics 770 [8086:xxxx]`.
2. **Bind to vfio-pci** — add both IDs to `options vfio-pci ids=...` in
   `/etc/modprobe.d/vfio.conf`, same as a discrete card.
3. **Blacklist `i915` on the host** so the kernel doesn't grab the iGPU first
   (`/etc/modprobe.d/i915.conf` → `blacklist i915`; make sure it isn't force-
   loaded in the initramfs). The host display must come from another GPU (your
   case: the NVIDIA card). Rebuild initramfs + reboot, then confirm
   `Kernel driver in use: vfio-pci` with `lspci -nnk -s <bdf>`.
4. **Attach both functions to the VM** (Part 0.3), then clear Code 43. The Intel
   iGPU ladder is **different from NVIDIA's** — don't just "hide the
   hypervisor." In order (deep dive: `references/config-reference.md` →
   "Intel iGPU Code 43"):
   a. **Turn OFF Windows nested virtualization in the guest** — the #1 cause.
      Disable **Memory Integrity** (Windows Security → Device security → Core
      isolation), **Virtual Machine Platform** (Optional Features → More Windows
      Features), **Hyper-V**, and don't install **WSL2 / Windows Sandbox** (each
      flips Code 43 on).
   b. **Use `<hyperv mode='passthrough'/>`** in the domain XML — *not* a custom
      `vendor_id` block. A `<hyperv mode='custom'>` "everything-on" block (and
      the NVIDIA `<kvm><hidden state='on'/>` + `vendor_id` trick) actually
      *causes* Intel iGPU Code 43 (confirmed by multiple users on the
      `i915-sriov-dkms` tracker).
   c. **Driver version + stop the downgrade.** Install Intel's "Recommended"
      build for your gen from the standalone .exe (the latest isn't always best —
      some report Code 43 only *after* updating). Then Group Policy → "Do not
      include drivers with Windows Update" = **Enabled**, so Windows can't swap
      in a broken 2022 driver.
   d. **Make the iGPU the primary display** — remove the `<video>` element, or
      use `<video><model type='none'/></video>`. ⚠️ `<video model='none'/>`
      (model as an *attribute*) is **invalid schema** and is silently rejected;
      `model` must be a child `<model type='none'/>`. See
      `references/field-lessons.md` §4.
   e. **Disable → Enable** the device in Device Manager, or reboot, after the
      driver install (clears it for some).
   f. **GOP romfile** as a last resort: `<rom file='/usr/share/kvm/igpu.rom'/>`
      on the GPU `<hostdev>`, sourced from `github.com/LongQT-sea/intel-igpu-passthru`
      or extracted from your BIOS (`references/config-reference.md` → "Sourcing
      the iGPU VBIOS/GOP"). Usually *not* needed for Alder Lake once (a)–(e) are
      done.

If Code 43 persists across all of the above, the **SR-IOV virtual-function**
path is often more reliable on Alder Lake than plain GVT-d — see
`references/config-reference.md` → "SR-IOV alternative."

### 0.3 Create the VM and attach the GPU

Create a Windows 11 VM in virt-manager with **Q35** machine type and **UEFI/OVMF**
firmware. Add **both** PCI functions as PCI Host Devices (`0000:08:00.0` and
`0000:08:00.1`).

### 0.4 CPU mode (critical for VFIO + IVSHMEM)

Wrong CPU mode makes QEMU crash with `vfio_container_region_add` when a shared
memory device is present. Use **`host-model`**, **no `migratable` attribute at
all** (not `'off'` — just absent):

```xml
<cpu mode='host-model' check='none'>
  <topology sockets='1' dies='1' cores='18' threads='1'/>
</cpu>
```

- `host-passthrough` with `migratable='on'` → memory-mapping conflicts between
  GPU passthrough and IVSHMEM → QEMU crash.
- The `<topology>` block tells Windows the real vCPU layout (otherwise e.g.
  "2 sockets" instead of "18 cores"). vCPUs are shared, not reserved.

### 0.5 Bridge networking (optional, needed for SPICE/LAN access)

```bash
sudo nmcli connection add type bridge ifname br0 con-name br0
sudo nmcli connection add type ethernet ifname enp6s0 master br0   # your iface (ip link)
sudo nmcli connection up br0
```

VM NIC:

```xml
<interface type='bridge'>
  <mac address='52:54:00:xx:xx:xx'/>
  <source bridge='br0'/>
  <model type='e1000e'/>
</interface>
```

### 0.6 Guest drivers

Install the **guest** GPU driver manually (NVIDIA from nvidia.com; Intel drivers
for the iGPU). Bundled "Experience"/auto-detect tools frequently fail in VMs due
to hardware-detection checks — get the standalone driver.

---

## Part 1: KVMFR shared-memory module

### 1.1 Build via DKMS (same release as the client/host!)

Client, host, and module must all be the **same tagged release**. Mixing B7 host
with git-master client (or vice versa) breaks the protocol. Clone the **B7**
release:

```bash
cd ~/source
git clone --branch B7 --single-branch https://github.com/gnif/LookingGlass.git looking-glass-B7
cd looking-glass-B7/module
sudo dkms install "."
```

### 1.2 Size the shared memory

Formula: `width × height × 4 × 2` bytes, rounded up to the next power of 2 in MB.

- 3440×1440 → 39,628,800 B ≈ 37.8 MB → 64 MB on paper.
- **Use 128 MB** for ultrawide headroom (avoids frame corruption).

### 1.3 Configure, load, and own the device

`/etc/modprobe.d/kvmfr.conf`:

```
options kvmfr static_size_mb=128
```

`/etc/modules-load.d/kvmfr.conf`:

```
kvmfr
```

`/etc/udev/rules.d/99-kvmfr.rules` (replace `your_username`):

```
SUBSYSTEM=="kvmfr", OWNER="your_username", GROUP="kvm", MODE="0660"
```

```bash
sudo udevadm control --reload-rules && sudo udevadm trigger
sudo modprobe kvmfr
ls -l /dev/kvmfr0   # expect: crw-rw---- 1 your_username kvm 508, 0 ... /dev/kvmfr0
```

**Gotchas (very common):**
- Device owned by `root:root` 600 → udev didn't (re)trigger (e.g. module
  reloaded after device creation). Fix the session, then fix the rule:
  `sudo chown your_username:kvm /dev/kvmfr0 && sudo chmod 660 /dev/kvmfr0`.
- Device is a **regular file** (`-rw...`) not a char device (`crw...`) → the VM
  was started **before** the module loaded and QEMU fell back to a file. Stop the
  VM, `sudo rm /dev/kvmfr0`, `sudo modprobe kvmfr`, then start the VM.

---

## Part 2: libvirt / QEMU configuration

All edits via `virsh edit <domain>` (e.g. `win11`).

### 2.1 Add the QEMU namespace

```xml
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
```

Required before the commandline block below.

### 2.2 Add the IVSHMEM/KVMFR device (JSON syntax — QEMU 6.2+)

Just before `</domain>`:

```xml
<qemu:commandline>
  <qemu:arg value="-device"/>
  <qemu:arg value="{'driver':'ivshmem-plain','id':'shmem0','memdev':'looking-glass'}"/>
  <qemu:arg value="-object"/>
  <qemu:arg value="{'qom-type':'memory-backend-file','id':'looking-glass','mem-path':'/dev/kvmfr0','size':134217728,'share':true}"/>
</qemu:commandline>
```

Size is bytes: 128 MB × 1024 × 1024 = 134,217,728. The **older comma-separated
syntax on QEMU 6.2+/libvirt 7.9+ causes "PCI slot not available"** — use JSON.

### 2.3 Remove conflicts & misbehaving devices

- **Delete any `<shmem>` block** (e.g. `<shmem name='looking-glass'>...`) — the
  commandline block replaces it; both at once = conflicts.
- **Disable ballooning**: `<memballoon model="none"/>` (virtio balloon fights
  VFIO).
- **Remove the USB tablet**: Looking Glass is incompatible with absolute
  pointing. Delete any `<input type="tablet" bus="usb">...`.

### 2.4 SPICE for input only (video comes from Looking Glass)

```xml
<graphics type="spice" autoport="yes">
  <listen type="address" address="127.0.0.1"/>
</graphics>
```

Install SPICE guest tools in Windows for clipboard sync.

### 2.5 cgroups: let QEMU touch /dev/kvmfr0

In `/etc/libvirt/qemu.conf`, uncomment `cgroup_device_acl` and add the device:

```
cgroup_device_acl = [
    "/dev/null", "/dev/full", "/dev/zero",
    "/dev/random", "/dev/urandom",
    "/dev/ptmx", "/dev/kvm",
    "/dev/kvmfr0"
]
```

```bash
sudo systemctl restart libvirtd
```

A "Permission denied" on `/dev/kvmfr0` at VM start is **always** cgroups here,
even when file perms look fine — confirm the ACL entry and restart libvirtd.

---

## Part 3: Build the Looking Glass client (same release!)

> The client release **must match** the host release (B7 host ↔ B7 client).

### 3.1 Dependencies (Arch/CachyOS)

```bash
sudo pacman -S cmake gcc libgl libegl fontconfig spice-protocol make nettle \
  pkgconf binutils libxi libxinerama libxss libxcursor libxpresent \
  libxkbcommon wayland-protocols ttf-dejavu libsamplerate
```

### 3.2 Build

```bash
cd ~/source/looking-glass-B7/client
mkdir build && cd build
cmake ..
make -j$(nproc)
```

Confirm X11 is on (needed for XWayland/OpenGL rendering):

```bash
grep ENABLE_X11 CMakeCache.txt   # ENABLE_X11:BOOL=ON
```

### 3.3 Install

```bash
sudo cp looking-glass-client /usr/local/bin/
```

---

## Part 4: Windows guest

1. **Looking Glass host** — install the **B7** host in Windows (runs as a SYSTEM
   service so it can capture the login screen). Binary:
   `C:\Program Files\Looking Glass (host)\looking-glass-host.exe`; log:
   `%ProgramData%\Looking Glass (host)\looking-glass-host.txt`.
2. **Virtual display** (**mandatory** for an SR-IOV VF — it has no physical
   output; optional for full passthrough with a real monitor) — install
   **Virtual Display Driver (VDD)**, `github.com/VirtualDrivers/Virtual-Display-Driver`
   (the maintained, *signed* successor to IddSampleDriver): in the guest,
   `winget install --id=VirtualDrivers.Virtual-Display-Driver -e`, then add a
   virtual display at the target resolution. Without one the LG host log shows
   `Failed to locate a valid output device`. **Do not** make it the primary
   display until LG is confirmed capturing it — too early can lock you out. See
   `references/field-lessons.md` §6.
3. **SPICE guest tools** for clipboard sync.

Locked out after making the virtual display primary? Recovery is to mount the QCOW2
from the host and remove the driver files — see
`references/troubleshooting.md` → "Locked out after display change".

---

## Part 5: Run + the one fix that matters on NVIDIA/Wayland

```bash
looking-glass-client
```

### The flickering-black-boxes fix (NVIDIA + Wayland)

On NVIDIA + Hyprland/Wayland you will almost certainly get flickering black
rectangles. **Root cause:** the default EGL renderer has a compositing
incompatibility with the NVIDIA driver under Wayland — *not* a D12/IddSampleDriver
problem. **Force the OpenGL renderer.** `~/.looking-glass-client.ini`:

```ini
[app]
renderer=opengl
```

That's the clean fix. (For the flicker specifically — don't waste time on
`trackDamage=false`, DXGI, downgrading egl-wayland, `egl:doubleBuffer=true`,
`app:allowDMA=no`, or unsetting `WAYLAND_DISPLAY`. Note:
`__NV_DISABLE_EXPLICIT_SYNC=1` does *not* fix this flicker, but it **is** the
fix for the separate explicit-sync **crash** just below.)

### The client-crash fix (NVIDIA + Wayland): explicit sync

A *different* NVIDIA+Wayland failure: the LG client dies with
`wp_linux_drm_syncobj_surface_v1 … error 3: Missing buffer` — the
`linux-drm-syncobj-v1` (explicit-sync) protocol erroring between the NVIDIA EGL
driver and the compositor. The flicker fix above does **not** address this.
Disable NVIDIA explicit sync:

```bash
__NV_DISABLE_EXPLICIT_SYNC=1 looking-glass-client
```

Make it permanent with a **user-level `.desktop` override**
(`~/.local/share/applications/looking-glass-client.desktop`, `Exec=env
__NV_DISABLE_EXPLICIT_SYNC=1 /usr/bin/looking-glass-client`) so package updates
don't clobber it. Full detail: `references/field-lessons.md` §5.

### Autostart the VM

```bash
virsh autostart win11           # enable at boot
virsh autostart --disable win11 # disable
```

---

## References (read on demand)

- **`references/field-lessons.md`** — ⚠️ **read this first.** Scars from a real
  kernel-7.x Alder Lake SR-IOV + Looking Glass install: the module-build traps
  (`BUILD_EXCLUSIVE_KERNEL` + per-kernel branches + initramfs), reliable VF
  creation, the PF/VF teardown trap, libvirt-XML sharp edges, the NVIDIA
  explicit-sync crash, clipboard-after-reboot, and a "verify the running state"
  cheatsheet. The procedure files below are the *happy path*; this is what
  actually happens when it isn't.
- **`references/troubleshooting.md`** — full symptom/cause/fix table; recovery
  for "locked out after display change" (mount QCOW2 via qemu-nbd); KVMFR/VFIO
  diagnostics. Read when a step fails or the VM misbehaves.
- **`references/config-reference.md`** — complete annotated VM XML, host config
  file table, Windows guest component table, the architecture explainer, and the
  evidence-backed **"Intel iGPU Code 43: root causes & fixes"** deep dive plus
  VBIOS/GOP sourcing. Read when assembling the final config or clearing Code 43.
- **`references/sr-iov-vgpu.md`** — the full **SR-IOV vGPU** procedure (share
  one iGPU across up to 7 VMs via virtual functions), ported to Arch/CachyOS +
  libvirt from Derek Seaman's Proxmox guide. Read when full GVT-d won't clear
  Code 43, or when you want to keep the iGPU on the host too.

## Quick command cheat sheet

```bash
# KVMFR/module health
lsmod | grep kvmfr ; dmesg | grep kvmfr ; ls -la /dev/kvmfr0

# VM lifecycle
virsh start win11 ; virsh shutdown win11 ; virsh destroy win11 ; virsh autostart win11

# Emergency disk access (mount Windows NTFS partition from host)
sudo modprobe nbd max_part=8
sudo qemu-nbd --connect=/dev/nbd0 /path/to/win11.qcow2
sudo mount -t ntfs-3g -o rw,remove_hiberfile /dev/nbd0p3 /mnt   # partition via fdisk -l
# ... work ...
sudo umount /mnt ; sudo qemu-nbd --disconnect /dev/nbd0

# Verbose client
looking-glass-client -d
```
