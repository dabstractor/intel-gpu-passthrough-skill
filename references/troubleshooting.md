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
| No display in Looking Glass | Host service not running, or wrong/virtual display not captured | Check `%ProgramData%\Looking Glass (host)\looking-glass-host.txt`; confirm a virtual display (VDD) is present and capturing. On a headless SR-IOV VF this is the `Failed to locate a valid output device` case (`field-lessons.md` §6). |
| Mouse input problems in Looking Glass | Absolute-pointing USB tablet present in VM | Remove `<input type="tablet" bus="usb">` from the VM XML. |
| No clipboard sync | SPICE guest tools missing | Install SPICE guest tools in Windows. |
| Windows guest shows **Code 43** on the GPU | **Intel iGPU:** usually Windows nested virtualization (VBS / Memory Integrity / **Virtual Machine Platform** / WSL2) or `<hyperv mode='custom'>` over-config — NOT a "can't POST" failure (it enumerates fine). **NVIDIA:** hypervisor detection. | **Intel iGPU:** (1) guest: OFF — Memory Integrity, Virtual Machine Platform, Hyper-V, no WSL2; (2) libvirt `<hyperv mode='passthrough'/>` (do **not** use the NVIDIA `kvm-hidden`/`vendor_id` trick); (3) Intel "Recommended" driver .exe + Group Policy "Do not include drivers with Windows Update"; (4) remove `<video>` so the iGPU is primary; (5) Disable→Enable in Device Manager; (6) GOP romfile (LongQT-sea); (7) switch to SR-IOV VFs. **NVIDIA:** `<kvm><hidden state='on'/>` + hyperv `vendor_id`. Deep dive: `references/config-reference.md` → "Intel iGPU Code 43". |
| **Code 10**: "This device cannot start. The driver trying to start is not the same as the driver for the POSTed display adapter." | The adapter Windows POSTed with uses a different driver than the passed-through GPU (competing emulated `<video>`) | Install the correct driver on the display adapter the guest POSTed with; make the passed GPU the primary/only display (remove `<video>`, set Primary GPU / `x-vga=1`); supply a GOP romfile so OVMF POSTs the right device |
| SR-IOV: **no VFs** after a kernel update | DKMS module not rebuilt for the new kernel | Re-run the DKMS build/install for the new kernel; consider pinning it (`references/sr-iov-vgpu.md`) |
| `i915: module verification failed: signature and/or required key missing – tainting kernel` + no VFs | Secure Boot on but the DKMS MOK key wasn't enrolled | `sudo mokutil --import /var/lib/dkms/mok.pub`, reboot, Enroll MOK; or disable Secure Boot |
| VFs/Code 43 appear right after enabling WSL2 or "Virtual Machine Platform" | Windows nested virt breaks the Intel iGPU driver | Disable Virtual Machine Platform / WSL2 / Memory Integrity in the guest |
| Windows guest **Code 43** on the Intel VF right after a strongtz module update (≥ 2026.09.16) | The new module **disables CCS0 by default on Xe_LP platforms** (ADL-S iGPU = UHD 770); the Windows driver then fails to start the adapter. The module's own README names this exact case under "issues running Windows guest VMs". Confirmed 2026-10-16: guest driver 32.0.101.7088 = Code 43; LG host log shows only "Microsoft Basic Render Driver" adapters → `Failed to locate a valid output device` → LG host service exits → client stuck on startup splash. | Host kernel cmdline: **`i915.xelp_enable_ccs=1`** (param ships in the DKMS module — `modinfo -p /lib/modules/$(uname -r)/updates/dkms/i915.ko.zst | grep ccs`). Append to the systemd-boot entry's `options` line, reboot host, restart VM, then restart the LG client (field-lessons §7). No guest driver change: README's tested drivers (32.0.101.6460/6259) are *older* than 7088. |
| VM reaches its gateway (192.168.122.1) but no internet — ALL forwarded protocols dead (ICMP/TCP/UDP); appeared after docker ≥1.29.8 | Docker's rewritten **`DOCKER-FORWARD` chain terminates non-docker forwarded traffic** before libvirt's ACCEPT rules later in the FORWARD chain (libvirt ≤12.7 chains sit behind it). Confirmed 2026-10-16 by counter step-down: 7753 pkts entered DOCKER-USER/DOCKER-FORWARD, only 1103 reached the next rule. | `iptables-nft -I DOCKER-USER 1 -i virbr0 -j ACCEPT` + `... -I DOCKER-USER 2 -o virbr0 -j ACCEPT` (DOCKER-USER is docker's documented, restart-preserved hook, evaluated FIRST). Permanent: `/etc/systemd/system/docker-user-libvirt.service` (After=docker.service; installed 2026-10-16). |
| Guest packets die with `iptables` dumps looking totally clean: requests visible on `vnet0` via tcpdump, **nothing** on the egress NIC, MASQUERADE counter stuck at 0, and `conntrack -L -s <guest-ip>` empty | **A native nftables `inet`-family table is dropping at a pre-conntrack hook** — `iptables-nft -S` only shows the `ip`/`ip6` compat families and cannot see it. Here: Private Internet Access killswitch (`table inet pia_qbit`), engaged since the wifi→ethernet switch (tunnel drop → killswitch), surviving because the host never rebooted. | Diagnose: `nft list tables` — any `inet` table you don't own is suspect; `nft list table inet pia_qbit`. **Zero conntrack entries = death pre-conntrack = look at inet/raw-priority hooks, not iptables.** Fix: `nft insert rule inet pia_qbit <chain> iifname "virbr0" accept` (+ `oifname`) at the top of every base chain (input+forward both — the input hook was also eating DHCP broadcasts and DNS to dnsmasq). nft inserts do NOT survive reboot: delete the stale table if PIA is unused (`nft delete table inet pia_qbit`), or install `virbr0-nft-exempt.service` (see field-lessons §10). |
| Host loses its display / can't boot graphically (GVT-d) | `i915` still claiming the iGPU, or host has no other GPU | Blacklist `i915` and ensure a separate GPU drives the host. |
| SR-IOV: `dkms status` shows `added`, never `installed`; no error surfaced | Kernel newer than the module's `BUILD_EXCLUSIVE_KERNEL` → build silently skipped | Clone the matching `kernel-vX.Y` branch (not `master`); check `make.log`; see `field-lessons.md` §1 |
| SR-IOV: `dkms` says `installed` but `sriov_numvfs=0` and no `max_vfs` param | Initramfs still packs stock `i915`; strongtz never actually loaded | `sudo mkinitcpio -P`; verify `lsinitcpio /boot/initramfs-linux.img \| grep i915` shows `updates/dkms/`; reboot |
| SR-IOV: VM start fails `Device 0000:00:02.1 not found` | VF doesn't exist — a prior start grabbed the PF and tore down the VFs | Ensure `numvfs=7`; pass a **VF** (`00:02.1`), never the PF (`field-lessons.md` §3) |
| `numvfs` resets 7→0 every time the VM starts | VM hostdev points at the **PF** `00:02.0`; detaching the PF kills its VFs | Rewire hostdev source to a VF (`function='0x1'`); PF stays on `i915` |
| VM won't start: `Hook script execution failed: unexpected exit status 1` | `/etc/libvirt/hooks/qemu` returns non-zero on the `prepare` phase (no `exit 0`) | End every hook with `exit 0` (`field-lessons.md` §4) |
| `XML document failed to validate against schema` after an edit | Invalid XML — classic: `<video model='none'/>` (model as attribute) | Use `<video><model type='none'/></video>` or omit `<video>`; `virt-xml-validate` |
| LG client crash: `wp_linux_drm_syncobj_surface_v1 … error 3: Missing buffer` | NVIDIA EGL explicit-sync protocol erroring with the compositor | Launch with `__NV_DISABLE_EXPLICIT_SYNC=1`; make permanent via a `~/.local/share/applications/` `.desktop` override (`field-lessons.md` §5) |
| LG host log: `Failed to locate a valid output device` | Headless GPU (SR-IOV VF) has no monitor → no desktop for D12 to capture | Install a virtual display driver (VDD) on the VF (`field-lessons.md` §6) |
| Clipboard sync stops; video still fine | Rebooting the VM killed the LG client's SPICE connection (IVSHMEM video survives) | Restart the Looking Glass client; verify `ss -tnp \| grep 5900` |
| Mouse feels laggy/"molasses" in LG | LG mouse smoothing / host acceleration | `[input] rawMouse=yes`, `mouseSmoothing=no`; Windows: off "Enhance pointer precision" |

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

## Locked out after display change (virtual display set primary)

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
