# ADK 17704 imagestorageservice.dll 深度研究 — 存储服务原生实现

> 生成日期：2026-08-27
> 研究范围：81个导出函数、VHD创建/分区/存储池/存储空间、VDS COM API、virtio API
> 数据来源：PE导出表分析 + IDA Hex-Rays反编译 (imagestorageservice.c) + 托管封装层分析

---

## 一、DLL 概览

### 1.1 基本信息

| 属性 | 值 |
|------|-----|
| 文件名 | imagestorageservice.dll |
| 大小 | 286,720 字节 (280KB) |
| 架构 | 32位 (x86) ★必须在32位进程中调用 |
| 导出函数 | 81 个 |
| 调用约定 | stdcall (名称修饰: _FunctionName@N) |
| 依赖 | virtdisk.dll (VHD API), vds_ps.dll (VDS COM), kernel32.dll |

### 1.2 服务句柄验证

所有函数第一个参数都是服务句柄指针，验证 magic value：
```c
// 验证模式 (所有导出函数通用)
if (param_1 == NULL || param_1[1] != -0x2c691091) {
    return -0x7ff8ffa9;  // E_INVALIDARG
}
iVar1 = param_1[2];  // logger 指针
```

---

## 二、81 个导出函数完整分类

### 2.1 服务生命周期 (4)

| 函数 | 序数 | 功能 |
|------|------|------|
| `_CreateImageStorageService@16` | 14 | 创建存储服务实例 |
| `_CreateStorageService@16` | 15 | 创建存储服务（别名） |
| `_CloseImageStorageService@4` | 6 | 关闭存储服务 |
| `_RefreshImageStorageService@12` | 57 | 刷新服务状态 |

### 2.2 VHD 操作 (8)

| 函数 | 序数 | 功能 |
|------|------|------|
| `_CreateVirtualHardDisk@44` | 14 | 创建 VHD/VHDX |
| `_OpenVirtualHardDisk@20` | 54 | 打开已有 VHD |
| `_DismountVirtualHardDisk@32` | 15 | 卸载 VHD |
| `_DismountVirtualHardDiskByFileName@12` | 16 | 按文件名卸载 VHD |
| `_AttachToMountedImage@20` | 3 | 附加到已挂载镜像 |
| `_GetVirtualHardDiskFileName@32` | 49 | 获取 VHD 文件名 (IOCTL) |
| `_GetVirtualDiskImagePath@16` | 48 | 获取虚拟磁盘镜像路径 |
| `_GetVirtualHardDiskMetadata@32` | 50 | 获取 VHD 元数据 |
| `_SetVirtualHardDiskMetadata@32` | 74 | 设置 VHD 元数据 |
| `_CreateDiffDisk@8` | 8 | 创建差异磁盘 |

### 2.3 分区操作 (14)

| 函数 | 序数 | 功能 |
|------|------|------|
| `_PartitionVirtualHardDisk@20` | 56 | 分区 VHD (VDS COM) |
| `_ExtendPartition@36` | 17 | 扩展分区 |
| `_FormatPartition@36` | 19 | 格式化分区 |
| `_FormatPartitions@36` | 20 | 格式化所有分区 |
| `_FormatPartitionWithPath@12` | 21 | 按路径格式化分区 |
| `_FlushPartition@12` | 22 | 刷新分区 |
| `_SetPartitionId@28` | 72 | 设置分区 ID (GPT GUID) |
| `_SetPartitionType@44` | 75 | 设置分区类型 |
| `_SetPartitionAttributes@36` | 73 | 设置分区属性 |
| `_RemovePartitionAttributes@20` | 60 | 移除分区属性 |
| `_UpdatePartitionProperties@36` | 76 | 更新分区属性 (VDS COM) |
| `_GetPartitionAttributes@32` | 35 | 获取分区属性 |
| `_GetPartitionType@32` | 40 | 获取分区类型 |
| `_GetPartitionFileSystem@36` | 37 | 获取分区文件系统 |

### 2.4 存储池 (8)

| 函数 | 序数 | 功能 |
|------|------|------|
| `_CreateStoragePool@16` | 10 | 创建存储池 |
| `_AddDriveToStoragePool@24` | 1 | 添加磁盘到存储池 |
| `_RemoveStoragePool@20` | 60 | 移除存储池 |
| `_SetStoragePoolName@24` | 71 | 设置存储池名称 |
| `_GetPoolIdFromPoolName@12` | 39 | 按名称获取池 ID |
| `_GetStorageAllocationBitmap@20` | 43 | 获取存储分配位图 |
| `_GetBlockAllocationBitmap@40` | 27 | 获取块分配位图 |
| `_IsStoreSpace@28` | 44 | 判断是否为存储空间 |

### 2.5 存储空间 (6)

| 函数 | 序数 | 功能 |
|------|------|------|
| `_CreateStorageSpace@40` | 12 | 创建存储空间 (虚拟磁盘) |
| `_RemoveStorageSpace@20` | 61 | 移除存储空间 |
| `_SetStorageSpaceName@24` | 72 | 设置存储空间名称 |
| `_GetStorageSpace@32` | 44 | 获取存储空间信息 |
| `_GetMainOSStoreId@4` | 31 | 获取 MainOS 存储 ID |
| `_GetStoreId@12` | 45 | 获取存储 ID |

### 2.6 卷/挂载点 (12)

| 函数 | 序数 | 功能 |
|------|------|------|
| `_AssignMountPoints@36` | 2 | 分配挂载点 (MainOS) |
| `_NormalizeVolumeMountPoints@40` | 55 | 规范化卷挂载点 |
| `_WriteVolumeMountPoints@20` | 80 | 写入卷挂载点 |
| `_WriteMountManagerRegistry@24` | 78 | 写入挂载管理器注册表 |
| `_WriteMountManagerRegistry2@36` | 79 | 写入挂载管理器注册表 v2 |
| `_CreateJunction@40` | 9 | 创建联接点 |
| `_OpenVolume@44` | 56 | 打开卷 |
| `_CloseVolumeHandle@8` | 7 | 关闭卷句柄 |
| `_LockAndDismountVolumeByHandle@12` | 47 | 锁定并卸载卷 |
| `_UnlockVolumeByHandle@8` | 77 | 解锁卷 |
| `_AttachWOFToVolume@8` | 4 | 附加 WOF (Windows Overlay Filter) |
| `_CreateUsnJournal@28` | 13 | 创建 USN 日志 |

### 2.7 查询/信息 (18)

| 函数 | 序数 | 功能 |
|------|------|------|
| `_GetDiskLayout@12` | 28 | 获取磁盘布局 |
| `_GetDiskName@32` | 29 | 获取磁盘名称 |
| `_GetDiskSize@12` | 30 | 获取磁盘大小 |
| `_GetDiskNumAndPartitionCount@40` | 30 | 获取磁盘号和分区数 |
| `_GetPartitionStyle@12` | 38 | 获取分区样式 (GPT/MBR) |
| `_GetPartitionOffset@32` | 36 | 获取分区偏移 |
| `_GetPartitionSizeInSectors@32` | 39 | 获取分区大小(扇区) |
| `_GetPartitionPath@40` | 38 | 获取分区路径 |
| `_GetPartitionPathNoContext@16` | 37 | 获取分区路径(无上下文) |
| `_GetFreeBytesOnVolume@32` | 32 | 获取卷空闲字节 |
| `_GetSectorCount@12` | 41 | 获取扇区数 |
| `_GetSectorSize@28` | 42 | 获取扇区大小 |
| `_GetSectorSizeFromHandle@12` | 42 | 从句柄获取扇区大小 |
| `_GetStoreIdByPath@8` | 46 | 按路径获取存储 ID |
| `_GetStoreIdFromPartitionName@16` | 46 | 按分区名获取存储 ID |
| `_GetETWLogPath@12` | 30 | 获取 ETW 日志路径 |
| `_ScanPartitionPath@24` | 58 | 扫描分区路径 |
| `_SafeFreeDiskLayout@4` | 62 | 安全释放磁盘布局 |

### 2.8 磁盘状态 (6)

| 函数 | 序数 | 功能 |
|------|------|------|
| `_SetDiskId@28` | 70 | 设置磁盘 ID |
| `_SetDiskOffline@8` | 67 | 设置磁盘离线 |
| `_SetDiskOnline@8` | 68 | 设置磁盘在线 |
| `_ReauctionDisk@8` | 59 | 重新拍卖磁盘 |
| `_UpdateDiskLayout@8` | 77 | 更新磁盘布局 |
| `_EnableAutoMount@4` | 18 | 启用自动挂载 |

### 2.9 日志 (1)

| 函数 | 序数 | 功能 |
|------|------|------|
| `_SetLoggingFunction@12` | 69 | 设置日志回调函数 |

---

## 三、核心函数深度分析

### 3.1 CreateVirtualHardDisk — 创建 VHD

**调用链**：
```
_CreateVirtualHardDisk@44(serviceHandle, fileName, maxSize, storeId, sectorSize, phDisk)
  │
  ├─ 1. 验证服务句柄 (magic -0x2c691091)
  │
  ├─ 2. 验证 storeId.partitionStyle (0=MBR, 1=GPT)
  │
  ├─ 3. 获取卷信息 (GetVolumePathNameW → GetDriveTypeW)
  │   ├─ 可移动驱动器 → 警告日志
  │   └─ 远程驱动器 → 警告日志
  │
  ├─ 4. 获取卷 GUID (GetVolumeNameForVolumeMountPointW)
  │
  ├─ 5. OpenDiskInternal() → 核心创建
  │   ├─ 判断扩展名 .vhdx → VIRTUAL_STORAGE_TYPE_VENDOR_MICROSOFT, DeviceType=VHDX
  │   ├─ 否则 → DeviceType=VHD
  │   ├─ OpenVirtualDisk() → 如果文件已存在
  │   └─ CreateVirtualDisk() → 创建新 VHD
  │       ├─ VIRTUAL_DISK_ACCESS_ALL (0x3f0000)
  │       ├─ CREATE_VIRTUAL_DISK_FLAG_NONE
  │       └─ FixedSize = maxSize
  │
  ├─ 6. AttachVirtualDisk() → 附加 VHD 到系统
  │   ├─ ATTACH_VIRTUAL_DISK_FLAG_PERMANENT_LIFETIME
  │   └─ 重试最多5次 (ERROR_DEVICE_NOT_AVAILABLE = 0x79)
  │
  ├─ 7. GetVirtualDiskPhysicalPath() → 获取物理路径 (\\.\PhysicalDriveN)
  │
  ├─ 8. WaitForOnlineDisk() → 等待磁盘上线
  │
  ├─ 9. _EnableAutoMount() → 启用自动挂载
  │
  └─ 10. 返回磁盘句柄 phDisk
```

**关键参数**：
```c
// VIRTUAL_STORAGE_TYPE
typedef struct _VIRTUAL_STORAGE_TYPE {
    ULONG DeviceId;      // VIRTUAL_STORAGE_TYPE_DEVICE_VHD=2, VHDX=3
    GUID  VendorId;      // VIRTUAL_STORAGE_TYPE_VENDOR_MICROSOFTO
} VIRTUAL_STORAGE_TYPE;
```

### 3.2 PartitionVirtualHardDisk — 分区 VHD (VDS COM)

**核心实现**：使用 **VDS (Virtual Disk Service) COM API** 进行分区操作。

```
_PartitionVirtualHardDisk@20(serviceHandle, hDisk, storeId, partitions, count)
  │
  ├─ 1. GPT 路径 (storeId.partitionStyle == 1)
  │   ├─ IVdsDisk::SetDiskAttributes() → 设置 GPT 磁盘属性
  │   ├─ IVdsDisk::GetProperties() → 获取磁盘属性
  │   ├─ IVdsDisk::PartitionDiskEx() → ★ 分区磁盘 (扩展API)
  │   │   └─ VDS_PARTITION_INFO_GPT 数组
  │   ├─ IVdsDisk::SetDiskAttributes() → 清除属性
  │   ├─ WaitForOnlineDisk() → 等待上线
  │   ├─ WaitForVolumesReady() → 等待卷就绪
  │   ├─ VolumeTrackerList::WaitForVolume() → 等待每个卷
  │   ├─ VolumeTrackerList::UpdatePartitionNames() → 更新分区名
  │   └─ _FormatPartitions() → 格式化所有分区
  │
  └─ 2. MBR 路径 (storeId.partitionStyle == 0)
      ├─ DeviceIoControl(IOCTL_DISK_CREATE_DISK) → 创建 MBR 磁盘
      ├─ 分配分区条目数组
      ├─ SetDiskAttributes() → 设置属性
      ├─ IVdsDisk::PartitionDiskEx() → 分区
      ├─ SetDiskAttributes() → 清除属性
      ├─ WaitForOnlineDisk() → 等待上线
      ├─ WaitForVolumesReady() → 等待卷就绪
      ├─ GetStoreId() → 获取存储 ID
      ├─ VolumeTrackerList::WaitForVolume() → 等待卷
      ├─ VolumeTrackerList::UpdatePartitionNames() → 更新分区名
      └─ _FormatPartitions() → 格式化
```

**VDS_PARTITION_INFO_GPT 结构**：
```c
typedef struct _VDS_PARTITION_INFO_GPT {
    GUID     partitionType;     // 分区类型 GUID
    GUID     partitionId;       // 分区唯一 GUID
    ULONGLONG ullSize;          // 分区大小
    ULONG    ulAttributes;      // GPT 属性标志
    LPWSTR   pwszName;          // 分区名称
} VDS_PARTITION_INFO_GPT;
```

### 3.3 CreateStoragePool — 创建存储池

```
_CreateStoragePool@16(serviceHandle, hDisk, poolName, pPoolId)
  │
  ├─ 1. DeviceIoControl(hDisk, IOCTL 0x2d1080) → 获取存储池句柄
  │   (IOCTL_STORAGE_MANAGE_DATA_SET_ATTRIBUTES 变体)
  │
  └─ 2. CreatePoolHelper() → 创建存储池
      ├─ 设置池名称
      ├─ 生成池 ID (GUID)
      └─ 调用存储池管理 API
```

**存储池分区 Type GUID**：
```
{5708A6E0-9001-4b99-b064-1fe564896bdb}
```

### 3.4 CreateStorageSpace — 创建存储空间

```
_CreateStorageSpace@40(serviceHandle, poolId, spaceName, description,
                         capacityGB, pSpaceId, phDisk)
  │
  └─ CreateSpaceHelper() → 创建存储空间
      ├─ 输入: 池 ID, 空间名, 描述, 容量(GB)
      ├─ 输出: 空间 ID (GUID), 磁盘句柄
      └─ 返回的磁盘句柄可直接传给 PartitionVirtualHardDisk
```

**关键特性**：
- Thin Provisioning（精简配置）
- Simple Resiliency（简单复原，无冗余）
- 返回的 diskHandle 是虚拟磁盘句柄，可像物理磁盘一样分区

### 3.5 OpenVirtualHardDisk — 打开已有 VHD

```
_OpenVirtualHardDisk@20(serviceHandle, fileName, readOnly, pStoreId, phDisk)
  │
  ├─ 1. OpenDiskInternal() → 打开 VHD
  │   ├─ OpenVirtualDisk(VIRTUAL_DISK_ACCESS_NONE)
  │   └─ 附加到系统
  │
  ├─ 2. GetDiskLayoutInternal() → 获取磁盘布局
  │
  ├─ 3. 构建 STORE_ID (GPT GUID 或 MBR 签名)
  │
  ├─ 4. GetPartitionList() → 获取分区列表
  │
  ├─ 5. _WaitForPartitions() → 等待所有分区就绪
  │
  └─ 6. 返回磁盘句柄和 STORE_ID
```

### 3.6 AssignMountPoints — 分配挂载点

```
_AssignMountPoints@36(serviceHandle)
  │
  ├─ 1. GetVolumeFromPartitionName("MAINOS") → 获取 MainOS 卷
  │
  ├─ 2. 获取卷 GUID 路径 (\\?\Volume{GUID}\)
  │
  ├─ 3. 移除 MainOS 已有挂载点
  │
  ├─ 4. CreateVolumeMountPointChangeNotification() → 创建变更通知
  │
  ├─ 5. SetVolumeMountPoint(mountPath, volumeGuid) → 设置挂载点
  │
  ├─ 6. WaitForNewMountPoint() → 等待新挂载点生效
  │
  └─ 7. WriteVolumeMountPoints2() → 写入卷挂载点配置
```

### 3.7 UpdatePartitionProperties — 更新分区属性

```
_UpdatePartitionProperties@36(serviceHandle, hDisk, partitionArray)
  │
  ├─ 1. 分配分区属性通知数组
  │
  ├─ 2. IVdsDisk::GetPartitionList() → 获取当前分区列表
  │
  ├─ 3. 对每个分区:
  │   ├─ 按名称匹配分区
  │   ├─ 比较分区类型 (GPT GUID)
  │   ├─ 比较分区属性 (GPT attributes)
  │   ├─ 如果类型不同:
  │   │   ├─ IVdsDisk::SetPartitionType() → 更新类型
  │   │   └─ 创建分区类型变更通知
  │   ├─ 如果属性不同:
  │   │   ├─ IVdsDisk::SetPartitionAttributes() → 更新属性
  │   │   └─ 创建分区属性变更通知
  │   └─ 记录变更
  │
  ├─ 4. IVdsDisk::SetDiskLayout() → 写入磁盘布局
  │
  ├─ 5. 等待所有分区通知到达
  │
  └─ 6. 清理通知数组
```

### 3.8 GetVirtualHardDiskFileName — IOCTL 查询

```
_GetVirtualHardDiskFileName@32(serviceHandle, hDisk, diskName, size)
  │
  ├─ 1. GetDiskHandleInternal() → 获取磁盘句柄
  │
  └─ 2. DeviceIoControl(hDisk, IOCTL_STORAGE_QUERY_VHD_FILE_NAME)
       ├─ IOCTL code: 0x2d5928
       ├─ 输入: 无
       └─ 输出: VHD 文件路径字符串
```

---

## 四、关键 IOCTL 代码

| IOCTL | 值 | 功能 |
|-------|-----|------|
| `IOCTL_DISK_CREATE_DISK` | 0x000700C0 | 创建 MBR/GPT 磁盘 |
| `IOCTL_STORAGE_QUERY_VHD_FILE_NAME` | 0x002D5928 | 查询 VHD 文件名 |
| `IOCTL_STORAGE_MANAGE_DATA_SET_ATTRIBUTES` (存储池) | 0x002D1080 | 存储池管理 |

---

## 五、STORE_ID 结构

```c
// GPT 模式
typedef struct _STORE_ID_GPT {
    ULONG StoreType;  // 1 = GPT
    GUID  DiskId;     // GPT 磁盘 GUID
} STORE_ID_GPT;

// MBR 模式
typedef struct _STORE_ID_MBR {
    ULONG StoreType;      // 0 = MBR
    ULONG DiskSignature;  // MBR 磁盘签名
} STORE_ID_MBR;

// 联合体
typedef union _STORE_ID {
    ULONG         StoreType;
    STORE_ID_GPT  Gpt;
    STORE_ID_MBR  Mbr;
} STORE_ID;
```

---

## 六、托管封装层 (imagestorageservicemanaged.dll)

### 6.1 ImageStorageManager 类

托管层通过 P/Invoke 调用原生 DLL，提供面向对象的 API：

```csharp
public class ImageStorageManager : IDisposable
{
    private IntPtr _serviceHandle;  // 原生服务句柄

    // VHD 操作
    public SafeFileHandle CreateVirtualHardDisk(string path, ulong size,
        STORE_ID storeId, uint sectorSize);
    public SafeFileHandle OpenVirtualHardDisk(string path, bool readOnly,
        out STORE_ID storeId);
    public void DismountVirtualHardDisk(STORE_ID storeId, bool deleteFile);

    // 存储操作
    public void CreateStore(STORE_ID storeId, PARTITION_ENTRY[] partitions);
    public void CreatePool(STORE_ID storeId, string poolName, Guid poolId);
    public SafeFileHandle CreateSpace(Guid poolId, string name,
        string description, uint capacityGB, out Guid spaceId);

    // 分区操作
    public void PartitionVirtualHardDisk(SafeFileHandle disk, ref STORE_ID storeId,
        PARTITION_ENTRY[] partitions);
    public void ExtendPartition(SafeFileHandle disk, STORE_ID storeId,
        string partitionName, bool extendVolume);

    // 查询
    public string GetPartitionPath(string partitionName);
    public ulong GetPartitionSizeInSectors(string partitionName);
    public ulong GetFreeBytesOnVolume(string partitionName);
    public uint GetClusterSize(string partitionName);
    public void WaitForVolume(string partitionName);
}
```

### 6.2 PARTITION_ENTRY 托管结构

```csharp
[StructLayout(LayoutKind.Explicit, Size = 201)]
public struct PARTITION_ENTRY
{
    [FieldOffset(0)]   public char[] Name;           // 72字节 (36 UTF-16)
    [FieldOffset(72)]  public ulong SectorCount;      // 分区大小(扇区)
    [FieldOffset(80)]  public uint AlignmentSize;      // 对齐
    [FieldOffset(84)]  public uint ClusterSize;        // 簇大小
    [FieldOffset(88)]  public char[] FileSystem;       // 64字节
    [FieldOffset(152)] public Guid Id;                 // 分区唯一 GUID
    [FieldOffset(168)] public Guid Type;               // 分区类型 GUID
    [FieldOffset(184)] public ulong Flags;             // GPT 属性
    [FieldOffset(192)] public ulong OffsetInSectors;   // 偏移
    [FieldOffset(200)] public byte FvePrep;            // BitLocker 准备
}
```

---

## 七、重建项目要点

### 7.1 必须保留的依赖

| 组件 | 原因 |
|------|------|
| **imagestorageservice.dll (32位)** | 81个导出函数，VDS COM + virtio + DeviceIoControl 封装 |
| **virtdisk.dll** | VHD API (OpenVirtualDisk/CreateVirtualDisk/AttachVirtualDisk) |
| **vds_ps.dll** | VDS COM 代理存根 (分区操作) |
| **rs5_spaceutil.exe** | 存储池命令行工具 (根据目标Build选择版本) |

### 7.2 可重写的部分

| 组件 | 难度 | 说明 |
|------|------|------|
| imagestorageservicemanaged.dll | 中 | P/Invoke 封装，已完整反编译 |
| ImageStorageManager 类 | 低 | 面向对象封装，逻辑清晰 |
| PARTITION_ENTRY/STORE_ID 结构 | 低 | 精确布局，已完整分析 |

### 7.3 关键技术约束

1. **32位进程** — imagestorageservice.dll 是32位，必须在32位进程中调用
2. **管理员权限** — VHD 创建/附加、存储池操作都需要管理员权限
3. **VDS 服务** — 分区操作依赖 Virtual Disk Service 服务运行
4. **全局互斥锁** — imaging.dll 使用 `Global\VHDMutex_{585b0806-...}` 防止并发
5. **SeRestorePrivilege** — 必须启用此权限才能正确设置文件权限
6. **VHDX vs VHD** — 根据文件扩展名自动选择，VHDX 使用 DeviceType=3
7. **存储池分区 GUID** — `{5708A6E0-9001-4b99-b064-1fe564896bdb}` 是固定的

### 7.4 调用顺序约束

```
正确顺序:
  CreateImageStorageService()
    → CreateVirtualHardDisk()
      → PartitionVirtualHardDisk()  [普通磁盘]
      → CreateStoragePool() + CreateStorageSpace() + PartitionVirtualHardDisk()  [存储池]
        → FormatPartition()
        → AssignMountPoints()
        → ... 使用镜像 ...
        → DismountVirtualHardDisk()
  CloseImageStorageService()
```

---

*报告基于 imagestorageservice.dll (280KB, 81个导出) PE分析 + IDA反编译生成*
*核心技术: VDS COM API + virtio API (virtdisk.dll) + DeviceIoControl*
*托管封装: imagestorageservicemanaged.dll (ImageStorageManager 类)*
