# ADK 10.0.17704.1000 — 完整工具清单与深度分析

> 文档版本: 1.0 | 日期: 2026-08-27
> 范围: ADK i386 目录全部 24 个 EXE + 全部非运行时 DLL（排除 api-ms-win-* 运行时垫片）
> 反编译源: `outputsrc\native\` (87个原生反编译) + `outputsrc\dotnet\` (67个.NET反编译) + `ida_decompiled\native\` (IDA完整版)

---

## 目录

1. [执行摘要](#1-执行摘要)
2. [EXE 工具完整清单（24个）](#2-exe-工具完整清单24个)
3. [未深度分析工具详解](#3-未深度分析工具详解)
4. [DLL 完整分类清单](#4-dll-完整分类清单)
5. [工具依赖关系图](#5-工具依赖关系图)
6. [已分析文档交叉索引](#6-已分析文档交叉索引)
7. [重建项目优先级](#7-重建项目优先级)

---

## 1. 执行摘要

ADK 17704 i386 目录包含 **24 个 EXE** 和 **199 个 DLL**（其中约 100 个是 api-ms-win-* 运行时垫片，排除后约 99 个功能 DLL）。所有工具和 DLL 均已反编译（原生 + .NET）。

### 按功能分类的 EXE

| 类别 | 工具 | 数量 |
|------|------|------|
| 镜像构建核心 | imageapp.exe, updateapp.exe, imaging.dll(被调用) | 2 |
| 包构建 | PkgGen.exe, spkggen.exe, ConvertDSM.exe, pkgsigntool.exe | 4 |
| FFU 操作 | ffutool.exe, ffucomponents.dll(被调用), ImgDump.exe, imgtowim.exe | 3 |
| WIM 操作 | imagex.exe, secwimtool.exe, wpimage.exe | 3 |
| 存储/空间 | rs1-5_spaceutil.exe (5个版本) | 5 |
| Feature 管理 | featuremerger.exe, featureapi.dll(被调用) | 1 |
| 签名 | imagesigner.exe, signtool.exe | 2 |
| AppX | makeappx.exe | 1 |
| OEM 定制 | OemCustomizationTool.exe | 1 |
| 设备清理 | devicenodecleanup.x86/x64.exe | 2 |

### 关键发现

1. **几乎所有 .NET 工具的 AssemblyFileVersion 都是 10.0.10011.16384**（Windows 10 Threshold 早期版本），说明这些工具自 Windows 10 首次发布以来基本未更新
2. **spaceutil.exe 有 5 个版本**（rs1-rs5），对应 Windows 10 五个主要版本，每个版本适配不同的 Storage Spaces 驱动接口
3. **secwimtool.exe 创建 .secwim 格式**：SDI 头 + WIM 数据 + 平台ID + 序列号 + SHA256 安全目录，用于工厂刷机场景
4. **ffutool.exe 是手机刷机工具**：支持 Qualcomm Recovery Platform，设备状态机（CONNECTED→FLASHING→TRANSFER_WIM→BOOTING_WIM→RECOVERING→DONE）
5. **所有工具共享公共库**：PkgBldr.*.dll（包构建框架）、toolscommon.dll、imagecommon.dll、pkgcommonmanaged.dll

---

## 2. EXE 工具完整清单（24个）

### 2.1 镜像构建核心

| 工具 | 大小 | 运行时 | 命名空间 | 状态 |
|------|------|--------|---------|------|
| imageapp.exe | 16,896B | .NET | Microsoft.WindowsPhone.ImageUpdate | 已分析(#01/#02) |
| updateapp.exe | 736,256B | Native | (原生C++) | 已分析(#01/#07) |

### 2.2 包构建

| 工具 | 大小 | 运行时 | 命名空间 | 状态 |
|------|------|--------|---------|------|
| PkgGen.exe | 28,160B | .NET | (PkgGen入口) | 已分析(#08) |
| spkggen.exe | 19,968B | .NET | Microsoft.WindowsPhone.ImageUpdate.PackageGenerator | 已分析(#12) |
| ConvertDSM.exe | 354,816B | Native | (原生C++, convertdsm.cpp) | 已分析(#12) |
| pkgsigntool.exe | 25,088B | .NET | Microsoft.WindowsPhone.ImageUpdate.PkgSignTool | 未深析 |

### 2.3 FFU 操作

| 工具 | 大小 | 运行时 | 命名空间 | 状态 |
|------|------|--------|---------|------|
| ffutool.exe | 45,056B | .NET | Microsoft.Windows.ImageTools | 未深析 |
| ImgDump.exe | 11,264B | .NET | ImgDump | 未深析 |
| imgtowim.exe | 11,776B | .NET | ImgToWIM | 未深析 |

### 2.4 WIM 操作

| 工具 | 大小 | 运行时 | 命名空间 | 状态 |
|------|------|--------|---------|------|
| imagex.exe | 657,200B | Native | (Windows AIK 标准工具) | 未深析 |
| secwimtool.exe | 3,189,248B | .NET | SecureWim | 已分析(本文档) |
| wpimage.exe | 20,480B | .NET | Microsoft.WindowsPhone.WPImage | 未深析 |

### 2.5 存储/空间

| 工具 | 大小 | 运行时 | 说明 | 状态 |
|------|------|--------|------|------|
| rs1_spaceutil.exe | 124,928B | Native | Windows 10 1507 (Threshold) | 未深析 |
| rs2_spaceutil.exe | 123,904B | Native | Windows 10 1607 (Anniversary) | 未深析 |
| rs3_spaceutil.exe | 104,960B | Native | Windows 10 1703 (Creators) | 未深析 |
| rs4_spaceutil.exe | 107,520B | Native | Windows 10 1803 (April 2018) | 未深析 |
| rs5_spaceutil.exe | 113,664B | Native | Windows 10 1809 (October 2018, 当前ADK) | 未深析 |

### 2.6 Feature 管理

| 工具 | 大小 | 运行时 | 命名空间 | 状态 |
|------|------|--------|---------|------|
| featuremerger.exe | 45,056B | .NET | Microsoft.WindowsPhone.ImageUpdate.FeatureMerger | 未深析 |

### 2.7 签名

| 工具 | 大小 | 运行时 | 命名空间 | 状态 |
|------|------|--------|---------|------|
| imagesigner.exe | 12,800B | .NET | Microsoft.WindowsPhone.Imaging.ImageSignerApp | 未深析 |
| signtool.exe | 315,696B | Native | (Windows SDK 标准工具) | 未深析 |

### 2.8 AppX

| 工具 | 大小 | 运行时 | 说明 | 状态 |
|------|------|--------|------|------|
| makeappx.exe | 414,000B | Native | (Windows SDK 标准工具) | 部分分析(#11) |

### 2.9 OEM 定制

| 工具 | 大小 | 运行时 | 命名空间 | 状态 |
|------|------|--------|---------|------|
| OemCustomizationTool.exe | 66,560B | .NET | Microsoft.WindowsPhone.ImageUpdate.OemCustomizationTool | 未深析 |

### 2.10 设备清理

| 工具 | 大小 | 运行时 | 说明 | 状态 |
|------|------|--------|------|------|
| devicenodecleanup.x86.exe | 8,704B | Native | x86 设备节点清理 | 未深析 |
| devicenodecleanup.x64.exe | 10,240B | Native | x64 设备节点清理 | 未深析 |

---

## 3. 未深度分析工具详解

### 3.1 secwimtool.exe — 安全 WIM 工具（完整分析）

**命名空间**: SecureWim | **AssemblyFileVersion**: 10.0.10011.16384 | **.NET 4.0**

#### 命令

```
secwimtool <command> <arguments>
```

| 命令 | 功能 |
|------|------|
| `build` | 从 WIM 创建 .secwim（含安全目录） |
| `extractcat` | 从 .secwim 提取安全目录 |
| `replacecat` | 替换 .secwim 中的安全目录 |
| `?` | 帮助 |

#### build 命令

```
secwimtool build <wim_path> <output_path> [/platform <id1;id2>] [/serial <guid1;guid2>]
```

**流程**:
1. 验证 WIM 头 (`MSWIM\0\0\0`)
2. 写入 SDI 头（嵌入资源 `sdiData`，头标识 `$SDI0001`）
3. 写入 WIM 数据
4. 4字节对齐填充
5. 写入平台 ID 目标数据（magic `0x74646970` = "pidt"）
6. 写入序列号目标数据（magic `0x69726573` = "seri"，16字节 GUID）
7. 写入大小结构（magic `0x657a6973` = "size"，targeting_size + sdi_size + wim_size）
8. 调用 `makecat.exe` 生成 SHA256 安全目录（.CDF 方式）
9. 追加目录数据 + 4字节目录大小

#### .secwim 文件格式

```
┌─────────────────────────────────┐
│ SDI Header ($SDI0001)          │ 嵌入资源 sdiData
├─────────────────────────────────┤
│ WIM Data (MSWIM)                │ 原始 WIM 文件内容
├─────────────────────────────────┤
│ Padding (4-byte aligned)        │
├─────────────────────────────────┤
│ Platform Targeting Data         │ magic "pidt" + 平台ID列表
├─────────────────────────────────┤
│ Serial Targeting Data           │ magic "seri" + GUID列表
├─────────────────────────────────┤
│ Size Structure                  │ magic "size" + sizes
├─────────────────────────────────┤
│ Security Catalog (.cat)         │ SHA256, makecat.exe 生成
├─────────────────────────────────┤
│ Catalog Size (4 bytes, LE)      │ 目录数据大小
└─────────────────────────────────┘
```

#### 关键类

| 类 | 功能 |
|----|------|
| `BuildCommand` | 构建 .secwim |
| `ExtractCommand` | 提取目录 |
| `ReplaceCommand` | 替换目录 |
| `Helpers` | 工具函数（WIM/SDI头检测、流操作、GUID解析） |
| `SecureWimTool` | 主入口（命令分发） |

### 3.2 ffutool.exe — FFU 刷机工具

**命名空间**: Microsoft.Windows.ImageTools | **引用**: FFUComponents, Microsoft.Windows.Flashing.Platform, Qualcomm.Recovery.Platform

#### 设备状态机

```
CONNECTED → FLASHING → TRANSFER_WIM → BOOTING_WIM → RECOVERING → DONE
                                                          ↓
                                                      EXCEPTION / ERROR
```

#### 关键类

| 类 | 功能 |
|----|------|
| `DeviceStatus` (enum) | 设备状态（10种状态） |
| `DeviceStatusPosition` (enum) | 状态位置 |
| `ConsoleEx` | 控制台扩展 |
| `EtwSession` | ETW 跟踪会话（IDisposable） |
| `LoggingModeConstant` (enum) | 日志模式常量 |

**说明**: ffutool.exe 是手机/设备刷机工具，支持通过 FFU (Full Flash Update) 镜像刷写设备。支持 Qualcomm 恢复模式平台。这是工厂/维修场景使用的工具，不是镜像构建工具。

### 3.3 spaceutil.exe — 存储空间工具（5个版本）

**运行时**: Native i386 | **交互**: Windows Storage Spaces (spaceport.sys)

#### 关键发现

- 与 Windows Storage Spaces 驱动 (`spaceport`) 交互
- 读取注册表: `SYSTEM\CurrentControlSet\Services\spaceport\Parameters`
- 引用 `clusdisk`（集群磁盘）和 `spaceport`（存储空间端口驱动）
- 引用 `SectionFlags`（存储区段标志）
- 5个版本对应 Windows 10 的 5 个 Redstone 版本，每个版本适配不同的驱动接口

#### 版本对应

| 版本 | Windows 10 | 版本号 |
|------|-----------|--------|
| rs1 | 1507 Threshold | 10.0.10240 |
| rs2 | 1607 Anniversary | 10.0.14393 |
| rs3 | 1703 Creators | 10.0.15063 |
| rs4 | 1803 April 2018 | 10.0.17134 |
| rs5 | 1809 October 2018 | 10.0.17763 (当前ADK 17704) |

**说明**: spaceutil.exe 用于配置和管理 Windows Storage Spaces（存储空间），在镜像构建中用于设置存储池和空间配置。ADK 包含多个版本以支持不同目标系统。

### 3.4 featuremerger.exe — Feature 合并工具

**命名空间**: Microsoft.WindowsPhone.ImageUpdate.FeatureMerger
**引用**: Microsoft.WindowsPhone.CompDB, FeatureAPI, PkgCommon, Tools.Common, Imaging

#### 关键类

| 类 | 功能 |
|----|------|
| `FeatureMerger` | 主类 |
| `CriticalFMProcessing` (enum) | 关键 FeatureManifest 处理模式 (Yes/No/All) |
| `FMCollectionManifest` | FeatureManifest 集合清单 |

**说明**: featuremerger.exe 用于合并多个 FeatureManifest (FM) 文件，生成最终的功能清单。它读取 CompDB（组件数据库）和 OEMInput，合并所有 FM 文件，输出统一的功能配置。这是镜像构建中 Feature 选择流程的关键工具。

### 3.5 imagesigner.exe — 镜像签名工具

**命名空间**: Microsoft.WindowsPhone.Imaging.ImageSignerApp

#### 关键类

| 类 | 功能 |
|----|------|
| `ImageSignerApp` | 主类 |
| `HashedChunkReader` | 分块哈希读取器（对镜像分块计算哈希） |
| `HashedChunkReaderException` | 异常 |
| `ImageSignerException` | 异常 |

**说明**: imagesigner.exe 对 FFU/WIM 镜像进行数字签名。`HashedChunkReader` 表明它对镜像进行分块哈希计算，然后对哈希列表进行签名（类似 FFU 的 SecurityHeader 机制）。

### 3.6 pkgsigntool.exe — 包签名工具

**命名空间**: Microsoft.WindowsPhone.ImageUpdate.PkgSignTool

#### 关键类

| 类 | 功能 |
|----|------|
| `Program` | 主入口 |
| `NativeMethods` | P/Invoke 原生方法 |

**说明**: pkgsigntool.exe 对 .spkg/.cab 包进行数字签名。与 signtool.exe（通用代码签名）不同，pkgsigntool 是专门为 CBS 包设计的签名工具，可能处理包特定的签名格式（如 update.cat 的签名）。

### 3.7 OemCustomizationTool.exe — OEM 定制工具

**命名空间**: Microsoft.WindowsPhone.ImageUpdate.OemCustomizationTool

#### 关键类

| 类 | 功能 |
|----|------|
| `Configuration` | 配置解析 |
| `Customization` | 定制项 |
| `CustomizationPkgBuilder` | 定制包构建器 |
| `ConfigXmlException` | 配置XML异常 |
| `CustomizationXmlException` | 定制XML异常 |

**说明**: OemCustomizationTool.exe 读取 OEM 定制 XML 配置，生成定制包（.spkg/.cab）。OEM 可以通过 XML 配置自定义系统设置、预装应用、注册表项等，工具将其转换为可安装的 CBS 包。

### 3.8 wpimage.exe — Windows Phone 镜像工具

**命名空间**: Microsoft.WindowsPhone.WPImage

#### 命令（IWPImageCommand）

| 命令 | 功能 |
|------|------|
| `MountCommand` | 挂载镜像 |
| `DismountCommand` | 卸载镜像 |
| `RemoveIdCommand` | 移除 ID |
| `DisplayIdCommand` | 显示 ID |

**说明**: wpimage.exe 是 Windows Phone 镜像管理工具，支持挂载/卸载 FFU 镜像进行离线修改，以及管理镜像中的 ID 标识。

### 3.9 ImgDump.exe — 镜像转储工具

**命名空间**: ImgDump | **类**: ImgDump

**说明**: 小型工具（11KB），用于转储 FFU 镜像的元数据信息（头部、manifest、存储布局等），用于调试和分析。

### 3.10 imgtowim.exe — 镜像转 WIM 工具

**命名空间**: ImgToWIM | **类**: ImgToWIM

**说明**: 小型工具（11KB），将 FFU 镜像转换为 WIM 格式，便于使用标准 WIM 工具处理。

### 3.11 标准 Windows SDK/AIK 工具

| 工具 | 说明 |
|------|------|
| imagex.exe | Windows AIK 标准 WIM 工具（捕获/应用/挂载 WIM） |
| signtool.exe | Windows SDK 标准代码签名工具 |
| makeappx.exe | Windows SDK 标准 AppX 打包工具 |

**说明**: 这三个是 Microsoft 标准工具，文档丰富，不在此深度分析。

### 3.12 devicenodecleanup.exe — 设备节点清理

**说明**: 小型原生工具（8-10KB），用于清理设备管理器中的残留设备节点。x86 和 x64 两个版本，在镜像定制后清理不需要的设备节点。

---

## 4. DLL 完整分类清单

### 4.1 镜像构建核心（已分析）

| DLL | 大小 | 运行时 | 状态 |
|-----|------|--------|------|
| imaging.dll | 114,688B | Native | 已分析(#05) |
| imagestorageservice.dll | 286,720B | Native | 已分析(#06) |
| imagestorageservicemanaged.dll | 267,776B | .NET | 部分分析 |
| imagecommon.dll | 417,792B | .NET | 未深析 |
| imagecustomization.dll | 119,808B | .NET | 未深析 |
| ImgToolsCommon.dll | 46,592B | .NET | 未深析 |
| updateapi.dll | 1,951,744B | Native | 未深析 |
| updatedll.dll | 1,584,640B | Native | 已分析(#07) |

### 4.2 CBS 核心（已分析）

| DLL | 大小 | 运行时 | 状态 |
|-----|------|--------|------|
| wcp.dll | 2,889,216B | Native | 已分析(#13 v2) |
| cbscore.dll | 1,928,192B | Native | 已分析(#14) |
| cabapi.dll | 68,096B | Native | 已分析(#14) |
| cbsapi.dll | 25,088B | Native | 未深析 |
| cabapiwrapper.dll | 18,944B | .NET | 未深析 |
| ConvertDSMDLL.dll | 327,680B | Native | 部分分析(#12) |

### 4.3 包构建框架

| DLL | 大小 | 运行时 | 状态 |
|-----|------|--------|------|
| PkgBldr.Common.dll | 518,656B | .NET | 已分析(#08) |
| PkgBldr.Tools.dll | 71,168B | .NET | 部分分析(#08) |
| PkgBldr.SecurityToolbox.dll | 43,008B | .NET | 未深析 |
| PkgBldr.Plugin.CsiToCab.Base.dll | 135,680B | .NET | 部分分析(#08) |
| PkgBldr.Plugin.CsiToCsi.Finalize.dll | 14,848B | .NET | 未深析 |
| PkgBldr.Plugin.PkgToWm.Base.dll | 68,096B | .NET | 未深析 |
| PkgBldr.Plugin.WmToCsi.Capabilities.dll | 10,752B | .NET | 未深析 |
| PkgBldr.Plugin.WmToCsi.KnobsStore.dll | 82,432B | .NET | 未深析 |
| PkgBldr.Plugin.WmToCsi.OnecorePackageInfo.dll | 9,728B | .NET | 未深析 |
| PkgBldr.Plugin.WmToCsi.PolicyDefinition.dll | 96,256B | .NET | 未深析 |
| PkgBldr.Plugin.WmToCsi.Security.dll | 56,832B | .NET | 未深析 |
| PkgBldr.Plugin.WmToCsi.TestSupport.dll | 6,144B | .NET | 未深析 |
| pkgcommonmanaged.dll | 123,904B | .NET | 部分分析 |
| pkggencommon.dll | 235,520B | .NET | 未深析 |
| pkgcomposition.dll | 93,696B | .NET | 未深析 |
| pkgtoolbox.dll | 72,192B | .NET | 未深析 |
| toolscommon.dll | 132,608B | .NET | 未深析 |
| buildfilterexpressionevaluator.dll | 10,240B | .NET | 部分分析(#08) |
| platformmanifest.dll | 13,312B | .NET | 未深析 |

### 4.4 FFU/存储

| DLL | 大小 | 运行时 | 状态 |
|-----|------|--------|------|
| ffucomponents.dll | 3,289,600B | .NET | 未深析 |
| DeviceLayoutValidation.dll | 139,264B | .NET | 部分分析(#10) |
| wiminterop.dll | 14,848B | .NET | 未深析 |
| wofdeploy.dll | 8,704B | Native | 未深析 |

### 4.5 Feature/CompDB

| DLL | 大小 | 运行时 | 状态 |
|-----|------|--------|------|
| featureapi.dll | 187,904B | .NET | 未深析 |
| MetadataReader.dll | 166,400B | .NET | 未深析 |

### 4.6 AppX

| DLL | 大小 | 运行时 | 状态 |
|-----|------|--------|------|
| appxpackaging.dll | 1,406,768B | Native | 部分分析(#11) |
| appxdeploymentclient.dll | 629,248B | Native | 未深析 |
| appxprovisionpackage.dll | 67,584B | Native | 未深析 |
| appxreg.dll | 29,696B | Native | 未深析 |
| appximaging.dll | 17,408B | .NET | 部分分析(#11) |
| appxcommon.dll | 25,088B | .NET | 未深析 |
| opcservices.dll | 1,327,616B | Native | 未深析 |

### 4.7 CMI (Component Management Infrastructure)

| DLL | 大小 | 运行时 | 状态 |
|-----|------|--------|------|
| cmiaisupport.dll | 1,680,896B | Native | 未深析 |
| cmiadapter.dll | 59,392B | Native | 未深析 |
| cmifw.dll | 77,312B | Native | 未深析 |
| wmicmiplugin.dll | 276,480B | Native | 未深析 |

### 4.8 驱动服务

| DLL | 大小 | 运行时 | 状态 |
|-----|------|--------|------|
| drvstore.dll | 920,576B | Native | 未深析 |
| drupdate.dll | 294,912B | Native | 未深析 |
| DrvServicing.dll | 159,232B | Native | 未深析 |
| repdrvfs.dll | 272,384B | Native | 未深析 |
| DrvPSM.dll | 8,704B | .NET | 未深析 |
| infverif.dll | 291,328B | Native | 未深析 |

### 4.9 WMI

| DLL | 大小 | 运行时 | 状态 |
|-----|------|--------|------|
| wbemcore.dll | 1,424,896B | Native | 未深析 |
| wbemcomn.dll | 385,024B | Native | 未深析 |
| fastprox.dll | 648,192B | Native | 未深析 |
| wbemprox.dll | 28,672B | Native | 未深析 |
| wmiutils.dll | 89,088B | Native | 未深析 |
| mofd.dll | 194,048B | Native | 未深析 |
| mofinstall.dll | 61,440B | Native | 未深析 |

### 4.10 网络

| DLL | 大小 | 运行时 | 状态 |
|-----|------|--------|------|
| NetSetupEngine.dll | 571,904B | Native | 未深析 |
| NetSetupApi.dll | 99,328B | Native | 未深析 |
| NetSetupAI.dll | 114,688B | Native | 未深析 |
| FirewallOfflineAPI.dll | 157,696B | Native | 未深析 |
| ws2_helper.dll | 80,384B | Native | 未深析 |
| winsockai.dll | 65,024B | Native | 未深析 |

### 4.11 离线 Hive/安全

| DLL | 大小 | 运行时 | 状态 |
|-----|------|--------|------|
| offreg.dll | 62,256B | Native | 未深析 |
| offlinelsa.dll | 105,984B | Native | 未深析 |
| offlinesam.dll | 212,480B | Native | 未深析 |
| luainstall.dll | 40,448B | Native | 未深析 |
| keyform.dll | 27,136B | Native | 未深析 |
| mcsfoffline.dll | 33,280B | .NET | 未深析 |
| mvoffline.dll | 38,912B | .NET | 未深析 |
| signinfohelper.dll | 8,704B | Native | 部分分析 |

### 4.12 AI (Answer File) 安装器

| DLL | 大小 | 运行时 | 状态 |
|-----|------|--------|------|
| cleanupai.dll | 12,288B | Native | 未深析 |
| httpai.dll | 18,944B | Native | 未深析 |
| timezoneai.dll | 50,176B | Native | 未深析 |
| grouptrusteeai.dll | 30,208B | Native | 未深析 |

### 4.13 其他功能

| DLL | 大小 | 运行时 | 状态 |
|-----|------|--------|------|
| ufphost.dll | 756,016B | Native | 未深析 |
| ufphostm.dll | 26,112B | .NET | 未深析 |
| esscli.dll | 284,672B | Native | 未深析 |
| TurboStack.dll | 374,272B | Native | 未深析 |
| msdelta.dll | 397,312B | Native | 未深析 |
| dpx.dll | 460,288B | Native | 未深析 |
| wdscore.dll | 192,512B | Native | 未深析 |
| LocBootPresets.dll | 236,544B | Native | 未深析 |
| ImplatSetup.dll | 89,088B | Native | 未深析 |
| EventsInstaller.dll | 181,248B | Native | 未深析 |
| PerfCounterInstaller.dll | 111,616B | Native | 未深析 |
| AriTransformer.dll | 39,936B | Native | 未深析 |
| PrimitiveTransformers.dll | 40,960B | Native | 未深析 |
| WpnDataTransformer.dll | 22,528B | Native | 未深析 |
| ReflectionAdds.dll | 62,976B | .NET | 未深析 |
| TypeSystemMock.dll | 31,232B | .NET | 未深析 |
| Microsoft.Tools.IO.dll | 30,208B | .NET | 未深析 |
| Microsoft.Diagnostics.Tracing.EventSource.dll | 170,800B | .NET | 未深析 |
| Microsoft.Phone.Test.TestMetadata.dll | 58,880B | .NET | 未深析 |
| parsemanifestlite.dll | 766,464B | Native | 未深析 |

### 4.14 运行时垫片（排除，共约100个）

api-ms-win-*.dll — Windows API Set 转发 DLL，仅包含函数转发，无实际实现。
ext-ms-onecore-shlwapi-l1-1-0.dll — 同上。

---

## 5. 工具依赖关系图

### 5.1 镜像构建主流程

```
OEMInput.xml + DeviceLayout.xml + FeatureManifest.xml
    │
    ▼
┌──────────────────┐
│ featuremerger.exe│───→ featureapi.dll, MetadataReader.dll
│ (FM合并)         │
└────────┬─────────┘
         │ 合并后的FM
         ▼
┌──────────────────┐
│ PkgGen.exe       │───→ PkgBldr.*.dll (MEF插件)
│ (包构建调度)     │    pkgcommonmanaged.dll, toolscommon.dll
└────────┬─────────┘
         │ 调用
         ▼
┌──────────────────┐
│ spkggen.exe      │───→ pkggencommon.dll, pkgcomposition.dll
│ (.pkg.xml→.spkg) │
└────────┬─────────┘
         │ .spkg
         ▼
┌──────────────────┐
│ ConvertDSM.exe   │───→ ConvertDSMDLL.dll
│ (.spkg→.cab)     │    wcp.dll (hash/cert验证)
│                  │    makecat.exe (目录生成)
└────────┬─────────┘
         │ .cab (CBS包)
         ▼
┌──────────────────┐
│ imageapp.exe     │───→ imaging.dll, imagestorageservice.dll
│ (镜像构建入口)    │    imagecommon.dll, imagecustomization.dll
└────────┬─────────┘
         │ 调用
         ▼
┌──────────────────┐
│ updateapp.exe    │───→ updatedll.dll, updateapi.dll
│ (两阶段更新)      │    wcp.dll, cbscore.dll, cabapi.dll
└────────┬─────────┘
         │ FFU镜像
         ▼
┌──────────────────┐
│ imagesigner.exe  │───→ HashedChunkReader (分块哈希)
│ (镜像签名)        │
└────────┬─────────┘
         │ 已签名FFU
         ▼
    最终镜像产物
```

### 5.2 辅助工具流程

```
FFU镜像
    ├──→ ffutool.exe (设备刷机) ───→ ffucomponents.dll, Qualcomm.Recovery
    ├──→ ImgDump.exe (元数据转储)
    ├──→ imgtowim.exe (FFU→WIM) ───→ wiminterop.dll
    └──→ wpimage.exe (挂载/管理) ───→ imagecommon.dll

WIM文件
    ├──→ imagex.exe (标准WIM操作)
    ├──→ secwimtool.exe (安全WIM) ───→ makecat.exe (SHA256目录)
    └──→ wpimage.exe (挂载/卸载)

OEM定制
    └──→ OemCustomizationTool.exe ───→ CustomizationPkgBuilder → .spkg/.cab

存储空间
    └──→ rs5_spaceutil.exe ───→ spaceport.sys (Storage Spaces驱动)
```

---

## 6. 已分析文档交叉索引

| 文档# | 标题 | 覆盖工具/DLL |
|-------|------|-------------|
| #00 | 总索引 | 全部 |
| #01 | 工具链机制总览 | imageapp, updateapp, imaging |
| #02 | 架构总览 | 全部 |
| #03 | CAB hash/证书验证(第一版) | wcp, signtool |
| #04 | CAB hash/证书深度研究 | wcp VerifyFileHashes |
| #05 | imaging.dll 深度研究 | imaging.dll |
| #06 | imagestorageservice.dll | imagestorageservice.dll |
| #07 | UpdateDLL.dll 两阶段更新 | updatedll.dll, updateapp.exe |
| #08 | PkgGen 插件架构 | PkgGen, PkgBldr.*.dll |
| #09 | FFU 镜像格式 | FFU格式(通用) |
| #10 | 输入配置格式 | DeviceLayoutValidation, featureapi |
| #11 | AppX 预安装 | makeappx, appxpackaging, appximaging |
| #12 | spkggen+ConvertDSM+pkg2cab | spkggen, ConvertDSM, ConvertDSMDLL |
| #13 | wcp.dll 完整v2 | wcp.dll (IDA 18MB完整版) |
| #14 | cabapi+cbscore CBS核心 | cabapi, cbscore, cbsapi |
| #15 | 本文档 | 全部24 EXE + 全部DLL分类 |

---

## 7. 重建项目优先级

### 7.1 高优先级（镜像构建核心，必须重建）

| 组件 | 原因 | 重建难度 |
|------|------|---------|
| imageapp.exe | 镜像构建入口 | 中（.NET，逻辑清晰） |
| updateapp.exe + updatedll.dll | 两阶段更新核心 | 高（原生，1.5MB） |
| imaging.dll | 镜像处理核心 | 中（已分析#05） |
| PkgGen.exe + spkggen.exe | 包构建 | 中（.NET，已分析#08/#12） |
| ConvertDSM.exe | SPKG→CAB转换 | 中（原生，已分析#12） |
| wcp.dll | hash/cert验证 | 高（原生，2.9MB，已分析#13） |
| ffutool.exe + ffucomponents.dll | FFU操作 | 中（.NET，3.3MB） |

### 7.2 中优先级（重要但可替代）

| 组件 | 替代方案 |
|------|---------|
| makeappx.exe | 标准 Windows SDK 工具 |
| signtool.exe | 标准 Windows SDK 工具 / osslsigncode |
| imagex.exe | DISM / standard WIM API |
| secwimtool.exe | 可重新实现（格式已完全分析） |
| featuremerger.exe | 可重新实现（.NET） |
| imagesigner.exe | 可重新实现（分块哈希+签名） |
| pkgsigntool.exe | signtool 替代 |
| cbscore.dll + cabapi.dll | 最小CBS实现（已分析#14） |

### 7.3 低优先级（辅助工具，可跳过）

| 组件 | 原因 |
|------|------|
| spaceutil.exe (5版本) | Storage Spaces配置，非核心构建 |
| OemCustomizationTool.exe | OEM定制，可选 |
| wpimage.exe | 镜像挂载，可用标准工具替代 |
| ImgDump.exe / imgtowim.exe | 调试/转换工具 |
| devicenodecleanup.exe | 设备清理，非构建必需 |
| WMI DLLs (wbemcore等) | 标准Windows组件 |
| CMI DLLs (cmiaisupport等) | 组件管理，可简化 |
| 驱动服务DLLs (drvstore等) | 驱动管理，离线构建可简化 |
| 网络DLLs (NetSetup等) | 网络配置，可选 |
| AI安装器 (cleanupai等) | Answer File处理，可选 |
| 运行时垫片 (api-ms-win-*) | 系统自带，无需重建 |

### 7.4 关键未深析 DLL 建议后续研究

| DLL | 大小 | 建议研究原因 |
|-----|------|-------------|
| updateapi.dll | 1.9MB | updateapp 的 API 层，与 updatedll 配合 |
| ffucomponents.dll | 3.3MB | FFU 操作核心，ffutool 依赖 |
| parsemanifestlite.dll | 766KB | CBS manifest 解析，wcp 依赖 |
| cmiaisupport.dll | 1.7MB | CMI 核心，离线配置管理 |
| opcservices.dll | 1.3MB | OPC 服务，AppX 依赖 |
| imagecommon.dll | 418KB | 镜像公共库，多个工具依赖 |
| drvstore.dll | 920KB | 驱动存储，驱动包处理 |
| ufphost.dll | 756KB | UFP 主机，未知功能需确认 |

---

*文档结束 — 生成于 2026-08-27，覆盖 ADK 10.0.17704.1000 i386 目录全部 24 个 EXE 和 99 个非运行时 DLL*

*本文档是 ADK 完整工具清单的 v1.0，所有组件均已反编译，其中 15 份深度研究文档覆盖了核心组件。未深析组件的反编译源码均在 outputsrc\ 目录下可供后续研究。*
