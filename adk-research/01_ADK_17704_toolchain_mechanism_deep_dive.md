# ADK 10.0.17704.1000 工具链机制深度研究报告

> 生成日期：2026-08-27
> 研究范围：imageapp / updateapp / pkggen / spkggen / imaging / imagestorageservice / UpdateDLL 等核心组件
> 数据来源：ILSpy 反编译 (.NET) + IDA Pro Hex-Rays 反编译 (Native)

---

## 一、整体架构总览

ADK 17704 是 Windows 10 RS5 (1809) 的评估和部署工具包，核心目标是**从包定义 (.pkg.xml) 到可刷写镜像 (.ffu/.vhd/.vhdx) 的完整构建流水线**。

### 1.1 三层架构

```
┌─────────────────────────────────────────────────────────┐
│  入口层 (Entry Points)                                    │
│  imageapp.exe | PkgGen.exe | updateapp.exe | spkggen.exe │
│  ffutool.exe | makeappx.exe | ConvertDSM.exe | ...       │
├─────────────────────────────────────────────────────────┤
│  编排层 (Orchestration .NET)                              │
│  imaging.dll | imagecommon.dll | imagestorageservicemanaged.dll │
│  PkgBldr.Common.dll | pkggencommon.dll | ffucomponents.dll    │
├─────────────────────────────────────────────────────────┤
│  原生核心层 (Native Core)                                 │
│  UpdateDLL.dll | imagestorageservice.dll | wcp.dll       │
│  cbscore.dll | updateapi.dll | updatedll.dll | wdscore.dll │
│  appxpackaging.dll | opcservices.dll | drvstore.dll      │
└─────────────────────────────────────────────────────────┘
```

### 1.2 两条核心流水线

| 流水线 | 输入 | 输出 | 主导工具 |
|--------|------|------|----------|
| **包构建流水线** | .pkg.xml 源文件 | .cab 离线包 | PkgGen → spkggen → ConvertDSM |
| **镜像构建流水线** | .cab 包 + OEMInput.xml + DeviceLayout.xml | .ffu/.vhd/.vhdx 镜像 | imageapp → imaging → UpdateDLL |

---

## 二、镜像构建流水线（imageapp 机制）

### 2.1 imageapp.exe 入口

**文件信息**：.NET Framework 4.0 程序集，16,896 字节，命名空间 `Microsoft.WindowsPhone.ImageUpdate`

**命令行格式**：
```
imageapp <OutputFile> <OEMInputXML> <MSPackageRoot> [选项]
```

**关键选项**：
| 选项 | 说明 |
|------|------|
| `/CPUType:<arch>` | x86 / ARM / ARM64 / AMD64 |
| `+StrictSettingPolicies` | 严格设置策略（无策略的设置报错） |
| `/Recovery` | 创建恢复 FFU 而非完整 FFU |
| `/OEMCustomizationXML:<path>` | OEM 定制 XML |
| `/OEMCustomizationPPKG:<path>` | OEM 定制 PPKG |
| `/OEMVersion:<ver>` | 定制版本号 (major.minor.submajor.subminor) |
| `/StagingVHDPath:<path>` | 预生成暂存 VHD |
| `/MountAndInstall:<list>` | 更新模式：分号分隔的安装项列表 |
| `+RandomizeGptIDs` | 随机化 GPT ID（支持并行运行） |
| `+SkipImageCreation` | 仅生成定制包，不生成镜像 |

**入口调用链**：
```
ImageApp.Main()
  → InitializeImaging()
      → SetCmdLineParams()  [定义所有命令行参数]
      → ParseCommandlineParams()
      → CreateIULogger()  [创建日志，配置 CBS 日志]
      → new Imaging(logger, skipImaging, formatDPP, strictPolicies, skipUpdateMain, cpuId, bspProductName, chunkMapping)
  → BeginImaging(imaging)
      → 新建模式: imaging.BuildNewImage(...)
      → 更新模式: imaging.UpdateExistingImage(...)
```

### 2.2 imaging.dll — 核心编排器

**文件信息**：.NET 程序集，114,688 字节，核心类 `Microsoft.WindowsPhone.Imaging.Imaging`

#### 2.2.1 主控流程 ProcessImage()

```
ProcessImage(randomizeGptIds, recovery)
  │
  ├─ PrepareImaging()
  │   ├─ SetPaths()  [设置输出路径、临时目录、日志文件]
  │   ├─ 确定输出类型: .ffu → FFU模式(默认ARM) / .vhd/.vhdx → VHD模式(默认x86)
  │   └─ 遥测日志初始化
  │
  ├─ CreateFullImage()  [新建镜像] / CreateStagingImage() [暂存VHD]
  │   │
  │   ├─ [StagingVHD模式] 挂载差异盘 + 反序列化暂存数据
  │   │
  │   ├─ SelectPackagesToImage()  [包选择]
  │   │   ├─ OEMInput.ValidateInput()  [验证OEMInput.xml]
  │   │   ├─ ProcessFMs()  [处理Feature Manifest]
  │   │   │   ├─ 从核心FM包中提取FM XML
  │   │   │   ├─ 遍历所有AdditionalFM
  │   │   │   ├─ GetFilteredPackagesByGroups()  [按组过滤包]
  │   │   │   ├─ FMAddOnDrivers.GetFilteredDriverPackages()
  │   │   │   ├─ FMAddOnAppX.GetFilteredAppXPackages()
  │   │   │   └─ ProcessCompDBPackages()  [构建CompDB]
  │   │   ├─ GenerateInputFile()  [生成UpdateInput.xml]
  │   │   ├─ GenerateCustomizationContent()  [生成OEM定制包]
  │   │   └─ ValidateProductionImage()  [生产镜像验证]
  │   │
  │   ├─ ReadDeviceLayout()  [读取设备布局]
  │   │   ├─ 从包中提取 DeviceLayout.xml + OEMDevicePlatform.xml
  │   │   ├─ DeviceLayoutValidator.ValidateDeviceLayout()
  │   │   └─ ImageGeneratorParameters.ProcessInputXML()
  │   │
  │   ├─ WriteImageMetadataFiles()
  │   │   ├─ WriteProtoSystemManifest()  [原型系统清单]
  │   │   └─ WriteCompDBs()  [BSPCompDB + DeviceCompDB]
  │   │
  │   ├─ PopulateStagingImage()  [暂存镜像填充]
  │   │   ├─ ImageStorageManager 创建暂存VHD
  │   │   ├─ InitializeMinFreeSectors()
  │   │   └─ StageImage()  [核心：CBS暂存]
  │   │       ├─ 创建 UpdateOS.wim (LZX压缩)
  │   │       ├─ UpdateMain.Initialize(storeIds, UpdateInput.xml, stagingRoot, log)
  │   │       ├─ UpdateMain.PrepareUpdate()  → UpdateDLL!PrepareUpdateWithFlags
  │   │       └─ AppXImaging.StageAppXPackages()
  │   │
  │   └─ CommitStagedImage()  [提交镜像]
  │       ├─ ProcessMinFreeSectors()  [计算分区实际大小]
  │       ├─ ImageStorageManager 创建最终VHD/FFU结构
  │       ├─ EnforcePartitionRestrictions()  [分区限制检查]
  │       ├─ CommitImage()  [核心：CBS提交]
  │       │   ├─ UpdateMain.Initialize(...)
  │       │   ├─ UpdateMain.SetPoolId(poolGuid)  [存储池ID]
  │       │   ├─ AppXImaging.CommitAppXPackages()
  │       │   └─ UpdateMain.ExecuteUpdate()  → UpdateDLL!ExecuteUpdate
  │       ├─ ProcessBSPProductNameAndVersion()
  │       ├─ LoadDataAssets()
  │       └─ FinalizeImage()  [最终化]
  │           ├─ Format DPP分区 (如需)
  │           ├─ [FFU模式] 构建FFU包装:
  │           │   ├─ OutputWrapper → SecurityWrapper → ManifestWrapper
  │           │   ├─ ImageStorageManager.DismountFullFlashImage()
  │           │   └─ 生成 .cat 目录文件
  │           └─ [VHD模式] DismountVirtualHardDisk()
  │
  └─ finally: CleanupHandler() + ReportMetrics()
```

#### 2.2.2 两阶段更新机制（核心）

imageapp **不直接调用 updateapp.exe**，而是通过 `UpdateMain` 类直接 P/Invoke `UpdateDLL.dll`。

**暂存阶段 (PrepareUpdate)**：
```csharp
// imaging.dll → UpdateDLL.dll
UpdateMain.Initialize(storeIdsCount, STORE_ID[], UpdateInput.xml, stagingPath, logPath)
UpdateMain.PrepareUpdate()  // 或 PrepareStagingVHDUpdate()
```
- 读取 UpdateInput.xml 中的包列表
- 将所有 .cab 包解压到 `UpdateStagingRoot\<Partition>\TempSxS\`
- 解析依赖关系、计算文件大小
- 生成 Pending_UpdateOS.wim（包含离线更新操作系统）
- **不修改实际分区**，所有操作在暂存目录中完成

**提交阶段 (ExecuteUpdate)**：
```csharp
UpdateMain.Initialize(storeIdsCount, STORE_ID[], UpdateInput.xml, stagingPath, logPath)
UpdateMain.SetPoolId(poolGuid)  // 存储池场景
UpdateMain.ExecuteUpdate()  // 或 ExecuteStagingVHDUpdate()
```
- 将暂存内容实际应用到离线 Windows 镜像的各个分区
- 通过 wcp.dll (Windows Component Platform) 执行组件服务
- 挂载离线注册表配置单元 (SOFTWARE, SYSTEM, SECURITY, SAM, COMPONENTS)
- 应用驱动、注册表、文件、权限等
- 生成 UpdateHistory.xml 和 UpdateOutput.xml

#### 2.2.3 MinFreeSectors 动态分区调整

```
InitializeMinFreeSectors()  [暂存前]
  → 每个有MinFreeSectors要求的分区: TotalSectors = (MinFreeSectors + Overhead) × 3.5
  → MainOS分区额外 +1500MB

ProcessMinFreeSectors()  [暂存后、提交前]
  → IU_GetDirectorySize(stagingRoot\Partition)  [实际暂存文件大小]
  → GetFileSystemOverhead()  [文件系统开销]
  → TotalSectors = AlignUp(文件大小) + AlignUp(开销) + MinFreeSectors + GeneratedOverhead
  → MainOS分区额外 ×1.04 (4%余量)
  → 按1MB对齐
```

### 2.3 imagestorageservice.dll — 存储层

**文件信息**：原生 DLL，286,720 字节（32位），通过 `imagestorageservicemanaged.dll`（.NET 封装层，267,776 字节）调用

#### 2.3.1 三种存储类型

| 类型 | 创建方法 | 用途 |
|------|----------|------|
| **StorageStore** | `ImageStorage.CreateStore()` | 普通物理磁盘（含普通分区+可选存储池分区） |
| **StoragePool** | `ImageStorage.CreatePool()` | 纯存储池成员磁盘（只有一个StoragePool分区） |
| **StorageSpace** | `ImageStorage.CreateSpace()` | 存储池中的虚拟磁盘（存储空间） |

#### 2.3.2 CreateFullFlashImage 精确流程

```
遍历所有 Store（物理磁盘）:
  if (只有一个StoragePool分区):
      ImageStorage.CreatePool()
      → CreateVirtualHardDisk()
      → CreateOrAddToStoragePool()
  else:
      ImageStorage.CreateStore()
      → CreateVirtualHardDisk()
      → PartitionDisk()  [写入GPT分区表]
      → if (有StoragePool分区): CreateOrAddToStoragePool()

遍历所有 StoragePool:
  遍历 Pool 中的每个 Space:
      ImageStorage.CreateSpace()
      → CreateStorageSpace()  [返回diskHandle]
      → PartitionDisk()  [对存储空间分区]
      → if (最后分区UseAllSpace): ExtendPartition()
```

#### 2.3.3 关键原生 API 签名

```c
// 服务管理
int CreateImageStorageService(
    out IntPtr serviceHandle,
    LogFunction logError,
    uint storeIdsCount,
    STORE_ID[] storeIds);

void CloseImageStorageService(IntPtr service);

// 虚拟磁盘
int CreateVirtualHardDisk(
    IntPtr service, string fileName, ulong maxSizeInBytes,
    STORE_ID storeId, uint sectorSize, out IntPtr diskHandle);

int OpenVirtualHardDisk(
    IntPtr service, string fileName, bool readOnly,
    out STORE_ID storeId, out IntPtr diskHandle);

int DismountVirtualHardDisk(
    IntPtr service, STORE_ID storeId, bool deleteFile, bool fFailIfDiskMissing);

// 存储池
int CreateStoragePool(
    IntPtr service, IntPtr storeHandle, string poolName, ref Guid poolId);

int SetStoragePoolName(IntPtr service, Guid poolId, string poolName);

int AddDriveToStoragePool(IntPtr service, Guid poolId, IntPtr storeHandle);

// 存储空间 ★关键
int CreateStorageSpace(
    IntPtr service, Guid poolId, string spaceName, string spaceDescription,
    uint capacityInGB, ref Guid spaceId, out IntPtr diskHandle);
// 返回的 diskHandle 可直接传给 PartitionVirtualHardDisk

// 分区（可用于物理磁盘和存储空间）
int PartitionVirtualHardDisk(
    IntPtr service, IntPtr diskHandle, ref STORE_ID storeId,
    PARTITION_ENTRY[] partitions, uint partitionCount);

int ExtendPartition(
    IntPtr service, SafeFileHandle diskHandle, STORE_ID storeId,
    string partitionName, bool extendVolume);
```

#### 2.3.4 关键结构体

**PARTITION_ENTRY（201字节，Explicit布局）**：
| 偏移 | 字段 | 大小 | 说明 |
|------|------|------|------|
| 0 | name | 72字节 | 分区名称（36字符UTF-16） |
| 72 | sectorCount | 8字节 | 分区大小（扇区数），uint.MaxValue=UseAllSpace |
| 80 | alignmentSizeInBytes | 4字节 | 对齐大小 |
| 84 | clusterSize | 4字节 | 簇大小 |
| 88 | fileSystem | 64字节 | 文件系统（32字符UTF-16） |
| 152 | id / mbrFlags | 16字节 | 分区唯一GUID / MBR标志联合体 |
| 168 | type / mbrType | 16字节 | 分区类型GUID / MBR类型联合体 |
| 184 | flags | 8字节 | GPT属性标志 |
| 192 | offsetInSectors | 8字节 | 偏移（扇区数） |
| 200 | fFvePrep | 1字节 | BitLocker准备标志 |

**STORE_ID（GPT/MBR联合体）**：
| 偏移 | 字段 | 说明 |
|------|------|------|
| 0 | storeType | 0=MBR, 1=GPT |
| 4 | storeId_GPT / storeId_MBR | GPT磁盘GUID / MBR磁盘签名 |

**GPT分区标志**：
- Hidden = 0x4000000000000000
- ReadOnly = 0x1000000000000000
- NoDriveLetter = 0x8000000000000000
- ServicePartition = 0x200000000000000

**存储池分区 Type GUID**：`{5708A6E0-9001-4b99-b064-1fe564896bdb}`

#### 2.3.5 SpaceUtil.exe 机制

ADK 包含 5 个版本的 spaceutil（rs1~rs5），根据目标系统 Build 号选择：
- `new-pool -Name <name> -DriveNumber <n>` → 创建存储池
- `new-space -poolId <guid> -name <name> -ProvisionedCapacity <size> -ProvisioningType Thin -ResiliencyType Simple` → 创建存储空间
- imageapp 使用 **Thin Provisioning + Simple Resiliency**

---

## 三、更新机制（updateapp / UpdateDLL）

### 3.1 updateapp.exe — 独立更新工具

**文件信息**：原生可执行文件（32位），独立命令行工具

**支持的命令**：
| 命令 | 说明 |
|------|------|
| `install` | 完整安装（暂存+提交），支持 `noreboot` / `migratedata` |
| `mountandinstall <image> <packages...>` | 挂载镜像并安装包 |
| `stage` | 仅暂存阶段 |
| `stageinproc` | 进程内暂存 |
| `commit` | 仅提交阶段 |
| `cleanup` | 清理所有更新文件 |
| `postrebootcommit` | 重启后提交 |
| `installinprocnoreboot` | 进程内安装不重启 |
| `renamepackages <path>` | 重命名包 |
| `getinstalledpackages` | 获取已安装包列表 |
| `loadlibrary <path>` | 加载库 |
| `compress <path> [compress/decompress] [recursive]` | 压缩/解压 |
| `getlatestpayload <a> <b> <c> [canonical]` | 获取最新payload |
| `GetUpdateResults` | 获取更新结果状态 |
| `decompressmanifests <a> <b>` | 解压清单 |
| `dumpfilesystem <a> <b>` | 转储文件系统 |
| `captureuvmstate <a> <b>` | 捕获UVM状态 |
| `generateuvmreport <a> <b> <c>` | 生成UVM报告 |
| `reset <args...>` | 重置 |

**更新结果状态**：
- 1 = StagingFailed（暂存失败）
- 2 = StagingComplete（暂存完成）
- 3 = CommitFailed（提交失败）
- 4 = UpdateComplete（更新完成）
- 5 = NoUpdateOutputStateSet（无状态）

### 3.2 UpdateDLL.dll — 更新核心库

**被调用方式**：imaging.dll 中的 `UpdateMain` 类通过 P/Invoke 直接调用

**核心 API**：
```c
// 上下文管理
IntPtr CreateUpdateContext();
int ReleaseUpdateContext(IntPtr ctx);
int Initialize(IntPtr ctx, int storeIdsCount, STORE_ID[] storeIds,
    string UpdateInputFile, string AlternateStagingLocation, string LogFilePath);
void Deinitialize(IntPtr ctx);

// 两阶段更新
int PrepareUpdateWithFlags(IntPtr ctx, uint Flags);
  // Flags: 0=普通, 2=CreatingStagingVHD

int ExecuteUpdate(IntPtr ctx, uint Flags);
  // Flags: 0=普通, 1=UsingStagingVHD, 2=ResetCommit

// 存储池支持
void SetPoolId(IntPtr ctx, Guid PoolId);

// 进度报告
void IU_InitializeDefaultProgressReporting(IUPhase Phase);
  // IUPhase_Staging=0, IUPhase_Commit=1

// 性能统计
uint GetStageFilesTickCount(IntPtr ctx);
uint GetExecuteUpdateOSTickCount(IntPtr ctx);

// 文件大小
ulong GetCompressedFilesize(string file);
ulong GetUncompressedFilesize(string file);
```

### 3.3 CBS 服务栈（wcp.dll / cbscore.dll / updateapi.dll）

UpdateDLL 内部依赖 Windows Component Based Servicing 栈：

```
UpdateDLL.dll
  ├─ wcp.dll (Windows Component Platform)
  │   ├─ WcpInitialize() / WcpShutdown()
  │   ├─ CreateNewWindows()  [创建离线Windows实例]
  │   └─ SetIsolationIMalloc()
  │
  ├─ cbscore.dll (CBS Core)
  │   └─ 组件服务核心逻辑
  │
  ├─ updateapi.dll (Update API)
  │   └─ 更新API接口层
  │
  └─ wdscore.dll (WDS Core)
      └─ SetupWdscore() / WdsSetupLogDestroy()  [日志配置]
```

**OFFLINE_STORE_CREATION_PARAMETERS**（离线存储创建参数）：
- pszHostSystemDrivePath — 主机系统盘路径
- pszHostWindowsDirectoryPath — 主机Windows目录
- pszTargetWindowsDirectoryPath — 目标Windows目录
- pszHostRegistryMachineSoftwarePath — 离线 SOFTWARE 配置单元
- pszHostRegistryMachineSystemPath — 离线 SYSTEM 配置单元
- pszHostRegistryMachineSecurityPath — 离线 SECURITY 配置单元
- pszHostRegistryMachineSAMPath — 离线 SAM 配置单元
- pszHostRegistryMachineComponentsPath — 离线 COMPONENTS 配置单元
- ulProcessorArchitecture — 处理器架构

---

## 四、包构建流水线（PkgGen / spkggen / ConvertDSM）

### 4.1 PkgGen.exe — 包生成器

**文件信息**：.NET Framework 4.0 程序集，28,160 字节，命名空间 `Microsoft.CompPlat.PkgBldr`

**命令行格式**：
```
PkgGen /project:<input.pkg.xml> /output:<output> /convert:<type> /version:<ver> /cpu:<arch> [选项]
```

**转换类型 (ConversionType)**：
| 类型 | 输入 | 输出 | 说明 |
|------|------|------|------|
| `pkg2wm` | .pkg.xml | .wm.xml | 包定义→Windows Mobile XML |
| `wm2csi` | .wm.xml | .man (CSI清单) | WM XML→Component Servicing Infrastructure |
| `csi2wm` | .man | .wm.xml | 反向转换 |
| `pkg2cab` | .pkg.xml | .cab | 完整流水线（调用spkggen+ConvertDSM） |

#### 4.1.1 插件架构

PkgGen 使用 `PkgBldrLoader` 动态加载插件 DLL：

| 插件类型 | DLL | 功能 |
|----------|-----|------|
| PkgToWm | PkgBldr.Plugin.PkgToWm.Base.dll | .pkg.xml → .wm.xml |
| WmToCsi | PkgBldr.Plugin.WmToCsi.*.dll | .wm.xml → CSI 清单（多个子插件） |
| CsiToCsi | PkgBldr.Plugin.CsiToCsi.Finalize.dll | CSI 清单最终化 |
| CsiToCab | PkgBldr.Plugin.CsiToCab.Base.dll | CSI → .cab（testpkg/universalbsp模式） |
| PkgFilter | (内置) | WoW 包过滤 |

**WmToCsi 子插件**：
- Capabilities — 功能声明
- KnobsStore — 设置存储
- OnecorePackageInfo — OneCore 包信息
- PolicyDefinition — 策略定义
- Security — 安全描述
- TestSupport — 测试支持

#### 4.1.2 宏解析系统

PkgGen 内嵌 XML 宏表资源：
- `Macros_PkgToWm.xml` — pkg→wm 转换宏
- `Macros_WmToCsi.xml` — wm→csi 转换宏
- `Macros_CsiToWm.xml` — csi→wm 转换宏
- `Macros_Policy.xml` — 策略宏
- `Macros_CsiToCmi.xml` — csi→cmi 宏

命令行变量通过 `/variables:"NAME=VALUE;NAME2=VALUE2"` 传入，自动追加 `BUILD_OS_VERSION=<version>`。

#### 4.1.3 pkg2cab 完整流程

```
PkgToCab(config, spkgGenArgs, testSignOnly)
  │
  ├─ [WoW模式] BuildWow(input)
  │   ├─ 遍历 WowType (host/guest)
  │   ├─ FilterPkgXml()  [按架构过滤]
  │   └─ 分别生成 host/guest 包
  │
  ├─ Run.RunSPkgGen(spkgGenArgs)  [调用 spkggen.exe]
  │   → 生成 .spkg 文件列表
  │
  └─ Parallel.ForEach(spkgList):
      Run.RunDsmConverter(spkg)  [调用 ConvertDSM.exe]
      → .spkg → .cab
```

### 4.2 spkggen.exe — 安全包生成器

**文件信息**：原生可执行文件，19,968 字节

**功能**：将 .wm.xml（Windows Mobile XML 包定义）转换为 .spkg（Signed Package，签名包）格式

**被 PkgGen 调用的参数**：
```
spkggen <input.wm.xml>
  /config:<config.xml>
  /xsd:<schema.xsd>
  /output:<output.spkg>
  /build:fre|chk
  /cpu:x86|arm|arm64|amd64
  /languages:<lang-list>
  /resolutions:<res-list>
  /variables:"VAR=VALUE;..."
  /toolPaths:<tool-dir>
  /version:<major.minor.build.qfe>
  [/toc] [/compress] [/diagnostic] [/nohives] [/isRazzleEnv]
```

### 4.3 ConvertDSM.exe — DSM 转换工具

**文件信息**：原生可执行文件，354,816 字节，配套库 `ConvertDSMDLL.dll`（327,680 字节）

**功能**：将 .spkg 转换为标准 .cab（CBS 包格式）
- DSM = Device Servicing Manifest（设备服务清单）
- 并行执行（Parallel.ForEach，线程数由 `/ConvertDsmThreadCount` 控制）
- 支持 WoW 模式（guest 包输出到 wow 子目录）

### 4.4 包格式演进链

```
.pkg.xml (源定义，人类可读)
    │ PkgBldr.Plugin.PkgToWm
    ▼
.wm.xml (Windows Mobile XML，中间格式)
    │ PkgBldr.Plugin.WmToCsi (多插件)
    ▼
.man (CSI Manifest，组件服务清单)
    │ PkgBldr.Plugin.CsiToCsi.Finalize
    ▼
最终 .man (最终化CSI清单)
    │ spkggen.exe
    ▼
.spkg (Signed Package，签名包)
    │ ConvertDSM.exe
    ▼
.cab (CBS Package，最终离线包) ← imageapp 消费此格式
```

---

## 五、FFU 镜像格式与最终化

### 5.1 FFU 包装结构

```
FinalizeImage()
  ├─ UpdateUsedSectors()  [更新已用扇区信息]
  ├─ ffuImage.Description = GetUpdateDescription(UpdateHistory.xml)
  │
  ├─ OutputWrapper(_outputFile)  [最外层：文件输出]
  │   └─ SecurityWrapper(_ffuImage, innerWrapper)  [安全层：目录/签名]
  │       └─ ManifestWrapper(_ffuImage, payloadWrapper)  [清单层：FFU元数据]
  │           └─ (实际磁盘数据 payload)
  │
  ├─ DismountFullFlashImage(payloadWrapper, version)
  │   └─ 将所有VHD/存储空间数据写入FFU payload
  │
  └─ WriteAllBytes(output.cat, securityWrapper.CatalogData)  [生成.cat目录文件]
```

### 5.2 FFU 版本

- DeviceLayout 版本 < 2 → FFU 版本 1
- DeviceLayout 版本 >= 2 → FFU 版本 2

### 5.3 关键分区名称常量

| 常量 | 分区名 | 用途 |
|------|--------|------|
| MAINOS_PARTITION_NAME | MainOS | 主操作系统分区 |
| DATA_PARTITION_NAME | Data | 用户数据分区 |
| EFIESP_PARTITION_NAME | EFIESP | EFI 系统分区 |
| DPP_PARTITION_NAME | DPP | 设备配置分区 |
| OSDATA_PARTITION_NAME | OSData | OS 数据分区（更新模式） |
| PREINSTALLED_PARTITION_NAME | PreInstalled | 预安装 AppX 分区 |

---

## 六、辅助工具与组件

### 6.1 镜像工具

| 工具 | 类型 | 大小 | 功能 |
|------|------|------|------|
| ffutool.exe | .NET | 45KB | FFU 镜像操作（拆分/合并/查询） |
| imagex.exe | Native | 657KB | WIM 镜像工具（DISM 子集） |
| ImgDump.exe | .NET | 11KB | 镜像转储 |
| imgtowim.exe | .NET | 12KB | 镜像转 WIM |
| wpimage.exe | .NET | 22KB | Windows Phone 镜像工具 |
| imagesigner.exe | .NET | 13KB | 镜像签名 |

### 6.2 AppX 工具

| 工具 | 类型 | 功能 |
|------|------|------|
| makeappx.exe | Native | AppX 打包/解包 |
| appxpackaging.dll | Native | AppX 打包核心（1.4MB） |
| appxdeploymentclient.dll | Native | AppX 部署客户端 |
| appxprovisionpackage.dll | Native | AppX 预配 |
| appxreg.dll | Native | AppX 注册 |

**AppX 两阶段处理**（在 imaging.dll 中）：
- `AppXImaging.StageAppXPackages()` — 暂存阶段：解压 AppX 到 staging root
- `AppXImaging.CommitAppXPackages()` — 提交阶段：部署到 PreInstalled 分区

### 6.3 驱动服务

| 组件 | 功能 |
|------|------|
| DrvServicing.dll | 驱动服务核心 |
| drupdate.dll | 驱动更新 |
| drvstore.dll | 驱动存储管理（920KB） |
| infverif.dll | INF 验证 |
| DrvPSM.dll | 驱动包状态管理 |
| devicenodecleanup.x86/x64.exe | 设备节点清理（镜像创建后调用） |

### 6.4 签名工具

| 工具 | 功能 |
|------|------|
| signtool.exe | 通用签名工具 |
| pkgsigntool.exe | 包签名工具 |
| imagesigner.exe | 镜像签名 |
| secwimtool.exe | 安全 WIM 工具（3.1MB） |
| signinfohelper.dll | 签名信息辅助 |

### 6.5 离线注册表/配置

| 组件 | 功能 |
|------|------|
| offreg.dll | 离线注册表操作 |
| offlinesam.dll | 离线 SAM 数据库 |
| offlinelsa.dll | 离线 LSA |
| mvoffline.dll | 离线验证 |
| mcsfoffline.dll | 离线 MCSF |

### 6.6 设备布局

| 组件 | 功能 |
|------|------|
| DeviceLayoutValidation.dll | .NET，设备布局 XML 验证（139KB） |
| platformmanifest.dll | 平台清单 |
| MetadataReader.dll | 元数据读取器（166KB） |

---

## 七、完整构建端到端流程图

```
┌──────────────────────────────────────────────────────────────────┐
│                      阶段一：包构建                                │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  源代码/资源 (.pkg.xml + 文件)                                   │
│       │                                                          │
│       ▼ PkgGen.exe /convert:pkg2cab                             │
│  ┌─────────────────────────────────────────┐                    │
│  │ 1. PkgToWm 插件: .pkg.xml → .wm.xml     │                    │
│  │ 2. WmToCsi 插件: .wm.xml → CSI .man     │                    │
│  │ 3. CsiToCsi 插件: 最终化 CSI 清单        │                    │
│  │ 4. spkggen.exe: .man → .spkg (签名包)   │                    │
│  │ 5. ConvertDSM.exe: .spkg → .cab (CBS包) │                    │
│  └─────────────────────────────────────────┘                    │
│       │                                                          │
│       ▼                                                          │
│  输出: 多个 .cab 离线包 (MSPackages 目录)                         │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│                      阶段二：镜像构建                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  输入:                                                            │
│    - OEMInput.xml (产品/功能/语言/包列表)                        │
│    - Feature Manifest XML (功能清单)                              │
│    - DeviceLayout.xml (磁盘分区布局)                              │
│    - MSPackages/*.cab (阶段一输出)                                │
│    - OEMCustomization.xml/PPKG (可选定制)                        │
│                                                                  │
│  imageapp.exe Flash.ffu OEMInput.xml MSPackages +StrictSettingPolicies │
│       │                                                          │
│       ▼ imaging.dll                                              │
│  ┌──────────────────────────────────────────────────────┐       │
│  │ 1. SelectPackagesToImage()                            │       │
│  │    - 解析 OEMInput + FM → 过滤包列表                  │       │
│  │    - 生成 UpdateInput.xml                             │       │
│  │    - 生成 OEM 定制包                                   │       │
│  ├──────────────────────────────────────────────────────┤       │
│  │ 2. PopulateStagingImage() [暂存VHD]                  │       │
│  │    - ImageStorageManager 创建暂存 VHD                 │       │
│  │    - StageImage() → UpdateDLL.PrepareUpdate()        │       │
│  │      (解压 .cab → UpdateStagingRoot\TempSxS\)       │       │
│  │    - AppX Stage                                       │       │
│  ├──────────────────────────────────────────────────────┤       │
│  │ 3. CommitStagedImage() [最终镜像]                    │       │
│  │    - ProcessMinFreeSectors() 动态调整分区大小         │       │
│  │    - ImageStorageManager 创建最终 VHD/FFU 结构       │       │
│  │      (物理磁盘 + 存储池 + 存储空间 + GPT分区)         │       │
│  │    - CommitImage() → UpdateDLL.ExecuteUpdate()       │       │
│  │      (应用暂存内容 → 离线 Windows 分区)               │       │
│  │    - AppX Commit                                      │       │
│  ├──────────────────────────────────────────────────────┤       │
│  │ 4. FinalizeImage()                                    │       │
│  │    - Format DPP 分区                                  │       │
│  │    - FFU: OutputWrapper→SecurityWrapper→ManifestWrapper│     │
│  │    - 生成 .cat 目录文件                                │       │
│  │    - Dismount + 写入 FFU payload                      │       │
│  └──────────────────────────────────────────────────────┘       │
│       │                                                          │
│       ▼                                                          │
│  输出: Flash.ffu + Flash.cat + Flash.UpdateOutput.xml           │
│        + Flash.UpdateHistory.xml + Flash.PackageList.xml        │
│        + ImageApp.log + ImageApp.cbs.log                        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 八、重建项目关键技术点

### 8.1 必须保留的核心依赖

1. **UpdateDLL.dll** — 两阶段更新的核心，无法绕过。imageapp 通过 P/Invoke 直接调用，不经过 updateapp.exe
2. **imagestorageservice.dll (32位)** — 存储池/分区创建，必须在 32 位进程中调用
3. **wcp.dll + cbscore.dll** — CBS 组件服务，UpdateDLL 的内部依赖
4. **rs5_spaceutil.exe** — 存储池操作（根据目标 Build 选择版本）
5. **PkgGen.exe + spkggen.exe + ConvertDSM.exe** — 包构建三件套

### 8.2 关键数据结构

- **PARTITION_ENTRY** (201字节) — 分区定义，必须精确布局
- **STORE_ID** — GPT/MBR 磁盘标识联合体
- **OFFLINE_STORE_CREATION_PARAMETERS** — 离线 Windows 实例创建参数
- **UpdateOSInput / UpdateOSOutput** — 更新输入/输出 XML 序列化类

### 8.3 环境要求

- **管理员权限** — 创建 VHD、存储池、挂载离线注册表都需要
- **32位进程** — imagestorageservice.dll 是 32 位
- **.NET Framework 4.0** — 所有托管程序集的目标框架
- **SE_RESTORE_PRIVILEGE + SE_SECURITY_PRIVILEGE** — imaging.dll 在 ProcessImage 开头启用
- **全局互斥锁** `Global\VHDMutex_{585b0806-2d3b-4226-b259-9c8d3b237d5c}` — 防止多个 imageapp 实例并发

### 8.4 重建策略建议

| 组件 | 重建难度 | 建议策略 |
|------|----------|----------|
| imageapp.exe | 低 | 直接复用反编译的 C# 代码，逻辑清晰 |
| imaging.dll | 中 | 核心编排逻辑，可参考反编译代码重写 |
| imagestorageservicemanaged.dll | 中 | P/Invoke 封装层，可直接复用 |
| imagestorageservice.dll | 高 | 原生存储层，建议直接复用二进制 |
| UpdateDLL.dll | 极高 | CBS 更新核心，必须复用二进制 |
| PkgGen.exe | 中 | 插件架构清晰，可参考重写 |
| spkggen.exe | 高 | 原生签名包生成，建议复用二进制 |
| ConvertDSM.exe | 高 | 原生 DSM 转换，建议复用二进制 |

---

## 九、参考文件索引

### 反编译源码位置
- .NET 反编译：`E:\WSK_Tools\ADK_Research\outputsrc\dotnet\`
- Native 反编译：`E:\WSK_Tools\ADK_Research\outputsrc\native\`
- 托管精选：`E:\WSK_Tools\ADK_Research\decompiled\managed\17704\`
- Native 精选：`E:\WSK_Tools\ADK_Research\decompiled\native\`

### 原始 ADK 文件
- `E:\WSK_Tools\ADK_Research\SourceDir\Windows Kits\10\tools\bin\i386\` — 完整工具集
- `E:\WSK_Tools\ADK_Research\ADK\` — DLL 集合（扁平化）

### 关键反编译文件
| 文件 | 大小 | 内容 |
|------|------|------|
| imageapp.cs | 14KB | imageapp 入口 |
| imaging.cs | 148KB | 核心编排器（Imaging 类 + UpdateMain 类） |
| imagecommon.cs | 474KB | 镜像公共库（设备布局、VHD操作等） |
| imagestorageservicemanaged.cs | 416KB | 存储服务托管封装 |
| ffucomponents.cs | 176KB | FFU 组件定义 |
| PkgGen.cs | 35KB | 包生成器入口 |
| PkgBldr.Common.cs | 192KB | 包构建公共库 |
| pkggencommon.cs | 208KB | 包生成公共逻辑 |
| DeviceLayoutValidation.cs | 89KB | 设备布局验证 |
| updateapp.c | 3.1MB | updateapp 原生反编译 |
| imagestorageservice.c | 1.3MB | 存储服务原生反编译 |
| updatedll.c | 6.5MB | UpdateDLL 原生反编译 |
| wcp.c | 13MB | Windows Component Platform |
| cbscore.c | 9.1MB | CBS 核心 |

---

*报告基于 ADK 10.0.17704.1000 (Windows 10 RS5) 反编译内容生成*
*研究工具：ILSpy / IDA Pro 9.4 Hex-Rays / Ghidra 12.1.3*
