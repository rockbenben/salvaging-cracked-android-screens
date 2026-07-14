# salvaging-cracked-android-screens

> 365 Open Source Plan #024 · Keep a cracked-screen Android / e-ink tablet usable by confining the display to its intact area — the whole system via `wm overscan`, or one app via a drop-in cropper, with the framework for picking the right one.

[中文文档](README.zh.md) · [![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

An **agent skill** (works with Claude, Cursor, or any agent that reads the `SKILL.md` format) plus a drop-in Flutter widget, for one stubborn problem:

> Your Android e-ink tablet's inner panel physically cracked — a dead strip on one edge, a broken block in a corner — but most of the screen still works. You want to keep using it.

A cracked panel still has a full framebuffer; the dead pixels just don't show. So the fix is to confine what the device draws (and what it accepts touch from) to the **intact rectangle**, and let the broken edges be unused black margin.

The value here isn't a magic command — it's the **decision framework**, because the "obvious" fix is a trap on modern hardware.

## What it captures

- **Feasibility gates, in order** — decide the approach by these, not by what looks slick:
  1. **Android version.** `wm overscan` (the whole-system trick) was **removed in Android 11**. It only exists through Android 10.
  2. **Can ADB even be enabled?** Some e-ink units silently refuse to turn on Developer Options — no ADB means every overscan/scrcpy path is dead.
  3. **Whole system, or is one app enough?**
- **Two real mechanisms, honestly compared:**

  | | Whole-system `wm overscan` | App-level cropper |
  |---|---|---|
  | Confines | All of Android | One app |
  | Needs | ADB + Android ≤10 | Nothing — just install the app |
  | Survives reboot | **Yes** (persisted; AOSP-source-verified) | **Yes** |
  | Soft keyboard | Moves with it | Can't move it |

- **A drop-in Flutter `ScreenCropper`** ([`SKILL.md`](SKILL.md)) — wrap `MaterialApp.builder` once and the whole app (routes, dialogs, sheets, snackbars) renders inside an in-app-adjustable rectangle. No ADB, no root, survives reboot.
- **The traps that waste hours** — `wm size` *scales* instead of insetting (content lands right back on the cracks); store crop as **fractions, not pixels**; the soft keyboard is the one thing no app can relocate.

## Why a skill, not a blog post

The facts here are non-obvious and easy to get wrong from memory: a strong baseline agent, asked this cold, was *fuzzy* on the version gate, *wrong* on reboot persistence, and offered *no* no-ADB fallback at all. Every claim in the skill was re-verified against [AOSP source](https://cs.android.com/android/platform/superproject/+/android-10.0.0_r30:frameworks/base/services/core/java/com/android/server/wm/) and press coverage.

## Use it

```bash
# The skill is self-contained in SKILL.md — copy just that into a
# same-named folder in your agent's skills directory.
mkdir -p ~/.claude/skills/salvaging-cracked-android-screens
cp SKILL.md ~/.claude/skills/salvaging-cracked-android-screens/
```

Then just describe the problem to your agent ("my Boox screen cracked on the right edge, help me keep using it") and the skill triggers. Or read [`SKILL.md`](SKILL.md) directly — it's self-contained.

## Origin

Born from a real dead panel: a **BOOX Note X2** (Android 10) whose e-ink cracked along the right strip and bottom block. Developer Options wouldn't enable, so ADB was off the table — which is exactly why the no-ADB app-level cropper exists, and shipped in a real app before this skill distilled the lesson.

## License

[MIT](LICENSE) © 2026 rockbenben
