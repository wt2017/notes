# Android 软件架构分析（基于本机 AOSP 16 / Baklava 源码树）

> 分析对象：`/home/uu26wyou/projects/aosp`，本机已编译并在 Cuttlefish 运行。
> 树内实测：Android 16 "Baklava"，`ro.build.id=BP4A.251205.006`，SDK 36，eng/userdebug；
> 产品 `aosp_cf_x86_64_phone`，产物 `out/target/product/vsoc_x86_64/`。
> 内核：树内无源码（`kernel/` 只有 `configs/ prebuilts/ tests/`），GKI 预编译 + vendor 模块模型，
> 内核定制由 `kernel/configs/*/android-6.12/android-base.config` fragment 表达
> （`CONFIG_ANDROID_BINDERFS=y`、`# CONFIG_ANDROID_LOW_MEMORY_KILLER is not set` 等）。

---

## 第 1 部分：从 APK / App 到 Linux syscall 的逐层梳理

现代 Android 几乎所有功能都落在「App 进程 ↔ 系统进程」之间，进程间唯一主干道是 **Binder**。
整体是一张跨进程的栈：

```
 ┌────────────────────────────────────────────────────────────┐
 │ ① APK / App 进程            (uid≥10000 的隔离 Linux 进程)   │
 │    ActivityThread main · ART (dex/oat) · android.app API    │
 ├────────────────────────────────────────────────────────────┤
 │ ② Java Framework (android.jar)  IBinder proxy + Parcel     │
 ├────────────────────────────────────────────────────────────┤
 │ ③ Binder JNI      frameworks/base/core/jni/android_util_*  │
 ├────────────────────────────────────────────────────────────┤
 │ ④ libbinder(C++)  ProcessState / IPCThreadState::transact  │
 ├────────────────────────────────────────────────────────────┤
 │ ⑤ [ioctl(fd, BINDER_WRITE_READ)  → 真正 syscall 在这]      │
 │    /dev/binder  (binderfs)  ←─ 内核 binder 驱动(在 GKI)     │
 ├────────────────────────────────────────────────────────────┤
 │ ⑥ 服务端进程(如 system_server) 同栈反向消费 → 业务逻辑       │
 ├────────────────────────────────────────────────────────────┤
 │ ⑦ native daemon / HAL (netd/audioserver/SurfaceFlinger…)   │
 ├────────────────────────────────────────────────────────────┤
 │ ⑧ bionic libc 封装 → syscall → vDSO / 内核                  │
 └────────────────────────────────────────────────────────────┘
```

### ① App 进程诞生：zygote fork，不是 fork+exec

- zygote 由 init 拉起：`system/core/rootdir/init.zygote64.rc:1`
  `service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote ...`，监听 abstract socket `zygote`。
- 启动 App：`ActivityManagerService`（`frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java`）
  → `ProcessList.startProcess()` → `android.os.Process.start()`（`frameworks/base/core/java/android/os/Process.java`）
  → **`ZygoteProcess`** 经 socket 请求（`frameworks/base/core/java/android/os/ZygoteProcess.java`）。
- zygote 端：`ZygoteServer` / `ZygoteConnection`（`frameworks/base/core/java/com/android/internal/os/ZygoteConnection.java`），
  真正 `fork()` 在 JNI：`frameworks/base/core/jni/com_android_internal_os_Zygote.cpp`。
- fork 后子进程做三件隔离动作：`setuid/gid` 到该 App 的 uid、装 **seccomp BPF 过滤器**、`setexeccon` 切 SELinux 域。
- 子进程运行 `ActivityThread.main()`（`frameworks/base/core/java/android/app/ActivityThread.java`）= App 主线程 + Binder 回调入口。

> 代价与收益：App 进程只能是 ART 托管的 APK 代码（不能是任意可执行文件），但共享 zygote 已 mmap 的框架类/只读页（COW），
> 冷启动与内存开销都大幅降低。

### ②~④ 一次 Binder 调用穿过的层

| 层 | 代码位置 | 职责 |
|---|---|---|
| Java proxy | App 进程 `android.os.BinderProxy.transact()` | 方法名 + 参数 → `Parcel` |
| JNI | `frameworks/base/core/jni/android_util_Binder.cpp` | Java ↔ native `Parcel` 互转 |
| native libbinder | `frameworks/native/libs/binder/IPCThreadState.cpp` | 组 `binder_transaction_data`，线程进 `binder_loop` |
| **syscall** | `ioctl(fd, BINDER_WRITE_READ)` | 进内核 binder 驱动 |
| 内核驱动 | 不在本树（GKI prebuilt） | 复制数据、定位目标、唤醒服务端线程、做一次 SELinux 挂钩检查 |
| 服务端反方向 | 服务端 `IPCThreadState` 线程池 → `Binder.execTransact` → Java 服务实现 | 执行业务 |

设备节点来自 **binderfs**，`system/core/rootdir/init.rc:221-235`：

```
mkdir /dev/binderfs
mount binder binder /dev/binderfs stats=global
symlink /dev/binderfs/binder    /dev/binder
symlink /dev/binderfs/hwbinder  /dev/hwbinder      ← HIDL/AIDL HAL 用
symlink /dev/binderfs/vndbinder /dev/vndbinder     ← vendor 进程用
```

同一内核驱动隔离出三套上下文：framework（binder）、HAL（hwbinder）、vendor（vndbinder）。
`frameworks/native/libs/binder/ProcessState.cpp` 按进程角色选择节点并 `ioctl` 握手（`BINDER_VERSION` / `BINDER_SET_MAX_THREADS`）。

### ⑤ ART / AOT 与「App 能直呼 syscall 吗」

- App 字节码由 **ART** 承载（插在 zygote fork 之后、`ActivityThread.main` 之前）。启动图像**离线 AOT**：
  `out/target/product/vsoc_x86_64/system/framework/x86/boot.oat/.art`、各 APK 的 `.odex` 均在 out 中（`dex2oat` 构建期编译）。
- App 内**基本不直接 syscall**：Java/NDK 代码全经 bionic 封装；且每 App 进程带 **seccomp BPF 白名单**
  （见第 3 部分 §10），裸调会被过滤拒掉。时钟类在支持架构走 **vDSO**。

### ⑥ system_server：系统服务的"宿"

- `SystemServer.main`（`frameworks/base/services/java/com/android/server/SystemServer.java`）按
  `startBootstrapServices / startCoreServices / startOtherServices` 装配上百个 `SystemService`，
  各自 `ServiceManager.addService("package"|"activity"|...)` 注册到 **servicemanager**（`frameworks/native/cmds/servicemanager/`）。
- 整个系统 = "一堆经 Binder 相连的 Linux 进程"，system_server 是中枢。

---

## 第 2 部分：典型 Framework 一览与内部架构

> 大前提：**HIDL 时代 → stable AIDL 时代**。本树几乎所有子系统处于"HIDL 旧目录 与 `aidl/` 新目录并存"的迁移态，
> 服务端软件常带双 wrapper（如 `frameworks/native/services/sensorservice/{Hidl,Aidl}SensorHalWrapper.cpp`、
> SurfaceFlinger `DisplayHardware/{Hidl,Aidl}ComposerHal.cpp`）。新接口一律 AIDL。
> 部分系统服务已 mainline 化，源码迁至 `packages/modules/*`（网络/WiFi/蓝牙/调度等）。

### 2.1 应用框架核心：AMS / PMS / Zygote

- `ActivityManagerService`（进程/任务/生命周期）、`PackageManagerService`（扫描/解析/签名/权限）、zygote 族。
- `frameworks/base/services/core/java/com/android/server/{am,pm,wm}/`，每 manager 拆多类
  （`am/`：`ProcessList` `ActiveServices` `ActivityStack` `BatteryStatsService`…）。
- 与发行版对照：PMS = 包数据库（类比 rpm/dpkg）+ 安装/更新引擎；AMS 兼任 init 的进程生成器 + OOM 分级器。
  安装时真实文件操作由 **installd**（root daemon）执行。

### 2.2 图形与显示合成（最典型、最能串起问题 1 的栈）

- App 侧：`View/Surface` 画进 `BufferQueue`（`frameworks/base/libs/hwui/` 的 HWUI RenderThread，OpenGL/Vulkan 渲染）。
- **SurfaceFlinger**（native daemon，`frameworks/native/services/surfaceflinger/`），内部子模块：
  - `CompositionEngine/` 合成引擎（把各 App layer + 状态栏合成一帧）
  - `Scheduler/` + `VSyncDispatch` —— VSync/帧率调度/FrameTimeline（决定何时取 buffer 上屏）
  - `DisplayHardware/` `HWC2.cpp` —— 对接 HWC（显示控制器）
- HAL：`hardware/interfaces/graphics/*`（`mapper`/`allocator` = **Gralloc** 分配 buffer；`composer3` AIDL = HWC）。
- **Buffer 跨进程共享不靠 binder 拷贝**：Gralloc 分配的是 **dma-buf**，binder 只传 fd，各进程 mmap 同一块物理内存，
  用 **sync_file/fence** 同步。

### 2.3 Window / Input

- Java：`com.android.server.wm.*`（WMS，`frameworks/base/services/core/java/com/android/server/wm/`）+ `InputManagerService`（`server/input/`）。
- **Input 管道**（`frameworks/native/services/inputflinger/`）拆两半：
  - `reader/EventHub.cpp` 读 `/dev/input/*`（内核 evdev）→ `InputReader` 归一化
  - `dispatcher/InputDispatcher.cpp` 按窗口焦点/`InputChannel` 投递到目标 App
- 一次触摸：内核 evdev → EventHub → Reader → Dispatcher → App InputChannel → `ViewRootImpl`/`View.dispatchTouchEvent`。

### 2.4 音频（最"教科书"的分层）

- App API：`AudioTrack/AudioRecord/SoundPool/MediaPlayer`（`frameworks/base/media/java/android/media/`）、NDK 的 **AAudio**。
- Java 策略：`AudioService`（`frameworks/base/services/core/java/com/android/server/audio/`：音量/焦点/设备切换策略）。
- native 主进程 **audioserver**（`frameworks/av/media/audioserver/main_audioserver.cpp`），实体在 `frameworks/av/services/`：
  - **AudioFlinger**（`audioflinger/`）混音引擎：按输出设备维护一组音频线程（混音/直通 offload/fast mixer）。
  - **AudioPolicyService**（`audiopolicy/`）路由决策：哪个输出设备、用哪个 HAL module/stream。
- 客户端 `frameworks/av/media/libaudioclient/`；HAL：`hardware/interfaces/audio/`（HIDL 2.0~7.1 与 `audio/aidl/.../core` 并存）。

### 2.5 视频 / 媒体（Codec2）

- API：`MediaPlayer / MediaCodec / MediaExtractor / MediaMuxer`（`frameworks/base/media/java/android/media/`）；ExoPlayer 是普通应用库（`external/exoplayer/`）。
- 核心：**Stagefright + Codec2**，`frameworks/av/media/libstagefright/`（`MediaCodec.cpp` 状态机、`ACodec.cpp`、`MediaExtractor*`）；
  组件模型在 `frameworks/av/media/codec2/`：`core/` 定义 `C2Component`；`client/` `sfplugin/` 桥到 MediaCodec；
  `components/` 每软编解码器一目录；硬件 codec 走 C2 HAL（`hardware/interfaces/media/c2/aidl`）；
  软 codec 由 `packages/modules/Media/` APEX 提供（可独立升级）。

### 2.6 相机

- **CameraService**（`frameworks/av/services/camera/libcameraservice/`）：`CameraProviderManager` 发现 provider →
  按 provider/device 多层 binder 模型（`CameraDeviceClient`），走 **ICameraProvider / ICameraDevice AIDL**（`hardware/interfaces/camera/{provider,device}/aidl`）。
- App 侧 camera2 API（`frameworks/base/core/java/android/hardware/camera2/`）；帧 buffer 走 Gralloc/dma-buf 直通。
- Cuttlefish 用虚拟相机（`libcameraservice/virtualcamera` + vsock HAL）。

### 2.7 网络

- 已 mainline 化：**ConnectivityService** 真身在 `packages/modules/Connectivity/service/src/com/android/server/ConnectivityService.java`。
- native daemon **netd**（`system/netd/server/`）内部按能力拆控制器：`NetworkController`（多网络路由/fwmark）、
  `FirewallController`、`BandwidthController`、`IptablesRestoreController`、`FwmarkServer`（给 socket 打标实现 per-network 路由）；
  上层 AIDL `NetdNativeAidlService`，下层 iptables/nftables + netlink。
- 每网络一条 `DnsResolver`；**WiFi** 模块化于 `packages/modules/Wifi/`（`WifiService` → `hardware/interfaces/wifi/aidl`）。
  **NetworkStack** 是特权普通 APK（`packages/modules/NetworkStack/`）。

### 2.8 传感器

- **sensorservice**（`frameworks/native/services/sensorservice/`）：`SensorDevice` 读 HAL + 传感器融合（`fusion/`：
  加速度+陀螺+磁力合成旋转向量/方向等虚拟传感器）。
- HAL：`hardware/interfaces/sensors/`（AIDL `ISensors`）；事件经 Binder 定向给 app 的 `SensorEventConnection`。

### 2.9 存储 / 文件 / 加密

- **vold**（`system/vold/`，root daemon）：卷管理、`fscrypt`（`FsCrypt.cpp`）、binder 服务 `VoldNativeService`。
- App 的"SD 卡"由 **FUSE** 提供（`system/core/sdcard/`、`AppFuse`），配 scoped storage 按 app 授权暴露文件（不给挂载点权限）。
- 密钥：**Keystore2**（`system/security/keystore2/`，Rust+C）→ keymint HAL（`hardware/interfaces/security/keymint/aidl`，TEE 内）。

### 2.10 电源 / Doze / 省电

- `PowerManagerService`（`frameworks/base/services/core/java/com/android/server/power/`：`WakelockTracer`、`PowerGroup`）+ display/brightness。
- Doze / AppStandby / JobScheduler 已整包进 jobscheduler APEX（`packages/modules/Scheduling/`、`frameworks/base/apex/jobscheduler/`：`DeviceIdleController`、`AppStandbyController`）。
- 性能提示到内核：`hardware/interfaces/power/aidl`。

### 2.11 其它常用（点到为止）

| 领域 | Java/系统服务 | native/daemon | HAL |
|---|---|---|---|
| 定位 | `LocationManagerService`（`server/location/`） | — | `hardware/interfaces/gnss/aidl` |
| 蓝牙 | `packages/modules/Bluetooth/`（含 Rust 栈） | — | `bluetooth/aidl/IBluetoothHci` |
| 电话/RIL | `packages/services/Telephony/` | rild | `radio/aidl` |
| NFC/UWB | `packages/modules/Nfc|Uwb/` | | `interfaces/nfc` |
| 通知 | `NotificationManagerService`（`server/notification/`） | | |
| 无障碍 | `services/accessibility/java/com/android/server/accessibility` | | |
| 输入法 | `InputMethodManagerService`（`server/inputmethod/`） | | |
| 密钥身份 | Keystore2 | | `keymint/aidl`（TEE） |

---

## 第 3 部分：Android ↔ Linux 的"粘合层"，以及与 Fedora/Ubuntu 的差异

Android 内核是主流 Linux + 少量 GKI 定制；真正特殊的是**内核之上的用户态系统架构决策**：
Android 故意绕开发行版那套通用 Linux 约定（systemd / glibc / D-Bus / sysctl / NSS / 可写根分区），自建了一套。
逐机制对照如下。

### §1 进程模型：zygote fork + per-app 沙箱（vs fork+exec）

- App uid = `userid*100000 + appid`；普通 app ≥ 10000，isolated（沙箱/webview）99000+
  （`frameworks/base/core/java/android/os/UserHandle.java`、`Process.java`）。
- 发行版：任意可执行文件 + 用户；Android：App = ART 托管代码 + 专属 uid + 专属 SELinux domain + seccomp，全在 zygote fork 一刻刻好。

### §2 IPC：Binder 内核驱动（vs socket / D-Bus）

- 驱动在 GKI；用户态在 `frameworks/native/libs/binder/`。服务经 **servicemanager** 注册/查找；binderfs 分 `binder/hwbinder/vndbinder`。
- binder 每次调用做一次 SELinux 钩子检查——IPC 被强制类型化。发行版 D-Bus/local socket 靠连接者身份授权，远不如 binder 严格。

### §3 init + .rc 声明式触发器（vs systemd）

- `system/core/init/`（`init.cpp`、`service_parser.cpp`、`property_service.cpp`、`ueventd`）；`.rc` 按事件/属性触发、跨分区 import
  （`/system|vendor/etc/init/hw/`）。
- 不用 systemd：Android 启动目标单一、分区只读、无系统管理员、需可预测启动，且 OTA 要能整体覆盖分区；
  systemd 假设可写根 + 并行依赖图 + 一套守护日志的通用发行版世界观。

### §4 bionic libc / 专有 linker（vs glibc）

- 无 `LD_PRELOAD`、无 glibc symbol versioning；专有 linker（`system/core/linker` + `/system/etc/ld.config*.txt` 白名单）+ fortify + 专有 TLS。
- App 只用 **NDK 稳定 ABI** → 系统库怎么换都不破坏 App。
- syscall 封装由 `bionic/libc/SYSCALLS.txt` + `tools/gensyscalls.py` 按架构**生成**（同一张表同时产出汇编 stub 与 seccomp 白名单）；
  `__NR_*` 来自内核 uapi 拷入（`bionic/libc/kernel/uapi/.../unistd_64.h`）。

### §5 强制 SELinux Type Enforcement（vs 可选加固）

- 策略树 `system/sepolicy/{public,private,vendor}`；**`seapp_contexts`**（`private/`）固化"app→domain"
  （untrusted_app / platform_app / isolated_app），zygote fork 用 `setexeccon` 切域。
- **`neverallow`** = 白名单收紧，App 域明令禁止 setuid / binder 越权等。Fedora 的 SELinux 只是可选层，Android 每进程默认强制。

### §6 Property 系统（vs /etc 配置 + /proc/sys）

- 全局键值共享内存 `/dev/__properties__`（init `property_service` 建）；**读** = mmap 共享内存零拷贝；**写**（`setprop`）经 socket 给 init 校验。
- `ro.*` 只读，`ctl.*`（重启服务）被 sepolicy 收紧。发行版各服务读 /etc/xxx.conf、改 /proc/sys；Android 全组件读同一 property 命名空间。

### §7 只读分区 + OTA（vs 可写 /usr + 包管理）

- `system/vendor/product/odm/boot/vendor_boot/init_boot` 各自只读；动态分区 `super`（`system/core/fs_mgr/liblp/`）；
  `adb remount` 走 overlayfs（`system/core/fs_mgr/fs_mgr_remount.cpp`）。**GKI** 把内核与 vendor 驱动解耦。
- 升级模型 = 整分区镜像 OTA，不是 `dnf update` 就地改写 /usr。系统 / 厂商 / App 三个升级域互不干扰（Treble / APEX）。

### §8 HAL + VINTF（Treble 边界，vs 驱动随便装）

- 接口树 `hardware/interfaces/*`（新 AIDL）；厂商实现独立分区。**VINTF**：设备 manifest vs `compatibility_matrix`
  （`hardware/interfaces/compatibility_matrices/`）boot 时校验 → 框架与厂商可独立演进。
- 发行版硬件 = 内核驱动 + udev + 用户库，没有"接口层 + 兼容矩阵"这种强契约。

### §9 cgroup v2 + 用户态 lmkd（内存/性能调度粘合）

- `system/core/libprocessgroup/`：`task_profiles.json` / `cgroups.json` 把 App 分到 `top-app/foreground/background` cpuset，
  用 `cpu.uclamp.min/max`（**schedtune 已移除**）做容量控制。
- 内核 LMK 已关闭（config 实测 `# CONFIG_ANDROID_LOW_MEMORY_KILLER is not set`），低内存回收由 **lmkd**
  （`system/memory/lmkd/`）读 **PSI** 用户态裁决杀后台 App。发行版用内核 OOM-killer/kswapd，不按"前台体验"精确调杀。

### §10 seccomp：每个 App 进程默认 syscall 白名单

- BPF 由 `bionic/libc/seccomp/seccomp_policy.cpp` 按 arch 生成（`x86_64_app_filter` 等），zygote fork 出 App child 时
  经 `frameworks/base/core/jni/com_android_internal_os_Zygote.cpp` 的 `set_app_seccomp_filter` 装入；
  媒体等另有 `out/target/product/vsoc_x86_64/system/etc/seccomp_policy/*.policy`。
- 发行版普通进程默认可用几乎全部 syscall；Android 的 untrusted App 一出生就被剪掉大量内核接口。

### §11 一句话本质差异

> Fedora/Ubuntu = 通用 Linux 用户态 + 包管理安装任意程序：任意可执行、任意 syscall、root/管理员模型、systemd、glibc、/etc。
> Android = 受管 App 农场：App 是沙箱租户（zygote 定制 fork + seccomp + SELinux + 专属 uid），系统能力经 Binder 中转、
> HAL 收口到厂商，系统以只读分区整体 OTA。内核还是那个 Linux 内核，但用户态契约几乎全部重写。

---

## 附：本构建（Cuttlefish / vsoc_x86_64）特有的粘合

- VMM 是 **crosvm**，guest 走 **virtio** + 少量 goldfish 传统模块（`goldfish_pipe/battery/address_space.ko` 随
  `kernel/prebuilts/common-modules/virtual-device/6.12/x86-64/` 提供）；不是 goldfish 整机模拟。
- guest 的"HAL 硬件"由 **vsock 协议代理到 host 进程**实现：`device/google/cuttlefish/guest/hals/{audio,camera,light,keymint,ril,...}`
  （如 `vsock_camera_*`）；宿主侧由 `device/google/cuttlefish/host/commands/{assemble_cvd,run_cvd,cvd,...}` 驱动。
- 即便没有真机 SoC，树内仍保有完整 HAL 边界、VINTF manifest（`device/google/cuttlefish/shared/config/manifest.xml`）
  与整套 framework —— 这让 Cuttlefish 成为研究这条完整调用链的理想对象。

---

## 取证依据（关键路径速查）

- 构建/版本：`out/target/product/vsoc_x86_64/system/build.prop`、`build_fingerprint-aosp_cf_x86_64_phone.txt`
- zygote：`system/core/rootdir/init.zygote64.rc`、`frameworks/base/cmds/app_process/app_main.cpp`、
  `frameworks/base/core/java/com/android/internal/os/ZygoteInit.java`、`ZygoteConnection.java`
- fork JNI：`frameworks/base/core/jni/com_android_internal_os_Zygote.cpp`
- binderfs 节点：`system/core/rootdir/init.rc:221-235`
- libbinder：`frameworks/native/libs/binder/{ProcessState,IPCThreadState,Binder}.cpp`
- servicemanager：`frameworks/native/cmds/servicemanager/`
- 系统服务装配：`frameworks/base/services/java/com/android/server/SystemServer.java`
- ART/AOT：`art/`、out 下 `system/framework/x86/boot.oat`
- 网络栈：`packages/modules/Connectivity/`、`system/netd/server/`、`packages/modules/Wifi/`
- 媒体/音频：`frameworks/av/media/`、`frameworks/av/services/{audioflinger,audiopolicy}`、`frameworks/av/media/codec2/`
- 图形：`frameworks/native/services/surfaceflinger/`、`hardware/interfaces/graphics/*`
- 输入：`frameworks/native/services/inputflinger/`
- init/property/seccomp/SELinux：`system/core/init/`、`system/core/rootdir/`、`bionic/libc/{SYSCALLS.txt,seccomp}`、`system/sepolicy/`
- 分区/挂载：`system/core/fs_mgr/`（liblp / overlayfs remount）、`system/core/rootdir/init.rc`
- 性能/回收：`system/core/libprocessgroup/`、`system/memory/lmkd/`
