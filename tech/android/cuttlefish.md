# AOSP + Cuttlefish 完整指南

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

---

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

```bash
cd ~/aosp
launch_cvd --daemon
```

验证：
```bash
adb devices           # 应看到 localhost:6520  device
```

浏览器打开 `https://localhost:8443` 看到手机界面。

```bash
adb root && adb shell
whoami                # root
id                    # uid=0
getprop ro.build.type # userdebug
```

**常用参数：**
```bash
# 指定内存和核数
launch_cvd --cpus 4 --memory_mb 4096

# 指定分辨率
launch_cvd --x_res 1080 --y_res 2340 --dpi 420

# 带 WiFi 模拟
launch_cvd --wifi

# 多设备
launch_cvd --instance_n 2   # 第二个实例端口 +10
```

**停止：**
```bash
stop_cvd
HOME=$PWD stop_cvd
```

---

## 六、开发-修改-验证闭环

### 全量编译
```bash
cd ~/aosp && source build/envsetup.sh
m -j$(nproc)
stop_cvd && launch_cvd --daemon
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

# 5. 运行
mkdir ~/cf-demo && cd ~/cf-demo
tar xzf ~/Downloads/cvd-host_package.tar.gz
unzip ~/Downloads/aosp_cf_x86_64_phone-img-*.zip
HOME=$PWD ./bin/launch_cvd --daemon

# 直接就有 root
./bin/adb shell    # uid=0
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
launch_cvd --kernel /path/to/bzImage
```

---

## 十、常见问题

| 问题 | 解决 |
|------|------|
| 编译内存不够 | `make -j4` 或 `make droid -j4` |
| 磁盘空间不够 | `make installclean` 或 `make clean` |
| repo sync 太慢 | 加 `--current-branch` 只同步当前分支 |
| Cuttlefish 起不来 | 检查 `/dev/kvm`、`groups`、`/var/log/cuttlefish*` |

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
