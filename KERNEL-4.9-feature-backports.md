# XZ2 Compact (sdm845) / RisingOS 17 — 4.9.337 内核特性回移清单

> 设备：Sony Xperia XZ2 Compact（apollo / sdm845）
> 系统：RisingOS Revived 9.0（Android 17）
> 内核：Linux 4.9.337-perf+（LineageOS `android_kernel_sony_sdm845`，branch `lineage-23.0`）
> 整理日期：2026-11-06

## 0. 结论

- Android 侧内核上限就是 **4.9.337**（4.9 LTS 已于 2023-01 EOL），没有可用的 4.14/5.x Android 路径。
- RisingOS 17 在 4.9.337 上已能正常启动，**框架侧目前没有硬性缺口**。
- 继续获得新能力的方式是"按需回移新特性"，而不是换内核。本树已有多个成功先例（MADV_WIPEONFORK、pidfd、PSI、WireGuard），路线成熟。
- 换内核（4.14/5.4/主线 6.x/7.x）不可行的根因是 Sony/Qualcomm 闭源 vendor 栈（libexcal 相机、Adreno/kgsl、显示/音频/QMI）按 4.9 ABI 编译。

## 1. 本树已具备的新特性（回移先例）

| 特性 | 上游版本 | 本树证据 |
|---|---|---|
| `MADV_WIPEONFORK` / `MADV_KEEPONFORK` | 4.14 | `mm/madvise.c:82`、`include/uapi/asm-generic/mman-common.h:61` |
| `pidfd_open` / `pidfd_send_signal` | 5.3 / 5.1 | `kernel/pid.c:611`、`kernel/signal.c:3028`；bionic `pidfd_open()` 可用 |
| PSI（内存压力信息） | 4.20 | 内核 `.config`：`CONFIG_PSI=y`（lmkd PSI 模式） |
| eBPF / cgroup-bpf | 4.19 部分 | `CONFIG_BPF_SYSCALL=y`、`CONFIG_CGROUP_BPF=y`（netd 流量统计） |
| WireGuard | 5.6 | 内核树内已有，`CONFIG_WIREGUARD` 已开 |
| Spectre-BHB 缓解 | 5.x 回移 | `CONFIG_MITIGATE_SPECTRE_BRANCH_HISTORY=y` |
| 基础硬化 | — | `CONFIG_RANDOMIZE_BASE=y`、`CONFIG_HARDENED_USERCOPY=y`、`CONFIG_FORTIFY_SOURCE=y`、`CONFIG_CC_STACKPROTECTOR_STRONG=y` |

构建配置来源：`out/target/product/xz2c/obj/KERNEL_OBJ/.config`（由 `tama_apollo_defconfig` 生成）。

## 2. 推荐回移（按收益 / 成本排序）

### 2.1 USERFAULTFD + UFFDIO_WRITEPROTECT / UFFDIO_ZEROPAGE（优先）

- **来源**：userfaultfd 4.3；`UFFDIO_WRITEPROTECT` 5.10；`UFFDIO_ZEROPAGE` 4.11。
- **收益**：解锁 ART mark-compact GC 的 uffd 并发压缩路径，减少 GC 停顿，对 4GB 内存机器体感明显。
- **现状**：`fs/userfaultfd.c` 已存在于树中，但 `CONFIG_USERFAULTFD` 未开启（`CONFIG_USERFAULTFD` 不在 .config 中）。
- **成本 / 风险**：中。需要开启配置，并回移 WP/zeropage ioctl 及对应的 mm 钩子（页表写保护、mmap_lock 语义）。
- **参考**：`art/runtime/gc/collector/mark_compact.cc` 已有 uffd 特性检测逻辑。

### 2.2 MADV_COLD / MADV_PAGEOUT + process_madvise

- **来源**：`MADV_COLD`/`MADV_PAGEOUT` 5.4；`process_madvise` 5.10。
- **收益**：整进程级冷页回收，lmkd/AMS 可以更精准地释放后台内存，替代粗暴杀进程。
- **现状**：`MADV_COLD`/`MADV_PAGEOUT` 未定义；`process_madvise` 未实现；但所需 `pidfd_open` 已具备。
- **成本 / 风险**：中。涉及 `mm/vmscan.c` reclaim 接口和跨进程权限检查（`PTRACE_MODE_ATTACH_REALCREDS`）。

### 2.3 zstd 压缩后端 + 开启 BBR

- **来源**：zstd 内核库 4.14；`CONFIG_TCP_CONG_BBR` 4.9 原生已有选项。
- **收益**：
  - zram 使用 zstd 压缩，压缩率更高、内存更省（配合 4GB 机型）；
  - BBR 拥塞控制改善移动网络吞吐/延迟。
- **现状**：
  - `lib/zstd/` 已存在于树中，`lib/Makefile:114-115` 有 `CONFIG_ZSTD_COMPRESS/DECOMPRESS` 构建钩子；
  - `.config` 中 `CONFIG_CRYPTO_ZSTD` 未开；
  - `CONFIG_TCP_CONG_BBR` 选项存在（`net/ipv4/Kconfig:654`）但未启用。
- **成本 / 风险**：低。主要改 defconfig；zram zstd 后端需要少量适配（4.9 的 zram 默认只有 lz4/lzo）。
- **注意**：zstd 还需在 `CONFIG_ZSTD_COMPRESS/CONFIG_ZSTD_DECOMPRESS` 打开后确认镜像体积增长可接受。

### 2.4 statx / close_range / clone3 兼容性补洞

- **来源**：`statx` 4.11；`close_range` 5.9；`clone3` 5.3。
- **收益**：
  - bionic 的 `statx()` 是裸系统调用，没有用户态回退，部分 API 30+ 应用会拿到 `ENOSYS`；
  - `close_range` 影响 `posix_spawn()` 路径；
  - `clone3` bionic 已有 clone 回退，优先级最低。
- **现状**：这三个系统调用在内核树中均不存在（`__NR_statx`/`__NR_clone3`/`close_range` 均无实现）。
- **成本 / 风险**：低-中。`close_range`/`clone3` 代码量小；`statx` 需要新的 `struct statx` 与 vfs 接口，中等。

### 2.5 ZRAM writeback

- **来源**：4.14/4.19。
- **收益**：冷页写回 UFS backing device，缓解 4GB 物理内存压力。
- **现状**：`.config` 中有 `CONFIG_ZRAM_WRITEBACK` 选项但为 `not set`。
- **成本 / 风险**：中低。需要配置 backing device 和回写策略参数。

### 2.6 CAKE qdisc

- **来源**：4.19。
- **收益**：移动网络 bufferbloat 改善，延迟更稳定。
- **现状**：`net/sched/Kconfig` 中不存在 `NET_SCH_CAKE`。
- **成本 / 风险**：低。独立模块，不涉及 vendor ABI。

### 2.7 安全硬化补强

- **候选**：`SLAB_FREELIST_HARDENED`、`SLAB_FREELIST_RANDOM`、`BUG_ON_DATA_CORRUPTION`、`REFCOUNT_FULL`、`INIT_ON_ALLOC/FREE`、`PAGE_POISONING`、`SHUFFLE_PAGE_ALLOCATOR`。
- **收益**：提升内核内存安全，减少漏洞利用面。
- **成本 / 风险**：低（内核内部），但 `INIT_ON_ALLOC/FREE` 有性能开销，需要压测取舍。

### 2.8 eBPF 补全（按需）

- **来源**：5.x 的 map type/helper 回移。
- **收益**：让 netd 的带宽/UID 统计少走 legacy 路径。
- **成本 / 风险**：按实际需求评估。4.9 已有 `CONFIG_BPF_SYSCALL`/`CONFIG_CGROUP_BPF` 基础。

## 3. 不建议移植

| 特性 | 原因 |
|---|---|
| MGLRU（6.1） | 收益最大，但依赖 folio/maple tree/memcg 整体重构，4.9 等于重写 mm，工程不可行 |
| folio / maple tree / GKI 化 | 同上，牵一发动全身；sdm845 早于 GKI，vendor blob 不可模块化 |
| MTE（内存标记） | sdm845 无 ARMv8.5 MTE 硬件 |
| sched_ext | Android 不使用，风险大于收益 |
| io_uring | Android 框架/NDK 基本不用 |
| memfd_secret | 收益极低 |
| mseal 完整语义（6.10） | 可做简化版 `VM_SEALED` 检查，但价值主要是加固；依赖新的 mmap lock 语义 |
| AVF/pKVM Linux Terminal | 需要 Android common 5.10+ 的 pKVM，4.9 无法实现 |
| FUSE passthrough（6.9） | /sdcard 性能收益大，但 4.9→6.x FUSE 内部差异过大，风险最高，暂不做 |

## 4. 落地顺序建议

1. **零风险配置项**（单独编译验证启动）：
   - `CONFIG_USERFAULTFD=y`
   - `CONFIG_ZSTD_COMPRESS=y` / `CONFIG_ZSTD_DECOMPRESS=y`
   - `CONFIG_TCP_CONG_BBR=y`
2. **小代码回移**（每项独立 commit）：
   - `close_range` → `statx` → `clone3`
3. **mm 类中等项**（必须做完整回归）：
   - `UFFDIO_WRITEPROTECT`/`UFFDIO_ZEROPAGE`
   - `MADV_COLD`/`MADV_PAGEOUT` + `process_madvise`
   - ZRAM writeback
4. **可选低风险项**：
   - CAKE qdisc、安全硬化配置

原则：像 `MADV_WIPEONFORK`、WireGuard 那样，一次只回移一个特性，独立提交、独立验证，避免 mm 改动互相掩盖问题。

## 5. 每次回移后的验证清单

- [ ] 内核编译通过（`m kernel` 或直接 ninja 对应目标）
- [ ] `boot_completed=1`，SELinux enforcing
- [ ] GMS / Play 商店正常，无 ANR 循环
- [ ] 相机前后置拍照正常（libexcal / msm_camera 未受影响）
- [ ] 蜂窝网络 IN_SERVICE，VoLTE 拨号正常（10010）
- [ ] 触摸 HIDL HAL 运行（`init.svc.vendor.touch-hal-1-0=running`）
- [ ] GNSS 无 `ctrl_port_init` 报错，能报出 modem 型号串
- [ ] 蓝牙 MAC 为工厂地址（`D4:38:9C:9A:9C:6A`）
- [ ] 无新增 `avc: denied`（排除 adbd 自身回显）
- [ ] crash buffer 为空
- [ ] 内存压力测试（后台多任务 + 大应用）无异常 OOM/lmkd 误杀

## 6. 参考路径

- 内核源码：`/root/android/rising/kernel/sony/sdm845`（WSL）
- defconfig：`kernel/sony/sdm845/arch/arm64/configs/tama_apollo_defconfig`
- 编译产物配置：`out/target/product/xz2c/obj/KERNEL_OBJ/.config`
- 设备树 sepolicy 热修复：`/vendor/etc/selinux/`（`.bak2`~`.bak5` 为历史备份）
