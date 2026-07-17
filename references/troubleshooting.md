# Troubleshooting Reference

Loaded on demand from `SKILL.md`. Use the symptom→cause→fix table first, then the
deeper sections.

## Symptom / cause / fix

| Symptom | Cause | Fix |
|---|---|---|
| `/sys/kernel/iommu_groups/` is empty despite VT-d + `intel_iommu=on` | `iommu=pt` (passthrough mode) blocks group population; dmesg shows `Default domain type: Passthrough` | Remove `iommu=pt` entirely (or use `iommu=1`). The binding param that matters is `rd.driver.pre=vfio-pci`. |
| VFIO params vanish after a system update | CachyOS regenerates systemd-boot entries, overwriting manual `/boot/loader/entries/*.conf` edits | Set `LINUX_OPTIONS` in `/etc/sdboot-manage.conf` and run `sudo sdboot-manage gen`. Never edit boot entries directly. |
| `lspci -nnk` shows **no** `Kernel driver in use` for the GPU | VFIO modules aren't in the initramfs | Confirm in image: `sudo lsinitcpio /boot/initramfs-linux-cachyos.img \| grep vfio` (name may be `-lts`). Re-check `/etc/mkinitcpio.conf` `MODULES=(vfio_pci vfio vfio_iommu_type1)` and `sudo mkinitcpio -P`. |
| VM won't start: `can't open backing store /dev/kvmfr0 ... Permission denied` | cgroups policy blocks QEMU from the device even when file perms are fine | Add `/dev/kvmfr0` to `cgroup_device_acl` in `/etc/libvirt/qemu.conf`; `sudo systemctl restart libvirtd`. |
| `/dev/kvmfr0` owned by `root:root` mode 600 | udev rule didn't (re)trigger (often after reloading the module) | Session: `sudo chown $USER:kvm /dev/kvmfr0 && sudo chmod 660 /dev/kvmfr0`. Persistent: confirm `/etc/udev/rules.d/99-kvmfr.rules`, then `sudo udevadm control --reload-rules && sudo udevadm trigger`. |
| `/dev/kvmfr0` is a **regular file** (`-rw...`) not a char device (`crw...`) | VM started **before** the module loaded; QEMU fell back to a plain file | `virsh destroy win11` → `sudo rm /dev/kvmfr0` → `sudo modprobe kvmfr` → `virsh start win11`. Ensure `kvmfr` is in `/etc/modules-load.d/kvmfr.conf`. |
| VM won't start: **"PCI slot not available"** | Legacy comma-separated QEMU args on QEMU 6.2+ / libvirt 7.9+ | Use the **JSON-style** `qemu:commandline` block from Part 2.2. |
| QEMU crashes with `vfio_container_region_add` | CPU `host-passthrough` + `migratable` conflicts with IVSHMEM memory regions | Use `<cpu mode='host-model' check='none'>` with **no** `migratable` attribute at all; include the `<topology>` block. |
| Flickering black boxes/rectangles in Looking Glass (NVIDIA + Wayland) | EGL renderer compositing incompatibility with NVIDIA under Wayland | `~/.looking-glass-client.ini` → `[app]` `renderer=opengl`. Not a D12/IddSampleDriver issue. |
| No display in Looking Glass | Host service not running, or wrong/virtual display not captured | Check `%ProgramData%\Looking Glass (host)\looking-glass-host.txt`; confirm IddSampleDriver virtual display is present and capturing. |
| Mouse input problems in Looking Glass | Absolute-pointing USB tablet present in VM | Remove `<input type="tablet" bus="usb">` from the VM XML. |
| No clipboard sync | SPICE guest tools missing | Install SPICE guest tools in Windows. |
| Windows guest shows **Code 43** on the GPU | **Intel iGPU:** usually Windows nested virtualization (VBS / Memory Integrity / **Virtual Machine Platform** / WSL2) or `<hyperv mode='custom'>` over-config — NOT a "can't POST" failure (it enumerates fine). **NVIDIA:** hypervisor detection. | **Intel iGPU:** (1) guest: OFF — Memory Integrity, Virtual Machine Platform, Hyper-V, no WSL2; (2) libvirt `<hyperv mode='passthrough'/>` (do **not** use the NVIDIA `kvm-hidden`/`vendor_id` trick); (3) Intel "Recommended" driver .exe + Group Policy "Do not include drivers with Windows Update"; (4) remove `<video>` so the iGPU is primary; (5) Disable→Enable in Device Manager; (6) GOP romfile (LongQT-sea); (7) switch to SR-IOV VFs. **NVIDIA:** `<kvm><hidden state='on'/>` + hyperv `vendor_id`. Deep dive: `references/config-reference.md` → "Intel iGPU Code 43". |
| **Code 10**: "This device cannot start. The driver trying to start is not the same as the driver for the POSTed display adapter." | The adapter Windows POSTed with uses a different driver than the passed-through GPU (competing emulated `<video>`) | Install the correct driver on the display adapter the guest POSTed with; make the passed GPU the primary/only display (remove `<video>`, set Primary GPU / `x-vga=1`); supply a GOP romfile so OVMF POSTs the right device |
| SR-IOV: **no VFs** after a kernel update | DKMS module not rebuilt for the new kernel | Re-run the DKMS build/install for the new kernel; consider pinning it (`references/sr-iov-vgpu.md`) |
| `i915: module verification failed: signature and/or required key missing – tainting kernel` + no VFs | Secure Boot on but the DKMS MOK key wasn't enrolled | `sudo mokutil --import /var/lib/dkms/mok.pub`, reboot, Enroll MOK; or disable Secure Boot |
| VFs/Code 43 appear right after enabling WSL2 or "Virtual Machine Platform" | Windows nested virt breaks the Intel iGPU driver | Disable Virtual Machine Platform / WSL2 / Memory Integrity in the guest |
| Host loses its display / can't boot graphically (GVT-d) | `i915` still claiming the iGPU, or host has no other GPU | Blacklist `i915` and ensure a separate GPU drives the host. |

## KVMFR / module diagnostics

```bash
lsmod | grep kvmfr          # is it loaded?
dmesg | grep kvmfr          # load errors / size warnings
ls -la /dev/kvmfr0          # char device? correct owner/perm?
cat /proc/cmdline           # confirm intel_iommu=on, rd.driver.pre=vfio-pci
ls /sys/kernel/iommu_groups/  # should NOT be empty
```

If the module rebuild is broken after a kernel update, rebuild via DKMS:

```bash
sudo dkms remove kvmfr/<version> --all
sudo dkms install "."
```

## Locked out after display change (IddSampleDriver set primary)

If the virtual display was made primary before Looking Glass could capture it,
SPICE/RDP may be unreachable. Mount the QCOW2 from the host and disable the
driver:

```bash
# Ensure the VM is stopped
virsh destroy win11

# Attach the disk via NBD
sudo modprobe nbd max_part=8
sudo qemu-nbd --connect=/dev/nbd0 /path/to/win11.qcow2
sudo fdisk -l /dev/nbd0          # find the Windows NTFS partition (often p3)

# remove_hiberfile is required: Windows Fast Startup leaves the FS dirty
sudo mount -t ntfs-3g -o rw,remove_hiberfile /dev/nbd0p3 /mnt

# Disable the driver by renaming its directory
sudo mv /mnt/path/to/IddSampleDriver /mnt/path/to/IddSampleDriver.bak

# Clean up
sudo umount /mnt
sudo qemu-nbd --disconnect /dev/nbd0
```

Start the VM; it should fall back to the SPICE display.

## I can't see the guest GPU at all in `lspci`

- Confirm VT-d is on in BIOS (not just VT-x).
- Re-check `cat /proc/cmdline` for `intel_iommu=on`.
- For the iGPU under GVT-d, confirm `i915` is actually blacklisted
  (`lsmod | grep i915` should be empty) and that the iGPU IDs are in
  `vfio-pci ids=...`.
- Some boards hide the iGPU when a discrete GPU is the primary display device —
  check BIOS "primary display" / "iGPU multi-monitor" settings.
