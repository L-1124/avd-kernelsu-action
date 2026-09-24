# AVD KernelSU Action 🚀

Automated GitHub Actions CI pipeline to build an **x86_64 Android Emulator (AVD) Kernel with integrated KernelSU / KernelSU-Next** using Google's official Kleaf / Bazel build system.

[English](#english) | [中文说明](#中文说明)

---

<a name="中文说明"></a>
## 中文说明

本项目提供一套开箱即用的 GitHub Actions 工作流，专门用于在 GitHub 免费云端 Runner 上为 **Android 虚拟机 (AVD / Android Emulator x86_64)** 自动化源码编译内置 **KernelSU** 或 **KernelSU-Next** 的内核镜像 (`bzImage`)。

无需搭建本地数百 GB 的繁琐 AOSP 构建环境，仅需几步配置即可产出与 AVD 原装系统 100% 二进制兼容的专属内核。

---

### 1. 为什么需要本项目？（技术背景与原理）

在 Android 模拟器（AVD）上集成 Root 权限历来存在以下核心痛点：

1. **官方预编译 LKM 模块为何在 AVD 上无法加载？**
   - 官方发布的通用 GKI 内核模块（如 `lkm-x86_64-android16-6.12_kernelsu.ko`）在模拟器中执行 `insmod` 会报错 `Unknown symbol ... (err -2)`，缺失大量的 SELinux 内部符号（如 `avtab_search_node`、`sidtab_*`、`ebitmap_*`、`avc_has_perm`）。
   - **根本原因**：Google AVD 虚拟化内核（ranchu/goldfish）在编译时虽然开启了 `CONFIG_KPROBES` 和 `CONFIG_MODULES`，但其 defconfig **并未导出内核私有 SELinux 符号**（真机 GKI 符号清单中有导出）。因此，外部 LKM 模块无法在运行时解析这些符号。
2. **为什么替换内核 (`-kernel bzImage`) 是最优雅的方案？**
   - **零破坏**：无需使用 `-writable-system` 或破解 `vbmeta`。
   - **免修补**：无需解包或篡改 `ramdisk.img`，避免 Magisk 常见的冷启动配置丢失或引导循环问题。
   - **完全驱动兼容**：通过严格对齐 AVD 镜像对应的 Android CI 构建分支（如 `common-android16-6.12-2025-09`）与编译目标（`//common-modules/virtual-device:virtual_device_x86_64_dist`），产出的 `bzImage` 与 AVD 镜像自带的 `/vendor/lib/modules/` 及 `/system/lib/modules/` 驱动保持 100% KMI 二进制兼容。

---

### 2. 当前预设目标内核参数 (Target Specs)

工作流默认配置已严格对齐 Android 16.1 官方模拟器镜像：

| 项目 | 参数 / 取值 |
| :--- | :--- |
| **Android 版本** | Android 16.1 (API 36.1 / Baklava) |
| **SDK 镜像路径** | `system-images;android-36.1;google_apis;x86_64` |
| **目标架构** | `x86_64` |
| **原装内核版本** | `6.12.38-android16-5-gbb9513914902-ab13996879` |
| **Android CI 构建 ID** | `13996879` |
| **Kernel Manifest 分支** | `common-android16-6.12-2025-09` |
| **Kleaf 编译目标** | `//common-modules/virtual-device:virtual_device_x86_64_dist` |
| **默认编译产物** | `bzImage` (x86_64 内核镜像) + `bzImage.sha256sum` |

> 💡 本项目不仅支持 Android 16.1，还支持通过修改分支参数无缝编译 **Android 13 / 14 / 15 / 16** 各版本的 AVD 内核（详见下文[通用适配指引](#通用适配指引)）。

---

### 3. GitHub Actions 使用步骤

#### 步骤一：Fork 本仓库
点击仓库右上角 **Fork** 按钮，将本项目 Fork 到您自己的 GitHub 账号中。

#### 步骤二：启用 Actions 读写权限
在您 Fork 后的仓库主页中：
1. 点击 **Settings** -> 左侧侧边栏 **Actions** -> **General**。
2. 找到 **Workflow permissions**，选择 **Read and write permissions**。
3. 点击 **Save** 保存。

#### 步骤三：触发工作流
1. 进入仓库的 **Actions** 标签页。
2. 在左侧列表中点击 **Build AVD KernelSU**。
3. 点击右侧 **Run workflow** 下拉菜单，根据需要调整参数后点击绿色按钮启动编译：

| 输入参数 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `kernel_manifest_branch` | `common-android16-6.12-2025-09` | AOSP 内核 Manifest 分支，与 AVD 镜像完全对应 |
| `ksu_flavor` | `KernelSU-Next` | KernelSU 变种：推荐选择 `KernelSU-Next`（持续跟进最新 6.12+ 内核并支持内置集成）或 `KernelSU` |
| `ksu_version` | `next` | 选定变种的分支名、Tag 或 Commit ID |
| `build_target` | `//common-modules/virtual-device:virtual_device_x86_64_dist` | Kleaf 构建目标（**严禁**修改为普通 GKI 目标，否则会缺失虚拟外设驱动） |
| `lto_mode` | `none` | 链接时优化：默认 `none` 可大幅节省内存并防止 Runner OOM，同时编译速度提升 2~3 倍 |
| `use_fast_config` | `true` | 是否附加 `--config=fast` 快速编译标志 |
| `extra_bazel_args` | `--jobs=4` | 传递给 Kleaf/Bazel 的额外并发与控制参数 |
| `create_release` | `false` | 是否在编译成功后自动在仓库发布 GitHub Release |

#### 步骤四：下载产物
- 编译大约耗时 **35 ~ 45 分钟**。
- 任务执行完毕后，在构建详情页面的 **Artifacts** 区域即可下载打包好的 `bzImage`、`bzImage.sha256sum` 以及可能编译出的 `virtual_device_modules.tar.gz`。

---

### 4. AVD 加载与持久化实操

解压下载的 Artifact，获得 `bzImage` 文件（例如存放于 `F:\kernels\bzImage`）。

#### 方式 A：命令行即时启动 (CLI Boot)
适合临时调试与测试：
```powershell
# 启动命令中通过 -kernel 参数指定自定义 bzImage
emulator @Pixel_10 -kernel F:\kernels\bzImage -show-kernel
```
> *若您的 AVD 名称不同，请替换 `@Pixel_10` 为您的实际 AVD 名称（可通过 `emulator -list-avds` 查看）。*

#### 方式 B：Android Studio 原生常驻持久化 (持久生效，推荐)
只需修改 AVD 的配置文件，之后直接在 Android Studio Device Manager 中点击运行按钮即可自动加载 KernelSU 内核：

1. 打开 AVD 的配置目录（通常位于 `C:\Users\<你的用户名>\.android\avd\<AVD名字>.avd\`）。
2. 使用文本编辑器打开 `config.ini`。
3. 在末尾追加或修改 `kernel.path` 项（**注意路径中的斜杠使用正斜杠 `/`**）：
   ```ini
   kernel.path = F:/kernels/bzImage
   ```
4. 保存 `config.ini`。
5. 在 Android Studio 中直接启动该 AVD。

---

### 5. 功能验证与 Root 使用

AVD 启动进入桌面后，打开命令行执行以下验证步骤：

#### 1. 验证内核版本
```powershell
adb shell uname -a
```
预期输出包含您编译内核的构建时间和版本号（如 `6.12.xx-android16-...`），且与产物 `bzImage` 同源（可用 `vmlinux` 中的 `Linux version` 字符串交叉核对）。

#### 2. 安装管理器 App
下载并安装对应变种的管理器 APK：
- **KernelSU-Next 管理器**：前往 [KernelSU-Next Releases](https://github.com/KernelSU-Next/KernelSU-Next/releases) 下载最新 APK。
- **KernelSU 官方管理器**：前往 [KernelSU Releases](https://github.com/tiann/KernelSU/releases) 下载最新 APK。

安装到 AVD：
```powershell
adb install KernelSU_Next_v3.4.0_33294-release.apk
```
在模拟器内打开管理器，主页显示 **“工作中 (Working)”** 即表示内核态 KernelSU 已正常工作，可在授权界面中为目标 App 分配 Root 权限及模块管理能力。

> ⚠️ 安装管理器后请**重启一次 AVD**：`/data/adb/ksud` 由管理器生成，而 KernelSU 注入的 `init.rc` 启动项在首次开机时该文件尚不存在。

#### 3. 验证内核态 KernelSU（推荐，最可靠的验证方式）
```powershell
adb shell /data/adb/ksud debug version
# 预期输出：Kernel Version: 33310

adb shell /data/adb/ksud debug info
# 预期输出：runtime_mode: built-in / lkm: false / uapi_version: 4
```
可通过 `adb shell 'cat /proc/kallsyms | grep -E " (kernelsu_init|ksu_cred)$"'` 进一步确认 KernelSU 符号已内置到运行中的内核。

#### 4. 关于 `adb shell su -v`
- `su` 依赖 **KernelSU 授权名单**：`adb shell` 的 uid 2000 默认不在名单内，需先在管理器界面把 `Shell` 加入允许列表。
- 若 AVD 的 ramdisk 曾被 **Magisk** 修补过，`/system/xbin/su` 会残留非 KernelSU 的 su，从而遮蔽 `su` 查找（表现为 `su: invalid uid/gid '-v'`）。**建议使用全新未打 Magisk 的 AVD** 来获得干净的 `su` 路径。

---

### 6. 通用适配指引（如何适配其他 AVD 镜像）

如果您使用的是 Android 14、15 或其它系统架构镜像，可以按照以下标准步骤提取对应的分支参数：

1. **启动您当前的目标 AVD**。
2. **提取系统运行的内核信息**：
   ```powershell
   adb shell cat /proc/version
   ```
   例如终端输出：
   ```text
   Linux version 6.12.38-android16-5-gbb9513914902-ab13996879 (android-build@...) ...
   ```
   - 提取中的提交编号：`-ab13996879`，即 CI 构建 ID 为 `13996879`。
3. **定位官方 Manifest 分支**：
   - 打开浏览器访问：`https://ci.android.com/builds/submitted/<构建ID>/kernel_virt_x86_64/latest`
     *(例如：`https://ci.android.com/builds/submitted/13996879/kernel_virt_x86_64/latest`)*
   - 点击并查看下载列表中的 `BUILD_INFO` 文件。
   - 在 `BUILD_INFO` 中找到 `"branch": "..."` 字段（例如 `"branch": "common-android16-6.12-2025-09"`）。
4. **运行 Action**：将查到的分支填入工作流的 `kernel_manifest_branch`，即可自动为您特定的 AVD 编译内核！

---

### 8. 已实机验证记录 (Verified on AVD)

以下结论来自一次真实构建与真机启动验证（Actions Run `35982444708`，构建耗时 31m46s，产物 `bzImage` 22 MB）：

| 验证项 | 结果 |
| :--- | :--- |
| **启动方式** | `emulator @Pixel_10 -kernel bzImage -no-window` |
| **`uname -r`** | `6.12.38-android16-5-gdfed788d0bca-dirty`（与 vmlinux 内 `Linux version` 完全一致） |
| **`/proc/kallsyms`** | 存在 `kernelsu_init` (T)、`ksu_cred` (B)、`ksu_syscall_dispatcher` (t) → KernelSU 已内置 |
| **`ksud debug version`** | `Kernel Version: 33310`（KernelSU-Next v3.4.0） |
| **`ksud debug info`** | `runtime_mode: built-in`、`lkm: false`、`uapi_version: 4` |
| **Manager 握手** | 内核日志出现 `KernelSU: allow root for: 10226`（成功为管理器进程授权） |
| **驱动兼容性** | 内核日志中 `unknown symbol` / `disagrees about version` / `module verification failed` 计数均为 **0**；AVD 原装 `.ko`（`vivid`、`virtio_*`、`v4l2loopback`、`system_heap` 等）全部正常加载 |
| **SELinux** | `Enforcing`，系统正常启动至桌面 |

产物内容（Artifact 共 5 个文件）：

```text
bzImage                        22 MB   ← 用于 -kernel / config.ini
bzImage.sha256sum              74 B    ← 校验值
vmlinux                        204 MB  ← 带符号调试内核
System.map                     9.4 MB  ← 符号表（含 281 个 KernelSU 符号）
virtual_device_modules.tar.gz  2.2 MB  ← 45 个虚拟设备驱动（goldfish_*/virtio-*/vkms 等）
```

#### ⚠️ 三个实测注意事项

1. **首次开机后必须重启一次**：KernelSU 会向 `init.rc` 注入 `exec u:r:ksu:s0 root -- /data/adb/ksud ...`，而 `/data/adb/ksud` 只有在安装 Manager App 之后才会生成。因此首次开机日志会出现 `Cannot find '/data/adb/ksud'`，属正常现象；安装 Manager 后重启一次，`ksud` 即可正常工作。
2. **Manager APK 版本可略旧于内核**：实测内核 `33310`（v3.4.0）与官方 Manager APK `33294` 握手成功（签名一致即可）。但建议尽量使用与内核同一版本系列的官方 APK。
3. **不要在曾被 Magisk 修补过 ramdisk 的 AVD 上验证 `su -v`**：这类 AVD 的 `/system/xbin/su` 是 Magisk 残留存根（仅接受 `uid/gid` 参数），会遮蔽 `su` 查找；且 `adb shell` 的 uid 2000 默认不在 KernelSU 授权列表中。正确做法是：
   - 用 `adb shell /data/adb/ksud debug version`（内核态验证），或在 Manager 界面为 App 授权；
   - **推荐使用全新未打过 Magisk 的 AVD**，以获得干净的 `su` 路径。

---

### 9. 常见问题 (FAQ)

- **Q: 为什么不编译 `//common:kernel_x86_64_dist` 而是 `//common-modules/virtual-device:virtual_device_x86_64_dist`？**
  - **A**: `//common:kernel_x86_64_dist` 仅包含纯净 Generic Kernel Image，缺少虚拟化设备所需的关键外设驱动和 goldfish/virtio 模块配置。只有编译 `virtual_device_x86_64_dist`，产出的 `bzImage` 才能完整驱动 AVD 的虚拟硬件。
- **Q: 为什么构建参数中默认使用 `--lto=none`？**
  - **A**: GitHub Actions 免费 Runner 的物理内存为 16GB。Android 6.12 内核使用 ThinLTO 链接时，LLD 峰值内存会突破 18GB，导致 Runner 频繁遭遇 OOM 崩溃退出。设置为 `none` 既能保证内核功能完全正常，又能避免内存溢出，并将链接耗时从 20 分钟压缩至 2 分钟。
- **Q: 工作流如何解决 GitHub Actions 磁盘空间不足（No space left on device）？**
  - **A**: 工作流首个步骤通过 `easimon/maximize-build-space` 卸载无用的预置 Android SDK、.NET 环境与 Docker 镜像，挂载出 55GB+ 的可用根分区空间，并配置了 8GB 的 Swap 交换分区。

---

<a name="english"></a>
## English Description

This repository provides an automated GitHub Actions CI workflow to compile an **x86_64 Android Emulator (AVD) Kernel with integrated KernelSU / KernelSU-Next** (`bzImage`) on free GitHub-hosted runners using Google's Kleaf (Bazel) build toolchain.

### Key Highlights
- **Direct Kernel Boot**: Boots directly via emulator `-kernel bzImage` or `config.ini` parameter (`kernel.path`).
- **Zero Tampering**: No need for `-writable-system`, no ramdisk patching, and zero modification of system partitions.
- **Full Driver Compatibility**: Compiled against target virtual device definitions (`//common-modules/virtual-device:virtual_device_x86_64_dist`), ensuring complete binary KMI compatibility with stock AVD `.ko` vendor drivers.
- **Optimized for CI**: Pre-configured with disk space maximization (55GB+ free space), 8GB swap memory, and fast Bazel build flags (`--lto=none --config=fast`) to prevent runner OOM errors.

### Quick Start
1. **Fork** this repository.
2. In your fork, go to **Settings** -> **Actions** -> **General** -> **Workflow permissions** -> Select **Read and write permissions**.
3. Navigate to the **Actions** tab -> Select **Build AVD KernelSU** -> Click **Run workflow**.
4. Wait ~40 minutes and download `bzImage` from the build artifacts.
5. Launch your AVD:
   ```powershell
   emulator @<AVD_NAME> -kernel path\to\bzImage -show-kernel
   ```
   Or persist in `~/.android/avd/<AVD_NAME>.avd/config.ini`:
   ```ini
   kernel.path = path/to/bzImage
   ```
6. Install the matching KernelSU Manager APK, then verify in-kernel status:
   ```powershell
   adb shell /data/adb/ksud debug version   # -> "Kernel Version: 33310"
   adb shell /data/adb/ksud debug info      # -> runtime_mode: built-in
   ```
   Grant root per-app from the Manager UI. Note: `/data/adb/ksud` only exists after the Manager is installed — reboot once after installing it.

---

## License

This project is licensed under the [MIT License](LICENSE) (or Apache-2.0). Kernel sources and KernelSU adhere to their respective GPL-2.0 upstream licenses.
