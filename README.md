# reclock-nv50 — GeForce 9600 GT (G94 / NV50·Tesla) on modern CachyOS

> 🇷🇺 Русская версия: [README.ru.md](README.ru.md)

Getting an old **GeForce 9600 GT (G94)** running on a current CachyOS (kernel 7.x,
clang/LTO). **The main path is the proprietary NVIDIA 340.108 driver via DKMS.** The
attempt to unlock full clocks through nouveau-reclocking turned out to be a dead end
and was moved to `nouveau-attic/` (for reference).

Target is one specific machine: **mlamp**, i3-2120, 9600 GT (`10de:0622`), CachyOS,
kernel `7.1.x-cachyos` (clang+ThinLTO). Values are hardcoded on purpose; nothing here
is meant to be generic.

## Quick start

```
userspace/install-cachyos.sh      # installs 340.108: utils (AUR) + patched DKMS module
sudo reboot                       # pick the Plasma (X11) session
```

After reboot:
```
nvidia-smi                          # driver 340.108
lspci -k | grep -A3 -Ei 'vga|3d'    # Kernel driver in use: nvidia
```

Switch between drivers at any time:
```
userspace/nv-switch.sh status       # what's installed / loaded right now
userspace/nv-switch.sh nvidia       # to proprietary 340.108 (blacklists nouveau)
userspace/nv-switch.sh nouveau      # back to nouveau
# then sudo reboot
```

## Why a vendored package and not AUR

- Bare AUR `nvidia-340xx-dkms` is **stale 340.76 (Linux 4.0)** — it does NOT build.
- The live `nvidia-340xx` (340.108-39) only patches up to kernel 6.15.
- So the driver is vendored in `proprietary-340xx/`: PKGBUILD + patches `0001-0019` (AUR)
  **+ our `0020-kernel-7.0-7.1.patch`** for kernels 7.0/7.1 and clang/LTO. See
  `proprietary-340xx/README.md`.

`0020` fixes: conftest (`-fms-extensions` — otherwise `static_assert` in `linux/fs.h`
fails every probe), `in_irq()`→`in_hardirq()`, the removed `screen_info` global
(conftest fallback), and `dkms.conf` (build via `SYSSRC=` + auto-detect clang →
`CC=clang LLVM=1`).

**Build-verified:** 7.0 gcc (7.0.13-zen) and 7.1 clang (7.1.0-cachyos), plus a
**DKMS install on mlamp itself** (7.1-rc6 clang): `dkms status … installed`.

## Hard hardware ceilings (chip physics, not bugs)

- No hardware Vulkan on Tesla → OpenGL only. DXVK/VKD3D/gamescope are impossible;
  gaming goes through WineD3D (OpenGL), DXVK is disabled on purpose.
- The proprietary 340.108 is **Xorg/X11 only**, no Wayland. Log in to Plasma (X11).
- Secure Boot must be off (unsigned module) or the module signed via MOK.

## Layout

| Path | What |
|---|---|
| `proprietary-340xx/` | 340.108 DKMS vendor package (PKGBUILD + patches 0001-0020) — **the main thing** |
| `userspace/install-cachyos.sh` | install the proprietary path |
| `userspace/nv-switch.sh` | nvidia ↔ nouveau switcher (both directions) |
| `userspace/optimize-system-cachyos.sh` | tuning for the weak CPU / 8 GB RAM |
| `userspace/setup-gaming-9600gt.sh` | WineD3D gaming (DX9/DX10) |
| `userspace/{diagnose,fix}-display-cachyos.sh` | diagnose / recover graphical login |
| `userspace/nv9600gt.py` | control panel (GUI/TUI) |
| `nouveau-attic/` | **archive**: the old nouveau-reclocking (patches, src, docs, traces, scripts) |

## Safety

- Switching drivers and building the module are safe. The live nouveau-reclocking
  (writing pstate to hardware) lives only in `nouveau-attic/` and requires a
  confirmation phrase.
- Don't commit build artifacts: `proprietary-340xx/{src,pkg,*.run,*.pkg.tar.*}` — in `.gitignore`.
