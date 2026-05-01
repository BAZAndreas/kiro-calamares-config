# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Calamares installer configuration for the Kiro Linux distribution (Arch-based). Contains the full installation pipeline: module configs, custom Python extension modules, branding/QML slideshow, bundled microcode packages, and a custom Calamares PKGBUILD.

Part of the broader Kiro ecosystem:
- **kiro-pkgbuild** — upstream Calamares fork source (copied here by `up.sh`)
- **kiro-iso** — ISO build scripts (sibling repo)
- **kiro-repo** — custom pacman repository at `https://kirodubes.github.io/$repo/$arch`

## Common Commands

```bash
# Deploy: update PKGBUILD from kiro-pkgbuild, check microcode versions, git commit+push
./up.sh

# Build Calamares (inside pkgbuild dir):
cd etc/calamares/pkgbuild && makepkg -si

# Run tests for the packages module:
cd etc/calamares/pkgbuild/modules/packages/tests && python -m pytest
```

There is no linter configured for this repo. The Python modules in `usr/lib/calamares/modules/` are run by Calamares at install time inside a chroot — they cannot be run standalone.

## Installer Pipeline Architecture

**Entry point:** [etc/calamares/settings.conf](etc/calamares/settings.conf) — defines the full execution sequence.

**Show phase** (UI pages shown to user):
`welcome → locale → keyboard → partition → users → summary`

**Exec phase** (automated steps, in order):
`partition → mount → unpackfs (rootfs, kernel) → packages → locale → keyboard → fstab → kiro_before → initcpio → services → kiro_remove_nvidia → kiro_ucode → bootloader → kiro_final → removeuser → umount`

**Finish phase:** `finished` page.

## Custom Python Modules

All four live in [usr/lib/calamares/modules/](usr/lib/calamares/modules/). Each has a `main.py` and `module.desc`. They run inside the target chroot via `libcalamares.utils.host_env_process_output()`.

**Return convention:** all functions return `None` on success or a `(error_title, error_description)` tuple on failure. The `run()` entrypoint aggregates these.

| Module | When | Purpose |
|---|---|---|
| `kiro_before` | Before initcpio | Pacman lock wait, makepkg optimization, keyring init, mkinitcpio preset rename |
| `kiro_remove_nvidia` | Conditional | Reads `driver=free` kernel param; if set, removes NVIDIA packages |
| `kiro_ucode` | After NVIDIA step | Detects CPU (AMD/Intel via hwinfo), installs bundled `.pkg.tar.zst` from `/etc/calamares/packages/` |
| `kiro_final` | Last exec step | File permissions, skel copy, cleanup of live-only files, VM detection + package removal, systemd-boot detection + GRUB removal |

### VM Detection (kiro_final)
Uses `systemd-detect-virt` output to selectively remove guest agents:
- `vmware` → removes open-vm-tools, xf86-video-vmware
- `oracle` → removes virtualbox-guest-utils/nox
- `qemu`/`kvm` → removes qemu-guest-agent

## Module Configs

All in [etc/calamares/modules/](etc/calamares/modules/). Key non-obvious settings:

- **partition.conf** — EFI min 2GB, swap as file (no partition), LUKS v1, auto-partitioning disabled
- **unpackfs1.conf / unpackfs2.conf** — two separate unpack steps (rootfs + kernel)
- **packages.conf** — removes calamares, mkinitcpio-archiso, memtest86+ after install

## Branding

[etc/calamares/branding/kiro/](etc/calamares/branding/kiro/) — dark theme, 1200×800 window, sidebar widget layout.

**Slideshow:** `show.qml` (QML API v2) cycles through `01cal.jpg`–`12cal.jpg`. To add/remove slides, edit both the QML and the image files.

**Translations:** Qt `.ts` format in `lang/` — en, fr, nl, ar, eo.

## Microcode Bundling

Pre-downloaded `.pkg.tar.zst` files in [etc/calamares/packages/](etc/calamares/packages/) allow offline microcode installation. `up.sh` checks for newer versions using `vercmp` and downloads via `pacman -Sw --cachedir` if an update exists. The `kiro_ucode` module installs them with `pacman -U`.

When adding a new microcode version, the old file can be removed — `kiro_ucode` uses `glob` to find the latest matching package.

## PKGBUILD Notes

[etc/calamares/pkgbuild/PKGBUILD](etc/calamares/pkgbuild/PKGBUILD) tracks a git snapshot of a Calamares fork on Codeberg. It:
- conflicts with `calamares` and `calamares-git`
- provides `calamares-next`
- applies two patches: enables config file installation, increases fstab `desired_size` to 8589MiB
- bundles the custom modules from `pkgbuild/modules/` (bootloader + packages overrides)

`up.sh` copies the PKGBUILD from `~/KIRO/kiro-pkgbuild/` — do not edit the local copy directly; edit the source repo instead.
