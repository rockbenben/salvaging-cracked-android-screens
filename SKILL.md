---
name: salvaging-cracked-android-screens
version: 1.0.0
description: Use when an Android device or Android e-ink tablet (Onyx Boox, Bigme, and similar) has a physically cracked/dead screen region — dead edge strips, broken corners, cracked panel — and the user wants to keep using it by restricting output to the intact area. Triggers include "broken screen", "dead pixels area", "cracked e-ink", "confine/crop display", "only show part of screen", "wm overscan", "Boox screen broke", 墨水屏碎屏/坏屏/内屏爆裂, 只显示屏幕一部分, 文石.
tags: [android, e-ink, boox, cracked-screen, wm-overscan, display, adb, flutter, repair, accessibility]
homepage: https://github.com/rockbenben/salvaging-cracked-android-screens

metadata:
  clawdbot:
    emoji: "🩹"
---

# Salvaging a Cracked Android / E-ink Screen

## Overview

A cracked panel still has a full framebuffer; the dead pixels just don't show. Confine *rendering + touch* to the intact rectangle (usually anchored top-left) and let the dead edges be unused black margin.

**Pick the approach by feasibility gates, not by what's technically coolest.** Two real mechanisms exist, and the "obvious" one (`wm overscan`) is gone on modern Android.

## Feasibility gates (in order)

1. **Android version?** `wm overscan` was **removed in Android 11 (API 30)**; present through **10**. On 11+ go straight to app-level. Verify on the device (Settings → About) — don't assume.
2. **Can Developer Options / ADB be enabled at all?** Some Boox/e-ink units silently refuse (tapping build number does nothing) → **no ADB → every ADB/overscan/scrcpy path is dead.** Confirm before planning around it. This is the gate people skip.
3. **Whole system, or is one app enough?** If the user mainly lives in one app (reader/browser/notes), app-level avoids ADB entirely.

Neither mechanism needs root.

## The two mechanisms

| | Whole-system `wm overscan` | App-level cropper |
|---|---|---|
| Confines | All of Android (launcher + every app) | One app only |
| Needs | ADB (dev options on), Android ≤10 | Nothing — just install the app |
| Survives reboot | **Yes** — persisted to `/data/system/display_settings.xml`, re-applied at boot (AOSP 10 source-verified; OEM builds may deviate — confirm with one reboot) | **Yes** |
| Soft keyboard | Moves with it | **Can't move it** (OS draws it at physical bottom) |

*Note:* in AOSP 10 `setOverscan`'s only gate is `WRITE_SECURE_SETTINGS`, which is adb-grantable to a normal app (`pm grant <pkg> android.permission.WRITE_SECURE_SETTINGS`). Calling it from that app still needs hidden-API reflection on `IWindowManager`, subject to non-SDK restrictions on Android 9+ — a plausible route to PC-free re-tuning, but untested here; since overscan already persists you rarely need it.

## Approach A — `wm overscan` (Android ≤10 + ADB)

```bash
adb shell wm size                     # confirm resolution & orientation
adb shell settings put system accelerometer_rotation 0   # lock rotation — dead zones must not move
adb shell wm overscan 0,0,<right>,<bottom>   # left,top,right,bottom insets in px
adb shell wm density <n>              # optional: lower density so more fits the smaller area
adb shell wm overscan reset           # undo anytime
```

- Right + bottom damage → set only right & bottom insets; left/top = 0 → good rect anchors top-left. Touch is clipped to the inset region too (dead-zone taps ignored — usually desired).
- Measure margins: `adb shell screencap` captures the *full* framebuffer — display a fullscreen grid image, compare screenshot ("what should show") against what's visible. Or iterate values live; `scrcpy --crop W:H:X:Y` lets you see and drive the good region from a PC while tuning.
- Some OEM builds clamp overscan or let a few system popups ignore it — test on the real firmware.

## Approach B — App-level cropper (install-only, no ADB)

Build or modify one app to render only in the intact rectangle, black elsewhere, with **in-app adjustable** insets — the only path that needs nothing from the device.

- **Store insets as fractions (0.0–0.5) per edge, not pixels** — survives rotation/resolution changes. Default 0 = no-op for normal users. **Clamp each edge ≤0.5** so a usable area — and your reset control — always remains.
- Calibration screen: per-edge −/+ steppers (beat sliders on laggy e-ink), live preview, reset button; persist the fractions. Wiring at `MaterialApp.builder` means the calibration screen is **itself inside the crop**, so its own steppers shift/shrink as you adjust — honest live feedback (the whole app is really cropping), but the controls move under the finger. Lean into it, or exempt the calibration route from the cropper so the controls stay put. (Validated 2026-07 building this in a Flutter app.)
- Native Android: give the root view per-edge margins over a black background. Flutter: wrap once via `MaterialApp.builder` — confines routes, dialogs, sheets, snackbars together:

```dart
import 'dart:math' as math;
// Wire once: MaterialApp(builder: (ctx, child) => ScreenCropper(
//   right: f.right, bottom: f.bottom, child: child ?? const SizedBox()))
// builder's `child` can be null mid-route — `child!` crashes; guard it.
class ScreenCropper extends StatelessWidget {
  const ScreenCropper({super.key, required this.child,
      this.left = 0, this.top = 0, this.right = 0, this.bottom = 0});
  final Widget child;
  final double left, top, right, bottom; // fractions (0.0–0.5) hidden per edge
  @override
  Widget build(BuildContext context) {
    if (left<=0 && top<=0 && right<=0 && bottom<=0) return child; // no-op
    return LayoutBuilder(builder: (ctx, c) {
      final w=c.maxWidth, h=c.maxHeight;
      final l=left*w, t=top*h;
      final gw=math.max(1.0, w-l-right*w), gh=math.max(1.0, h-t-bottom*h);
      final mq=MediaQuery.of(ctx);
      return ColoredBox(color: const Color(0xFF000000), child: Stack(children:[
        Positioned(left:l, top:t, width:gw, height:gh,
          // Override size so layouts adapt instead of clipping.
          child: MediaQuery(data: mq.copyWith(size: Size(gw,gh)), child: child)),
      ]));
    });
  }
}
```

**Ceiling:** the system soft keyboard is drawn by the OS at the physical bottom — no app can relocate it. Typing lands in the dead zone; fine for tap-first apps, call it out for text-heavy ones.

## Common mistakes

- **Using `wm size` to "shrink" the screen** — SurfaceFlinger *scales it to fill the whole panel*; content still lands on the cracks. Only `overscan` insets.
- **Believing overscan is lost on reboot** — it persists (`display_settings.xml`); don't build per-boot re-apply machinery you don't need.
- **Absolute-pixel crop in the app** — breaks on rotation/other resolution. Use fractions.
- **Forgetting rotation lock** — if the panel rotates, the dead zones move relative to content.
