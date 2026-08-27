# ADK 10.0.17704.1000 — FFU 镜像格式深度研究

> **研究对象**: Full Flash Update (FFU) 镜像格式 v2.0
> **源码路径**: `E:\WSK_Tools\ADK_Research\outputsrc\dotnet\imagecommon.cs` (474KB)
> **相关文件**: `E:\WSK_Tools\ADK_Research\outputsrc\dotnet\ffucomponents.cs` (176KB, 设备刷写管理)
> **命名空间**: `Microsoft.Windows.Image.Common`
> **格式版本**: 2.0
> **默认分区对齐**: 65536 字节 (64KB)

---

## 1. 概述

FFU (Full Flash Update) 是微软为 Windows Mobile / OneCore / IoT Core 设备设计的**整盘镜像格式**。与 WIM（文件级镜像）不同，FFU 是**块级镜像**——它直接描述磁盘上的分区布局和每个分区的原始块数据，支持通过 USB/SD 卡快速刷写到裸设备。

FFU 的核心设计目标：
- **原子性刷写**: 整个镜像一次写入，不依赖目标设备上的操作系统
- **完整性验证**: 三层 hash 链（Catalog → HashTable → Payload chunks）确保障数据未被篡改
- **安全启动**: 签名的 SecurityHeader 确保镜像来自可信来源
- **跨平台**: 支持 MBR 和 GPT 分区表，支持普通磁盘和 Storage Spaces

---

## 2. FFU 文件物理布局

### 2.1 整体结构

```
偏移 0
┌─────────────────────────────────────────────────────────────┐
│  "SignedImage "  (12字节 ASCII 签名)                         │
├─────────────────────────────────────────────────────────────┤
│  SecurityHeader  (20字节)                                    │
│    ByteCount, ChunkSize, HashAlgorithmID,                    │
│    CatalogSize, HashTableSize                                 │
├─────────────────────────────────────────────────────────────┤
│  Catalog  (安全目录, PKCS#7 签名)                            │
│    大小 = SecurityHeader.CatalogSize                          │
├─────────────────────────────────────────────────────────────┤
│  HashTable  (SHA256 hash 数组)                               │
│    大小 = SecurityHeader.HashTableSize                        │
│    每个条目 = 32字节 (SHA256)                                 │
│    条目数 = HashTableSize / 32                                │
├─────────────────────────────────────────────────────────────┤
│  Security Padding  (对齐到 ChunkSize × 1024)                │
│    大小 = CalculateAlignment(HeaderSize+Catalog+HashTable,   │
│                                ChunkSize×1024)                │
├─────────────────────────────────────────────────────────────┤
│  "ImageFlash  "  (12字节 ASCII 签名)                         │
├─────────────────────────────────────────────────────────────┤
│  ImageHeader  (12字节)                                        │
│    ByteCount, ManifestLength, ChunkSize                       │
├─────────────────────────────────────────────────────────────┤
│  Manifest  (INI 格式文本, 描述分区/存储池/设备信息)           │
│    大小 = ImageHeader.ManifestLength                          │
├─────────────────────────────────────────────────────────────┤
│  Image Padding  (对齐到 ChunkSize × 1024)                   │
├─────────────────────────────────────────────────────────────┤
│  Payload  (原始块数据, 按 ChunkSize 分块)                    │
│    每个 chunk 对应 HashTable 中的一个 SHA256                  │
│    chunk 大小 = ChunkSize × 1024 字节                        │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 关键偏移计算

```csharp
// ImageHeader 的起始偏移
public uint StartOfImageHeader
{
    get
    {
        uint offset = 0;
        if (Manifest != null)  // 有安全头时
        {
            offset += FullFlashUpdateHeaders.SecurityHeaderSize;  // 12 + 20 = 32
            offset += _securityHeader.CatalogSize;
            offset += _securityHeader.HashTableSize;
        }
        return offset;  // 加上 SecurityPadding 后才是实际位置
    }
}
```

---

## 3. 核心数据结构

### 3.1 SecurityHeader（安全头）

```csharp
public struct SecurityHeader
{
    public uint ByteCount { get; set; }        // 整个安全区域的字节数（含签名+头+目录+哈希表+填充）
    public uint ChunkSize { get; set; }        // payload 分块大小（单位：KB）
    public uint HashAlgorithmID { get; set; }  // 哈希算法 ID（3=SHA256）
    public uint CatalogSize { get; set; }      // 安全目录（PKCS#7）大小
    public uint HashTableSize { get; set; }    // 哈希表大小（字节）

    public static bool ValidateSignature(byte[] signature)
    {
        // 验证前12字节是否为 "SignedImage "
        return signature.SequenceEqual(Encoding.ASCII.GetBytes("SignedImage "));
    }
}
```

**字段说明**:
- `ByteCount`: 从文件开头到 ImageHeader 之前的总字节数，用于快速定位 ImageHeader
- `ChunkSize`: payload 分块大小，单位 KB。实际 chunk 字节数 = `ChunkSize × 1024`
- `HashAlgorithmID`: 与 wcp.dll 中 `RtlGetHashAlgorithmHashLength` 的 ID 映射一致：
  - 1 = 8字节（自定义）
  - 2 = SHA1（20字节）
  - **3 = SHA256（32字节）★ FFU 默认**
  - 4 = SHA384（48字节）
  - 5 = SHA512（64字节）
  - 6-8 = MD5（16字节）
- `CatalogSize`: PKCS#7 签名的安全目录大小，包含证书链和签名
- `HashTableSize`: 哈希表总字节数 = `chunk数量 × hash长度`

### 3.2 ImageHeader（镜像头）

```csharp
public struct ImageHeader
{
    public uint ByteCount { get; set; }        // 整个镜像区域字节数（含签名+头+清单+填充+payload）
    public uint ManifestLength { get; set; }   // Manifest 文本长度
    public uint ChunkSize { get; set; }        // 与 SecurityHeader.ChunkSize 一致

    public static bool ValidateSignature(byte[] signature)
    {
        // 验证前12字节是否为 "ImageFlash  "
        return signature.SequenceEqual(Encoding.ASCII.GetBytes("ImageFlash  "));
    }
}
```

### 3.3 FullFlashUpdateHeaders（常量辅助类）

```csharp
public static class FullFlashUpdateHeaders
{
    // SecurityHeader 总大小 = 结构体(20B) + 签名(12B) = 32B
    public static uint SecurityHeaderSize =>
        (uint)(Marshal.SizeOf(default(SecurityHeader)) + GetSecurityHeaderSignature().Length);

    // ImageHeader 总大小 = 结构体(12B) + 签名(12B) = 24B
    public static uint ImageHeaderSize =>
        (uint)(FullFlashUpdateImage.ImageHeaderSize + GetImageHeaderSignature().Length);

    public static byte[] GetSecurityHeaderSignature() => Encoding.ASCII.GetBytes("SignedImage ");
    public static byte[] GetImageHeaderSignature()    => Encoding.ASCII.GetBytes("ImageFlash  ");
}
```

**签名字符串注意**:
- `"SignedImage "` — 12字节，末尾有一个空格
- `"ImageFlash  "` — 12字节，末尾有两个空格

---

## 4. 三层 Hash 链

### 4.1 链结构

```
Catalog (PKCS#7 签名)
  │  包含: SecurityHeader 的 SHA256 + 证书链 + 签名
  │
  └─> SecurityHeader
       │  包含: HashTableSize, CatalogSize, HashAlgorithmID
       │
       └─> HashTable (SHA256 数组)
            │  每个条目 = 对应 payload chunk 的 SHA256
            │  条目数 = HashTableSize / 32
            │
            └─> Payload Chunks
                 每个 chunk = ChunkSize × 1024 字节
                 最后一个 chunk 可能不足，用 0 填充
```

### 4.2 验证流程（设备端刷写时）

1. **验证 Catalog 签名**: 用设备中的 Microsoft 根证书验证 PKCS#7 签名
   - 失败 → `FFUFlashException.ErrorCode.SignatureCheckFailed = 24`
2. **验证 SecurityHeader hash**: Catalog 中包含 SecurityHeader 的 hash，比对实际读取的 SecurityHeader
3. **验证 HashTable**: SecurityHeader 描述了 HashTable 的大小和算法
4. **逐 chunk 验证 payload**: 读取每个 chunk，计算 SHA256，与 HashTable 中对应条目比对
   - 失败 → `FFUFlashException.ErrorCode.HashCheckFailed = 23`

### 4.3 HashTable 条目计算

```csharp
// 每个 chunk 的 hash 长度（SHA256 = 32字节）
int hashLength = RtlGetHashAlgorithmHashLength(HashAlgorithmID);  // 3=32

// chunk 数量
int chunkCount = HashTableSize / hashLength;

// 每个 chunk 的字节数
int chunkBytes = ChunkSize * 1024;

// payload 总大小（理论值）
long payloadSize = (long)chunkCount * chunkBytes;
```

---

## 5. Manifest 格式（INI 风格）

### 5.1 概述

FFU 的 Manifest 不是 XML，而是 **INI 风格的纯文本**，使用 `[Category]` 分段和 `key = value` 键值对。这是为了在最小化的 bootloader 环境中也能解析。

### 5.2 ManifestCategory 类

```csharp
public class ManifestCategory
{
    private Hashtable _keyValues = new Hashtable();
    private int _maxKeySize;  // 用于对齐输出

    public string Category { get; set; }  // [Category] 名称
    public string Name { get; private set; }

    public string this[string name]
    {
        get => (string)_keyValues[name];
        set { /* 添加或更新，跟踪最大 key 长度 */ }
    }

    internal void WriteToStream(Stream targetStream)
    {
        // 输出格式:
        // [Category]\r\n
        // key        = value\r\n    (key 右对齐到 _maxKeySize+1)
        // \r\n
        ASCIIEncoding enc = new ASCIIEncoding();
        targetStream.Write(enc.GetBytes("[" + Category + "]\r\n"), ...);
        foreach (DictionaryEntry kv in _keyValues)
        {
            targetStream.Write(enc.GetBytes(kv.Key), ...);
            // 填充空格使所有 = 对齐
            for (int i = 0; i < _maxKeySize + 1 - ((string)kv.Key).Length; i++)
                targetStream.Write(enc.GetBytes(" "), ...);
            targetStream.Write(enc.GetBytes("= " + kv.Value + "\r\n"), ...);
        }
        targetStream.Write(enc.GetBytes("\r\n"), ...);
    }
}
```

### 5.3 Manifest 类别列表

| 类别 | 内容 |
|------|------|
| `[FullFlash]` | 镜像元数据：Version, OSVersion, Description, UEFI, DevicePlatformId0..N, AntiTheftVersion, CanFlashToRemovableMedia, StateSeparationLevel |
| `[StoragePool]` | 存储池信息：Name |
| `[Store]` | 存储信息：StoreId, StoreType, IsSpace, IsMainOSStore, SectorSize, MinSectorCount, DevicePath, OnlyAllocateDefinedGptEntries, Provisioning |
| `[Partition]` | 分区信息：Name, Type, Id, TotalSectors, UsedSectors, UseAllSpace, FileSystem, Bootable, ReadOnly, Hidden, AttachDriveLetter, ServicePartition, Primary, RequiredToFlash, ByteAlignment, ClusterSize, SectorAlignment, PrepareFveMetadata |

### 5.4 [FullFlash] 类别关键字段

```ini
[FullFlash]
Version                  = 2.0
OSVersion                = 10.0.17704.1000
Description              = Windows 10 IoT Core Image
UEFI                     = True
DevicePlatformId0        = {DEVICE-PLATFORM-GUID}
DevicePlatformId1        = {SECOND-PLATFORM-GUID}
AntiTheftVersion         = 1.0
CanFlashToRemovableMedia = False
StateSeparationLevel     = 0
```

**字段说明**:
- `DevicePlatformId0..N`: 允许刷写此镜像的设备平台 ID 列表。设备 bootloader 会检查自身平台 ID 是否在此列表中
  - 不匹配 → `FFUFlashException.ErrorCode.InvalidPlatformId = 22`
- `UEFI`: 是否为 UEFI 系统（True=GPT, False=MBR）
- `AntiTheftVersion`: 防盗版本，用于重置保护
- `CanFlashToRemovableMedia`: 是否允许刷写到可移动介质（SD卡/USB）
  - 不允许时设备检测到可移动介质 → `RemoveableMediaCheckFailed = 32`
- `StateSeparationLevel`: 状态分离级别（用于 Factory OS 的数据/系统分离）

---

## 6. 存储层次结构

### 6.1 三层模型

```
FullFlashUpdateImage (整个镜像)
  └─ List<FullFlashUpdateStoragePool> (存储池列表, 可选)
       └─ List<FullFlashUpdateStore> (存储列表)
            └─ List<FullFlashUpdatePartition> (分区列表)
```

### 6.2 FullFlashUpdateImage（镜像根）

```csharp
public class FullFlashUpdateImage
{
    private const string _version = "2.0";
    private const uint _DefaultPartitionByteAlignment = 65536;  // 64KB

    private ImageHeader _imageHeader;
    private SecurityHeader _securityHeader;
    private List<FullFlashUpdateStoragePool> _ffuStoragePools;
    private List<FullFlashUpdateStore> _ffuStores;
    private FullFlashUpdateManifest Manifest;
    private long _payloadOffset;

    public byte[] CatalogData { get; set; }     // PKCS#7 签名数据
    public byte[] HashTableData { get; set; }   // SHA256 数组

    public uint ChunkSize { get => _imageHeader.ChunkSize; set => ...; }
    public uint ChunkSizeInBytes => ChunkSize * 1024;
    public uint HashAlgorithmID { get => _securityHeader.HashAlgorithmID; set => ...; }
    public uint ManifestLength { get => _imageHeader.ManifestLength; set => ...; }

    public uint ImageStyle  // 0=MBR, 1=GPT (根据第一个分区的 PartitionType 判断)
    public uint StartOfImageHeader  // 计算偏移
    public uint SecurityPadding     // 安全区域填充大小
    public uint DefaultPartitionAlignmentInBytes  // 64KB, 必须是2的幂

    // 便捷属性
    public string Description { get; set; }
    public List<string> DevicePlatformIDs { get; set; }
    public string Version { get; }
    public string OSVersion { get; set; }
    public bool UEFI { get; set; }
    public uint StateSeparationLevel { get; set; }
    public string AntiTheftVersion { get; set; }
    public string CanFlashToRemovableMedia { get; set; }
}
```

### 6.3 FullFlashUpdateStoragePool（存储池）

```csharp
public class FullFlashUpdateStoragePool
{
    public FullFlashUpdateImage Image { get; private set; }
    public string Name { get; private set; }
    public List<FullFlashUpdateStore> Stores { get; }
    public int StoreCount => Stores.Count;

    public void Fixup(IULogger logger)     // 修正所有子 store
    public void Validate(uint blockSize)    // 验证所有子 store
    public void LogInfo(IULogger logger)    // 日志输出
    internal void ToCategory(ManifestCategory category)  // 序列化为 Manifest
}
```

存储池用于 Storage Spaces 模式——多个物理磁盘组合成一个逻辑存储池，从中创建虚拟磁盘（Store）。

### 6.4 FullFlashUpdateStore（存储/虚拟磁盘）

```csharp
public class FullFlashUpdateStore : IDisposable
{
    public FullFlashUpdateImage Image { get; private set; }
    public FullFlashUpdateStoragePool StoragePool { get; private set; }

    public string Id { get; private set; }              // 存储 ID
    public string Type { get; private set; }            // 存储类型
    public bool IsMainOSStore { get; private set; }     // 是否包含 MainOS
    public bool IsSpace { get; private set; }           // 是否为 Storage Space
    public string Provisioning { get; private set; }    // 配置类型 (Thin/Fixed)
    public string DevicePath { get; private set; }      // 设备路径
    public bool OnlyAllocateDefinedGptEntries { get; private set; }

    public uint MinSectorCount { get; set; }   // 最小扇区数（镜像总大小）
    public uint SectorSize { get; set; }        // 扇区大小（通常512）
    public List<FullFlashUpdatePartition> Partitions { get; }
    public int PartitionCount => Partitions.Count;

    public string BackingFile => _tempBackingStoreFile;  // 临时 VHD  backing 文件

    public void Fixup(IULogger logger)     // 修正分区大小
    public void Validate(uint blockSize)    // 验证：扇区大小≤块大小, 块大小是扇区大小倍数, MinSectorCount>0
    public void LogInfo(IULogger logger)
    internal void AddPartition(FullFlashUpdatePartition partition)  // 添加分区（含大小检查）
    internal void ToCategory(ManifestCategory category)
}
```

**验证规则**:
1. `SectorSize <= blockSize`（扇区大小不能超过块大小）
2. `blockSize % SectorSize == 0`（块大小必须是扇区大小的整数倍）
3. `MinSectorCount > 0`（必须指定镜像大小）
4. `MinSectorCount * SectorSize % blockSize == 0`（镜像大小必须是块大小的整数倍）
5. 所有分区的总扇区数 ≤ MinSectorCount（分区不能超出镜像大小）
6. 同一 Store 中最多一个分区设置 `UseAllSpace`

### 6.5 FullFlashUpdatePartition（分区）

```csharp
public class FullFlashUpdatePartition
{
    public FullFlashUpdateStore Store { get; private set; }

    public string Name { get; set; }              // 分区名称 (如 "MainOS", "EFIESP", "Data")
    public uint TotalSectors { get; set; }        // 总扇区数
    public uint SectorsInUse { get; set; }        // 已用扇区数
    public uint LastUsedSector => SectorsInUse - 1;  // 最后一个已用扇区

    public string PartitionType { get; set; }     // 分区类型 GUID (GPT) 或 类型码 (MBR)
    public string PartitionId { get; set; }       // 分区唯一 ID (GUID)

    public bool Bootable { get; set; }
    public bool ReadOnly { get; set; }
    public bool Hidden { get; set; }
    public bool AttachDriveLetter { get; set; }
    public bool ServicePartition { get; set; }
    public bool RequiredToFlash { get; set; }
    public bool UseAllSpace { get; set; }         // 使用剩余所有空间
    public bool PrepareFveMetadata { get; set; }  // 预配 BitLocker 元数据

    public string PrimaryPartition { get; set; }   // 主分区关联
    public string FileSystem { get; set; }         // 文件系统 (NTFS/FAT32/exFAT)

    public uint ByteAlignment { get; set; }        // 字节对齐
    public uint ClusterSize { get; set; }          // 簇大小
    public uint SectorAlignment { get; set; }      // 扇区对齐
    public ulong OffsetInSectors { get; set; }     // 在 Store 中的偏移（扇区）

    public bool IsStoragePool  // PartitionType == STORAGE_POOL_PARTITION_GUID
    {
        get => Guid.TryParse(PartitionType, out var g) &&
               g == ImageConstants.STORAGE_POOL_PARTITION_GUID;
    }
}
```

**关键常量**:
- `STORAGE_POOL_PARTITION_GUID = {5708A6E0-9001-4b99-b064-1fe564896bdb}`（与 imagestorageservice 研究一致）

---

## 7. Security Padding（安全区域填充）

### 7.1 计算逻辑

```csharp
internal uint SecurityPadding
{
    get
    {
        uint blockSize = 1024;  // 基础 1KB
        if (_imageHeader.ChunkSize != 0)
            blockSize *= _imageHeader.ChunkSize;   // 使用 ImageHeader 的 ChunkSize
        else if (_securityHeader.ChunkSize != 0)
            blockSize *= _securityHeader.ChunkSize; // 回退到 SecurityHeader 的
        else
            throw new ImageCommonException("Neither headers have chunk size.");

        // 当前位置 = SecurityHeader签名(12) + SecurityHeader(20) + Catalog + HashTable
        uint currentPosition = (uint)FullFlashUpdateHeaders.SecurityHeaderSize
                             + (uint)(CatalogData?.Length ?? 0)
                             + (uint)(HashTableData?.Length ?? 0);

        // 填充到下一个 blockSize 边界
        return CalculateAlignment(currentPosition, blockSize);
    }
}

private uint CalculateAlignment(uint currentPosition, uint blockSize)
{
    uint remainder = currentPosition % blockSize;
    return (remainder == 0) ? 0 : blockSize - remainder;
}
```

### 7.2 填充的目的

- 确保 ImageHeader 和后续的 Manifest/Payload 都从 `ChunkSize × 1024` 字节边界开始
- 这使得设备端 bootloader 可以直接按块读取，无需处理跨块的头部数据
- 与 HashTable 的 chunk 划分对齐——第一个 payload chunk 从块边界开始

---

## 8. FFUFlashException 错误码

```csharp
public class FFUFlashException : FFUException
{
    public enum ErrorCode
    {
        None = 0,
        FlashError = 2,
        InvalidStoreHeader = 8,
        DescriptorAllocationFailed = 9,
        DescriptorReadFailed = 11,
        BlockReadFailed = 12,
        BlockWriteFailed = 13,
        CrcError = 14,
        SecureHeaderReadFailed = 15,
        InvalidSecureHeader = 16,
        InsufficientSecurityPadding = 17,
        InvalidImageHeader = 18,
        InsufficientImagePadding = 19,
        BufferingFailed = 20,
        ExcessBlocks = 21,
        InvalidPlatformId = 22,
        HashCheckFailed = 23,           // ★ payload chunk hash 不匹配
        SignatureCheckFailed = 24,       // ★ Catalog 签名验证失败
        DesyncFailed = 26,
        FailedBcdQuery = 27,
        InvalidWriteDescriptors = 28,
        AntiTheftCheckFailed = 29,
        RemoveableMediaCheckFailed = 32,
        UseOptimizedSettingsFailed = 33
    }
}
```

---

## 9. FFU 生成流程（imageapp 内部）

### 9.1 高层流程

```
imageapp ProcessImage
  ├─ 创建 VHD 并分区 (imagestorageservice)
  ├─ 应用 CBS 包到 VHD (UpdateDLL: PrepareUpdate + ExecuteUpdate)
  ├─ 配置 BCD / 注册表
  ├─ FinalizeImage
  │   └─ 生成 FFU:
  │       1. 读取 VHD 中每个分区的原始数据
  │       2. 按 ChunkSize 分块，计算每个 chunk 的 SHA256
  │       3. 构建 HashTable（所有 chunk hash 的数组）
  │       4. 构建 SecurityHeader（描述 Catalog/HashTable 大小）
  │       5. 用 Microsoft 证书签名 SecurityHeader → Catalog (PKCS#7)
  │       6. 构建 Manifest（INI 文本，描述分区布局）
  │       7. 构建 ImageHeader
  │       8. 按物理布局写入 FFU 文件
  │       └─ [SignedImage ][SecurityHeader][Catalog][HashTable][Padding]
  │          [ImageFlash  ][ImageHeader][Manifest][Padding][Payload...]
```

### 9.2 签名过程

FFU 的签名使用 `ImageSigner` 类（在 imagecommon.cs 中），支持 4 种证书：

| 证书类型 | 用途 | 指纹 |
|----------|------|------|
| ProdCertRoot | 生产镜像 | `3B1EFD3A66EA28B16697394703A72CA340A05BD5` |
| TestCertRoot | 测试镜像 | `8A334AA8052DD244A647306A76B8178FA215F344` |
| FlightCertPCA | Flight 签名 | `9E594333273339A97051B0F82E86F266B917EDB3` |
| FlightCertWindows | Windows Flight | `5f444a6740b7ca2434c7a5925222c2339ee0f1b7` |

签名工具: `imagesigner.exe`（ADK 工具）

---

## 10. 重建要点

### 10.1 必须保留的机制

1. **双 Header 结构**: SecurityHeader + ImageHeader，各带 12 字节 ASCII 签名
2. **三层 Hash 链**: Catalog → SecurityHeader → HashTable → Payload chunks，不可跳过任何一层
3. **Chunk 对齐**: 所有头部区域必须填充到 `ChunkSize × 1024` 字节边界
4. **INI Manifest**: 不是 XML，是 `[Category]` + `key = value` 的纯文本格式
5. **平台 ID 检查**: DevicePlatformId 列表必须包含目标设备的平台 ID
6. **扇区/块大小约束**: 块大小必须是扇区大小的整数倍，镜像大小必须是块大小的整数倍
7. **UseAllSpace 互斥**: 同一 Store 中最多一个分区使用 UseAllSpace

### 10.2 可简化的部分

1. **Storage Spaces 支持**: 如果目标设备不使用存储空间，可以跳过 StoragePool 层
2. **AntiTheft**: 防盗检查对普通镜像构建非必需
3. **可移动介质检查**: CanFlashToRemovableMedia 可默认设为 True
4. **多平台 ID**: 通常只需要一个 DevicePlatformId0

### 10.3 关键依赖

| 依赖 | 用途 | 可替代方案 |
|------|------|-----------|
| imagesigner.exe | FFU 签名 | 自己实现 PKCS#7 签名（使用 BouncyCastle 或 .NET SignedCms） |
| SHA256 | Hash 计算 | .NET 内置 SHA256Managed |
| 证书存储 | 微软签名证书 | 测试证书可自签名，生产需要微软交叉签名 |

### 10.4 最小 FFU 生成器伪代码

```csharp
// 1. 准备参数
uint chunkSize = 2;  // 2MB chunks (2 * 1024 = 2048KB? 不, ChunkSize单位是KB)
uint chunkBytes = chunkSize * 1024;  // 实际: ChunkSize=2 → 2048字节? 
// 注意: 从代码看 ChunkSizeInBytes = ChunkSize * 1024
// 所以 ChunkSize=2 → 2048字节 (2KB), 典型值可能是 2048 (2MB)

// 2. 读取分区数据并分块
List<byte[]> chunks = new List<byte[]>();
foreach (var partition in partitions)
{
    byte[] data = ReadPartitionRawData(partition);
    for (int i = 0; i < data.Length; i += (int)chunkBytes)
    {
        byte[] chunk = new byte[chunkBytes];
        Array.Copy(data, i, chunk, 0, Math.Min(chunkBytes, data.Length - i));
        chunks.Add(chunk);
    }
}

// 3. 构建 HashTable
byte[] hashTable = new byte[chunks.Count * 32];  // SHA256 = 32字节
for (int i = 0; i < chunks.Count; i++)
{
    byte[] hash = SHA256.Create().ComputeHash(chunks[i]);
    Array.Copy(hash, 0, hashTable, i * 32, 32);
}

// 4. 构建 SecurityHeader
SecurityHeader secHeader = new SecurityHeader
{
    ChunkSize = chunkSize,
    HashAlgorithmID = 3,  // SHA256
    HashTableSize = (uint)hashTable.Length,
    CatalogSize = (uint)catalogData.Length,
    // ByteCount 在签名后计算
};

// 5. 签名 SecurityHeader → Catalog (PKCS#7)
byte[] catalogData = SignSecurityHeader(secHeader, certificate);

// 6. 构建 Manifest (INI)
StringBuilder manifest = new StringBuilder();
manifest.AppendLine("[FullFlash]");
manifest.AppendLine("Version = 2.0");
manifest.AppendLine("OSVersion = 10.0.17704.1000");
manifest.AppendLine("UEFI = True");
manifest.AppendLine($"DevicePlatformId0 = {{{platformId}}}");
manifest.AppendLine();
// ... Store, Partition 类别

// 7. 构建 ImageHeader
ImageHeader imgHeader = new ImageHeader
{
    ChunkSize = chunkSize,
    ManifestLength = (uint)Encoding.ASCII.GetByteCount(manifest.ToString()),
    // ByteCount 在最后计算
};

// 8. 计算填充并写入文件
using (BinaryWriter writer = new BinaryWriter(File.Create(outputPath)))
{
    // Security 区域
    writer.Write(Encoding.ASCII.GetBytes("SignedImage "));  // 12B
    writer.WriteStruct(secHeader);                            // 20B
    writer.Write(catalogData);                                 // CatalogSize
    writer.Write(hashTable);                                   // HashTableSize
    writer.Write(new byte[securityPadding]);                   // 填充

    // Image 区域
    writer.Write(Encoding.ASCII.GetBytes("ImageFlash  "));   // 12B
    writer.WriteStruct(imgHeader);                             // 12B
    writer.Write(Encoding.ASCII.GetBytes(manifest.ToString())); // ManifestLength
    writer.Write(new byte[imagePadding]);                      // 填充

    // Payload
    foreach (var chunk in chunks)
        writer.Write(chunk);
}
```

---

## 11. 源码行号索引

| 功能 | imagecommon.cs 行号 |
|------|---------------------|
| FullFlashUpdateHeaders 类 | 6953 |
| SecurityHeader 结构体 | 6969 |
| ImageHeader 结构体 | 6994 |
| ManifestCategory 类 | 7007 |
| FullFlashUpdateImage 类 | 7171 |
| FullFlashUpdateImage.StartOfImageHeader | 7306 |
| FullFlashUpdateImage.SecurityPadding | 7338 |
| FullFlashUpdateImage.DevicePlatformIDs | 7382 |
| FullFlashUpdateStoragePool 类 | 7994 |
| FullFlashUpdateStore 类 | 8076 |
| FullFlashUpdateStore.Validate | 8231 |
| FullFlashUpdateStore.AddPartition | 8284 |
| FullFlashUpdatePartition 类 | 8387 |
| FullFlashUpdatePartition.IsStoragePool | 8441 |
| FFUFlashException.ErrorCode 枚举 | ffucomponents.cs 833 |

---

*文档生成时间: 2026-08-27*
*研究工具: ILSpy 反编译 .NET 程序集 + VS Code 源码分析*
*ADK 版本: 10.0.17704.1000 (Windows 10 RS5)*
