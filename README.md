# Denial on the Pixel 7 Pro (cheetah) — findings and fixes

Practical notes from getting Denial's mobile shell running well on a Google
Pixel 7 Pro with the Omarchy Mobile / Omamobile stack. Everything here was
verified on the device described below; commands assume root over SSH unless
noted.

Last updated: 2026-11-24

---

## 1. Device and software baseline

| Item | Value |
|---|---|
| Device | Pixel 7 Pro (`cheetah`), Google Tensor G2 (`gs201`) |
| GPU | Mali-G710 MC7, kernel driver **panthor** (CSF), Mesa 26.2.4 |
| Kernel | `7.3.0-rc6-omamobile-gda171eae4cb7` |
| Denial | dev branch `28838fbb`, package `0.2.1.r521.g28838fbb` |
| Denial renderer | Impeller **Vulkan** (PanVK) + Vulkan scene composer |
| Display | DSI-1, 1440×3120 @ 120 Hz (native mode) |
| Session | SDDM autologin → `denial-session` → `deniald` |
| Sensor stack | `omamobile-sensor-proxy` (net.hadess.SensorProxy) → `omamobile-steinerd` bridge → Denial |

Denial is pinned to the dev build so package upgrades cannot replace it with
the 0.6.8 release:

```ini
# /etc/pacman.conf
IgnorePkg = denial denial-flutter-engine denial-plugin-manager
```

### Session environment that matters

The phone needs these in `/etc/denial/session.conf` (sourced by
`denial-session` before `deniald` starts):

```sh
# The launcher defaults to the desktop shell when this is unset.
export DENIAL_SHELL_PROFILE=mobile

# Mesa opens EGL on the render node; without this Denial compares it with the
# KMS device (card2) and aborts with a fatal device mismatch.
DENIAL_RENDER_DEVICE=/dev/dri/renderD128

# sddm can start the session before pam_systemd exports one.
export XDG_RUNTIME_DIR="${XDG_RUNTIME_DIR:-/run/user/$(id -u)}"
```

---

## 2. Cameras: all four sensors now stream

### Symptom

`cam -l` found nothing, `omacam --list` was empty, the media graph only showed
`gs201-csis`, and `/dev/v4l-subdev*` was missing. Driver probes existed for
`s5kgn1`, `s5k3j1`, `imx386`, `s5kgm5` but the V4L2 async graph never
completed.

### Root cause

The device tree gives each sensor a `firmware-name`:

```
omamobile/camera/s5kgn1-main.bin
omamobile/camera/s5k3j1-front.bin
omamobile/camera/imx386-uw.bin
omamobile/camera/s5kgm5-tele.bin
omamobile/camera/gs201-dcphy.bin
```

Those are **not vendor blobs**. They are sensor initialisation and readout-mode
register tables that the stock Pixel camera HAL builds at runtime in
`liblyric_hwl.so` (inside `com.google.pixel.camera.hal.apex`). `gs201-dcphy.bin`
is mandatory: without it the D-PHY/C-PHY refuses power-on, so every probe dies.

### Procedure (one-off, ~4 GB download)

```sh
# 1. Stock factory image for the phone model (verified build)
#    cheetah CP2A.260705.006
#    sha256 23d564ad5d0db2d04fa4bbde43930c9b0f8e570a51e8090c8d37c71dc96a2c0c
curl -L -o /var/tmp/cheetah/factory.zip \
  https://dl.google.com/dl/android/aosp/cheetah-cp2a.260705.006-factory-23d564ad.zip

# 2. Extract vendor.img, mount read-only, take the camera HAL APEX
unzip -j factory.zip 'cheetah-*/image-*.zip' -d /var/tmp/cheetah
unzip -o image-cheetah-*.zip vendor.img
mount -o loop,ro vendor.img /mnt/vendor
cp /mnt/vendor/apex/com.google.pixel.camera.hal.apex /var/tmp/cheetah/

# 3. Extract the register tables on the phone
mkdir -p /usr/lib/firmware/omamobile/camera
python3 extract-camera-data --apex /var/tmp/cheetah/com.google.pixel.camera.hal.apex \
  -d /usr/lib/firmware/omamobile/camera
```

`extract-camera-data` ships with the Omamobile project (`tools/extract-camera-data`).

### Pitfall: Unicorn SVE assertion

With Unicorn 2.1.4 the tool aborts with:

```
sve_zcr_get_valid_len: Assertion `end_len < start_len' failed.
```

The tool enables SVE in `CPACR_EL1.ZEN`; Unicorn then asserts on
`UC_CPU_ARM64_MAX`. Remove the ZEN bit so the emulator does not advertise SVE
(the HAL does not need it):

```diff
- self.uc.reg_write(A.UC_ARM64_REG_CPACR_EL1, cpacr | (3 << 16) | (3 << 20))
+ self.uc.reg_write(A.UC_ARM64_REG_CPACR_EL1, cpacr | (3 << 20))
```

Then rerun. Output was, for example:

```
wrote .../imx386-uw.bin: 4 modes
wrote .../gs201-dcphy.bin: D-PHY 32 writes, C-PHY 20 writes
wrote .../s5kgn1-main.bin: 4 modes
wrote .../s5k3j1-front.bin: 5 modes
wrote .../s5kgm5-tele.bin: 4 modes
```

### Result and durability

- `omacam --list` reports front 1.00×, main 1.00×, ultrawide 0.45×, tele 4.78×.
- `cam -l` lists four libcamera cameras; stills capture through both `cam` and
  `omacam`.
- Drivers load from `omamobile-camera.timer` (`OnBootSec=40s`). **Enable it** —
  the service is static and the timer was disabled on this install:

```sh
systemctl enable --now omamobile-camera.timer
```

- The files are unowned by pacman, so package upgrades will not remove them.
  A rootfs rebuild will. Re-extract from your own phone firmware; **do not
  redistribute the extracted files** — they are derived from Google's HAL for
  your device.

---

## 3. Auto-rotate: hysteresis instead of a hair trigger

Chain: `AccelerometerOrientation` from `net.hadess.SensorProxy` →
`omamobile-steinerd` (the `org.steinerd.Sensors1` bridge Denial expects) →
`deniald`.

The stock iio-sensor-proxy rule entered landscape at ~55° but **left** it again
at 35°, so the 35–55° band was unstable. The bridge added only 350 ms of
stability.

Fix, in two places:

1. `omamobile-sensor-proxy` replaced by a wrapper that swaps only the
   orientation function for a stateful rule:

   - enter landscape from portrait: **55°**
   - leave landscape for portrait: **25°**
   - diagonal tilt never forces a change

   Installed as `/usr/local/lib/omamobile/sensor_proxy_hysteresis.py`, with a
   systemd drop-in for `omamobile-sensor-proxy.service` pointing `ExecStart` at
   it.

2. `omamobile-steinerd` timings: stable **500 ms**, cooldown **800 ms**.

Both read one tunable file, `/etc/omamobile/rotation.conf`:

```ini
enter = 55
exit = 25
stable_ms = 500
cooldown_ms = 800
```

Apply changes with:

```sh
systemctl restart omamobile-sensor-proxy.service omamobile-steinerd.service
```

Verified with a synthetic sweep: enters at 56°, returns at 25°, and a phone
lying flat never changes orientation.

---

## 4. GPU: what the Mali-G710 can actually do here

### Findings

- The shell already runs Impeller **Vulkan** (PanVK) with the Vulkan scene
  composer. GLES is used only for auxiliary contexts (screencopy/transfer),
  and those fall back to **GLES 3.0** because a 3.2 context cannot be created.
- Benchmarks at 720×1280 (both far beyond 120 fps):

  | Stack | Score |
  |---|---|
  | glmark2 (GLES/Panfrost) | 705 |
  | vkmark (Vulkan/PanVK) | 1014 |

  So shell jank is engine/compositor overhead, not raw fill rate.

- **The GPU clamps at 572 MHz.** `available_frequencies` lists up to 848 MHz,
  but writes above 572 are silently clamped. Panthor probes with:

  ```
  panthor 28000000.gpu: error -ENODEV: _opp_set_regulators: no regulator (mali) found
  ```

  The GPU DT node has **no `mali-supply`**, and the OPP table has **no
  `opp-microvolt` entries at all**, so voltage scaling never initialises and
  the high OPPs are unreachable. This is a DTB/kernel fix: add a GPU regulator
  and voltage OPPs. It cannot be fixed at runtime on this image (no `/boot`,
  no configfs DT overlays).

- `cur_freq` is unreliable while boosted: it can report 572 while
  `trans_stat` shows time spent at 848. Trust `trans_stat` for frequency
  accounting.

### Fix: interaction floor for Panthor

Denial's built-in GPU floor only recognises a devfreq device whose driver is
`adreno`, so on Panthor it is silently inert. `omamobile-gpu-boost.service`
replaces it:

- watches `/dev/input/event*` for the touchscreen
- on touch, raises `min_freq` to `max_freq` for **2500 ms**
- restores the **lowest** operating point afterwards
- heals a floor left raised by a previous restart (`848 → 151`)

Install at `/usr/local/lib/omamobile/gpu-boost.py` with a systemd unit of the
same name (`Restart=always`, `Nice=-5`). Idle is 151 MHz; touch takes it to
848 (kernel-clamped to 572 during voltage-less operation).

---

## 5. Shell performance fixes

All shell changes were done in the plugin **build kit** (the runtime copy that
`denial-plugins apply` compiles) and applied with:

```sh
denial-plugins apply      # rebuilds builtins and restarts the shell
denial-plugins undo       # reverts
denialctl ui status       # shows active mode/generation
```

Build kit path (machine-specific hash):

```
~/.local/state/denial/plugins/build-kits/<hash>/runtime/plugins/denial_desktop/lib/src/...
```

Keep a `.bak-*` beside each edited file. A build-kit re-extract loses these
patches.

### 5.1 App drawer: settle time follows distance and flick

The drawer always ran a full 320 ms animation from wherever the finger lifted,
so releasing at 90 % open spent a third of a second crawling through the last
10 %. `home_surface.dart` now scales the animation:

- base duration: 320 ms
- distance scale: `(remaining * 1.6).clamp(0.18, 1.0)`
- speed scale: `1 - 0.4 * (velocity / 2400).clamp(0, 1)`
- curve: `Curves.easeOutCubic`

Effect: a normal drag still eases for ~320 ms; a flick or a near-open release
settles in ~60–170 ms with no perceptible tail.

### 5.2 Drawer glass: slice a captured frost, and keep it planted

The glass drawer used `ShellBackdropBlur`, i.e. a fresh full-screen Gaussian
**every frame**. At 1440×3120 that is the single most expensive thing on the
screen and it dropped frames during the open animation.

The shell already captures the wallpaper once as a `FrostedBackdrop`
(`mobileWallpaperGlassProvider`) and hands it to glass surfaces through
`WindowBackdropCache`; `WindowBackdropSlice` draws a region of it. The drawer
was not using it.

Two changes in `home_app_drawer.dart`:

1. `_DrawerGlass` uses `WindowBackdropSlice` when a cache with a backdrop is in
   scope, and falls back to the old live blur otherwise.
2. The frost is **planted in screen space**. The sheet slides over it; the
   frost is clipped to the revealed area by `_BottomRevealClipper`
   (`Rect.fromLTWH(0, h*(1-openness), w, h*openness)`). The previous version
   translated the frost with the sheet, which dragged the whole background and
   looked stretched.

Result: one image sample per pixel instead of a per-frame Gaussian, with the
drawer's look preserved. The frost re-captures automatically when the wallpaper
repaints (image, zoom, alignment) or the theme changes.

### 5.3 Already-cached surfaces

- Quick settings / control centre: `_frostedShadeBackdrop` + `WindowBackdropSlice`
  (full-screen, planted) when panel opacity < 1.
- Notification cards in the shade: `NotificationCard` slices the shared frost
  (`cachedBackdrop`).
- Home folders: `home_folder_panel._Frost` uses `WindowBackdropCache` when one
  is in scope.

If a glass surface is not inside a `WindowBackdropCache`, it still live-blurs.
On the mobile shell that applies to smaller surfaces only: notification banner,
volume/HUD, power menu, Wi-Fi/Bluetooth detail sheets, edge panels, inactive
quick-settings tiles.

---

## 6. Boot robustness (these caused real black-screen incidents)

### 6.1 Panel connector stuck at `unknown`

After a soft reboot the DSI connector can report `unknown` with **no modes**.
Denial's launcher waits for a connected output, so the session never starts.

Fix: probe unknown connectors before the display manager starts.

`/usr/local/bin/omamobile-display-probe` writes `detect` to every
`/sys/class/drm/card*-*/status` that reports `unknown`, up to 20 s, and exits
when an output is connected.

```ini
# /etc/systemd/system/sddm.service.d/50-display-probe.conf
[Service]
ExecStartPre=/usr/local/bin/omamobile-display-probe
```

### 6.2 `droidloom-rkp-fix.service` blocked boot

The RKP/keystore relaxation loop waits for Android `sys.boot_completed=1` for
up to 360 s and was ordered into `multi-user.target`, so graphical boot waited
on it (splash forever). Move it off the boot path:

```sh
systemctl disable droidloom-rkp-fix.service
# droidloom-rkp-fix.timer: OnBootSec=120s, WantedBy=timers.target
systemctl enable --now droidloom-rkp-fix.timer
```

### 6.3 `XDG_RUNTIME_DIR` race

`deniald` aborts with `XDG_RUNTIME_DIR is required for the Xcursor sentinel
theme` when the session starts before `pam_systemd` exported one. The export in
`/etc/denial/session.conf` (section 1) is the fix.

### 6.4 Restarting SDDM mid-session pauses the seat

`systemctl restart sddm` while a session is live can leave logind with an
`Active=no` session in `closing`, and Denial pauses libinput/KMS — the lock
screen looks frozen and ignores touch. **Reboot instead of restarting SDDM**
while a session is running.

### 6.5 Recovery checklist when the screen is black but SSH works

```sh
pgrep -a deniald
cat /sys/class/drm/card*-DSI-1/status
cat /sys/class/drm/card*-DSI-1/modes
tail -50 /home/<user>/.local/share/sddm/wayland-session.log
journalctl -b -u sddm --no-pager | tail -30
```

- `unknown` → run `omamobile-display-probe` or `echo detect > .../status`
- `fatal error: ... no mode compatible` → the saved mode in
  `~/.local/state/denial/outputs.conf` is not offered by the panel; set
  `mode=DSI-1,1440,3120,120000` and `scale=DSI-1,3.0` (or 2.5)
- `EGL selected ... instead of requested scene GPU` → set
  `DENIAL_RENDER_DEVICE=/dev/dri/renderD128`
- desktop layout instead of mobile → `DENIAL_SHELL_PROFILE=mobile`

---

## 7. Games / Steam / NTE status

- **Geometry Dash** runs.
- **Clash Royale** reaches ~70–79 % then self-exits (`foremk.aB: 16`, a
  deliberate error, not a crash).
- **NTE** dies in `libUnreal.so` static init (allocator path → `pause()`),
  and needs its hot-update payload (`Android/data/com.hottagames.nte/files/HT/`).
- **Steam** (`steam 1.0.0.87`, native arm64 + box64): client starts under
  software GL, but its CEF login window is a popup Denial does not focus, and
  Steam Guard blocked login (no email code, no mobile authenticator, no
  `RefreshToken` in `loginusers.vdf`; needs an `ssfn*` sentry file from a PC).
- **x86_64 Steam games**: the kernel has `CONFIG_BINFMT_MISC` unset, so
  box64/FEX cannot run them.

---

## 8. Useful files on this device

| Path | What |
|---|---|
| `/etc/denial/session.conf` | session env (mobile profile, render device, runtime dir) |
| `~/.local/state/denial/outputs.conf` | mode/scale/transform per output |
| `/etc/omamobile/rotation.conf` | auto-rotate hysteresis + timings |
| `/usr/local/lib/omamobile/gpu-boost.py` | Panthor interaction floor |
| `/usr/local/lib/omamobile/sensor_proxy_hysteresis.py` | SensorProxy wrapper |
| `/usr/local/bin/omamobile-display-probe` | connector `detect` before sddm |
| `/usr/local/bin/omamobile-steinerd` | steinerd bridge |
| `/usr/lib/firmware/omamobile/camera/` | extracted sensor data (local only) |
| `~/.local/share/sddm/wayland-session.log` | compositor log |
| `~/.local/state/denial/plugins/build-kits/<hash>/` | shell source that gets rebuilt |

---

## 9. Open items

1. **Unlock 701–848 MHz**: add `mali-supply` and voltage OPPs to the GPU DT
   node, then verify with `trans_stat`. Biggest remaining driver win.
2. Convert the remaining live-blur surfaces (banner, HUDs, power menu,
   Wi-Fi/Bluetooth detail, edge panels) to the cached/planted frost pattern.
3. Vulkan vs GLES shell A/B (`DENIAL_NO_VULKAN=1`, requires a reboot). PanVK
   wins raw throughput; Impeller GLES may still win on latency.
4. `CONFIG_BINFMT_MISC=y` for box64/FEX.
5. NTE hot-update payload and Steam sentry file.

---

## Changelog

- 2026-11-24: initial document. Cameras, rotation, GPU findings and floor,
  drawer settle + planted frost, boot robustness.
