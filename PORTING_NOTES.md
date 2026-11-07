# Xperia XZ2 Compact (xz2c) Android 17 适配全流程与问题解决思路

> 覆盖三个 ROM：**LineageOS 23（底座/device tree）**、**PixelOS 17（首个完整跑通的 A17）**、**RisingOS 9.0（当前主力，enforcing，功能 16/16）**。
> 设备：Sony Xperia XZ2 Compact（apollo / H8324 / xz2c），SoC **sdm845**，内核 **4.9.337-perf+ #18**，A/B 槽位 `_b`，序列号 `BH901L93BZ`。
> 时间跨度：2026-10 → 2026-11-07。本文是流程与方法论的总汇，具体逐字证据见 `functest/FINDINGS.md`、`KERNEL-4.9-feature-backports.md`、`KERNEL-4.9-perf-backports.md`。

---

## 目录

0. [核心结论（一页速览）](#0-核心结论一页速览)
1. [设备与内核基线](#1-设备与内核基线一切适配的前提)
2. [问题解决总方法论](#2-问题解决总方法论贯穿三个-rom)
3. [LineageOS 23 基础树适配](#3-lineageos-23-基础树适配)
4. [PixelOS 17 适配流程](#4-pixelos-17-适配流程首个跑通的-a17)
5. [RisingOS 9.0 适配流程](#5-risingos-90-适配流程当前主力)
6. [跨 ROM 共性问题类与修复模式](#6-跨-rom-共性问题类与修复模式)
7. [内核 4.9 特性回移系列](#7-内核-49-特性回移系列逐个验证)
8. [构建 / 刷机 / 部署环境](#8-构建--刷机--部署环境)
9. [发布 / 推送工作流](#9-发布--推送工作流)
10. [当前状态与遗留问题](#10-当前状态与遗留问题)

---

## 0. 核心结论（一页速览）

- **能在 4.9 内核上跑 Android 17，靠的不是 bionic 差异，而是内核回移。** 最关键的一笔是 `MADV_WIPEONFORK`（4.14 特性）回移进 4.9——没有它，A17 bionic 的 `arc4random`/栈金丝雀初始化在进程启动即 `EINVAL → abort → _exit(127)`，init 在挂载 root 后 ~7 ms 死亡，零日志。PixelOS 与 RisingOS 都靠同一回移才启动。
- **永不修改 `/system/bin/init` 二进制。** 任何替换 init 的镜像都在第一阶段启动失败（静态 logo、无 adb、~10 min 回 fastboot、pstore 空）。`adb root` 根本不需要改 init——它由开发者选项「以 root 身份调试」（`/data/adbroot/enabled`）控制。init.rc / ramdisk / cmdline 的修改是允许的。
- **A17 + 4.9 + A10 vendor blobs 的矛盾在每个子系统复现**，归为几大类：①内核特性缺失（回移，不改 bionic）；②SELinux `/dev/ion` 拒绝家族（单例缓存陷阱）；③vendor blob 与 A17 平台库尺寸/布局失配（GraphicBuffer sizeof、focus-stats 表项）；④GMS 的 EROFS APEX 在 4.9 不可挂（重打包为 priv-app APK）；⑤Sony ABL 启动模式 / vbmeta / slot 状态。
- **方法论是「证据优先」**：先读日志（dmesg/ramoops/tombstone）再动手；permissive 验证必须重启属主进程（单例把 `fd=-1` 永久缓存）；用时间戳而非行数算时长；用重启 cadence 区分 crash-loop（~55 s/次）与首刷 dexopt（30–40 min）；刷机只用原版 stock TWRP `fastboot boot → dd → hash 校验`，绝不 `fastboot flash`。
- **当前状态**：RisingOS 9.0，enforcing 默认，内核 #18，全功能 sweep **16/16 PASS**；性能默认 `bbr / fq / bpf_jit=1 / zram=zstd`（压缩比 ~5×）。源码已发布到 `xing29391` 的四个仓库。

---

## 1. 设备与内核基线（一切适配的前提）

### 1.1 硬件 / 分区拓扑

- **A/B 无缝更新**，测试都在槽位 `_b`。
- **recovery-as-boot**：`boot_b` 既是 boot 又是 recovery，二者不能共存——读任何失败启动的 pstore 都要先把 stock recovery 刷回 `boot_b`，读完再刷回 ROM 的 boot。
- **system-as-root**：槽位 `_b` 激活时 `system_b` 挂为 `/`；镜像内路径是 `/system/...`（如 `/system/build.prop`、`/system/product/priv-app/...`）。
- **无 dynamic partitions**（仅 APEX 用 dm-linear），**无 `metadata` 分区**（`getvar partition-size:metadata` → Variable not found）。
- 关键块设备：`boot_a=sda36`、`boot_b=sda46`、`system_b=sda52`、`userdata=sda73`（51,448,823,808 B）。`system` 分区精确 **4,227,858,432 B**（`build_image.py` 只允许 ~3968 MB 文件）。
- **vendor 镜像不含任何 `.ko`**（`/lib/modules`、`/vendor/lib/modules` 均不存在），所有驱动 built-in 进 `Image.gz-dtb`——所以「内核 vs 模块 vermagic 失配」**不是**可能的失败模式，别在这上面浪费 cycle。
- vbmeta：`vbmeta-off.img` = **Flags 3**（verification + verity 全关，仅 2 个 hash descriptor，无 hashtree）= 安全可刷；任一棵树新构建的 `vbmeta.img` = Flags 2（仅关 verity）。用 `external/avb/avbtool.py info_image` 读，手解析 header 会得到 nonsense。

### 1.2 内核天花板（为什么不能升级内核）

- 运行内核 `4.9.337-perf+`，**4.9.337 是 Linux 4.9 LTS 的最终版**（EOL 2023-01）。来源 LineageOS `android_kernel_sony_sdm845`，加本地回移。
- **4.14** 是唯一理论上的 Android  stepping stone，但 Sony 只为 kumano（SM8150）出过 4.14，sdm845 无官方 4.14 BSP；需要把平台代码 + kgsl + msm_camera + display/audio 从 4.9 移植，无完整功能 Android 先例。5.4/5.10 CAF 分支不含完整 sdm845 支持 ≈ BSP 重写。
- **Mainline**：apollo DTS 在 Linux 5.17 落地，postmarketOS 能启动 SoC，但缺 kgsl（Adreno userspace/blobs 依赖）、msm_camera、VoLTE/QMI 闭源栈——**跑不了 RisingOS/PixelOS**。mainline 版本号已到 7.x，不改变 Android 天花板。
- **硬件门**：SDM845（Kryo 385）是 **ARMv8.2-A**，所以 arm64 **MTE（8.5）/ PAC / BTI（8.3+）物理不可用**，与内核版本无关，别提议回移它们。
- 真正的阻塞因素是**闭源 Sony/Qualcomm userspace 编译 against 4.9 ABI**：libexcal 相机、Adreno/gralloc（kgsl ioctls）、display HWC、sensors SEE、ION。

> **结论 / 应用原则**：未来 Android  release 需要新内核特性时，**回移进 4.9**（范式：`MADV_WIPEONFORK`、`F_SEAL_FUTURE_WRITE` 两侧修复），而不是提议内核版本升级。被问「最大内核版本」时，答 Android 天花板 4.9.337，mainline 7.x 仅 postmarketOS。

### 1.3 核心矛盾

Android 17 的 userspace 假设 **post-4.9 内核 API** + **最新 vendor blobs**；本机是 **4.9 内核 + A10 vendor blobs**。这个落差在每个被触及的子系统里都会撞墙——适配工作本质上就是逐处弥合这个落差。

---

## 2. 问题解决总方法论（贯穿三个 ROM）

这些原则被反复验证，写在最前面以免重蹈覆辙：

1. **证据优先，不盲目重刷。** 启动失败先读 `/sys/fs/pstore/console-ramoops-0`、tombstone、dmesg，再决定下一步。用户明确纠正过「进入 twrp 分析日志」而不是迭代刷机。
2. **先怀疑平台内核版本门槛，再怀疑 device tree bug。** 早期 boot 失败往往是 A17 假设了远新于 4.9 的内核（bpfloader/netd/tethering APEX、bionic arc4random、memfd seal）。
3. **屏幕状态不是启动判决**：
   - Xperia 的 **fastboot 模式显示 Sony logo**——「卡在 sony 页面」可能设备其实在 fastboot，先查 `fastboot devices`。
   - **静态 Sony logo + 既无 adb 也无 fastboot** = adbd 启动前 hang。
   - **「还是充电页面」可能 TWRP 正常运行**（其灭屏显示像充电页）；查 PnP `05C6:F000` 证明 kernel+init 活着，让用户**短按电源键 ~1 s** 唤醒 TWRP UI。
4. **用时间戳算时长，绝不用行数**（有的迭代慢，行 ≠ 分钟）。**用 cadence 区分**：每 ~55 s 自重启一次（7 次/6.5 min）= crash-loop，再等也没用；首刷新 system 镜像的 dexopt 需 **30–40 min**（PixelOS 首刷 ~34 min），5–10 min 无 adb 不能判失败。
5. **permissive 验证必须重启属主进程。** `IonAlloc::Init()` 是单例，enforcing 下首次 open `/dev/ion` 失败（EACCES）后把 `ion_dev_fd_=-1` 永久缓存且不重试；`setenforce 0` 后旧进程仍用 fd=-1，ioctl 全 EBADF——「permissive 仍失败」是假象。必须 `kill $(pidof media.codec)` / `ctl.restart vendor.audio-hal` 强制重新 Init。
6. **avc 用 dmesg**：`adb shell 'dmesg | grep -a "avc:  denied"'`（**两个空格**）。`logcat -b kernel` 在同一窗口可能完全没有 avc。内核 audit 抑制重复拒绝（open 常被抑制而 read/ioctl 出现），相机崩溃刷屏会在几分钟内覆盖环形缓冲——触发失败后立即抓。
7. **镜像 / 部署一律 hash 校验**：刷后读回 md5/sha256 对比 host；离线用 `simg2img → debugfs dump → sha256sum` 验证**打包进镜像**的内容，而不是构建结果（`m systemimage` 可能 exit 0 却打包了旧镜像）。
8. **一次只回移一个内核特性，独立提交、独立验证**，当前镜像启动且 raw-syscall 测试通过后再叠下一个。
9. **永不改 `/system/bin/init` 二进制**（见 §4.3）。
10. **刷机只用原版 stock TWRP**：`fastboot boot twrp-3.7.1_12-0_LOCAL-20251206-01-apollo.img` → adb recovery → push → `dd` → hash 校验，**绝不 `fastboot flash`**。
11. **风险 / 不可逆动作先确认**：擦除 `userdata` 已获用户批准；擦除 `misc`/`persist`、force-push **未获批准**。`fastboot --set-active=b` 是廉价非破坏性的 A/B retry 重置。
12. **vendor blob 问题优先用分区放置 / 配置解决**，而不是重编 vendor HAL。

---

## 3. LineageOS 23 基础树适配

LOS23 是在 Sony 官方 device tree 之上的**底座**，其修复后来被 PixelOS / RisingOS 复用。

> 命名澄清：RisingOS 9.0 的 manifest 默认 revision 是 `refs/heads/lineage-24.0`，即 **LineageOS 24 = Android 17**。仓库名 `los23` 只是用户选定的名字，内容是 LineageOS 侧的适配文件。

### 3.1 发布内容

- **`github.com/xing29391/los23`（private）**：22 个 LineageOS 树文件——`build/soong` prebuilt_etc、`device/sony` 树、Connectivity bpf、ProcessList、qcom audio soundtrigger、vendor configs。
- **`github.com/xing29391/sony_xz2c_tree`（public）**：完整 `device/sony/{xz2c,tama-common}` 文本树 + `vendor/sony/*` 的 mk/bp（476 文件），**不含 proprietary blobs**。

### 3.2 关键适配点（被后续 ROM 继承）

- **cpuset / camera-daemon 修复**：`device/sony/tama-common/rootdir/etc/init.qcom{.power,}.rc`。4.9 不支持 `cpuset_v2_mode` → cgroup 初始化失败；这些 rc 修复在 Rising 树里与已发布版本 **byte-identical**，所以 Rising 自建的 boot.img 可直接用，无需借 PixelOS 的 boot。
- **Connectivity bpf 放松**、**ProcessList**、**qcom audio soundtrigger**、**vendor configs**：A17 的 bpf/netd 假设新内核，需放松。
- **vendor blobs** 提取自 stock firmware，保存在 `/root/keep/vendor-sony`（797 MB）；device tree 仓库故意不带 proprietary 文件（重建 blobs 需要 stock firmware 提取）。

---

## 4. PixelOS 17 适配流程（首个跑通的 A17）

目标：让 PixelOS 17（Android 17，API 37）在 xz2c 上完整启动可用（GMS/Play Store/相机/root），复用 LineageOS device trees + PixelOS 源码。

### 4.1 构建配置

- 树 `/root/android/pixelos`，`TARGET_PRODUCT=custom_xz2c`，`TARGET_RELEASE=cp2a`。
- **lunch 陷阱**：PixelOS 里 `lunch custom_xz2c-userdebug` 失败（"No release config set for target"），必须 `breakfast xz2c`；Rising 里相反（见 §5.1）。
- 触碰 init 后增量 `m systemimage` ~5 min。

### 4.2 启动门槛逐个击破

早期 boot 在多处失败，按触及顺序：
- `init_second_stage` 链接失败 → 修到能构建。
- **cpuset_v2_mode**（4.9 不支持）→ cgroup 初始化失败 → `init.qcom.rc` 修复。
- **vendor HAL 缺库**（RIL / 相机 / light）。
- **FCM 5/6 matrices** + light/touch HAL。
- **bpfloader / netd / tethering APEX** 假设远新于 4.9 的内核 → 放松。
- **GraphicBuffer sizeof**、**RefBase 栈检查放松**（见 §6.3）。
- **IMS 放 system_ext**：`libimsmedia_jni.so` 需解析 `libutils.so`；把 IMS APK + JNI 从 product 移到 system_ext（早期曾 disable `org.codeaurora.ims` 止崩溃循环，代价是丢 VoLTE，后由 system_ext 放置真正解决）。

### 4.3 init 危险（硬规则的由来）

- PixelOS 的 `SetSafetyNetProps()`（`system/core/init/property_service.cpp`）在 `PropertyInit()` 里、**任何分区 `.prop` 加载之前**被调用，force-set `ro.build.type=user` + `ro.debuggable=0`（除非 ENG_BUILD）。`ro.` 属性写一次，logcat 出现 `Could not set 'ro.build.type' to 'userdebug' ... already set`。**build.prop 一直是对的**，vendor build.prop 不是元凶。
- 构建系统坑：`PRODUCT_NOT_DEBUGGABLE_IN_USERDEBUG := false` 序列化成 JSON `true`（`add_json_bool` 把任何非空值当 true）——要禁用必须**留空**。
- 尝试修复：删掉那两行 `InitPropertySet` 重编 init → **永不启动**（静态 Google logo、无 adb、无 fastboot、~10 min 回 fastboot、pstore 空）。甚至**最小 2-string 等长字节补丁**（把 `ro.build.type`→`ro.build.mode`、`ro.debuggable`→`ro.debug_mode`，让 spoof 写变成无害 no-op）也失败。
- **结论：永不修改 `/system/bin/init`。** 而且 init 补丁**根本不需要**（见下）。

### 4.4 adb root 的真相（不需要改 init）

- adbd 的 `adb root` gate 在 `packages/modules/adb/daemon/restart_service.cpp`：要求 `(ANDROID_DEBUGGABLE || __android_log_is_debuggable())` **且** framework `AdbRootService` 报告 enabled。
- `AdbRootService`（`packages/modules/adb/root/adbroot_service.cpp`）只是读 `/data/adbroot/enabled`。开关是**开发者选项 → 「以 root 身份调试」**（`adb_enable_root`，zh 标签「以 root 身份调试」）。
- 启用方式：`settings put global development_settings_enabled 1` → 打开 `android.settings.APPLICATION_DEVELOPMENT_SETTINGS` → 下滑点该行 → `adb root` 返回 `restarting adbd as root`，shell = `uid=0(root) context=u:r:su:s0`。
- **持久跨重启**；data wipe 会重置，需重开（或一旦有 root 重写 `/data/adbroot/enabled`）。
- `ro.build.type` 仍报 `user`、`ro.debuggable` 仍 `0`（SetSafetyNetProps spoof 仍在）**不阻止 root**——gate 因镜像本身是 userdebug 构建已通过。所以**放着 init 别动**。

### 4.5 GMS 交付（EROFS 不可用）

- shipped GMS APEX payload 是 **EROFS**，4.9 内核无 `fs/erofs` → 不可挂。
- 解决：重打包为**特权预置 app** `/system/product/priv-app/PrebuiltGmsCore`（217 MB，`preprocessed: true`）+ 特权 allowlist `product/etc/permissions/privapp-permissions-google-p.xml`。
- 运行时验证：Play Store 可用、`com.google.android.gms.persistent` 无崩溃循环。

### 4.6 MADV_WIPEONFORK 内核回移（为什么 PixelOS 能在 4.9 跑）

- 2026-10-24 用 `/root/patch-kernel.py` 手工回移 4.14 的 `MADV_WIPEONFORK`（6 文件：`mman-common.h`、`mm.h` VM_ARCH_2→VM_WIPEONFORK、`mmflags.h`、`task_mmu.c`、`fork.c`、`madvise.c`）。
- 这让 A17 bionic 的 `arc4random` / 栈金丝雀初始化成功。
- **证明**：PixelOS 自己的 bootstrap `linker64`/`libc.so` 在**未打补丁**的 4.9 内核下 chroot（OrangeFox）会 abort：`arc4random data MADV_WIPEONFORK failed: Invalid argument`（rc 134）。
- **应用原则**：把内核回移当作「PixelOS 17 在此能跑」配方的一部分；任何 A17 ROM 在此设备都需要它。PixelOS 树已删，但补丁脚本与移植副本（`xz2flash/rising9/patch-kernel-rising.py`）存活——**重新应用，不要重新推导**。

### 4.7 相机修复

见 §6.3（GraphicBuffer sizeof + focus-stats 表项），在 vendor47/vendor50 落地。

### 4.8 导航栏 setup 故障（「半截显示」）

- 3-button 导航时，nav bar 是 **Launcher 的 Taskbar 窗口**（type NAVIGATION_BAR，uid `com.google.android.apps.nexuslauncher`），不是 SystemUI。窗口高 48dp（144px），但 `user_setup_complete=0` 时 launcher 把内层 NearestTouchFrame 设为 setup 高度 96dp（288px，`taskbar_suw_frame`）；容器底锚定 → 中心落在窗口顶边 → 图标只画下半进窗口。
- setup 完成即自愈（`TaskbarManagerImpl` 观察 USER_SETUP_COMPLETE 重建 taskbar），无需重启。
- 为何只在此 port：launcher 是 **Google prebuilt**（`NexusLauncherRelease.apk`，不能改 Launcher3 源码）；AOSP 手机 setup 期走 gesture nav；xz2 device tree 无 nav-mode override → `config_navBarInteractionMode` 默认 0（3-button）。
- **修复（新装默认 gestural）**：在 `device/sony/xz2c/overlay/frameworks/base/core/res/res/values/config.xml` 加 `<integer name="config_navBarInteractionMode">2</integer>`。
- **坑**：`PRODUCT_ENFORCE_RRO_TARGETS := *` 时 `DEVICE_PACKAGE_OVERLAYS` **不并入** `framework-res.apk`（重编它得到 byte-identical APK），而是编译成 **vendor 分区上的静态 RRO**：`/vendor/overlay/framework-res__custom_xz2c__auto_generated_rro_vendor.apk`（package `android.auto_generated_rro_vendor__`，isStatic，priority 2，target android）。用 `m FrameworkOverlayCustom` 构建，**只把该 APK 补进 vendor 镜像**——别重编整个 vendor 镜像（会丢手动相机/libutils 补丁）。
- 运行时 navbar RRO（`com.android.internal.systemui.navbar.*`）priority=2147483647，用户显式选择仍覆盖新默认；默认只在无 navbar RRO 启用时（新 userdata）生效。
- 落地为 `vendor52.img`（md5 `974ddcd4b1ef1677115cfd80e2e61371`）。framework-res.apk / system 镜像无需为此改动。

### 4.9 已知良好镜像与遗留

- **`system24.img`**（md5 `327b1c17842278972ee0547f975f3248`）= `pixelos17-system-gms.raw` + build.prop `ro.debuggable=1` / `ro.adb.secure=0` / `persist.sys.usb.config=adb,mtp`，**保留旧 init**。验证：GMS/Play Store 在位、`gms.persistent` 稳定、相机 HAL 出流（ICP 固件加载、无 focus-stats 崩溃）、pm-service 稳定、冷启动 ~50 s。
- **通话 / VoLTE 已验证**（system24，2026-10-30）：VoLTE 在 boot 后 ~20 s 注册（`ImsPhone: onImsMmTelConnected`，`isVolteEnabled=true`，「联通IMS」APN 带 P-CSCF），拨 10010 经 LTE 接通（networkType=13），shell `am start -a android.intent.action.CALL` 与 Google Dialer UI 均可，call log 有记录。
- 遗留（非致命）：`vendor.qti.hardware.alarm@1.0-service` 每 boot abort 一次（旧 vendor `libhidlbase` ClientCounterCallback vs A17 hwservicemanager，仅影响关机闹钟）；IMS workaround；状态栏警告三角清理。
- **重要教训**：「卡在 Google logo」从来不是 hang—— pristine `gms.raw` 镜像确实能完成启动（`sys.boot_completed=1`），首刷因 dexopt + 内存压力（~50 MB free，swap 670 MB）耗时 ~34 min。**别凭 logo 判启动，先查 `adb devices`/`sys.boot_completed`。**

---

## 5. RisingOS 9.0 适配流程（当前主力）

目标（2026-10-31 起）：PixelOS 17 完成后，为同一设备适配 RisingOS Revived 9.0（"elefthería"，Android 17）。因为 PixelOS bring-up 已证明 A17 在此 4.9 内核 + 旧 blobs 需要什么，此 port 主要是「**重新应用已知良好的平台补丁**」，而非新诊断。

### 5.1 源码同步与产品配置

- manifest `https://github.com/RisingOS-Revived/android`，分支 **`seventeen`**；`default.xml` 用 AOSP `android-17.0.0_r1` tag + 默认 revision `refs/heads/lineage-24.0`（= LineageOS 24 = A17）。`vendor/rising` 与 `vendor/lineage` 都在 manifest 里。
- **`repo init` 自更新失败**（`gerrit.googlesource.com` 超时）→ 传 `--repo-url=https://mirrors.tuna.tsinghua.edu.cn/git/git-repo`；AOSP 侧已在 `/root/.gitconfig` 重写为 tuna 镜像。
- 树 `/root/android/rising`；同步 `repo sync -c -j6 --no-clone-bundle --optimized-fetch`。
- **lunch**：`lunch rising_xz2c-userdebug` 可用；`breakfast xz2c` 非交互运行会 abort（与 PixelOS 相反）。
- 产品 `device/sony/xz2c/rising_xz2c.mk` + 重写 `AndroidProducts.mk`；删 `custom_xz2c.mk`（RisingOS 无 `vendor/custom`）。
- **LINEAGE_BUILD 坑**：`build/make/core/config.mk` 只在 `LINEAGE_BUILD` 设置时 include `vendor/lineage/config/BoardConfigLineage.mk`，而 `check_product()` 只为 `lineage_*` lunch 目标设它；RisingOS lunch `rising_*` → 内核变量（`KERNEL_BUILD_OUT_PREFIX`、`KERNEL_MAKE_CMD`…）从未导出到 Soong → `generated_kernel_includes` 报 "unknown variable"。**修复**：在 `device/sony/xz2c/BoardConfig.mk` 里，`TARGET_KERNEL_CONFIG` 之后，当 `LINEAGE_BUILD` 为空时手动 `include vendor/lineage/config/BoardConfigLineage.mk`（及 `device/lineage/sepolicy/common/sepolicy.mk`）。

### 5.2 复用 PixelOS 平台补丁（只取我们的 hunk）

移植并验证为 only-our-hunks：Connectivity bpf set（NetBpfLoad、netbpfload rc ×2、BpfHandler、NetdUpdatable、libeventpolling ×3、LocalNetEventListener）、`system/bpf/loader/Loader.cpp`、`RefBase.cpp`、`libmeminfo/sysmeminfo.cpp`、FCM 5/6 matrices + Android.bp（从安装列表删 `6.android17` 模块）、gpt-utils `"false"` select key。**GraphicBuffer 只取 lazy `unique_ptr<DependencyMonitor>` 布局改动**（保留 Rising 更新的 getDataspace/GraphicTypes 代码）。`system/core/init/*` **故意不移植**（硬规则）。

### 5.3 配置阶段 blocker

- **ModemConfig 幽灵模块**：`vendor/lineage/config/common.mk` 把它列进 `PRODUCT_PACKAGES` 但全树无此模块 → kati "non-existent modules" abort（PixelOS 从未强制）。修复：`device/sony/tama-common/Android.mk` 里加 `BUILD_PHONY_PACKAGE` no-op。
- `vendor/microg/FDroid/Android.bp` 改名 `.disabled`（FDroid.apk 缺失且 f-droid.org 不可达；`WITH_GMS=true` 下 microG 无用）。之后 `m nothing` 达 EXIT=0。

### 5.4 GMS LFS 陷阱 + system 分区容量

- **LFS 陷阱**：`vendor/gms` 是 Git-LFS 仓库，`repo sync` 静默把 7 个大 blob 留成 134 B 指针文件（Phonesky、Maps、Photos、Velvet、PrebuiltBugle、DevicePersonalizationAiAiPrebuiltPixel2026、`apex/com.google.android.gmssystem.prodvic.apex`）。失败只在构建 ~6 h 后 `enforce_uses_libraries`（"failed opening zip: Invalid file"）才暴露。**任何 sync 后**用 `find vendor/gms -size -200c` + `head -c 20` 检查，`git lfs pull --include=...`（gh-proxy remote 使其快）。
- **system 容量**：product+system_ext 合并进 `system.img`，Sony 分区精确 4,227,858,432 B → `build_image.py` 只允许 3968 MB 文件。`gms_full.mk` 下树是 4995 MB → 打包失败（`e2fsdroid ... Could not allocate block`）。erofs 不可选（4.9 无 `fs/erofs`）。**解决**：在 `device/sony/xz2c/rising_xz2c.mk` 里、`common_full_phone.mk` inherit **之前**设 `TARGET_USES_PICO_GAPPS := true`—— dropping 39 模块省 ~1233 MB（Velvet 351、Photos 180、Bugle 161、DevicePersonalizationAiAi 102…），加回 27 MB VelvetGo；Play Store 可重装被 drop 的 app。Mini（~500 MB）不够。
- **有效 makefile 是 `vendor/gms/gms_pico.mk`**（xz2c 设了 pico），`gms_full.mk` 的编辑在此设备**无效**。编辑前用 `get_build_var PRODUCT_PACKAGES | tr ' ' '\n' | grep -x <module>` 确认有效变体。

### 5.5 内核 vs 新 clang + 构建提速

- `KERNEL_MAKE_FLAGS += KCFLAGS=-Wno-error=implicit-enum-enum-cast`（QCACLD 在 QDF_STATUS 函数里 `return A_ERROR;`），内核随后 green。
- **固定构建日期**：`RISING_BUILD_DATE ?=` in `vendor/rising/config/version.mk` + `export RISING_BUILD_DATE=261101-0820`。日期落进 `soong.variables`，每分钟变化的值会强制每次 ~50 min 全量重分析。`export NINJA_ARGS="-k 0"` 让一遍报告所有错误。
- **每次 `m` 都重跑全量 Soong 分析**（~50 min；`soong_build` ~21 GB RSS 对 22 GB WSL VM，重 swap， persistent "memory stall"）。**批量编辑一次构建**；长构建用 Bash 后台任务（存活），日志放 `/root`。
- **快速路径（无 Soong 分析）**：`prebuilts/build-tools/linux-x86/bin/ninja -f out/combined-rising_xz2c.ninja <target>`——内核/uapi 编辑后几分钟出 `boot.img`；sepolicy 增量 ~2 min。**wrapper 可能 exit 0 但 ninja 失败**——判 `grep 'error:'` 日志 + 产物 mtime/md5，别只看 exit code。

### 5.6 启动诊断长跑（两个独立 bug + 内核，大量假设被关闭）

首刷 system_b+boot_b+dtbo_b（vendor_b 留 vendor52）→ ~9 min 无 adb 回 fastboot；第二次用 Rising 自己的 vbmeta 经 `--disable-verity --disable-verification` → **"Your device is corrupt"**。诊断关闭了一长串假设：

- **`/data` 被 erase 但从未 format**（真实 bug，但非根因）：`userdata`（sda73）LBA0 与 1024 处 superblock 全 `00`，dmesg `EXT4-fs (sda73): VFS: Can't find ext4 filesystem`。`fastboot erase userdata` 不留文件系统，Rising `fstab.qcom` 用 `wait,check`（只修不格），无 metadata 分区 → **任何 ROM 都挂不了 /data**。TWRP 里 `mke2fs -F -t ext4 -b 4096 -O project,quota -L userdata -m 0 /dev/block/sda73` + `restorecon -RF /data` 修复。注意 `check`/`reservedsize`/`project` 不是内核 ext4 挂载选项（fs_mgr 消费），用 `mount -o` 测会得误导性 Invalid argument。
- /data 修好后仍 **~55 s crash-loop**（7 次/6.5 min，205 s/3 次≈68 s），全程零 USB → ABL 自动进 fastboot（`slot-retry-count:b:0 slot-unbootable:b:yes`）+ "corrupt" 屏。`getvar all` 报 **`unlocked:yes secure:no`** → 该屏是 ABL 的 **OS-failed-to-boot fallback，不是 AVB**。「corrupt=混合 vbmeta」「AVB」理论均关闭。~55 s cycle 是 crash/reboot loop，**不是 dexopt**，「等 30–40 min」不适用此 ROM。
- **静态比较排除**：ramdisk 结构（661 条目相同，`fstab.qcom` byte-identical md5 `04c5bde5…`）；ext4 feature set（`dumpe2fs` == 工作的 system25：`uninit_bg dir_nlink extra_isize`，4K blocks，256 B inodes）→ 无新 fs 特性 / 4.9 失配。
- **vendor 无 .ko**（全 built-in）→ vermagic mismatch 不可能。
- **电源两次排除**：TWRP 会话 `healthd: battery l=100 v=4293` 持续 300 s；fastboot `getvar battery-voltage` → 4274 mV。`usbin-collapse`/`typec-change: Type-C NONE` 是 gadget 重绑引起的 VBUS 重检测，不是电池 dying。
- **ramoops 取证陷阱**：运行中 `cat /sys/fs/pstore/console-ramoops-0` 返回字符交错垃圾（本会话 console 覆盖同一 1 MB 记录）；唯一可用形式是 `on early-init` 的 `copy`。每次 ABL retry 重置 1 MB ramoops → 7 次后撕裂（`232 corrected bytes, 1068 unrecoverable blocks`，丢 `Kernel command line`）。**所有早期 ramoops 捕获其实都是 TWRP 自己的 boot，不是 Rising 的**（cmdline 指纹 `twrpfastboot=1`/`zram.backend=z3fold` 匹配 TWRP）。

### 5.7 根因：内核缺 MADV_WIPEONFORK（kernel bug，非 ROM/ramdisk/init/AVB）

- **证据链**：ramoops `pstore_harvest/console-ramoops-0.txt` 显示 `VFS: Mounted root (ext4) readonly` @4.108 s → `Freeing unused kernel memory` @4.110 s → `Kernel panic - not syncing: Attempted to kill init! exitcode=0x00007f00` @4.117 s，**零 init 日志**（0x7f00 = exit 127）。OrangeFox 里 on-device 复现：`chroot /mnt/sb /system/bin/bootstrap/linker64 --help` → `arc4random data MADV_WIPEONFORK failed: Invalid argument` / `Aborted`，rc=134。
- **机制**：A17 bionic 在进程启动播种栈金丝雀（`bionic/libc/bionic/__libc_init_main_thread.cpp:126`，`__libc_arc4random_buf_or_die`），其 `_rs_allocate()`（`arc4random.h`）做 `madvise(p,size,MADV_WIPEONFORK)` 并在 `-1` 时 `async_safe_fatal`。MADV_WIPEONFORK 是 **4.14 特性**，`kernel/sony/sdm845/mm/madvise.c` 无此 case → EINVAL。`abort()` 终于 `_exit(127)`，PID 1 忽略 SIGABRT → init 在 root mount 后 ~7 ms 死亡，来不及日志。（ABL 传 `skip_initramfs … init=/init root=PARTUUID=<sys_b>`，boot ramdisk 无关——kernel + /system 是唯一变量；`/system/bin/init` 仍永不可改。）
- **为何 PixelOS 能在「同一」内核跑**：它从未跑这个内核——其树在 2026-10-24 打了同一回移。PixelOS 自己的 bootstrap libc/linker 在未打补丁 4.9 下同样 EINVAL abort，所以**差异从来不是 bionic，而是内核**。
- **修复**：把证明过的补丁移植到 `/root/android/rising/kernel/sony/sdm845`（6 文件：`mman-common.h` +MADV_WIPEONFORK 18/KEEPONFORK 19、`mm.h` VM_ARCH_2→VM_WIPEONFORK 0x02000000、`mmflags.h` smaps "wf"、`task_mmu.c`、`fork.c`（child anon_vma=NULL + 跳过 copy_page_range）、`madvise.c`）。备份 `/root/kern-backup-rising/`，脚本 `xz2flash/rising9/patch-kernel-rising.py`。`m bootimage -j20` EXIT=0 ~5 min；解压 Image.gz-dtb 含 `wipeonfork` 字符串验证。
- **应用原则**：这类失败**别改 bionic，回移内核 flag**。init 若再 exit 127/0x7f00 且无日志，先跑 chroot `linker64 --help` 探针——5 s 复现整个 fault，无需 boot cycle。

### 5.8 Sony ABL recovery-mode 循环（补丁内核后的下一关）

- 补丁内核到 init @4.43 s（无早期崩溃，`wipeonfork` 在），但 init 日志 `First stage mount skipped (recovery mode)`，加载 boot ramdisk 的 `/sepolicy` 失败（`SELinux: avtab: invalid type or class` → `InitFatalReboot: signal 6` → `reboot: Restarting system with command 'recovery'`）。
- **原因是 boot 模式，不是 ROM**：Sony ABL 用存储的 warm-reset reason。循环 boot 都 `warmboot=0x77665502`（= reboot recovery，无 `skip_initramfs`）；正常 boot `warmboot=0x77665501` 带 `skip_initramfs`。`misc` 全零（TWRP 运行时）→ BCB 不是触发；init 自己的 fatal restart 重新武装 recovery reason，循环自维持。`adb reboot` 不清它。
- **修复（device tree）**：`BOARD_KERNEL_CMDLINE += androidboot.force_normal_boot=1` in `device/sony/tama-common/BoardConfigCommon.mk`。init 的 `IsForceNormalBoot()` 只看 bootconfig/cmdline 里该字面量 → 无论 ABL reset reason 都把 Rising 钉在正常第一阶段 mount 路径。
- **快速迭代替代**：原地补 boot header 的 cmdline 字段（`0x40..0x240`，512 B cap）——秒级出 `rising9-boot-wf-fnb.img`，而全量构建要跑（改 BoardConfig 强制 ~46 min 全量 Soong 分析）。

### 5.9 userspace stall：SELinux enforcing + /dev/ion 标签

- 每个早期「init 后无 USB」症状背后的 userspace stall 根因：SELinux **enforcing** 而 `/dev/ion` 标为 `u:object_r:device:s0`（无 file_contexts 匹配）→ `vendor_hal_keymaster_qti` / `vendor_hal_gatekeeper_qti` 每 ~5 s 在 `avc: denied { read } … name="ion" … permissive=0` 上 abort → keystore2 永远不能 init/注册 `android.security.maintenance` → vold 阻塞在 `AServiceManager_waitForService("android.security.maintenance")`（在 `on post-fs`/`on post-fs-data` 的**阻塞** `exec vdc checkpoint …` 里）→ init 永远到不了 `class_start early_hal`/zygote → RESIN reset。
- PixelOS 已知良好 cmdline 是 `printk.devkmsg=on androidboot.selinux=permissive`，其相同 ion 拒绝显示 `permissive=1`。
- **临时修复（无需重编内核）**：header 补 v1 cmdline 追加 `printk.devkmsg=on androidboot.selinux=permissive` → boot ~60–135 s 完成（`sys.boot_completed=1`，Android 17，RisingOS CP2A.260605.016，permissive，keystore2/zygote/system_server/SystemUI/NexusLauncher 起来，UI 渲染）。**permissive 只是 bring-up 拐杖，最终 shipped 状态是 enforcing**（见 §5.11）。

### 5.10 GMS 无 EROFS 交付 + 版本一致

- **GMS 缺失因为内核缺 `CONFIG_EROFS_FS`**（不是 apexd/dm-verity bug）：`com.google.android.gmssystem.prodvic.apex`（146 MB，EROFS payload）激活失败 `apexd: Mounting failed … No such device`，唯一的 `com.google.android.gms` base APK 在该 payload 里。**先查 `/proc/filesystems` 有无 erofs 再 theorize**。
- **交付**：`unzip` apex → `apex_payload.img`，WSL 里 mount EROFS，取 `PrebuiltGmsCoreVic.apk`（217,107,692 B，version 26.11.36）+ 两个 apex XML，作为正常 product priv-app 装到 `/system/product/priv-app/PrebuiltGmsCore/PrebuiltGmsCore.apk`（0644 root:root，restorecon），XML 放 `product/etc/permissions` + `product/etc/sysconfig`。permissive + `ro.control_privapp_permissions=log` → 无 allowlist fatal。树集成在 `vendor/gms/gms_pico.mk`（apex 条目移除 + PRODUCT_PACKAGES + PRODUCT_COPY_FILES）。
- **「重启后 Google 服务异常」根因（已修）**：pico bundle 内部不一致——base 容器 `PrebuiltGmsCore` **26.11.36** vs 预载 Chimera 模块 **26.18.43**。Chimera 因缺 `CronetDynamiteModuleApi[48]` 丢弃模块，从不写 `current_config.fb`，`GmsAppComponentFactory` 每 boot ANR 循环。**修复**：只在 `/system/product/priv-app/PrebuiltGmsCore` 装**版本匹配**的模块（AdsDynamite、AndroidPlatformServices、TfliteDynamitePrebuilt）；Play 按需装新模块。验证：`current_config.fb` 生成，两次重启 0 ANR，Play Store 渲染登录页。
- 设备quirk：GMS 装后首 boot 跑重 dex2oat 风暴，忙时每 ~8–30 min watchdog 重启，空闲稳定；约半数 warm reboot 在第一阶段 abort 一次（`init: Cannot create /first_stage_ramdisk/system/bin: Read-only file system` → InitFatalReboot → recovery），下次成功。

### 5.11 默认 enforcing + 逐个 SELinux 功能修复

cmdline 前置 `androidboot.selinux=enforcing`（permissive 退役）。随后逐个修 enforcing 下的功能缺陷（全部 on-device 验证）：

- **开机动画白屏**：enforcing 下 `avc: denied { open read ioctl(0x4905) } comm="BootAnimation" name="ion" … ion_device … permissive=0`；无 ION 则 in-process gralloc mapper 不能分配缓冲，`eglSwapBuffers` 循环 `EGL_BAD_SURFACE (300d)`，面板白。`setenforce 0` + `runcon u:r:bootanim:s0 /system/bin/bootanimation` 正常渲染（关闭了「坏 zip/GL 驱动/错 layer」理论；动画 zip 与内置 byte-identical）。**修复**：`allow bootanim ion_device:chr_file rw_file_perms;` in `sepolicy/private/legacy_ion.te`（落 **product** policy）+ composer 读 `bootanim_system_prop` 规则 in 新 `sepolicy/vendor/bootanim.te`。
- **Bilibili 视频播放闪退（F_SEAL_FUTURE_WRITE 两侧修复）**：A17 framework 把 >12KB bitmap 用 `F_SEAL_FUTURE_WRITE`（Linux 5.1+ flag）封 memfd 并以 `MAP_SHARED` 读回；4.9 无此 seal → 原调用 EINVAL，每个 >12KB bitmap 写失败。**两侧**：写回退 `F_SEAL_WRITE` 修写，但读者 `MAP_SHARED` mmap 又 EPERM（`mapping_deny_writable`→`mapping_map_writable` 因 VM_SHARED 隐含 VM_MAYWRITE 连 PROT_READ 都拒）→ 读侧对 immutable blob 改 `MAP_PRIVATE`。两处都在 `frameworks/base/libs/hwui/jni/Bitmap.cpp`。**通用 port 约束**：其他依赖 5.1+ memfd seal（及 sealed memfd 的 MAP_SHARED 读者）的 A17 路径会同样失败。
- **Touch HAL 崩溃循环**：上游 Lineage 删了 HIDL touch 规则 → `vendor.lineage.touch@1.0-service.tama` 把 `IGloveMode` 注册到 `default_android_hwservice`，add/find 被拒 → `registerAsService()` 失败，每 ~5 s exit 1。恢复 `hal_attribute_hwservice` for `hal_lineage_touch` + `hal_lineage_touch_hwservice` 类型 + `vendor_hwservice_contexts` 里 `vendor.lineage.touch::IGloveMode`。验证：`init.svc.vendor.touch-hal-1-0=running`，稳定 pid，`lshal` 列 IGloveMode/default。
- **GNSS/QMI**：`android.hardware.gnss@2.0-service-qti` 不能 create 控制 socket（`qmi_cci_xport_port_init … EACCES -13`，每秒，GPS 无定位），只有 `self:socket read`。给 `vendor_hal_gnss_qti`/`vendor_location` `self:socket { create ioctl write getattr setopt getopt connect }`。验证：零重复 QMI 错误，`dumpsys location` 报 modem 串 `qcom;MPSS.AT.4.0.c2.16-00052-SDM845_GEN_PACK-1`（HAL/modem 链路初始化）。（未做室外 fix 测试。）
- **蓝牙工厂 MAC**：`hal_bluetooth_default` 无读 `bluetooth_vendor_data_file` → `/data/vendor/etc/bluetooth_bdaddr` 不可读，HAL 用生成地址。加规则后 `settings get secure bluetooth_address` == 工厂值 **D4:38:9C:9A:9C:6A**。
- **通话（「打不了电话」）**：Sony modem-switcher 用 `persist.vendor.somc.cust.modem0` 组合 `/vendor/modemconf/<值>/modem.conf`，无值则选非 IMS MCFG。设备设 `persist.vendor.somc.cust.modem0=S285.1` + `persist.vendor.somc.sim0=S285.1`；真实拨 **10010 ACTIVE 24 s over IMS/LTE**（`dumpsys telecom` state=ACTIVE，QImsService accTechStatus mode=14，CallFailCause Code=0）。默认值加进 `device/sony/xz2c/vendor.prop`，但只在重编/刷 vendor.img 后落地——data wipe 后 persist prop 需重设直到该镜像 ship。
- **录像**：mediacodec/mediaserver → ion_device 拒绝（见 §6.2）。
- **通话音频（对方无声 + 下行杂音）**：hal_audio_default → ion_device 拒绝 → ACDB 失败（见 §6.2）。

> 这些修复运行时 push 到 `/vendor/etc/selinux/`（vendor_sepolicy.cil + vendor_hwservice_contexts + device 编译的 precompiled_sepolicy），设备上留 `.bak3`–`.bak5`/`.bakvid` 备份。**源码已更新，但当前刷的 vendor.img 早于它们**——未来从当前树刷 vendor/vendor_b 才能使修复 image 级永久。每次策略改动需重启；init 每 boot 日志 "Compiling SELinux policy"（product sha256 与 vendor 内嵌副本不匹配，是支持路径，非错误）。

### 5.12 Lawnicons / RisingWalls（两个「空白 app」报告，一不可修一真 bug）

- **Lawnicons 不可从此树修**：`/system/product/app/Lawnicons/Lawnicons.apk` 是 prebuilt，渲染需 **35–40 s**（worker 线程阻塞在 `Resources.getIdentifier` 遍历整套图标，invalid ART image，ProfileInstaller skipped）。`cmd package compile -m speed -f` 无效，树里无源码。**「慢但可用」即答案，别再调查。**
- **RisingWalls 两层叠加失败（均已在源码 addressed）**：①`raw.githubusercontent.com` 此网络不可达，原代码 IOException 抛 RuntimeException 杀解析线程 → 主页永久空。②即使用镜像，Glide 4.16 硬编码 2.5 s connect/read 超时 + Parser 启动预载**全部 116 张扁平壁纸**饱和 Glide 4 个 source 线程 → 4 张主页图总 `SocketException(Socket is closed)`。**重写**（APK md5 `da842b06…`）：去掉 mass preload，每张图走 `NetworkUtils` 管线（12 s connect / 25 s read，三镜像 failover 带记忆赢家，LruCache + 磁盘缓存，lazy per-view 加载带 tag rebind 检查），只预取 4 张主页图。部署无需 TWRP：`adb root`+`adb remount`，push 到 `/system/system_ext/priv-app/RisingWalls/`，`am force-stop` + 重启（package `org.risingos.wallpaperpicker`，无磁盘 manifest 缓存）。

### 5.13 内核特性回移系列

见 §7。statx 验证后逐个叠加 process_madvise/clone3/CAKE/BBR/zram-zstd/FQ/BPF_JIT，KSM 试过回滚。

### 5.14 全功能 sweep 16/16 + 性能基线

- **功能 sweep 16/16 PASS**（kernel #18，enforcing；报告 `functest/FINDINGS.md`）：后摄拍照、前摄、录像、显示/亮度、触控、传感器、蜂窝/SIM/数据、VoLTE/IMS、WiFi、蓝牙、GNSS、音频、USB/adb、充电/电池、回移 syscall、内存/zram/网络默认。
- **性能基线**：MemTotal 3.76 GB，MemAvailable ~1.41 GB（37%）；zram zstd 实测 **5.16×**（orig 103.9 MB→compr 20.1 MB）；CPU ~15% 总，**0% iowait**，8 核 ~80% idle，governor schedutil；thermal ~33–35°C（无 throttling）；电池 100%，health 93%。无 panic/Oops/BUG，无 avc denied，无 OOM，crash buffer 空。

### 5.15 良性噪声清单（不要当回归）

- `system_server: Unable to access /sys/kernel/dmabuf/buffers` + "DMA-BUF heaps not supported" 每几秒 = 4.9 缺 `CONFIG_DMABUF_HEAPS`（5.6+），A17 memstats  polling 读不到，纯 cosmetic。
- 相机 `CAM_ERR cam_vfe_bus_error_irq Bus Err IRQ (0x4000)` + `Override width 648/656` @ provider init = 已知 focus-stats 残留；相机 2 设备 + 拍照稳定。
- GMS `binder transaction failed -22/-3`、`binder_alloc ENOSPC`（4 MB tx）@ gms.persistent 启动 = churn，非内核 fault。
- `DGFRR … TimeoutException 60000ms`（DroidGuard）= 此中国网络 Google 服务器可达性，非设备 fault。
- boot 期 i2c_geni -107 / gpio_to_desc WARNING / sde virt-plane / ipa-wan bad-response / Nl80211 band 8/16 = 平台固有。
- `pm-service`（vendor.per_mgr）SIGABRT `RefBase used with stack pointer argument` = benign（同 PixelOS）。
- **早期查询时序陷阱**：刚 boot `ping 1.1.1.1` 可能 "Network is unreachable"（WiFi 未关联）、相机 count 读 0（provider 启动）——~20–30 s 后重查再判回归。
- **WLED OVP 中断风暴**（历史一次性）：dmesg 31670 条 `WLED OVP fault detected`，集中 ~196 次/秒后**自停**；受控灭屏+唤醒未复发，显示健康。背光 boost 过压保护瞬态，自限，仅监控。
- **regulator WARN**（非致命）：`WARNING … drivers/regulator/core.c:2283 _regulator_disable`，comm=cammw:sensor；链 `panel power off → tcs3490 REGULATOR_EVENT_DISABLE → unbalanced disables for pm8998_l22 → WARN`，即相机传感器电源轨下电时被重复 disable（vendor 相机电源序列缺陷），触发=相机开着时灭屏/流 teardown。仅 WARN，系统继续正常。

---

## 6. 跨 ROM 共性问题类与修复模式

三个 ROM 反复撞同一几类墙。归纳成可复用的诊断/修复模式：

### 6.1 内核特性缺失 → 回移内核，不改 bionic

| 特性 | 症状 | 修复 |
|---|---|---|
| `MADV_WIPEONFORK`（4.14） | init exit 127/0x7f00，root mount 后 ~7 ms 死，零日志 | 回移 6 文件进 4.9；chroot `linker64 --help` 5 s 复现 |
| `F_SEAL_FUTURE_WRITE`（5.1） | >12KB bitmap Parcel 写 EINVAL，B 站视频闪退 | 写回退 `F_SEAL_WRITE` + 读改 `MAP_PRIVATE`（两侧，Bitmap.cpp） |

**模式**：A17 userspace 调新内核 API 失败时，回移该 API 进 4.9，而不是改 bionic/framework 绕过（除非像 memfd seal 那样框架侧本就有兼容路径）。

### 6.2 SELinux `/dev/ion` 拒绝家族（单例缓存陷阱）

sdm845 的 in-process gralloc mapper 通过 `/dev/ion` 分配缓冲；任何 system/vendor 域触及它都需显式 allow。**enforcing 下首次 open 被拒 → `IonAlloc::Init()` 单例永久缓存 `fd=-1` → 后续 ioctl 全 EBADF**。

| 域 | 类型命名空间 | 规则位置 | 症状 |
|---|---|---|---|
| `bootanim` | product/private | `sepolicy/private/legacy_ion.te` | 开机动画白屏（EGL_BAD_SURFACE 循环） |
| `mediacodec` | **vendor** | `sepolicy/vendor/legacy_ion.te` | 录像无关键帧/无 mp4 |
| `mediaserver` | public | `sepolicy/private/legacy_ion.te` | 录像（同上链） |
| `hal_audio_default` | **vendor** | `sepolicy/vendor/hal_audio_default.te` | 通话对方无声 + 下行杂音（ACDB 失败） |
| `vendor_hal_keymaster_qti` / `gatekeeper_qti` | vendor | （bring-up 期） | keystore2 不注册 → vold 阻塞 → init 不到 zygote |

- **类型命名空间决定规则放哪**：`mediacodec` 声明在 `system/sepolicy/vendor/mediacodec.te`（vendor 类型）→ 规则必须在 `BOARD_VENDOR_SEPOLICY_DIRS`（vendor policy）；放 product 失败（product 只见 public 类型）。`mediaserver` 是 public → product-private 可以。
- **A17 宏**：`r_file_perms` 已含 `{ getattr open read ioctl lock map watch watch_reads }`，`rw_file_perms` 再加 write/append；`{ rw_file_perms ioctl }` 是 belt-and-braces。vendor CIL 显示版本化名（`ion_device_202604`）= 正常。
- **录像 EBADF 链**：open 被拒 → Init 缓存 fd=-1 → `ioctl(-1, ION_IOC_IMPORT)` EBADF → `Failed to import ion buffer` → importBuffer 失败 → 无视频关键帧 → 无 mp4。**非 fd 传输/HIDL 缺陷，内核 ion 无缺陷**。源码链：QtiMapper::importBuffer → BufferManager::RetainBuffer → ImportHandleLocked → IonAlloc::ImportBuffer（gr_ion_alloc.cpp:157）；Init 在 gr_ion_alloc.cpp:48，失败不重试。
- **通话音频 ACDB 链**：`hal_audio_default` ion 拒绝 → `ACDB-LOADER: Cannot open /dev/ion errno:-1` → `Cannot allocate memory!` → `msm8974_platform: ACDB initialization failed` → 通话时 `Not correctly initialized!` → `voice_get_cal: No cal data for cal 7/4/2/3` → `voice_send_cvs/cvp_register_*_cal_cmd` 失败 → `voice_send_vol_step_cmd: ADSP_EFAILED`（上行无声）+ `platform_check_codec_backend_cfg: Use default bw and sr`（下行回退默认带宽/采样率 → 杂音）。ACDB 数据文件齐全（`/vendor/etc/acdbdata/Sony_tama_*.acdb`），非缺文件，是 loader 需 /dev/ion 被拒。**一行修复**：`allow hal_audio_default ion_device:chr_file rw_file_perms;`。
- **验证铁律**：permissive 对照必须 `setenforce 0` + **重启属主进程**（`kill $(pidof media.codec)` / `ctl.restart vendor.audio-hal`）绕过单例缓存，否则假阴性。
- **port 约束**：任何 system 域通过 sdm845 in-process gralloc mapper 触及 `/dev/ion` 都需显式 allow。**白面板 + EGL_BAD_SURFACE / 录像无 mp4 / 通话无声杂音 → 立即查该域的 ion 拒绝。**

### 6.3 vendor blob 与 A17 平台库尺寸/布局失配

port 把 **A10 vendor blobs** 与 **A17 平台库**混用；blob 自己 alloc 的类若在 release 间长大，就是 latent heap overflow。

- **GraphicBuffer sizeof（vendor47）**：A10 blob `libsomc_chokoballpal.so` 用 `movs r0,#0xa0` new 一个 `android::GraphicBuffer`；A17 的 GraphicBuffer 因 `DependencyMonitor` 内联在 offset 0x98 长到 0x6f8 → 每 CKB buffer ~1.3 KB heap overflow（`Ckb apply fail`、Scudo misaligned-dealloc、`ReqCtrl::stopSnapshotForHalCtrl` SIGSEGV、每拍 provider 重启）。**修复**：`frameworks/native/libs/ui/include/ui/GraphicBuffer.h` 把 `mDependencyMonitor` 改 `std::unique_ptr`（lazy make_unique）→ `sizeof==0xA0`；只替换 vendor `lib/libui.so` + `lib64/libui.so`。
- **focus-stats 容器尺寸（vendor50）**：后摄拍照崩在 `excal:FocusStat` `makePdafHwStats`→`PAL_MemSet`，SIGSEGV SEGV_ACCERR 恰在 buffer base+0x10800。stock converter（`libexcal_focus_stats_conv_plugin.so`）布局并 memset **0x11218** 字节（PDAF HW +0x297c，CAF HW +0x10BC4），但容器 ImageBuf 尺寸来自 `/vendor/lib/libexcal_core.so` `.rodata` @0x10230 的 StatsFormat 表 `[0x800,0xa000,0x800,0x10800]`（format 7/8/9/10）。**修复**：把 format-10 表项 @file offset 0x1023c 从 `00 08 01 00`（0x10800）改 `18 12 01 00`（0x11218），用 stock converter 不打补丁。**clamp/skip converter 写不行**——消费者（IdtCtrl 读 +0x10BC4，LensCtrl 校验 AF stats）需要完整 0x11218 布局。
- **RefBase 栈检查放松**：A17 libutils `check_not_on_stack` 让 `pm-service`（vendor.per_mgr）SIGABRT 循环（`RefBase used with stack pointer argument`），也影响 SIM/modem 注册；放松 vendor lib64/libutils.so 的栈检查。pm-service abort 残留为 benign。
- **模式**：vendor blob 在精确 `base + N` 崩 SEGV_ACCERR → 找**分配尺寸源**（这里是 StatsFormat→size 表），**扩大分配**而不是 clamp writer。任何 blob 自己 alloc 的、在 release 间长大的 A17/libui 类都有此 latent overflow。
- 工具：`llvm-objdump --triple=thumbv7-linux-gnueabi`（`prebuilts/clang/.../llvm-binutils-stable`），`tapseq.sh`，`prep-vendor-patch50.sh`（mount+replace+fsck）。

### 6.4 GMS：EROFS APEX 不可挂 → priv-app APK + 版本一致

4.9 无 `CONFIG_EROFS_FS` → GMS apex payload 不可挂。统一解法：解 apex → WSL mount EROFS → 取 base APK + XML → 装为 product priv-app。**模块版本必须与 base 容器匹配**（否则 Chimera 丢弃 → ANR 循环）。先查 `/proc/filesystems` 有无 erofs。

### 6.5 Sony ABL 启动模式 / vbmeta / slot 状态

- **boot 模式来自 warm-reset reason，不是 BCB**：`warmboot=0x77665502`→recovery（无 skip_initramfs）；`0x77665501`→normal。`androidboot.force_normal_boot=1` 钉死正常路径。
- **"Your device is corrupt" 在 `unlocked:yes secure:no` 下是 ABL OS-failed-to-boot fallback，不是 AVB**。混合 vbmeta（PixelOS hash vs Rising 分区）才会真 AVB 报错；用 `avbtool info_image` 比对 boot Hash descriptor 的 Salt+Digest 证明镜像匹配其 vbmeta。
- **slot 状态导致 `fastboot boot` Load Error**：retry 耗尽后 `slot-unbootable:b:yes`/`slot-retry-count:b:0` → 每个 `fastboot boot` 都 Load Error；`fastboot --set-active=b` 清 unbootable + 重置 retry=7 即恢复。**先 `getvar all`（grep slot|unlocked|secure）再怪镜像。**

### 6.6 前摄 EGL 间歇崩溃（CKB GPU 初始化时序）

- 历史 SIGSEGV：`excal:ImageConv → raisin → chokoballcmn EGLImage::createInstance → eglCreateImageKHR → eglSubDriverAndroid+0x5bec` 解引用 NULL adreno dispatch（fault 0x44，r6=0），前置 `DeviceType=17 is not ready / Cannot open ckb`（CKB GPU 后端初始化失败）。
- **结论（2026-11-07 15:27）**：sizeof(GraphicBuffer) 修复已确认部署（设备 libui md5 == out 构建）；干净启动 + 策略部署后的稳定状态下，激进复现（前摄首开/快速来回切换/连拍/录像）**零崩溃**，provider pid 全程稳定。历史 EGL SIGSEGV 为**间歇性**（与 CKB GPU 初始化时序/状态相关），不再触发，**无需额外 vendor blob 补丁**。残留非致命：ObjectDetector `Cannot open ckb`（人脸检测回退）、hal_camera_default sysfs graphics/battery 拒绝（曝光/闪光调节）。

---

## 7. 内核 4.9 特性回移系列（逐个验证）

把现代 syscall/特性回移进 `/root/android/rising/kernel/sony/sdm845`，让 A17 在 4.9 上跑得好。计划文档 `KERNEL-4.9-feature-backports.md` / `KERNEL-4.9-perf-backports.md`。

**原则**：一次只回移一个，独立提交、独立验证（freestanding raw-syscall 测试二进制），当前镜像 boot + 测试通过再叠下一个；dd 新 boot.img 前备份当前 boot_b；内核编辑保持 uncommitted 除非用户要求。

### 7.1 已完成并设备验证

| 特性 | 内核# | 验证 |
|---|---|---|
| `close_range`（436） | — | raw-syscall |
| `statx`（291 native / 397 compat） | — | boot ~82 s，enforcing，GMS/VoLTE/相机/touch 回归干净 |
| `process_madvise`（440）+ `MADV_COLD`=20/`PAGEOUT`=21 | #9 | pmtest64 ×10 stress + pmtest32 |
| `clone3`（435） | #10 | c3test64 |
| `SLAB_FREELIST_RANDOM` | #12 | — |
| **CAKE** qdisc（4.19→4.9，built-in） | #15 | caketest64 attach→ping→dump（kind=cake）→detach + 全回归 |
| **BBR 默认** | #16 | runtime `tcp_congestion_control=bbr` |
| **zram→zstd** | — | `comp_algorithm=[zstd]`，压缩比 ~5.07×（vs lz4 ~2.5–3×） |
| **FQ 默认 qdisc** | #17 | `default_qdisc=fq`，配 BBR pacing |
| **BPF_JIT** | #18 | `bpf_jit_enable=1`，netd bpf maps 加载 |

- defconfig 还启用 `USERFAULTFD`、`CRYPTO_ZSTD`、`TCP_CONG_BBR`、`PROCESS_RECLAIM`。
- **当前主力内核 #18**（boot md5 `4a699133f798abdf8102da4bac064d7b`）。
- zram 算法由 `device/sony/tama-common/rootdir/etc/init.qcom.rc:66`（落 `/vendor/etc/init/hw/init.qcom.rc`）设，**不是 fs_mgr**；可 `adb remount`+push 运行时部署，无需 50 min vendor 重编。
- BBR 默认改 Kconfig **choice** 符号 `CONFIG_DEFAULT_BBR=y`，**不是**派生字符串 `CONFIG_DEFAULT_TCP_CONG`。

### 7.2 试过并回滚 / 已具备 / 推迟

- **KSM（#19，回滚）**：`CONFIG_KSM=y` + init.rc 启用，ksmd 0.0% CPU 全回归通过，**但零收益**（pages_sharing/unshared/full_scans 全 0）。精确扫每个 `/proc/*/smaps` VmFlags：**0 进程标 `MADV_MERGEABLE`**——现代 A17 弃 KSM（KitKat 时代）改用 zram，userspace 不标。要生效需侵入式 userspace madvise 标记 daemon（收益不确定、有风险）→ 不值。回滚 defconfig+init.rc，刷回 #18。
- **RACK-TLP**：已具备且默认启用，无需回移。
- **推迟/拒绝**：UFFD-WP（4.9 arm64 无 PTE_UFFD_WP，高风险，待用户决定）；ZRAM writeback（无专用 backing 分区，用 userdata 会损坏）；THP（4 GB Android 碎片/compaction stall）；ZSWAP（冗余，swap 本就是 zram）；BFQ/KYBER IO sched（快 UFS 上 marginal，BFQ 需回移）；LZ4HC（zstd 已更好）。

### 7.3 不可行 / 硬件门

- **不可行（需 VM/sched/arch 重写）**：folio（5.16）/MGLRU（6.1）/DAMON（5.15）/maple-tree（6.1）/EEVDF（6.6）/sched_ext（6.12）/io_uring（5.1）/PREEMPT_RT（6.12）。
- **硬件门**：arm64 MTE/PAC/BTI 需 ARMv8.4/8.5，SDM845 是 ARMv8.2-A，物理不支持。
- 高价值 clean 回移基本耗尽；剩余小/中候选：erofs（此树无 fs/erofs）、kTLS、pidfd_open（5.3）/cachestat（6.5）/mseal（6.10）/futex2（6.6）、dmabuf-heaps（仅为消 A17 日志 spam）、uclamp（5.3，但此内核用 WALT+schedtune，uclamp 会冲突）。

### 7.4 树特定约束（回移时必踩）

- **4.9 的 `import_iovec` 与 `compat_import_iovec` 分开**（无自动 compat 检测）→ 任何取 iovec 的新 syscall 需显式 `COMPAT_SYSCALL_DEFINE` + `include/linux/compat.h` 声明 + arm64 `unistd32.h` 映射到 `compat_sys_*`（并 bump `__NR_compat_syscalls`）。
- `CONFIG_TRANSPARENT_HUGEPAGE` off；Sony `try_to_unmap` 取 3 参；`reclaim_pages_from_list` 是 Sony 特有（非上游 `reclaim_pages`）。
- `test_and_clear_page_young`/`page_is_file_cache` 在 mm/madvise.c 默认不可见 → 需 `<linux/page_idle.h>` + `<linux/mm_inline.h>`。
- **无 `tcf_block` 基础设施、无 `netlink_ext_ack`** → qdisc 回移用 `Qdisc_class_ops` `.get`/`.tcf_chain`（非 `.find`/`.tcf_block`）、`tcf_destroy_chain`、2 参 init/change、4 参 `nla_parse_nested`；`ktime_t` 仍是 union（`{ .tv64 = ... }`）；`sk_buff` 用旧 `struct nf_conntrack *nfct`；无 `kvcalloc`/`kvzalloc`（用 `vzalloc`+`kvfree`）。CAKE 还需在 nf_conntrack_core.c 直接实现 `nf_ct_get_tuple_skb`。
- **4.9 `register_qdisc()` 拒绝缺 `get && put && walk && leaf` 的 classful qdisc**（initcall 返 -EINVAL，kind 静默缺失，后续 RTM_NEWQDISC 给 -ENOENT）。4.19 去了 `.put` 要求 → 上游 CAKE 无 `.put`，回移须加 no-op `.put`。诊断 tell：已知 built-in kind（pfifo）能 attach 而新 kind -ENOENT 尽管符号/initcall/RELA 都在 vmlinux。
- **无 Kconfig default-qdisc choice**（`default_qdisc_ops` 硬编码 `&pfifo_fast_ops`）→ 运行时 init.rc `write /proc/sys/net/core/default_qdisc fq`（需 `CONFIG_NET_SCH_FQ=y`）。multiqueue netdev（wlan0）root 保持 `mq`，但每 tx-queue leaf 用 default_qdisc_ops，fq 仍给 BBR pacing。
- **BPF_JIT** 依赖 `HAVE_EBPF_JIT`（arm64 select）/`MODULES=y`/`!CFI`（均满足）；此 4.9 **无 `BPF_JIT_ALWAYS_ON` Kconfig** → `bpf_jit_enable` 默认 0，需 `CONFIG_BPF_JIT=y` + init.rc `write … bpf_jit_enable 1`。benign boot 噪声：`bpf_jit_kallsyms` sysctl 缺失、单条 `bpf_jit: *** NOT YET: opcode c3 ***`（4.9 arm64 JIT 缺新 opcode → 该程序回退解释器）。
- **raw-syscall 测试二进制**（AOSP clang `-nostdlib -static`）：加 `-fno-builtin`（否则 clang 把 static memcpy/memset 改写成 extern libc 调用 → undefined symbol）；naked `_start` 的 `bl cmain` 需 cmain 是 global（static callee 链接时 undefined）。

---

## 8. 构建 / 刷机 / 部署环境

### 8.1 WSL / 树 / lunch

- WSL distro **Ubuntu-24.04**（`wsl.exe -d Ubuntu` 失败 WSL_E_DISTRO_NOT_FOUND）；`wsl.exe -l -v` 输出 UTF-16LE，过 iconv。
- 树：`/root/android/rising`（`rising_xz2c-userdebug`）；PixelOS `/root/android/pixelos` 已删。
- 磁盘：1007 GB ext4 @ `/dev/sdd`；vhdx 保留已删块，回收需 `wsl --shutdown` + compact，留 host headroom。WSL swap 是真实 host 文件（Temp 下 swap.vhdx），构建时别清 Temp。

### 8.2 镜像（sparse vs raw，离线验证）

- `out/.../system.img` 是 **sparse**；刷前 `simg2img` 转 raw（raw size == 分区 size）。`dumpe2fs -h` 对 sparse 输出**空** = 「不是 raw」不是「损坏」。
- 离线验证：`debugfs -R "cat /system/build.prop" x.raw`、`debugfs -R "dump <path> /tmp/f" x.raw && sha256sum f`、`dumpe2fs -h | grep "Block count|Free blocks"`。验证**打包进镜像**的内容。
- **`m systemimage` 可能 exit 0 却打包旧镜像**：product-makefile 改动后复用旧 `file_list.txt` + 旧中间镜像。修复：`rm -f out/.../obj/PACKAGING/system_intermediates/{file_list.txt,file_list.txt.rsp,system_image_info.txt,system.img}` + 删 staging 里被 drop 的包（out/ 从不移除 PRODUCT_PACKAGES drop 的文件）。
- **检查新鲜中间产物，不是 staging 副本**：`out/soong/.intermediates/system/sepolicy/<target>/android_common/xz2c/*.cil` 是重编落地处；`out/target/product/xz2c/{system,vendor}/etc/selinux/*.cil` 可能几天旧。

### 8.3 刷机循环（stock TWRP，唯一批准方式）

`adb reboot bootloader` → fastboot（无需物理动作）→ `fastboot boot twrp-3.7.1_12-0_LOCAL-20251206-01-apollo.img` → 等 adb state `recovery`（1–2 min）→ `adb push` raw 镜像到 `/data`（4.2 GB ~2–3 min）→ `dd if=/data/x.raw of=/dev/block/by-name/system_b bs=4194304 conv=fsync`（1008 records ~30 s，135 MB/s）→ `head -c <size> /dev/block/by-name/system_b | md5sum` 对比 host md5（Windows `certutil -hashfile` 比 WSL md5sum over /mnt/c 快）。写当前正在运行的分区是安全的。

- **绝不 `fastboot flash`**（用户硬约束）。boot_b=sda46，system_b=sda52。
- `fastboot boot` 临时 TWRP 只在 active slot 未标 unbootable 时工作（见 §6.5）。
- patched TWRP 的 root adb 仍需用户在 UI 点一下（MSC only 直到用户启用 ADB）；推一个 20–25 min 的「等 recovery 状态再 dump」后台脚本，然后请用户切换。

### 8.4 运行时部署（无需 TWRP / 无需重刷）

- **APK / 单文件**：`adb root` + `adb remount`（`Remounted / as RW`，AVB 关）→ push 到真实路径（system_ext 在 `/system/system_ext/...`）→ `chmod 644` + `restorecon -v`（须读 `u:object_r:system_file:s0`）→ `sync` → force-stop + 重启。替换 APK 时删旁边旧 `oat/`。
- **vendor SELinux 增量（无 Soong）**：只编辑**已存在**的 `.te`（新文件不在 ninja 图，需 ~50 min Soong）→ `ninja -f out/combined-rising_xz2c.ninja out/.../vendor/etc/selinux/vendor_sepolicy.cil out/.../vendor_hwservice_contexts`（~2 min）→ push + chown root:root + chmod 644 + restorecon -F（`vendor_configs_file`）。boot 前用 `test_policy.sh` 复刻 init 的 secilc 参数列表验证 EXIT=0，输出覆盖 precompiled_sepolicy。加 hwservice context 时连 vendor_hwservice_contexts 一起 push。
- **完整 sepolicy 7-file precompiled set**（`m selinux_policy` 后，无需刷机）：`adb root`+`adb remount`，push `product_sepolicy.cil` + `product_sepolicy_and_mapping.sha256` → `/product/etc/selinux/`；`vendor_sepolicy.cil` + `precompiled_sepolicy` + `precompiled_sepolicy.{plat,product,system_ext}_sepolicy_and_mapping.sha256` → `/vendor/etc/selinux/`。precompiled 仅当三个 sha256 副本都等于分区对应物时被采用——重启前 `cat` 验证；chmod 644 + restorecon -R 两目录 + 重启。先备份（`.bakvid`）。prep 脚本把产物 copy 到 `/mnt/c/...`（adb push 读不了 `\\wsl.localhost`）。
- **A17 sepolicy 目录路由**：`PRODUCT_PRIVATE_SEPOLICY_DIRS` → `/product/etc/selinux/product_sepolicy.cil`（**不是 plat**）；`BOARD_PLAT_PRIVATE_SEPOLICY_DIR` 在此树 obsolete（用 SYSTEM_EXT_PRIVATE_SEPOLICY_DIRS）；`plat_sepolicy.cil` 由 checkpolicy 从 plat_sepolicy.conf 生成，输入**无任何 device/ 文件** → device .te 永不出现在 plat（别从 plat grep  conclude「规则被忽略」）。`vendor_sepolicy.cil` 在 **vendor 分区** → `m systemimage` 从不刷新它，vendor-policy 编辑需 `m vendorimage` + 刷 vendor_b（或运行时 push）。

### 8.5 boot header cmdline 快补

v1 header cmdline 是 `0x40..0x240` 的固定 512 B 字段；读当前 NUL 终止串，append ` b" androidboot.xxx=1"`，zero-pad，写回，总长 ≤512 B。AVB 在 orange/verity-off 状态不强制。秒级测试 cmdline 串，避开改 BoardConfig 的 ~46 min 全量 Soong。重打包用 `unpack_bootimg.py --format mkbootimg` 打印的精确 arg 行（flag 是 `--pagesize`/`--kernel_offset` 等，**不是** `--page-size`/`--out`）；dtb 已在 Image.gz-dtb 内，无需 `--dtb`。

### 8.6 pstore / ramoops 取证

- 失败 boot 的 kernel/init 日志在 `/sys/fs/pstore/console-ramoops-0`（重 bit-flip，strip NUL，tolerant grep）；logcat 丢失。crash 在 `/data/tombstones/tombstone_XX`（TWRP 可读）。
- **warm reset**（hold power+vol up ~10 s → fastboot）保留 ramoops；recovery-as-boot 意味着读 pstore 要把 stock recovery 刷回 boot_b，读完刷回 ROM boot。
- **boot-retry loop 自毁证据**（每 retry 重置 1 MB ramoops）→ 要可读日志，只允许**一次** Rising 尝试，force-power-off（不新 kernel boot），再 boot TWRP 一次 dump。pstore 只能 mount 一次，直接从 `/sys/fs/pstore` dump。
- **无 boot 诊断 A17-bionic-vs-4.9**：recovery 里 mount 镜像到 `/mnt/sb`（绝不 TWRP 的 /system），`chroot /mnt/sb /system/bin/bootstrap/linker64 --help`——任何 A14+ bionic 在缺 4.14 flag 的内核上打印 MADV_WIPEONFORK EINVAL 并 exit 134，5 s 解释 init exit 127。

### 8.7 工具陷阱（忘记会浪费真实时间）

- **Git Bash 本地展开 `$VAR`/`$(...)`**（即使在传给 wsl.exe 的双引号里）；**单引号也被剥**（`grep -E 'A|B'` 的 `|` 变成 pipe）→ wsl 用脚本文件（Write 进 WSL，`wsl.exe -- bash /root/x.sh`）+ `grep -e A -e B`。**但 adb shell 设备侧循环用单引号**（`adb shell 'for z in …; do echo "$(cat $z/type)"; done'`）让设备 shell 展开。
- **`MSYS_NO_PATHCONV=1` + `MSYS2_ARG_CONV_EXCL='*'`** 用于 adb/fastboot/wsl 路径参数。adb.exe 不能 stat MSYS 路径（`/c/...`）→ 本地文件用 `'C:/Users/...'`；remote 路径需 MSYS_NO_PATHCONV 否则被重写。adb push 读不了 `\\wsl.localhost` → 先 copy 到 `/mnt/c`。
- **adb/fastboot 从 Windows Git Bash 跑**（platform-tools 在 PATH，`C:\Env\platform-tools`），**不在 WSL**（`wsl -- adb` command not found）。`adb kill-server/start-server` 救不了 PnP 层缺失的手机（无 VID_0FCE），只有物理重连/开机。
- **`adb devices | grep -qE 'recovery|device'` 永真**（header "List of devices attached" 含 "device"）→ 匹配 tab 分隔状态 `grep -qE '[[:space:]](recovery|device|sideload)[[:space:];]'`；recovery 里用 `adb wait-for-recovery`（**不是** wait-for-device，后者在 TWRP 永挂）。
- **绝不把 shell 变量命名 `OUT`**（lunch 设 OUT 为 out/target/product/<device>，`>> "$OUT"` 写到目录）。用 LOGF。
- **`nohup … &` 在一次性 wsl.exe 调用里不可靠**（distro 随 client  teardown）→ 用 setsid/launch 脚本并验证 log 存在，或用 Bash 工具后台任务（存活）。
- **前台 Bash 调用超 timeout 不像你想的那样被杀**：wrapper 报 success 但 WSL soong_ui 继续跑完；follow-up `m` abort 在 out/.lock。诊断 `pgrep -af 'soong_ui|soong_build|siso'` + `fuser -v out/.lock`； orphan 还在进展就让它完成查产物 mtime/md5。长构建从一开始就用后台任务 + `/root/*.log`。
- **别用 grep "error" 判内核构建**：warning 引用源行如 `return A_ERROR;` → `grep -ic error` 在 green build 也命中。查 `grep -c 'error:'` + boot.img mtime/md5 + System.map 新符号。
- **`get_build_var PRODUCT_PACKAGES` 打印单行空格分隔** → `tr ' ' '\n'` 再测成员。
- **screencap boot 期是陷阱**：display Dozing/off 时返回 14,050 B 纯白 PNG；~1 MB 才是真内容。先 wake。手动 launch 的 bootanimation layer 在 lockscreen 之下，截图看不到它渲染——用 logcat（EGL_BAD_SURFACE）+ avc 作证据。
- **后台任务输出 temp 文件很快被清**：把需要的事实（md5、readback hash、state）随任务运行捕获进对话或 /root log。
- **手机可能消失数小时**：watcher 90+ min 见不到任何东西而 offline host 工作继续；别 conclude「线/adb 问题」，保持后台 watcher + 中文报告，只有物理重连带回 VID_0FCE。`Get-PnpDevice | ? InstanceId -match 'VID_0FCE'` 全 Status Unknown = 手机关/拔；PID：51F9=MTP+ADB，0DDE=Sony fastboot，B00B=SOMC flash。
- **watcher 卫生**：绝不并发跑两个 boot watcher，绝不跨重叠测试归因 PnP 事件。`Get-PnpDevice` 每次 ~10 s，t=N s 标签漂移；从时间戳算 elapsed。

---

## 9. 发布 / 推送工作流

### 9.1 仓库（GitHub 账号 `xing29391`）

| 仓库 | 可见性 | 内容 |
|---|---|---|
| `los23` | **private** | 22 个 LineageOS 侧适配文件 |
| `sony_xz2c_tree` | public | device/sony/{xz2c,tama-common} 文本树 + vendor/sony mk/bp（476 文件，无 blobs） |
| `pixelos_17` | public | 38 个平台/Pixel 修复（GraphicBuffer sizeof、FCM 5/6 + light/touch HAL、Connectivity bpf、RefBase、libmeminfo、vendor/pixel configs）+ README 记录 init 危险 |
| `rising_9` | public（master） | README/artifacts/device/docs/patches/scripts；镜像 device/sony/{tama-common,xz2c} |

所有从 WSL staging 目录推送，单 "Import adapted sources" commit，只含修改/新增文件，路径镜像树结构。

### 9.2 中国网络推送

- `git push` 到 github.com 经常死于 `GnuTLS recv error (-110)` 或 130 s connect timeout，第 3–5 次成功。
- gh-proxy `insteadOf` 重写**只对 fetch 有效**；push 必须 `GIT_CONFIG_GLOBAL=/dev/null` + inline `https://x-access-token:<PAT>@github.com/...` URL（**仅作参数，不持久化**）。
- **PAT 安全约束**：PAT 是 **push-only**；**绝不写进 git config/remotes 或 memory 文件**；用 inline token URL + `GIT_CONFIG_GLOBAL=/dev/null`。旧 PAT 已过期（Bad credentials），新 PAT 仅 inline 使用、验证未持久化到任何 config。本文档不含 PAT 值。
- 镜像（fetch 用）：`raw.githubusercontent.com` 被墙；`gh-proxy.com`/`ghfast.top`/`ghproxy.net` 可用但 50–100 KB/s。

### 9.3 本次 rising_9 推送

- clone `/root/rising_9_push`（--depth 1）；rsync -a --delete 本地 device/sony/{tama-common,xz2c}。
- commit `9ee784f` "xz2c: grant ion access for video encode and voice-call ACDB; enable zstd/fq/bpf_jit"（4 文件 +21/−1）：`rootdir/etc/init.qcom.rc`（zram lz4→zstd、+default_qdisc fq、+bpf_jit_enable 1）、`sepolicy/private/legacy_ion.te`（+mediaserver ion）、`sepolicy/vendor/hal_audio_default.te`（+hal_audio ion）、`sepolicy/vendor/legacy_ion.te`（+mediacodec ion）。
- push `0f89e89..9ee784f master→master`；remote HEAD 验证 `9ee784fe822a8fa0841433ecf6d9500dfc97b9d2`；clone .git/config remote = 纯 `https://github.com/xing29391/rising_9.git`（无 PAT）。
- **vendor/sony 改动未推**（libexcal_core.so、Android.bp、tama-common-vendor.mk）：rising_9 不 track vendor，vendor origin=upstream sony-sdm845 无 push 权。

---

## 10. 当前状态与遗留问题

### 10.1 当前状态（2026-11-07）

- **RisingOS 9.0，enforcing 默认，内核 #18**（4.9.337-perf+），全功能 sweep **16/16 PASS**。
- 性能默认 `bbr / fq / bpf_jit=1 / zram=zstd`（~5× 压缩）；MemAvailable ~1.4 GB，0% iowait，无 throttling。
- 源码已发布到 xing29391 四仓库；rising_9 最新 commit 9ee784f（ion for video+ACDB、zstd/fq/bpf_jit）。
- 录像、通话音频修复均 enforcing on-device 验证通过（录像 mp4 vide 310 帧；通话 `send_voice_cal acdb_rx=7 acdb_tx=6` 无 "Not correctly initialized"，零 hal_audio ion avc，crash buffer 0）。

### 10.2 遗留 / 待确认

1. **通话上行/下行听感需用户物理复听确认**（信令/校准链已验证正常，但真实听感需人耳）。
2. **GNSS 室外 fix 未测**（HAL/modem 链路已初始化，室内无 active request 无 fix）。
3. **vendor.img 需从当前树重刷**使运行时 push 的 SELinux 修复（touch/gnss/bt/bootanim/ion）image 级永久；当前刷的 vendor.img 早于它们。
4. **vendor/sony 改动未推 GitHub**（rising_9 不 track vendor）——如需保存需另议仓库。
5. **benign 残留**（不修，见 §5.15）：pm-service RefBase abort、alarm@1.0-service 每 boot abort、次要 hal_camera_default sysfs graphics/battery avc、WLED OVP 历史自限风暴、regulator pm8998_l22 下电 WARN、ObjectDetector Cannot open ckb 回退。
6. **内核回移候选**（按需）：RACK-TLP 已具备；下一步 Kyber IO sched（marginal）；erofs/kTLS/pidfd/cachestat/mseal 仍 open；UFFD-WP 待用户决定。

### 10.3 一句话总结

> 在 4.9.337 + A10 blobs 的 Sony XZ2 Compact 上跑 Android 17，本质是**逐处弥合「A17 userspace 假设」与「4.9 内核/旧 blobs 现实」的落差**：内核缺的特性回移（MADV_WIPEONFORK 是启动前提），SELinux 缺的 ion 授权按域补（注意单例缓存与类型命名空间），blob 失配的尺寸/布局扩大分配而非 clamp writer，GMS 的 EROFS 不可挂就重打包 priv-app 并保证版本一致——全程证据优先、hash 校验、stock TWRP dd 刷机、永不动 init 二进制。

---

*本文档由本次适配会话整理，逐字日志证据见 `functest/FINDINGS.md`；内核回移细节见 `KERNEL-4.9-feature-backports.md` 与 `KERNEL-4.9-perf-backports.md`。*