# android-mainline-port

把主线 Linux 内核和桌面发行版移植到 Android 手机、平板的操作手册与经验库，以 Agent Skill 的形式组织，Claude Code、Codex、Cursor、Gemini CLI 等 AI 编程助手都能直接使用，人也可以直接阅读。[English](README.en.md)

经验来自三台高通平板从零到公开发布的完整移植（主线内核 6.17 到 7.4，Ubuntu 26.04 + GNOME）。目标只有一个：从头再移植一台设备时，效率最高、少走弯路。

## 来自这些移植项目

| 设备 | SoC | 项目 |
|---|---|---|
| 小米平板 6 Pro | 骁龙 8+ Gen 1（SM8475） | [yzddmr6/xiaomipad-6pro-mainline](https://github.com/yzddmr6/xiaomipad-6pro-mainline) |
| 一加平板 2 Pro | 骁龙 8 至尊版（SM8750） | [yzddmr6/onepluspad-2pro-mainline](https://github.com/yzddmr6/onepluspad-2pro-mainline) |
| 小米平板 8 Pro | 骁龙 8 至尊版（SM8750） | [yzddmr6/xiaomipad-8pro-mainline](https://github.com/yzddmr6/xiaomipad-8pro-mainline) |

三个项目都在 Android 设备上原生运行 Ubuntu GNOME 桌面，与 Android 双系统共存，不是容器，也不是远程桌面。这份 skill 把三个项目里共同出现、反复出现、花很久才查清的问题，整理成了通用的流程、决策规则和经验条目。

## 📮 关注与交流

技术长文、经验分享和进展动态在公众号；交流群在小红书，扫码进群交流。

<div align="center">
<table>
<tr><th>公众号：熵减矩阵</th><th>小红书：yzddmr6</th></tr>
<tr>
<td align="center"><img src="docs/assets/gzh-qr.png" width="300" alt="公众号：熵减矩阵"></td>
<td align="center"><img src="docs/assets/xhs-qr.jpg" width="300" alt="小红书：yzddmr6"></td>
</tr>
<tr><td align="center">技术长文 · 经验分享 · 进展动态</td><td align="center">进群交流 · 问题反馈</td></tr>
</table>
</div>

## 能帮你做什么

- **评估新设备**：能不能移植、走哪条路线、要多久、需要你决定哪些事。
- **按流程推进**：从立项、构建环境、取证备份、首次启动、点亮桌面、安装到存储、逐个外设，到发布和维护，每一步写清做什么、做到什么程度算完成。
- **做技术决策**：40 多个关键决策点，每个都写了默认做法、"遇到什么情况怎么做"、背后的原因和不适用的情况；包括怎么比较两条实现路线、内核换代和几台设备共用一棵内核、怎么接收社区贡献。
- **排查故障**：十一类常见问题模式和通用排查步骤，再按领域查具体的坑；日志里的错误码、函数名、配置项可以直接搜索。
- **中途接手**：规定了必须写哪些文档、技术债怎么登记，下一个人或 agent 只看文档就能继续。
- **和用户、AI 执行者协作**：哪些决定自己做、哪些交给用户；怎么少占用户的时间；怎么给子 agent 写任务而不把思考外包出去。

几条最省时间的经验：

- `fastboot boot` 返回 OKAY 却一直停在 Fastboot，先查设备树的 msm-id/board-id 是否和 bootloader 期望一致，而不是镜像大小。
- 借用固件画面（simpledrm）只用于首次点亮，原生显示接管要尽早并行开始，亮度、熄屏、休眠、外接显示都依赖它。
- 高通平台休眠默认用 s2idle：远端处理器的消息会把 deep 睡眠完整唤醒，而且不出现在唤醒记录里。
- 睡眠时长看墙上时间或挂起前后 BOOTTIME 与 MONOTONIC 差值的变化，dmesg 和 journal 的单调时钟在睡眠中会停。
- ccache 按实测约 0.3 GB 一次完整内核构建来配，10 GB 足够，同时设置 `base_dir`。
- 为了快速跑通而用的临时做法，全部登记进一份统一的技术债表，链路跑通后集中还债。
- 找现成实现按接口词搜（芯片名、驱动名、设备节点、协议），不按项目名搜；在定设计之前搜。
- 请用户到场测试之前，先在真机上把能自动跑的部分完整空跑一遍。

## 安装

### 推荐：npx skills

一条命令装给你在用的所有 AI 编程助手（Claude Code、Codex、Cursor、Gemini CLI、GitHub Copilot、OpenCode 等），需要 Node.js 22 或更高版本：

```bash
npx skills add yzddmr6/android-mainline-port -g
```

- `-g` 装到用户目录，所有项目都能用；不加则只装到当前项目。
- 运行后会让你选择装给哪些助手；也可以直接指定，例如 `-a claude-code -a codex`。
- 更新：`npx skills update android-mainline-port -g`；卸载：`npx skills remove android-mainline-port -g`。

### Claude Code 插件

在 Claude Code 会话里输入：

```text
/plugin marketplace add yzddmr6/android-mainline-port
/plugin install android-mainline-port@android-mainline-port
```

或者在终端里：

```bash
claude plugin marketplace add yzddmr6/android-mainline-port
claude plugin install android-mainline-port@android-mainline-port
```

更新：在 `/plugin` 的 Marketplaces 页打开自动更新，或手动执行 `claude plugin marketplace update android-mainline-port` 后再 `claude plugin update android-mainline-port@android-mainline-port`。

### Gemini CLI

```bash
gemini skills install https://github.com/yzddmr6/android-mainline-port.git
```

### 手动 git clone

把仓库克隆到助手的 skill 目录，目录名必须是 `android-mainline-port`：

| 助手 | 用户级目录 | 项目级目录 |
|---|---|---|
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |
| Codex | `~/.agents/skills/` | `.agents/skills/` |
| Gemini CLI | `~/.gemini/skills/` 或 `~/.agents/skills/` | `.gemini/skills/` |
| Cursor | `~/.cursor/skills/` 或 `~/.agents/skills/` | `.cursor/skills/` |
| GitHub Copilot | `~/.copilot/skills/` 或 `~/.agents/skills/` | `.github/skills/` |
| OpenCode | `~/.config/opencode/skills/` 或 `~/.agents/skills/` | `.opencode/skills/` |

以 Claude Code 为例：

```bash
git clone --depth 1 https://github.com/yzddmr6/android-mainline-port ~/.claude/skills/android-mainline-port
```

更新时在该目录里执行 `git pull`。不要用 GitHub 的 "Download ZIP"，解压出来的目录名带 `-main`，和 skill 名不一致，部分助手会不识别。

### claude.ai 网页与桌面 App

1. 从 [Releases](https://github.com/yzddmr6/android-mainline-port/releases/latest) 下载 `android-mainline-port.zip`。
2. 在 Settings → Capabilities 打开 "Code execution and file creation"。
3. 打开 Customize → Skills，点 "+"，选择上传 skill，选中这个 zip。

### 让 AI 帮你装

把下面这段话发给你的 AI 编程助手：

```text
请帮我安装这个 Agent Skill：https://github.com/yzddmr6/android-mainline-port
1. 先判断你自己的用户级 skill 目录：Claude Code 是 ~/.claude/skills，Codex、Gemini CLI、Cursor、GitHub Copilot、OpenCode 可用 ~/.agents/skills，或以你自己文档里写的目录为准。
2. 执行 git clone --depth 1 https://github.com/yzddmr6/android-mainline-port <skill目录>/android-mainline-port；如果目录已存在，改为在里面执行 git pull --ff-only。
3. 目录名必须是 android-mainline-port，和 SKILL.md 里的 name 一致。
4. 确认目录里有 SKILL.md、references/、experience/，且 SKILL.md 开头写着 name: android-mainline-port。
5. 不要改动其他 skill 和设置。最后告诉我装在哪里，是否需要重启或重新加载。
```

### 确认装好了

- Claude Code：输入 `/android-mainline-port`（插件方式是 `/android-mainline-port:android-mainline-port`），能看到这个 skill 即可。
- 其他助手：直接问一句"我想把主线 Linux 移植到一台安卓平板上，先帮我评估"，助手会读取这份 skill 再回答。

## 使用示例

装好之后不需要特别的命令，像平常一样描述你的情况即可：

- "我有一台骁龙 8 Gen 3 的平板，BL 能解锁，帮我评估能不能跑主线 Linux + Ubuntu，大概要多久。"
- "`fastboot boot` 返回 OKAY，但屏幕一直停在 Fastboot 界面，pstore 也是空的，帮我排查。"
- "休眠一两秒就被唤醒，帮我查唤醒源。"
- "接手这个移植项目，先读文档，告诉我现在在哪一步、下一步做什么。"

## 内容结构

- **外层框架**：`SKILL.md` 和 `references/`。先做什么后做什么、工作纪律、技术选型、构建环境、怎么判断通过、Ubuntu 集成与解耦。相对稳定。
- **经验库**：`experience/`。按领域记录推荐路线、完成标准和具体的坑（现象、原因、怎么确认、怎么解决、怎么预防），随项目持续追加。

```
SKILL.md                         流程总览、顺序要点、工作纪律、选型原则、用户决策
references/
  process.md                     每一步的详细动作、产出、完成标准
  discipline.md                  文档要记什么、技术债、目录结构、署名、估算、子 agent
  build-environment.md           主机、工具链、ccache、OUT、根文件系统构建、缓存
  decisions.md                   关键技术决策：默认做法、条件与做法、机制、边界
  verification.md                状态词、证据、冷启动、测试自动化、性能预算
  ubuntu.md                      Ubuntu 分层与解耦、包拆分建议、启动钩子、换发行版
experience/
  README.md                      条目格式和维护规则
  patterns.md                    十一类问题模式与通用处理
  bootloader-and-boot.md         ABL、启动格式、首次启动、调试通道、恢复
  kernel-and-resources.md        内建与模块、资源回收、PMIC 电压、设备树数值
  display.md                     simpledrm 与原生接管、DSC、熄屏亮屏、背光
  gpu.md                         Adreno、固件、自编 Mesa、新 GPU 的 Mesa 缺陷
  touch-and-input.md             触控（TDDI）、按键、键盘配件、指纹
  audio.md                       AudioReach、功放、麦克风、UCM、音量
  wireless.md                    Wi-Fi、蓝牙
  power-and-time.md              电量、充电、休眠、系统时间
  sensors-camera-media.md        传感器与旋转、相机、视频硬解
  usb-and-storage.md             USB、外接显示、分区、双系统、刷写、安装
  ubuntu-userspace.md            Ubuntu 打包、dracut、snap、GNOME 默认值
  build-and-release.md           构建流水线、缓存、可复现性、发布
  testing-and-process.md         判断标准、自动化测试、协作、子 agent
```

移植流程分九步：

| 步 | 做什么 |
|---|---|
| 0 立项 | 资料池、复用调研、硬件清单、选型、估算，和用户确认决策 |
| 1 构建环境 | 工具链、ccache、长期 OUT，原样构建一次社区参考内核 |
| 2 取证与备份 | root，备份并校验，取出运行时设备树、固件、电压、显示参数 |
| 3 首次启动 | 搞清 bootloader，最小设备树，USB 网络 shell，超时自动回原系统 |
| 4 点亮与内存桌面 | 显示、触控、GPU，内存里跑起 GNOME，搭好桌面自动化测试 |
| 5 安装到存储 | 分区、双系统、启动切换、调试通道，冷启动验证 |
| 6 外设 | 一次一个：查复用、对照原厂、单独测试、真机、合并、回归 |
| 7 发布 | 功能冻结、分层打包、按公开文档从零安装验收 |
| 8 维护 | 上游合入就删本地实现，还技术债，每次更新冷启动验证 |

## 适用范围与时效

- 高通平台写得最具体；其他 SoC 的流程和纪律相同，硬件细节需要自己调研。
- Ubuntu 相关部分换发行版时按 `references/ubuntu.md` 最后一节调整。
- 内容基于 2026 年的上游状态。版本号、接口名、上游进度有时效性，使用前请核对；上游已经解决的问题，直接用更简单的办法。
- 正文是中文，各家 AI 都能正常阅读和使用。

## ⚠️ 风险提示

- 解锁 bootloader 会清空设备数据，可能影响保修；刷写分区操作不当可能导致设备无法启动。
- 动手前按 skill 里的要求完整备份分区并校验，确认有恢复手段。
- 本仓库只提供方法和经验，不提供刷机工具、镜像或固件。内容按"原样"提供，使用风险自负。

## 贡献

欢迎通过 Issue 和 Pull Request 补充经验：

- 在 `experience/` 对应文件里按 `experience/README.md` 的格式追加条目。
- 只写确认了根因的问题，写清现象、原因、怎么确认、怎么解决、适用范围。
- 不写设备型号、序列号、地址、校准数据、个人信息；芯片型号、驱动名、错误码可以写，它们是搜索关键字。

## 许可

[MIT](LICENSE)
