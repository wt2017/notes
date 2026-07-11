# Android Root

## Root 的本质

**让自己的进程获得 root UID (uid=0)，突破 Android 的沙箱权限模型。**

Android 的权限基于 Linux 的 UID/GID 机制：

```
普通应用 → uid=100xx (app_xxx)
系统服务 → uid=1000 (system)
shell    → uid=2000
root     → uid=0  ← 正常 Android 下无法获取
```

正常 Android 系统中，用户空间所有进程都不可能是 uid=0。`su` 二进制不存在，adbd 以 uid=2000 降权运行。

**所以 root 的本质就是打破这个限制：让一个进程有能力把自身或其他进程的 UID 设为 0，从而绕过 Android 的所有安全检查。**

## Android 砍掉了 root 通路

Android 在构建层面就把获取 root 的路径彻底删除了，不是"藏起来"，是"建造时就没放进去"：

| 组件 | 标准 Linux | Android (user build) |
|------|-----------|----------------------|
| **init** | pid=1, uid=0 | pid=1, uid=0 ✅ (相同) |
| **shadow 文件** | `/etc/shadow` 有 root 密码 | ❌ 不存在 |
| **su 二进制** | 有 suid bit，存在于 `/usr/bin/su` | ❌ 不存在，未预装 |
| **adbd** | — | uid=2000 (shell)，故意降权运行 |
| **getty/登录** | 提供控制台 root 登录 | ❌ 无终端登录概念 |

adb shell 进去的环境**就是 Linux 系统本身**，不是被隔离的——只是身份是 `uid=2000`，跟服务器上别人给你开了一个没有 sudo 的普通账号是一个道理。

三层防护确保无法获取 root：

```
内核层面:   uid=0 的能力存在       ← 内核认得 root，Linux 天然的
系统层面:   获取 uid=0 的路径被删除  ← su / adbd root 都被移除
策略层面:   SELinux 进一步限制      ← 即使内核有 root 能力，SELinux 也会阻止
```

## Root 的方法

### 路径 A：Magisk 修补 boot.img（主流方式）

```
正常启动:
  bootloader → kernel → init → ... → adbd (uid=2000)

Magisk 修改后:
  bootloader → 修补过的 boot.img → init 阶段注入 magiskd (uid=0)
  → 提供 su 二进制接口 → 应用通过 su 向 magiskd 请求提权
  → 弹窗询问用户是否授权
```

特点：
- 只动了 boot 分区，**没有替换 system 分区**
- 没有修改 Android Framework
- 本质上是在 init 阶段注入了一个 root 守护进程，由它当"守门员"
- **不需要刷机，但需要解锁 bootloader**

### 路径 B：利用漏洞提权（不需要解锁 bootloader）

利用内核漏洞（如脏牛 Dirty Cow）修改进程凭证结构中的 UID。

特点：
- 无需解锁 bootloader，无需修改任何分区
- 现代 Android 内核安全补丁更新快，漏洞越来越少
- 新手机上基本不可行

### 例外情况

| 场景 | root 状态 |
|------|-----------|
| **Engineering build (工程机)** | 默认就是 root，adb root 直接可用 |
| **Userdebug build** | adb root 可用，带额外权限 |
| **旧版本 Android (< 4.x)** | adbd 默认以 root 运行 |
| **生产版本 (user build)** | ⛔ 所有通路被封死 |

## Root 的限制

root 不等于无所不能：

| 限制 | 原因 |
|------|------|
| **Bootloader 锁** | root 不解决这个，需要先解锁 |
| **TEE / Secure World** | 指纹、支付密钥在 TrustZone 里，root 权限在 normal world 进不去 |
| **eFuse 熔断 (Samsung Knox)** | 物理熔断，root 也恢复不了 |

## Root vs 刷机

root 和刷机是两回事，只是经常一起出现：

| | Root | 刷机 (Custom ROM) |
|---|---|---|
| 动什么 | boot 分区 | system + 可能 vendor 分区 |
| 系统版本 | 保留原厂系统 | 替换为第三方 ROM |
| 目的 | 获得 uid=0 权限 | 换系统、升级 Android 版本 |

不刷机也能 root（Magisk 只动 boot），但两者都依赖 bootloader 解锁。

## Root 后的影响

- 可以访问 `/data/data/` 下任何应用的数据
- 可以修改 `/system/build.prop` 等系统文件
- 可以冻结/卸载系统应用
- 绕过所有 Android Permission 检查
- 可被部分应用检测（银行、支付、游戏），但现在有隐藏方案

## 一句话总结

> 你已经在 Linux 里面了，你只是被限制了权限。想要 root，你必须在系统启动链条的某个环节插入你自己的代码，为自己重建一条通往 uid=0 的路。
