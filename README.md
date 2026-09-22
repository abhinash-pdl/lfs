# LFS 13.0 (Systemd) Build Notes
> Personal build log — LFS 13.0 stable systemd version  
> Kernel 7.0 | Intel integrated GPU | XFS filesystem | Hyprland on Wayland

---


---

## System Specs
- **CPU/GPU:** Intel (integrated graphics)
- **Kernel:** Linux 7.0
- **Init system:** Systemd
- **Filesystem:** XFS (root + separate home) + EFI FAT32
- **Display server:** Wayland (Hyprland)
- **Login:** TTY (no display manager)

---

## Partition Layout
```
/boot/efi   → FAT32 (EFI)
/           → XFS
/home       → XFS (separate partition)
```

### Why XFS
- Better performance than ext4 for this use case
- Kernel 7.0 introduced **online autorepair** for XFS — filesystem can repair itself without unmounting
- Additional I/O improvements in 7.0 kernel made it worthwhile over btrfs

---

## Build Order Overview

### Phase 1 — Base LFS
Follow LFS 13.0 book strictly. No deviations here.

### Phase 2 — BLFS / Network First
After base LFS, first priority is getting network up since everything else depends on it.

**Initial network setup (in host kernel environment):**
```bash
# Used wpa_supplicant temporarily to get network going
# during the BLFS compilation phase
```
> **Note:** wpa_supplicant was removed later and replaced with NetworkManager once the full system was stable.

### Phase 3 — Core BLFS packages
In rough order:
1. PAM
2. Shadow (with PAM support — critical, see gotcha below)
3. Systemd recompile (critical, see gotcha below)
4. Mesa (OpenGL/Vulkan userspace drivers — critical, see gotcha below)
5. Wayland + dependencies
6. Wayland protocols
7. Xwayland
8. Pipewire
9. Qt Base (built **without** QtWebEngine — see gotcha below)
10. Fonts (basics)
11. NetworkManager (replaced wpa_supplicant)

### Phase 4 — Wayland Desktop
Following **SLFS (Supplemental Linux From Scratch)** book for Hyprland:
- Hyprland compiled from source per SLFS guide
- Waybar
- Wofi
- Foot terminal
- Swaync
- Xwayland (worked without issues)

---

## Critical Gotchas

### ⚠️ PAM → Shadow → Systemd Recompile Order
**This is the most important thing in this entire document.**
Your compiled Shadow without PAM support in LFS will not work since Systemd will not pick it up correctly and authentication will silently fail or behave unexpectedly.

**Correct order — do not deviate:**
```
1. Compile PAM
2. Recompile Shadow WITH PAM support enabled
3. Recompile Systemd after both are in place
```

If Hyprland or your login is failing for no obvious reason after a complete build, this is almost certainly why. Recompile in this order and it will fix it.

---

### ⚠️ iwlwifi Network Firmware (Intel WiFi)
**Problem:** iwlwifi firmware not present in new system on first boot.

**Step 1 — Identify which ucode your card needs:**
```bash
dmesg | grep iwl
```
This tells you the exact firmware file being requested. Example output:
```
iwlwifi-7265D-29.ucode
```
Do not guess — the filename must match exactly.

**Step 2 — Get the firmware:**
Copy from your host system:
```bash
ls /lib/firmware/iwlwifi-*
# copy the relevant .ucode file to $LFS/lib/firmware/
```
Or download from [linux-firmware](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git)

**Step 3 — Build firmware into kernel via menuconfig:**
```
Device Drivers
  → Generic Driver Options
    → Firmware loader base path: /lib/firmware
    → Build named firmware blobs into the kernel binary:
       iwlwifi-7265D-29.ucode
```
Space-separate multiple files if needed. Then reconfigure and recompile kernel.

**Step 4 — Kernel config flags:**
```
CONFIG_IWLWIFI=y
CONFIG_IWLDVM=y   # or CONFIG_IWLMVM depending on your card
```

> **Why builtin over module:** As a module it requires initramfs to load firmware early enough. As builtin (`=y`) the driver is always available without initramfs complexity. Simpler and more reliable on LFS.

**Could've also gone the module route.** `CONFIG_IWLWIFI=m` works fine too, and you skip baking the firmware into the kernel binary entirely. Only catch is the firmware then needs to sit at `/lib/firmware/` on the actual root filesystem, and something has to load it early enough at boot — which means getting an initramfs (mkinitcpio, dracut, whatever) set up properly. Went with builtin purely to dodge dealing with initramfs on top of everything else. Module isn't wrong, just more moving parts.

---

### ⚠️ Don't Forget Mesa, or You'll Get Software Rendering and Not Know Why
Wayland and Hyprland will both compile fine and even run without Mesa in place — which is exactly what makes this annoying. You just end up with software rendering, or Hyprland refusing to give you a proper hardware-accelerated session, and nothing in the build log tells you that's the problem.

The kernel's `i915` driver only gets you hardware detection and mode-setting. The actual OpenGL/Vulkan/EGL implementation that Wayland compositors link against comes from Mesa, in userspace. Kernel driver ≠ graphics driver, basically.

**Compile Mesa before Wayland/Hyprland.** Hyprland's build looks for EGL/GBM at configure time, so if Mesa isn't there yet it'll either fail or silently fall back to something worse.

**What I used for the Intel iGPU:**
```
-Dgallium-drivers=iris        # modern Intel Gallium driver
-Dvulkan-drivers=intel        # ANV Vulkan driver
-Dplatforms=wayland,x11       # keep x11 too — Xwayland still needs it
-Degl=enabled
-Dgbm=enabled
```
> `iris` over `i965`: `i965` is the old classic driver, `iris` is the one actually maintained for newer Intel iGPUs and what Wayland/DRI3 expects these days. Don't bother with the legacy one.

---

### ⚠️ Qt Base — Skip QtWebEngine
**Problem:** Some of the Wayland ecosystem/portal tooling links against Qt Base, but QtWebEngine (Qt's bundled Chromium) is not needed for a lightweight Hyprland setup and adds a massive amount of build time and disk space (it vendors and compiles its own Chromium).

**Build Qt Base with WebEngine explicitly excluded:**
```bash
./configure -skip qtwebengine \
            -no-feature-sql \
            -opensource -confirm-license
```
Or, if building modules individually rather than the full src tree, simply don't build the `qtwebengine` module at all — it's a separate repo/tarball and can just be left out.

> **Why bother excluding it:** QtWebEngine alone can take longer to compile than the rest of the Qt stack combined, and unless something in your stack specifically needs an embedded Chromium view, it's dead weight on an LFS build where every compile is already manual and slow.

---

## Kernel Configuration

**Approach:** Started with a hardware-specific base config, then used `make menuconfig` for manual adjustments.

**Key areas configured manually:**
- iwlwifi builtin (see above)
- Intel integrated graphics (i915)
- XFS filesystem support + online autorepair (kernel 7.0 feature)
- Wayland/DRM requirements for Hyprland
- Binder/binderfs (for future Waydroid)

**XFS autorepair kernel flag (7.0+):**
```
CONFIG_XFS_ONLINE_SCRUB=y
CONFIG_XFS_ONLINE_REPAIR=y
```

**Intel GPU:**
```
CONFIG_DRM_I915=y
CONFIG_DRM=y
CONFIG_FB=y
```

**Wayland/Hyprland minimum requirements:**
```
CONFIG_DRM=y
CONFIG_DRM_KMS_HELPER=y
CONFIG_PACKET=y
CONFIG_INPUT_EVDEV=y
CONFIG_INPUT_UINPUT=y
```

---

## Why the First Build Was Scrapped

First build accumulated too many fixes over a week without documentation. System worked but the mental model of what was installed and why was lost. Rather than maintain a system that couldn't be reasoned about, it was rebuilt from scratch.

**Lesson:** On LFS, an undocumented working system and the system with user having no idea how it worked is already broken.

Second build was faster, cleaner, and every fix was understood.

---

## Current Userspace Stack

| Purpose | Tool |
|---|---|
| Compositor | Hyprland |
| Bar | Waybar |
| Launcher | Wofi |
| Notifications | Swaync |
| Terminal | Foot |
| Network | NetworkManager |
| Audio | Pipewire |
| Login | TTY (no DM) |
| Filesystem | XFS |
| Graphics drivers | Mesa (iris/ANV) |

---

## Things Still TODO
- [ ] Waydroid setup (kernel binder config, ARM translation via libndk)
- [ ] Document exact Hyprland SLFS build steps
- [ ] Kernel config file export
- [ ] Exact BLFS package list with versions

---
<img width="5600" height="5713" alt="lfs" src="https://github.com/user-attachments/assets/65f171a7-1189-49a1-b5c6-d6ff0293e10b" />

*How everything in this build actually depends on everything else, mapped out so I stop forgetting the order. Solid arrows mean "needs this first," dotted ones are looser — more like "this made that possible" than a hard requirement.*

---

## Notes
- This document is incomplete by nature — built from memory after the fact
- Gaps will be filled as things are encountered again
- SLFS book was used for Hyprland specific compilation steps
