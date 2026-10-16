# Field Lessons: Alder Lake iGPU → Windows via SR-IOV + Looking Glass

The rest of this skill is the *procedure*. This file is the **scars**: everything
that bit a real, multi-day install on an **i9-12900K / UHD 770 / RTX 3080 Ti,
kernel 7.1.x, Hyprland, Looking Glass B7**, that the procedure files either got
wrong or didn't say. Read it before you start, and again when something silently
fails. Every item here was reproduced and resolved.

## 0. Meta-lessons (read these first — they save the most time)

- **Pick a lane and align *everything* to it.** GVT-d and SR-IOV are mutually
  exclusive *host* configurations:
  - **GVT-d:** iGPU bound to `vfio-pci` (`vfio-pci.ids=8086:4680`), `i915`
    blacklisted, whole device (`00:02.0`) passed to the VM.
  - **SR-IOV:** `i915` (the strongtz fork) *owns* the iGPU as the PF
    (`i915.enable_guc=3`, **no** `vfio-pci.ids` for the iGPU), the VM gets a
    **virtual function** (`00:02.1`).
  One cmdline, one binding model. Mixing them produces silent, confusing
  failures. Decide first, then make the cmdline + the VM XML + the binding all
  match.
- **Verify the RUNNING state, not the config files.** Config and reality diverge
  silently in this stack. Before you assert anything, check:
  `cat /proc/cmdline` (live params), `readlink .../driver`, `cat sriov_numvfs`,
  `modinfo i915` (`filename:` → `updates/dkms/` = strongtz; `kernel/drivers/...`
  = stock), `lsinitcpio /boot/initramfs-linux.img | grep i915` (which one is
  *packed*). "I set it in the file" is not "it is running."
- **Keep a fallback initramfs entry and know the chroot recovery.** A bad DKMS
  module or cmdline can make the host unbootable. You *will* need it during
  SR-IOV bring-up. (Arch ISO → mount root → `arch-chroot` → `mkinitcpio -P` →
  boot the `-fallback` entry.)
- **`virt-xml-validate <file> domain` after every XML edit.** libvirt silently
  rejects invalid XML ("XML document failed to validate against schema") and your
  change simply doesn't apply — while *looking* like it did.
- **Don't rush to revert; don't operate on sunk cost.** Several wrong turns here
  came from either panic-reverting or sticking with a half-done path because work
  was already invested. Diagnose the actual running state, then act.

## 1. The SR-IOV module build on a modern kernel (the single hardest part)

The strongtz `i915-sriov-dkms` module does **not** "just build." Four traps:

1. **`BUILD_EXCLUSIVE_KERNEL` silently refuses to build on too-new kernels.**
   The module's `dkms.conf` carries a regex like `^(6\.1[2-7]|7\.0)` (varies by
   version). If your kernel doesn't match, **DKMS won't even attempt the build**:
   `dkms status` shows `added`, never `installed`, and there is **no error on
   stderr**. Always check `cat /var/lib/dkms/i915-sriov-dkms/<ver>/build/make.log`
   and the `BUILD_EXCLUSIVE_KERNEL` line in `dkms.conf`.
2. **There are per-kernel branches.** If your kernel is past what `master`
   (or your distro package) supports, the fix is a **matching branch**, e.g.
   `kernel-v6.12`, `kernel-v6.19`, `kernel-v7.0`, `kernel-v7.1`. Check
   `git branch -a` upstream. The right branch both relaxes `BUILD_EXCLUSIVE` for
   your kernel *and* carries the compat fixes for it. Don't fight the build on
   the wrong branch.
3. **The conftest error.** On a nominally-supported kernel the build can still
   die with `conftest/results.h: error: unterminated #ifndef`. That's a known
   conftest-framework bug; it's fixed in newer versions/branches (commit messages
   literally say "fix conftest on gcc 13"). Move to a newer branch/version.
4. **"Built" ≠ "loaded." The initramfs wins.** `dkms status` can show
   `installed` and the running kernel still loads **stock** `i915`, because the
   initramfs was packed *before* the build (or the DKMS hook didn't rebuild
   mkinitcpio). Stock `i915` has **no `max_vfs` parameter** and creates **0
   VFs**, so `i915.max_vfs=7` on the cmdline is silently ignored.
   - Diagnose: `cat /sys/module/i915/parameters/max_vfs` → if it errors/empty,
     stock is loaded. `lsinitcpio /boot/initramfs-linux.img | grep i915` → if it
     shows `kernel/drivers/gpu/drm/i915/` (not `updates/dkms/`), the initramfs
     has stock.
   - Fix: `sudo mkinitcpio -P`, then **verify the initramfs now contains
     `updates/dkms/i915.ko`** before rebooting.
5. **Secure Boot "taint" is not fatal.** `i915: module verification failed:
   signature and/or required key missing – tainting kernel` looks alarming but
   the module **still loads and creates VFs** — the kernel is merely "tainted."
   Only chase MOK enrollment (`mokutil --import /var/lib/dkms/mok.pub`) if the
   module is actually *rejected* (won't load at all). If you see the taint line
   *and* VFs exist, you're done.

## 2. Creating the VFs reliably

- **`i915.max_vfs=7` on the cmdline may be silently ignored** (see §1.4). Don't
  rely on it as the sole mechanism.
- **The `/etc/sysfs.conf` `sriov_numvfs = 7` line is dead unless `sysfsutils` is
  actually installed AND its service runs** — and even then it can lose the race
  with `i915` binding the PF. Confirm with `systemctl status sysfsutils`; on
  minimal Arch it's often absent.
- **Reliable method: a udev rule that fires when `i915` binds the PF.** Target
  the PF by PCI address so it doesn't re-fire for the VFs (which share the
  device ID):
  ```
  # /etc/udev/rules.d/99-i915-sriov.rules
  ACTION=="bind", SUBSYSTEM=="pci", DRIVER=="i915", KERNEL=="0000:00:02.0", RUN+="/bin/sh -c 'echo 7 > /sys/bus/pci/devices/0000:00:02.0/sriov_numvfs'"
  ```
  then `sudo udevadm control --reload-rules`. Verify after reboot:
  `cat /sys/bus/pci/devices/0000:00:02.0/sriov_numvfs` should read `7` with no
  manual step.
- **Writing `sriov_numvfs` requires the PF to be on `i915`** (not vfio-bound).
  If a VM is holding the PF, the write fails silently and `numvfs` stays 0.

## 3. The PF/VF teardown trap (cost us hours — memorize this)

**Passing the PF (`00:02.0`) to a VM tears down the VFs.** Detaching the PF to
`vfio-pci` kills its virtual functions, so `numvfs` drops 7→0. The VM may even
"start," but the VFs are gone — and you can't re-create them because the PF is
now vfio-bound. Symptom at next start: `Device 0000:00:02.1 not found`.

**Always pass a VF (`00:02.1`…`00:02.7`), never the PF.** Once the VM attaches a
VF, the PF stays on `i915` and the *other* VFs persist (`numvfs` holds at 7).
This is the difference between a stable SR-IOV setup and one that breaks every
start.

## 4. libvirt XML sharp edges

- **`<video model='none'/>` is INVALID schema.** `model` is an attribute of the
  child `<model>`, not of `<video>`. The valid no-video forms are
  `<video><model type='none'/></video>` or simply **omit `<video>` entirely**.
  The invalid form produces `XML document failed to validate against schema` and
  the change silently doesn't apply.
- **A libvirt hook that doesn't `exit 0` aborts VM start.** libvirt calls
  `/etc/libvirt/hooks/qemu` on *every* phase (`prepare`, `start`, `started`,
  …). A hook like `[ "$2" = started ] && setfacl …` returns non-zero on
  `prepare` → `Hook script execution failed: unexpected exit status 1`. **Every
  hook ends with `exit 0`.** Useful pattern (self-healing LG shared-memory file
  in `/dev/shm`, which is tmpfs and gets wiped on host reboot):
  ```sh
  #!/bin/sh
  if [ "$1" = qmk-win ] && [ "$2" = prepare ]; then
      truncate -s 128M /dev/shm/looking-glass
      chown libvirt-qemu:libvirt-qemu /dev/shm/looking-glass
      chmod 660 /dev/shm/looking-glass
      setfacl -m u:<youruser>:rw /dev/shm/looking-glass 2>/dev/null || true
  fi
  exit 0
  ```
- **Editing a hostdev PCI address: don't greedy-match.** The substring
  `bus='0x00' slot='0x02' function='0x0'` can appear in a PCIe-root-port
  controller's *guest* address **and** in the hostdev `<source>`. Match the full
  `<address domain='0x0000' bus='0x00' slot='0x02' function='0x0'/>` line (the
  source one lacks `type='pci'`/`multifunction`), or you'll rewrite the wrong
  device and get `Attempted double use of PCI Address`.
- **Use `/dev/shm` for the IVSHMEM file if you skip the KVMFR module.** The
  memory-backend-file can point at `/dev/shm/looking-glass` (a tmpfs file) with
  `size:134217728,share:true`. It's slightly higher-latency than the KVMFR
  kernel module but works without a DKMS build — and the hook above keeps it
  alive across reboots.

## 5. Looking Glass on NVIDIA + Wayland

- **Crash: `wp_linux_drm_syncobj_surface_v1 … error 3: Missing buffer`.** This
  is the `linux-drm-syncobj-v1` (explicit sync) protocol erroring between the
  NVIDIA EGL driver and the compositor; the compositor kills the client. The fix
  is **`__NV_DISABLE_EXPLICIT_SYNC=1`** — disable NVIDIA explicit sync so the
  protocol isn't used. Make it permanent with a **user-level `.desktop`
  override** (survives package updates; takes precedence over
  `/usr/share/applications/`):
  ```ini
  # ~/.local/share/applications/looking-glass-client.desktop  (copy of the system one)
  Exec=env __NV_DISABLE_EXPLICIT_SYNC=1 /usr/bin/looking-glass-client
  ```
  NOTE: this is a *different* symptom from the flickering-black-boxes problem —
  that one is fixed by `[app] renderer=opengl`. The two fixes solve two
  different NVIDIA+Wayland failures; you may need both.
- **Config file is `~/.looking-glass-client.ini`** (flat INI in your home, *not*
  `~/.config/...`). The client logs `Loading config from:
  /home/<you>/.looking-glass-client.ini` on startup — trust that line.
- **Mouse input latency ("molasses"):** `[input] rawMouse=yes` +
  `mouseSmoothing=no`. In Windows, "Enhance pointer precision" *is* mouse
  acceleration (no separate slider) — turn it off. Diagnose with
  `[win] showFPS=yes`: idle FPS well above 60 means you are **not** VSync-capped
  (so don't chase a VSync toggle); a low **UPS** during 3D work means the GPU is
  render-bound (lower the virtual-display resolution).
- **Debug/verbose flag is `-d`** (`looking-glass-client -d`), not `--verbose`.
- **Keyboard grab is exclusive — there is no per-keybind exemption.** While
  grabbed, every key goes to the guest; the `escapeKey` (default `KEY_SCROLLLOCK`
  = 70) releases. If you want your compositor keybinds to keep working, **don't
  grab** (`grabKeyboardOnFocus=no`, the default) and accept that the Super key
  is absorbed by the compositor and won't reach the guest. `escapeKey` values
  must be the full input-code name (`KEY_RIGHTCTRL`), not `rightctrl`.

## 6. A headless VF needs a virtual display

An SR-IOV VF has **no physical output**, so Windows builds no desktop on it, and
LG's D12/Desktop Duplication reports `Failed to locate a valid output device`.
You must give it a fake monitor.

- Use **Virtual Display Driver (VDD)** — `github.com/VirtualDrivers/Virtual-Display-Driver`
  (the maintained, *signed*, IddCx-1.10 successor to IddSampleDriver; the
  `MetaTubeSoft` URL is dead). In the guest: `winget install
  --id=VirtualDrivers.Virtual-Display-Driver -e`, then add a virtual display at
  your resolution via its control app.
- Once the virtual display exists on the Intel VF, D12 captures it. Confirm in
  the LG host log: `Device Description: Intel(R) UHD Graphics 770` — that's your
  proof the VF (not software WARP) is rendering.
- To install VDD you need *some* display first: temporarily add
  `<video><model type='qxl'/></video>` so virt-manager's SPICE console works,
  install VDD, confirm LG captures, then revert to `model='none'`.

## 7. Clipboard

- **Clipboard travels over SPICE, not IVSHMEM.** IVSHMEM carries only video.
- **Rebooting the VM kills the LG client's SPICE connection** — and the client
  does *not* reliably re-establish clipboard after that. Video keeps working
  (separate IVSHMEM path), so it *looks* fine, but clipboard (and SPICE input)
  silently die. **After any VM reboot, restart the Looking Glass client.**
  Diagnose from the host: `ss -tnp | grep 5900` — if nothing is connected to the
  SPICE port, the link is down.
- **Two SPICE-agent installs look like a conflict but usually aren't.** Both
  spice-guest-tools (`spice-agent` service) and virtio-win (`vdservice`) provide
  an agent — and confusingly the `spice-agent` service's binary *is*
  `vdservice.exe`. "I see `vdservice.exe` running" does **not** mean two agents
  are fighting. Check each service's `PathName` (e.g. via
  `Get-CimInstance Win32_Service`) to tell them apart; disable the redundant one
  by service, not by process name.

## 8. Host footguns

- **`options vfio-pci ids=10de:…` (your host NVIDIA GPU) in
  `/etc/modprobe.d/vfio.conf` is a loaded gun.** If the cmdline ever loses its
  iGPU vfio id (e.g. you switch lanes), vfio-pci will grab your **host display
  GPU** on the next boot → black screen → "can't boot." For SR-IOV, comment out
  *all* `vfio-pci ids` in modprobe.d — libvirt binds the VF to vfio-pci at VM
  start automatically.
- **`vfio-pci.disable_vga=1` on the cmdline** disables legacy VGA on vfio
  devices. Harmless for UEFI/OVMF guests but worth knowing if an iGPU init path
  misbehaves.
- **Cmdline `vfio-pci.ids=` vs modprobe.d `options vfio-pci ids=`:** the cmdline
  one is read at module load from the initramfs; the modprobe.d one applies when
  the module is modprobed. They can disagree. Align them, and remember the one
  in the initramfs wins for early binding.

## 9. Diagnostic cheatsheet — "is it actually running?"

```bash
# Lane / binding
cat /proc/cmdline                              # GVT-d (vfio-pci.ids=8086:4680) vs SR-IOV (i915.max_vfs)
readlink /sys/bus/pci/devices/0000:00:02.0/driver   # i915 vs vfio-pci
cat /sys/bus/pci/devices/0000:00:02.0/sriov_numvfs  # 0 = no VFs; 7 = good

# Which i915?
modinfo i915 | grep -E 'filename|^version'     # updates/dkms = strongtz; kernel/ = stock
ls /sys/module/i915/parameters/ | grep max_vfs # exists ONLY on the strongtz fork
lsinitcpio /boot/initramfs-linux.img | grep i915  # which one is *packed*

# SR-IOV build
dkms status                                    # must say "installed", not "added"
cat /var/lib/dkms/i915-sriov-dkms/*/build/make.log | tail   # real errors live here

# Looking Glass transport + clipboard
ss -tnp | grep 5900                            # SPICE connection present? (clipboard needs it)
pgrep -fa looking-glass                        # client running? (use -f; name is >15 chars)

# Libvirt XML
virt-xml-validate /path/to/domain.xml domain   # after EVERY edit
```

## 10. The invisible firewall layers (2026-10-16 network saga)

Symptom: VM up, gateway pingable, ALL internet dead — DHCP, DNS-to-dnsmasq,
forwarded everything. Took a full afternoon because **three independent
	diagnostic tools each showed a clean system**. The layered reality:

1. **`iptables-nft -S` shows only the `ip`/`ip6` compat families.** Native nftables
   `inet`-family tables (here: PIA's killswitch, `table inet pia_qbit`) evaluate
   at hooks BEFORE them and are invisible to that dump. When "iptables looks
   clean," run `nft list tables` and inventory every table you don't recognize.
2. **Zero conntrack entries for a live flow = the packet died pre-conntrack**
   (inet/raw-priority hook). `conntrack -L -s <guest-ip>` empty while tcpdump
   shows requests on `vnet0` is an instant verdict: stop looking at iptables.
3. **FORWARD counter arithmetic localizes chain-level death**: if rule N shows
   7753 pkts and rule N+1 shows 1103, the packet died inside rule N's chain
   (docker's `DOCKER-FORWARD`, which since ~1.29.8 terminates non-docker
   forwarded traffic instead of falling through).
4. **An engaged VPN killswitch outlives the VPN.** PIA's tunnel dropped when
   the host switched wifi→ethernet; its killswitch table stayed armed for
   weeks because the host never rebooted. "It used to work flawlessly" ended
   the day the uplink changed, not the day anything VM-related changed.

Permanent artifacts from this saga (2026-10-16):
- `/etc/systemd/system/docker-user-libvirt.service` — re-asserts DOCKER-USER
  accepts for virbr0 after every docker start.
- The nft `inet` accepts for virbr0 are **runtime-only** — after a reboot,
  re-apply via `~/fix-qmk-persist.sh` (installs `virbr0-nft-exempt.service`),
  or if PIA is gone for good: `sudo nft delete table inet pia_qbit`.
- Guest runs a static IP (192.168.122.50/24, gw 192.168.122.1, DNS via
  192.168.122.1 + 1.1.1.1) — DHCP should work again now that the input hook
  accepts virbr0; flip the adapter back if you prefer DHCP.
