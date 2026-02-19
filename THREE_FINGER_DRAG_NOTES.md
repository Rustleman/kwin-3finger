# KWin 3-Finger Drag Work Log

## Goal
Add libinput 3-finger drag support to this AUR package (`kwin-3finger`) and make it usable in KWin/Wayland.

## Repository Context
- This repo is packaging-only (`PKGBUILD`, `kwin.install`), not full KWin source.
- Upstream KWin source is downloaded/extracted by `makepkg`.

## What Was Found First
1. Existing package already had a custom `prepare()` hack:
- `sed` in `src/plugins/overview/overvieweffect.cpp`
- It duplicated touchpad 4-finger overview gestures to 3-finger.
- This was called the “magic fix”.

2. KWin/libinput source locations used:
- `src/backends/libinput/device.cpp`
- `src/backends/libinput/device.h`
- `src/virtualdesktops.cpp`
- `src/plugins/overview/overvieweffect.cpp`

3. libinput API available on system headers (`/usr/include/libinput.h`):
- `libinput_device_config_3fg_drag_get_finger_count`
- `libinput_device_config_3fg_drag_set_enabled`
- enum values:
  - `LIBINPUT_CONFIG_3FG_DRAG_DISABLED`
  - `LIBINPUT_CONFIG_3FG_DRAG_ENABLED_3FG`
  - `LIBINPUT_CONFIG_3FG_DRAG_ENABLED_4FG`

## Build/Environment Issues Encountered
1. Initial `makepkg -o` failed due missing makedepends in environment.
- Worked around for source extraction with `--nodeps`.

2. Initial source download failed due DNS/network restrictions.
- Later succeeded after network access became available.

3. PGP verification failed (missing key in environment).
- Worked around with `--skippgpcheck` for local iteration.

4. One patch file became malformed during manual edits.
- Regenerated patch from clean upstream snapshots with `diff -u`.

## Changes Implemented (Chronological)

### A) Added proper patch-based packaging flow
- Added `0001-libinput-enable-3fg-drag-always.patch` to `source=()`.
- Added checksum in `sha256sums`.
- Applied patch in `prepare()` via `patch -Np1 -i ...`.
- Replaced fragile direct source mutation approach.

### B) Added libinput-side always-on 3fg drag in KWin
Patch hunk in `src/backends/libinput/device.cpp`:
- Added helper in anonymous namespace:
  - `enableThreeFingerDrag(libinput_device *device)`
- Compile-time guard:
  - `#ifdef LIBINPUT_CONFIG_3FG_DRAG_ENABLED_3FG`
- Runtime logic:
  - Check `libinput_device_config_3fg_drag_get_finger_count(device)`
  - Enable 3fg drag when `fingerCount >= 3`:
    - `libinput_device_config_3fg_drag_set_enabled(device, LIBINPUT_CONFIG_3FG_DRAG_ENABLED_3FG)`
- Called during device init in constructor:
  - right after DBus object registration in `Device::Device(...)`.

### C) Removed old “magic fix” in PKGBUILD (per user request)
- Deleted the `sed` block that modified `overvieweffect.cpp`.
- Kept only patch application in `prepare()`.

### D) Runtime verification findings
After install and relogin:
- Installed package confirmed: `kwin-3finger`.
- Running compositor confirmed: `kwin_wayland` in Wayland session.

Observed behavior reported by user:
- 3-finger up/down still switched virtual desktops.
- 3-finger drag did not drag windows.

### E) Important correction
The initial libinput patch version preferred `ENABLED_4FG` when available, which could preserve 3-finger swipe behavior.
- This was identified as wrong for requested behavior.
- Changed to force `ENABLED_3FG` mode.

### F) Additional root cause found and fixed
KWin still explicitly registered 3-finger touchpad swipes for virtual desktop switching in `src/virtualdesktops.cpp`:
- Removed these touchpad 3-finger registrations:
  - Left/Right 3-finger touchpad swipe
  - Up/Down 3-finger touchpad swipe
- Kept:
  - 4-finger touchpad desktop switching
  - 3-finger touchscreen swipe shortcuts

This prevents KWin’s own touchpad 3-finger desktop-switch gesture bindings from conflicting with 3-finger drag.

## Final Patch Contents
`0001-libinput-enable-3fg-drag-always.patch` now contains two file hunks:
1. `src/backends/libinput/device.cpp`
- enable 3fg drag on libinput devices (3FG mode)

2. `src/virtualdesktops.cpp`
- remove touchpad 3-finger desktop-swipe registrations
- keep touchpad 4-finger + touchscreen gestures

## Package Metadata Changes
`PKGBUILD` was updated multiple times during iteration. Final effective state:
- `pkgrel` incremented to `11`
- `source` includes `0001-libinput-enable-3fg-drag-always.patch`
- `sha256sums` includes current patch checksum
- `prepare()` applies patch and no longer includes old `sed` magic fix

## Validation Performed
Repeatedly validated with:
- `makepkg -o -f --nodeps --skippgpcheck`

Successful final `prepare()` output showed:
- patch applied to `src/backends/libinput/device.cpp`
- patch applied to `src/virtualdesktops.cpp`

## Build/Install Command Used
For local rebuild with multicore and skipped PGP checks:

```bash
MAKEFLAGS="-j$(nproc)" makepkg -si --skippgpcheck
```

## Known Caveats
- `--skippgpcheck` was used due local keyring state, not ideal long-term.
- Actual gesture behavior can vary by touchpad firmware/libinput stack, but code now removes the main KWin-side 3-finger swipe conflicts and forces libinput 3FG drag mode.

## Files Modified In This Repo
- `PKGBUILD`
- `0001-libinput-enable-3fg-drag-always.patch`
- `THREE_FINGER_DRAG_NOTES.md` (this file)
