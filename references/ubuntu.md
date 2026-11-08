# Ubuntu 集成与解耦

本 skill 的经验来自 Ubuntu 26.04 LTS + GNOME（Wayland、GDM、dracut、snap）。下面是推荐的做法，是实际项目最终收敛到的结构；具体拆多细、叫什么名字，按项目实际情况调整。

## 1. 目标：发行版可替换

```
┌─────────────────────────────────────────────┐
│ 桌面策略层：GNOME 默认值、缩放、电源键、     │  ← 换桌面或发行版时重写
│ 首启服务、浏览器替换                         │
├─────────────────────────────────────────────┤
│ 集成层：启动镜像钩子、dracut 配置、          │  ← 换发行版时改钩子位置
│ 关闭 flash-kernel、切换工具                   │
├─────────────────────────────────────────────┤
│ 设备层：udev、设备服务、UCM 与拓扑、传感器栈、│  ← 基本不变
│ 充电认证、本机数据导入                        │
├─────────────────────────────────────────────┤
│ 硬件层：内核 Image、设备树、模块、固件         │  ← 不变
├─────────────────────────────────────────────┤
│ 发行版：官方根文件系统，不修改，只加包         │
└─────────────────────────────────────────────┘
```

规则：
- 硬件相关的部分不依赖特定的包管理行为、显示管理器、固定账户或桌面。
- 不修改发行版文件；必须替换时用 `dpkg-divert`，卸载即恢复。
- 本机唯一数据（校准、MAC、蓝牙地址、传感器配置）不进任何包，安装时从本机数据分区导入，缺失时明确报错。
- 不做一个"大管家"包或常驻服务统管一切。

## 2. 包拆分建议

按"谁维护、多久更新一次、属于哪一层"拆。`<dev>` 换成设备代号：

| 包 | 内容 | 层 | 说明 |
|---|---|---|---|
| `linux-image-<版本>-<dev><ABI>` | `/boot/vmlinuz-<release>`、`/lib/modules/<release>/`（完整模块目录）、设备树放 `/usr/lib/linux-image-<release>/` | 硬件 | 布局照上游 `scripts/package/builddeb`；名字以 `linux-image` 开头，APT 才会保护它不被自动删除 |
| `<dev>-kernel` | 只依赖当前的内核包 | 硬件 | 升级和回滚只改它 |
| `<dev>-firmware` | 固件文件放 `/lib/firmware/updates/`，附逐文件清单（来源、路径、哈希） | 硬件 | 不和 `linux-firmware` 抢路径；冲突用 `dpkg-divert` |
| `<dev>-device-support` | udev 规则、设备服务（触控用户态、充电、槽成功标记）、启动镜像钩子、dracut 模块和配置、`/etc/flash-kernel/machine`、切换工具、调试通道（默认关闭）、polkit 规则 | 设备 + 集成 | 体积小，变化最频繁 |
| `<dev>-audio`（可并入 device-support） | UCM、ALSA 拓扑文件 | 设备 | |
| `<dev>-sensors` | 远端处理器 RPC 守护进程、传感器库、替换版 iio-sensor-proxy（`dpkg-divert`） | 设备 | 发行版版本支持后删除 |
| `<dev>-mesa` | 新 GPU 需要的 Mesa，装在独立前缀，经 `/etc/ld.so.conf.d/` 成为系统默认（不只靠环境变量） | 设备 | 发行版 Mesa 支持后删除 |
| `<dev>-snap-graphics` | `gpu-2404` 提供者 snap 和自动连接服务 | 桌面 | 只有 snap 发行版需要 |
| `<dev>-desktop` | GNOME 默认值（gschema override）、`/etc/xdg/monitors.xml` 缩放、电源键行为、chrony 和 fake-hwclock 配置、zram、PipeWire 上混、console-setup 修正、浏览器替换 | 桌面策略 | 换桌面时重写 |
| `<dev>-debug` | 诊断工具、调试 shell | 可选 | 默认不装 |
| `<dev>-device` | 元包，依赖以上基础包 | — | 用户只需要装它 |

最少也要把**内核、固件、桌面策略**三者分开：它们的更新频率和归属完全不同。

## 3. 内核包

- 版本名：`<上游版本>-<dev><ABI>`，例如 `7.2.6-foo3`。**每个打包的构建都升一次 ABI**，包括只改设备树或工具链的构建，从不复用、从不回退。关闭 `MODVERSIONS` 时，模块 ABI 破坏无法被检测，复用版本名会让旧模块被新内核加载。
- 维护一个 ABI 账本：版本名、内核提交、`.config` 的哈希。打包脚本只打包账本里登记过的组合。
- 不要用 `+` 后缀（`linux-version sort` 会排错）；关闭 `LOCALVERSION_AUTO`。
- 打包脚本找版本标签时限定匹配模式，避免被项目发布标签截住。

## 4. 启动镜像钩子

Android 设备的启动镜像要由项目自己生成：

- 在 `/etc/kernel/postinst.d/zz-<dev>-boot`、`/etc/kernel/postrm.d/` 和 `/etc/initramfs/post-update.d/` 放钩子：内核或 initramfs 变化时，生成新的启动镜像集（按本机格式），存到切换存储里，并（可选）写入活动槽、读回校验。
- 保留上一版作为回滚集。
- 钩子在前置文件（签名密钥、分区映射等）缺失时打印提示并**退出 0**，不写槽、不删除镜像集。任何包更新都可能触发 initramfs 重建（例如用户在首次设置向导里改键盘布局），钩子出错会让 dpkg 半配置。
- 钩子能识别自己运行在构建 chroot 里，此时不写设备。
- `/etc/flash-kernel/machine` 写 `none`，关掉发行版自带的启动部署工具。

## 5. initramfs（dracut）

```
# /etc/dracut.conf.d/90-<dev>.conf
hostonly="no"            # 在 QEMU/chroot 构建时，hostonly 会按构建机挑模块
hostonly_cmdline="no"
add_dracutmodules+=" <dev> "   # 早期需要的设备逻辑（例如 hypervisor 节点遮蔽）
filesystems+=" ext4 "
```

- 启动和恢复需要的驱动必须在内核里或在 initramfs 里；固件在早期就要用的（例如触控、显示）放进 initramfs。
- `hostonly="no"` 会打包全部模块，体积和生成时间都大：核对启动分区余量（例如 vendor_boot）；生成慢的问题靠"在设备上生成"解决，见 `build-environment.md`。
- 注意 Ubuntu 的 dracut 用 3cpio，压缩等级参数可能不生效。

另一种选择是自写 BusyBox initramfs（早期调试和恢复更可控，但要自己维护），见 `decisions.md`。

## 6. 根文件系统组成

1. 官方 ARM64 Desktop ISO 的 squashfs 层（minimal + standard），地址和哈希固定。
2. 安装发行版更新和项目包（`<dev>-device` 等）。
3. purge 通用内核、HWE 元包、headers、`kdump-tools`；再装项目内核，让 initramfs 只生成一次。
4. 安装 Mozilla APT 源的 Firefox deb（加源、密钥、pin），首启服务在 snap seeding 完成后移除 snap 版。
5. fstab 按 PARTLABEL；不预置账户（走 GNOME 首次设置）；系统默认语言按目标用户设置。
6. 清理 `/var/lib/apt/lists`、构建机痕迹；`fstrim` 后交付（tar 或只编码已用块）。

## 7. 内核配置对齐发行版

首版前把内核配置与 Ubuntu 通用内核对比，至少补齐：`UHID`、`PSI`、zram、nf_tables 全套、WireGuard、Yama、Landlock、NTFS3、CIFS、ISO9660、dm-crypt、`SND_SEQUENCER`、`FW_LOADER_COMPRESS`、SquashFS XZ/LZO/xattr、FUSE、AppArmor（snap 需要）。要跑 Waydroid 一类的 Android 容器，还要 Binder、legacy iptables 和 xtables/nftables 兼容、策略路由。按功能做成配置片段，每项单独提交，方便回退。

## 8. snap 与图形

- 严格沙箱 snap 的 GPU 用户态来自 `gpu-2404` 接口的提供者，不是系统 Mesa。新 GPU 不被 `mesa-2404` 支持时：以原版 `mesa-2404` 的完整内容为底，只替换 Mesa 库，做一个提供者 snap；首启服务安装并连接所有带 `gpu-2404` 插头的 snap；离线验证要模拟消费者加载。core26 的 snap 用 `gpu-2604`。
- 浏览器优先用 deb（直接用系统 Mesa）。

## 9. GNOME 默认值

见 `../experience/ubuntu-userspace.md` 的具体条目，要点：
- 缩放放 `/etc/xdg/monitors.xml`（首次设置向导和 GDM 也读）。
- 电源键：挂起未完成时设为无动作或保留对话框，logind 忽略；挂起完成后改为熄屏锁屏。
- 合盖：logind 处理 `SW_LID`；接外部电源时只锁屏（`HandleLidSwitchExternalPower=lock`）。
- 没有挂起时禁用挂起（`sleep.conf.d`）并把空闲动作设为无。
- Chassis：遮蔽 hypervisor 节点后系统才不会判成虚拟机；`tablet` 会让 GNOME 隐藏电源键设置，需要这个设置时设 `convertible`（副作用见 `../experience/touch-and-input.md`）。设备身份判断不要依赖 Chassis 的值。
- 时间：RTC 偏移方案落地前，fake-hwclock + chrony 放宽配置（过渡，见 `decisions.md` 的"系统时间"）。
- 电源模式：Ubuntu 26.04 用 tuned-ppd 取代 power-profiles-daemon，没有 `powerprofilesctl`，脚本经 D-Bus（`net.hadess.PowerProfiles` 的 `ActiveProfile`）切换。
- PipeWire 多扬声器上混。
- 清理首次设置向导写入的非法键盘布局。

## 10. 调试通道（产品）

- systemd socket 激活的 SSH，绑定固定的 USB 网段地址并 `FreeBind=yes`，用 `IPAddressAllow`/`IPAddressDeny` 只接受 USB 网段；不要用 `BindToDevice`。
- 默认关闭，安装时临时开启，命令行或安装参数显式打开；开启状态重启后保持。
- 冷启动验证开启和关闭两种状态。

## 11. 发布与升级

- 发布附件：启动镜像包、工具包（切换、恢复、安装）、软件包包、根文件系统分卷、校验和、清单。
- 用户升级：下载新的软件包，`apt install ./*.deb`，钩子自动重建启动镜像集，保留回滚集；以后可以提供 APT 仓库。
- 安装文档写清：本机数据导入、首次设置、调试通道开关、回到 Android 的方法、风险。

## 12. 换成其他发行版时要改什么

| 部分 | Ubuntu/Debian | Fedora | Arch | postmarketOS |
|---|---|---|---|---|
| 内核和固件 | deb | rpm（内容不变） | pkg（内容不变） | apk（内容不变） |
| 启动镜像钩子 | `/etc/kernel/postinst.d` | `kernel-install` 插件 | pacman hook | mkinitfs / boot-deploy |
| initramfs | dracut | dracut（原生） | mkinitcpio 或 dracut | mkinitfs |
| 替换发行版文件 | `dpkg-divert` | 尽量避免；用 drop-in | 尽量避免 | 包覆盖 |
| 沙箱应用 GPU | snap `gpu-2404` | flatpak 的 GL 扩展 | flatpak | 原生包为主 |
| 桌面策略 | GNOME gschema override | 同 GNOME | 按桌面 | Phosh/GNOME/Plasma Mobile |

硬件层和设备层基本不用改；集成层和桌面策略层按上表重写。
