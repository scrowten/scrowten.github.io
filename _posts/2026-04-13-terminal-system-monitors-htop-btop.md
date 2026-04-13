---
layout: post
title: "🖥️ Terminal System Monitors: Why I Keep Coming Back to htop"
date: 2026-04-13 00:00:00
description: A five-year htop user tries btop, gets dazzled, gets confused, and finds out what actually matters in a system monitor.
tags: linux terminal sysadmin htop btop tools
categories: self-hosting
toc:
  sidebar: left
---

*A colleague leans over your shoulder, types `btop`, and suddenly your terminal looks like the bridge of a spaceship. It is gorgeous. So why, two weeks later, are you back to typing `htop`?*

---

# 🗺️ 1. The Landscape: More Tools Than You Think

Before we get into the htop-vs-btop argument, it is worth stepping back. Most people know two or three terminal monitors. There are actually a dozen worth knowing about, each solving a slightly different problem.

Here is the full map:

| Tool | Language | Stars | Best For |
|------|----------|-------|----------|
| `top` | C | built-in | Any system, no install needed |
| `htop` | C | ~8k | Daily interactive monitoring |
| `btop++` | C++ | ~32k | Dashboard overview + GPU |
| `glances` | Python | ~32k | Remote monitoring + plugins |
| `atop` | C | ~1k | Historical forensics |
| `nmon` | C | n/a | HPC / enterprise batch capture |
| `nvtop` | C | ~10k | GPU-only specialist |
| `bottom` (`btm`) | Rust | ~13k | Cross-platform incl. Windows |
| `gotop` | Go | ~3k | Static binary + Prometheus |
| `vtop` | Node.js | ~4k | Pretty screenshots (abandoned) |
| `bashtop` | Bash | ~11k | Historical curiosity (superseded) |
| `bpytop` | Python | ~11k | btop predecessor (superseded) |

The last two — bashtop and bpytop — are the ancestors of btop++. Same author (aristocratos), same vision, progressively rewritten for speed: Bash → Python → C++.

If that list feels overwhelming, good. That is the point. The ecosystem is rich and each tool has a reason to exist.

---

# 🏛️ 2. `top` — The Ancestor You Can Never Escape

Before anything else, there is `top`. Installed on every Linux system, every macOS, every container with a base image. No dependencies, no install step, runs over a 9600 baud serial console if you need it to.

```bash
top
```

That is it. That is the whole install.

Its weaknesses are well known: cryptic keybindings (`Shift+P` to sort by CPU, `Shift+M` for memory, `k` then type a PID to kill), no per-core breakdown in older versions, no scrolling. But its superpower is availability. When you SSH into an unknown machine to debug a production incident, `top` is always there.

For scripting, `top -b -n 1` gives you a single-pass batch output you can parse with `awk`. That alone makes it irreplaceable.

---

# 💚 3. `htop` — The Five-Year Companion

I switched to htop sometime around 2020 and it became muscle memory fast. Here is what hooked me.

**Per-core CPU bars.** The moment you see all 8 (or 32, or 128) cores visualized as individual colored bars at the top of the screen, you never go back to `top`'s single aggregate percentage. Hot core? You see it immediately.

**The F-key bar.** The bottom row of the screen tells you exactly what each function key does: F1=Help, F2=Setup, F3=Search, F4=Filter, F5=Tree, F6=SortBy, F7=Nice-, F8=Nice+, F9=Kill, F10=Quit. No memorization required. Contrast with `top` where you need to remember that `k` kills and `r` renices.

**Tree view (F5).** Press F5 and your process list reorganizes as a hierarchy. You can instantly see that the runaway `python3` process was spawned by your cron job, which was called by systemd. This is the feature sysadmins actually rely on.

**Signal menu (F9).** `top` lets you kill. htop lets you send any Unix signal: SIGTERM, SIGHUP, SIGSTOP, SIGUSR1, SIGUSR2 — the full list, with names. For debugging a misbehaving daemon, this is indispensable.

**One config file.** All your customizations live in `~/.config/htop/htoprc`. Drop that in your dotfiles, deploy it with Ansible, and every server you touch looks exactly how you expect.

---

# ✨ 4. `btop++` — The Beautiful Detour

My colleague ran `btop` and the screen transformed.

```bash
sudo apt install btop   # Ubuntu 22.04+
# or
brew install btop       # macOS
```

What you see: a full dashboard. CPU history graph across the top — not a live bar, but a scrolling 60-second timeline. Memory, swap, network, and disk I/O in the middle, each with their own graphs. Process list at the bottom. All of it in 24-bit truecolor with Unicode braille rendering that makes graphs look almost pixel-perfect.

The history graphs are btop's genuine killer feature. You glance at the screen and immediately see that your CPU spiked 45 seconds ago, peaked at 80%, and has been cooling down since. With htop, you only see right now. That context can matter.

btop also has GPU monitoring built in — NVIDIA, AMD (ROCm), and Intel iGPU on Linux. No need to run nvtop in a separate pane. And it has themes. Many themes. Press `Esc` to open the menu and you can pick from Nord, Dracula, Gruvbox, and a dozen others.

On first launch, it is hard not to be impressed.

---

# 🤔 5. Why I Came Back — The Honest Critique

Two weeks in, I noticed something. I was spending more time navigating btop's menus than actually diagnosing problems.

**The layout competes with itself.** btop allocates screen space to CPU graphs, memory graphs, network, and disk before it shows you the process list. When a server is under load and you need to find the offending process fast, those graphs are beautiful noise. You have to scroll down, or make the process pane taller, or toggle panes with keyboard shortcuts (D for disk, 3 for network). htop opens full-screen on a process list immediately.

**The gradient edge effect.** The bottom edges of btop's display fade out — a design choice for aesthetics. In practice, the last few processes in the list are hard to read. There is no option to disable this.

**ZFS users, beware.** btop's disk I/O monitoring does not work correctly on ZFS. If you run FreeBSD, TrueNAS, or any ZFS-on-Linux setup, the disk pane is unreliable.

**Console rendering breaks.** btop requires a UTF-8 capable terminal with 24-bit color support for its full experience. On out-of-band management consoles (iDRAC, iLO, iKVM), serial consoles, or some legacy SSH clients, it renders as scrambled characters. htop degrades gracefully to 8-color mode and still works perfectly.

**An open CPU measurement bug.** There is a known GitHub issue (#665) where btop reports per-process CPU at roughly 1/10th the value shown by htop and top. It is not universal, but it appears in specific kernel and environment combinations. For a monitoring tool, that is a confidence-eroding bug.

One framing I found that resonated: *btop is optimized for looking impressive at a glance; htop is optimized for the repeated 8-hour sysadmin workflow of scanning a process list*.

That distinction is real.

---

# 🔬 6. The Right Tool for Each Job

After spending time with all of these tools, here is where I landed on what to actually reach for:

**`top`** — when you SSH into an unfamiliar machine and cannot install anything. Always there. `top -b -n 1` for scripting.

**`htop`** — daily driver. Interactive monitoring, process hunting, killing, renicing, tree view. Fast to install, fast to use, works everywhere, one config file for your dotfiles.

**`btop`** — overview mode and GPU monitoring. When you want a dashboard-at-a-glance or you are watching GPU utilization during a model training run. Keep it in a dedicated tmux pane.

**`atop`** — the forensics tool no one talks about enough. `atop` runs a daemon that logs system state to `/var/log/atop/atop_YYYYMMDD` every few minutes. When something happened at 3am and you need to reconstruct exactly what your system was doing, `atop -r /var/log/atop/atop_20260101` replays it. No other tool in this list has this capability.

**`glances`** — remote monitoring and integrations. Glances has a web UI mode (`glances -w`), a REST API, and plugins for InfluxDB, Prometheus, Docker, and Kafka. If you need to monitor multiple hosts from a browser or push metrics to a stack, glances is the right choice. Be aware of its Python overhead — benchmarks show 10–45% CPU in some environments, which is significant on low-power hardware.

**`nvtop`** — dedicated GPU monitoring, especially if you have multiple GPUs or exotic hardware (AMD, Intel, Qualcomm, Rockchip). More vendor support than btop's built-in GPU pane.

**`bottom` (`btm`)** — if you work across Linux, macOS, and Windows. The Rust binary deploys cleanly everywhere and has the best Windows story of any tool here.

---

# 🏆 7. The htop Sweet Spot

Why does htop remain the default recommendation after 20+ years and a wave of prettier alternatives?

It comes down to what sysadmins actually do. When production is on fire, you SSH in, you type `htop`, and within three seconds you have a sorted, scrollable, searchable process list with per-core CPU visualization. You press F4 to filter by a service name, F9 to send a signal, F5 to see the process tree. No menu navigation required. The whole workflow is encoded in muscle memory and it works the same on a 2006 server as a 2026 one.

btop is a text version of a GUI system monitor — beautiful, information-rich, genuinely impressive. For sustained daily use in ops workflows, that richness becomes friction.

The tldr: **htop found the 80/20 sweet spot and has stayed there**. btop overshot into the territory where more is more, not more is better.

That said — install both. They are not mutually exclusive. I run htop for process work and keep btop in a corner pane when I want the historical CPU view. The two tools complement each other.

---

# 🚀 Quick Install Reference

```bash
# htop
sudo apt install htop          # Debian/Ubuntu
sudo dnf install htop          # Fedora/RHEL
brew install htop              # macOS
apk add htop                   # Alpine

# btop
sudo apt install btop          # Ubuntu 22.04+
sudo dnf install btop          # Fedora 36+
sudo pacman -S btop            # Arch
brew install btop              # macOS

# atop (+ start the logging daemon)
sudo apt install atop
sudo systemctl enable --now atop

# glances
pip install glances
glances -w                     # web UI on :61208

# nvtop (GPU monitoring)
sudo apt install nvtop

# bottom (Rust, cross-platform)
# https://github.com/ClementTsang/bottom/releases
cargo install bottom           # via Cargo
```

---

*Five years in, my `.bashrc` still has `alias top='htop'`. Some defaults earn their place.*
