# SR-IOV vGPU: share one iGPU across VMs (Alder Lake)

The **most reliable path** for Alder Lake (12th-gen) iGPU virtualization, ported
to Arch/CachyOS + libvirt from Derek Seaman's Proxmox guide (the Proxmox GUI
steps are translated to libvirt XML below). Creates up to **7 virtual functions
(VFs)** from the UHD 770; each VM gets its own VF, and the host keeps using the
iGPU. Tradeoff: **no physical monitor output** (HDMI/DP is tied to the root
function) — display via RDP / SPICE / Looking Glass only.

## Requirements

- **11th-gen Intel or newer** for iGPU VT-d / SR-IOV. 12th gen (Alder Lake)
  confirmed working; 11th gen is hit-or-miss; pre-11th gen unsupported.
- **VT-d enabled in BIOS**, plus an **"SR-IOV"** toggle on some boards (enable
  if present). "NO IOMMU detected" when adding the VF = VT-d not enabled.
- Host display from another GPU (your NVIDIA card).
- The host **keeps `i915` loaded** — you are NOT blacklisting it here (unlike
  full GVT-d passthrough). `i915` runs the SR-IOV PF and spawns the VFs.

## 1. Install the SR-IOV i915 module (DKMS)

Out-of-tree module: `github.com/strongtz/i915-sriov-dkms`.

```bash
sudo pacman -S linux-cachyos-headers dkms sysfsutils git   # headers for your kernel
git clone https://github.com/strongtz/i915-sriov-dkms.git
cd i915-sriov-dkms
sudo dkms add .
V=$(dkms status -m i915-sriov-dkms | head -1 | sed -E 's/.*,\s*([^,]+):.*/\1/')
sudo dkms install -m i915-sriov-dkms/"$V" --force
dkms status        # MUST show "installed"
```

(An AUR package exists too — `yay -S i915-sriov-dkms-dkms` — if you prefer.)

**Secure Boot:** if enabled, enroll the DKMS key (`sudo mokutil --import
/var/lib/dkms/mok.pub`, reboot, Enroll MOK). CachyOS often auto-signs; if you see
`i915: module verification failed: signature and/or required key missing –
tainting kernel` **and no VFs**, that's the cause.

> Newer kernels sometimes break this module — check the repo's Issues before
> upgrading. Consider pinning your kernel (see step 7).

## 2. Kernel cmdline (SR-IOV uses `iommu=pt` — different from full passthrough!)

CachyOS uses systemd-boot, so set this persistently in `/etc/sdboot-manage.conf`:

```
LINUX_OPTIONS="intel_iommu=on iommu=pt i915.enable_guc=3 i915.max_vfs=7"
```

```bash
sudo sdboot-manage gen
```

> **`iommu=pt` is deliberate here** and is part of the proven SR-IOV recipe
> (confirmed in Derek Seaman's guide and by image analysis of his GRUB line).
> This **contradicts** the full-passthrough advice in `SKILL.md` Part 0.1 (which
> says *remove* `iommu=pt` because it empties IOMMU groups). They're different
> paths: `iommu=pt` is correct for SR-IOV virtual functions; avoid it for
> whole-device vfio-pci binding.

## 3. Create the 7 VFs

The iGPU is usually at `00:02.0`. Persist 7 VFs via sysfs (`sysfsutils`):

```bash
echo "devices/pci0000:00/0000:00:02.0/sriov_numvfs = 7" | sudo tee /etc/sysfs.conf
sudo reboot
```

Verify (1 root + 7 VFs, and `Enabled 7 VFs` in dmesg):

```bash
lspci | grep VGA          # eight lines: 00:02.0 root + 00:02.1 .. 00:02.7
dmesg | grep i915         # ... i915 0000:00:02.0: Enabled 7 VFs
```

## 4. Pass one VF to the VM (libvirt XML)

Pick **one** VF (e.g. `00:02.1`). **Never** pass the root `00:02.0` to a VM, and
give each VM a **unique** VF. Make the VF the primary display by removing the
emulated video:

```xml
<!-- REMOVE the <video> element entirely (or set <video model='none'/>) -->
<!-- <graphics>: keep SPICE/RDP for input + display; the VF has no physical output -->

<hostdev mode='subsystem' type='pci' managed='yes'>
  <source>
    <address domain='0x0000' bus='0x00' slot='0x02' function='0x1'/>
  </source>
  <!-- add <rom file='/usr/share/kvm/igpu.rom'/> only if you need the GOP fallback -->
</hostdev>
```

Proxmox GUI equivalents (for cross-reference): Add PCI Device → **Raw Device**,
pick a `.1`–`.7` function, **do not** tick "All Functions", **do** tick "Primary
GPU", and set Display → **none**. (`x-vga=1` / `vga: none` in `qemu-server`.)

## 5. Windows guest

1. Install Windows, then the **Intel "Recommended" graphics driver** (a specific
   version — e.g. `32.0.101.7085` for Alder Lake). Windows sees **"Intel Iris
   Xe"**; the stock WHQL driver works **without Code 43** in a correct SR-IOV
   setup.
2. **Stop Windows Update downgrading the driver** (it loves to swap in a broken
   2022 build): Group Policy → Computer Configuration → Administrative Templates
   → Windows Components → Windows Update → **Do not include drivers with Windows
   Update** = Enabled.
3. Display is **RDP / SPICE / Looking Glass only** — no physical monitor. Looking
   Glass works with a VF because it captures via D12 Desktop Duplication (no
   physical output needed); pair it with the KVMFR/IVSHMEM device from `SKILL.md`
   Part 1–2 if you want low-latency local display instead of RDP.

## 6. The Code 43 trap (if it appears)

SR-IOV setups hit Code 43 almost exclusively from **Windows nested
virtualization** — not the hypervisor hiding that fixes NVIDIA. In the guest,
turn **OFF** Memory Integrity, **Virtual Machine Platform**, Hyper-V, and don't
install WSL2/Sandbox; and use `<hyperv mode='passthrough'/>` (not a custom
`vendor_id` block). Full triage: `references/config-reference.md` → "Intel iGPU
Code 43."

## 7. Kernel upgrades

Every new kernel needs a DKMS rebuild (re-run step 1's `dkms install`) or the VFs
vanish. Pin your kernel if you want upgrades on your own schedule (CachyOS: hold
the kernel package; Proxmox: `proxmox-boot-tool kernel pin`).
