# AVD KernelSU Action

Automated GitHub Actions CI pipeline to build an **x86_64 Android Emulator (AVD) Kernel with integrated KernelSU / KernelSU-Next / SukiSU-Ultra / ReSukiSU** using Google's official Kleaf / Bazel build system.

[English](#english) | [中文说明](#中文说明)

---

<a name="中文说明"></a>
## 中文说明

在 GitHub Actions 上为 Android 虚拟机 (AVD x86_64) 源码编译内置 KernelSU / KernelSU-Next / SukiSU-Ultra / ReSukiSU 的内核镜像 (`bzImage`)。构建目标严格对齐官方 `virtual_device_x86_64_dist`，与原装系统驱动 100% KMI 兼容。

### 1. 工作流参数说明

进入 **Actions** -> **Build AVD KernelSU** -> **Run workflow**：

| 输入参数 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `kernel_manifest_branch` | `common-android16-6.12-2025-09` | AOSP Manifest 分支（其他系统从 `adb shell cat /proc/version` 提取 `-ab<ID>` 到 [Android CI](https://ci.android.com/) 查看） |
| `ksu_flavor` | `KernelSU-Next` | 变种：`KernelSU-Next`、`KernelSU`、`SukiSU-Ultra`、`ReSukiSU` |
| `ksu_version` | `dev` | 分支/Tag/Commit（KernelSU-Next 默认为 `dev`，其余变种自适应为 `main`） |
| `build_target` | `//common-modules/virtual-device:virtual_device_x86_64_dist` | Kleaf 构建目标（切勿修改为普通 GKI 目标） |
| `lto_mode` | `none` | 默认 `none` 防止 Runner OOM，加速编译 2~3 倍 |
| `use_fast_config` | `true` | 是否附加 `--config=fast` |
| `extra_bazel_args` | `--jobs=4` | 传递给 Kleaf/Bazel 的额外并发参数 |
| `enable_susfs` | `false` | 是否编入 SuSFS 内核级 Root 隐匿补丁（自动匹配分支并生成 x86_64 `ksu_susfs` 工具） |
| `susfs_branch` | `""` | SuSFS 分支（留空则根据内核版本自动匹配） |
| `create_release` | `false` | 是否自动发布 GitHub Release |

编译耗时约 25~35 分钟。构建产物（Artifacts / Release）包含 `bzImage`、校验码、配套管理器 `.apk`、`virtual_device_modules.tar.gz` 及 `ksu_susfs`（若启用）。

### 2. AVD 部署与持久化

获得 `bzImage` 后，根据启动方式部署：

#### 命令行启动 (CLI)
```powershell
emulator @<AVD_NAME> -kernel path\to\bzImage -show-kernel
```

#### Android Studio 常驻（替换系统内核镜像）
`config.ini` 中的 `kernel.path` 会被模拟器覆盖因而无效，正确方式为替换系统镜像的 `kernel-ranchu` 并丢弃旧快照：

```powershell
$sys = "F:\android\system-images\android-36.1\google_apis\x86_64"  # 替换为实际 SDK 镜像路径
Copy-Item "$sys\kernel-ranchu" "$sys\kernel-ranchu.stock"          # 首次备份
Copy-Item path\to\bzImage "$sys\kernel-ranchu" -Force

# 必须重命名或删除旧快照，防止恢复旧内核内存状态
Rename-Item "$env:USERPROFILE\.android\avd\<AVD_NAME>.avd\snapshots\default_boot" "default_boot.bak"
```

### 3. 验证与安装

1. **安装管理器**：工作流已自动打包匹配驱动版本与签名的 Manager APK，直接安装：
   ```powershell
   adb install -r <Manager>.apk
   ```
2. **状态验证**（首次安装 Manager 后需冷重启一次使 init.rc 钩子生效）：
   ```powershell
   adb root
   adb shell uname -r                              # 验证内核构建串
   adb shell /data/adb/ksud debug version          # 期望返回驱动版本（返回 0 表示仍是原装内核）
   adb shell /data/adb/ksud debug info             # 期望 runtime_mode: built-in
   ```
3. **SuSFS 验证**（若启用）：
   ```powershell
   adb push path\to\ksu_susfs /data/adb/ksu/bin/ksu_susfs
   adb shell chmod 755 /data/adb/ksu/bin/ksu_susfs
   adb shell /data/adb/ksu/bin/ksu_susfs show
   ```

---

<a name="english"></a>
## English Description

Automated GitHub Actions CI pipeline to compile an x86_64 Android Emulator (AVD) Kernel with built-in KernelSU / KernelSU-Next / SukiSU-Ultra / ReSukiSU (`bzImage`) via Google's Kleaf build system.

### Quick Start
1. **Fork** repository and enable **Read and write permissions** in Actions settings.
2. Run workflow **Build AVD KernelSU** with target branch and flavor.
3. Download artifacts (`bzImage` and matching Manager APK).
4. Deploy to AVD:
   * **CLI**: `emulator @<AVD> -kernel path\to\bzImage -show-kernel`
   * **Android Studio**: Replace `<sysdir>\kernel-ranchu` with `bzImage` and delete `<avd>\snapshots\default_boot`.
5. Install bundled Manager APK (`adb install -r *.apk`) and verify:
   ```powershell
   adb root
   adb shell uname -r
   adb shell /data/adb/ksud debug version
   ```

## License
[MIT License](LICENSE).
