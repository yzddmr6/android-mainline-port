# Bootloader、首次启动、调试通道与恢复

## 推荐路线（高通 ABL 设备）

1. **只读取证**：在原厂系统里取运行时设备树 `/sys/firmware/fdt`、各启动分区的镜像头版本、`fastboot getvar all`、`avbtool info_image` 看 vbmeta 链。
2. **离线复现 bootloader 的设备树处理**：用 libufdt（ABL 实际用的实现）把原厂 dtbo 叠加到候选设备树上，先拿原厂设备树做正对照。
3. **打包期注入符号表**：把 dtbo 实际引用的符号（加上 hypervisor 运行时 overlay 需要的符号，例如 `arch_timer`）导出到一个 `status = "disabled"` 的容器节点里。内核 DTS 保持上游风格，不在源码里加这些东西。
4. **第一次上机用时标探针**：`fastboot boot` 一个只打印带时间标记、设 `panic=N`、到时间 `reboot` 的镜像，证明"交接了、跑了多久、能自动回去"。
5. **最小 initramfs**：BusyBox，所有块设备设只读并逐个复核，起 USB NCM + DHCP + telnet/SSH，计时退出。
6. **RAM 根**：官方发行版根打成 SquashFS，HTTP 经 USB 网络传入，OverlayFS，`switch_root`。
7. **持久启动**：按本机实测决定镜像格式（常见是与原厂同构的 boot/init_boot/vendor_boot 三件套加自签 AVB 尾部），根分区按 PARTLABEL 挂载。

格式结论每台设备都要实测，不要照搬别的设备，哪怕是同一 SoC。

## 完成标准

- RAM 阶段：USB 在交接后约 1 秒内断开再以新身份出现；能双向执行命令、取 dmesg；到时间自动回原系统。
- 持久阶段：启动分区按精确字节读回一致；运行时 cmdline 和内核版本与清单一致；运行时设备树里注入的容器是 disabled；`/sys/kernel/debug/devices_deferred` 为空或可解释；sync_state 没有长期挂起的提供者。
- 首启检查清单：`/proc/device-tree/compatible` 和 model、`systemd-detect-virt`、`hostnamectl` 的 Chassis、最终 `.config` 的关键项、`devices_deferred`、sync_state 状态。

## 坑

### 主线设备树没有 `__symbols__`，ABL 静默拒绝，`fastboot boot` 却返回 OKAY
标记：【高频】【耗时】
- 现象：`fastboot boot` 显示 `Booting OKAY`，之后没有 USB 网络、没有任何日志；有时看起来像"几秒后回到 Fastboot"。
- 原因：ABL 会把 dtbo 分区（以及不在任何分区里的 hypervisor 运行时 overlay）叠加到你的设备树上，叠加需要符号表；主线设备树默认不带，叠加失败后 ABL 不回退也不报错。
- 怎么确认：取原厂系统的 `/sys/firmware/fdt` 和自己打包的设备树对比；用 libufdt 离线叠加并以原厂设备树做正对照。
- 怎么解决：打包时导出 dtbo 引用的全部符号（或所有 base DTB 符号的并集），都指向一个 disabled 容器里的占位节点；hypervisor overlay 引用的符号（如 `arch_timer`）也要导出。
- 提前预防：首启前完成离线叠加复现；判断交接用 `udevadm monitor` 看 USB 事件，不看 Fastboot 返回值。
- 适用范围：高通 ABL（SM8450 之后普遍）；2026。

### 用 dtc 的 fdtoverlay 离线复现，连原厂设备树都叠加失败
- 现象：离线模拟得出"原厂也不行"的荒谬结论。
- 原因：ABL 用的是 libufdt，行为与 dtc 的实现不同。
- 怎么解决：用目标系统实际使用的实现复现；永远先跑一个已知会成功的正对照。
- 适用范围：通用方法。

### 占位节点放错位置：保留内存被映射到物理地址 0 导致崩溃，或成为"假消费者"让资源永不释放
标记：【耗时】【致命】
- 现象一：ABL 交接后静默挂死，earlycon 上看到崩在 `persistent_ram_new`。
- 现象二：rpmhpd、NoC、GCC 的 sync_state 一直挂起，SoC 一直跑在最高电压，idle 状态从未进入；这个问题会掩盖其他电源问题长达数周。
- 原因：占位节点放在根下或带了 `#address-cells`/`#size-cells`/`ranges`，叠加进来的保留内存被翻译到地址 0；或者叠加后占位节点带上了 compatible 和 `status = "ok"`，成为永远不绑定的设备，fw_devlink 把它的参数当成 phandle 解析，正好命中电源提供者。
- 怎么解决：占位节点放进 `status = "disabled"`、无 compatible、无地址属性的容器；`of/platform` 只遍历 `/reserved-memory` 的直接子节点。
- 提前预防：首启当天检查 `state_synced` 和 sync_state 挂起列表；解除挂起后整个电源生命周期要重测。
- 适用范围：高通 ABL；2026。

### bootloader 改写根节点的 compatible 和 model，所有按它判断设备的代码都失效
标记：【高频】
- 现象：PD mapper 打印 `no support for the platform`，声卡和远端处理器服务不出现；显示驱动的 UBWC 配置认不出 SoC，没有 `card0`；音频 quirk 永远选不中；安装器报"设备型号不符"。
- 原因：ABL 把 dtbo 根上的 compatible、model、msm-id、board-id 覆盖到运行时设备树根上，变成平台通用名（例如 `qcom,<平台>-mtp`）。
- 怎么确认：首启时 `cat /proc/device-tree/compatible` 与源码对比。
- 怎么解决：内核早期恢复根身份（推荐，一处修复所有消费者）；或在各驱动匹配表加入被改写后的名字；quirk 改用不受影响的键（例如 ASoC 声卡名）；设备身份改用 Fastboot product、serial 和运行时值。
- 提前预防：凡是 bootloader 会改写的属性，都不能用来选择 quirk 或判定设备身份。
- 适用范围：高通 ABL；2026。

### hypervisor 节点让 systemd 把平板判成虚拟机，短按电源键直接关机
标记：【高频】
- 现象：`systemd-detect-virt` 输出 `vm-other`，`hostnamectl` 的 Chassis 是 `vm`；登录界面短按电源键关机；GNOME 设置里没有电源键选项；带 `ConditionVirtualization=no` 的服务被跳过。
- 原因：ABL 注入 `/hypervisor`（Gunyah）节点，systemd 据此判定虚拟化。
- 怎么解决：在 systemd 判定前遮住用户态看到的那份节点（initramfs 里或 sysinit 早期用空目录 bind-mount 覆盖 `/sys/firmware/devicetree/base/hypervisor`）；不要从内核的设备树里删掉它，以后要用 hypervisor 的虚拟机管理接口时内核还需要这些数据。Chassis 按桌面策略设（见 `../references/ubuntu.md`）。
- 提前预防：首启检查清单里固定检查 `systemd-detect-virt`、Chassis，以及遮罩是否在 initramfs 和按需加载的模块之前生效。
- 适用范围：高通 SM8450 之后带 Gunyah 的平台；2026。

### 内核命令行来自设备树 `/chosen/bootargs`，而且有长度上限，超长部分被静默截断
标记：【高频】
- 现象：改镜像头里的 cmdline 不生效；或者命令行末尾的模块黑名单失效，被屏蔽的驱动照样加载。
- 原因：ABL 读设备树的 bootargs 并原地追加几百字节自己的参数（可能含 `block2mtd` 等危险项），总长度有上限（常见约 512 字节）。
- 怎么解决：构建时校验镜像头和设备树的 cmdline 一致、断言内容，并拒绝超长；黑名单不要放命令行。
- 适用范围：高通 ABL；2026。

### `fastboot boot` 和冷启动需要的镜像格式不同
- 现象：v4 镜像 `fastboot boot` 后 rdinit 从未执行，早期崩溃；v2 能 RAM 启动但刷进分区冷启动直接进 Fastboot。
- 原因：GKI 设备上 ABL 对 v4 boot 镜像的 ramdisk 字段视而不见（ramdisk 在 init_boot/vendor_boot）；冷启动时 vbmeta 链要求 boot/vendor_boot 带 AVB 尾部。
- 怎么解决：RAM 测试用 v2（kernel + ramdisk + DTB）；持久启动按原厂结构生成三件套，自签 AVB 尾部（测试密钥在解锁状态下接受），rollback index 照原厂；镜像头字段和 AVB 属性复刻原厂。
- 适用范围：高通 GKI 设备；2026。

### 序列号只在 `/proc/bootconfig`，读 `/proc/cmdline` 的工具全部报身份不符
- 现象：系统内切换工具、内核更新钩子报 serial mismatch，拒绝执行。
- 原因：有的 ABL 对主线内核把 `androidboot.serialno` 只放进 bootconfig，有的放在 cmdline，有的都不给。
- 怎么解决：读取顺序 cmdline → bootconfig → 失败时明确报错；加回归测试。
- 适用范围：高通 ABL；2026。

### `fastboot boot` 返回 OKAY 后停在 Fastboot、pstore 为空：先查设备树身份
标记：【耗时】
- 现象：Sending 和 Booting 都返回 OKAY，没有 USB 网络，几十秒后设备仍在 Fastboot，pstore 为空。
- 原因：ABL 在跳到内核之前按设备树根上的 `qcom,msm-id`、`qcom,board-id` 选择和校验设备树，身份不匹配就不跳转，内核侧不会留下任何证据。打包工具从"原厂基础设备树"复制这些值，换了一份原厂基础设备树就会悄悄打出一个 ABL 不接受的镜像。
- 怎么确认：反编译打包后的设备树，与历史上能启动的设备树逐项对比根身份。
- 怎么解决：打包时对身份做准入检查，与本机已知能启动的值不一致就拒绝。
- 提前预防：不要用"镜像超过 boot 分区大小"解释这个现象，换一个更小的镜像再测一次就能排除；`fastboot boot` 的大小限制是 `max-download-size`（常见几百 MiB），远大于 boot 分区。
- 适用范围：高通 ABL；2026。

### 用候选内核测试已安装的系统，不要直接装包
标记：【高频】【致命】
- 现象：直接 `dpkg -i` 候选内核包，钩子重建启动镜像集并写入活动槽；候选有问题时设备反复重启。更糟的组合：GPU 挂死（`timeout waiting for GMU OOB set GPU_SET`）后发出普通 `reboot`，关机卡在 DRM 收尾，USB 通道已断，回退槽又在之前重装同版本包时被换成了同一个候选，设备只能等人长按电源键。
- 怎么解决：在设备上 `dpkg-deb -x` 解出模块目录，`depmod`，在设备上 `dracut --kver` 生成 initramfs（不写槽；设备上的 dracut 可能没有 `--kconfig`，配置放 `/lib/modules/<rel>/config`）；取回内核、设备树（核对身份）、initramfs，用 v2 格式打 RAM 镜像，命令行加 `softlockup_panic=1 hung_task_panic=1 oops=panic`、`panic=N`，让挂死变成重启回已安装系统；回程机制随镜像生效；结束后删除暂存的文件。RAM 测试通过后才走正式升级路径持久安装。强制重启的写法见 `../references/decisions.md` 的"候选内核的测试方式"。
- 提前预防：`panic_on_oops` 只是 sysctl，命令行要写 `oops=panic`；`hung_task_panic`、`softlockup_panic` 要内核编进对应检测器才存在，上机前确认 `/proc/sys/kernel/hung_task_panic` 存在。这些约束写进每一份设备任务说明。
- 适用范围：自管启动镜像的设备；2026。

### Fastboot 长时间空闲，或后台仍在写数据时，`fastboot boot` 卡住、设备不再应答
标记：【致命】
- 现象：屏幕停在 Fastboot，`getvar` 全无回应，只能长按电源键。
- 原因：刷写零填充镜像时 ABL 先回 OKAY 再在后台慢速写零（可达几十分钟），期间后续命令排队或不应答；长时间空闲后的状态也不可靠。
- 怎么解决：不发零填充镜像（见 `usb-and-storage.md`）；进入 Fastboot 后立即执行下一步，两步之间回到系统等待。
- 适用范围：高通 ABL；2026。

### 调试通道建立之前访问外设寄存器或时钟调试接口，整机挂死且无法远程恢复
标记：【高频】【致命】
- 现象：启动后没有 USB 网络、黑屏，长按键才能恢复；同一错误在复盘后又复发。
- 原因：initramfs 或早期服务在 USB 起来前读显示寄存器、`clk_summary`、全部 LUN 的 GPT，碰到未上电或受保护的硬件。
- 怎么解决：默认启动路径里删掉这些探测；需要时在通道建立后再做。
- 提前预防：构建时静态检查默认启动路径里没有 `/dev/mem`、`devmem`、clk debugfs 访问。
- 适用范围：通用；2026。

### 用 `/dev/mem` 读显示控制器寄存器触发安全世界致命错误，SoC 复位
- 现象：只读取证却导致设备重启。
- 原因：读到了新一代 DPU 已移除的寄存器区（例如 MDP top 的某段），触发 TZ fatal。
- 怎么解决：显示寄存器的"标准答案"用原厂驱动自己的 debugfs 导出；`/dev/mem` 只读验证过的地址。
- 适用范围：高通；2026。

### 静态 BusyBox 的程序名或路径被改，引用它的服务退出码 127
标记：【高频】
- 现象：交接后白屏、调试通道消失，日志里服务 `status=127`。
- 原因：服务按旧名调用 applet。
- 怎么解决：BusyBox 保持原名，用 `/usr/local/libexec/<项目>/busybox <applet>` 方式调用；交接前清理旧服务。
- 适用范围：通用。

### 在原厂内核上拼最小 RAM 探针，几轮都拿不到 USB
- 现象：原厂内核加自制 ramdisk 启动后 USB 不枚举，反复尝试没有新信息。
- 原因：原厂内核的 USB 依赖显示、相机等时钟控制器先就绪；v4 镜像 ramdisk 不交付等。
- 怎么解决：不要走这条路，直接 root 原厂系统做备份和取证，主线内核从时标探针开始。
- 适用范围：高通 GKI 设备；2026。

### switch_root 交接时的连锁问题
- 现象：telnet 接受连接后立刻断开；端口被占用；网卡名变成 `enx*`；固件请求在新根里失败，或两个根里都有固件仍偶发 `-2`；桌面运行一段时间后被旧看门狗杀掉。
- 原因：initramfs 的孤儿进程仍占着端口；udev 给 USB 网卡改名；预建的 `/dev/console` 让条件挂载 devtmpfs 被跳过，没有 `/dev/ptmx`；固件只放在 initramfs；BusyBox `switch_root` 先删 initramfs 再接上新根，异步的固件请求落在中间的空窗；initramfs 的计时器没有撤销。
- 怎么解决：切根前结束旧进程、在新根内重建通道；无条件挂载 devtmpfs；用 `.link` 文件按 MAC 固定网卡名；固件放在根文件系统，交接期间保证固件路径连续可见（例如先移动新根再 `chroot`）；交接时新旧保护要重叠、旧保护在新保护生效后再撤。
- 适用范围：自写 initramfs 时通用。

### 恢复路径本身会卡住
标记：【致命】
- 现象：计时到期后没有回到 Fastboot 或原系统，只能人工按键。
- 原因：到期路径 `sleep; sync; reboot` 中 `sync` 没有时限（`reboot -f` 也会 sync）；硬件看门狗从未实测。
- 怎么解决：先写标记，`sync` 限时，`reboot -f -n`；看门狗做受控失喂测试，确认真的会复位。
- 提前预防：恢复路径每一步都有时限，并在最终组合上实测。
- 适用范围：通用。

### systemd 默认会去打开 hypervisor 看门狗，超时值超过硬件上限
- 现象：普通重启或关机过程中意外复位。
- 原因：systemd 默认 `RebootWatchdogSec=10min`，而 Gunyah 看门狗上限约 32 秒；`wdctl` 读不到 sysfs 时会打开设备。
- 怎么解决：drop-in 设 `RuntimeWatchdogSec=off`、`RebootWatchdogSec=off`，开启 `WATCHDOG_SYSFS`；不要加 MMIO 看门狗节点（会让 SCM 放弃 hypervisor 路径）。
- 适用范围：高通 Gunyah 平台；2026。

### 软件重启进不了 Fastboot，被误判为"bootloader 不支持"
标记：【耗时】
- 现象：`reboot bootloader` 后回到普通系统。
- 原因：重启原因存在 PMIC 的 SDAM 单元里，设备树定义是 1 字节 7 位，主线 `nvmem-reboot-mode` 无条件写 4 字节，NVMEM 层直接返回 `-EINVAL`。
- 怎么解决：按单元实际宽度写入；同时确认 `NVMEM_REBOOT_MODE`、`NVMEM_SPMI_SDAM` 内建。
- 提前预防：宣布"bootloader 不支持"前，先读通用驱动源码和设备树里该单元的定义。
- 适用范围：高通；2026（上游可能已修，先核对）。

### 插着电时设备不断自动开机进入充电模式
- 现象：关机后反复自启，或以 `androidboot.mode=charger` 进入你的系统。
- 原因：bootloader 检测到外电会自动开机。
- 怎么解决：initramfs 第一个分支识别充电模式，进入只读低功耗等待，长按电源键再继续；不要用关机应对（插着电会再次开机）。
- 适用范围：通用。

### 早期日志不可靠
- 现象：pstore 内容损坏或为空；内核日志级别被改；PID1 报 `unable to open an initial console` 后退出；同一系列的另一种内存容量的机型一开机就复位。
- 原因：ramoops 地址不是固件保留的安全区，或固定地址只在开发机的内存容量下落在 RAM 里；ABL 追加 `loglevel=6`；ABL 追加 `console=ttynull` 而内核没开 `CONFIG_NULL_TTY`；`console=tty0` 在 initcall 前注销 bootconsole；部分配置下裸 `earlycon` 不生效；`/sys/fs/pstore` 为空可能只是 systemd-pstore 已经把文件搬到 `/var/lib/systemd/pstore`。健康系统的受控 panic 下 pstore 也可能损坏。
- 怎么解决：ramoops 用原厂设备树里的固定保留区，没有就用运行时 netconsole；固定地址的保留区核对每一种受支持的内存容量；测试环境和真机的控制台参数逐条对比；崩溃证据优先从实时通道取，损坏的 pstore 只取完整的行。
- 适用范围：通用。

### 产品命令行带着串口控制台，错误风暴时外设中断被饿死
标记：【耗时】
- 现象：蓝牙 `command 0x202a tx timeout`、`Reading QCA version information failed (-110)`，同时有 `/dev/kmsg buffer overrun` 和大量 printk；控制器随后消失。
- 原因：legacy 串口控制台在调用者上下文里关中断同步输出，低速串口在错误风暴下长时间占住 CPU，同一 CPU 上的外设中断（例如蓝牙 UART）得不到处理。
- 怎么确认：`/proc/consoles` 看串口是否是实际生效的控制台；对照外设 IRQ 的 `effective_affinity_list` 和日志时间；用等量日志负载做对照。
- 怎么解决：产品命令行不带串口控制台，调试时显式打开；同时修掉风暴源。命令行里已有 `console=` 时只删设备树的 `stdout-path` 不起作用。不要靠延长超时或重试掩盖。
- 适用范围：legacy（非 nbcon）控制台；2026。

## 未解决

- 早期硬卡死（看门狗还没被武装之前）如何无人值守恢复：现有方案依赖内建硬件看门狗和超时重启，更早的阶段仍要人工按键。
- Fastboot 长时间空闲后不响应的确切机制。
