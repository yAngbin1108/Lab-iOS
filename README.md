<div align="center">

<img src="Lab.png" alt="Lab" width="140">

# Lab

**iOS 内核工具箱 · IPA 分析 · 设备诊断**

基于 [lara](https://github.com/rooootdev/lara) 二次开发

iOS 16.0 – 18.7.1 · iOS 26.0.x

</div>

---

Lab 把两类工具合进了同一个 App：

- **内核工具箱** —— 通过 DarkSword 漏洞利用获得内核读写与沙盒外文件访问
- **分析工具** —— IPA 差异比对、本地化资源提取、包结构分析、崩溃日志归类

分析工具全部基于公开 API，**不依赖漏洞利用是否成功**。两者互不影响。

---

## 目录

- [快速开始](#快速开始)
- [功能](#功能)
- [安装说明](#安装说明)
- [从源码构建](#从源码构建)
- [架构设计](#架构设计)
- [与上游的差异](#与上游-lara-的差异)
- [常见问题](#常见问题)
- [许可与致谢](#许可与致谢)

---

## 快速开始

不想自己编译？直接用发布版：

1. 下载最新 Release 里的 `Lab.ipa`
2. 用 **ESign / AltStore / SideLoaded** 等工具，以你自己的 Apple ID 签名安装
3. 打开 App → 「内核」标签页 → 点 **运行漏洞利用** → 再点 **初始化系统**

想用 GitHub Actions 自动构建，见 [从源码构建](#从源码构建)。

---

## 功能

### 🔧 内核工具箱

| 功能 | 说明 |
|---|---|
| **运行漏洞利用** | 执行 DarkSword，建立内核读写通道 |
| **获取内核缓存** | 抓取并解析内核缓存，得到符号偏移 |
| **初始化系统** | 完成后利用（VFS + 沙盒逃逸），开放全盘读写 |
| **重启桌面** | 软重启 SpringBoard |
| **触发 Panic** | 强制内核崩溃，设备立即重启 |

三种利用方式可在设置中切换：`VFS` / `SBX` / `Hybrid`（默认 Hybrid）。

### 📁 文件管理器

浏览沙盒之外的完整文件系统（`/System`、`/var`、`/private` 等），支持查看文件信息、读写、删除、递归搜索。

### 🧰 工具箱（分析模块）

五个模块都只使用公开接口，无需任何内核权限：

**IPA 差异分析**
对比两个 IPA 的文件增删改。两级比对：路径集合 → CRC32 → SHA-256 内容确认，输出可分享的文本报告。

**IPA 本地化管理**
提取 `.strings` / `.stringsdict`，支持 binary plist 与 UTF-8/UTF-16 两种格式，可导出 JSON / TXT 对照表。

**应用行为分析**
解析 Info.plist 全部键值、隐私用途声明、`embedded.mobileprovision` 里的授权信息、动态库清单与资源类型分布。

**运行监控**
导入 `.ips` / `.crash` 日志，识别异常类型与终止原因，按内存、断言、动态链接、权限等维度自动归类。

**系统能力检测**
设备与系统信息，以及内核读写通道的实时状态。

---

## 安装说明

### 关于签名的一个常见误解

Lab 的内核功能**不依赖安装时注入的私有 entitlements**。

它的机制是：App 以普通方式签名安装 → 运行时通过 DarkSword 获得内核读写 → 再用 `sbx_escape` 从沙盒内部完成逃逸。也就是说，**系统是运行时被"说服"的，不是靠签名预先授权的**。

因此：

| 安装方式 | 能否工作 |
|---|---|
| Apple ID 自签（ESign / AltStore） | ✅ 正常 |
| TrollStore | ✅ 正常 |
| Xcode 真机调试 | ⚠️ 调试器附加时功能会被禁用 |

> 本项目早期文档曾要求必须使用 TrollStore，那是错误的判断，已更正。

### 支持范围

| 系统版本 | 状态 |
|---|---|
| iOS 17.0 – 18.7.1 | ✅ 支持 |
| iOS 26.0 – 26.0.1 | ✅ 支持 |
| iOS 16.x | ⚠️ 理论受影响，偏移未完整验证 |
| iOS 18.7.2+ / 26.1+ | ❌ 不支持 |
| A19 / M5 及更新芯片 | ❌ 因 MIE 无法使用 |

M 系列 CPU 设备如遇问题，可在 **设置 → 修改偏移** 中将 `t1sz_boot` 设为 `0x11`。

### ⚠️ 风险提示

- 漏洞利用存在失败概率，失败通常无害，重试即可
- **触发 Panic 会让设备立即重启**，正在写入的数据可能丢失
- 内核内存写入若越界，最坏情况需要恢复模式重刷系统
- 请先备份重要数据

本工具仅供在**你自己的设备**上研究与学习使用。

---

## 从源码构建

### 方式一：GitHub Actions（推荐，无需 Mac）

仓库根目录只需两个文件：

```
your-repo/
├── .github/workflows/build-from-zip.yml   ← 构建脚本
└── Lab-1.0.0-source.zip                   ← 源码包
```

Actions 会自动解压源码包并完成编译，产物在 **Artifacts → Lab-ipa**（一个未签名裸包，交给 ESign 等工具自签）。

也可以手动触发：Actions → *Build Lab IPA* → **Run workflow**，可选 Release / Debug。

### 方式二：本地 Xcode

```bash
unzip Lab-1.0.0-source.zip && cd Lab
./scripts/build_ipa.sh            # Release，产出 build/Lab.ipa
./scripts/build_ipa.sh --debug    # Debug 构建
```

| 项目 | 要求 |
|---|---|
| Xcode | **16 或更新**（工程为 `objectVersion = 77`） |
| SDK | 需 iOS 26 SDK（PartyUI 用到 `.glassEffect`） |
| 部署目标 | iOS 16.0 |
| 架构 | arm64e |
| 第三方依赖 | **无** |

### CI 自检

每次构建前，`validate` job 会在 Ubuntu runner 上跑四项检查（约 40 秒）：

| 脚本 | 作用 |
|---|---|
| `Checks/preflight.py` | 关键文件是否齐全、pbxproj 内部引用是否一致 |
| `Checks/validate_structure.py` | Swift 括号与字符串插值配平、顶层类型重名、View 属性冲突、macOS 专属 API |
| `Checks/test_bridging_header.py` | 桥接头引用的头文件是否存在且有实现 |
| `Checks/test_build_workflow.py` | 测试脚本与 workflow 步骤是否一一对应 |

这些检查专门用来拦截「要等 macOS 编译几分钟才暴露」的问题。

### 更换 App 图标

图标有三个外观变体（标准 / 深色 / 着色），`Contents.json` 按固定文件名引用，**必须一起更新**，否则深色模式下会露出旧图。

用脚本一条命令搞定：

```bash
python3 Checks/make_icon.py <你的源图.png>
```

它会自动生成三种变体、缩放到 1024×1024、强制去除 alpha 通道，并校验 `Contents.json` 的引用一致性。需要本机装有 ffmpeg（`brew install ffmpeg`）。

---

## 架构设计

```
Lab/
├── Lab.swift                    入口，装配标签页
├── Lab-Bridging-Header.h        Swift ↔ Objective-C 桥接
│
├── kexploit/                    内核漏洞利用（Objective-C）
│   ├── darksword.m              主利用链
│   ├── offsets.m                内核符号偏移
│   ├── TaskRop/                 ROP 原语、线程与 VM 操作
│   └── pe/                      后利用：APFS / vnode / VFS / 沙盒逃逸
│
├── classes/
│   ├── LabMgr.swift             内核状态的唯一来源
│   ├── zipmgr.swift             ZIP 归档
│   └── Logger.swift
│
├── views/                       内核侧界面（SwiftUI）
│   ├── app/                     内核主页、日志、设置
│   └── fm/                      文件管理器
│
├── Analyzer/                    ★ 分析模块（本项目新增）
│   ├── Core/
│   │   ├── CapabilityDetector.swift   能力探测
│   │   ├── LabAppModel.swift          分析侧全局状态
│   │   ├── IPAArchive.swift           纯 Swift ZIP 解析器
│   │   ├── DiagnosticsManager.swift   操作记录
│   │   └── ToolboxView.swift          工具箱入口页
│   ├── Modules/                 五个分析模块
│   └── Shared/LabTheme.swift    统一视觉规范
│
└── PartyUI/                     lara 的设计系统
```

### 三条设计原则

**1. 能力判定只信实测，不信推断**

`CapabilityDetector` 不允许出现「设备是 iPhone 15 且系统是 iOS 17.5.1 → 所以某能力可用」这类逻辑。每项能力的结论都必须来自一次真实探测。

**2. 运行时状态不做快照缓存**

内核状态（漏洞利用是否成功、偏移是否就绪）由界面**每次直接从 `LabMgr` 读取**。早期版本曾在启动时拍一次快照，导致「内核页显示已成功、能力检测页显示全不可用」的自相矛盾。

**3. 分析模块与内核解耦**

`Analyzer/` 不直接引用 `kexploit`，只通过 `LabAppModel` 读取内核状态。这样即使漏洞利用完全不可用，五个分析模块也能独立工作。

### 无 Swift Package 依赖

分析模块的所有能力都用系统框架自行实现：

| 需求 | 实现 |
|---|---|
| ZIP 解析 | `IPAArchive.swift`（store / deflate / ZIP64 / CRC32） |
| SHA-256 | CryptoKit |
| zlib 解压 | Compression 框架 |
| plist 解析 | PropertyListSerialization |

工程里**没有任何 `XCRemoteSwiftPackageReference`**，CI 因此可以完全离线构建，不会因为拉取 SPM 包超时或版本漂移而失败。

需要说明的是，内核部分仍然带有 lara 随附的两个预编译二进制：`Lab/lib/libgrabkernel2.dylib` 与 `Lab/lib/libxpf.dylib`。它们是 vendored 的二进制依赖（非 SPM），许可见 `Lab/licenses/`。

---

## 与上游 lara 的差异

### 移除的功能

只保留内核核心与分析工具，以下 tweak 全部删除：

字体替换 · 表情符号 · IPA 解密 · OTA 延迟 · 屏幕使用时间 · DarkBoard 图标主题 · 锁屏密码主题 · LiquidGlass · dirtyZero · SpringBoard 自定义 · MobileGestalt 编辑 · var 清理 · 白名单 · JIT · RemoteCall

> **RemoteCall 已完整移除**，包括其界面、设置项与后台保活逻辑。它相对不稳定且需要 PAC 上下文，多数场景下漏洞利用自带的文件操作已经够用。

### 新增的功能

- 五个 IPA 分析与诊断模块
- 纯 Swift 实现的 ZIP 解析器（去除第三方依赖）
- 工具箱卡片式入口页
- 结构性 CI 自检体系
- 中文界面

### 修复的上游构建问题

| 问题 | 影响 |
|---|---|
| `CURRENT_PROJECT_VERSION = Release` | 非数字值，xcodebuild 直接报错 |
| 项目级 `IPHONEOS_DEPLOYMENT_TARGET = 26.0` | 未来版本号，构建失败 |
| 缺少共享 scheme | CI 里 `xcodebuild -scheme` 找不到目标 |
| 桥接头残留无实现的函数声明 | 编译能过但链接期失败 |

---

## 常见问题

**Q：点了「运行漏洞利用」没反应？**
A：检查是否有调试器附加（Xcode 运行时会出现此限制）。另外该功能本身有失败概率，重试是正常的。

**Q：文件管理器提示"未就绪"？**
A：需要先依次完成「运行漏洞利用」→「获取内核缓存」→「初始化系统」。

**Q：本地化管理显示"共 0 个本地化文件"？**
A：这是正常的。很多 App 把界面文字硬编码在代码里，不使用 `.strings` 文件。系统应用、社交类和游戏类应用的本地化文件通常较多。

**Q：能不能读取其他 App 的沙盒数据？**
A：完成「初始化系统」后可以。在此之前受沙盒限制。请注意这属于高权限操作，谨慎使用。

**Q：iOS 升级后还能用吗？**
A：不保证。漏洞利用与具体系统版本和内核布局强相关。

---

## 许可与致谢

本项目基于 **[lara](https://github.com/rooootdev/lara)** 二次开发，继承其 **AGPL-3.0** 授权。完整条款见 [`LICENSE`](LICENSE)。

如果你分发本项目的修改版或编译产物，请遵守 AGPL-3.0：保留原作者署名与许可证文本，并以相同授权开源你的修改。

### 原始项目

- **[rooootdev](https://github.com/rooootdev)** — lara 原作者
- **ruter** — lara 主程序与内核工具链
- **neon** — respring 脚本、zipmgr、锁屏主题修复、应用解密
- lara 社区与全部贡献者 · [Discord](https://discord.gg/gw8PcRF3Jr)

### 底层组件

| 组件 | 用途 |
|---|---|
| [ChOma](https://github.com/alfiecg24/ChOma) | Mach-O 与代码签名解析 |
| [libgrabkernel2](https://github.com/wh1te4ever/libgrabkernel2) | 内核缓存抓取 |
| [XPF](https://github.com/0xH1v3/XPF) | 内核补丁框架 |
| DarkSword | 内核漏洞利用 |

各组件的许可文本见 `Lab/licenses/`。

---

<div align="center">

**免责声明**

本项目仅供在你自己的设备上进行安全研究与学习。
使用者需自行承担设备数据丢失或系统损坏的风险，
并确保行为符合当地法律法规。
本项目不提供任何形式的明示或暗示担保。

</div>
