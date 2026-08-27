# ADK 10.0.17704.1000 — 输入配置格式深度研究（DeviceLayout / OEMInput / CompDB / FeatureManifest）

> **研究对象**: ADK 镜像构建的所有输入配置 XML 格式
> **源码路径**:
> - `E:\WSK_Tools\ADK_Research\outputsrc\dotnet\imagecommon.cs` (474KB)
> - `E:\WSK_Tools\ADK_Research\outputsrc\dotnet\featureapi.cs`
> - `E:\WSK_Tools\ADK_Research\outputsrc\dotnet\DeviceLayoutValidation.cs` (89KB)
> **命名空间**: `Microsoft.Windows.Image.Common` / `Microsoft.WindowsPhone.CompDB`
> **公共 XML 命名空间**: `http://schemas.microsoft.com/embedded/2004/10/ImageUpdate`

---

## 1. 概述

ADK 镜像构建使用**四层输入配置**，从高层到低层依次为：

```
OEMInput.xml          ← 产品级配置（选哪些 Feature、语言、分辨率）
  └─ FeatureManifest (*.fm.xml)  ← Feature 定义（每个 Feature 包含哪些包）
       └─ CompDB (*.CompDB.xml)  ← 组件数据库（所有可用包的索引）
            └─ .cab 包            ← 实际 CBS 包

DeviceLayout.xml      ← 磁盘布局配置（分区表、存储池、扇区大小）
  └─ ImageGeneratorParameters     ← 内部表示（InputStore/InputPartition）
```

### 1.1 各格式的角色

| 格式 | 扩展名 | 角色 | 生成者 | 消费者 |
|------|--------|------|--------|--------|
| **OEMInput** | .xml | 产品配置：选择 Feature、语言、分辨率、SOC | OEM/用户 | imageapp / imggen.cmd |
| **FeatureManifest** | .fm.xml | Feature 定义：Feature ID → 包列表映射 | 微软/OEM | BSPCompDB 生成器 |
| **CompDB** | .CompDB.xml | 组件数据库：所有可用包的完整索引 | 构建系统 | imageapp 包解析 |
| **DeviceLayout** | .xml | 磁盘布局：分区表、存储池、扇区大小 | OEM/用户 | imageapp 分区创建 |

---

## 2. DeviceLayout 格式（ImageGeneratorParameters）

### 2.1 概述

DeviceLayout 描述目标设备的**磁盘物理布局**，包括分区表、存储池、扇区大小、FFU 分块大小等。它是 imageapp 创建 VHD/物理磁盘分区的依据。

内部表示为 `ImageGeneratorParameters` 类，通过 XML 反序列化加载。

### 2.2 XML 结构

```xml
<ImageGeneratorParameters xmlns="http://schemas.microsoft.com/embedded/2004/10/ImageUpdate">
  <DevicePlatformIDs>
    <ID>{DEVICE-PLATFORM-GUID}</ID>
  </DevicePlatformIDs>
  <UEFI>True</UEFI>
  <Description>Windows 10 IoT Core Image</Description>
  <SectorSize>512</SectorSize>
  <ChunkSize>256</ChunkSize>
  <MinSectorCount>0</MinSectorCount>
  <DefaultPartitionByteAlignment>65536</DefaultPartitionByteAlignment>

  <Stores>
    <Store>
      <Id>MainOSStore</Id>
      <StoreType>Standard</StoreType>
      <SizeInSectors>0</SizeInSectors>
      <OnlyAllocateDefinedGptEntries>False</OnlyAllocateDefinedGptEntries>
      <Partitions>
        <Partition>
          <Name>EFIESP</Name>
          <Type>{c12a7328-f81f-11d2-ba4b-00a0c93ec93b}</Type>
          <Id>{...}</Id>
          <TotalSectors>102400</TotalSectors>
          <FileSystem>FAT32</FileSystem>
          <Bootable>True</Bootable>
          <ReadOnly>False</ReadOnly>
          <Hidden>False</Hidden>
          <RequiredToFlash>True</RequiredToFlash>
        </Partition>
        <Partition>
          <Name>MainOS</Name>
          <Type>{ebd0a0a2-b9e5-4433-87c0-68b6b72699c7}</Type>
          <UseAllSpace>True</UseAllSpace>
          <FileSystem>NTFS</FileSystem>
          <Bootable>True</Bootable>
        </Partition>
        <Partition>
          <Name>Data</Name>
          <Type>{ebd0a0a2-b9e5-4433-87c0-68b6b72699c7}</Type>
          <MinFreeSectors>8192</MinFreeSectors>
          <FileSystem>NTFS</FileSystem>
        </Partition>
      </Partitions>
    </Store>
  </Stores>

  <StoragePools>
    <StoragePool>
      <Name>PrimaryPool</Name>
      <Stores>...</Stores>
    </StoragePool>
  </StoragePools>
</ImageGeneratorParameters>
```

### 2.3 ImageGeneratorParameters 类

```csharp
public class ImageGeneratorParameters
{
    public const uint DefaultChunkSize = 256;        // 默认 256KB chunks
    public const uint DevicePlatformIDSize = 192;     // 平台 ID 最大长度

    public List<InputStoragePool> StoragePools;        // 存储池列表
    public List<InputStore> Stores;                    // 存储列表（非存储池模式）
    public string[] DevicePlatformIDs;                  // 允许的设备平台 ID
    public bool UEFI;                                   // 是否 UEFI (GPT)
    public string Description;                          // 镜像描述
    public uint ChunkSize { get; set; }                // FFU 分块大小 (KB)
    public uint Algid { get; set; } = 32780;          // 哈希算法 (CALG_SHA_256)
    public uint SectorSize { get; set; }               // 扇区大小 (默认512)
    public uint MinSectorCount { get; set; }           // 最小扇区数（镜像总大小）
    public uint DefaultPartitionByteAlignment { get; set; }  // 默认分区对齐 (64KB)
    public uint StateSeparationLevel { get; set; }     // 状态分离级别
    public uint VirtualHardDiskSectorSize { get; set; } // VHD 扇区大小
    public InputRules Rules { get; set; }               // 验证规则
    public uint DeviceLayoutVersion { get; private set; } = 1;

    public InputStore MainOSStore  // 查找包含 "MainOS" 分区的 Store
    {
        get => Stores.FirstOrDefault(s => s.IsMainOSStore)
            ?? StoragePools.SelectMany(p => p.Stores).FirstOrDefault(s => s.IsMainOSStore);
    }

    public void Validate()
    {
        // 1. ChunkSize × 1024 必须 < 4GB
        if (uint.MaxValue / ChunkSize < 1024) throw ...;
        // 2. SectorSize >= 512 且是 2 的幂
        if (SectorSize < 512 || !IsPowerOfTwo(SectorSize)) throw ...;
        // 3. 验证所有 Store 和 Partition
        foreach (var store in Stores) store.Validate(SectorSize);
    }
}
```

### 2.4 InputStore 类

```csharp
public class InputStore
{
    public InputPartition[] Partitions;
    public string Id { get; set; }              // 存储 ID
    public string StoreType { get; set; }       // 类型: Standard / ...
    public string Provisioning { get; set; }    // 配置: Thin / Fixed
    public string DevicePath { get; set; }      // 设备路径
    public uint SizeInSectors { get; set; }     // 大小（扇区）
    public bool OnlyAllocateDefinedGptEntries { get; set; }  // 仅分配已定义 GPT 条目（≤32分区）

    // 是否包含名为 "MainOS" 的分区
    public bool IsMainOSStore =>
        Partitions.SingleOrDefault(p => p.Name == "MainOS") != null;

    public void Validate(uint sectorSize)
    {
        if (string.IsNullOrEmpty(StoreType)) throw ...;
        // OnlyAllocateDefinedGptEntries 要求 ≤32 分区
        if (OnlyAllocateDefinedGptEntries && Partitions.Count() > 32) throw ...;
        // 非 MainOS Store 必须有 Id 和 SizeInSectors
        if (!IsMainOSStore)
        {
            if (string.IsNullOrEmpty(Id)) throw ...;
            if (SizeInSectors == 0) throw ...;
            // 最小 3MB
            if ((ulong)SizeInSectors * sectorSize < 3145728) throw ...;
        }
        foreach (var p in Partitions) p.Validate();
    }
}
```

### 2.5 InputPartition 类

```csharp
public class InputPartition
{
    private const uint _MinimumSectorFreeCount = 8192;  // MinFreeSectors 最小值

    public string Name { get; set; }              // 分区名 (MainOS, EFIESP, Data, ...)
    public string Type { get; set; }              // 分区类型 GUID (GPT) 或类型码 (MBR)
    public string Id { get; set; }                // 分区唯一 GUID
    public bool ReadOnly { get; set; }
    public bool AttachDriveLetter { get; set; }
    public bool Hidden { get; set; }
    public bool ServicePartition { get; set; }
    public bool Bootable { get; set; }
    public uint TotalSectors { get; set; }        // 总扇区数（固定大小）
    public bool UseAllSpace { get; set; }         // 使用剩余所有空间
    public uint MinFreeSectors { get; set; }      // 最小空闲扇区（动态大小）
    public uint GeneratedFileOverheadSectors { get; set; }  // 生成文件开销扇区
    public string FileSystem { get; set; }        // NTFS / FAT32 / exFAT
    public string UpdateType { get; set; }        // 更新类型
    public bool Compressed { get; set; }
    public bool RequiresCompression { get; set; }
    public string PrimaryPartition { get; set; }   // 主分区关联（默认=Name）
    public bool RequiredToFlash { get; set; }
    public bool SingleSectorAlignment { get; set; }
    public uint ByteAlignment { get; set; }        // 字节对齐
    public uint ClusterSize { get; set; }          // 文件系统簇大小
    public ulong OffsetInSectors { get; set; }     // 在 Store 中的偏移
    public bool PrepareFveMetadata { get; set; }   // 预配 BitLocker 元数据

    public bool IsStoragePool => Type == "StoragePool";

    public void Validate()
    {
        // StoragePool 类型分区有大量限制（不能设置 Id/ReadOnly/Bootable/TotalSectors 等）
        if (IsStoragePool) { /* 检查所有不允许的字段 */ }

        // MinFreeSectors 与 TotalSectors/UseAllSpace 互斥
        if (MinFreeSectors != 0 && (TotalSectors != 0 || UseAllSpace)) throw ...;
        // MinFreeSectors >= 8192
        if (MinFreeSectors != 0 && MinFreeSectors < 8192) throw ...;
        // GeneratedFileOverheadSectors 需要 MinFreeSectors
        if (GeneratedFileOverheadSectors != 0 && MinFreeSectors == 0) throw ...;
    }
}
```

### 2.6 关键分区类型 GUID

| 分区名 | Type GUID | 说明 |
|--------|-----------|------|
| EFIESP | `{c12a7328-f81f-11d2-ba4b-00a0c93ec93b}` | EFI 系统分区 (FAT32) |
| MainOS | `{ebd0a0a2-b9e5-4433-87c0-68b6b72699c7}` | 基本数据分区 (NTFS) |
| Data | `{ebd0a0a2-b9e5-4433-87c0-68b6b72699c7}` | 数据分区 (NTFS) |
| StoragePool | `{5708A6E0-9001-4b99-b064-1fe564896bdb}` | 存储池分区 |
| CRASHDUMP | `{d6c10b80-2000-4a74-8e23-8c61d772a8e7}` | 崩溃转储分区 |

### 2.7 三种分区大小模式

| 模式 | 字段 | 说明 |
|------|------|------|
| **固定大小** | `TotalSectors` | 分区大小固定为指定扇区数 |
| **使用所有空间** | `UseAllSpace=True` | 分区使用 Store 中剩余所有空间（每 Store 最多一个） |
| **动态最小空闲** | `MinFreeSectors` | 分区大小 = 实际文件大小 + 开销 + MinFreeSectors（≥8192扇区） |

---

## 3. OEMInput 格式

### 3.1 概述

OEMInput 是**产品级配置文件**，描述要构建的 Windows 镜像包含哪些 Feature、支持哪些语言/分辨率、使用哪个 SOC/SV 等。它是 imggen.cmd 和 imageapp 的主要输入。

### 3.2 XML 结构

```xml
<OEMInput xmlns="http://schemas.microsoft.com/embedded/2004/10/ImageUpdate">
  <Description>Windows 10 IoT Core Product</Description>
  <SOC>QC8996</SOC>
  <SV>SV.Prod</SV>
  <Device>DeviceName</Device>
  <ReleaseType>Production</ReleaseType>
  <BuildType>fre</BuildType>
  <Product>WindowsCore</Product>

  <SupportedLanguages>
    <UserInterface>
      <Language>en-US</Language>
      <Language>zh-CN</Language>
    </UserInterface>
    <Keyboard>
      <Language>0409:00000409</Language>
    </Keyboard>
    <Speech>
      <Language>en-US</Language>
    </Speech>
  </SupportedLanguages>

  <BootUILanguage>en-US</BootUILanguage>
  <BootLocale>en-US</BootLocale>

  <Resolutions>
    <Resolution>1024x768</Resolution>
    <Resolution>1920x1080</Resolution>
  </Resolutions>

  <AdditionalFMs>
    <AdditionalFM>$(FMDIRECTORY)\MSCORE.feature.xml</AdditionalFM>
    <AdditionalFM>$(FMDIRECTORY)\OEM.feature.xml</AdditionalFM>
  </AdditionalFMs>

  <Features>
    <Microsoft>
      <Feature>IOT_BERTHA</Feature>
      <Feature>IOT_UAP_OOBE</Feature>
      <Feature>IOT_APP_TOOLING</Feature>
    </Microsoft>
    <OEM>
      <Feature>OEM_CUSTOM_FEATURE</Feature>
    </OEM>
  </Features>

  <AppXOptionalPackages>
    <AppXID>AppPackageName_1.0.0.0_arm</AppXID>
  </AppXOptionalPackages>

  <AdditionalSKUs>
    <AdditionalSKU>SKU_Name</AdditionalSKU>
  </AdditionalSKUs>

  <OptionalFeatures>
    <OptionalFeature>NetFx3</OptionalFeature>
  </OptionalFeatures>

  <PackageFiles>
    <PackageFile>C:\path\to\custom.cab</PackageFile>
  </PackageFiles>

  <ExcludePrereleaseFeatures>False</ExcludePrereleaseFeatures>
  <FormatDPP></FormatDPP>
</OEMInput>
```

### 3.3 OEMInput 类关键字段

```csharp
public class OEMInput
{
    public string Description;
    public string SOC;                    // SoC 标识 (如 QC8996, IntelApollo)
    public string SV;                     // 硅供应商版本
    public string Device;                 // 设备名
    public string ReleaseType;            // Production / Test
    public string BuildType;              // fre (release) / chk (checked)
    public string Product;                // 产品名 (默认 WindowsCore)

    public SupportedLangs SupportedLanguages;  // UI/键盘/语音语言
    public string BootUILanguage;              // 启动 UI 语言
    public string BootLocale;                  // 启动区域设置

    public List<string> Resolutions;           // 支持的分辨率列表
    public string MachineInfoFile;             // 机器信息文件
    public List<string> AdditionalFMs;         // 额外 FeatureManifest 文件
    public OEMInputFeatures Features;          // 选中的 Feature (Microsoft + OEM)
    public List<string> AppXOptionalPackages;  // 可选 AppX 包
    public List<string> AdditionalSKUs;        // 额外 SKU
    public List<string> PackageFiles;          // 直接包含的 .cab 包
    public bool ExcludePrereleaseFeatures;     // 排除预发布 Feature
    public UserStoreMapData UserStoreMapData;  // 用户存储映射

    [XmlIgnore] public string CPUType;         // 架构 (x86/amd64/arm/arm64)
    [XmlIgnore] public Edition Edition;         // 版本信息
    [XmlIgnore] public List<string> FeatureIDs; // 所有选中的 Feature ID 列表

    // 变量替换: $(device), $(releasetype), $(buildtype), $(cputype),
    //           $(bootuilanguage), $(bootlocale), $(mspackageroot)
    public string ProcessOEMInputVariables(string value) { ... }
}
```

### 3.4 OEMFeatureTypes 标志枚举

```csharp
[Flags]
public enum OEMFeatureTypes
{
    NONE = 0,
    BASE = 1,           // 基础 Feature
    BOOTUI = 2,         // 启动 UI
    BOOTLOCALE = 4,     // 启动区域
    RELEASE = 8,        // 发布相关
    SV = 0x20,          // 硅供应商
    SOC = 0x40,         // SoC
    DEVICE = 0x80,      // 设备
    KEYBOARD = 0x100,   // 键盘
    SPEECH = 0x200,     // 语音
    MSFEATURES = 0x400, // 微软 Feature
    OEMFEATURES = 0x800,// OEM Feature
    PRERELEASE = 0x1000,// 预发布
    UILANGS = 0x2000,   // UI 语言
    RESOULTIONS = 0x4000// 分辨率
}
```

---

## 4. FeatureManifest 格式

### 4.1 概述

FeatureManifest (FM) 定义**Feature 与包的映射关系**。每个 Feature 是一个逻辑功能单元（如 `IOT_BERTHA`、`IOT_UAP_OOBE`），包含一个或多个 .cab 包。FM 文件由微软和 OEM 分别提供。

### 4.2 XML 结构

```xml
<FeatureManifest xmlns="http://schemas.microsoft.com/embedded/2004/10/ImageUpdate">
  <Identity>
    <Name>MSCORE</Name>
    <Version>10.0.17704.1000</Version>
    <OwnerType>Microsoft</OwnerType>
    <ReleaseType>Production</ReleaseType>
  </Identity>

  <Features>
    <Feature>
      <FeatureID>IOT_BERTHA</FeatureID>
      <FeatureGroup>Base</FeatureGroup>
      <Packages>
        <PackageFile Path="Microsoft-IoTUAP-Bertha-Package~31bf3856ad364e35~arm~~10.0.17704.1000.cab" />
        <PackageFile Path="Microsoft-Windows-InternetExplorer-Optional-Package~...cab" />
      </Packages>
    </Feature>

    <Feature>
      <FeatureID>IOT_UAP_OOBE</FeatureID>
      <FeatureGroup>Apps</FeatureGroup>
      <Packages>
        <PackageFile Path="Microsoft-Windows-COMRuntime-OneCore-Package~...cab" />
      </Packages>
    </Feature>
  </Features>

  <ConditionalFeatures>
    <ConditionalFeature>
      <FeatureID>IOT_SHELL</FeatureID>
      <Conditions>
        <Condition Type="Resolution" Value="1024x768" />
      </Conditions>
      <Packages>...</Packages>
    </ConditionalFeature>
  </ConditionalFeatures>
</FeatureManifest>
```

### 4.3 FeatureManifest 类

```csharp
public class FeatureManifest
{
    public string ID { get; set; }
    public string OSVersion { get; set; }
    public OwnerType OwnerType { get; set; }   // Microsoft / OEM
    public ReleaseType ReleaseType { get; set; } // Production / Test
    public string Owner { get; set; }

    public List<FMFeature> Features;            // Feature 列表
    public List<FMConditionalFeature> ConditionalFeatures;  // 条件 Feature

    public static void ValidateAndLoad(ref FeatureManifest fm, string path, IULogger logger)
    {
        // 1. XSD 验证（嵌入资源中的 FM schema）
        // 2. XML 反序列化
        // 3. 检查 OwnerType / ReleaseType
    }
}
```

---

## 5. CompDB 格式

### 5.1 概述

CompDB (Component Database) 是**所有可用包的完整索引**，由构建系统从所有 FM 文件聚合生成。它包含每个包的完整元数据（名称、版本、架构、公钥令牌、依赖关系、卫星信息）。imageapp 使用 CompDB 来解析 OEMInput 中选中的 Feature 对应的实际 .cab 包。

### 5.2 CompDB 类型

```csharp
public enum CompDBType
{
    Invalid = -1,
    Build,          // 构建 CompDB（所有包）
    Update,         // 更新 CompDB（仅差异包）
    Device,         // 设备 CompDB
    BSP,            // BSP CompDB（OEM 输入生成）
    Baseless,       // 无基础 CompDB
    BuildUpdate,    // 构建+更新
    Supplemental,   // 补充 CompDB
    Standalone      // 独立 CompDB
}
```

### 5.3 BuildCompDB 类

```csharp
[XmlRoot(ElementName = "CompDB", Namespace = "http://schemas.microsoft.com/embedded/2004/10/ImageUpdate")]
public class BuildCompDB
{
    public const string c_BuildCompDBSchemaVersion = "1.2";

    [XmlAttribute] public DateTime CreatedDate;
    [XmlAttribute] public string Revision = "1";
    [XmlAttribute] public string SchemaVersion = "1.2";
    [XmlAttribute] public string Product;
    [XmlAttribute] public Guid BuildID;
    [XmlAttribute] public string BuildInfo;       // %_RELEASELABEL%.%_PARENTBRANCHBUILDNUMBER%...
    [XmlAttribute] public string OSVersion;
    [XmlAttribute] public string BuildArch;       // x86/amd64/arm
    [XmlAttribute] public ReleaseType ReleaseType;

    public List<CompDBFeature> Features;           // Feature 列表
    public List<FMConditionalFeature> MSConditionalFeatures;
    public List<CompDBPackageInfo> Packages;       // 包列表（核心）
    public CDBAddOnAppX AppX;                      // AppX 附加包
    public ChunkTags CompDBTags;

    public void GenerateCompDB(FMCollection fms, string fmDir, string pkgRoot,
                                string buildType, CpuId arch, string buildInfo)
    {
        // 1. 加载所有 FM 文件
        // 2. 聚合所有 Feature 和 Package
        // 3. 解析包路径（$(mspackageroot) 替换）
        // 4. 生成 CompDB XML
    }
}
```

### 5.4 BSPCompDB 类

```csharp
public class BSPCompDB : BuildCompDB
{
    public const string c_BSPCompDBSchemaVersion = "1.3";

    [XmlAttribute] public string BSPVersion;
    [XmlAttribute] public string BSPProductName;

    public List<FMConditionalFeature> OEMConditionalFeatures;

    public void GenerateBSPCompDB(string oemInputXML, string fmDir, string msPkgRoot,
                                    string buildType, CpuId arch, string buildInfo)
    {
        // 1. 加载并验证 OEMInput.xml
        // 2. 加载 OEMInput 中引用的所有 AdditionalFM
        // 3. 生成 BuildCompDB（所有 FM 的聚合）
        // 4. 过滤：仅保留 OEMInput.FeatureIDs 中选中的 Feature
        // 5. 过滤包：仅保留选中 Feature 包含的包
        // 6. BSPVersion = 第一个包的版本
    }
}
```

### 5.5 CompDBPackageInfo 类

```csharp
public class CompDBPackageInfo
{
    [XmlAttribute] public string ID;           // 包标识 (如 Microsoft-Windows-COM-Package)
    [XmlAttribute] public string Version;      // 版本 (10.0.17704.1000)
    [XmlAttribute] public string CPUType;      // 架构
    [XmlAttribute] public string PackagePath;  // .cab 文件路径
    [XmlAttribute] public string PublicKeyToken; // 公钥令牌 (31bf3856ad364e35)

    public CompDBSatelliteInfo SatelliteInfo;  // 卫星信息 (语言/分辨率)
    public List<CompDBPackageDependency> Dependencies;  // 依赖关系

    public string VersionStr => Version;
    public ReleaseType ReleaseType { get; set; }

    public CompDBPackageInfo SetParentDB(BuildCompDB parent) { ... }
}
```

### 5.6 CompDBSatelliteInfo 类

```csharp
public class CompDBSatelliteInfo
{
    public const string c_Arch = "arch";
    public const string c_Langauge = "language";

    public List<CompDBSatelliteInfoElement> RequireInfo;   // 要求的卫星属性
    public List<CompDBSatelliteInfoElement> ApplyToInfo;   // 适用于的卫星属性
    public List<CompDBSatelliteInfoElement> DeclareInfo;   // 声明的卫星属性

    public bool IsEmpty() => RequireInfo?.Count == 0 && ApplyToInfo?.Count == 0 && DeclareInfo?.Count == 0;
}

public class CompDBSatelliteInfoElement
{
    [XmlAttribute] public string Type;   // "arch" / "language" / "resolution"
    [XmlAttribute] public string Value;  // "arm" / "en-US" / "1024x768"
}
```

---

## 6. 格式间的数据流

### 6.1 BSPCompDB 生成流程

```
OEMInput.xml
  ├─ AdditionalFMs[] → 加载每个 .fm.xml
  │    └─ FeatureManifest (FeatureID → PackageFile[])
  ├─ Features.Microsoft[] → 选中的微软 Feature ID
  ├─ Features.OEM[] → 选中的 OEM Feature ID
  ├─ SupportedLanguages → 语言列表
  └─ Resolutions → 分辨率列表

BSPCompDB.GenerateBSPCompDB():
  1. 验证 OEMInput.xml (XSD)
  2. 对每个 AdditionalFM:
     a. 展开 $(FMDIRECTORY) 变量
     b. 加载并验证 FeatureManifest
     c. 如果是 Microsoft FM → 记录 OSVersion
     d. 如果是 OEM FM → 记录 ReleaseType
  3. GenerateCompDB(): 聚合所有 FM 的 Feature 和 Package
  4. 过滤 Features: 仅保留 OEMInput.FeatureIDs 中的 Feature
  5. 过滤 Packages: 仅保留选中 Feature 引用的 Package
  6. BSPVersion = 第一个 Package 的 Version
  7. 输出 BSPCompDB.xml (SchemaVersion=1.3)
```

### 6.2 imageapp 中的包解析

```
imageapp.ProcessImage():
  1. 加载 OEMInput.xml
  2. 加载 BSPCompDB.xml (或从 OEMInput + FM 即时生成)
  3. 对每个选中的 Feature:
     a. 在 CompDB.Features 中查找 FeatureID
     b. 获取 Feature.Packages[] (包 ID 列表)
     c. 在 CompDB.Packages 中查找每个包的详细信息
     d. 解析包路径 (PackagePath)
  4. 按分区/依赖关系排序包
  5. 调用 UpdateDLL.PrepareUpdate + ExecuteUpdate 应用包
```

---

## 7. 验证规则（InputRules）

DeviceLayout 支持声明式验证规则，用于在构建前检查配置约束：

```csharp
public class InputRules
{
    public InputIntegerRule[] IntegerRules;  // 整数规则
    public InputStringRule[] StringRules;    // 字符串规则
}

public abstract class InputRule
{
    public string Property { get; set; }  // 要验证的属性名
    public string Mode { get; set; }       // AFFIRMATIVE / NEGATIVE / OPTIONAL

    public char ModeCharacter
    {
        get
        {
            switch (Mode)
            {
                case "AFFIRMATIVE": return 'A';  // 必须匹配
                case "NEGATIVE":    return 'N';  // 必须不匹配
                case "OPTIONAL":    return 'O';  // 可选
            }
        }
    }
}

public class InputIntegerRule : InputRule
{
    public ulong[] Values;   // 允许的值列表
    public ulong? Max;       // 最大值
    public ulong? Min;       // 最小值
}

public class InputStringRule : InputRule
{
    public string[] Values;  // 允许的值列表
}
```

---

## 8. 重建要点

### 8.1 必须保留的机制

1. **四层配置分离**: OEMInput → FeatureManifest → CompDB → .cab，每层职责清晰
2. **XML 命名空间**: 所有配置使用 `http://schemas.microsoft.com/embedded/2004/10/ImageUpdate`
3. **XSD 验证**: 每种格式都有嵌入的 XSD schema，加载时必须验证
4. **变量替换**: OEMInput 支持 `$(device)`/`$(releasetype)`/`$(buildtype)`/`$(cputype)`/`$(mspackageroot)` 等变量
5. **Feature 过滤**: BSPCompDB 生成时必须按 OEMInput.FeatureIDs 过滤
6. **分区大小三模式**: 固定大小 / UseAllSpace / MinFreeSectors 动态大小
7. **StoragePool 分区限制**: Type=StoragePool 的分区不能设置大多数字段
8. **卫星信息**: CompDB 包必须支持语言/分辨率卫星属性

### 8.2 可简化的部分

1. **CompDB 预生成**: 可以从 OEMInput + FM 即时生成 CompDB，不需要预生成的 .CompDB.xml
2. **InputRules**: 声明式验证规则对简单构建非必需
3. **多种 CompDBType**: 通常只需要 Build 类型
4. **ConditionalFeatures**: 条件 Feature 对基础构建可简化

### 8.3 关键常量

| 常量 | 值 | 说明 |
|------|-----|------|
| DefaultChunkSize | 256 | FFU 默认分块大小 (KB) |
| DefaultPartitionByteAlignment | 65536 | 默认分区对齐 (64KB) |
| MinimumSectorSize | 512 | 最小扇区大小 |
| MinimumSectorFreeCount | 8192 | MinFreeSectors 最小值 |
| MinimumStoreSize | 3145728 | 最小 Store 大小 (3MB) |
| GPTBasicDataPartitionType | `{ebd0a0a2-b9e5-4433-87c0-68b6b72699c7}` | GPT 基本数据分区 |
| CALG_SHA_256 | 32780 | FFU 哈希算法 ID |
| DevicePlatformIDSize | 192 | 平台 ID 最大长度 |

---

## 9. 源码行号索引

| 功能 | 文件 | 行号 |
|------|------|------|
| ImageGeneratorParameters 类 | imagecommon.cs | 9560 |
| InputStore 类 | imagecommon.cs | 9226 |
| InputPartition 类 | imagecommon.cs | 9288 |
| InputStoragePool 类 | imagecommon.cs | 9210 |
| InputRules / InputRule | imagecommon.cs | 9508-9559 |
| BuildCompDB 类 | imagecommon.cs | 354 |
| BSPCompDB 类 | imagecommon.cs | 151 |
| CompDBSatelliteInfo | imagecommon.cs | 50 |
| OEMInput 类 | featureapi.cs | 73 |
| OEMFeatureTypes 枚举 | featureapi.cs | 76 |
| FullFlashUpdateImage | imagecommon.cs | 7171 |

---

*文档生成时间: 2026-08-27*
*研究工具: ILSpy 反编译 .NET 程序集 + VS Code 源码分析*
*ADK 版本: 10.0.17704.1000 (Windows 10 RS5)*
