# Linux 6.12.6 NAPI 实现与 sk_buff 分配失败原因分析

> 基于内核源码树 `/home/wyou/kernel/6.12.6-200/linux-6.12.6` 的代码走读。

## 一、NAPI 实现概览（6.12.6）

### 核心调度循环

- `napi_struct`（include/linux/netdevice.h:348）：每个 NAPI 实例挂到 per-CPU 的 `softnet_data.poll_list`，由 `NAPI_STATE_SCHED` 位控制入队/出队。
- 中断中驱动调 `__napi_schedule()`（net/core/dev.c:6159）→ `____napi_schedule()`（dev.c:4511）将 napi 挂到本 CPU 的 `poll_list` 并 raise `NET_RX_SOFTIRQ`。
- `net_rx_action()`（dev.c:6931）是 softirq 主循环：受两个配额约束——`netdev_budget`（默认 300 包）和 `netdev_budget_usecs`（默认 2 jiffies 时间窗，dev.c:6934-6937）。耗尽则 `sd->time_squeeze++` 并重新 raise softirq（dev.c:6976-6989）。
- `__napi_poll()`（dev.c:6765）调驱动的 `n->poll(n, weight)`，weight 耗尽时执行 `napi_gro_flush()` + `gro_normal_list()` 并把 napi 移到 repoll 链表。
- 6.12 支持 threaded NAPI（`NAPI_STATE_THREADED`，dev.c:6882-6929），但即使在线程里也是 `local_bh_disable()` 上下文——**对内存分配器而言依然是 atomic 上下文**。

### RX 路径上的 sk_buff 分配入口（驱动在 poll 里调用）

| 入口 | 位置 | 结构体来源 | 数据区来源 |
|---|---|---|---|
| `napi_alloc_skb()` | skbuff.c:803 | per-CPU skb cache / slab | per-CPU page_frag（含 1K 小页）或 kmalloc |
| `__netdev_alloc_skb()` | skbuff.c:726 | slab | page_frag / kmalloc |
| `build_skb()` / `napi_build_skb()` | skbuff.c:489 / 557 | slab（`GFP_ATOMIC`）| 驱动自己供的 page/page_pool buffer |
| `napi_get_frags()` | skbuff.c | slab | page_frag（frags 模式） |
| page_pool（mlx5/ice 等）| page_pool.c | 驱动 build_skb | pool 分配的 page |

## 二、sk_buff 分配失败的可能原因

### 1. GFP_ATOMIC 上下文约束（最根本的系统性原因）

NAPI 在 softirq 或关 BH 上下文运行，所有 RX 分配统一使用 `GFP_ATOMIC | __GFP_NOWARN`（skbuff.c:805, 424, 476）：

- **不允许 direct reclaim / compaction / IO**：分配器只能靠 zone 空闲页、per-CPU pageset、slab 现有对象和 atomic reserves。空闲内存跌破 `WMARK_MIN` 且预留耗尽即返回 NULL，**没有任何重试**。
- `__GFP_NOWARN`：失败**不会打 dmesg**，只能通过计数器发现（见第四节）。
- `__GFP_MEMALLOC`（紧急预留）仅当系统里存在 `SO_MEMALLOC` 套接字时才生效（skbuff.c:660 `sk_memalloc_socks()`；829-830）。普通服务器上 RX 分配**无权动用 emergency reserves**。
- threaded NAPI 看似进程上下文，但 `local_bh_disable()` + 驱动传 GFP_ATOMIC，行为相同。

### 2. sk_buff 结构体（slab）分配失败

- `__alloc_skb()` 第一步 `napi_skb_cache_get()`（skbuff.c:666）：per-CPU 缓存（64 个槽位）为空时，用 `kmem_cache_alloc_bulk(..., GFP_ATOMIC, 16, ...)` 补充（skbuff.c:353），**bulk 分配一个都拿不到就直接返回 NULL**（skbuff.c:357-359）。
- 非 NAPI 路径或跨 NUMA 节点时（`node != numa_mem_id()`，skbuff.c:664-668）退化为 `kmem_cache_alloc_node()`：slab 需要向伙伴系统申请新 slab 页，GFP_ATOMIC 页分配失败 → NULL。
- `build_skb()` / `napi_build_skb()` / `slab_build_skb()` 直接 `kmem_cache_alloc(skbuff_cache, GFP_ATOMIC)`（skbuff.c:475, 423）：失败时返回 NULL。注意 **build_skb 失败时 data 缓冲区由调用方负责**——驱动若处理不当还会伴随 page/page_pool 泄漏。

### 3. skb->head 数据区分配失败

`napi_alloc_skb()` 按长度分流（skbuff.c:819-820）：

- **len ≤ ~1K（4K 页系统）**：`page_frag_alloc_1k()`（skbuff.c:240），补充页是 order-0 `alloc_pages_node(NUMA_NO_NODE, gfp, 0)`（skbuff.c:249），GFP_ATOMIC 失败 → NULL。
- **1K < len ≤ PAGE_SIZE**：`page_frag_alloc()` → refill 走 `__page_frag_cache_refill()`（mm/page_alloc.c:4868）：4K 页系统上**优先申请 order-3（32KB）复合页**，且带 `__GFP_NORETRY | __GFP_NOMEMALLOC | ~__GFP_DIRECT_RECLAIM`（page_alloc.c:4874-4878）。**内存碎片化导致 order-3 失败是这里最典型的失败原因**；随后回落 order-0（page_alloc.c:4882）仍可能失败。
- **len > PAGE_SIZE（jumbo 线性 buffer 等）**：走 `__alloc_skb()` → `kmalloc_reserve()`（skbuff.c:577-622）：高阶 kmalloc。先带 `__GFP_NOMEMALLOC` 试一次（skbuff.c:609），失败且当前上下文无权用 pfmemalloc reserves 就直接 NULL。**jumbo frame（如 9K MTU 线性分配需要 order-3 以上连续内存）在长期运行的系统上因碎片化失败是经典问题**。
- `kmalloc_reserve` 的二次尝试用 pfmemalloc 预留（skbuff.c:615-617），但前提是 `gfp_pfmemalloc_allowed()`——NAPI 默认 GFP_ATOMIC 不具备该权限（无 `__GFP_MEMALLOC`），所以这条退路通常无效。

### 4. page_pool 分配失败（主流高速驱动的主要路径）

mlx5/ice/mvneta/mvpp2 等驱动不直接分配 skb，而是从 page_pool 拿 buffer 再 `build_skb()`：

- fast path（`__page_pool_get_cached`）缓存空 → slow path `__page_pool_alloc_pages_slow()`（page_pool.c:518）：
  - order-0 pool：`alloc_pages_bulk_array_node(gfp, nid, ...)` 一个页都拿不到 → 0（page_pool.c:538-542）。
  - 高阶 pool（`p.order > 0`，jumbo）：`alloc_pages_node(nid, gfp, order)`（page_pool.c:498）——同样受碎片化影响，且**绑定设备 NUMA 节点**，节点级压力即失败，不会跨节点 fallback。
- **DMA 映射失败**：`page_pool_dma_map()` 中 `dma_map_page_attrs()` 返回 `dma_mapping_error`（page_pool.c:470-475, 502-504, 549-551）——IOMMU IOVA 空间耗尽或 DMA 子系统错误时，即使物理页分配成功也会整体失败。这在 32-bit 寻址受限设备或 IOMMU 压力大的场景会被忽视。
- dmabuf memory provider（devmem TCP 等）路径 `mp_dmabuf_devmem_alloc_netmems` 失败（page_pool.c:587-588）。

### 5. NUMA 拓扑因素

- page_pool 固定 `p.nid`；`__alloc_skb` 的 `node` 参数；kmalloc/page_frag 均是 `alloc_pages_node` / `kmalloc_node`——**只在本节点分配**。本节点内存紧张而其它节点空闲时，GFP_ATOMIC 不会迁移，直接失败。
- per-CPU `napi_alloc_cache` 的 page_frag 是 `NUMA_NO_NODE`，倾向当前节点，同样受节点压力影响。

### 6. "伪失败"：pfmemalloc skb 在协议栈被丢弃

分配器动用了 reserves 时页被标记 pfmemalloc，`skb->pfmemalloc = 1`（skbuff.c:778, 866）。这类 skb **分配成功**了，但后续在 UDP/TCP 层若找不到 memalloc 套接字会被丢弃（`sk_filter_trim_cap`/`__udp_enqueue_schedule_skb` 等处的 pfmemalloc 检查）。从观测上看（rx_missed/丢包）与分配失败几乎无法区分，是排查时容易混淆的一点。

### 7. 相邻但非"分配失败"的丢包点（排查时需区分）

- `netdev_budget` / 2-jiffies 时间窗耗尽 → `time_squeeze++`（dev.c:6978），不丢包但加大延迟；驱动侧 ring 满 → NIC 自身 rx_missed。
- 非 NAPI 入口（RPS、无 NAPI 驱动）经 `enqueue_to_backlog()`：`input_pkt_queue` 长度 ≥ `netdev_max_backlog` → `sd->dropped++`（dev.c:4874-4901）。
- GRO 合并中 `napi_get_frags` 失败、skb_gro_receive 等处的次要分配失败。
- 驱动拿到 NULL 后的行为差异：有的计 `rx_dropped`，有的计 `alloc_fail_buff`，并且无法给 ring 补充描述符 → 后续包被硬件丢弃（rx_no_buffer/rx_missed 计数上涨）。

## 三、sk_buff 的"小池子"全景图

> 问题现场：`/proc/meminfo` 中 `MemAvailable` 还有很多，但 sk_buff 分配失败。
> 关键认知：一次 RX 分配其实是**两个对象**——`struct sk_buff` 结构体（约 256B 元数据）和 `skb->head` 数据区，各有各的池子链，任何一个失败整体返回 NULL。

### 结构体 struct sk_buff 的供应链

```
napi_alloc_skb (skbuff.c:803)
  └─ __alloc_skb (skbuff.c:649)
      └─ napi_skb_cache_get (skbuff.c:346)
          ├─ 池① per-CPU napi_alloc_cache.skb_cache
          │     64 个槽位，skbuff.c:219/285
          │     ↓ 空了：kmem_cache_alloc_bulk 补 16 个 (skbuff.c:353)
          ├─ 池② SLUB per-cpu freelist（当前 cpu slab 页内空闲对象）
          │     ↓ 空了：换 slab
          ├─ 池③ per-node slab partial 链表（有空闲对象的旧 slab 页）
          │     ↓ 空了：向伙伴系统要新 slab 页  ★ 第一次碰页分配
          ├─ 池④ per-CPU pageset（PCP，每 zone 每 CPU 一小批页）
          │     ↓ 空了：
          └─ 池⑤ zone->free_area[本节点][UNMOVABLE/RECLAIMABLE]
                水位检查：NR_FREE_PAGES > WMARK_MIN（可再借一点预留）
                ✗ 失败 → 返回 NULL —— 到此为止，不能再往下
```

### 数据区 skb->head 的供应链

```
napi_alloc_skb 按长度分流 (skbuff.c:819)
  ├─ ≤1K：池⑥ per-CPU page_frag_1k：一个 order-0 页切 4 片 (skbuff.c:240)
  │        ↓ 用完：alloc_pages_node(order-0) → 池④⑤
  ├─ ≤PAGE_SIZE：池⑦ per-CPU page_frag_cache：一个 order-3(32KB) 页 (skbuff.c:282)
  │        ↓ 用完：__page_frag_cache_refill (mm/page_alloc.c:4868)
  │              先要 order-3（池⑤的 free_area[3]）→ 失败回落 order-0
  └─ >PAGE_SIZE：kmalloc_reserve (skbuff.c:577) → kmalloc 大对象
                 → 直接池④⑤的高阶 free_area[order]
```

### page_pool 驱动（mlx5/ice 等）的数据区供应链

```
驱动 poll → page_pool_alloc_netmem (page_pool.c:577)
  ├─ 池⑧ pool->alloc.cache：128 槽 (include/net/page_pool/types.h:50)，每条 RX 队列独占
  │     ↓ 空了：page_pool_refill_alloc_cache → 从 ptr_ring 回收跨 CPU 归还的页
  ├─ 池⑨ pool->ring（ptr_ring，page_pool.c:215：其它 CPU 释放回来的页）
  │     ↓ 也空了：__page_pool_alloc_pages_slow (page_pool.c:518)
  │           order-0：alloc_pages_bulk_array_node 一次补 64 页
  │           高阶：  alloc_pages_node(nid, order)
  └─ → 池④⑤ → DMA 映射（IOMMU IOVA 是另一个独立池子⑩）
```

### 终止线：GFP_ATOMIC 的视野边界

所有链条最终汇聚到池⑤（zone 空闲链表），GFP_ATOMIC 在这里的约束：

- 只看 `NR_FREE_PAGES` **当前快照**（mm/page_alloc.c:3213），不许回收；
- 水位门槛是 **WMARK_MIN**（page_alloc.c:3995），只能借两小笔钱：`ALLOC_MIN_RESERVE`（page_alloc.c:4011）和 order>0 时的 `ALLOC_HIGHATOMIC`（page_alloc.c:4023）——总量约是 `min_free_kbytes` 的一小部分；
- 基本不从 **MOVABLE** migratetype fallback（UNMOVABLE 仅回落 RECLAIMABLE；pageblock 整偷条件苛刻）；**MIGRATE_CMA 对 GFP_ATOMIC 完全不可见**；
- 不许跨 **NUMA 节点**、不许跨 **zone**；
- 高阶请求还要求**物理连续**。

### 为什么 MemAvailable 帮不上忙

`MemAvailable ≈ MemFree + 可回收 page cache + SReclaimable 估算`。对照池子：

| MemAvailable 的成分 | GFP_ATOMIC 能用吗 |
|---|---|
| 可回收 page cache（大头） | ✗ 需要 direct reclaim，禁止 |
| SReclaimable（可回收 slab） | ✗ 同上 |
| MemFree 中 MOVABLE migratetype 的页 | ✗ 基本不可 fallback |
| MemFree 中 CMA（MIGRATE_CMA）的页 | ✗ 永不可用（Qualcomm 等平台重点排查） |
| MemFree 中其它 NUMA 节点/其它 zone 的页 | ✗ 拓扑隔离 |
| MemFree 中本节点本 zone 的页，但低于 WMARK_MIN | ✗ 只能借一小点预留 |
| 高阶请求所需 order-N 连续块 | ✗ free 总量够但没有连续块（碎片化） |

**"小池子"的精确定义**：

> 本节点 × 本 zone × UNMOVABLE/RECLAIMABLE migratetype × 目标 order 的 `free_area` 当前快照，扣除 WMARK_MIN，最多再借一点 min/high-atomic 预留。

在 4K 页系统上，对普通帧（order-3 page_frag 补充）这个池子往往只有**几 MB 到几十 MB** 量级——`MemAvailable` 显示几十 GB 也完全可能它已见底。

### 现场定位：到底是哪一级池子见底

```bash
# 1. 碎片化：看目标节点的 order-3 和 order-0 还剩多少
grep -A1 "Node 0" /proc/buddyinfo
# 2. migratetype 偏科：UNMOVABLE/RECLAIMABLE vs MOVABLE vs CMA 的 free 对比
cat /proc/pagetypeinfo
# 3. 水位距离：本 zone nr_free_pages 离 min 有多远
awk '/Node|nr_free_pages|^  min|protection/' /proc/zoneinfo
# 4. 结构体池：skbuff_head_cache 是否在挣扎
grep skbuff /proc/slabinfo
# 5. 区分结构体失败还是数据区失败（order/gfp_flags 直接告诉你层级）
#    tracepoint: kmem:mm_page_alloc 失败事件；或 kprobe napi_alloc_skb 返回值
```

**快速判别口诀**：

- buddyinfo 里 order-3 是 0 但 order-0 很多 → **碎片化**（池⑦/高阶 kmalloc 见底），系统内存其实很富裕；
- order-0 也很少且 nr_free_pages 贴着 min → 真实的局部内存压力；
- pagetypeinfo 里 MOVABLE/CMA 很多、UNMOVABLE 很少 → **migratetype 偏科**（典型于内存被用户态页或 CMA 预留吃满的机器）；
- slabinfo 正常、内存也正常 → 查 page_pool 计数和 IOMMU（池⑧⑨⑩）。

### 与 Linux 内存管理的关系（在 MM 全景中的位置）

```
┌─ 用户态/虚拟内存层 ────────────────────────────────────────────
│  匿名页、page cache、tmpfs/shmem
│  GFP_HIGHUSER_MOVABLE → MIGRATE_MOVABLE
│  可回收（回收/换页）→ MemAvailable 的大头来自这里
├─ 内核对象层（SLUB/SLAB）──────────────────────────────────────
│  可回收 slab：dentry/inode（注册 shrinker）→ SReclaimable
│  不可回收 slab：skbuff_head_cache、kmalloc-*  → SUnreclaim
├─ 网络子系统私有缓存层 ← 【sk_buff"小池子"全景图所在层】────────
│  池① napi skb_cache │ 池⑥⑦ page_frag │ 池⑧⑨ page_pool
│  ★ 对 MM 完全不可见：无 shrinker、无回收接口、无 meminfo 统计
├─ 页分配器（buddy）────────────────────────────────────────────
│  池④ PCP → 池⑤ free_area[node][zone][migratetype][order]
│  水位 min/low/high ＋ 预留（min-reserve、high-atomic、CMA 隔离）
└─ 物理内存：node × zone × migratetype × CMA/reserved ──────────
```

与 MM 的 5 层关系：

1. **消费者-供应商关系**：全景图所有池子都是 buddy 分配器的"下游消费者缓存"。MM 把它们持有的内存一律视为**已分配、不可回收**——`MemAvailable` 统计不到它们，shrinker 也够不着它们。它们归还真内存的唯一途径是协议栈消费完 skb、驱动释放 buffer。
2. **回收方向与分配方向相反**：压力时 MM 从上往下"催收"（page cache → 可回收 slab → 换页）；而 GFP_ATOMIC 只能"取现"，不能触发催收。RX 路径只能吃到 MM 已经催收完、沉淀在池⑤底部的存量。
3. **GFP 标志是层级通行证**：`__GFP_HIGH`（含于 GFP_ATOMIC）换来 `ALLOC_MIN_RESERVE`——6.12 的精确语义是 `min -= min/2` 再 `min -= min/4`（mm/page_alloc.c:3137-3148），即 **GFP_ATOMIC 可以把 min 水位借到只剩 3/8**；order>0 额外走 `MIGRATE_HIGHATOMIC` 预留块。这是网络层在 MM 里唯一的"特权"，也是全部特权。
4. **RX 是原子预留的最大宗消费者之一**：`min_free_kbytes`（= `4×√lowmem_kbytes`，page_alloc.c:6124-6125）划出的预留池，主要给中断上下文里的网络 RX、存储 IO 这类"不能等"的分配准备。但它是**全系统所有 atomic 消费者共享**的——网卡、NVMe 中断、其它外设在抢同一笔钱。
5. **与虚拟内存层共享同一物理池**：用户态 page cache/匿名页的膨胀会挤压池⑤中 UNMOVABLE/RECLAIMABLE 的占比（migratetype 偏科）；CMA 划走的物理范围对 RX 永久关闭。"用户态内存很健康"和"RX 分配失败"可以同时成立。

**一句话**：sk_buff 全景图是架在 buddy 分配器之上、MM 回收体系之外的一层**私有、不可回收、只许取现不许赊账**的缓存栈；其总供给 = 池⑤中那个多维小池子的瞬时值，与 MemAvailable 只有微弱的间接关系。

### 每级池子的容量（4K 页、64 位内核）

| 池 | 单次可获取 | 池容量 | 补充粒度 | 源码依据 |
|---|---|---|---|---|
| ① napi skb_cache | 1 个 skb 结构体（256B） | **64 个 ≈ 16KB/CPU** | 空时补 16 个 | skbuff.c:219-220, 353 |
| ② SLUB cpu slab | 1 个对象 | 1 个 slab（skbuff 对象 256B，约 **16~128 个 ≈ 4~32KB**） | 换 slab | skbuff.c:5104（HWCACHE_ALIGN） |
| ③ node partial | 1 个对象 | 无硬上限，常见 **几十 KB ~ 几 MB** | 逐 slab | 动态 |
| ④ PCP | 1 ~ batch 页 | batch ≈ min(zone页数>>10, 256)/4 取整；high ≈ 6×batch → **典型几百页 ≈ 1~3MB/CPU**（3 种 pcptype 共享） | 按 batch | page_alloc.c:5523-5535 |
| ⑤ free_area 可用额 | 所需 order | **NR_FREE_PAGES − min×3/8**（即可借 min 的 5/8）；order>0 另有 high-atomic 预留，**上限 = zone 的 1%** | — | page_alloc.c:3137-3148, 2046 |
| ⑥ page_frag_1k | **1KB** | 1 个 order-0 页 = **4 片** | 整页补充 | skbuff.c:240-261 |
| ⑦ page_frag_cache | 1 片（≤PAGE_SIZE 减对齐） | 1 个 **32KB**（order-3）页，约 20 片 1.5K MTU 帧 | 整页补充 | page_alloc.c:4874-4879 |
| ⑧ page_pool alloc cache | 1 页 | **128 页 = 512KB**/RX 队列 | 空时补 64 页 | include/net/page_pool/types.h:50-51 |
| ⑨ page_pool ptr_ring | 1 页 | **默认 1024 项 = 4MB**，上限 32768 项 = 128MB | 跨 CPU 归还 | page_pool.c:194, 213, 262 |
| ⑩ IOVA（SMMU） | 1 页映射 | 由 SMMU 地址窗口/共享外设决定，**与 RAM 无关** | — | page_pool.c:470 |

### 算例：4GB 内存的 Qualcomm 设备

```
min_free_kbytes = 4 × √(4G/1K) = 4 × 2048 ≈ 8MB   （page_alloc.c:6124-6125）
├─ GFP_ATOMIC order-0 可借：5/8 × 8MB ≈ 5MB
└─ high-atomic 预留上限：1% × 4GB = 40MB（需历史高阶流量积累才有）

池④ PCP：zone 1M 页 → batch=127 → high≈762 页 ≈ 3MB/CPU
池① 16KB/CPU │ 池⑦ 32KB/CPU │ 池⑧ 512KB/队列 │ 池⑨ 4MB/pool
```

**含义**：4GB 设备上即使 `MemAvailable` 显示 2GB，RX 路径真正能瞬时动用的供给 = 各 CPU 的 ①④⑥⑦⑧⑨ 存量 + 池⑤ free 超过 min 的部分 + 约 5MB 可借预留——**典型总量只有几十 MB 量级**。其中任何一级先断供（池⑤只剩 MOVABLE/CMA、或 order-3 连续块为零），分配即失败。这就是"MemAvailable 充足却分配失败"在数字上的完整解释。

## 四、失败点速查表

| 函数/位置 | 分配对象 | 失败条件 | 触发场景 |
|---|---|---|---|
| `napi_skb_cache_get` skbuff.c:353 | skb 结构体 | bulk GFP_ATOMIC slab 分配失败 | slab 无空闲对象 + 低内存 |
| `kmem_cache_alloc_node` skbuff.c:668 | skb 结构体 | 节点 slab 扩页失败 | NUMA 节点压力 |
| `page_frag_alloc_1k` skbuff.c:249 | 小帧数据区 | order-0 GFP_ATOMIC 失败 | 低内存 |
| `__page_frag_cache_refill` page_alloc.c:4877 | 数据区（≤PAGE_SIZE） | order-3 高阶碎片化 → order-0 也失败 | **碎片化**、低内存 |
| `kmalloc_reserve` skbuff.c:609 | 大数据区（jumbo） | 高阶 kmalloc 失败 | **jumbo frame + 碎片化** |
| `__page_pool_alloc_pages_slow` page_pool.c:538 | pool 页 | bulk/高阶 alloc 失败 | 低内存、碎片化、节点压力 |
| `page_pool_dma_map` page_pool.c:470 | DMA 映射 | dma_mapping_error | IOMMU IOVA 耗尽 |
| `build_skb` skbuff.c:475 | skb 结构体 | slab GFP_ATOMIC 失败 | 低内存（驱动 buffer 已分配，白忙一场） |

## 五、观测与排查手段

- `/proc/net/softnet_stat`：第 2 列 `dropped`、第 3 列 `time_squeeze`（CPU 十六进制）。
- `ethtool -S <dev>`：`rx_dropped`、`rx_alloc_failed`、`alloc_fail`、`rx_missed_errors`、`pp_alloc_fail`（驱动相关）。
- page_pool 状态：`ethtool -S` 里的 `page_pool` 计数或 netlink `PAGE_POOL` 接口的 `alloc_fail`。
- 内核内存侧：`/proc/buddyinfo` 看高阶空闲块（碎片化），`/proc/zoneinfo` 看 watermark 与 `allocstall*`，`/proc/vmstat` 的 `allocfail`。
- tracepoint：`kmem:mm_page_alloc` 失败、`page_pool:page_pool_state_*`、`skb:kfree_skb`（dropwatch 可定位丢包点）。

## 结论

在 6.12.6 的 NAPI 中，sk_buff 分配失败几乎从不源于"内存真的耗尽"，绝大多数是：

1. GFP_ATOMIC 下低水位快速失败；
2. order-3 页 / jumbo 线性 buffer 的碎片化；
3. page_pool 的 NUMA 节点级压力或 DMA 映射失败。

且因为 `__GFP_NOWARN`，这些失败在内核日志中完全静默，必须靠计数器和 tracepoint 观测。
