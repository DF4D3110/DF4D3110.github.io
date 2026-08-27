# ADK 10.0.17704.1000 深度研究总索引

> **研究目标**: 逆向分析 ADK 10.0.17704.1000 (Windows 10 RS5) 工具链，为重建项目提供完整技术参考
> **研究根目录**: `E:\WSK_Tools\ADK_Research\`
> **文档目录**: `E:\WSK_Tools\ADK_Research\docs\`
> **研究时间**: 2026-08-26 ~ 2026-08-27
> **研究工具**: IDA Pro (原生反编译), ILSpy (.NET 反编译), VS Code (源码分析), PowerShell (PE 导出表解析)

---

## 文档清单（17 份，总计 ~490KB）

### 第一阶段：整体概览

| # | 文档 | 大小 | 内容 |
|---|------|------|------|
| 01 | `01_ADK_17704_toolchain_mechanism_deep_dive.md` | 35.4KB | 整体工具链机制：两条核心流水线（包构建 PkgGen→spkggen→ConvertDSM，镜像构建 imageapp→imaging→UpdateDLL）、关键组件架构、重建策略建议 |
| 02 | `02_ADK_17704_complete_toolchain_analysis_early.md` | 33.9KB | 完整工具链分析（早期版本，与 #01 有重叠但视角不同，保留供参考） |

### 第二阶段：CAB 包 Hash 与证书验证

| # | 文档 | 大小 | 内容 |
|---|------|------|------|
| 03 | `03_ADK_17704_cab_hash_cert_verification_v1.md` | 29.8KB | CAB 包 hash 与证书验证机制 v1（早期版）：ImageSigner 类、4 个硬编码 Microsoft 证书指纹、Package.LoadFromCab、FileEntryBase、PkgManifest.Load_CBS |
| 04 | `04_ADK_17704_cab_hash_cert_deep_dive_v2.md` | 31.8KB | CAB hash/证书深度研究 v2：wcp.dll!CCSDirectTransaction::VerifyFileHashes 完整原生实现、RtlGetHashAlgorithmHashLength 算法 ID 映射、CCatalog 类、三层 hash 链、完整 API 列表、实际 .cat 文件分析 |
| 05 | `05_ADK_17704_wcp_dll_complete_deep_dive_v3.md` | 38.4KB | wcp.dll 完整深度逆向 v3 (基于IDA 18MB完整版)：CCatalog类5个核心方法完整实现、CertCreateCTLContext+CertFindSubjectInCTL+CryptMsgGetAndVerifySigner+CertGetCertificateChain+CertVerifyCertificateChainPolicy完整调用链、CmsVerifyFileHash备用hash回退、ValidateComponentSignature PublicKeyToken匹配、关键OID、完整源码行号索引 |

### 第三阶段：核心 DLL 深度分析

| # | 文档 | 大小 | 内容 |
|---|------|------|------|
| 06 | `06_ADK_17704_imaging_dll_deep_dive.md` | 34.2KB | imaging.dll 深度研究：Imaging 类 ProcessImage 全流程、UpdateMain 完整 P/Invoke 声明（21个 UpdateDLL + 4个 wcp + 1个 Ole32）、OFFLINE_STORE_CREATION_PARAMETERS、StageImage/CommitImage、MinFreeSectors 动态分区公式、EnforcePartitionRestrictions、FinalizeImage FFU wrapper 链 |
| 07 | `07_ADK_17704_imagestorageservice_deep_dive.md` | 19.3KB | imagestorageservice.dll 深度研究：81个导出函数9类分类、CreateVirtualDisk/PartitionVirtualHardDisk/CreateStoragePool/CreateStorageSpace 核心实现、VDS COM API、virtio API、关键 IOCTL、STORE_ID 结构、托管封装层、PARTITION_ENTRY 201字节布局 |
| 08 | `08_ADK_17704_updatedll_deep_dive.md` | 33.5KB | UpdateDLL.dll 两阶段更新机制：116个导出函数完整分类、CUpdateContext 数据结构（0x280+字节布局）、PrepareUpdateWithFlags 完整流程（包验证→空间预留→逐分区文件准备→写历史）、ExecuteUpdate 完整流程（加载UpdateOutput.xml→SBCP检查→核心提交→BCD复制→A/B交换→Point of No Return）、hash验证调用链、存储空间模式、错误码 |
| 09 | `09_ADK_17704_cabapi_cbscore_cbs_core_deep_dive.md` | 13.8KB | cabapi.dll + cbscore.dll CBS核心引擎：cabapi 415函数(WIL框架/IULogger/6种子阶段映射STAGING/COMMIT/RESET/FINALIZE)、cbscore 230+核心类函数(CCbsSession状态机/CCbsSessionManager 0x120字节/CCbsStack XML解析/PackageStore会话ID FILETIME持久化/CCbsLockMonitor死锁检测)、COM-style CCbsIUnknownImpl基类、完整CBS安装流程、与wcp/UpdateDLL交互关系 |

### 第四阶段：包构建流水线

| # | 文档 | 大小 | 内容 |
|---|------|------|------|
| 10 | `10_ADK_17704_pkggen_plugin_architecture_deep_dive.md` | 30.0KB | PkgGen 插件架构与包构建流水线：MEF 插件体系（IPkgPlugin/PkgPlugin/PkgBldrLoader）、6种 PluginType、宏系统（5层宏表）、BuildPackage 核心流程、pkg2cab 完整流水线（PkgGen→spkggen→ConvertDSM）、Wow64交叉编译、卫星构建（语言/分辨率）、BuildFilter表达式计算器、INI Manifest格式、命令行参数 |
| 11 | `11_ADK_17704_spkggen_convertdsm_pkg2cab_deep_dive.md` | 36.5KB | spkggen.exe + ConvertDSM.exe + pkg2cab 流水线深度逆向：spkggen .NET 完整逆向（Program.Main/MergingSchemaValidator/MEF插件加载/19个命令行参数）、ConvertDSM native 完整逆向（RunConvertDSM wmain/14个单字符标志/AddFileEntry CBS manifest生成/UpdateCatAndSign双工具路径）、FNV-1a文件名哈希、SHA256 asmv2:hash格式、boot文件不压缩规则、makecat.exe vs updcat.exe、完整源码行号索引 |

### 第五阶段：镜像格式与输入配置

| # | 文档 | 大小 | 内容 |
|---|------|------|------|
| 12 | `12_ADK_17704_ffu_image_format_deep_dive.md` | 29.2KB | FFU 镜像格式深度研究：完整文件物理布局（SecurityHeader→Catalog→HashTable→Padding→ImageHeader→Manifest→Payload）、SecurityHeader/ImageHeader 结构体、三层 hash 链（Catalog→HashTable→Payload chunks）、INI Manifest 格式、存储层次结构（Image→StoragePool→Store→Partition）、SecurityPadding 计算、FFUFlashException 错误码、最小 FFU 生成器伪代码 |
| 13 | `13_ADK_17704_input_config_formats_deep_dive.md` | 27.2KB | 输入配置格式深度研究：DeviceLayout (ImageGeneratorParameters/InputStore/InputPartition)、OEMInput (产品配置/Feature选择/语言分辨率)、FeatureManifest (Feature→包映射)、CompDB (BuildCompDB/BSPCompDB/CompDBPackageInfo/CompDBSatelliteInfo)、四层配置数据流、验证规则(InputRules)、关键分区类型GUID、三种分区大小模式 |
| 14 | `14_ADK_17704_appx_preinstallation_deep_dive.md` | 19.6KB | AppX 预安装机制：两阶段模型（Stage用MakeAppx解压/Commit复制+注册表）、appxpackaging.dll COM API（IAppxFactory/IAppxBundleFactory/IAppxManifestReader等）、AppxInfoManager包管理、架构过滤、Bundle处理、依赖排序、许可证复制、RegLoadAppKey离线注册表hive、WindowsApps/AppData目录结构、与CBS包的协同关系 |

### 第六阶段：工具清单与 spaceutil

| # | 文档 | 大小 | 内容 |
|---|------|------|------|
| 15 | `15_ADK_17704_complete_tool_inventory_and_analysis.md` | 28.9KB | ADK完整工具清单与分析：全部24个EXE分类详解(镜像构建/包构建/FFU/WIM/存储/Feature/签名/AppX/OEM/设备清理)、secwimtool.exe完整逆向(.secwim格式=SDI+WIM+平台ID+序列号+SHA256目录)、ffutool设备状态机、spaceutil 5版本(Storage Spaces)、99个非运行时DLL分类(14大类)、完整工具依赖关系图、重建项目优先级(高/中/低) |
| 16 | `16_ADK_17704_spaceutil_and_remaining_tools_deep_dive.md` | 31.7KB | spaceutil.exe深度逆向+剩余工具完整分析：rs5_spaceutil.exe wmain完整还原(38命令/命令表结构)、53个IOCTL码完整列表(Spaceport 0xE7/VolMgr 0x70)、3个设备路径、注册表交互、参数类型(Boolean/Guid/String/Number/Size/Auto)、5版本差异、剩余12个EXE详解、剩余.NET DLL和Native DLL分类分析 |
| 17 | `17_ADK_17704_spaceutil_five_versions_deep_comparison.md` | 30.7KB | spaceutil.exe五版本(rs1-rs5)深度对比：38个Su*命令完整还原(7大类)、9个对象查询类型、53个IOCTL码100%跨版本相同(46 Spaceport+5 VolMgr+4其他)、IOCTL码格式解析、3个设备路径交互原理、驱动类型选择(spaceport/clusdisk)、版本演化(函数数216→303/+40%、rs3开始命令名混淆)、完整工作流程、关键数据结构推断 |

---

## 关键发现速查

### 核心数据

| 项目 | 值 |
|------|-----|
| ADK 版本 | 10.0.17704.1000 (Windows 10 RS5) |
| wcp.dll 导出函数 | 443 个 |
| updatedll.dll 导出函数 | 116 个 |
| imagestorageservice.dll 导出函数 | 81 个 |
| signinfohelper.dll 导出函数 | 1 个 (IsSignInfoRequired) |
| FFU 格式版本 | 2.0 |
| 默认 FFU ChunkSize | 256 KB |
| 默认分区对齐 | 65536 字节 (64KB) |
| 默认扇区大小 | 512 字节 |
| spaceutil 命令数 | 38 个 |
| spaceutil IOCTL 数 | 53 个（跨版本100%相同） |

### 关键常量

| 常量 | 值 | 来源文档 |
|------|-----|---------|
| CBS_E_CORRUPT_FILE | 0xc015001b | #04 |
| HashAlgorithmID SHA256 | 3 | #04 |
| 存储池分区 Type GUID | {5708A6E0-9001-4b99-b064-1fe564896bdb} | #07, #12 |
| imagestorageservice magic | -0x2c691091 | #07 |
| 全局互斥锁 | Global\VHDMutex_{585b0806-2d3b-4226-b259-9c8d3b237d5c} | #07 |
| FFU 安全签名 | "SignedImage " (12B) | #12 |
| FFU 镜像签名 | "ImageFlash  " (12B) | #12 |
| CALG_SHA_256 | 32780 | #13 |
| GPT 基本数据分区 | {ebd0a0a2-b9e5-4433-87c0-68b6b72699c7} | #13 |

### 4 个硬编码 Microsoft 证书指纹

| 证书 | 指纹 |
|------|------|
| ProdCertRoot | 3B1EFD3A66EA28B16697394703A72CA340A05BD5 |
| TestCertRoot | 8A334AA8052DD244A647306A76B8178FA215F344 |
| FlightCertPCA | 9E594333273339A97051B0F82E86F266B917EDB3 |
| FlightCertWindows | 5f444a6740b7ca2434c7a5925222c2339ee0f1b7 |

---

## 源码路径索引

### .NET 反编译源码 (`outputsrc\dotnet\`)

| 文件 | 大小 | 主要内容 |
|------|------|---------|
| imaging.cs | 148KB | Imaging 类、UpdateMain P/Invoke、AppXImaging 引用、ProcessImage 全流程 |
| imagecommon.cs | 474KB | FFU 格式 (FullFlashUpdateImage/SecurityHeader/ImageHeader)、CompDB、DeviceLayout、ImageSigner、Package 类 |
| pkgcommonmanaged.cs | 429KB | Package.LoadFromCab、FileEntryBase、PkgManifest、CBS 包管理 |
| imagestorageservicemanaged.cs | 416KB | ImageStorageManager 托管封装层 |
| ffucomponents.cs | 176KB | FFU 设备刷写管理 (FFUManager/FFUDevice) |
| PkgGen.cs | 35KB | PkgGen 主入口、BuildPackage、PkgToCab |
| PkgBldr.Common.cs | 192KB | PkgBldrLoader、IPkgPlugin、PkgPlugin、PluginType、插件实现 |
| pkggencommon.cs | 208KB | PkgPlugin 抽象类、Config、SatelliteId、MacroResolver |
| pkgtoolbox.cs | 253KB | 包工具函数 |
| DeviceLayoutValidation.cs | 89KB | DeviceLayout XML 验证 |
| appximaging.cs | ~20KB | AppXImaging 类、StageAppXPackages、CommitAppXPackages |
| spkggen.cs | 18.4KB | spkggen.exe 主入口：Program.Main (19个命令行参数)、MergingSchemaValidator、LoadPackagePlugins (MEF) |
| featureapi.cs | - | OEMInput、FeatureManifest、FMCollection |

### 原生反编译源码 (`outputsrc\native\`)

| 文件 | 大小 | 主要内容 |
|------|------|---------|
| wcp.c | 13MB | CBS 核心：CCSDirectTransaction::VerifyFileHashes、CCatalog、RtlGetHashAlgorithmHashLength |
| updatedll.c | 6.5MB | UpdateDLL：PrepareUpdateWithFlags、ExecuteUpdate、CreateUpdateContext |
| imagestorageservice.c | - | 存储服务：CreateVirtualHardDisk、PartitionVirtualHardDisk |
| updateapp.c | 3.1MB | updateapp.exe |
| cbscore.c | 9.1MB | CBS 核心原生实现 |
| ConvertDSM.c | 1.6MB/53376行 | ConvertDSM.exe：RunConvertDSM (wmain)、AddFileEntry (CBS manifest生成)、UpdateCatAndSign |
| ConvertDSMDLL.c | 1.4MB | ConvertDSM DLL 导出函数 |
| rs{1-5}_spaceutil.c | 374-411KB each | spaceutil.exe 五版本反编译 |

### IDA 反编译源码 (`ida_decompiled\native\`)

| 文件 | 大小 | 说明 |
|------|------|------|
| wcp.c | 18.9MB | IDA 更完整的 wcp.dll 反编译 |

### 原始 ADK 工具 (`SourceDir\Windows Kits\10\tools\bin\i386\`)

| 工具 | 大小 | 用途 |
|------|------|------|
| imageapp.exe | 16896B | 镜像构建主入口 |
| updateapp.exe | 736256B | 两阶段更新应用 |
| PkgGen.exe | 28160B | 包构建前端 |
| spkggen.exe | 19968B | .pkg.xml → .spkg，MEF插件加载 |
| ConvertDSM.exe | 354816B | .spkg → .cab |
| ffutool.exe | 45056B | FFU 刷写工具 |
| makeappx.exe | 414000B | AppX 打包/解包（标准SDK） |
| signtool.exe | 315696B | 签名工具（标准SDK） |
| pkgsigntool.exe | 25088B | 包签名工具 |
| imagesigner.exe | 12800B | FFU 签名工具 |
| secwimtool.exe | 3.1MB | 安全 WIM 工具 |
| rs{1-5}_spaceutil.exe | 105-125KB | 存储空间工具 (5个版本) |
| imagex.exe | 657200B | WIM 工具（标准AIK） |
| imggen.cmd | 2505B | 镜像构建脚本（封装imageapp） |
| customizationgen.cmd | 2461B | 定制化生成脚本 |
| sign.cmd | 25960B | 签名脚本 |
| pkggen.cfg.xml | 22904B | PkgGen 宏定义配置 |
| LocaleInfoTable.xml | 38825B | 区域信息表 |

---

## 重建项目路线图建议

### 阶段 1：基础框架（优先级最高）

1. **FFU 生成器** — 基于文档 #12，实现最小 FFU 写入器（SecurityHeader + ImageHeader + Manifest + Payload）
2. **DeviceLayout 解析器** — 基于文档 #13，解析 DeviceLayout XML → ImageGeneratorParameters
3. **VHD 创建与分区** — 基于文档 #07，用 virtio API (OpenVirtualDisk/CreateVirtualDisk/AttachVirtualDisk) + VDS COM (IVdsDisk::PartitionDiskEx) 创建分区 VHD

### 阶段 2：包处理

4. **CBS 包解析器** — 基于文档 #03/#04，解析 .cab 中的 update.mum + update.cat + payload，实现 hash 验证
5. **PkgGen 简化版** — 基于文档 #10，实现 pkg2cab 流水线（可先用 ConvertDSM.exe 作为后端）
6. **离线注册表操作** — 集成 offreg.dll，操作注册表 hive

### 阶段 3：镜像构建核心

7. **UpdateDLL 替代层** — 基于文档 #08，实现两阶段更新（PrepareUpdate 解压+验证，ExecuteUpdate 文件替换+注册表+BCD）
8. **imaging.dll 替代层** — 基于文档 #06，实现 ProcessImage 编排（分区→暂存→提交→FFU封装）
9. **AppX 预安装** — 基于文档 #14，实现 Stage (MakeAppx 解压) + Commit (复制+注册表hive)

### 阶段 4：完善

10. **存储空间支持** — 基于文档 #07，实现 Storage Pool + Storage Space
11. **A/B 分区交换** — 基于文档 #08，实现 GPT 分区表交换
12. **签名集成** — 基于文档 #03/#04/#05，集成 imagesigner.exe + signtool.exe
13. **spaceutil 替代** — 基于文档 #16/#17，实现 Storage Spaces 配置（或直接用 PowerShell cmdlets）

---

## 跨设备继续研究指南

### 文件位置
- 所有文档在 `E:\WSK_Tools\ADK_Research\docs\`
- 反编译源码在 `E:\WSK_Tools\ADK_Research\outputsrc\` (dotnet/ + native/)
- IDA 反编译在 `E:\WSK_Tools\ADK_Research\ida_decompiled\native\`
- 原始 ADK 文件在 `E:\WSK_Tools\ADK_Research\SourceDir\`
- DLL 集合在 `E:\WSK_Tools\ADK_Research\ADK\`

### 继续研究的入口点

| 目标 | 从哪里开始 |
|------|-----------|
| 深入 wcp.dll hash 验证 | `outputsrc\native\wcp.c` 搜索 `VerifyFileHashes`，或 `ida_decompiled\native\wcp.c` (18.9MB 更完整) |
| 深入 UpdateDLL 内部函数 | `outputsrc\native\updatedll.c`，关键行号见文档 #08 末尾索引 |
| 深入 spaceutil 内部 | `outputsrc\native\rs5_spaceutil.c`，命令表在 wmain 中，IOCTL 码见文档 #17 |
| 研究 appxpackaging.dll | COM API，可用 OLE/COM Object Viewer 查看类型库 |
| 研究 imggen.cmd 流程 | `SourceDir\Windows Kits\10\tools\bin\i386\imggen.cmd` |
| 研究 pkggen.cfg.xml | `SourceDir\Windows Kits\10\tools\bin\i386\pkggen.cfg.xml` |

---

## 分析完成状态确认

> 截至 2026-08-27，ADK i386 目录全部 233 个文件 + en-us 子目录已完成分析。

| 文件类型 | 总数 | 已分析 | 排除(运行库) | 说明 |
|---------|------|--------|-------------|------|
| EXE | 24 | 24 | 0 | 全部深度分析（含5个spaceutil版本） |
| 功能DLL | 98 | 98 | 0 | 全部按14大类分类分析 |
| 运行时垫片DLL | 101 | 0 | 101 | api-ms-win-*/ext-ms-*，用户要求排除 |
| CMD | 5 | 5 | 0 | imggen/customizationgen/sign/installoemcerts/makeoemcerts |
| XML | 2 | 2 | 0 | LocaleInfoTable(区域信息表) + pkggen.cfg(宏定义) |
| Manifest | 2 | 2 | 0 | AppxPackaging(9个COM类) + OpcServices(1个COM类) |
| IDA数据库 | 1 | 0 | 1 | spkggen.exe.i64，非二进制 |
| 资源MUI | 1 | 0 | 1 | en-us/ufphost.dll.mui(2KB) |
| **合计** | **234** | **131** | **103** | **功能文件100%分析完成** |

### 24个EXE分析状态

| EXE | 分析文档 | 深度 |
|-----|---------|------|
| imageapp.exe | #01, #06 | 深 |
| updateapp.exe | #01, #08 | 深 |
| PkgGen.exe | #10, #11 | 深 |
| spkggen.exe | #11 | 深 |
| ConvertDSM.exe | #11 | 深 |
| imaging.dll | #06 | 深 |
| ffutool.exe | #15, #16 | 中 |
| ImgDump.exe | #16 | 中 |
| imgtowim.exe | #16 | 中 |
| wpimage.exe | #16 | 中 |
| featuremerger.exe | #16 | 中 |
| imagesigner.exe | #16 | 中 |
| pkgsigntool.exe | #16 | 中 |
| OemCustomizationTool.exe | #16 | 中 |
| secwimtool.exe | #15 | 深 |
| rs1-5_spaceutil.exe | #16, #17 | 深 |
| imagex.exe | #15 | 浅(标准工具) |
| signtool.exe | #15 | 浅(标准工具) |
| makeappx.exe | #15 | 浅(标准工具) |
| devicenodecleanup.x86/x64 | #16 | 浅 |

### 98个功能DLL分析状态

全部在 #15, #16 中按14大类分类分析，核心DLL(wcp/updatedll/imagestorageservice/cbscore/cabapi/parsemanifestlite)在 #04, #05, #06, #07, #08, #09 中深度分析。

---

*总索引最后更新: 2026-08-27*
*研究文档总数: 17 份*
*总研究内容: ~490 KB*
*ADK 版本: 10.0.17704.1000 (Windows 10 RS5)*
*功能文件分析完成率: 100% (131/131，排除103个运行库/资源文件)*
