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

> ⚠️ **不要改 `config.ini` 的 `kernel.path`——实测无效**。模拟器启动时会用系统镜像路径把它覆盖掉，生成的 `hardware-qemu.ini` 里实际变成：
> ```ini
> kernel.path = F:\android\system-images\android-36.1\google_apis\x86_64\\kernel-ranchu
> ```
> 同时该版本模拟器二进制中**不存在** `ANDROID_EMULATOR_KERNEL_FILE` 环境变量（实测查无此串），也无法通过环境变量指定内核。

要让 Android Studio（它只执行 `emulator -avd <AVD>`，不带 `-kernel`，见其记录的 `emu-launch-params.txt`）也用上自定义内核，正确做法是**替换系统镜像的默认内核 `kernel-ranchu`**：

```powershell
$sys = "F:\android\system-images\android-36.1\google_apis\x86_64"

# 1) 备份原装内核（只需一次）
if (-not (Test-Path "$sys\kernel-ranchu.stock")) {
    Copy-Item "$sys\kernel-ranchu" "$sys\kernel-ranchu.stock"
}

# 2) 用编译产物替换默认内核
Copy-Item F:\kernels\bzImage "$sys\kernel-ranchu" -Force

# 3) 丢弃旧快照（关键！否则快速启动会恢复旧内核的 RAM 镜像，KernelSU 依旧不生效）
Rename-Item "$env:USERPROFILE\.android\avd\Pixel_10.avd\snapshots\default_boot" `
            "default_boot.bak-stockkernel"
```

之后在 Android Studio Device Manager 中直接点运行即可，无需任何额外参数。

**验证**（同样不带 `-kernel`，与 Studio 启动方式一致）：
```powershell
adb shell uname -r                                  # 应为你编译的内核
adb root; adb shell /data/adb/ksud debug version    # 应为与内核一致的 KSU 版本号
```
> `ksud debug version` 返回 **0** 即表示当前仍是原装内核（内核里没有 KernelSU）。

**回退**：`Copy-Item "$sys\kernel-ranchu.stock" "$sys\kernel-ranchu" -Force`，并把快照目录名改回 `default_boot`。

> 注意：替换 `kernel-ranchu` 会影响**所有**使用该 system image 的 AVD（本例仅 Pixel_10）。若不同 AVD 需要不同内核，请改用方式 A 的 `-kernel` 参数，或为每个 AVD 复制一份独立 system image。

---

### 5. 功能验证与 Root 使用

AVD 启动进入桌面后，打开命令行执行以下验证步骤：

#### 1. 验证内核版本
```powershell
adb shell uname -a
```
预期输出包含您编译内核的构建时间和版本号（如 `6.12.xx-android16-...`），且与产物 `bzImage` 同源（可用 `vmlinux` 中的 `Linux version` 字符串交叉核对）。

#### 2. 安装管理器 App
**按内核版本对齐安装**（详见 [第 7 节](#7-管理器-apk-与内核版本对齐manager-apk-matching)）——官方 Release 通常落后于内核所在的 `dev`，建议取同 commit 的 CI 产物：

```powershell
adb install -r KernelSU_Next_v3.4.0-16-g32e897e2_33310-release.apk   # versionCode 需与内核驱动版本一致
```
在模拟器内打开管理器，主页显示 **“工作中 (Working)”** 即表示内核态 KernelSU 已正常工作，可在授权界面中为目标 App 分配 Root 权限及模块管理能力。

> ⚠️ 两点务必注意：
> 1. 安装管理器后请**重启一次 AVD**：`/data/adb/ksud` 由管理器生成，而 KernelSU 注入的 `init.rc` 启动项在首次开机时该文件尚不存在。
> 2. 安装前先校验 APK 签名（`apksigner verify --print-certs`）是否等于内核的 `Manager signature hash`，否则管理器拿不到 root。

#### 3. 验证内核态 KernelSU（推荐，最可靠的验证方式）
```powershell
adb root        # /data/adb 权限为 0700 root:root，必须先提权才能访问 ksud
adb shell /data/adb/ksud debug version
# 预期输出：Kernel Version: 33310

adb shell /data/adb/ksud debug info
# 预期输出：runtime_mode: built-in / lkm: false / uapi_version: 4
```
若不加 `adb root`，会得到 `/data/adb/ksud: inaccessible or not found`（这是 `/data/adb` 目录权限导致，不代表 ksud 不存在）。

可通过 `adb shell 'cat /proc/kallsyms | grep -E " (kernelsu_init|ksu_cred)$"'` 进一步确认 KernelSU 符号已内置到运行中的内核；另可查看 KernelSU 注入的 init.rc 钩子是否生效：
```powershell
adb shell dmesg | findstr /C:"/data/adb/ksud"
# 预期：Command 'exec u:r:ksu:s0 root -- /data/adb/ksud services' ... succeeded
#       Command 'exec u:r:ksu:s0 root -- /data/adb/ksud boot-completed' ... succeeded
```

#### 4. 关于 `adb shell su -v`
- **它不是有效的验证手段**：模拟器 `google_apis` 镜像自带 `/system/xbin/su`（setuid root 的 AOSP 调试 su，位于只读系统分区 `dm-5`，与 KernelSU/Magisk 无关）。该 su 只接受 `su [uid] [gid] [命令]` 形式，因此 `su -v` / `su -c id` 必然报 `su: invalid uid/gid '-v'`；而它本身是 setuid root，不经 KernelSU 也可提权，故不能用来证明 KernelSU 生效。
- **KernelSU 的 root 由授权名单控制**：`adb shell` 的 uid 2000 默认不在名单内，需在管理器界面手动授权；普通 App 则在其首次请求 root 时由管理器弹窗授权。
- **不要在此 AVD 上再装 Magisk**：若 `/data/adb/magisk` 存在，Magisk 注入的 su 会与 KernelSU 争夺 `su` 路径。验证 KernelSU 请使用全新未装 Magisk 的 AVD。

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

### 7. 管理器 APK 与内核版本对齐（Manager APK matching）

KernelSU 的**内核驱动**内置了"可信任管理器"的签名哈希，管理器 APK 的版本与签名必须与内核匹配，否则管理器界面拿不到 root（表现：无 supercall fd、无法授权、模块页面异常）。

#### 7.1 版本号是怎么来的

KernelSU-Next 的 `kernel/Kbuild` 明确定义：

```make
KSU_GIT_VERSION := $(shell cd $(GIT_ROOT) && git rev-list --count HEAD)
KSU_VERSION = $(shell expr 30000 + $(KSU_GIT_VERSION))
```

即 **versionCode = 30000 + 内核所集成的 KernelSU 仓库提交数**。所以 `33310` 表示该 checkout 有 3310 个提交，而官方 Release `v3.4.0`（APK versionCode `33294`）= 3294 个提交 —— 两者差 16 个提交。

#### 7.2 两个必须对齐的值

构建完成后，从 **Job Summary** 或产物中的 `bazel-build.log` 读取：

| 值 | 含义 | 实测示例（本次内核） |
| :--- | :--- | :--- |
| `KernelSU driver version` | 内核驱动版本号（= `30000 + 提交数`） | `33310` |
| `Manager signature hash` | 内核要求的管理器签名证书 SHA-256 | `79e590113c4c4c0c222978e413a5faa801666957b1212a328e46c00c69821bf7` |
| `KernelSU ref` | 本次集成所用的 ref（`dev` / tag / commit） | `dev` @ `32e897e2` |

驱动版本也可在设备上直接查询：

```powershell
adb root
adb shell /data/adb/ksud debug version     # Kernel Version: 33310
```

#### 7.3 去哪里找那个版本的管理器 APK

**官方 Release 往往落后于 `dev`**（本例 Release 只有 33294）。要拿到与内核严格同版本的管理器，用上游 CI 产物 —— workflow `Build Manager CI` 在 `dev` 每次提交都会产出 `manager` artifact：

```powershell
# 1) 找到与内核同一 commit 的那次 CI（本例 commit 32e897e2）
gh run list --repo KernelSU-Next/KernelSU-Next --workflow "Build Manager CI" --branch dev --limit 5

# 2) 下载 manager 产物
gh run download <RUN_ID> --repo KernelSU-Next/KernelSU-Next --name manager

# 产物命名规律：KernelSU_Next_<tag>-<n>-g<shortsha>_<versionCode>-release.apk
# 例：KernelSU_Next_v3.4.0-16-g32e897e2_33310-release.apk
```

#### 7.4 安装前必须校验签名（关键步骤）

CI 产物与官方 Release 使用**同一签名密钥**，但自己编译的 APK 不是，装了会被内核静默拒绝。校验方法：

```powershell
# 需要 Android SDK build-tools 里的 apksigner
apksigner verify --print-certs KernelSU_Next_v3.4.0-16-g32e897e2_33310-release.apk
# 关注输出中的：
#   Signer #1 certificate SHA-256 digest: 79e590113c4c4c0c222978e413a5faa801666957b1212a328e46c00c69821bf7
#                                            ↑ 必须等于内核的 Manager signature hash
```

签名一致后再安装并验证内核确已接受：

```powershell
adb install -r KernelSU_Next_v3.4.0-16-g32e897e2_33310-release.apk
adb shell 'dumpsys package com.rifsxd.ksunext | grep versionCode'   # 期望 33310
adb shell 'dmesg | tail -40 | grep -E "ksu fd installed|allow root for"'
#   期望： KernelSU: ksu fd installed: 5 for pid xxxx
#          KernelSU: allow root for: <uid>          ← 管理器进程的 uid
```

也可用最小模块从安装器视角验证两个版本号是否一致（会打印 `KSU_VER_CODE` / `KSU_KERNEL_VER_CODE`）：

```powershell
# 装一个只做 ui_print 的探针模块，安装日志应为 MATCH
#   - KSU_VER_CODE=33310        (manager)
#   - KSU_KERNEL_VER_CODE=33310 (kernel)
```

#### 7.5 常见误区

- **不要用 `next` 作为 ref**：KernelSU-Next 的 `next` 分支已不存在（现为 `dev`，另有 `legacy`）。`git checkout next` 会失败并被 setup.sh 静默回退到默认分支，导致编译出的其实不是你以为的版本。本工作流已默认 `dev`，且当 ref 不存在时**直接报错退出**（不再静默回退）。
- **签名不符 ≠ 版本不符**：版本不符通常只是警告（管理器与新驱动一般仍可用）；签名不符则管理器完全拿不到 root。
- **管理器版本可以略旧**：实测 33294 管理器配 33310 内核仍能正常工作（`allow root for` 成功）；对齐只是为了消除隐患与警告。

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

产物内容（Artifact 共 6 个文件）：

```text
bzImage                        22 MB   ← 用于 -kernel，或替换 <sysdir>\kernel-ranchu（见第 4 节方式 B）
bzImage.sha256sum              74 B    ← 校验值
vmlinux                        204 MB  ← 带符号调试内核
System.map                     9.4 MB  ← 符号表（含 281 个 KernelSU 符号）
virtual_device_modules.tar.gz  2.2 MB  ← 45 个虚拟设备驱动（goldfish_*/virtio-*/vkms 等）
bazel-build.log                ~100 KB ← 记录 KernelSU 驱动版本与管理器签名哈希
```

#### ⚠️ 四个实测注意事项

1. **首次开机后必须重启一次**：KernelSU 会向 `init.rc` 注入 `exec u:r:ksu:s0 root -- /data/adb/ksud ...`，而 `/data/adb/ksud` 只有在安装 Manager App 之后才会生成。因此首次开机日志会出现 `Cannot find '/data/adb/ksud'`，属正常现象；安装 Manager 后重启一次，`ksud` 即可正常工作。
2. **Android Studio 启动不会带上自定义内核**：Studio 执行的是 `emulator -avd <AVD>`（实测其 `emu-launch-params.txt` 中无 `-kernel`），且 `config.ini` 的 `kernel.path` 会被模拟器覆盖为 `<sysdir>\\kernel-ranchu`（实测启动后 `ksud debug version` 返回 **0**，即原装内核）。必须在 Studio 中使用 KernelSU，请按 [第 4 节方式 B](#方式-bandroid-studio-原生常驻持久化-持久生效推荐) 替换 `<sysdir>\kernel-ranchu`（并丢弃旧快照），替换后不带任何参数启动即得 `Kernel Version: 33310`。
3. **管理器版本务必与内核对齐**：实测官方 Release `33294` 管理器配 `33310` 内核可正常运行（本仓库的 HMA-OSS 安装器会给出"管理器与驱动版本不匹配"警告），但最终已改用同 commit 的 CI 产物 `..._33310-release.apk`；对齐后安装器侧的 `KSU_VER_CODE == KSU_KERNEL_VER_CODE` 才成立。完整流程与签名校验见 [第 7 节](#7-管理器-apk-与内核版本对齐manager-apk-matching)。
4. **`adb shell su -v` 不能用于验证 KernelSU**：`google_apis` 镜像自带 setuid root 的 AOSP 调试 su（`/system/xbin/su`，在只读系统分区上），它只接受 `su [uid] [gid] [命令]`，且不经 KernelSU 也能提权。正确的验证方式是 `ksud debug version` / `ksud debug info`，或观察内核日志中的 `KernelSU: ksu fd installed` 与 `allow root for: <uid>`。若要给 `adb shell` 提权，请在管理器界面将 Shell 加入授权名单。

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
- **Direct Kernel Boot**: Boots directly via emulator `-kernel bzImage`, or by replacing the system image's default kernel `<sysdir>\kernel-ranchu` (required for Android Studio launches — see below).
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
   For **Android Studio** (which runs `emulator -avd <AVD>` with no `-kernel`), replace the system
   image's default kernel instead — `config.ini`'s `kernel.path` is overwritten by the emulator:
   ```powershell
   $sys = "F:\android\system-images\android-36.1\google_apis\x86_64"
   Copy-Item "$sys\kernel-ranchu" "$sys\kernel-ranchu.stock"   # backup once
   Copy-Item path\to\bzImage "$sys\kernel-ranchu" -Force
   Rename-Item "$env:USERPROFILE\.android\avd\<AVD_NAME>.avd\snapshots\default_boot" "default_boot.bak-stockkernel"
   ```
   Verify with `adb shell uname -r` and `adb shell /data/adb/ksud debug version` (`0` means the stock kernel is still in use).
6. Install the matching KernelSU Manager APK, then verify in-kernel status:
   ```powershell
   adb shell /data/adb/ksud debug version   # -> "Kernel Version: 33310"
   adb shell /data/adb/ksud debug info      # -> runtime_mode: built-in
   ```
   Grant root per-app from the Manager UI. Note: `/data/adb/ksud` only exists after the Manager is installed — reboot once after installing it.

   > **Manager APK matching**: the kernel embeds the trusted manager signature hash, and the
   > KernelSU versionCode equals `30000 + <commit count>` of the KernelSU checkout. The official
   > release usually lags behind `dev`, so fetch the `manager` artifact from upstream's
   > `Build Manager CI` run at the *same commit*, then verify with
   > `apksigner verify --print-certs` that its certificate SHA-256 equals the
   > **Manager signature hash** reported in this workflow's job summary.
   > Full procedure: [section 7](#7-管理器-apk-与内核版本对齐manager-apk-matching).

---

## License

This project is licensed under the [MIT License](LICENSE) (or Apache-2.0). Kernel sources and KernelSU adhere to their respective GPL-2.0 upstream licenses.
