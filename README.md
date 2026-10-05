# FreeSync HDR for Hyprland (personal patch set)

Ports KWin MR `plasma/kwin!8423` ("use the FreeSync 2 HDR mode of the display")
to the Hyprland stack: native primaries + gamma 2.2 + `eotf=0` metadata instead
of BT2020 + PQ, when HDR is active on a capable display.

## Patches

1. `kernel-patches-7.3-rc6/0001..0003` — rebased kernel UAPI (base: `v7.3-rc6`).
   Apply with `git am`. Rebuild + boot, then check
   `drm_info | grep -i "freesync hdr"` on your DP output.
2. `aquamarine-freesync-hdr.patch` — DRM prop plumbing (base: aquamarine `52ea11f`).
   New `FreeSync HDR Mode` connector prop, `freeSyncHDR` output state, capability flag.
3. `hyprland-freesync-hdr.patch` — compositor side (base: Hyprland `19fb395`).
   Must be built against the patched aquamarine (`freeSyncHDRCapable` field).

Build order: kernel → aquamarine → hyprland.

## Arch packages (`pkgbuilds/`)

Pre-written PKGBUILDs so you can `makepkg -s` instead of hand-rolling:

- `pkgbuilds/linux-freesync-hdr/` → `linux-freesync-hdr` + `linux-freesync-hdr-headers`.
  Builds the `v7.3-rc6` tarball with the 3 patches. Reuses your running
  kernel's `/proc/config.gz` when present (plus forced `DRM_AMDGPU`), else
  `defconfig`. Coexists with stock `linux` — pick it in your bootloader to test.
  Run `updpkgsums` in the dir first if you want the tarball hash pinned.
- `pkgbuilds/aquamarine-freesync/` → `aquamarine-freesync` (provides/conflicts
  `aquamarine`). Pinned to aquamarine `52ea11f`. Install before Hyprland.
- `pkgbuilds/hyprland-freesync/` → `hyprland-freesync` (provides/conflicts
  `hyprland`). Pinned to Hyprland `19fb395`, needs `aquamarine-freesync`
  at build time. Toggle at runtime with `render:freesync_hdr = 0/1` (default 1).

```sh
cd pkgbuilds/linux-freesync-hdr && makepkg -s   # then install both packages
sudo pacman -U linux-freesync-hdr-*.pkg.tar.zst linux-freesync-hdr-headers-*.pkg.tar.zst
# reboot into it, verify with drm_info, then:
cd ../aquamarine-freesync && makepkg -s && sudo pacman -U aquamarine-freesync-*.pkg.tar.zst
cd ../hyprland-freesync && makepkg -s && sudo pacman -U hyprland-freesync-*.pkg.tar.zst
```

Untested builds (written without a compiler/makepkg on hand) — if `makepkg -s`
complains about a dependency name or a build step, paste the error and it gets fixed.

## Behavior

- Tied to HDR like KWin: engages automatically when the monitor is in HDR mode
  (`cm_type` hdr/hdredid or auto-HDR) **and** the connector exposes the
  `FreeSync 2 Native` enum. Kill switch: `render:freesync_hdr = 0` (default 1).
- Direct-scanout HDR metadata is bypassed while engaged (surface speaks PQ,
  signal must be gamma22); the linear FP16 workbuffer is kept so highlights survive.
- Stock kernels are unaffected: without the connector property the capability
  is false and everything behaves as before.

## Caveats

- Nothing here is compile-tested (no toolchain where it was written). The
  riskiest identifiers: `ADAPTIVE_SYNC_TYPE_DP` (kernel, assumed present as in
  the original series) and `(1 << 16)` state bit handling.
- KWin notes you may need to re-do HDR calibration (his panel went dark) and
  absolute brightness may still be off (his emits ~75% of nominal).
- If your monitor's AMD VSDB lives only in a DisplayID block (not CEA),
  kernel patch 2/3 won't find it — check `drm_info --edid` / driver logs.
