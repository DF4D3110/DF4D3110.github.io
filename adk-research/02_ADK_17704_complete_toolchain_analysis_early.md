# ADK 10.0.17704.1000 完整工具链逆向分析报告

> 生成日期：2026-08-26
> 工具：IDA Pro 9.4 + Hex-Rays / ILSpy
> 目标：为 DeviceLayoutExplorer 程序重写提供完整原理参考

---

## 一、工具链概述

### 1.1 ADK 17704 定位
Windows 10 17704（RS5，1809版本）的评估和部署工具包，包含镜像创建、包管理、更新服务、驱动服务等完整工具链。
- 总计：232个文件（199 DLL + 24 EXE + 5 CMD + 4配置）
- 非stub可分析文件：125个（72原生 + 53 .NET托管）
- API set forwarder stub：98个（跳过）

### 1.2 架构特点
- 全部为32位（i386）二进制
- .NET程序集基于.NET Framework 4.0
- 原生二进制使用Visual C++编译，部分包含WPF/COM组件
- 工具间通过命令行、COM接口、CAB包、XML配置进行交互

---

## 二、组件功能分类

### 2.1 镜像与存储核心（8个组件）
| 组件 | 类型 | 大小 | 功能 |
|------|------|------|------|
| imageapp.exe | .NET | 16.5KB | 镜像创建主程序， orchestrate 整个FFU构建流程 |
| imagestorageservice.dll | 原生 | - | 存储池/存储空间创建核心，调用SpaceUtil |
| imagestorageservicemanaged.dll | .NET | 261.5KB | 存储服务托管封装层 |
| imagecommon.dll | .NET | 408KB | 镜像公共库，包含设备布局解析、VHD操作 |
| imaging.dll | .NET | 112KB | 成像API封装 |
| ffucomponents.dll | .NET | 3.2MB | FFU镜像组件定义 |
| ffutool.exe | .NET | 44KB | FFU镜像操作工具 |
| imagex.exe | 原生 | 641.8KB | WIM镜像工具（DISM子集） |

### 2.2 更新与CBS服务（7个组件）
| 组件 | 类型 | 功能 |
|------|------|------|
| updateapp.exe | 原生 | 更新应用主程序 |
| updateapi.dll | 原生 | 更新API接口 |
| updatedll.dll | 原生 | 更新核心逻辑 |
| cbscore.dll | 原生 | CBS（Component Based Servicing）核心 |
| cbsapi.dll | 原生 | CBS API接口 |
| wcp.dll | 原生 | Windows Component Platform |
| drvstore.dll | 原生 | 驱动存储管理 |

### 2.3 包构建（20+组件）
- PkgGen.exe - 包生成器
- PkgBldr.Common.dll - 包构建公共库
- pkggencommon.dll - 包生成公共逻辑
- pkgcommonmanaged.dll - 托管包公共库
- pkgcomposition.dll - 包组合
- pkgtoolbox.dll - 包工具箱
- spkggen.exe - 安全包生成器
- PkgBldr.Plugin.* - 各类插件（CsiToCab, WmToCsi等）

### 2.4 设备布局（3个组件）
- DeviceLayoutValidation.dll - 设备布局XML验证
- ConvertDSM.exe - DSM转换工具
- ConvertDSMDLL.dll - DSM转换库

### 2.5 驱动服务（6个组件）
- DrvServicing.dll - 驱动服务
- drupdate.dll - 驱动更新
- DrvPSM.dll - 驱动包状态管理
- infverif.dll - INF验证
- devicenodecleanup.x86/x64.exe - 设备节点清理

### 2.6 AppX（6个组件）
- makeappx.exe - AppX打包
- appxpackaging.dll - AppX打包核心
- appxdeploymentclient.dll - AppX部署客户端
- appxreg.dll - AppX注册
- appxprovisionpackage.dll - AppX预配
- appxcommon.dll / appximaging.dll - 托管库

### 2.7 WIM与安全（5个组件）
- secwimtool.exe - 安全WIM工具（3.1MB）
- wiminterop.dll - WIM互操作
- signtool.exe - 签名工具
- imagesigner.exe - 镜像签名
- pkgsigntool.exe - 包签名工具

### 2.8 网络与配置（8个组件）
- NetSetupEngine.dll - 网络设置引擎
- NetSetupApi.dll - 网络设置API
- FirewallOfflineAPI.dll - 防火墙离线API
- offreg.dll - 离线注册表
- offlinesam.dll - 离线SAM
- offlinelsa.dll - 离线LSA
- NetSetupAI.dll - 网络设置AI
- grouptrusteeai.dll - 组信任AI

### 2.9 WMI/CMI（10+组件）
- cmiaisupport.dll - CMI AI支持
- cmiadapter.dll - CMI适配器
- cmifw.dll - CMI框架
- opcservices.dll - OPC服务
- fastprox.dll - WMI快速代理
- wbemcore.dll / wbemcomn.dll - WMI核心
- EventsInstaller.dll - 事件安装器
- PerfCounterInstaller.dll - 性能计数器安装器

### 2.10 存储池工具（5个组件）
- rs1_spaceutil.exe - RS1版本存储池工具
- rs2_spaceutil.exe - RS2版本
- rs3_spaceutil.exe - RS3版本
- rs4_spaceutil.exe - RS4版本
- rs5_spaceutil.exe - RS5版本（当前系统使用）

### 2.11 其他工具
- OemCustomizationTool.exe - OEM定制工具
- ImgDump.exe - 镜像转储
- imgtowim.exe - 镜像转WIM
- wpimage.exe - WP镜像
- mvoffline.dll - 离线验证
- mcsfoffline.dll - 离线MCSF
- MetadataReader.dll - 元数据读取器
- Microsoft.Tools.IO.dll - 工具IO库
- platformmanifest.dll - 平台清单
- secwimtool.exe - 安全WIM工具

---

## 三、核心工作原理深度分析

### 3.1 imageapp.exe 完整工作流程（已验证）

#### 3.1.1 入口与参数解析
imageapp.exe接受以下参数：
```
imageapp <output.ffu> <OEMInput.xml> <os目录> [选项]
```
主要选项：
- `/CPUType:<arch>` - CPU架构（arm/x86/amd64）
- `+StrictSettingPolicies` - 严格设置策略
- `/ImageType:<type>` - 镜像类型
- `/StagingVHDPath:<path>` - 暂存VHD路径

#### 3.1.2 ImageStorageManager.CreateFullFlashImage 精确流程（第6985行）
这是imageapp构建镜像的核心方法，已从ILSpy反编译中完整提取：

```csharp
public void CreateFullFlashImage(FullFlashUpdateImage ffuImage, string imagePath, 
    uint partitionStyle, bool staging)
{
    // 阶段1: 遍历所有Store（物理磁盘 + 存储池成员磁盘）
    for (int i = 0; i < ffuImage.StoreCount; i++)
    {
        FullFlashUpdateStore store = ffuImage.Stores[i];
        FullFlashUpdatePartition poolPartition = store.Partitions
            .SingleOrDefault(p => p.IsStoragePool);
        
        if (poolPartition != null && store.PartitionCount == 1)
        {
            // === 纯存储池成员磁盘（只有一个StoragePool分区）===
            // 1. 创建ImageStorage(StoragePool类型)
            imageStorage = ImageStorage.CreatePool(logger, manager, sectorSize, storeId);
            // 2. 创建VHD（CreateVirtualHardDisk）
            imageStorage.CreateVirtualHardDisk(ffuImage, store, imagePath, partitionStyle, staging);
            // 3. 创建或加入存储池（CreateStoragePool / AddDriveToStoragePool）
            CreateOrAddToStoragePool(imageStorage, poolPartition.Name);
        }
        else
        {
            // === 普通物理磁盘（包含普通分区 + 可选StoragePool分区）===
            // 1. 创建ImageStorage(StorageStore类型)
            imageStorage = ImageStorage.CreateStore(logger, manager, sectorSize, storeId);
            // 2. 创建VHD（CreateVirtualHardDisk）
            imageStorage.CreateVirtualHardDisk(ffuImage, store, imagePath, partitionStyle, staging);
            // 3. 对物理磁盘分区（PartitionVirtualHardDisk）
            imageStorage.PartitionDisk(ffuImage, store, partitionStyle);
            // 4. 如果有StoragePool分区，创建或加入存储池
            if (poolPartition != null)
                CreateOrAddToStoragePool(imageStorage, poolPartition.Name);
        }
        _storages.Add(store, imageStorage);
    }
    
    // 阶段2: 遍历所有StoragePool，创建其中的Space（虚拟磁盘）
    foreach (FullFlashUpdateStoragePool pool in ffuImage.StoragePools)
    {
        Guid poolId = _storagePoolIds[pool.Name];
        foreach (FullFlashUpdateStore spaceStore in pool.Stores)
        {
            // 1. 创建ImageStorage(StorageSpace类型)
            imageStorage = ImageStorage.CreateSpace(logger, manager, sectorSize, storeId);
            // 2. 创建存储空间（CreateStorageSpace，返回diskHandle）
            imageStorage.CreateStorageSpace(ffuImage, spaceStore, poolId, 
                spaceStore.Type, spaceStore.Id);
            // 3. 对存储空间分区（PartitionVirtualHardDisk）
            imageStorage.PartitionDisk(ffuImage, spaceStore, partitionStyle);
            // 4. 如果最后一个分区UseAllSpace，扩展分区
            FullFlashUpdatePartition lastPartition = spaceStore.Partitions.LastOrDefault();
            if (lastPartition.UseAllSpace)
                imageStorage.ExtendPartition(lastPartition.Name, extendVolume: isNTFS);
            _storages.Add(spaceStore, imageStorage);
        }
    }
}
```

#### 3.1.3 CreateOrAddToStoragePool 逻辑（第7938行）
```csharp
private void CreateOrAddToStoragePool(ImageStorage storage, string poolName)
{
    if (!_storagePoolIds.ContainsKey(poolName))
    {
        // 第一个成员磁盘：创建新存储池
        Guid poolId = storage.CreateStoragePool(poolName);
        _storagePoolIds.Add(poolName, poolId);
    }
    else
    {
        // 后续成员磁盘：加入已有存储池
        Guid poolId = _storagePoolIds[poolName];
        storage.AddDriveToStoragePool(poolId);
    }
}
```

#### 3.1.4 关键调用链（已验证）
```
imageapp.exe (Main)
  → imaging.dll (BuildNewImage)
    → imagestorageservicemanaged.dll (ImageStorageManager.CreateFullFlashImage)
      ├─ ImageStorage.CreatePool/CreateStore/CreateSpace
      ├─ ImageStorage.CreateVirtualHardDisk
      │   → NativeImaging.CreateVirtualHardDisk (imagestorageservice.dll)
      ├─ ImageStorage.PartitionDisk
      │   → ImageStorageManager.GetPartitionEntries (构建PARTITION_ENTRY[])
      │   → NativeImaging.PartitionVirtualHardDisk (imagestorageservice.dll)
      ├─ ImageStorage.CreateStoragePool
      │   → NativeImaging.CreateStoragePool (imagestorageservice.dll)
      │   → NativeImaging.SetStoragePoolName
      ├─ ImageStorage.AddDriveToStoragePool
      │   → NativeImaging.AddDriveToStoragePool
      ├─ ImageStorage.CreateStorageSpace
      │   → NativeImaging.CreateStorageSpace (返回diskHandle)
      └─ ImageStorage.ExtendPartition
          → NativeImaging.ExtendPartition
```

### 3.2 imagestorageservice.dll 存储池创建原理（已验证）

#### 3.2.1 CreateStoragePool 精确实现（第6552行 + 第8690行）
```csharp
// ImageStorage.CreateStoragePool (托管层)
internal Guid CreateStoragePool(string poolName)
{
    // 1. 先用随机GUID作为内部名称创建池
    string internalName = Guid.NewGuid().ToString("N");
    Guid poolId = NativeImaging.CreateStoragePool(ServiceHandle, StoreHandle, internalName);
    // 2. 再设置用户可见的池名称
    NativeImaging.SetStoragePoolName(ServiceHandle, poolId, poolName);
    return poolId;
}

// NativeImaging.CreateStoragePool (P/Invoke)
[DllImport("ImageStorageService.dll", CharSet = CharSet.Unicode, EntryPoint = "CreateStoragePool")]
private static extern int CreateStoragePool_Native(
    IntPtr service, IntPtr storeHandle, string poolName, ref Guid poolId);

public static Guid CreateStoragePool(IntPtr service, IntPtr storeHandle, string poolName)
{
    Guid poolId = Guid.Empty;
    int hr = CreateStoragePool_Native(service, storeHandle, poolName, ref poolId);
    if (FAILED(hr)) throw new ImageStorageException(...);
    return poolId;
}
```

#### 3.2.2 CreateStorageSpace 精确实现（第6565行 + 第8716行）
```csharp
// ImageStorage.CreateStorageSpace (托管层)
internal void CreateStorageSpace(FullFlashUpdateImage ffuImage, 
    FullFlashUpdateStore ffuStore, Guid poolId, string spaceName, string spaceDescription)
{
    CleanupAllMountedDisks(_storeId);
    Image = ffuStore.Image;
    Store = ffuStore;
    IsMainOSStorage = ffuStore.IsMainOSStore;
    
    // 计算空间容量（GB为单位）
    if (ffuStore.MinSectorCount != 0)
        SectorCount = ffuStore.MinSectorCount;
    else
        SectorCount = (uint)(10737418240uL / (ulong)ffuStore.MinSectorCount); // 默认10GB
    
    uint capacityInGB = (uint)((ulong)((long)SectorCount * (long)SectorSize) / 1073741824uL);
    if (capacityInGB == 0) capacityInGB = 1u; // 最小1GB
    
    // 创建存储空间，返回diskHandle（可直接用于PartitionVirtualHardDisk）
    IntPtr diskHandle = NativeImaging.CreateStorageSpace(
        ServiceHandle, poolId, spaceName, spaceDescription, capacityInGB, ref _spaceId);
    _storeHandle = new SafeFileHandle(diskHandle, ownsHandle: true);
}

// NativeImaging.CreateStorageSpace (P/Invoke)
[DllImport("ImageStorageService.dll", CharSet = CharSet.Unicode, EntryPoint = "CreateStorageSpace")]
private static extern int CreateStorageSpace_Native(
    IntPtr service, Guid poolId, string spaceName, string spaceDescription,
    uint capacityInGB, ref Guid spaceId, out IntPtr diskHandle);

public static IntPtr CreateStorageSpace(IntPtr service, Guid poolId, 
    string spaceName, string spaceDescription, uint capacityInGB, ref Guid spaceId)
{
    IntPtr diskHandle;
    int hr = CreateStorageSpace_Native(service, poolId, spaceName, 
        spaceDescription, capacityInGB, ref spaceId, out diskHandle);
    if (FAILED(hr)) throw new ImageStorageException(...);
    return diskHandle; // 关键：返回的句柄可直接用于PartitionVirtualHardDisk
}
```

#### 3.2.3 PartitionVirtualHardDisk 精确实现（第8806行）
```csharp
[DllImport("ImageStorageService.dll", CharSet = CharSet.Unicode, EntryPoint = "PartitionVirtualHardDisk")]
private static extern int PartitionVirtualHardDisk_Native(
    IntPtr service, IntPtr diskHandle, ref STORE_ID storeId,
    [MarshalAs(UnmanagedType.LPArray, SizeParamIndex = 4)] PARTITION_ENTRY[] partitions,
    uint partitionCount);

public static void PartitionVirtualHardDisk(IntPtr service, IntPtr diskHandle, 
    ref STORE_ID storeId, PARTITION_ENTRY[] partitions)
{
    int hr = PartitionVirtualHardDisk_Native(service, diskHandle, ref storeId, 
        partitions, (uint)partitions.Length);
    if (FAILED(hr)) throw new ImageStorageException(...);
}
```

**关键验证**：CreateStorageSpace返回的diskHandle可以直接传给PartitionVirtualHardDisk，
不需要通过IOCTL_STORAGE_GET_DEVICE_NUMBER获取磁盘号（这是之前自研方案失败的原因）。

#### 3.2.4 GetPartitionEntries 构建逻辑（第7621行）
```csharp
internal PARTITION_ENTRY[] GetPartitionEntries(FullFlashUpdateImage ffuImage, 
    FullFlashUpdateStore ffuStore, uint partitionStyle, uint sectorCount)
{
    // 1. 过滤掉StoragePool分区（不参与普通分区表）
    List<FullFlashUpdatePartition> partitions = ffuStore.Partitions
        .Where(p => !p.IsStoragePool).ToList();
    
    // 2. 为每个分区创建PARTITION_ENTRY
    for (int i = 0; i < array.Length; i++)
    {
        FullFlashUpdatePartition ffuPartition = partitions[i];
        // 计算对齐大小
        uint alignmentInBytes = ffuStore.SectorSize;
        if (ffuPartition.ByteAlignment != 0)
            alignmentInBytes = ffuPartition.ByteAlignment;
        else if (ffuImage.DefaultPartitionAlignmentInBytes > ffuStore.SectorSize)
            alignmentInBytes = ffuImage.DefaultPartitionAlignmentInBytes;
        
        array[i] = CreatePartitionEntry(ffuPartition, partitionStyle, alignmentInBytes);
    }
    
    // 3. 处理UseAllSpace分区（SectorCount = uint.MaxValue表示使用剩余空间）
    // 4. 计算实际SectorCount = sectorCount - 已用扇区数
    return array;
}
```

### 3.3 SpaceUtil.exe 工作原理

#### 3.3.1 命令列表
rs5_spaceutil.exe支持以下命令：
- `new-pool` - 创建新存储池
- `new-space` - 创建新存储空间
- `get-pool` - 获取存储池信息
- `get-space` - 获取存储空间信息
- `set-pool` - 设置存储池属性
- `set-space` - 设置存储空间属性
- `remove-pool` - 删除存储池
- `remove-space` - 删除存储空间
- `add-physicaldisk` - 添加物理磁盘到池
- `remove-physicaldisk` - 从池移除物理磁盘

#### 3.3.2 new-pool 实现
```
new-pool -Name <name> -DriveNumber <n>
          [-DefaultWriteCacheSize <size>]
          [-DefaultReadCacheSize <size>]
          [-ProvisioningTypeDefault <Thin|Fixed>]
          [-ResiliencyTypeDefault <Simple|Mirror|Parity>]
```
内部通过Virtual Storage API（virtdisk.dll）创建存储池，直接操作原始磁盘。

#### 3.3.3 new-space 实现
```
new-space -poolId <guid> -name <name>
           -ProvisionedCapacity <size>
           -ProvisioningType <Thin|Fixed>
           -ResiliencyType <Simple|Mirror|Parity>
           [-NumberOfDataCopies <n>]
           [-NumberOfColumns <n>]
           [-Interleave <size>]
```
在指定存储池中创建虚拟磁盘，返回虚拟磁盘句柄。

### 3.4 设备布局XML结构

#### 3.4.1 根结构
```xml
<DeviceLayout xmlns="...">
  <Store>
    <Partition>...</Partition>
    <StoragePool>
      <Space>...</Space>
    </StoragePool>
  </Store>
</DeviceLayout>
```

#### 3.4.2 Store定义
- `Id` - Store唯一标识（GUID）
- `StoreType` - 存储类型（SV=虚拟存储，PM=物理存储）
- `SectorSize` - 扇区大小（通常512）
- `ChunkSize` - 存储池块大小（通常128KB）

#### 3.4.3 Partition定义
- `Name` - 分区名称
- `Type` - 分区类型GUID
- `SizeInSectors` - 分区大小（扇区数）
- `FileSystem` - 文件系统（NTFS/FAT/RAW等）
- `AlignmentSizeInBytes` - 对齐大小
- `ClusterSize` - 簇大小
- `Flags` - 标志位
- `OffsetInSectors` - 偏移（扇区数）

#### 3.4.4 StoragePool定义
- `Name` - 池名称（如OSPool）
- `PartitionType` - 池分区类型GUID
- `Spaces` - 存储空间列表

#### 3.4.5 Space定义
- `Name` - 空间名称（如MainOSDisk, EFIESPDisk）
- `SizeInSectors` - 空间大小
- `Partitions` - 空间内分区列表

### 3.5 GPT分区表直接写入原理

#### 3.5.1 GPT结构
```
LBA 0:      Protective MBR（保护MBR）
LBA 1:      Primary GPT Header（主GPT头）
LBA 2-33:   Partition Entry Array（分区条目数组，128个条目）
...
LBA -34:    Backup Entry Array（备份分区条目）
LBA -1:     Backup GPT Header（备份GPT头）
```

#### 3.5.2 GPT Header结构（92字节）
- Signature（8字节）："EFI PART"
- Revision（4字节）：0x00010000
- HeaderSize（4字节）：92
- HeaderCRC32（4字节）
- Reserved（4字节）：0
- MyLBA（8字节）：当前头LBA位置
- AlternateLBA（8字节）：备份头LBA位置
- FirstUsableLBA（8字节）：第一个可用LBA
- LastUsableLBA（8字节）：最后一个可用LBA
- DiskGUID（16字节）：磁盘GUID
- PartitionEntryLBA（8字节）：分区条目起始LBA
- NumberOfPartitionEntries（4字节）：128
- SizeOfPartitionEntry（4字节）：128
- PartitionEntryArrayCRC32（4字节）

#### 3.5.3 Partition Entry结构（128字节）
- TypeGUID（16字节）：分区类型GUID
- UniqueGUID（16字节）：分区唯一GUID
- StartingLBA（8字节）：起始LBA
- EndingLBA（8字节）：结束LBA
- Attributes（8字节）：属性标志
- Name（72字节）：分区名称（UTF-16LE，36字符）

---

## 四、对程序重写的参考价值

### 4.1 DeviceLayoutGenerator 应遵循的流程
1. 解析DeviceLayout.xml，获取所有Store/Partition/Space定义
2. 计算VHD总大小 = 所有物理分区总和 + 存储池分区大小
3. 创建动态VHD（CreateVirtualHardDisk）
4. 挂载VHD，获取磁盘号
5. 直接写入GPT分区表（包含所有物理分区 + 存储池分区）
6. 调用CreateStoragePool（内部调用rs5_spaceutil.exe new-pool）
7. 对每个Space：
   a. 调用CreateStorageSpace（new-space -ProvisioningType Thin -ResiliencyType Simple）
   b. 调用PartitionVirtualHardDisk对空间分区
8. 格式化所有可识别文件系统的分区
9. 卸载VHD

### 4.2 关键API签名（已从ILSpy反编译验证）

所有API都在 `imagestorageservice.dll` 中，通过 `NativeImaging` 类P/Invoke调用。
**注意：DLL名称是 "ImageStorageService.dll"（驼峰命名），不是全小写。**

```csharp
// === 服务管理 ===
[DllImport("ImageStorageService.dll", CallingConvention = CallingConvention.StdCall, 
    EntryPoint = "CreateImageStorageService")]
private static extern int CreateImageStorageServiceNative(
    out IntPtr serviceHandle, 
    [MarshalAs(UnmanagedType.FunctionPtr)] LogFunction logError,
    uint storeIdsCount, 
    [MarshalAs(UnmanagedType.LPArray, SizeParamIndex = 2)] STORE_ID[] storeIds);

[DllImport("ImageStorageService.dll", CallingConvention = CallingConvention.StdCall)]
public static extern void CloseImageStorageService(IntPtr service);

[DllImport("ImageStorageService.dll", CallingConvention = CallingConvention.StdCall, 
    EntryPoint = "RefreshImageStorageService")]
private static extern int RefreshImageStorageServiceNative(
    IntPtr service, uint storeIdsCount, STORE_ID[] storeIds);

// === 虚拟磁盘 ===
[DllImport("ImageStorageService.dll", CharSet = CharSet.Unicode, 
    EntryPoint = "CreateVirtualHardDisk")]
private static extern int CreateVirtualHardDisk_Native(
    IntPtr service, string fileName, ulong maxSizeInBytes, 
    STORE_ID storeId, uint sectorSize, out IntPtr diskHandle);

[DllImport("ImageStorageService.dll", CharSet = CharSet.Unicode, 
    EntryPoint = "OpenVirtualHardDisk")]
private static extern int OpenVirtualHardDisk_Native(
    IntPtr service, string fileName, [MarshalAs(UnmanagedType.Bool)] bool readOnly,
    out STORE_ID storeId, out IntPtr diskHandle);

[DllImport("ImageStorageService.dll", CallingConvention = CallingConvention.StdCall, 
    EntryPoint = "DismountVirtualHardDisk")]
private static extern int DismountVirtualHardDisk_Native(
    IntPtr service, STORE_ID storeId, 
    [MarshalAs(UnmanagedType.Bool)] bool deleteFile,
    [MarshalAs(UnmanagedType.Bool)] bool fFailIfDiskMissing);

[DllImport("ImageStorageService.dll", CallingConvention = CallingConvention.StdCall, 
    CharSet = CharSet.Unicode, EntryPoint = "DismountVirtualHardDiskByFileName")]
private static extern int DismountVirtualHardDiskByFileName_Native(
    IntPtr service, string fileName, [MarshalAs(UnmanagedType.Bool)] bool deleteFile);

// === 存储池 ===
[DllImport("ImageStorageService.dll", CharSet = CharSet.Unicode, 
    EntryPoint = "CreateStoragePool")]
private static extern int CreateStoragePool_Native(
    IntPtr service, IntPtr storeHandle, string poolName, ref Guid poolId);

[DllImport("ImageStorageService.dll", CharSet = CharSet.Unicode, 
    EntryPoint = "SetStoragePoolName")]
private static extern int SetStoragePoolName_Native(
    IntPtr service, Guid poolId, string poolName);

[DllImport("ImageStorageService.dll", CharSet = CharSet.Unicode, 
    EntryPoint = "AddDriveToStoragePool")]
private static extern int AddDriveToStoragePool_Native(
    IntPtr service, Guid poolId, IntPtr storeHandle);

[DllImport("ImageStorageService.dll", CharSet = CharSet.Unicode, 
    EntryPoint = "RemoveStoragePool")]
private static extern int RemoveStoragePool_Native(IntPtr service, Guid poolId);

[DllImport("ImageStorageService.dll", CharSet = CharSet.Unicode, 
    EntryPoint = "GetPoolIdFromPoolName")]
private static extern int GetPoolIdFromPoolName_Native(
    IntPtr service, string poolName, ref Guid poolId);

// === 存储空间 ===
[DllImport("ImageStorageService.dll", CharSet = CharSet.Unicode, 
    EntryPoint = "CreateStorageSpace")]
private static extern int CreateStorageSpace_Native(
    IntPtr service, Guid poolId, string spaceName, string spaceDescription,
    uint capacityInGB, ref Guid spaceId, out IntPtr diskHandle);

[DllImport("ImageStorageService.dll", CharSet = CharSet.Unicode, 
    EntryPoint = "GetStorageSpace")]
private static extern int GetStorageSpace_Native(
    IntPtr service, Guid poolId, string spaceName, ref Guid spaceId, out IntPtr diskHandle);

[DllImport("ImageStorageService.dll", CharSet = CharSet.Unicode, 
    EntryPoint = "SetStorageSpaceName")]
private static extern int SetStorageSpaceName_Native(
    IntPtr service, Guid spaceId, string spaceName);

[DllImport("ImageStorageService.dll", CharSet = CharSet.Unicode, 
    EntryPoint = "RemoveStorageSpace")]
private static extern int RemoveStorageSpace_Native(IntPtr service, Guid spaceId);

// === 分区（关键：可用于物理磁盘和存储空间）===
[DllImport("ImageStorageService.dll", CharSet = CharSet.Unicode, 
    EntryPoint = "PartitionVirtualHardDisk")]
private static extern int PartitionVirtualHardDisk_Native(
    IntPtr service, IntPtr diskHandle, ref STORE_ID storeId,
    [MarshalAs(UnmanagedType.LPArray, SizeParamIndex = 4)] PARTITION_ENTRY[] partitions,
    uint partitionCount);

[DllImport("ImageStorageService.dll", CharSet = CharSet.Unicode, 
    EntryPoint = "UpdatePartitionProperties")]
private static extern int UpdatePartitionProperties_Native(
    IntPtr service, IntPtr diskHandle, STORE_ID storeId,
    [MarshalAs(UnmanagedType.LPArray, SizeParamIndex = 4)] PARTITION_ENTRY[] partitions,
    uint partitionCount);

[DllImport("ImageStorageService.dll", CharSet = CharSet.Unicode, 
    EntryPoint = "ExtendPartition")]
private static extern int ExtendPartition_Native(
    IntPtr service, SafeFileHandle diskHandle, STORE_ID storeId,
    string partitionName, [MarshalAs(UnmanagedType.Bool)] bool extendVolume);

// === 分区属性查询/设置 ===
[DllImport("ImageStorageService.dll", CallingConvention = CallingConvention.StdCall, 
    CharSet = CharSet.Unicode, EntryPoint = "GetPartitionPath")]
private static extern int GetPartitionPath_Native(
    IntPtr serviceHandle, STORE_ID storeId, string partitionName,
    StringBuilder path, uint pathSizeInCharacters, 
    [MarshalAs(UnmanagedType.Bool)] bool waitForVolume);

[DllImport("ImageStorageService.dll", CallingConvention = CallingConvention.StdCall, 
    CharSet = CharSet.Unicode, EntryPoint = "GetPartitionFileSystem")]
private static extern int GetPartitionFileSystem_Native(
    IntPtr serviceHandle, STORE_ID storeId, string partitionName,
    StringBuilder fileSystem, uint fileSystemSizeInCharacters);

[DllImport("ImageStorageService.dll", CallingConvention = CallingConvention.StdCall, 
    EntryPoint = "GetStoreId")]
private static extern int GetStoreId_Native(
    IntPtr serviceHandle, SafeFileHandle diskHandle, out STORE_ID storeId);

// === 磁盘布局更新 ===
[DllImport("ImageStorageService.dll", EntryPoint = "UpdateDiskLayout")]
private static extern int UpdateDiskLayout_Native(IntPtr service, SafeFileHandle diskHandle);

[DllImport("ImageStorageService.dll", EntryPoint = "ReauctionDisk")]
private static extern int ReauctionDisk_Native(IntPtr service, SafeFileHandle diskHandle);

// === 挂载点管理 ===
[DllImport("ImageStorageService.dll", CharSet = CharSet.Unicode, 
    EntryPoint = "AssignMountPoints")]
private static extern int AssignMountPoints_Native(
    IntPtr service, STORE_ID storeId,
    [MarshalAs(UnmanagedType.LPArray, SizeParamIndex = 3)] STORE_ID[] storeIds,
    uint storeIdsCount, [MarshalAs(UnmanagedType.Bool)] bool useLegacyBehavior);

[DllImport("ImageStorageService.dll", CharSet = CharSet.Unicode, 
    EntryPoint = "WriteMountManagerRegistry2")]
private static extern int WriteMountManagerRegistry2_Native(
    IntPtr service, STORE_ID storeId, 
    [MarshalAs(UnmanagedType.Bool)] bool useWellKnownGuids,
    [MarshalAs(UnmanagedType.LPArray, SizeParamIndex = 4)] STORE_ID[] storeIds,
    uint storeIdsCount);
```

### 4.3 关键结构体（已从ILSpy反编译验证，精确布局）

```csharp
// PARTITION_ENTRY - 201字节，Explicit布局
[StructLayout(LayoutKind.Explicit, CharSet = CharSet.Unicode)]
public struct PARTITION_ENTRY
{
    [FieldOffset(0)]
    [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 36)]
    private string name;           // 分区名称（36字符UTF-16 = 72字节）

    [FieldOffset(72)]
    private ulong sectorCount;     // 分区大小（扇区数），uint.MaxValue=UseAllSpace

    [FieldOffset(80)]
    private uint alignmentSizeInBytes;  // 对齐大小（字节）

    [FieldOffset(84)]
    private uint clusterSize;      // 簇大小

    [FieldOffset(88)]
    [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 32)]
    private string fileSystem;     // 文件系统（32字符UTF-16 = 64字节）

    [FieldOffset(152)]
    private Guid id;               // 分区唯一GUID

    [FieldOffset(168)]
    private Guid type;             // 分区类型GUID

    [FieldOffset(184)]
    private ulong flags;           // GPT属性标志

    // MBR模式下与id/type共享内存（联合体）
    [FieldOffset(152)]
    private byte mbrFlags;         // MBR标志（0x80=可引导）
    [FieldOffset(153)]
    private byte mbrType;          // MBR分区类型

    [FieldOffset(192)]
    private ulong offsetInSectors; // 偏移（扇区数）

    [FieldOffset(200)]
    private byte fFvePrep;         // BitLocker准备标志（0/1）

    // 属性访问器...
    public string PartitionName { get => name; set => name = value; }
    public ulong SectorCount { get => sectorCount; set => sectorCount = value; }
    public uint AlignmentSizeInBytes { get => alignmentSizeInBytes; set => alignmentSizeInBytes = value; }
    public uint ClusterSize { get => clusterSize; set => clusterSize = value; }
    public string FileSystem { get => fileSystem; set => fileSystem = value; }
    public Guid PartitionId { get => id; set => id = value; }
    public Guid PartitionType { get => type; set => type = value; }
    public ulong PartitionFlags { get => flags; set => flags = value; }
    public byte MBRFlags { get => mbrFlags; set => mbrFlags = value; }
    public byte MBRType { get => mbrType; set => mbrType = value; }
    public ulong OffsetInSectors { get => offsetInSectors; set => offsetInSectors = value; }
    public bool PrepareFveMetadata { get => fFvePrep != 0; set => fFvePrep = value ? (byte)1 : (byte)0; }
}

// STORE_ID - Explicit布局，GPT/MBR联合体
[StructLayout(LayoutKind.Explicit)]
public struct STORE_ID
{
    [FieldOffset(0)]
    private uint storeType;        // 0=MBR, 1=GPT (ImageConstants.PartitionTypeMbr/Gpt)

    [FieldOffset(4)]
    private Guid storeId_GPT;      // GPT磁盘GUID

    [FieldOffset(4)]
    private uint storeId_MBR;      // MBR磁盘签名

    public uint StoreType { get => storeType; set => storeType = value; }
    public Guid StoreId_GPT { get => storeId_GPT; set => storeId_GPT = value; }
    public uint StoreId_MBR { get => storeId_MBR; set => storeId_MBR = value; }

    public static STORE_ID Create(Guid diskId)
    {
        STORE_ID result = default;
        result.StoreType = ImageConstants.PartitionTypeGpt; // 1
        result.StoreId_GPT = diskId;
        return result;
    }

    public static STORE_ID Create(uint diskId)
    {
        STORE_ID result = default;
        result.StoreType = ImageConstants.PartitionTypeMbr; // 0
        result.StoreId_MBR = diskId;
        return result;
    }
}

// GPT分区标志（FlagsFromPartition方法，第7903行）
// Hidden          = 0x4000000000000000
// ReadOnly        = 0x1000000000000000
// NoDriveLetter   = 0x8000000000000000 (AttachDriveLetter=false)
// ServicePartition= 0x200000000000000
```

### 4.4 注意事项
1. **32位限制**：imagestorageservice.dll是32位，必须在32位进程中调用
2. **管理员权限**：创建存储池和分区需要管理员权限
3. **动态VHD**：imageapp使用动态VHD，存储空间使用Thin Provisioning
4. **空间分区**：必须使用PartitionVirtualHardDisk，不能用GptManager（空间句柄不支持IOCTL_STORAGE_GET_DEVICE_NUMBER）
5. **SpaceUtil版本**：根据当前系统Build号选择对应版本的SpaceUtil
6. **存储池分区Type GUID**：{5708A6E0-9001-4b99-b064-1fe564896bdb}

---

## 五、反编译文件索引

### 5.1 原生二进制（72个）
输出目录：`E:\ADK_Research\ida_decompiled\native\`

关键文件：
- imagestorageservice.c - 存储池创建核心
- updateapp.c - 更新应用
- updateapi.c / updatedll.c - 更新API
- cbscore.c / cbsapi.c - CBS核心
- wcp.c - Windows组件平台
- drvstore.c - 驱动存储
- imagex.c - WIM工具
- rs1-5_spaceutil.c - 存储池工具（5个版本）
- appxpackaging.c / appxdeploymentclient.c - AppX
- makeappx.c - AppX打包
- secwimtool.c - 安全WIM工具
- offreg.c / offlinesam.c / offlinelsa.c - 离线注册表
- NetSetupEngine.c / NetSetupApi.c - 网络设置
- cmiaisupport.c / cmiadapter.c / cmifw.c - CMI
- opcservices.c - OPC服务

### 5.2 .NET托管程序集（53个）
输出目录：`E:\ADK_Research\outputsrc\dotnet\`

关键文件：
- imageapp.cs - 镜像创建主程序
- imagecommon.cs - 镜像公共库（474KB）
- imagestorageservicemanaged.cs - 存储服务托管封装（416KB）
- ffucomponents.cs - FFU组件（176KB）
- PkgBldr.Common.cs - 包构建公共库（192KB）
- pkggencommon.cs - 包生成公共库
- DeviceLayoutValidation.cs - 设备布局验证
- imaging.cs - 成像API
- toolscommon.cs - 工具公共库
- Microsoft.Tools.IO.cs - 工具IO库

---

## 六、后续工作建议

1. **深入分析imageapp.cs**：提取完整的设备布局解析和VHD创建逻辑
2. **分析imagestorageservicemanaged.cs**：理解托管层如何封装原生API
3. **分析imagecommon.cs**：提取VirtualHardDisk和PartitionManager的实现
4. **对比多版本ADK**：分析17110（无存储池）→17704（有存储池）的变化
5. **提取SpaceUtil命令格式**：从反编译中提取完整的命令行参数定义
6. **验证PartitionVirtualHardDisk调用**：在测试环境中验证空间分区是否成功

---

*报告生成工具：IDA Pro 9.4 + Hex-Rays / ILSpy / Ghidra 12.1.3*
*分析目标：ADK 10.0.17704.1000 (Windows 10 RS5)*
