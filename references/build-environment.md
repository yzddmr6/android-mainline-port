# 构建环境与效率

目标：从零开始一个新项目时，环境一次配好，日常迭代以分钟计，不在构建上浪费时间。

核心原则：**先测量，再优化**。任何超过 5 分钟的步骤，第一次出现就分段计时；不变的部分按输入哈希缓存；给每个阶段定时间预算；每次发布后回顾一次流水线。

## 1. 主机

| 项 | 建议 | 说明 |
|---|---|---|
| 系统 | Linux x86_64 或 ARM64 | 有原生 ARM64 机器（或 CI 的 arm64 runner）时，根文件系统和 Mesa 构建快一个数量级 |
| CPU / 内存 | 8 核以上 / 16 GB 以上 | 内核全量模块构建、Mesa 编译、QEMU 同时跑时内存很紧 |
| 磁盘 | 300 GB 以上可用空间 | 根文件系统镜像、多个 OUT、备份、发布物很快就会占满；剩余低于约 30 GB 时构建会半途失败 |
| 权限 | 构建要能保留数字属主、权限、xattr、能力位和设备节点 | 只映射一个 UID 的 `unshare -r` 做不出正确的根文件系统；用真正的 root（只允许固定构建脚本免密 sudo），或经过验证的无特权构建工具 |

主机密码等凭据只通过环境变量传给脚本，不写进文件或日志。

## 2. 工具链

- 交叉编译器：发行版的 `gcc-aarch64-linux-gnu`（或 LLVM），版本固定并记录。
- 内核构建依赖：make、bc、bison、flex、libssl-dev、libelf-dev、cpio、kmod、dtc、python3。
- 镜像工具：AOSP `mkbootimg`/`avbtool`（固定版本）、`img2simg`、e2fsprogs、GNU tar。
- 设备工具：`android-tools`（adb、fastboot）、`sgdisk`。
- QEMU：`qemu-system-aarch64`（只用于必须验证 init 行为的测试）、`qemu-user-static` + binfmt（chroot 构建）。

## 3. ccache

ccache 是内核迭代速度的关键。先按实测估容量：一次带约 1300 个模块的 arm64 内核完整构建（无调试信息），在 ccache 里压缩后约 0.3 GB，其中编译结果约 0.1 GB，直接模式的 manifest 约 0.2 GB（压缩率约 5 倍）。manifest 会随"同一源码在不同路径下构建"不断变大，所以不设 `base_dir`、经常换 OUT 路径时，缓存涨得快、命中率低。

容量建议：只做内核时 10 GB 足够（能保留约 30 次完整构建）；同时用 ccache 编 Mesa 等大型 C++ 项目时再加到 20 GB 左右。上限太小（例如 4 GB）时会被反复清理，命中率掉到一半以下。

建议配置（`~/.config/ccache/ccache.conf`）：

```ini
max_size = 10G
# 让不同路径下的源码和 OUT 共享缓存（配合 -ffile-prefix-map）
base_dir = /path/to/workspace
hash_dir = false
# 以编译器内容判断是否同一编译器，换路径或升级时自动失效
compiler_check = content
```

内核构建时：

```bash
make O=$OUT ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
     CC="ccache aarch64-linux-gnu-gcc" -j$(nproc) Image dtbs modules
```

要点：
- 固定 `KBUILD_BUILD_TIMESTAMP`、`KBUILD_BUILD_USER`、`KBUILD_BUILD_HOST`，并给 `KCFLAGS`、`KAFLAGS` 都加 `-ffile-prefix-map=$SRC=/linux -ffile-prefix-map=$OUT=/kbuild`。这既让产物可复现，也让缓存不受路径影响。
- 关掉 `LOCALVERSION_AUTO`，版本号由打包时的 ABI 决定，避免每次提交都改版本字符串导致大面积重编。
- 每次构建前 `ccache -z`，构建后 `ccache -s` 看命中率；命中率突然下降，先查是不是换了 OUT 路径、编译参数或编译器。
- CI 里缓存 ccache 目录（例如 GitHub Actions 的 cache，按内核分支和配置做键）。

## 4. 内核 OUT 目录

- **每种配置一个长期 OUT**（例如产品配置、RAM 诊断配置），不要为了"干净"每次新开路径：换路径等于一次冷构建（几十分钟），增量只要一两分钟。正式发布前做一次干净构建即可。
- 用 `flock` 保证同一 OUT 同一时间只有一个构建；在 OUT 里记录构建输入（源码路径、提交、工具链、编译参数），不一致就拒绝构建。
- 带模块和不带模块的构建不要混在同一个 OUT。
- 只构建需要的目标：只改设备树时 `make dtbs`；只改一个驱动时 `make drivers/.../foo.ko` 或 `make M=`；不要每次 `modules_install` 到新目录。
- 工作 OUT 加锁后继续增量；被测试或发布引用的产物（镜像、配置、模块、清单）另存一份不可变的快照，记录它的身份。

## 5. 根文件系统构建

按速度从快到慢：

| 方式 | 速度 | 适用 |
|---|---|---|
| 原生 ARM64（本机或 CI arm64 runner） | 最快，完整根构建十几分钟 | 首选 |
| root 权限 + qemu-user chroot（binfmt） | 比全系统模拟快约一倍 | 没有 ARM64 机器时 |
| QEMU 全系统模拟（TCG） | 最慢，同样的构建 40 分钟以上 | 只用于验证 init、systemd 行为 |

QEMU 模拟 aarch64 时把指针认证换成实现定义的算法：qemu-user/binfmt 设 `QEMU_CPU=max,pauth-impdef=on`，qemu-system 用 `-cpu max,pauth-impdef=on`。默认的 PAC 算法模拟起来很慢，2026 年实测要慢 5 倍以上。

做法：
- 基础层来自发行版官方镜像（例如 Ubuntu Desktop ISO 里的 squashfs），按 ISO 哈希缓存。
- 装包顺序：先装发行版更新和项目的用户态包，再 purge 通用内核，最后装项目内核，让 initramfs 只生成一次。
- dracut 往往是最大的单项耗时（`hostonly=no` 的通用模式会打包大量模块）。日常迭代不要重建整个根：**在设备上装新包、在设备上生成 initramfs**，几十秒就完成。只在发布时构建完整根。
- 不要同时跑多个全系统模拟的虚拟机，它们互相拖慢，计时失真。
- 哈希给有用途的载荷算：上机的镜像、发布物、缓存复用的输入、恢复用的镜像；不为没有决策用途的大型中间文件反复算。
- 根文件系统交付用 tar（或只编码已用块的 sparse），见 `../experience/usb-and-storage.md`。

## 6. 其他大件

- **Mesa**：交叉编译或原生编译；不要在 qemu-user 下编译（代码生成器会慢上百倍）。源码和补丁固定版本；编译参数跟发行版一致（`-O2`），见 `../experience/gpu.md`。
- **内核外的用户态组件**（传感器栈、认证守护进程等）：慢的产物按"输入提交 + 构建脚本哈希"缓存，复用前复核哈希。
- **个人或小团队维护的构建依赖**：在自己的账号下保全重建所需的源码与补丁（只保全重建需要的部分，先量体积）；机构维护的来源固定版本和哈希即可。

## 7. 参考仓库

- 放在工作区的单独目录（例如 `sources/`），只读使用，记录地址和提交。
- 大仓库用部分克隆：`git clone --filter=blob:none`（保留完整历史，文件内容按需下载）；只需要最近的代码、不看历史时才用 `--depth` 浅克隆。
- 内核仓保留完整上游历史（第三方作者、署名），用一个本地上游镜像作为对象来源可以省下载。
- 不依赖参考仓库的绝对路径或构建输出。

## 8. 磁盘与清理

- 定期看 `df`；剩余低于约 30 GB 时先清理再开大构建。大体量任务开工前，核对它的峰值磁盘需求和同时在跑的任务加起来够不够。
- 可以删：失败实验的半成品镜像、已被取代的发布目录、中间 tar、不再被当前构建引用、可以重建的 OUT。
- 不要删：已记录身份的 OUT、发布流程引用的文件（例如 GPL 源码包、签名密钥、原厂输入）、带未合并提交的 worktree、私有证据；也不要缩小或清空有效的编译缓存。
- 发布流水线常从看起来过时的旧版本工作目录读固定输入：删之前解析流水线的输入清单里的绝对路径，被引用的留下；更好的做法是把固定输入迁到一个专门的目录。
- `du` 会把硬链接的稀疏文件算到它先遇到的目录，删之前用 `find -samefile` 核对。
- 清理前把身份记录移到待删目录之外。

## 9. 并行与长任务

- 内核构建共享 ccache，用一把锁串行，避免两个冷构建互相拖慢。
- 主机空闲时，编译和压缩写明用满线程（`-j$(nproc)`、`DEB_BUILD_OPTIONS=parallel=$(nproc)`），委派后看一次实际并发；结合温度开散热，任务完成后恢复。
- 子 agent 各自在独立的 worktree 和分支工作，不共享 OUT 和设备。
- 要活过当前会话的主机进程（监视器、记录器）交给生命周期明确的进程管理器（例如 `systemd-run --user`），并验证会话退出后它还在；`setsid` 逃不出按 cgroup 的清理。设备一侧优先用自主运行的 systemd 单元，通道断了也不受影响。
- 设备永远只有一个操作者。
- 大镜像构建和大拷贝不并行：消费级 QLC 盘持续写入可能只有几十 MB/s。
- GB 级的推送或上传前先量（例如 `git rev-list --objects --disk-usage <远端分支>..HEAD`、`du`），只推目标需要的部分。

## 10. 不必要的开销（可以省掉）

- BTF 和完整调试信息：需要 pahole、显著变慢；除非要用 eBPF CO-RE，否则不开。
- 每次改动都重建完整根文件系统：改为在设备上装包。
- 每次都跑全部 KUnit 或 QEMU 测试：只跑受影响的。
- 对原始镜像算哈希、反复做"可复现性"双构建：发布前做一次即可。
- 为每个实验建一套完整 worktree 和 OUT。
- 在 QEMU 里验证 snap、AppArmor、真实硬件时序：结论不能外推到真机，直接上真机。
