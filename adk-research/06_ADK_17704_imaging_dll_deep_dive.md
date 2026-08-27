# ADK 17704 imaging.dll 深度研究 — 核心编排器完整分析

> 生成日期：2026-08-27
> 研究范围：Imaging 类完整编排、ProcessImage 全流程、UpdateMain P/Invoke、MinFreeSectors 动态分区、存储池集成
> 数据来源：ILSpy 反编译 imaging.cs (148KB, 3655行)

---

## 一、Imaging 类概览

### 1.1 类定义与命名空间

```csharp
namespace Microsoft.WindowsPhone.Imaging
{
    public class Imaging : IDisposable
    {
        // 核心字段
        private IULogger _logger;
        private string _oemInputFile, _msPackagesRoot, _outputFile;
        private bool _bDoingFFU = true;        // FFU 模式 (默认)
        private bool _bDoingUpdate;             // 更新模式
        private bool _bCreatingStagingVHD;      // 创建暂存 VHD
        private bool _bUsingStagingVHD;         // 使用暂存 VHD
        private DeviceLayoutValidator _deviceLayoutValidator;
        private ReleaseType _releaseType;
        private Dictionary<string, IPkgInfo> _packageInfoList;
        private ImageStorageManager _storageManager;
        private ImageGeneratorParameters _parameters;
        private FullFlashUpdateImage _ffuImage;
        private string _updateStagingRoot;
        // ... 共约 60 个私有字段
    }
}
```

### 1.2 构造函数

```csharp
public Imaging(
    IULogger logger,
    bool skipImaging,           // 跳过镜像生成 (仅生成定制包)
    bool formatDPP,             // 格式化 DPP 分区
    bool strictSettingPolicies, // 严格设置策略
    bool skipUpdateMain,        // 跳过 UpdateMain (不调用 CBS)
    CpuId cpuId,                // CPU 架构
    string bspProductName,      // BSP 产品名
    string chunkMapping)        // chunk 映射
```

### 1.3 公共入口方法

| 方法 | 用途 |
|------|------|
| `BuildNewImage(...)` | 新建完整镜像 |
| `UpdateExistingImage(...)` | 更新已有镜像 |
| `CreateStagingImage(...)` | 创建暂存 VHD |
| `CommitStagedImage(...)` | 提交暂存镜像 |
| `Dispose()` | 清理资源 |

---

## 二、ProcessImage — 主控编排流程

### 2.1 完整调用链

```
BuildNewImage(oemInput, msPackagesRoot, outputFile, ...)
  └─ ProcessImage(randomizeGptIds, recovery=false)
      │
      ├─ 1. 全局互斥锁
      │   Mutex mutex = new Mutex(initiallyOwned: false,
      │       "Global\\VHDMutex_{585b0806-2d3b-4226-b259-9c8d3b237d5c}")
      │   mutex.WaitOne()  // 防止多个 imageapp 实例并发
      │
      ├─ 2. 权限提升
      │   SecurityUtil.SetPrivilege("SeRestorePrivilege", true)
      │   SecurityUtil.SetPrivilege("SeSecurityPrivilege", true)
      │
      ├─ 3. PrepareImaging()
      │   ├─ SetPaths()  设置输出路径、临时目录、日志
      │   ├─ 确定输出类型: .ffu→FFU模式 / .vhd/.vhdx→VHD模式
      │   ├─ _bDoingFFU = (outputExtension == ".ffu")
      │   └─ 遥测日志初始化 (ImagingTelemetryLogger)
      │
      ├─ 4. CreateFullImage()  [新建镜像核心]
      │   │
      │   ├─ 4a. [StagingVHD模式] 挂载差异盘 + 反序列化暂存数据
      │   │
      │   ├─ 4b. SelectPackagesToImage()  [包选择]
      │   │   ├─ OEMInput.ValidateInput()
      │   │   ├─ ProcessFMs()  处理 Feature Manifest
      │   │   │   ├─ 从核心FM包提取 FM XML
      │   │   │   ├─ 遍历所有 AdditionalFM
      │   │   │   ├─ GetFilteredPackagesByGroups()  按组过滤
      │   │   │   ├─ FMAddOnDrivers.GetFilteredDriverPackages()
      │   │   │   ├─ FMAddOnAppX.GetFilteredAppXPackages()
      │   │   │   └─ ProcessCompDBPackages()  构建 CompDB
      │   │   ├─ GenerateInputFile()  生成 UpdateInput.xml
      │   │   ├─ GenerateCustomizationContent()  生成 OEM 定制包
      │   │   └─ ValidateProductionImage()  生产镜像验证 (仅 Production)
      │   │
      │   ├─ 4c. ReadDeviceLayout()  [读取设备布局]
      │   │   ├─ 从包中提取 DeviceLayout.xml + OEMDevicePlatform.xml
      │   │   ├─ DeviceLayoutValidator.ValidateDeviceLayout()
      │   │   └─ ImageGeneratorParameters.ProcessInputXML()
      │   │
      │   ├─ 4d. WriteImageMetadataFiles()
      │   │   ├─ WriteProtoSystemManifest()  原型系统清单
      │   │   └─ WriteCompDBs()  BSPCompDB + DeviceCompDB
      │   │
      │   ├─ 4e. PopulateStagingImage()  [暂存镜像填充]
      │   │   ├─ ImageStorageManager 创建暂存 VHD
      │   │   ├─ InitializeMinFreeSectors()  初始化分区大小 (×3.5)
      │   │   └─ StageImage()  [★ CBS 暂存]
      │   │       ├─ 创建 UpdateOS.wim (LZX 压缩)
      │   │       ├─ UpdateMain.Initialize(storeIds, UpdateInput.xml, stagingRoot, log)
      │   │       ├─ UpdateMain.PrepareUpdate()  → UpdateDLL!PrepareUpdateWithFlags
      │   │       └─ AppXImaging.StageAppXPackages()
      │   │
      │   └─ 4f. CommitStagedImage()  [提交镜像]
      │       ├─ ProcessMinFreeSectors()  动态调整分区大小
      │       ├─ ImageStorageManager 创建最终 VHD/FFU 结构
      │       ├─ EnforcePartitionRestrictions()  分区限制检查
      │       ├─ CommitImage()  [★ CBS 提交]
      │       │   ├─ UpdateMain.Initialize(...)
      │       │   ├─ UpdateMain.SetPoolId(poolGuid)  存储池ID
      │       │   ├─ AppXImaging.CommitAppXPackages()
      │       │   └─ UpdateMain.ExecuteUpdate()  → UpdateDLL!ExecuteUpdate
      │       ├─ ProcessBSPProductNameAndVersion()
      │       ├─ LoadDataAssets()
      │       └─ FinalizeImage()  [最终化]
      │           ├─ Format DPP 分区 (如需)
      │           ├─ [FFU] 构建 FFU 包装:
      │           │   OutputWrapper → SecurityWrapper → ManifestWrapper
      │           ├─ ImageStorageManager.DismountFullFlashImage()
      │           └─ 生成 .cat 目录文件
      │
      └─ 5. finally: CleanupHandler() + ReportMetrics()
```

### 2.2 新建 vs 更新 vs 暂存模式

| 模式 | 入口方法 | 关键标志 | 流程差异 |
|------|----------|----------|----------|
| **新建完整镜像** | `BuildNewImage` | `_bDoingUpdate=false` | 完整 ProcessImage |
| **更新已有镜像** | `UpdateExistingImage` | `_bDoingUpdate=true` | 跳过 SelectPackages，直接 MountAndInstall |
| **创建暂存 VHD** | `CreateStagingImage` | `_bCreatingStagingVHD=true` | 只执行到 StageImage |
| **提交暂存镜像** | `CommitStagedImage` | `_bUsingStagingVHD=true` | 从暂存 VHD 继续到 Finalize |

---

## 三、UpdateMain 类 — CBS 更新 P/Invoke 封装

### 3.1 类结构

```csharp
internal sealed class UpdateMain : IDisposable
{
    private IntPtr UpdateContext = IntPtr.Zero;  // 原生更新上下文

    internal enum IUPhase { IUPhase_Staging, IUPhase_Commit }

    [Flags] public enum PrepareUpdateFlags {
        None = 0,
        ImperfectUpdateAllowed = 1,
        CreatingStagingVHD = 2      // ★ 创建暂存 VHD 时使用
    }

    [Flags] public enum ExecuteUpdateFlags {
        None = 0,
        UsingStagingVHD = 1,        // ★ 使用暂存 VHD 时使用
        ResetCommit = 2
    }
}
```

### 3.2 OFFLINE_STORE_CREATION_PARAMETERS — 离线存储创建参数

```csharp
public struct OFFLINE_STORE_CREATION_PARAMETERS
{
    public UIntPtr cbSize;
    public uint dwFlags;
    public string pszHostSystemDrivePath;              // 主机系统盘
    public string pszHostWindowsDirectoryPath;          // 主机 Windows 目录
    public string pszTargetWindowsDirectoryPath;        // 目标 Windows 目录
    public string pszHostRegistryMachineSoftwarePath;   // 离线 SOFTWARE 配置单元
    public string pszHostRegistryMachineSystemPath;     // 离线 SYSTEM 配置单元
    public string pszHostRegistryMachineSecurityPath;   // 离线 SECURITY 配置单元
    public string pszHostRegistryMachineSAMPath;        // 离线 SAM 配置单元
    public string pszHostRegistryMachineComponentsPath; // 离线 COMPONENTS 配置单元
    public string pszHostRegistryUserDotDefaultPath;    // 离线用户默认配置
    public string pszHostRegistryDefaultUserPath;       // 离线默认用户
    public uint ulProcessorArchitecture;                 // 处理器架构
    public string pszHostRegistryMachineOfflineSchemaPath; // 离线 schema
}
```

### 3.3 完整 P/Invoke 声明

#### UpdateDLL.dll (cdecl 调用约定)

```csharp
// 上下文管理
[DllImport("UpdateDLL.dll", CallingConvention=Cdecl, CharSet=Auto)]
private static extern IntPtr CreateUpdateContext();

[DllImport("UpdateDLL.dll", CallingConvention=Cdecl, CharSet=Auto)]
private static extern int ReleaseUpdateContext(IntPtr UpdateContext);

[DllImport("UpdateDLL.dll", CallingConvention=Cdecl, CharSet=Auto)]
private static extern int Initialize(
    IntPtr UpdateContext,
    int storeIdsCount,
    [MarshalAs(UnmanagedType.LPArray, SizeParamIndex=1)] STORE_ID[] storeIds,
    string UpdateInputFile,
    string AlternateStagingLocation,
    string LogFilePath);

[DllImport("UpdateDLL.dll", CallingConvention=Cdecl, CharSet=Auto)]
private static extern void Deinitialize(IntPtr UpdateContext);

// 两阶段更新 ★核心
[DllImport("UpdateDLL.dll", CallingConvention=Cdecl, CharSet=Auto)]
private static extern int PrepareUpdate(IntPtr UpdateContext);

[DllImport("UpdateDLL.dll", CallingConvention=Cdecl, CharSet=Auto)]
private static extern int PrepareUpdateWithFlags(IntPtr UpdateContext, uint Flags);
// Flags: 0=普通, 2=CreatingStagingVHD

[DllImport("UpdateDLL.dll", CallingConvention=Cdecl, CharSet=Auto)]
private static extern int ExecuteUpdate(IntPtr UpdateContext, uint Flags);
// Flags: 0=普通, 1=UsingStagingVHD, 2=ResetCommit

[DllImport("UpdateDLL.dll", CallingConvention=Cdecl, CharSet=Unicode)]
private static extern void SetPoolId(IntPtr UpdateContext, Guid PoolId);

// 进度报告
[DllImport("UpdateDLL.dll", CallingConvention=Cdecl, CharSet=Auto)]
private static extern void IU_InitializeDefaultProgressReporting(IUPhase Phase);

// 性能统计
[DllImport("UpdateDLL.dll", CallingConvention=Cdecl, CharSet=Auto)]
private static extern uint GetStageFilesTickCount(IntPtr UpdateContext);

[DllImport("UpdateDLL.dll", CallingConvention=Cdecl, CharSet=Auto)]
private static extern uint GetExecuteUpdateOSTickCount(IntPtr UpdateContext);

// 文件大小
[DllImport("UpdateDLL.dll", CallingConvention=Cdecl, CharSet=Auto)]
private static extern ulong GetCompressedFilesize(string file);

[DllImport("UpdateDLL.dll", CallingConvention=Cdecl, CharSet=Auto)]
private static extern ulong GetUncompressedFilesize(string file);

// 目录大小
[DllImport("UpdateDLL.dll", CallingConvention=Cdecl, CharSet=Unicode)]
private static extern int IU_GetDirectorySize(
    string folder, bool recursive, uint clusterSize, out ulong folderSize);

[DllImport("UpdateDLL.dll", CallingConvention=Cdecl, CharSet=Unicode)]
public static extern uint IU_GetClusterSize(string folder);

// 文件复制
[DllImport("UpdateDLL.dll", CallingConvention=StdCall, CharSet=Unicode)]
private static extern int CopyAllFiles(string source, string dest, bool recursive, bool mirror);

// 日志
[DllImport("UpdateDLL.dll", CallingConvention=Cdecl, CharSet=Auto, SetLastError=true)]
private static extern void IU_LogTo(
    InteropLogString ErrorMsgHandler,
    InteropLogString WarningMsgHandler,
    InteropLogString InfoMsgHandler,
    InteropLogString DebugMsgHandler);
```

#### wcp.dll (StdCall 调用约定)

```csharp
[DllImport("wcp.dll", CallingConvention=StdCall, CharSet=Unicode)]
public static extern int WcpInitialize(out UIntPtr InitCookie);

[DllImport("wcp.dll", CallingConvention=StdCall, CharSet=Unicode)]
public static extern void WcpShutdown(UIntPtr InitCookie);

[DllImport("wcp.dll", CallingConvention=StdCall, CharSet=Unicode)]
public static extern int SetIsolationIMalloc(UIntPtr IMalloc);

[DllImport("wcp.dll", CallingConvention=StdCall, CharSet=Unicode)]
public static extern int CreateNewWindows(
    uint dwFlags,
    string szSystemDrive,
    ref OFFLINE_STORE_CREATION_PARAMETERS pParameters,
    UIntPtr ppvKeys,
    out uint pdwDisposition);

[DllImport("Ole32.dll", CallingConvention=StdCall, CharSet=Unicode)]
public static extern int CoGetMalloc(uint dwMemContext, out UIntPtr pMalloc);
```

### 3.4 公共包装方法

```csharp
public int Initialize(int storeIdsCount, STORE_ID[] storeIds,
    string UpdateInputFile, string AlternateStagingLocation, string LogFilePath,
    InteropLogString Error, InteropLogString Warning,
    InteropLogString Info, InteropLogString Debug)
{
    LogUtil.IULogTo(Error, Warning, Info, Debug);  // 注册日志回调
    return Initialize(UpdateContext, storeIdsCount, storeIds,
        UpdateInputFile, AlternateStagingLocation, LogFilePath);
}

public int PrepareUpdate()
{
    RegisterProgressCallback(IUPhase.IUPhase_Staging);
    return PrepareUpdateWithFlags(UpdateContext, 0);  // Flags=0
}

public int PrepareStagingVHDUpdate()
{
    RegisterProgressCallback(IUPhase.IUPhase_Staging);
    return PrepareUpdateWithFlags(UpdateContext, 2);  // Flags=CreatingStagingVHD
}

public int ExecuteUpdate()
{
    RegisterProgressCallback(IUPhase.IUPhase_Commit);
    return ExecuteUpdate(UpdateContext, 0);  // Flags=0
}

public int ExecuteStagingVHDUpdate()
{
    RegisterProgressCallback(IUPhase.IUPhase_Commit);
    return ExecuteUpdate(UpdateContext, 1);  // Flags=UsingStagingVHD
}

public void SetPoolId(Guid PoolId) { SetPoolId(UpdateContext, PoolId); }
public uint GetUpdateOSTime() { return GetExecuteUpdateOSTickCount(UpdateContext); }
public uint GetFileStageTime() { return GetStageFilesTickCount(UpdateContext); }
```

### 3.5 生命周期管理

```csharp
public UpdateMain() { UpdateContext = CreateUpdateContext(); }

~UpdateMain() { Dispose(isDisposing: false); }

public void Dispose()
{
    Dispose(isDisposing: true);
    GC.SuppressFinalize(this);
}

private void Dispose(bool isDisposing)
{
    if (_alreadyDisposed) return;
    if (UpdateContext != IntPtr.Zero)
    {
        Deinitialize(UpdateContext);
        if (ReleaseUpdateContext(UpdateContext) == 0)
            UpdateContext = IntPtr.Zero;
    }
    _alreadyDisposed = true;
}
```

---

## 四、StageImage — CBS 暂存阶段

### 4.1 完整流程

```csharp
private void StageImage()
{
    // 1. 创建 UpdateOS.wim (LZX 压缩)
    string pendingWimPath = GetPendingUpdateOSWimPath();
    using (FileStream wimStream = FileToolBox.Stream(pendingWimPath))
    {
        // WIM 包含离线更新操作系统 (UpdateOS)
    }

    // 2. 构建 STORE_ID 数组 (所有分区的磁盘标识)
    STORE_ID[] storeIds = BuildStoreIds();

    // 3. 初始化 UpdateMain
    using (UpdateMain updateMain = new UpdateMain())
    {
        int hr = updateMain.Initialize(
            storeIds.Length, storeIds,
            _updateInputFileGenerated,      // UpdateInput.xml
            _updateStagingRoot,              // 暂存根目录
            cbsLogPath,                      // CBS 日志路径
            ErrorHandler, WarningHandler, InfoHandler, DebugHandler);

        if (UpdateMain.FAILED(hr))
            throw new ImageCommonException("UpdateMain.Initialize failed");

        // 4. ★ 执行暂存 (PrepareUpdate)
        if (_bCreatingStagingVHD)
            hr = updateMain.PrepareStagingVHDUpdate();  // Flags=2
        else
            hr = updateMain.PrepareUpdate();              // Flags=0

        if (UpdateMain.FAILED(hr))
            throw new ImageCommonException($"PrepareUpdate failed: 0x{hr:X8}");

        // 5. 记录性能统计
        _logger.LogInfo($"Stage files time: {updateMain.GetFileStageTime()}ms");
    }

    // 6. AppX 暂存
    AppXImaging.StageAppXPackages(_updateStagingRoot, _packageInfoList);
}
```

### 4.2 UpdateInput.xml 格式

暂存阶段的输入文件 `UpdateInput.xml` 由 `GenerateInputFile()` 生成：

```xml
<UpdateInput>
  <Packages>
    <Package SourcePath="C:\...\package1.cab" Partition="MainOS" />
    <Package SourcePath="C:\...\package2.cab" Partition="MainOS" />
    ...
  </Packages>
  <Settings>
    <Setting Name="..." Value="..." />
  </Settings>
</UpdateInput>
```

包路径用 `|` 分隔 (`c_UpdateInputDelimiter = "|"`)。

### 4.3 暂存目录结构

```
_updateStagingRoot\
├── MainOS\
│   └── TempSxS\              ★ 暂存的 WinSxS 文件
│       ├── manifests\         CBS 清单
│       ├── packages\          包文件
│       └── ...
├── EFIESP\
│   └── TempSxS\
├── <OtherPartitions>\
│   └── TempSxS\
└── Pending_UpdateOS.wim       ★ 离线更新操作系统镜像
```

---

## 五、CommitImage — CBS 提交阶段

### 5.1 完整流程

```csharp
private void CommitImage()
{
    // 1. 构建最终 STORE_ID 数组 (包含实际分区的磁盘句柄)
    STORE_ID[] storeIds = BuildFinalStoreIds();

    // 2. 初始化 UpdateMain
    using (UpdateMain updateMain = new UpdateMain())
    {
        int hr = updateMain.Initialize(
            storeIds.Length, storeIds,
            _updateInputFileGenerated,
            _updateStagingRoot,          // 暂存根 (源)
            cbsLogPath,
            handlers...);

        // 3. ★ 设置存储池 ID (如有存储池)
        if (_ffuImage.StoragePools.Count > 0)
        {
            Guid poolId = _ffuImage.StoragePools[0].Id;
            updateMain.SetPoolId(poolId);
        }

        // 4. AppX 提交 (在 ExecuteUpdate 之前)
        AppXImaging.CommitAppXPackages(
            _storageManager, _packageInfoList, _ffuImage);

        // 5. ★ 执行提交 (ExecuteUpdate)
        if (_bUsingStagingVHD)
            hr = updateMain.ExecuteStagingVHDUpdate();  // Flags=1
        else
            hr = updateMain.ExecuteUpdate();              // Flags=0

        if (UpdateMain.FAILED(hr))
            throw new ImageCommonException($"ExecuteUpdate failed: 0x{hr:X8}");

        // 6. 记录性能统计
        _logger.LogInfo($"UpdateOS time: {updateMain.GetUpdateOSTime()}ms");
    }
}
```

### 5.2 提交阶段的关键操作

ExecuteUpdate 内部执行以下操作（由 UpdateDLL/wcp/cbscore 完成）：

1. **挂载离线注册表配置单元**
   - SOFTWARE, SYSTEM, SECURITY, SAM, COMPONENTS
   - 通过 `OFFLINE_STORE_CREATION_PARAMETERS` 指定路径

2. **创建离线 Windows 实例**
   - `wcp.dll!CreateNewWindows()` 创建 WCP 隔离环境
   - `WcpInitialize()` 初始化 WCP
   - `SetIsolationIMalloc()` 设置内存分配器

3. **应用组件服务**
   - 从 TempSxS 复制文件到目标分区
   - 验证每个文件的 hash (CCSDirectTransaction::VerifyFileHashes)
   - 应用注册表修改
   - 应用安全描述符/权限
   - 注册服务/驱动
   - 处理 MUI/语言包

4. **生成输出**
   - UpdateHistory.xml — 更新历史
   - UpdateOutput.xml — 更新输出
   - PackageList.xml — 包列表

---

## 六、MinFreeSectors — 动态分区大小调整

### 6.1 InitializeMinFreeSectors — 暂存前初始化

```csharp
private void InitializeMinFreeSectors()
{
    _parameters.MinSectorCount = MBToSectors(102400);  // 默认 100GB

    foreach (InputPartition partition in
        _parameters.MainOSStore.Partitions.Where(x => x.MinFreeSectors != 0))
    {
        // ★ 初始大小 = (MinFreeSectors + GeneratedFileOverhead) × 3.5
        partition.TotalSectors = (uint)Math.Ceiling(
            (double)(partition.MinFreeSectors + partition.GeneratedFileOverheadSectors) * 3.5);

        // MainOS 分区不加额外空间，其他分区 +1500MB
        if (!partition.Name.Equals(MAINOS_PARTITION_NAME, OrdinalIgnoreCase))
            partition.TotalSectors += MBToSectors(1500);
    }
}
```

### 6.2 ProcessMinFreeSectors — 暂存后精确调整

```csharp
private void ProcessMinFreeSectors()
{
    // 存储池场景不适用
    if (_ffuImage.StoragePools.Count > 0) return;

    foreach (InputPartition partition in
        _parameters.MainOSStore.Partitions.Where(x => x.MinFreeSectors != 0))
    {
        _storageManager.WaitForVolume(partition.Name);

        // 1. 获取簇大小
        uint clusterSize = partition.ClusterSize;
        if (clusterSize == 0)
        {
            clusterSize = IU_GetClusterSize(_storageManager.GetPartitionPath(partition.Name));
            partition.ClusterSize = clusterSize;
        }

        // 2. 获取暂存目录实际大小
        string stagingFolder = Path.Combine(_updateStagingRoot, partition.Name);
        ulong folderSize = 0;
        if (_bUsingStagingVHD)
        {
            // 从暂存 VHD 的 CompDB 读取包大小映射
            foreach (IPkgInfo pkg in _packageInfoList.Values
                .Where(p => p.Partition.Equals(partition.Name, OrdinalIgnoreCase)))
            {
                folderSize += partition.Compressed
                    ? _stagingVHDCompDB.PackageSizeMap[pkg.Keyform].csize
                    : _stagingVHDCompDB.PackageSizeMap[pkg.Keyform].usize;
            }
        }
        else
        {
            // 调用原生 IU_GetDirectorySize 递归计算
            IU_GetDirectorySize(stagingFolder, recursive: true, clusterSize, out folderSize);
        }

        // 3. MainOS 分区加上 Pending_UpdateOS.wim 大小
        if (partition.Name == MAINOS_PARTITION_NAME && File.Exists(pendingWimPath))
        {
            folderSize += AlignUp((uint)new FileInfo(pendingWimPath).Length, clusterSize);
            if (_bUsingStagingVHD) folderSize += 800000000;  // 800MB 额外空间
        }

        // 4. 计算文件系统开销
        ulong overhead = _bUsingStagingVHD
            ? _stagingVHDCompDB.partitionOverheads.First(p => p.partitionName == partition.Name).overhead
            : GetFileSystemOverhead(partition, stagingFolder);

        // 5. 计算总扇区数
        ulong totalBytes = AlignUp(folderSize, clusterSize) + AlignUp(overhead, clusterSize);
        uint totalSectors = (uint)Math.Ceiling((double)totalBytes / _parameters.SectorSize);
        uint sectorsPerCluster = clusterSize / _parameters.SectorSize;
        totalSectors = (uint)AlignUp(totalSectors, sectorsPerCluster);

        // 6. 加上 MinFreeSectors 和 GeneratedFileOverhead
        partition.MinFreeSectors = (uint)AlignUp(partition.MinFreeSectors, sectorsPerCluster);
        partition.GeneratedFileOverheadSectors = (uint)AlignUp(
            partition.GeneratedFileOverheadSectors, sectorsPerCluster);
        partition.TotalSectors = totalSectors
            + partition.MinFreeSectors
            + partition.GeneratedFileOverheadSectors;

        // 7. ★ MainOS 分区额外 +4% 余量
        if (partition.Name == MAINOS_PARTITION_NAME)
            partition.TotalSectors += (uint)((double)partition.TotalSectors * 0.04);

        // 8. 按 1MB 对齐
        partition.TotalSectors = (uint)AlignUp(partition.TotalSectors, MBToSectors(1));
    }
}
```

### 6.3 分区大小计算公式

```
暂存前 (InitializeMinFreeSectors):
  TotalSectors = (MinFreeSectors + GeneratedFileOverhead) × 3.5
  if (!MainOS) TotalSectors += 1500MB

暂存后 (ProcessMinFreeSectors):
  folderSize = 实际暂存文件大小 (含 Pending_UpdateOS.wim)
  overhead = 文件系统开销
  totalBytes = AlignUp(folderSize, clusterSize) + AlignUp(overhead, clusterSize)
  totalSectors = AlignUp(ceil(totalBytes / sectorSize), sectorsPerCluster)
  TotalSectors = totalSectors + MinFreeSectors + GeneratedFileOverhead
  if (MainOS) TotalSectors += TotalSectors × 4%
  TotalSectors = AlignUp(TotalSectors, 1MB)
```

---

## 七、EnforcePartitionRestrictions — 分区限制检查

```csharp
private void EnforcePartitionRestrictions()
{
    InputPartition mainOS = GetMainOSPartition();

    // 1. 小于 5GB 平台必须启用 MainOS 压缩
    if (_parameters.MinSectorCount <= 5368709120 / sectorSize  // 5GB
        && !mainOS.Compressed)
        throw new ImagingException("MainOS not compressed but platform < 5GB");

    // 2. Data 分区最小大小 (默认 1.6GB，可由 OEMInput.Edition.MinimumUserStoreSize 覆盖)
    ulong minDataSize = _oemInput.Edition.MinimumUserStoreSize != 0
        ? _oemInput.Edition.MinimumUserStoreSize
        : 1717986918;  // 1.6GB
    ulong dataSectors = _storageManager.GetPartitionSizeInSectors(DATA_PARTITION_NAME);
    if (dataSectors < minDataSize / sectorSize && _releaseType == ReleaseType.Production)
        throw new ImageCommonException($"Data partition too small: {dataSectors * sectorSize} < {minDataSize}");

    // 3. 需要压缩的分区必须使用 4K 簇大小
    var invalidClusters = _parameters.MainOSStore.Partitions
        .Where(x => x.RequiresCompression && x.ClusterSize > 4096);
    if (invalidClusters.Count() > 0)
        throw new ImageCommonException("Compressed partitions require 4k cluster size");

    // 4. 需要压缩的分区必须标记为已压缩
    var notCompressed = _parameters.MainOSStore.Partitions
        .Where(x => x.RequiresCompression && !x.Compressed);
    if (notCompressed.Count() > 0)
        throw new ImagingException("Partitions require compression but not marked compressed");
}
```

---

## 八、FinalizeImage — 镜像最终化

### 8.1 FFU 模式最终化

```csharp
private void FinalizeImage()
{
    // 1. 更新已用扇区信息
    UpdateUsedSectors();

    // 2. 设置 FFU 描述 (从 UpdateHistory.xml 提取)
    _ffuImage.Description = GetUpdateDescription(_updateHistoryDestination);

    // 3. 构建 FFU Wrapper 链
    //    OutputWrapper (最外层) → SecurityWrapper → ManifestWrapper → Payload
    using (OutputWrapper output = new OutputWrapper(_outputFile))
    using (SecurityWrapper security = new SecurityWrapper(_ffuImage, output))
    using (ManifestWrapper manifest = new ManifestWrapper(_ffuImage, security))
    {
        // 4. 卸载所有 VHD/存储空间，写入 payload
        _storageManager.DismountFullFlashImage(manifest, _ffuImage.Version);
    }

    // 5. 生成 .cat 目录文件
    byte[] catalogData = _ffuImage.SecurityData;
    File.WriteAllBytes(_outputFile + ".cat", catalogData);
}
```

### 8.2 VHD 模式最终化

```csharp
// VHD 模式不需要 FFU wrapper
_storageManager.DismountVirtualHardDisk(storeId, deleteFile: false);
// 输出就是 .vhd/.vhdx 文件本身
```

### 8.3 FFU Wrapper 层结构

```
OutputWrapper (文件输出层)
  └─ SecurityWrapper (安全层)
      ├─ CatalogData (.cat 目录数据)
      ├─ HashTable (每个 chunk 的 SHA256)
      └─ ManifestWrapper (清单层)
          ├─ FFU Header (镜像元数据)
          ├─ Disk entries (磁盘布局)
          ├─ Partition entries (分区信息)
          ├─ Storage pool entries (存储池)
          └─ Payload (实际磁盘数据)
```

---

## 九、存储管理器集成 (ImageStorageManager)

### 9.1 imaging.dll 中的关键调用

```csharp
// 创建暂存 VHD
_storageManager.CreateVirtualHardDisk(
    _stagingVhdPath, maxSize, storeId, sectorSize, out diskHandle);

// 创建最终镜像结构
_storageManager.CreateStore(storeId, partitions);     // 普通磁盘
_storageManager.CreatePool(storeId, poolId);           // 存储池成员
_storageManager.CreateSpace(poolId, spaceId, size);    // 存储空间

// 分区操作
_storageManager.PartitionVirtualHardDisk(diskHandle, storeId, partitions);
_storageManager.ExtendPartition(diskHandle, storeId, partitionName, extendVolume);

// 卷操作
_storageManager.WaitForVolume(partitionName);
_storageManager.GetPartitionPath(partitionName);
_storageManager.GetPartitionSizeInSectors(partitionName);
_storageManager.GetFreeBytesOnVolume(partitionName);

// 卸载
_storageManager.DismountVirtualHardDisk(storeId, deleteFile);
_storageManager.DismountFullFlashImage(wrapper, version);
```

### 9.2 STORE_ID 构建

imaging.dll 为每个物理磁盘构建 `STORE_ID` 结构，传递给 UpdateMain.Initialize：

```csharp
STORE_ID[] BuildStoreIds()
{
    List<STORE_ID> ids = new List<STORE_ID>();
    foreach (ImageStorage store in _ffuImage.Stores)
    {
        STORE_ID id = new STORE_ID();
        id.StoreType = store.IsGpt ? 1 : 0;  // 0=MBR, 1=GPT
        if (store.IsGpt)
            id.StoreId_GPT = store.DiskId;     // GPT 磁盘 GUID
        else
            id.StoreId_MBR = store.DiskSignature; // MBR 磁盘签名
        ids.Add(id);
    }
    return ids.ToArray();
}
```

---

## 十、输出文件清单

### 10.1 完整镜像构建输出

| 文件 | 说明 |
|------|------|
| `Flash.ffu` | 最终 FFU 镜像 |
| `Flash.ffu.cat` | FFU 目录文件 (签名) |
| `Flash.UpdateOutput.xml` | 更新输出 |
| `Flash.UpdateHistory.xml` | 更新历史 |
| `Flash.PackageList.xml` | 包列表 |
| `ImageApp.log` | imageapp 日志 |
| `ImageApp.cbs.log` | CBS 服务日志 |

### 10.2 临时文件

| 路径 | 说明 |
|------|------|
| `%TEMP%\Imaging_<GUID>\` | 临时工作目录 |
| `UpdateInput.xml` | 生成的更新输入 |
| `UpdateStagingRoot\` | CBS 暂存根目录 |
| `Pending_UpdateOS.wim` | 离线更新操作系统镜像 |
| `ProtoSystemManifest.xml` | 原型系统清单 |
| `BSPCompDB.xml` | BSP 组件数据库 |
| `DeviceCompDB.xml` | 设备组件数据库 |

---

## 十一、关键常量

```csharp
// 分区名称
public const string MAINOS_PARTITION_NAME = "MainOS";
public const string DATA_PARTITION_NAME = "Data";
public const string EFIESP_PARTITION_NAME = "EFIESP";
public const string DPP_PARTITION_NAME = "DPP";
public const string OSDATA_PARTITION_NAME = "OSData";
public const string PREINSTALLED_PARTITION_NAME = "PreInstalled";

// 互斥锁
private const string VHD_MUTEX_NAME = "Global\\VHDMutex_{585b0806-2d3b-4226-b259-9c8d3b237d5c}";

// 分隔符
private const string c_UpdateInputDelimiter = "|";

// 版本
private const string c_AntiTheftMinVersion = "1.1";

// 默认值
private const ulong DEFAULT_MIN_SECTOR_COUNT_MB = 102400;  // 100GB
private const ulong DEFAULT_DATA_PARTITION_SIZE = 1717986918;  // 1.6GB
private const ulong STAGING_VHD_EXTRA_SPACE = 800000000;  // 800MB
private const double MAINTOS_EXTRA_MARGIN = 0.04;  // 4%
private const double INITIAL_SIZE_MULTIPLIER = 3.5;  // ×3.5
private const ulong EXTRA_PARTITION_SPACE_MB = 1500;  // 1.5GB
```

---

## 十二、错误处理与日志

### 12.1 异常类型

```csharp
[Serializable]
public class ImagingException : Exception
{
    public ImagingException(string message) : base(message) { }
    public ImagingException(string message, Exception inner) : base(message, inner) { }

    public override string ToString()
    {
        string text = Message;
        if (InnerException != null) text += InnerException.ToString();
        return text;
    }
}
```

### 12.2 CBS 日志回调

```csharp
// UpdateMain.Initialize 注册 4 个日志回调
LogUtil.IULogTo(
    ErrorMsgHandler,    // CBS 错误 → IULogger.LogError
    WarningMsgHandler,  // CBS 警告 → IULogger.LogWarning
    InfoMsgHandler,     // CBS 信息 → IULogger.LogInfo
    DebugMsgHandler);   // CBS 调试 → IULogger.LogDebug
```

### 12.3 遥测日志

```csharp
// ImagingTelemetryLogger
// EventSource: "Microsoft-Windows-Deployment-Imaging"
// 事件名: "Imaging"
// 记录: sessionId, EventName, Value1-4
```

---

## 十三、重建项目要点

### 13.1 必须保留的依赖

| 组件 | 类型 | 原因 |
|------|------|------|
| **UpdateDLL.dll** | Native | 两阶段更新核心，Initialize/PrepareUpdateWithFlags/ExecuteUpdate |
| **wcp.dll** | Native | WCP 初始化、CreateNewWindows、离线注册表 |
| **imagestorageservice.dll** | Native (32位) | VHD/存储池/分区创建 |
| **imagestorageservicemanaged.dll** | .NET | 存储服务托管封装 |
| **imagecommon.dll** | .NET | ImageSigner、PackageTools、设备布局 |
| **ffucomponents.dll** | .NET | FFU Wrapper 层 |

### 13.2 可重写的部分

| 组件 | 重写难度 | 说明 |
|------|----------|------|
| Imaging 类 | 中 | 编排逻辑清晰，可参考反编译代码 |
| UpdateMain 类 | 低 | P/Invoke 封装，已完整反编译 |
| MinFreeSectors 逻辑 | 低 | 纯数学计算，公式明确 |
| OEMInput/DeviceLayout 解析 | 中 | XML 解析，有验证器 |
| AppXImaging | 中 | AppX staging/commit 逻辑 |

### 13.3 关键集成点

1. **STORE_ID 传递** — imaging.dll 构建 STORE_ID 数组传给 UpdateMain.Initialize，UpdateDLL 用它定位离线分区
2. **SetPoolId** — 存储池场景必须在 ExecuteUpdate 前调用 SetPoolId
3. **UpdateInput.xml** — 包列表用 `|` 分隔，路径必须是绝对路径
4. **暂存目录结构** — `UpdateStagingRoot\<PartitionName>\TempSxS\` 是约定结构
5. **Pending_UpdateOS.wim** — MainOS 分区必须包含此文件，LZX 压缩
6. **全局互斥锁** — 必须获取 `Global\VHDMutex_{585b0806-...}` 防止并发
7. **权限** — 必须启用 SeRestorePrivilege 和 SeSecurityPrivilege

---

*报告基于 imaging.cs (148KB, 3655行) ILSpy 反编译内容生成*
*关键类：Microsoft.WindowsPhone.Imaging.Imaging + UpdateMain*
*UpdateMain P/Invoke 覆盖 UpdateDLL.dll (21个函数) + wcp.dll (4个函数) + Ole32.dll (1个函数)*
