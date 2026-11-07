# Kernel 4.9 性能特性回移计划（xz2c / SDM845 / Android 17）

设备：Sony Xperia XZ2 Compact (xz2c/apollo/H8324)，SDM845（ARMv8.2-A），kernel 4.9.337-perf+，A/B slot `_b`。
原则：**一次只回移一个特性，独立提交、独立编译/刷入/验证**。仅经 stock TWRP `fastboot boot` → adb recovery → `dd` → 哈希校验刷入 boot_b（`/dev/block/sda46`），绝不 `fastboot flash`。

## 现状前提（已核对 .config 与源码树）
- 调度器：高通 **WALT + schedtune**（`SCHED_WALT=y`/`SCHED_TUNE=y`），非 mainline EAS/uclamp。
- IO 默认 **cfq**（legacy）；kyber/bfq/blk-wbt 源码缺失。
- TCP：**BBRv1**（`tcp_bbr.c` 在树，无 bbr2 标记）；tcp_rack/mptcp/net-tls 缺失。
- zram：**zstd** 已激活，压缩比 ~5.16×；无 recompression/multi-comp；writeback 未开。
- 已落地：statx/close_range/clone3/process_madvise + CAKE/BBR/FQ/BPF_JIT + zram-zstd + WireGuard/PSI/UFFD/slab-rand。
- 设备健康：CPU ~15%、0% iowait、MemAvailable ~1.4GB、温度 ~33–35°C 无降频。剩余纯速度收益边际。

## 可补丁化候选（按收益/风险）

### 网络
| 特性 | 来源 | 风险 | 收益 | 状态 |
|---|---|---|---|---|
| RACK-TLP 丢包恢复 | 4.18 `tcp_recovery.c` | — | **已具备且默认启用**（`tcp_recovery=1`/`tcp_early_retrans=3`，TLP 激活） | 无需回移 |
| BBRv2 | 替换 `tcp_bbr.c` | 中高 | 公平性/低丢包，依赖新 TCP 钩子 | 暂缓 |

### IO 延迟（UFS 快设备，收益边际）
| 特性 | 来源 | 风险 | 收益 | 状态 |
|---|---|---|---|---|
| Kyber | 4.12 `block/kyber.c` | — | **不适用**：UFS sda 为 legacy 单队列（cfq），无 blk-mq，Kyber 无处挂载 | 不适用 |
| blk-wbt 写回限流 | 4.10 | 中 | 抑制写回延迟尖峰；设备 0% iowait，收益边际 | 仅剩可选 |

### 内存
| 特性 | 来源 | 风险 | 收益 | 状态 |
|---|---|---|---|---|
| **KSM** | `mm/ksm.c` 已在树，仅开关 | **低** | 页面去重省内存，CPU 扫描开销 | **已回退（无收益）** |
| zram recompression/multi-comp | 5.x | 中 | 空闲页二次压缩，zstd 已高比率收益有限 | 暂缓 |

### 文件系统
| 特性 | 来源 | 风险 | 收益 | 状态 |
|---|---|---|---|---|
| erofs | 5.4 `fs/erofs` | 中大 | 只读压缩分区随机读加速，**需重做镜像** | 暂缓 |

## 不可行（核心重构 / 硬件门槛）
- VM：MGLRU、folio、DAMON、maple tree、per-VMA lock。
- 调度：EEVDF、sched_ext（与 WALT 冲突）、PREEMPT_RT。
- IO：io_uring（依赖新 VM/block）。
- 硬件：MTE/PAC/BTI 需 ARMv8.4/8.5，SDM845=ARMv8.2 **不支持**。

## 适配顺序
1. **KSM** — 已试验并回退（无收益，见下）
2. **RACK-TLP** — 核查发现已具备且默认启用，无需回移（见下）
3. **Kyber** — 不适用（UFS legacy 单队列，无 blk-mq，见下）
4. 仅剩 blk-wbt（收益边际）/ zram recompression（zstd 已高比率）/ erofs（需重做镜像）

## Kyber 核查结论（不适用）
- 设备 `/sys/block/sda/queue/scheduler = noop deadline [cfq]`，且无 `/sys/block/sda/mq` → UFS 走 **legacy 单队列块层**，非 blk-mq。
- Kyber 是 blk-mq 多队列调度器（`blk_mq_register_sched`），只能挂 blk-mq 设备；本机无 blk-mq 设备（zram scheduler=none）→ 回移零收益。
- BFQ-mq 同理不适用；IO 侧仅剩 blk-wbt（sq/mq 通用）但设备 0% iowait、UFS 快，收益边际。

## 总体结论
本 4.9 + SDM845 平台的**有效性能回移空间已基本耗尽**：
- 网络：BBR/FQ/CAKE/RACK/TLP/BPF_JIT 全部具备并启用（RACK/TLP 原已在内核）。
- 内存：zram-zstd 已落地（5.16×）；KSM 无 mergeable 页无效；MGLRU/folio/DAMON 不可行。
- IO：legacy 单队列 + 快 UFS + 0% iowait；Kyber/BFQ-mq 不适用；blk-wbt 边际。
- 调度：高通 WALT+schedtune 已优化；EEVDF/sched_ext 不可行且冲突。
- 剩余可选项收益均边际，不建议再为微小收益反复刷机。

## RACK-TLP 核查结论（无需回移）
- 初判误查旧文件名 `net/ipv4/tcp_rack.c`（4.13–4.17 用名，缺失）；4.18+ 已改名为 **`net/ipv4/tcp_recovery.c`**。
- 该文件已在树（3549B，含 `tcp_rack_mark_lost`/`tcp_rack_advance`/`tp->rack`），Makefile 编译 `tcp_recovery.o`，`tcp_input.c` 已接入钩子。
- 默认 `sysctl_tcp_recovery = TCP_RACK_LOST_RETRANS`(0x1)；TLP 完整（`tcp_send_loss_probe`/`tlp_high_seq`/`tcp_schedule_loss_probe`）。
- 设备运行时 `tcp_recovery=1`、`tcp_early_retrans=3` → **RACK + TLP 均已激活**。Android/CAF common 内核早已回移，无需任何改动。

## KSM 实验结论（内核 #19，已回退到 #18）
- 启用 `CONFIG_KSM=y` + init.rc `run=1`/`pages_to_scan=256`/`sleep_millisecs=500`，ksmd 线程正常运行、CPU 0.0%、全套回归通过。
- **但收益为零**：`pages_sharing=0`、`pages_unshared=0`、`full_scans=0`。精确扫描全系统 `/proc/*/smaps` VmFlags，**0 个进程标记 `MADV_MERGEABLE`**（`TOTAL_MERGEABLE_PROCS=0`）。
- 根因：KSM 仅合并进程主动 `madvise(MADV_MERGEABLE)` 的内存；现代 Android 17 已放弃 KSM（KitKat 时代用法），改用 zram，userspace 不再标记 mergeable 页。
- 要生效需侵入式 userspace 改造（madvise 标记守护进程），收益不确定且有风险 → **不值得**。
- 处置：还原 defconfig/init.rc，刷回 `boot_b_backup_ksm18.img`（#18，md5 `4a699133`）；KSM 内核镜像备份为 `boot_b_backup_rollback19.img`。

## KSM 适配步骤
1. defconfig 追加 `CONFIG_KSM=y`（依赖 MMU/PROC_FS/SYSFS，均满足）。
2. init.qcom.rc 启用 ksmd：`write /sys/kernel/mm/ksm/run 1`，并设低开销扫描参数（`pages_to_scan`、`sleep_millisecs`）。
3. 编译 boot.img → 核对 `.config` `CONFIG_KSM=y` → TWRP 刷入 → 部署 rc。
4. 验证：`/sys/kernel/mm/ksm/run=1`、`pages_shared`/`pages_sharing` 增长、`full_scans` 推进；全套回归（syscall/SELinux/avc/崩溃/相机/SIM/BT/touch/网络）。
5. 对比内存（MemAvailable、zram mm_stat）与 CPU（ksmd 占用），评估收益/开销决定保留或回退。

## 验证脚本约定
- 等待 `sys.boot_completed=1`；相机检查带重试（避免 provider 启动时序误判）。
- 回归二进制：`/data/local/tmp/{pmtest64,pmtest32,statxtest64,statxtest32,c3test64}`，期望全 0。
- 刷入脚本带 `set -e` 哈希断言（push/readback md5 必须等于 EXPECT_MD5）；备份当前 boot_b 后再写。