---
date: '2026-10-05T22:13:56+08:00'
draft: false
title: 'Side Projects Issue 3: The New Mac Power Tool'
showtoc: true
tags: ["python", "open-source", "tools", "sideprojects"]
aliases: ["/side-projects/side-projects-issue-3/"]
---

## Why your Mac needs [Vorssaint](https://github.com/vorssaint/vorssaint-utils): the ultimate open-source menu bar toolkit.

Have you ever looked at your Mac's menu bar and realized you are running half a dozen different micro-utility apps? One for window snapping, another for clipboard history, a third for managing per-app audio, and a system monitor just to keep an eye on your battery. Before you know it, your menu bar is a cluttered mess, and you've spent way too much money on separate licenses and subscriptions.

Enter [Vorssaint](https://github.com/vorssaint/vorssaint-utils), a game-changing, free, and open-source macOS toolkit that essentially does the job of a dozen paid Mac apps, all tucked neatly behind a single menu bar icon.

## What Can Vorssaint Do?

Here are some of its most popular features:

- **Per-app volume mixer:** set volume for each app, boost quiet audio past 100%, and send music to your speakers while a call stays on your headset.
- **System monitor and fan control:** CPU, GPU, memory, temperatures, and battery health with history graphs, plus live readouts right in the menu bar.
- **Window snapping:** snap windows into layouts and move them between displays with shortcuts, screen edges, or drag gestures.
- **App switcher and Dock previews:** switch between apps and windows with live previews, or hover over Dock icons to see every open window.
- **Clipboard history:** search past text, images, and files, pin favorites, and paste as plain text.
- **Command Bar:** search apps, files, clipboard history, and menu commands from one field, and do quick math and unit conversions.
- **Dynamic Island:** music, timers, calendars, and downloads around the camera notch, or a simulated one on Macs without it.
- **Screenshots and screen recording:** annotate, redact, and pin screenshots, and record with separate system audio and microphone tracks.
- **Mouse fixes:** smooth scrolling, side buttons that work in Finder and browsers, and independent scroll direction for mouse and trackpad.
- **Text snippets:** expand short triggers into longer text, with date, time, and clipboard variables.

Here is why Vorssaint is arguably one of the best utility repositories on GitHub right now:

## 1. The "Everything App" for macOS

Vorssaint isn't just a basic tool; it's an absolute powerhouse. Out of the box, it provides a true per-app volume mixer (so you can turn down your browser while keeping Spotify loud), a comprehensive system monitor with temperature and fan controls, a robust window snapper, and a clipboard manager. It even includes an app uninstaller and a Homebrew manager.

## 2. Zero Bloat: Only Use What You Need

Usually, "all-in-one" apps are massive resource hogs. Vorssaint solves this brilliantly with a modular design. You can pick and choose exactly which features you want to activate. Don't need the mouse enhancements or text snippets? Don't install them. Disabled features stop loading completely and vanish from the UI. It only asks for the macOS permissions required for the specific tools you actually use.

## 3. Local-First, Privacy-First, and Truly Free

In an era where a simple volume slider app might ask for a monthly subscription and an email address, Vorssaint is incredibly refreshing. It is 100% free, open-source (GPL 3.0), and local-first.

- No accounts to create.
- No telemetry or data harvesting.
- Absolutely no subscriptions.

## 4. Next-Level Bonus Features

Beyond the standard utilities, Vorssaint packs in features that feel like macOS magic. It brings a fully functional "Dynamic Island" to any Mac screen (managing music, timers, and even tracking AI agent API limits). It also includes a screen recorder that captures separate system audio and microphone tracks, a native color picker, and tools to fix the scroll wheel behavior of third-party mice.

## The Verdict

Vorssaint is a masterclass in what open-source software can achieve. If you want to clean up your menu bar, take back your system resources, and save a significant amount of money, this toolkit is a must-have.

## Want to Try It Out?

You can check out the source code and download it directly from [GitHub](https://github.com/vorssaint/vorssaint-utils), or install it in seconds via Homebrew:

```bash
brew install --cask vorssaint
```
