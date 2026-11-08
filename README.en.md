# android-mainline-port

A handbook and experience library for porting mainline Linux and a desktop distribution to Android phones and tablets, packaged as an Agent Skill. Claude Code, Codex, Cursor, Gemini CLI and other AI coding agents can use it directly, and people can read it as is. [中文](README.md)

The experience comes from three complete Qualcomm tablet ports, each taken from nothing to a public release (mainline kernel 6.17 to 7.4, Ubuntu 26.04 + GNOME). The goal is simple: when you port the next device from scratch, get there as fast as possible with as few detours as possible.

## Source projects

| Device | SoC | Project |
|---|---|---|
| Xiaomi Pad 6 Pro | Snapdragon 8+ Gen 1 (SM8475) | [yzddmr6/xiaomipad-6pro-mainline](https://github.com/yzddmr6/xiaomipad-6pro-mainline) |
| OnePlus Pad 2 Pro | Snapdragon 8 Elite (SM8750) | [yzddmr6/onepluspad-2pro-mainline](https://github.com/yzddmr6/onepluspad-2pro-mainline) |
| Xiaomi Pad 8 Pro | Snapdragon 8 Elite (SM8750) | [yzddmr6/xiaomipad-8pro-mainline](https://github.com/yzddmr6/xiaomipad-8pro-mainline) |

All three run a native Ubuntu GNOME desktop on the device, dual-booting with Android — not a container and not a remote desktop. This skill turns the problems that showed up in all three, kept coming back, or took a long time to pin down into a generic process, decision rules and experience entries.

## 📮 Follow and discuss

Long-form technical articles, write-ups and progress updates are on the WeChat official account; the community group is on Xiaohongshu — scan to join. Both are in Chinese.

<div align="center">
<table>
<tr><th>WeChat: 熵减矩阵</th><th>Xiaohongshu: yzddmr6</th></tr>
<tr>
<td align="center"><img src="docs/assets/gzh-qr.png" width="300" alt="WeChat: 熵减矩阵"></td>
<td align="center"><img src="docs/assets/xhs-qr.jpg" width="300" alt="Xiaohongshu: yzddmr6"></td>
</tr>
<tr><td align="center">Articles · write-ups · progress</td><td align="center">Community group · feedback</td></tr>
</table>
</div>

## What it helps with

- **Assessing a new device**: whether it can be ported, which route to take, how long it will take, and which decisions the owner has to make.
- **Following a process**: project setup, build environment, evidence collection and backup, first boot, first light and a RAM desktop, installing to storage, peripherals one at a time, release and maintenance — each step says what to do and when it counts as done.
- **Technical decisions**: more than 40 key decision points, each with a default, "if this happens, do that" rules, the mechanism behind them, and where they do not apply — including how to compare two implementation routes, moving to a new kernel generation or sharing one kernel tree across devices, and how to take community contributions.
- **Debugging**: eleven common problem patterns with generic steps, then concrete pitfalls by area; error strings, function names and config options in your logs can be searched directly.
- **Taking over midway**: which documents must be kept and how technical debt is recorded, so the next person or agent can continue from the documents alone.
- **Working with the owner and with AI executors**: which decisions you make yourself and which go to the owner, how to spend as little of the owner's time as possible, and how to brief sub-agents without outsourcing the thinking.

A few lessons that save the most time:

- `fastboot boot` returns OKAY but the device stays in Fastboot: check that the DTB msm-id/board-id match what the bootloader expects, not the image size.
- Use the firmware framebuffer (simpledrm) only for first light; start the native display takeover early and in parallel — brightness, screen-off, suspend and external displays all depend on it.
- On Qualcomm, default to s2idle: messages from the remote processors fully wake the system from deep sleep and do not show up as wakeup sources.
- Measure sleep duration with wall-clock time only; the monotonic clock used by dmesg and the journal stops while suspended.
- Size ccache from the measured ~0.3 GB per full kernel build — 10 GB is enough — and set `base_dir`.
- Every shortcut taken to get something working goes into a single technical-debt register, and gets paid back once the chain works end to end.
- Search for existing implementations by interface keywords (chip, driver, device node, protocol), not by project name — and do it before you design.
- Before asking the owner to run a test in person, dry-run everything that can run unattended on the device, in the same order.

## Install

### Recommended: npx skills

One command installs it for all the AI coding agents you use (Claude Code, Codex, Cursor, Gemini CLI, GitHub Copilot, OpenCode and more). Requires Node.js 22 or later:

```bash
npx skills add yzddmr6/android-mainline-port -g
```

- `-g` installs into your user directory for all projects; without it, it installs into the current project only.
- It asks which agents to install for; you can also name them, for example `-a claude-code -a codex`.
- Update: `npx skills update android-mainline-port -g`; remove: `npx skills remove android-mainline-port -g`.

### Claude Code plugin

In a Claude Code session:

```text
/plugin marketplace add yzddmr6/android-mainline-port
/plugin install android-mainline-port@android-mainline-port
```

Or from a terminal:

```bash
claude plugin marketplace add yzddmr6/android-mainline-port
claude plugin install android-mainline-port@android-mainline-port
```

Update: enable auto-update on the Marketplaces page of `/plugin`, or run `claude plugin marketplace update android-mainline-port` followed by `claude plugin update android-mainline-port@android-mainline-port`.

### Gemini CLI

```bash
gemini skills install https://github.com/yzddmr6/android-mainline-port.git
```

### Manual git clone

Clone the repository into your agent's skills directory. The directory must be named `android-mainline-port`:

| Agent | User level | Project level |
|---|---|---|
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |
| Codex | `~/.agents/skills/` | `.agents/skills/` |
| Gemini CLI | `~/.gemini/skills/` or `~/.agents/skills/` | `.gemini/skills/` |
| Cursor | `~/.cursor/skills/` or `~/.agents/skills/` | `.cursor/skills/` |
| GitHub Copilot | `~/.copilot/skills/` or `~/.agents/skills/` | `.github/skills/` |
| OpenCode | `~/.config/opencode/skills/` or `~/.agents/skills/` | `.opencode/skills/` |

For Claude Code:

```bash
git clone --depth 1 https://github.com/yzddmr6/android-mainline-port ~/.claude/skills/android-mainline-port
```

Update with `git pull` in that directory. Do not use GitHub's "Download ZIP": the extracted folder ends in `-main`, does not match the skill name, and some agents will not recognise it.

### claude.ai and the desktop app

1. Download `android-mainline-port.zip` from [Releases](https://github.com/yzddmr6/android-mainline-port/releases/latest).
2. Turn on "Code execution and file creation" in Settings → Capabilities.
3. Open Customize → Skills, click "+", choose to upload a skill, and select the zip.

### Let your agent install it

Send this to your AI coding agent:

```text
Please install this Agent Skill for yourself: https://github.com/yzddmr6/android-mainline-port
1. Find your user-level skills directory: Claude Code uses ~/.claude/skills; Codex, Gemini CLI, Cursor, GitHub Copilot and OpenCode can use ~/.agents/skills, or whatever directory your own documentation specifies.
2. Run git clone --depth 1 https://github.com/yzddmr6/android-mainline-port <skills-dir>/android-mainline-port; if the directory already exists, run git pull --ff-only inside it instead.
3. The directory name must be exactly android-mainline-port, matching the name field in SKILL.md.
4. Check that SKILL.md, references/ and experience/ are there and that SKILL.md starts with name: android-mainline-port.
5. Do not change other skills or settings. Tell me where it was installed and whether I need to restart or reload.
```

### Check that it works

- Claude Code: type `/android-mainline-port` (with the plugin: `/android-mainline-port:android-mainline-port`) and the skill should appear.
- Other agents: ask "I want to port mainline Linux to an Android tablet, help me assess it first"; the agent should read the skill before answering.

## Examples

No special commands are needed after installing; describe your situation as usual:

- "I have a Snapdragon 8 Gen 3 tablet with an unlockable bootloader. Can it run mainline Linux + Ubuntu, and roughly how long would it take?"
- "`fastboot boot` returns OKAY but the screen stays in Fastboot and pstore is empty. Help me debug."
- "Suspend wakes up after a second or two. Help me find the wakeup source."
- "Take over this porting project: read the docs and tell me which step we are at and what comes next."

## Layout

- **Framework**: `SKILL.md` and `references/` — the order of work, working discipline, technology choices, build environment, what counts as passing, Ubuntu integration and decoupling. Fairly stable.
- **Experience library**: `experience/` — recommended routes, completion criteria and concrete pitfalls by area (symptom, cause, how to confirm, how to fix, how to prevent), extended with every project.

```
SKILL.md                         process overview, ordering pitfalls, discipline, selection principles, owner decisions
references/
  process.md                     detailed actions, outputs and completion criteria for each step
  discipline.md                  what to document, technical debt, directory layout, attribution, estimation, sub-agents
  build-environment.md           host, toolchain, ccache, OUT dirs, root filesystem builds, caching
  decisions.md                   key decisions: defaults, condition → action rules, mechanism, limits
  verification.md                status words, evidence, cold boots, test automation, performance budgets
  ubuntu.md                      Ubuntu layering and decoupling, package split, boot hooks, other distros
experience/
  README.md                      entry format and maintenance rules
  patterns.md                    eleven problem patterns and generic handling
  bootloader-and-boot.md         ABL, boot image formats, first boot, debug channel, recovery
  kernel-and-resources.md        built-in vs modules, resource cleanup, PMIC voltages, DT values
  display.md                     simpledrm and native takeover, DSC, screen off/on, backlight
  gpu.md                         Adreno, firmware, custom Mesa, Mesa bugs on new GPUs
  touch-and-input.md             touch (TDDI), keys, keyboard covers, fingerprint
  audio.md                       AudioReach, amplifiers, microphones, UCM, volume
  wireless.md                    Wi-Fi, Bluetooth
  power-and-time.md              battery, charging, suspend, system time
  sensors-camera-media.md        sensors and rotation, camera, hardware video decode
  usb-and-storage.md             USB, external display, partitions, dual boot, flashing, installer
  ubuntu-userspace.md            Ubuntu packaging, dracut, snap, GNOME defaults
  build-and-release.md           build pipeline, caching, reproducibility, releases
  testing-and-process.md         pass criteria, automated testing, collaboration, sub-agents
```

The porting process has nine steps:

| Step | What |
|---|---|
| 0 Scoping | sources, reuse survey, hardware inventory, choices, estimate, owner decisions |
| 1 Build environment | toolchain, ccache, long-lived OUT dirs, build a community reference kernel as is |
| 2 Evidence and backup | root, verified backups, runtime DT, firmware, voltages, display parameters |
| 3 First boot | understand the bootloader, minimal DT, USB network shell, automatic fallback |
| 4 First light and RAM desktop | display, touch, GPU, GNOME from RAM, desktop test automation |
| 5 Install to storage | partitions, dual boot, boot switching, debug channel, cold-boot checks |
| 6 Peripherals | one at a time: reuse, compare with stock, test alone, on device, merge, regress |
| 7 Release | feature freeze, layered packages, fresh install from the public docs |
| 8 Maintenance | drop local code once upstream lands, pay down debt, cold-boot every update |

## Scope and freshness

- Qualcomm platforms are covered in the most detail; the process and discipline apply to other SoCs, but hardware details need your own research.
- For a distribution other than Ubuntu, adapt the Ubuntu parts using the last section of `references/ubuntu.md`.
- The content reflects upstream as of 2026. Version numbers, interface names and upstream status age; check them before relying on them, and if upstream has already solved a problem, use the simpler route.
- The skill itself is written in Chinese; AI agents read and apply it without trouble.

## ⚠️ Risks

- Unlocking the bootloader wipes the device and may affect the warranty; flashing partitions incorrectly can leave the device unbootable.
- Before you start, back up and verify your partitions as the skill describes, and make sure you have a way to recover.
- This repository provides methods and experience only — no flashing tools, images or firmware. It is provided as is, at your own risk.

## Contributing

Issues and pull requests that add experience are welcome:

- Append entries to the matching file in `experience/`, following the format in `experience/README.md`.
- Only add problems whose root cause is confirmed; state the symptom, cause, how to confirm, how to fix and where it applies.
- Do not include device models, serial numbers, addresses, calibration data or personal information. Chip names, driver names and error strings are fine — they are the search keys.

## License

[MIT](LICENSE)
