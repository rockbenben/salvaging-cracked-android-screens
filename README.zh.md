# salvaging-cracked-android-screens

> 365 开源计划 #024 · 内屏碎裂、边缘不可用的安卓/墨水屏平板，把画面锁进完好区域继续用——整机用 `wm overscan`，或单个 app 接入一层裁剪，并含"该走哪条路"的决策框架。

[English](README.md) · [![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

一个 **agent skill**（适用于 Claude、Cursor，或任何读 `SKILL.md` 格式的 agent），外加一个可直接拿走的 Flutter 组件，专治一个难缠的问题：

> 安卓墨水屏平板的内屏物理碎裂——某条边死了一条、某个角坏了一块——但大部分屏幕还好。你想继续用它。

碎屏的帧缓冲其实还是完整的，只是坏掉的像素不显示而已。所以办法是：把设备**绘制的内容（以及接受触摸的区域）限制到完好的矩形里**，让坏掉的边缘当作无用的黑边。

这里真正的价值不是某条魔法命令，而是**决策框架**——因为那条"显而易见"的路子，在新硬件上是个坑。

## 它固化了什么

- **可行性闸门，按顺序问** —— 用它们决定走哪条路，而不是挑最酷的：
  1. **安卓版本。** 整系统方案 `wm overscan` 在 **安卓 11 被移除了**，只在 ≤10 存在。
  2. **开发者选项 / ADB 到底能不能开？** 有些墨水屏机型死活开不出开发者选项——没 ADB，overscan/scrcpy 全线报废。
  3. **要整系统，还是一个 app 就够？**
- **两条真实机制，如实对比：**

  | | 整系统 `wm overscan` | app 级裁剪 |
  |---|---|---|
  | 限制范围 | 整个安卓 | 单个 app |
  | 前提 | ADB + 安卓 ≤10 | 什么都不需要——装上就行 |
  | 重启保留 | **是**（持久化，已回 AOSP 源码验证） | **是** |
  | 软键盘 | 跟着一起移 | 搬不动 |

- **一个可直接用的 Flutter `ScreenCropper`**（见 [`SKILL.md`](SKILL.md)）—— 在 `MaterialApp.builder` 包一层，整个 app（路由页、对话框、底部弹层、SnackBar）就都渲染进一个**可在 app 内调节**的矩形。免 ADB、免 root、重启不丢。
- **会白白浪费几小时的坑** —— `wm size` 是**缩放**不是内缩（内容照样落回裂缝上）；裁剪要存**比例，不是像素**；软键盘是唯一 app 搬不动的东西。

## 为什么做成 skill，而不是一篇博客

这里的事实非显而易见、且凭记忆极易搞错：一个很强的 baseline agent 冷启动回答时，对版本闸门**含糊**、对重启持久化**答错**、且**完全没有**免 ADB 的兜底方案。skill 里每一条断言都回 [AOSP 源码](https://cs.android.com/android/platform/superproject/+/android-10.0.0_r30:frameworks/base/services/core/java/com/android/server/wm/) 和公开报道复核过。

## 怎么用

```bash
# skill 自包含在 SKILL.md 里 —— 只需把它拷进 skills 目录下的同名文件夹
mkdir -p ~/.claude/skills/salvaging-cracked-android-screens
cp SKILL.md ~/.claude/skills/salvaging-cracked-android-screens/
```

然后直接跟 agent 描述问题（"我的 Boox 屏幕右边裂了，帮我继续用"），skill 就会触发。或者直接读 [`SKILL.md`](SKILL.md)——它自成一体。

## 缘起

来自一块真实的坏屏：一台 **BOOX Note X2**（安卓 10），墨水屏沿右侧竖条 + 底部块碎裂。开发者选项开不出来、ADB 无从谈起——这恰恰是"免 ADB 的 app 级裁剪"存在的理由，它先在一个真实 app 里落地，之后才把这份经验蒸馏成 skill。

## 许可

[MIT](LICENSE) © 2026 rockbenben
