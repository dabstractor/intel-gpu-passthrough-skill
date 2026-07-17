# Alder Lake iGPU / GPU Passthrough + Looking Glass

An [Agent Skill](https://github.com/earendil-works/pi-coding-agent) that guides
configuring KVM/QEMU GPU passthrough (VFIO) on Intel Alder Lake (12th-gen)
systems and displaying the guest via [Looking Glass](https://looking-glass.io)
shared memory.

## What it covers

- The **Alder Lake iGPU reality**: 12th-gen UHD 770 has no GVT-g, but both
  **GVT-d** (full passthrough) and **SR-IOV** (virtual functions — share the
  iGPU across up to 7 VMs while the host keeps it) are viable. Code 43 is now
  well understood and fixable.
- An evidence-backed **Intel iGPU Code 43** root-causes deep dive — the real #1
  cause is Windows nested virtualization (VBS / Virtual Machine Platform), *not*
  the hypervisor hiding that fixes NVIDIA.
- The full proven pipeline: **VFIO** binding & IOMMU on Arch/CachyOS, the
  **KVMFR** shared-memory kernel module, **libvirt/QEMU** JSON-style IVSHMEM
  config, building the **Looking Glass client**, Windows guest setup, and the
  NVIDIA+Wayland flicker fix.
- Troubleshooting tables and emergency recovery (e.g. mounting a Windows QCOW2
  to undo a bad display change).

> Authored from a real install: CachyOS + Hyprland, i9-12900K, host NVIDIA RTX
> 3080 Ti, guest NVIDIA GTX 1080, Looking Glass B7, 3440×1440.

## Installation (Pi)

```bash
pi install ./intel-gpu-passthrough-skill
# or from a git repo:
pi install git:github.com/<you>/intel-gpu-passthrough-skill
```

Then invoke explicitly with `/skill:intel-gpu-passthrough-skill`, or let Pi
auto-trigger it from your request (it matches "gpu passthrough", "VFIO",
"looking glass", "ivshmem", "kvmfr", "GVT-d", "SR-IOV", "i915-sriov",
"Code 43", etc.).

## Layout

```
intel-gpu-passthrough-skill/
├── SKILL.md
├── README.md
└── references/
    ├── sr-iov-vgpu.md       # SR-IOV vGPU (share iGPU across VMs), libvirt/Arch
    ├── config-reference.md  # full XML, host files, architecture, Code 43 deep dive
    └── troubleshooting.md   # symptom/cause/fix + recovery
```
