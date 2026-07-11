# Android 系统架构

## Driver 架构

### 核心问题：Android 用 Linux 内核，驱动为什么还这么复杂？

Android 底层确实是 Linux 内核，基础硬件驱动（GPIO、I2C、显示控制器、触摸屏、WiFi、蓝牙、SD 卡等）都在内核中。但 Android 在标准 Linux 之上多加了一层**用户态 HAL**，这是驱动问题复杂化的根源。

### 驱动分层结构

```
应用层
  ↓ Android Framework API
用户态 HAL (vendor 分区)    ← 闭源二进制 (.so)，来自 SoC 厂商
  ↓ ioctl / /dev/ 节点
内核驱动 (boot.img)         ← 开源或半开源，ROM 维护者从 CAF 内核源码编译
  ↓
硬件
```

#### 1. 内核驱动层 (boot.img)

- 位于 Linux 内核中，随 boot image 刷入
- 包含显示、触摸、WiFi、蓝牙、DMA、时钟、电源管理等基础驱动
- 这些驱动多数不是 mainline Linux 标准驱动，而是 **SoC 厂商（Qualcomm、MediaTek）树外驱动**
- 依赖特定版本的内核（如 Android common kernel 5.10 + Qualcomm CAF patches）
- ROM 维护者需要从厂商发布的内核源码编译，配合正确的设备树（Device Tree）配置

#### 2. 用户态 HAL 层 (vendor 分区)

Android 在标准 Linux 之上增加了这一层，原因是：

> **Linux 内核是 GPL 协议** — 如果核心算法放在内核中，SoC 厂商就必须开源。
> 放在用户态 HAL 中（作为 `.so` 二进制），可以保持闭源。

典型闭源 HAL 包括：

| 组件 | 说明 | 影响 |
|------|------|------|
| 相机 ISP 管线 | 图像信号处理算法 | custom ROM 拍照质量通常不如原生 |
| 指纹处理 | 指纹录入、比对算法 | 不可替换 |
| 音频 DSP 调校 | 音频编解码、音效处理 | 可能影响通话质量 |
| 视频解码 DRM | 版权保护内容解码 | Widevine L1 可能丢失 |
| AI/NPU 加速 | AI 推理硬件加速 | 联发科、麒麟基本不可用 |

### Project Treble 的影响 (Android 8.0+)

Treble 架构将 system 分区和 vendor 分区解耦：

```
// Treble 之前
system (ROM) ←→ vendor HAL (紧耦合)
→ 升级系统必须厂商更新 vendor 驱动

// Treble 之后
system (ROM) ←→ HIDL / AIDL 接口 ←→ vendor 实现 (分离)
→ 刷 custom ROM 只需替换 system 分区，vendor 分区可复用
```

这使得自定义 ROM 的工作量大幅降低：**vendor 分区直接原封不动搬过来用**，ROM 只替换 system 分区。

### 刷 custom ROM 时驱动的来源

- **内核驱动**：ROM 维护者从厂商发布的 kernel source + CAF patches 编译
- **用户态 HAL**：从原生系统中完整提取 vendor 分区，直接复用
- **不再需要"安装驱动"** — 刷机前备份 vendor 分区，刷完后保持或恢复即可

### 内核版本升级难的根本原因

一台手机难以升级主线内核版本，不是因为内核驱动本身难写，而是：

**新内核的接口改变 → 旧厂商闭源 HAL 调用旧内核接口 → 崩溃**

vendor 分区里的 `.so` 文件通过 `ioctl` 或设备节点与内核交互，如果内核版本升级后这些接口发生变化，闭源 HAL 就会失效。而没有厂商配合重新编译，这些 HAL blob 是无法修改的。

### 哪些手机/平台适合刷机

| 平台 | 为什么适合/不适合 |
|------|------------------|
| **Qualcomm (小米、一加)** | HAL 架构清晰，接口规范；CAF 内核补丁完善；社区支持好 |
| **MediaTek (联发科)** | HAL 依赖性强，闭源 blob 耦合度高；内核源码发布不完整 |
| **HiSilicon (华为麒麟)** | 闭源程度极高；bootloader 锁定；社区几乎无法适配 |
| **Exynos (三星)** | 部分开源但文档不足；bootloader 限制多 |

### 总结

Android 的驱动问题本质上是**开放源码的内核 + 闭源的用户态驱动**这一混合架构导致的。内核驱动（开源）可以由社区维护，但用户态 HAL（闭源）必须依赖厂商提供。刷 custom ROM 后的驱动不是"获取和安装"的，而是**从原厂系统原封不动搬过来复用的**。
