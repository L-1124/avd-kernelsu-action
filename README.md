# AVD KernelSU Action

Automated GitHub Actions CI pipeline to build an **x86_64 Android Emulator (AVD) Kernel with integrated KernelSU / KernelSU-Next / SukiSU-Ultra / ReSukiSU** using Google's official Kleaf / Bazel build system.

[English](#english) | [中文说明](#中文说明)

---

<a name="中文说明"></a>
## 中文说明

本项目提供一套 GitHub Actions 工作流，专门用于在 GitHub 免费云端 Runner 上为 **Android 虚拟机 (AVD / Android Emulator x86_64)** 源码编译内置 **KernelSU**、**KernelSU-Next**、**SukiSU-Ultra** 或 **ReSukiSU** 的内核镜像 (`bzImage`)。

通过对齐 Google 官方虚拟设备构建目标，产出的内核镜像与 AVD 原装系统驱动保持 100% 二进制兼容，无需本地搭建数百 GB 的 AOSP 环境。

---

### 1. 原理与技术溯源

本项目的基础技术路线、问题归因与补丁机制主要来自开发者 [5ec1cff](https://github.com/5ec1cff) 的系列研究（参见 [5ec1cff's blog](https://5ec1cff.github.io/my-blog/archives/)）以及 KernelSU 社区未合并/分支变更：

#### 1.1 为什么官方 LKM 模块无法在 AVD 上使用
* **SELinux 符号未导出**：在 Android 虚拟机中执行 `insmod` 加载官方发布的通用 GKI 内核模块（如 `lkm-x86_64-android16-..._kernelsu.ko`）时，会抛出 `Unknown symbol ... (err -2)` 错误，缺失大量的 SELinux 内部符号（如 `avtab_search_node`、`sidtab_*`、`ebitmap_*`、`avc_has_perm`）。
* **根本机制**：正如 5ec1cff 在 2026 年文章《在内核模块中解析内核符号》中所分析，Android GKI 构建阶段开启了严格的符号白名单裁剪（`UNUSED_KSYMS_WHITELIST` / `kmi_symbol_list`）。AVD 的虚拟化内核（ranchu/goldfish）defconfig 并未向外部模块导出这些私有符号。
* **技术决策**：放弃外部 LKM 动态加载，选用**源码级静态内置编译（Built-in，`CONFIG_KSU=y`）**，在链接期直接绑定内部符号，彻底避开内核符号裁剪与运行时重定位问题。

#### 1.2 为什么必须使用 `virtual_device_x86_64_dist` 编译目标
* **外设驱动兼容性断层**：5ec1cff 在 2024 年文章《在 AVD 上使用 KernelSU》中初次探索时指出，直接编译通用 GKI 目标（`//common:kernel_x86_64_dist`）产出的内核缺少虚拟化硬件支持，导致必须手动解包 `ramdisk.img`、注入 `virtio-*.ko`、挂载 `-writable-system` 并执行 `adb remount` 推送驱动。
* **CI 编号精准对齐方案**：5ec1cff 在随后的《在 AVD 上使用 KernelSU - 第二回 -》中找到了终极解法：从 AVD 的 `/proc/version` 提取 `-ab<BUILD_ID>` 构建编号，检索 Google Android CI 的 `BUILD_INFO`。结果表明官方构建 AVD 内核所使用的是专用目标：
  ```bash
  //common-modules/virtual-device:virtual_device_x86_64_dist
  ```
  采用该目标编译出的 `bzImage` 与 AVD 原装系统自带的 `/vendor/lib/modules/` 及 `/system/lib/modules/` 驱动保持 100% KMI 二进制兼容，实现**零解包、零修补分区、纯净冷启动**。

#### 1.3 x86_64 系统调用动态分发补丁 (PR #3562)
* **间接调用加固**：现代 x86_64 Linux 内核对系统调用入口路径做了安全加固，将传统的 `sys_call_table` 间接分支改为了直接条件跳转，导致 KernelSU 传统的 `syscall_hook` 钩子被绕过，源码中会抛出 `#error "FATAL: Your kernel is missing the indirect syscall bypass patches!"`。
* **动态分发器**：本项目流水线整合了 5ec1cff 在上游贡献的 x86 补丁（如 PR #3562 等），在构建期向 `gki_defconfig`、Kbuild 及 Makefile 注入 `CONFIG_KSU_X86_PATCH_SYSCALL_DISPATCHER=y`，并在内核引导阶段动态修补 hardened dispatcher。

#### 1.4 文献引用与参考
* [在内核模块中解析内核符号 (2026-03-31)](https://5ec1cff.github.io/my-blog/2026/03/31/resolve-kallsyms-in-lkm/)
* [Linux kernel Kallsyms 构建与解析 (2026-03-18)](https://5ec1cff.github.io/my-blog/2026/03/18/kallsyms/)
* [在 AVD 上使用 KernelSU - 第二回 - (2024-01-31)](https://5ec1cff.github.io/my-blog/2024/01/31/avd-ksu2/)
* [在 AVD 上使用 KernelSU (2024-01-16)](https://5ec1cff.github.io/my-blog/2024/01/16/avd-ksu/)
* [KernelSU PR #3562: kernel: add syscall dispatcher patching for x64](https://github.com/tiann/KernelSU/pull/3562)

---

### 2. 目标规格与分支定位

工作流预设配置对齐 Android 16.1 官方模拟器镜像：

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

#### 如何适配其他 Android 版本 (Android 13 / 14 / 15 / 16)
1. 启动目标 AVD，执行命令提取内核版本串：
   ```powershell
   adb shell cat /proc/version
   ```
   例如输出包含 `-ab13996879`，则构建 ID 为 `13996879`。
2. 浏览器打开 Android CI 链接：
   `https://ci.android.com/builds/submitted/<构建ID>/kernel_virt_x86_64/latest`
3. 打开 `BUILD_INFO` 文件，找到 `"branch"` 字段（例如 `"branch": "common-android16-6.12-2025-09"`）。
4. 在 Actions 触发界面将该分支填入 `kernel_manifest_branch` 即可。

---

### 3. GitHub Actions 使用步骤

1. **Fork 本仓库**到您自己的 GitHub 账号。
2. 在仓库的 **Settings** -> **Actions** -> **General** -> **Workflow permissions** 中选择 **Read and write permissions** 并保存。
3. 进入 **Actions** 标签页，点击 **Build AVD KernelSU** -> **Run workflow**：

| 输入参数 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `kernel_manifest_branch` | `common-android16-6.12-2025-09` | 对应目标 AVD 的 Manifest 分支 |
| `ksu_flavor` | `KernelSU-Next` | 内核变种：`KernelSU-Next`（默认推荐）、`KernelSU`、`SukiSU-Ultra`、`ReSukiSU` |
| `ksu_version` | `dev` | 目标分支、Tag 或 Commit ID（KernelSU-Next 默认为 `dev`，其余变种默认为 `main`；工作流已支持跨变种分支自适应） |
| `build_target` | `//common-modules/virtual-device:virtual_device_x86_64_dist` | Kleaf 构建目标（勿修改为普通 GKI 目标） |
| `lto_mode` | `none` | 链接时优化：默认 `none` 防止 Runner OOM，加速编译 2~3 倍 |
| `use_fast_config` | `true` | 是否附加 `--config=fast` 标志 |
| `extra_bazel_args` | `--jobs=4` | 传递给 Kleaf/Bazel 的额外并发参数 |
| `enable_susfs` | `false` | 是否编入 SuSFS 内核级 Root 隐匿补丁（支持文件/挂载隐藏与特征伪造） |
| `susfs_branch` | `""` | SuSFS Git 分支（留空则根据内核版本自动匹配，如 `gki-android16-6.12`） |
| `create_release` | `false` | 编译成功后是否自动发布 GitHub Release |

编译耗时约 25~35 分钟。完成后在构建详情页的 **Artifacts** 下载产物包（包含 `bzImage`、`bzImage.sha256sum`、配套管理器 `.apk`、`vmlinux`、`System.map`、`virtual_device_modules.tar.gz` 与 `bazel-build.log`）。

---

### 4. AVD 部署与持久化运行

解压下载的 Artifact，获得内核文件 `bzImage`。

#### 方式 A：命令行即时启动 (调试推荐)
```powershell
emulator @<AVD_NAME> -kernel path\to\bzImage -show-kernel
```

#### 方式 B：Android Studio 原生常驻持久化
Android Studio 启动模拟器时直接调用 `emulator -avd <AVD>`，不带 `-kernel` 参数；**在 `config.ini` 中修改 `kernel.path` 会被模拟器启动器覆写为系统镜像路径因而无效**。

正确的持久化方式是直接替换 SDK 系统镜像中的默认内核：

```powershell
$sys = "F:\android\system-images\android-36.1\google_apis\x86_64"

# 1. 备份原装内核（仅需执行一次）
if (-not (Test-Path "$sys\kernel-ranchu.stock")) {
    Copy-Item "$sys\kernel-ranchu" "$sys\kernel-ranchu.stock"
}

# 2. 用编译好的 bzImage 替换默认内核
Copy-Item path\to\bzImage "$sys\kernel-ranchu" -Force

# 3. 丢弃旧快照（关键：防止快照恢复旧内核内存状态导致 KernelSU 不生效）
Rename-Item "$env:USERPROFILE\.android\avd\<AVD_NAME>.avd\snapshots\default_boot" "default_boot.bak-stockkernel"
```

配置后在 Android Studio Device Manager 中直接点击运行即可。
回退时将 `kernel-ranchu.stock` 还原为 `kernel-ranchu` 并恢复快照目录名即可。

---

### 5. 管理器 APK 与驱动版本/签名对齐

KernelSU 内核驱动在握手时对通信程序执行严格校验。管理器与内核之间有两个必须匹配的关键要素：

#### 5.1 版本号与签名要求
1. **驱动版本号 (Driver Version)**：
   KernelSU-Next 的 `kernel/Kbuild` 规定 `versionCode = 30000 + git rev-list --count HEAD`。构建日志中的 `KernelSU-Next version: <versionCode>` 即为驱动版本。
2. **管理器证书签名哈希 (Manager Signature Hash)**：
   内核驱动内部固化了可信任管理器签名的 SHA-256（构建日志输出 `Manager signature hash: <hash>`）。如果 APK 签名不一致，内核将拒绝提供 root 访问（不会创建 supercall fd）。

3. **各变种管理器对应关系**：
   * **KernelSU-Next**：使用同 Commit 的上游 `Build Manager CI` 产物。
   * **KernelSU (tiann)**：使用官方 KernelSU Manager 对应版本。
   * **SukiSU-Ultra**：使用 SukiSU 官方配套管理器 APK。
   * **ReSukiSU**：使用 ReSukiSU / SukiSU / MKSU / 原版 KernelSU 管理器（ReSukiSU 默认开启 `CONFIG_KSU_MULTI_MANAGER_SUPPORT=y` 多管理器支持）。

#### 5.2 自动捆绑与匹配校验
* **CI 自动下载捆绑**：工作流现已实现**全自动检索并打包配套 Manager APK**。构建产物（Artifacts 及 Release）中会直接附带与内核变种及 Commit 严格匹配好的 `.apk` 文件，解压后即可直接通过 `adb install -r <Manager>.apk` 安装，无需手动去上游 CI 搜寻。
* **手动校验说明（可选）**：若需自行验证 APK 签名证书，可使用 `apksigner` 确认证书 SHA-256 是否等于构建摘要中的 `Manager signature hash`：
  ```powershell
  apksigner verify --print-certs <Manager>.apk
  ```

---

### 6. 状态验证与避坑指南

#### 6.1 验证命令与预期结果
在 AVD 启动进入桌面后执行：

```powershell
# 1. 查看内核版本（应包含自定义构建时间与 dirty 标签）
adb shell uname -a

# 2. 检查内核内置符号
adb shell "cat /proc/kallsyms | grep -E ' (kernelsu_init|ksu_cred)$'"

# 3. 验证内核态 KernelSU 状态（/data/adb 权限为 0700，必须先 adb root）
adb root
adb shell /data/adb/ksud debug version
# 预期输出：Kernel Version: 33310 （若返回 0 则表示当前仍在运行未集成 KSU 的原装内核）

adb shell /data/adb/ksud debug info
# 预期输出：runtime_mode: built-in / lkm: false / uapi_version: 4
```

#### 6.2 关键注意事项
1. **安装管理器后必须重启一次 AVD**：
   KernelSU 注入在 `init.rc` 中的服务为 `exec u:r:ksu:s0 root -- /data/adb/ksud ...`。在首次开机且尚未安装 Manager App 时，`/data/adb/ksud` 尚不存在，日志提示 `Cannot find '/data/adb/ksud'` 属正常现象；安装 Manager 后重启一次即可正常引导。
2. **请勿使用 `adb shell su -v` 作为验证手段**：
   模拟器 `google_apis` 镜像自带 `/system/xbin/su`（AOSP 调试版 setuid root su），只支持 `su [uid] [gid] [cmd]` 语法，输入 `-v` 会报错 `invalid uid/gid`。且该 su 不受 KernelSU 管理，无法作为 KernelSU 的生效凭证。请通过 Manager 授权界面或 `ksud debug` 命令进行验证。
3. **避免与 Magisk 混合使用**：
   若此前在 AVD 上安装过 Magisk，其注入的 `/data/adb/magisk` 与二进制会产生冲突。建议在全新未装 Magisk 的 AVD 上使用本内核。

#### 6.3 SuSFS 隐匿支持说明
若在触发构建时启用了 `enable_susfs`：
1. **全自动内核注入**：流水线自动从 `simonpunk/susfs4ksu` 抓取与内核主版本（6.12 / 6.6 / 6.1 / 5.15 / 5.10）匹配的官方分支，将 `susfs.c` 与头文件拷贝入内核源码，并自动应用对应内核 patch 与 KSU 适配补丁，开启 `CONFIG_KSU_SUSFS=y`。
2. **配套 x86_64 静态工具**：流水线会自动为 x86_64 架构编译静态链接的 `ksu_susfs` 命令行工具，并随内核产物（Artifacts / Release）一并打包发布。
3. **AVD 部署与检验**：
   ```powershell
   adb root
   adb push path\to\ksu_susfs /data/adb/ksu/bin/ksu_susfs
   adb shell chmod 755 /data/adb/ksu/bin/ksu_susfs
   adb shell /data/adb/ksu/bin/ksu_susfs show
   ```
---

### 7. 常见问题 (FAQ)

* **Q: 为什么构建参数默认使用 `--lto=none`？**
  * **A**: GitHub Actions 免费 Runner 物理内存为 16GB。Android 6.12 内核在开启 ThinLTO 链接时，LLD 峰值内存会超过 18GB 导致 Runner 发生 OOM 崩溃退出。设为 `none` 可在完整保留内核功能的前提下将链接耗时由 20 分钟降至 2 分钟，稳定避开 OOM。
* **Q: 流水线如何解决 Runner 磁盘不足 (No space left on device)？**
  * **A**: 任务开始时通过 `easimon/maximize-build-space` 卸载预装的 .NET、Haskell、Android SDK 和 Docker 镜像，释放出 55GB+ 根目录可用空间，并配置 8GB Swap。
* **Q: 为什么不能直接编译通用 GKI `//common:kernel_x86_64_dist`？**
  * **A**: 通用 GKI 目标仅包含标准 Linux 驱动，不包含 AVD 运行所需的 virtio、goldfish 虚拟硬件设备驱动及内核模块定义。只有编译 `virtual_device_x86_64_dist` 才能保证虚拟化外设完整工作。

---

<a name="english"></a>
## English Description

This repository provides an automated GitHub Actions CI workflow to compile an **x86_64 Android Emulator (AVD) Kernel with integrated KernelSU / KernelSU-Next / SukiSU-Ultra / ReSukiSU** (`bzImage`) using Google's official Kleaf / Bazel build system.

### Highlights
- **Direct Kernel Boot**: Boot directly via emulator `-kernel bzImage`, or replace `<sysdir>\kernel-ranchu` in your Android SDK system images for persistent Android Studio integration.
- **Zero Tampering**: No need for `-writable-system`, ramdisk unpacking, or system partition modification.
- **Full Driver Compatibility**: Builds the virtual device target (`//common-modules/virtual-device:virtual_device_x86_64_dist`), ensuring complete binary KMI compatibility with stock AVD `.ko` drivers.
- **CI Resource Optimized**: Maximize build space step frees 55GB+ disk space, allocates 8GB swap, uses `-j12` network sync, and uses `--lto=none --config=fast --show_progress_rate_limit=5` to prevent runner OOM crashes and save 3-5 minutes.
- **Auto-Bundled Manager APK**: Automatically queries and packages the matching Manager APK directly into the build artifacts/releases, eliminating version/signature mismatch headaches.
- **SuSFS Root Hiding Support**: Optional integration of SuSFS (`enable_susfs`), including automatic kernel patch injection and pre-compiled static `ksu_susfs` userspace binary for x86_64.

### Technical Lineage
This project is built upon the foundational research of [5ec1cff](https://github.com/5ec1cff) (see [5ec1cff's blog](https://5ec1cff.github.io/my-blog/archives/)):
1. **LKM Failure on AVD**: Analysis in *Resolving Kernel Symbols in LKM (2026)* demonstrates that GKI symbol whitelisting strips private SELinux symbols in AVD's ranchu defconfig, making external LKM loading impossible without symbol resolution workarounds. A built-in kernel (`CONFIG_KSU=y`) completely circumvents this barrier.
2. **Target Alignment**: Findings in *Using KernelSU on AVD - Part 2 (2024)* identify that extracting the `-ab<BUILD_ID>` from `/proc/version` allows mapping to the exact Google Android CI branch and virtual device target (`virtual_device_x86_64_dist`), maintaining 100% binary compatibility with stock drivers.
3. **x86_64 Syscall Dispatcher Patch**: Integrates dynamic syscall dispatcher patching (originating from PR #3562 by 5ec1cff) to handle modern x86_64 hardened system call entries.

### Quick Start
1. **Fork** this repository.
2. Go to **Settings** -> **Actions** -> **General** -> **Workflow permissions** -> Enable **Read and write permissions**.
3. Go to **Actions** -> **Build AVD KernelSU** -> Click **Run workflow**.
4. Download `bzImage` from the build artifacts (~35-45 minutes build time).
5. Launch your AVD:
   ```powershell
   emulator @<AVD_NAME> -kernel path\to\bzImage -show-kernel
   ```
   For **Android Studio** launches (where `config.ini`'s `kernel.path` is ignored and overridden):
   ```powershell
   $sys = "F:\android\system-images\android-36.1\google_apis\x86_64"
   Copy-Item "$sys\kernel-ranchu" "$sys\kernel-ranchu.stock"   # backup once
   Copy-Item path\to\bzImage "$sys\kernel-ranchu" -Force
   Rename-Item "$env:USERPROFILE\.android\avd\<AVD_NAME>.avd\snapshots\default_boot" "default_boot.bak-stockkernel"
   ```
6. Install matching Manager APK and verify:
   ```powershell
   adb root
   adb shell /data/adb/ksud debug version   # Expected: Kernel Version: 33310
   adb shell /data/adb/ksud debug info      # Expected: runtime_mode: built-in
   ```
   Reboot once after installing the Manager app to activate the `/data/adb/ksud` init.rc hooks.

---

## License

This project is licensed under the [MIT License](LICENSE). Kernel sources and KernelSU adhere to their respective upstream GPL-2.0 licenses.
