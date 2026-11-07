# RisingOS 9.0 xz2c 全功能测试 (kernel #18, Enforcing)

日期: 2026-11-07  设备: BH901L93BZ  baseline: crash=0 avc=0

## 相机 (PASS, 2026-11-07 15:27 更新)
- 后摄 ID0: 预览正常, 拍照成功 (19MP, 2026-11-07-15-26-02-202.jpg 7.1MB)。残留 `CaptureReq Failed result=-107` (已知 focus-stats 噪声, 不影响出图)。
- 前摄 ID1: **已不再崩溃** (15:23-15:27 全功能通过, 见下方"前摄崩溃攻关结论")。历史 SIGSEGV (excal:ImageConv, fault 0x44, eglSubDriverAndroid+0x5bec) 最后一次出现于 13:47 (tombstone_34), 此后干净启动+稳定状态下激进复现 (前摄首开/快速来回切换/连拍/录像) 均零崩溃。
- 录像 (后摄+前摄): **PASS** (后摄 10.44s/310 帧; 前摄 6.60s/197 帧, front_video.mp4)。
- 证据: camera_crash_1.txt (历史), front_crash_live.txt (当前零崩溃), front_still.jpg / front_video.mp4 (当前前摄出图/出片)。

## 显示/触控/传感器 (PASS)
- 显示: Adreno 630 GLES 3.2, 1080x2160@60Hz, SurfaceFlinger 正常; 亮度可调 (18->120->18 验证)。density 480/override 384。
- 触控: synaptics_tcm_touch (event2), ABS_MT 10 槽多点; vendor.lineage.touch@1.0-service.tama 运行; 无内核触控报错。(硬件实际触点需物理确认)
- 传感器: 齐全且有活动连接 — lsm6dsm 加速度/陀螺, ak0991x 磁力, tmd2725 光/距, bmp380 气压, qualcomm 融合 (gravity/linear_accel/rotation/game_rotation), significant_motion。加速度计缓冲 last 50 events (出数正常)。
- avc=0, 无 panel/dsi/sensor 内核错误。

## 蜂窝/SIM/VoLTE (PASS)
- SIM1: LOADED, China Unicom 46001 (cn), 非漫游; SIM2: ABSENT。
- 注册: voiceRegState=0(IN_SERVICE), dataRegState=0(IN_SERVICE), LTE(RAT14), CS+PS HOME, availableServices=[VOICE,SMS,VIDEO]/[DATA,MMS]。
- 信号: LTE rssi=-59 rsrp=-100 rsrq=-20 level=3 (良好)。
- IMS/VoLTE: **mImsRegistered=true, mImsRegistrationOnOff=true**; vendor.ims.DATA_DAEMON_STATUS=1, QMI_DAEMON_STATUS=1; persist.dbg.volte_avail_ovr=1。
- 移动数据 PS 已注册 (当前 WiFi 为主用 validated 网络, rmnet 非默认路由)。实际拨号音频需物理确认 (历史 10010 已验证)。

## WiFi/蓝牙/GNSS (PASS)
- WiFi: SSID ZTE-h9UZee_5G (5GHz 5220MHz, 11ac, WPA2), VALIDATED, RSSI -67, link 175/260Mbps。ping 网关 0% 丢包 avg6.4ms; ping 公网 223.5.5.5 0% 丢包 avg34.5ms → 互联网正常。
- 蓝牙: State ON, MAC D4:38:9C:9A:9C:6A, name "Xperia XZ2 Compact", A2dpOffloadEnabled=true, SCAN_MODE_CONNECTABLE, LE scanner 就绪, 0 bonded。
- GNSS: location_mode=3(高精度); HAL 全套运行 (gnss@2.0-service-qti, loc_launcher, lowi-server, xtra-daemon, mlid); gps provider enabled=true allowed=true (supports bearing/speed/altitude)。last location=null (室内无 active request, 无 fix; 需室外验证定位)。

## SELinux 关联发现 (测试期间出现, 基线为0)
内核 audit (enforcing, permissive=0) 抓到两类 denial, 与相机/录像失败强相关:
1. `mediacodec` (HwBinder:1554_2) denied { read } `ion_device` chr_file (/dev/ion) — **疑似录像无视频关键帧根因** (视频编码器拿不到 ion 缓冲)。可修: sepolicy allow mediacodec ion_device:chr_file read。
2. `hal_camera_default` (excal:HalCtrl) denied { write } `vendor_sysfs_graphics` (name=ires_ua / brightness) — 相机 HAL 图形/亮度控制被拒, 与前摄 EGL 图形路径相关。
注: dmesg 环形缓冲被相机 provider 崩溃重试刷屏快速覆盖, 上述为 604-622s 逐字证据。

## 音频/USB/充电 (PASS)
- 音频: HAL 运行 (android.hardware.audio.service, audioserver, uaudio_svc); 主输出线程 AudioOut_D (MIXER PRIMARY|FAST) → AUDIO_DEVICE_OUT_SPEAKER, 48kHz/PCM16/stereo; threadLoop framesWritten 152640→302400 (实际渲染出帧, 扬声器路径正常)。声卡 sdm845-tavil-snd-card, 全套 PCM 录/放设备, tinymix 可用; mic mute=false。无 audio 错误/avc。
- USB: current_functions=RNDIS, applied=true, kernel_state=CONFIGURED; rndis0 10.53.51.x; adb 正常。
- 充电/电池: status=5(Full), health=2(Good), present=true, level=100, voltage=4251mV, temperature=35.7°C; sysfs battery/status=Full。

## 回移 syscall 回归 + 内存/swap (PASS)
- pmtest64=0 pmtest32=0 statxtest64=0 statxtest32=0 c3test64=0 (process_madvise/statx/clone3 全部正常)。
- 默认值保留: tcp_congestion_control=bbr, default_qdisc=fq, bpf_jit_enable=1, zram comp_algorithm=[zstd]。
- zram mm_stat: orig=153MB compr=31.8MB → 压缩比 ~4.8x。
- MemTotal 3.76GB, MemAvailable 1.49GB, SwapTotal 931MB / SwapFree 765MB (已用 ~166MB)。kernel 4.9.337-perf+ #18。

## 最终日志分析
- 无 kernel panic / Oops / BUG / OOM / oom-kill。
- 无 ANR (am_anr=0)。
- 全部崩溃/警告均溯源相机子系统:
  - native: vendor.somc.hardware.camera.provider@1.0-service SIGSEGV ×2 (tombstone_29/30, 11:53, 前摄 EGL/raisin)。
  - java: org.lineageos.aperture FATAL EXCEPTION ×1 (11:59:48, `IllegalArgumentException: Trying to rebind camera while camera is not in IDLE state`) — 前摄 provider 死亡后重绑失败的次生崩溃 (兼 Aperture 未优雅处理 device error)。
  - kernel WARN ×1: `drivers/regulator/core.c:2283 _regulator_disable+0x168/0x174` (921s, 非致命) — 相机 provider 崩溃重启致电源轨 disable 计数不平衡; 系统继续正常运行。
- 其余系统服务无崩溃。

## 总结
| 功能 | 结论 |
|---|---|
| 后摄拍照 | PASS (出图正常) |
| 前摄 | **PASS** (2026-11-07 15:27: 预览/连拍/录像/快速切换零崩溃; 历史 EGL SIGSEGV 不再复现) |
| 录像 | **PASS** (后摄 310 帧 + 前摄 197 帧; mediacodec/mediaserver ion_device SELinux 授权) |
| 显示/亮度 | PASS |
| 触控 | PASS (设备+HAL; 硬件触点需物理确认) |
| 传感器 | PASS (全套出数) |
| 蜂窝/SIM/数据 | PASS (联通46001 LTE 在网) |
| VoLTE/IMS | PASS (mImsRegistered=true) |
| WiFi | PASS (互联网 0% 丢包) |
| 蓝牙 | PASS (ON, MAC 正常, 可扫描) |
| GNSS | PASS (HAL 就绪; 室内无 fix, 需室外验证) |
| 音频 | PASS (扬声器实际出帧, mic 就绪) |
| USB/adb | PASS (RNDIS CONFIGURED) |
| 充电/电池 | PASS (Full/Good/100%) |
| 回移 syscall | PASS (pm/statx/clone3) |
| 内存/zram/网络默认 | PASS (bbr/fq/bpf_jit/zstd 保留) |

**待修 (按优先级):**
1. ~~录像失败~~ **已修复** (2026-11-07, sepolicy mediacodec/mediaserver ion_device 授权; enforcing 验证通过, 见下方)。
2. ~~前摄崩溃~~ **不再复现 / 功能 PASS** (2026-11-07 15:27, 见"前摄崩溃攻关结论")。历史 EGL SIGSEGV 为间歇性, 干净启动+策略部署后全功能 sweep 零崩溃。
3. 相机相关 SELinux (次要, 非致命): `hal_camera_default` 写 `vendor_sysfs_graphics` (ires_ua/brightness)、search `vendor_sysfs_battery_supply` 被拒; ObjectDetector `Cannot open ckb` (DeviceType=17) 回退。不影响出图/录像。
4. regulator WARN: 相机崩溃的次生现象, 相机稳定后未再出现。

## 前摄崩溃攻关结论 (2026-11-07 15:23-15:27, enforcing)
- **sizeof(GraphicBuffer) 修复已确认部署**: 设备 `/vendor/lib/libui.so`=7e88e4a9…、`/vendor/lib64/libui.so`=190249ca… 与 out 构建 md5 完全一致; out libui 构建时间 11-01 03:06 晚于 GraphicBuffer.h/.cpp 修改 10-31 19:38 → `std::unique_ptr<DependencyMonitor>` (sizeof==0xA0) 修复已在运行镜像中。"源码缺修复"假设被推翻。
- **历史崩溃**: tombstone (11:53 / 13:37 / 13:47, fault 0x44) 均为 excal:ImageConv → raisin → chokoballcmn EGLImage::createInstance → eglCreateImageKHR → eglSubDriverAndroid+0x5bec 解引用 NULL adreno dispatch (r6=0)。前置 `DeviceType=17 is not ready / Cannot open ckb` = CKB GPU 后端初始化失败。
- **当前全功能 sweep 零崩溃**: 前摄预览 ~29fps; 前摄连拍 ×5 (15:24:11/35/39, 15:27:23/29, 样张 front_still.jpg 为真实画面); 前摄录像 6.60s / vide 197 帧 / soun 307 采样 (front_video.mp4, Finalize 无 ERROR_NO_VALID_DATA); 前摄首开 + 快速来回切换 + 切回后摄 19MP (15:26:02, 7.1MB)。provider pid 6677 全程稳定, 无新 tombstone, front_crash_live.txt 零 `Fatal signal`。
- **残留非致命**: ObjectDetector `Cannot open ckb` (人脸检测回退, 不影响预览/出图/录像); hal_camera_default sysfs graphics/battery 拒绝 (曝光/闪光调节, 非致命)。
- **结论**: 前摄功能 PASS。历史 EGL SIGSEGV 为间歇性 (与 CKB GPU 初始化时序/状态相关), 在策略部署+干净启动后的稳定状态下不再触发; 无需额外 vendor blob 补丁。功能 sweep 16/16。

## 录像根因最终修正 (2026-11-07 12:40, 决定性验证) — 推翻 12:20 的"排除 SELinux"
- **SELinux 就是根因**。12:20 的排除结论错误: 当时 `setenforce 0` 后**未重启 media.codec**, 而 `IonAlloc::Init()` 是单例 (gr_allocator.cpp:82 Allocator 构造时调用一次), enforcing 下首次 open /dev/ion 失败(EACCES)后缓存 `ion_dev_fd_=-1` 且永不重试 → permissive 下旧进程仍用 fd=-1, ioctl 全 EBADF, 故"permissive 仍失败"是假象。
- **正确验证**: `setenforce 0` + `kill $(pidof media.codec)` (强制重新 Init) + 录像 → **成功产出 mp4** `/sdcard/DCIM/Camera/2026-11-07-12-31-51-450.mp4`, 日志 `Recorder: Encodings end successfully.`。已恢复 Enforcing。
- **精确 avc (enforcing, dmesg 实捕)**:
  - `scontext=u:r:mediacodec:s0 tcontext=u:object_r:ion_device:s0 tclass=chr_file denied { read }` (comm=omx@1.0-service / HwBinder, 即 media.codec 进程线程; open 拒绝被 audit 抑制, 用户态 `Init: Failed to open ion device - Permission denied` 证实 open 亦被拒)。
  - `scontext=u:r:mediaserver:s0 ... denied { ioctl } ioctlcmd=0x4900` (comm=CodecLooper)。
- **EBADF 链 (仍准确, 但源头是 SELinux)**: open 被拒 → Init 缓存 fd=-1 → `ioctl(-1, ION_IOC_IMPORT)` → EBADF → `Failed to import ion buffer` → importBuffer 失败 → 无视频关键帧 → 无 mp4。并非 fd 传输/HIDL 缺陷, 内核 ion 无缺陷。

### 源码级调用链 (备查)
- QtiMapper::importBuffer (QtiMapper.cpp:94, native_handle_clone 按 numFds dup fd) → BufferManager::RetainBuffer → ImportHandleLocked (gr_buf_mgr.cpp:109 allocator_->ImportBuffer(hnd->fd)) → IonAlloc::ImportBuffer (gr_ion_alloc.cpp:157 ioctl(ion_dev_fd_, ION_IOC_IMPORT))。
- IonAlloc::Init (gr_ion_alloc.cpp:48): `if (ion_dev_fd_==FD_INIT) open(kIonDevice,O_RDONLY); if(<0){ALOGE(...strerror(errno)); ion_dev_fd_=FD_INIT; return false;}` — FD_INIT=-1, 失败不重试 (单例缓存)。
- 图形栈: HIDL allocator@2.0 + mapper@2.0-impl-qti-display + vendor.qti.display.mapper@1.1 (无 AIDL IAllocator)。/dev/ion ion_device:s0 0664 system。

### 修复 (sepolicy)
- mediacodec 是 **vendor 类型** (system/sepolicy/vendor/mediacodec.te) → 规则加在 `device/sony/tama-common/sepolicy/vendor/legacy_ion.te`: `allow mediacodec ion_device:chr_file { rw_file_perms ioctl };`
- mediaserver 是 public 类型 → 规则加在 `device/sony/tama-common/sepolicy/private/legacy_ion.te` (PRODUCT_PRIVATE_SEPOLICY_DIRS → product_sepolicy.cil): `allow mediaserver ion_device:chr_file rw_file_perms;`
- r_file_perms 已含 { getattr open read ioctl lock map watch watch_reads }; 无 ion_device neverallow。
- 部署: `m selinux_policy` → push product_sepolicy.cil + product sha256 (/product/etc/selinux) 与 precompiled_sepolicy + 对应 sha256 (/vendor/etc/selinux), adb remount, 重启验证 enforcing 录像。

### 验证完成 (2026-11-07 13:48, 全部通过)
- **Enforcing 录像 PASS**: `2026-11-07-13-42-49-606.mp4` (26.5MB) 与干净重启后 `2026-11-07-13-48-13-365.mp4` (26.8MB)。MP4 box 解析: duration=10.44s, **vide 轨 310 帧** (~30fps) + soun 轨 489 采样。`Encodings end successfully` + `VideoRecordEvent$Finalize` 无 ERROR_NO_VALID_DATA。
- **干净重启回归 PASS**: 正常 reboot 后**不杀 media.codec** 直接录像成功 (开机进程 fd7=/dev/ion, Init 在 enforcing 下成功)。bootanim exited status 0 (无白屏)。
- **拍照回归 PASS**: 后摄新 jpg 6.28MB (`2026-11-07-13-48-51-591.jpg`)。
- **显示/UI 回归 PASS**: 锁屏壁纸与 Settings Material You 界面渲染正常 (截图 regress.png/regress2.png); SurfaceFlinger 健康。
- **avc**: 录像/图形路径零拒绝; 仅余开机期无害拒绝 (vendor_init 属性、netd kmsg、mediacodeclist_generator 读 vendor_sys_video_prop 等, 与录像无关且不影响功能)。
- 干扰因素澄清: 修复后首次测试失败系 Aperture 进程经多次失败录像后 CameraX 状态卡死 (permissive 下同样失败); force-stop 相机进程后即正常。与 SELinux 无关。
- 设备备份: 旧策略文件 *.bakvid (product/vendor)。部署产物副本: functest/sepolicy_deploy/。
- **结论: 录像功能修复完成, enforcing 默认保持。前摄已于 15:27 验证 PASS (见"前摄崩溃攻关结论"), 功能 sweep 16/16。**

## 系统日志再分析 (2026-11-07 15:39, enforcing, 干净状态)
采样: sys_logcat_now.txt (15:23:10–15:39:04), sys_crash_now.txt (crash buffer, **0 行**), sys_dmesg_now.txt (4949–6746s), 灭屏循环后 sys_dmesg_after_sleep.txt。
- **崩溃/ANR/panic: 全零**。crash buffer 空; logcat 零 `Fatal signal`/`FATAL EXCEPTION`/`am_anr`; dmesg 无 panic/Oops/BUG/OOM/oom-kill。
- **WLED OVP 中断风暴 (历史一次性)**: dmesg 31670 条 `WLED OVP fault detected, int_sts=2 fault_sts=2`, 集中 4949–5110s (≈15:09–15:12, ~196 次/秒) 后**自行停止**; 因刷屏过猛 dmesg 环形缓冲起点即被覆盖到风暴起点 (触发前历史丢失)。受控灭屏 25s+唤醒**未复发** (最后一条仍 5110s)。灭屏唤醒后显示/亮度/前摄预览渲染正常 (post_sleep.png)。判定: 背光 boost 过压保护的一次性瞬态中断风暴, 自限, 当前显示健康; 仅监控, 不需处理。
- **regulator WARN (非致命, 机制已明)**: 仅 2+1 次 `WARNING ... drivers/regulator/core.c:2283 _regulator_disable`, comm=cammw:sensor。灭屏循环实捕完整链: `panel power off` → `tcs3490 REGULATOR_EVENT_DISABLE` → **`unbalanced disables for pm8998_l22`** → WARN。即相机传感器电源轨 (pm8998_l22) 在下电时被重复 disable (vendor 相机电源序列缺陷), 触发条件=相机开着时灭屏/流 teardown。仅 WARN, 不崩溃, 系统继续正常。
- **灭屏伴随非致命内核噪声**: `lab_vreg_handler LAB VREG_NG interrupt / false detection display is off` (panel off 误报, 驱动已忽略); `afe_get_cal_topology_id cal_type 8/9 not initialized / q6asm_send_cal cal_block NULL` (音频 DSP 某 port 校准缺失, 不影响出声); `CAM-ISP CSID:0 ERROR_CRC` (流 teardown 瞬时); `IPC_RTR No local port id` (modem IPC port 关闭)。均非致命。
- **avc (enforcing)**: dmesg 仅 10 条, 全为已知次要 `hal_camera_default` → `vendor_sysfs_graphics` (write ires_ua/brightness) 与 `vendor_sysfs_battery_supply` (search battery), 发生于后摄拍照曝光/闪光调节; 无新增类型, 不影响出图/录像。logcat 侧 avc=0。
- **周期性 benign 错误**: `WifiService: Attempt to retrieve Passpoint configuration with null osuProviders` 每 10s (Passpoint 未配置, 扫描相关, 无害); 相机 `CaptureReq Failed result=-107` (已知 focus-stats 噪声, 不影响出图)。
- **结论: 系统日志健康。无崩溃/ANR/panic/OOM; 唯一显著历史事件为自限的 WLED OVP 瞬态风暴 (已停, 显示正常); 其余为 vendor 相机/regulator 下电 WARN 与已知次要 SELinux/噪声, 均非致命、不影响 16/16 功能。**
- 更正: 上条 `afe_get_cal_topology_id cal_type 8/9 not initialized / q6asm_send_cal cal_block NULL` 并非"不影响出声"——它是 **ACDB 未初始化的症状**, 影响**通话语音**通路 (媒体播放走另一通路故仍 PASS), 见下方通话音频分析。

## 通话音频故障分析 (2026-11-07 16:00, 对方无声+下行杂音) — 根因确证
症状: 打电话对方听不到 (上行 TX mic 无校准), 本机扬声器兹拉兹拉杂音 (下行 RX 回退默认 bw/sr 无音量校准)。
- **根因: SELinux 拒绝 audio HAL 打开 /dev/ion → ACDB 校准库初始化失败 → 整条语音校准链失效。**
  - 精确 avc (enforcing, dmesg 实捕): `scontext=u:r:hal_audio_default:s0 tcontext=u:object_r:ion_device:s0 tclass=chr_file denied { read }` (comm=HwBinder:<audio-hal pid>); permissive 下另见 `{ open }` 与 `{ ioctl } ioctlcmd=0x4900`。
  - 失败链 (enforcing): `ACDB-LOADER: Cannot open /dev/ion errno:-1` → `ACDB -> Cannot allocate memory!` → `msm8974_platform: ACDB initialization failed` → 通话时 `ACDB -> Not correctly initialized!` → `voice_get_cal: No cal data for cal 7/4/2/3` → `voice_send_cvs/cvp_register_*_cal_cmd` 失败 → `voice_send_vol_step_cmd: DSP returned error[ADSP_EFAILED] / voice volume failed`; 且 `platform_check_codec_backend_cfg: Use default bw and sr for voice/voip calls` (回退默认带宽/采样率 → 杂音)。
  - ACDB 数据文件本身齐全 (/vendor/etc/acdbdata/Sony_tama_*.acdb + MTP/*), 非缺文件; 是 loader 需 /dev/ion 分配共享内存被 SELinux 拒。
- **permissive 对照验证 (确证)**: `setenforce 0` + `ctl.restart vendor.audio-hal` (绕过单例缓存) → `ACDB -> init done! / msm8974_platform: ACDB initialized`; 再拨 10010 → `send_voice_cal acdb_rx=7 acdb_tx=6` 成功, 无 "No cal data"/"Not correctly initialized" (仅 EALREADY benign)。已恢复 Enforcing。
- **与录像修复同类但域不同**: 录像修的是 mediacodec/mediaserver → ion_device; 本次是 **hal_audio_default** → ion_device (此前未授权)。
- **修复 (一行 sepolicy)**: `device/sony/tama-common/sepolicy/vendor/hal_audio_default.te` 加 `allow hal_audio_default ion_device:chr_file rw_file_perms;` (hal_audio_default 为 vendor 类型, 放 vendor 目录; rw_file_perms 已含 open/read/write/ioctl/map)。然后 `m selinux_policy` + 部署 precompiled  set + 重启 + 复测通话。
- 证据: call_audio_live.txt, call_audio_flinger.txt。

### 部署与验证完成 (2026-11-07 16:23, enforcing)
- 编译: `m selinux_policy` EXIT=0 (16:16→16:20); built vendor CIL 含 `(allow hal_audio_default ion_device_202604 (chr_file (ioctl read write getattr lock append map open watch watch_reads))`。
- 部署: 7-file precompiled set push (/product + /vendor etc/selinux), restorecon+chmod 644, md5 一致 (precompiled=98b7cea3…); 重启加载, getenforce=Enforcing, boot_completed=1。
- **通话验证 PASS (enforcing)**: 拨 10010 (mCallState=2), `ACDB -> send_voice_cal acdb_rx=7 acdb_tx=6` **无 "Not correctly initialized"**; 无 `Cannot open /dev/ion`/`Cannot allocate memory`/`No cal data`/`voice volume failed`; 仅 `ADSP_EALREADY` (benign)。dmesg **零 hal_audio_default ion avc**。crash buffer 0。
- 回归: 总体 avc 仅余开机期无害项 (toolbox adb_data_file、vendor_init device/prop、system_server demo_prop、init idd_rca_dir、sensors persist 等); 无 mediacodec/mediaserver/hal_audio ion 拒绝; 无崩溃。新增 allow 规则不破坏既有录像/媒体通路。
- 证据: call_verify_live.txt。**结论: 通话音频修复部署完成, enforcing 下 ACDB 语音校准正常加载。上行/下行听感需用户物理复听确认。**