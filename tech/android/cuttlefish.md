# AOSP + Cuttlefish 完整指南

## 目录

- [AOSP + Cuttlefish 完整指南](#aosp--cuttlefish-完整指南)
  - [目录](#目录)
  - [概述](#概述)
  - [⚡ 核心概念：Host Cuttlefish vs VM Cuttlefish](#-核心概念host-cuttlefish-vs-vm-cuttlefish)
  - [一、环境要求](#一环境要求)
  - [二、安装宿主依赖](#二安装宿主依赖)
    - [1. 基础工具](#1-基础工具)
    - [2. 安装 repo 命令](#2-安装-repo-命令)
    - [3. 安装 Cuttlefish 宿主包](#3-安装-cuttlefish-宿主包)
  - [三、下载 AOSP 源码](#三下载-aosp-源码)
  - [四、编译 AOSP + Cuttlefish 镜像](#四编译-aosp--cuttlefish-镜像)
  - [五、启动 Cuttlefish](#五启动-cuttlefish)
    - [最简启动：三条命令](#最简启动三条命令)
    - [启动失败？本机必做的前置修复：放宽 crosvm 的 madvise seccomp](#启动失败本机必做的前置修复放宽-crosvm-的-madvise-seccomp)
    - [扩展：自定义 CPU / 内存 / 分辨率 / 多设备](#扩展自定义-cpu--内存--分辨率--多设备)
    - [扩展：停止与清理](#扩展停止与清理)
    - [cvd 运行机制：客户端-服务端-实例组](#cvd-运行机制客户端-服务端-实例组)
    - [实例数据存哪？能否自定义？](#实例数据存哪能否自定义)
    - [cvd 常用命令速查](#cvd-常用命令速查)
  - [六、开发-修改-验证闭环](#六开发-修改-验证闭环)
    - [全量编译](#全量编译)
    - [增量编译（只改一个模块时快得多）](#增量编译只改一个模块时快得多)
    - [典型修改实验](#典型修改实验)
  - [七、使用预编译镜像（跳过编译，快速上手）](#七使用预编译镜像跳过编译快速上手)
  - [八、AOSP 源码结构（关键目录）](#八aosp-源码结构关键目录)
  - [九、进阶技巧](#九进阶技巧)
    - [adb 日志调试](#adb-日志调试)
    - [Android Studio for Platform 调试](#android-studio-for-platform-调试)
    - [自定义内核](#自定义内核)
  - [十、常见问题](#十常见问题)
    - [实战排错笔记（本机实测）](#实战排错笔记本机实测)
  - [十一、推荐学习路径](#十一推荐学习路径)

---

## 概述

```
PC (Linux)
  ├── AOSP 源码 (/home/you/aosp/)
  ├── Cuttlefish (安卓虚拟设备管理器)
  └── Android 虚拟设备
       ├── Web UI (https://localhost:8443)
       ├── adb (localhost:6520)
       └── 默认 root 权限
```

不需要任何手机硬件，在 PC 上完成从源码到运行的全部流程。

## ⚡ 核心概念：Host Cuttlefish vs VM Cuttlefish

这是初学者最容易混淆的地方。Cuttlefish 有两个角色，来源和作用完全不同：

| | **Host Cuttlefish（宿主工具）** | **VM Cuttlefish（虚拟机内的 Android）** |
|---|---|---|
| **装在** | 你的 Linux PC 上 | Android 虚拟机内部 |
| **作用** | 创建/管理 Android 虚拟机 | 被宿主运行的 Android 系统 |
| **代码来源** | GitHub `google/android-cuttlefish` | AOSP `device/google/cuttlefish/` |
| **安装方式** | `apt install cuttlefish-*` 或 `dpkg -i` | 编译 AOSP 产出系统镜像 |
| **产物** | `cvd`（新统一入口，`cvd start` 取代旧 `launch_cvd`）、`crosvm`、`run_cvd` 等 CLI 工具 | `boot.img`, `system.img`, `vendor.img` ... |
| **何时装** | **先装**，在下载 AOSP 之前 | **后装**，编译 AOSP 时产出 |

**打个比喻：**

```
VMware Workstation（装在你的 PC 上）       ← Host Cuttlefish（cvd / crosvm）
     ↓ 负责创建和管理
Ubuntu VM（虚拟机里的操作系统）            ← VM Cuttlefish（AOSP 编译出的 Android 镜像）
```

`cvd` 相当于 VMware Workstation 的命令行入口，AOSP 编译产物相当于 Ubuntu 的 `.iso` 镜像。**装了 host 包但没编译 AOSP，就等于光驱里没有光盘**——`cvd create` 会报错找不到镜像。

**镜像定位（新版 `cvd`）：** 不再靠 launcher 挨个目录猜，而是由 `lunch` 设置的两个环境变量直接定位：

```
$ANDROID_PRODUCT_OUT   设备镜像目录（boot.img / system.img 等）
$ANDROID_HOST_OUT      宿主工具目录（crosvm / run_cvd 等）
```

所以在编译过 AOSP 的终端里（已 `source build/envsetup.sh && lunch`）直接 `cvd create` 就能启动自己的编译产物。镜像在别处时，用 `cvd create --config_file` 里 `disk.default_build` 指定目录（见下文 §五）。

> 旧版 `launch_cvd` 按 `$ANDROID_PRODUCT_OUT → ~/cuttlefish → 当前目录 → 命令行参数` 的顺序找镜像；该历史行为已并入 `cvd`。

## 一、环境要求

| 项目 | 最低 | 推荐 |
|------|------|------|
| **CPU** | 8 核 x86_64 | 16 核+ |
| **内存** | 16 GB | 32 GB+ |
| **磁盘** | 300 GB 空余 | 500 GB+ SSD |
| **OS** | Ubuntu 22.04 | Ubuntu 24.04 LTS |
| **KVM** | `grep -c "vmx\|svm" /proc/cpuinfo` > 0 | 必须 |

**磁盘空间估算：**
```
AOSP 源码检出         ~60 GB
AOSP 编译产物         ~100 GB
Cuttlefish 镜像        ~5 GB
ccache 编译缓存        ~50 GB (可选)
总计                   ~200+ GB
```

---

## 二、安装宿主依赖

### 1. 基础工具

```bash
sudo apt update
sudo apt install -y git-core gnupg flex bison build-essential \
  zip curl zlib1g-dev libc6-dev-i386 libncurses5-dev \
  x11proto-core-dev libx11-dev lib32z1-dev libgl1-mesa-dev \
  libxml2-utils xsltproc unzip fontconfig python3 python3-pip

git config --global user.name "Your Name"
git config --global user.email "your@email.com"
```

### 2. 安装 repo 命令

```bash
mkdir -p ~/bin
curl -sS https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
export PATH=$PATH:~/bin   # 建议写入 ~/.bashrc
```

### 3. 安装 Cuttlefish 宿主包

```bash
sudo apt install -y git devscripts equivs config-package-dev \
  debhelper-compat golang curl

git clone https://github.com/google/android-cuttlefish
cd android-cuttlefish
tools/buildutils/build_packages.sh

sudo dpkg -i ./cuttlefish-base_*_*64.deb || sudo apt-get install -f
sudo dpkg -i ./cuttlefish-user_*_*64.deb || sudo apt-get install -f
sudo usermod -aG kvm,cvdnetwork,render $USER

# 重启使用户组生效
sudo reboot
```

重启后验证：
```bash
ls -l /dev/kvm       # 确认 KVM 可用
groups               # 应包含 kvm cvdnetwork render
```

---

## 三、下载 AOSP 源码

2026 年使用 `android-latest-release` 分支：

```bash
mkdir -p ~/aosp && cd ~/aosp

repo init --partial-clone --no-use-superproject \
  -b android-latest-release \
  -u https://android.googlesource.com/platform/manifest

repo sync -c -j$(nproc)
```

**网络：** 需要能访问 `android.googlesource.com`。如在中国大陆，配置代理：

```bash
export HTTP_PROXY=http://127.0.0.1:7890
export HTTPS_PROXY=http://127.0.0.1:7890
# 或用 proxychains repo sync -c -j8
```

**耗时参考：** 100Mbps ~1-2 小时，50Mbps ~2-4 小时

---

## 四、编译 AOSP + Cuttlefish 镜像

```bash
cd ~/aosp
source build/envsetup.sh

# 取消 selinux 安全设置
sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0 # 已经 persiste 到 /etc/sysctl.d/60-cuttlefish.conf

# 选择编译目标
# aosp_cf_x86_64_phone = Cuttlefish x86_64 虚拟手机
# trunk_staging = 开发分支
# userdebug = 带 root 的调试版本
lunch aosp_cf_x86_64_phone-trunk_staging-userdebug

# 设置 ccache 加速二次编译
export USE_CCACHE=1
export CCACHE_DIR=~/.ccache
ccache -M 50G

# 开始编译
m -j$(nproc)
```

**首次编译时间参考：**
| 配置 | 时间 |
|------|------|
| 16 核 / 32GB / SSD | ~1.5-2 小时 |
| 8 核 / 16GB / SSD | ~3-4 小时 |
| 4 核 / 16GB / HDD | ~8-10 小时（不推荐） |

编译产物在 `out/target/product/vsoc_x86_64/`。

---

## 五、启动 Cuttlefish

新版入口统一为 `cvd`（旧 `launch_cvd` 已并入 `cvd start`，且不再装到 PATH）。`cvd` 采用客户端-服务端模型，设备自动在后台运行，**不需要 `--daemon`**。

**核心就三条命令**：加载环境 → 选目标 → `cvd create`。其余（排错、自定义、原理、命令清单）都在后面按需查。

### 最简启动：三条命令

```bash
cd $HOME/projects/aosp
source build/envsetup.sh                            # ① 加载构建环境
lunch aosp_cf_x86_64_phone-trunk_staging-userdebug  # ② 选目标（Cuttlefish x86_64 虚拟手机）
cvd create                                          # ③ 建实例组并自动开机
```

第 ② 步不能省：`lunch` 导出 `ANDROID_HOST_OUT` / `ANDROID_PRODUCT_OUT`，`cvd create` 靠这两个变量去 `out/` 找宿主工具和镜像。**产物留在 `out/` 就行，不需要安装到 PATH**——两个变量都没设时 cvd 会一路回退到 `$HOME`，然后报 `'/home/<user>/bin/' does not contain any of '[cvd_internal_start, launch_cvd]'`。真不想 `lunch`，也可以显式传目录（见文末速查）。

`cvd create` 会自动拉起后台守护进程，把终端还给你。查看设备状态：

```bash
cvd fleet            # 列出实例组 / 设备
adb devices          # 应看到 localhost:6520  device
```

浏览器打开 `https://localhost:8443` 看到手机界面（实际端口以启动时 launcher 打印的 `Point your browser to …` 为准，本机当前为 8443）。

```bash
adb root && adb shell
whoami                # root
id                    # uid=0
getprop ro.build.type # userdebug
```

**之后每次重新编译后重启：**

```bash
m                    # 增量编译
cvd stop && cvd start
```

### 启动失败？本机必做的前置修复：放宽 crosvm 的 madvise seccomp

> 本机 7.0 内核上，若上面 `cvd create` 一启动就挂，就是这里。沙箱保持开启即可，不用关。

这台 7.0 内核机器**不先做这步**，`cvd create` 一启动就挂：

```
VIRTUAL_DEVICE_BOOT_FAILED
launcher.log: … failed to create a PCI root hub: failed to create proxy device:
              Failed to configure tube: Connection reset by peer
内核 audit:  type=1326 … comm="pcivirtio-gpu" sig=31 syscall=28   ← 28 = madvise
```

**根因**：crosvm 沙箱子进程的 seccomp 白名单把 `madvise` 限制在 5 种 advice，而新 glibc/GPU 路径用了别的 advice → 进程被 SIGSYS 秒杀 → 父进程只见 connection reset。与命名空间 / AppArmor 无关（那些是另一层报错）。

**修改**（放宽 crosvm 运行时读取的 4 个 x86_64 policy，全放行 madvise）：

```bash
S=$HOME/projects/aosp/out/host/linux-x86/usr/share/crosvm/x86_64-linux-gnu/seccomp
sed -i 's/^madvise: arg2 ==.*/madvise: 1/' \
  "$S"/gpu_device.policy "$S"/gpu_render_server.policy \
  "$S"/video_device.policy "$S"/wl_device.policy
```

> 改的是 **`out/` 产物**，`m`/`installclean` 会被还原；**永久化 = 第 1 步**：改源码 `device/google/cuttlefish_vmm/x86_64-linux-gnu/etc/seccomp/` 里同 4 个文件后 `m cvd-host_package`（本机暂未做）。改完即可按上面的启动步骤跑；若 `adb devices` 没自动出现 `127.0.0.1:6520`，手动 `adb connect 127.0.0.1:6520`。

### 扩展：自定义 CPU / 内存 / 分辨率 / 多设备

新版参数走 JSON 配置文件，先写好再 `cvd create --config_file`（在已 `lunch` 的同一终端里执行）：

```bash
cat > cf.json <<EOF
{
  "instances": [
    {
      "vm": { "cpus": 4, "memory_mb": 4096 },
      "graphics": { "displays": [ { "width": 1080, "height": 2340, "dpi": 420 } ] },
      "disk": { "default_build": "$HOME/projects/aosp/out/target/product/vsoc_x86_64" }
    }
  ]
}
EOF
cvd create --config_file=cf.json
```

更多字段（WiFi、GPU、多显示器等）用 `cvd help create`、`cvd help start` 查看。

**多设备：** 在 JSON 的 `instances` 数组里加多个条目即可，同一实例组内管理，第二个实例的 adb/显示端口递增。

**只想建好先不开机：** `cvd create --start=false`（等价 `--nostart`），要跑时再 `cvd start`。这样 `cvd ps` / `cvd fleet` 里会看到一条 `Stopped` 的实例——这两个命令读的是实例数据库，停机实例照样列出来（`cvd status` 才会去连 launcher，停机时会超时）。

### 扩展：停止与清理

```bash
cvd stop              # 关机但保留数据（之后可再 cvd start）
cvd start             # 重新开机（可反复，状态保留）
cvd restart           # 重启
cvd remove            # 彻底删除实例组（日志、虚拟磁盘一并删除）
cvd reset             # 兜底：杀掉所有 cvd 进程、清理资源
```

---

### cvd 运行机制：客户端-服务端-实例组

`cvd create` 不是"一条命令起个虚拟机"，而是三层分工，真正 boot Android 的另有其"人"：

```
你敲的 cvd create / start / stop      ← 客户端命令（本身不跑虚拟机）
        │ gRPC
cvd_server 守护进程                   ← 分配组 id/端口/运行目录，维护实例数据库
        │
实例组 cvd_1  →  /var/tmp/cvd/<uid>/<group_id>/
   ├─ artifacts/0            → 软链 AOSP 镜像目录（不复制、不占这里空间）
   ├─ artifacts/host_tools   → 软链 AOSP 宿主工具目录
   └─ home/cuttlefish/instances/cvd-1/
        ├─ crosvm      ← 真正的虚拟机进程(KVM)，boot Android 的是它
        ├─ run_cvd     ← 这个实例的"总管"，拉起并监视下面的子进程
        ├─ modem_simulator / netsimd   模拟蜂窝 / WiFi·蓝牙
        ├─ secure_env                 KeyMint 等安全服务
        ├─ webRTC                     屏幕 + 键盘鼠标 → 浏览器
        └─ cf_vhost_user_input / console_forwarder   输入 / 串口
```

类比：`cvd` 之于 `crosvm` ≈ `docker` 之于 `runc`——CLI 管生命周期与资源分配，后台有个持久守护进程做"台账"。因此 `cvd` 命令在哪敲都行（只要环境变量对），多实例就是同一实例组里 `cvd_1/1`、`cvd_1/2`… 统一由服务端管理。

### 实例数据存哪？能否自定义？

| 命令 | 对数据的态度 |
|---|---|
| `cvd create` | 建组 + 分配资源 + 自动开机 |
| `cvd stop` / `cvd start` | 关机保留数据 / 重新开机（可反复，状态保留） |
| `cvd remove` | 彻底删组（虚拟磁盘、日志一起删） |
| `cvd reset` | 兜底清场（所有组 + 孤儿进程 + 数据库） |

运行目录固定为 `/var/tmp/cvd/<uid>/`（按登录 uid 分，如 `1000`）：

```
/var/tmp/cvd/
└── 1000/
    ├── instance_database.binpb   ← 服务端台账（上次失败的 Boot Failed 组就残留在这）
    ├── lock/                     ← 实例锁
    └── <group_id>/               ← 实例组
        └── home/cuttlefish/instances/cvd-1/   ← 可写磁盘(qcow2) + 日志，真正增长的地方
```

选 `/var/tmp` 的原因：**重启不丢数据**（`/tmp` 常被清空甚至是 tmpfs）、不进 `$HOME`、按 uid 隔离。镜像本体是软链到 `out/`，不占这里；真正增长的是每个实例的 overlay/qcow2 和日志。

**能否改目录？** 源码里 `CvdDir()` 是编译期常量（`common.cpp` 的 `kDefaultCvdDir[] = "/var/tmp/cvd"`），**没有环境变量/参数可改**。想换盘只能整棵搬走再软链：

```bash
cvd reset                                  # 先全部停掉
sudo mv /var/tmp/cvd /mnt/bigdisk/cvd      # 搬到空间大的位置
sudo ln -s /mnt/bigdisk/cvd /var/tmp/cvd   # 让 cvd 无感知
cvd create
```

### cvd 常用命令速查

```bash
# —— 每次开工前（核心三步，缺一不可）——
source build/envsetup.sh                     # 导出 ANDROID_HOST_OUT / ANDROID_PRODUCT_OUT
lunch aosp_cf_x86_64_phone-trunk_staging-userdebug
cvd create                                   # 建组并开机；本地构建模式读上面两个变量

# —— 创建 ——
cvd create --start=false                     # 只建组不开机（等价 --nostart），状态为 Stopped
cvd create --config_file=cf.json             # 用 JSON 指定 CPU/内存/分辨率/多设备
cvd create --product_path=DIR --host_path=DIR  # 显式指定镜像/宿主工具目录（不 lunch 或预编译场景）

# —— 生命周期 ——
cvd start                                    # 开机（重新编译后重启）
cvd stop                                     # 关机（保留数据）
cvd restart                                  # 重启
cvd remove                                   # 彻底删除实例组
cvd reset                                    # 兜底清场

# —— 查看 ——
cvd fleet                                    # 列出全部设备（JSON）
cvd ps                                       # 人类可读设备列表（含已停机实例）
cvd status                                   # 某实例组状态（会连 launcher，需在运行）
cvd logs                                     # 列日志文件；cvd logs -p launcher.log 看指定日志
cvd monitor                                  # 实时跟踪日志

# 多组时用 selector 指定目标
cvd -group_name=cvd_1 stop

# 设备内操作
cvd powerbtn / cvd powerwash / cvd bugreport
cvd version
cvd help <command>                           # 任何子命令的帮助
```

> **不想 `lunch` 时**，本机的等价路径写死如下：
> ```bash
> cvd create \
>   --host_path=$HOME/projects/aosp/out/host/linux-x86 \
>   --product_path=$HOME/projects/aosp/out/target/product/vsoc_x86_64
> ```
> 注意 **`--host_path` 要填 `bin/` 的上一级**（cvd 自己会拼 `bin/`，见 `host_tool_target.cpp` 的 `GetBinName()`）；填成 `.../linux-x86/bin` 它会去找 `.../bin/bin/cvd_internal_start` 而报错。

> 不想自己编译、想直接跑 Google CI 的现成构建：`--config_file` 里把 `disk.default_build` 写成 `@ab/aosp-android-latest-release/aosp_cf_x86_64_only_phone-userdebug` 即可（或先 `cvd fetch` 再 create）。

---

## 六、开发-修改-验证闭环

### 全量编译
```bash
cd ~/aosp && source build/envsetup.sh

m -j$(nproc)      # 重新编译
cvd stop          # 关机（保留数据）
cvd start         # 重新开机，加载新镜像
```

### 增量编译（只改一个模块时快得多）
```bash
make SystemUI -j$(nproc)
adb root && adb remount
adb push out/target/product/vsoc_x86_64/system/priv-app/SystemUI/SystemUI.apk \
       /system/priv-app/SystemUI/
adb reboot
```

### 典型修改实验

| 难度 | 实验 | 对应目录 |
|------|------|---------|
| ⭐ | 修改开机动画 | `frameworks/base/cmds/bootanimation/` |
| ⭐ | 修改系统属性 | `build/make/tools/buildinfo.sh` |
| ⭐⭐ | 添加自己的 system app | `packages/apps/` |
| ⭐⭐⭐ | 修改 SystemUI 状态栏 | `frameworks/base/packages/SystemUI/` |
| ⭐⭐⭐ | 修改 Settings 加设置项 | `packages/apps/Settings/` |
| ⭐⭐⭐⭐ | 添加一个系统服务 | `frameworks/base/services/` |
| ⭐⭐⭐⭐⭐ | 自定义内核 | `kernel/` |

---

## 七、使用预编译镜像（跳过编译，快速上手）

如果不想等编译，可以直接下载 Google CI 上编译好的镜像：

```bash
# 1. 访问 https://ci.android.com
# 2. 选择分支 android-latest-release
# 3. 点击 build target: aosp_cf_x86_64_phone-userdebug
# 4. 下载 cvd-host_package.tar.gz + aosp_cf_x86_64_phone-img-<buildid>.zip

# 5. 运行：把 host 包与镜像解压到同一目录，用系统已装的 cvd 客户端指向它
mkdir ~/cf-demo && cd ~/cf-demo
tar xzf ~/Downloads/cvd-host_package.tar.gz            # 解出 bin/ 宿主工具
unzip -o ~/Downloads/aosp_cf_x86_64_phone-img-*.zip    # 解出 system.img 等到当前目录

cvd create --product_path=$HOME/cf-demo --host_path=$HOME/cf-demo

# 直接就有 root
adb shell    # uid=0
```

---

## 八、AOSP 源码结构（关键目录）

```
~/aosp/
├── art/                  # Android Runtime (ART)
├── bionic/               # C 标准库 (libc, libm, libdl)
├── bootable/             # 引导相关
├── build/                # 编译系统 (Soong / Make)
│   ├── make/             # Make 构建规则
│   └── soong/            # Soong (Go 语言构建系统)
├── device/               # 设备配置
│   └── google/cuttlefish/
├── external/             # 外部开源库
├── frameworks/           # ★ 核心框架
│   ├── base/             # Framework 基础
│   │   ├── core/         #  核心 API
│   │   ├── services/     #  系统服务
│   │   └── packages/     #  内置包 (SystemUI, SettingsProvider)
│   ├── native/           # Native (C++) 框架
│   └── av/               # 音视频框架
├── hardware/             # HAL 接口定义
├── kernel/               # 内核源码
├── packages/             # 系统应用
│   ├── apps/             #  Settings, Calendar, Camera...
│   └── providers/        # 内容提供者
├── system/               # 核心系统组件
│   ├── core/             #  init, adbd, logd
│   ├── bt/               #  蓝牙
│   └── netd/             #  网络管理
├── vendor/               # 厂商专属
└── out/                  # ★ 编译产物
    └── target/product/vsoc_x86_64/
        ├── system/       #  编译出的 system 分区
        ├── vendor/       #  编译出的 vendor 分区
        └── *.img         #  分区镜像
```

---

## 九、进阶技巧

### adb 日志调试
```bash
# 实时 logcat
adb logcat -v threadtime *:V

# 只看 SystemUI
adb logcat -v threadtime SystemUI:V *:S

# 内核日志
adb shell dmesg -w

# 全量诊断
adb bugreport

# crash dump
adb shell ls /data/tombstones/
```

### Android Studio for Platform 调试
```bash
cd ~/aosp
source build/envsetup.sh
lunch aosp_cf_x86_64_phone-trunk_staging-userdebug
make idegen && development/tools/idegen/idegen.sh
# 然后用 Android Studio for Platform 打开 android.ipr
```

### 自定义内核
```bash
mkdir ~/kernel && cd ~/kernel
repo init -u https://android.googlesource.com/kernel/manifest -b common-android15-6.6
repo sync -c -j$(nproc)
tools/bazel run //common-modules/virtual-device:virtual_device_x86_64_dist
# cvd 会从镜像目录加载内核：把编出的 bzImage 拷过去，再重启
cp /path/to/bzImage "$ANDROID_PRODUCT_OUT/bzImage"
cvd stop && cvd start
```

---

## 十、常见问题

| 问题 | 解决 |
|------|------|
| 编译内存不够 | `make -j4` 或 `make droid -j4` |
| 磁盘空间不够 | `make installclean` 或 `make clean` |
| repo sync 太慢 | 加 `--current-branch` 只同步当前分支 |
| Cuttlefish 起不来 | 检查 `/dev/kvm`、`groups`、`cvd fleet`/`cvd logs` 与 `/var/log/cuttlefish*` |
| 启动即 `VIRTUAL_DEVICE_BOOT_FAILED`，日志有 `unshare(CLONE_NEWNS): Operation not permitted` | AppArmor 限制非特权 user namespace → `sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0`（永久化见下 ①） |
| `cvd create` 报 `New instance conflicts with existing instance` | 上次失败的 `Boot Failed` 组残留在数据库 → `cvd reset` 后再 create |
| 启动打印 `Unable to get group id for group virtaccess` | 无害噪音，仅用于判断是否 CrOS 环境，可忽略 |

### 实战排错笔记（本机实测）

**① minijail 起不来（最常见的启动失败）** `launcher.log` 里长这样：

```
crosvm … the architecture failed to build the vm
  … failed to create proxy device: … Connection reset by peer
log_tee … crosvm[1]: libminijail[1]: unshare(CLONE_NEWNS) failed: Operation not permitted
run_cvd … VIRTUAL_DEVICE_BOOT_FAILED
```

根因是 Ubuntu 24.04 的 AppArmor 限制非特权 user namespace，crosvm 的 minijail 建不了沙箱。修复：

```bash
sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0
# 永久生效（重启后保留）：
echo 'kernel.apparmor_restrict_unprivileged_userns=0' | sudo tee /etc/sysctl.d/60-cuttlefish.conf
```

注意：这条 sysctl 重启会还原成 1，不写 `/etc/sysctl.d/` 的话下次开机又起不来。

> 补充（本机实测）：sysctl 修好后仍失败——**最终根因其实是 crosvm seccomp 的 madvise**（audit `type=1326 … sig=31 syscall=28`），见 §五「本机必做前置修复」。AppArmor/userns 只是修复过程中遇到的第一层报错。

**② 失败的 create 会污染"台账"** 设备没起来但实例数据库里留下了 `Boot Failed` 组，之后 `cvd create` 就报 `conflicts with existing instance: cvd_1/1`。清法：`cvd reset`（删组 + 杀进程 + 清数据库）后再 `cvd create`。

**③ `virtaccess` 噪音** `Unable to get group id for group virtaccess` 不是报错。`host_configuration.cpp` 只拿这个组判断"是否 CrOS 环境"（是就放宽内核版本检查），普通 Linux 忽略即可。

**④ 日志去哪看**

```bash
cvd logs                       # 实例日志路径清单
cvd logs -p launcher.log       # 看某个日志
ls /var/tmp/cvd/1000/logs/cvd_*.log     # cvd 客户端/服务端日志
/var/tmp/cvd/1000/<group>/home/cuttlefish/instances/cvd-1/{launcher.log,kernel.log,logcat}  # 实例进程日志，真正排错看这里
```

**⑤ Web UI 端口** 以启动时 launcher 打印的 `Point your browser to …` 为准；本机当前为 `https://localhost:8443`。

---

## 十一、推荐学习路径

```
第 1 周   环境搭建 → 用预编译镜像体验 Cuttlefish → 熟悉 adb root / logcat
第 2 周   下载 AOSP 源码 → 首次编译 → 启动自己编译的镜像
第 3-4 周 改开机动画 → 改 build.prop → 改 Settings → 改 SystemUI
第 5-8 周 添加 system app → 改 framework 加系统服务 → 理解 init.rc → 编译自定义内核
之后      看 LineageOS 源码 → 理解它比 AOSP 多了什么 → 尝试给真机移植
```

整个流程不需要任何真机，不需要担心 BL 锁。
