# ADK 10.0.17704.1000 — AppX 预安装机制深度研究

> **研究对象**: AppX 预安装（离线预配 Universal Windows Platform 应用）
> **源码路径**: `E:\WSK_Tools\ADK_Research\outputsrc\dotnet\appximaging.cs`
> **相关 DLL**: `appxpackaging.dll` (1.4MB, COM API), `MakeAppx.exe` (ADK 工具), `offreg.dll` (离线注册表)
> **命名空间**: `Microsoft.ImagingTools.AppX`
> **程序集版本**: 10.0.10011.16384

---

## 1. 概述

AppX 预安装是 ADK 镜像构建中的**应用预配阶段**，负责在离线镜像中预装 Universal Windows Platform (UWP) 应用。与通过 Microsoft Store 在线安装不同，预安装将应用文件直接复制到镜像的 `WindowsApps` 目录，并在离线注册表 hive 中注册应用信息，使设备首次启动时应用即可使用。

### 1.1 两阶段模型

与 CBS 包的两阶段更新类似，AppX 预安装也分为两个阶段：

| 阶段 | 方法 | 作用 |
|------|------|------|
| **Stage（暂存）** | `StageAppXPackages` | 用 MakeAppx.exe 解压 .appx/.appxbundle 到临时暂存目录 |
| **Commit（提交）** | `CommitAppXPackages` | 复制应用文件到 WindowsApps、复制许可证、写注册表 hive |

### 1.2 在 imageapp 流水线中的位置

```
imageapp ProcessImage
  ├─ StageImage (CBS 包暂存)
  │   └─ UpdateDLL.PrepareUpdate
  ├─ AppX Stage (并行)
  │   └─ AppXImaging.StageAppXPackages
  ├─ CommitImage (CBS 包提交)
  │   └─ UpdateDLL.ExecuteUpdate
  └─ AppX Commit (在 CBS 提交之后)
      └─ AppXImaging.CommitAppXPackages
```

---

## 2. 核心常量与路径

```csharp
public class AppXImaging
{
    public static string c_AppLicensesFolder = "AppData";              // 许可证目录
    public static string c_WindowsAppsFolder = "WindowsApps";           // 应用文件目录
    public static string c_AppxAllUserStoreRegHive = "AppxAllUserStore.dat";  // 注册表 hive
    public static string c_StagingSubFolder = "appx";                   // 暂存子目录
    public static string c_MakeAppxExeName = "MakeAppx.exe";           // 解压工具
    public static string c_BundleTempSubFolder = "Bundle";              // Bundle 临时目录

    private static float c_CommitSizeEstimateMultiplier = 1.1f;        // 提交大小估算倍率（+10%）
}
```

### 2.1 目标路径（相对于预安装卷根目录）

| 路径 | 用途 |
|------|------|
| `\WindowsApps\<PackageFullName>\` | 应用文件（解压后的 .appx 内容） |
| `\AppData\<PackageFamilyName>.xml` | 应用许可证文件 |
| `\WindowsApps\AppxAllUserStore.dat` | 离线注册表 hive（应用注册信息） |

### 2.2 暂存路径（相对于 UpdateStagingRoot）

| 路径 | 用途 |
|------|------|
| `\appx\<PackageFullName>\` | 解压后的应用文件 |
| `\appx\Bundle\<GUID>\<BundleName>\` | Bundle 解压临时目录 |

---

## 3. Stage 阶段（StageAppXPackages）

### 3.1 方法签名

```csharp
public void StageAppXPackages(IULogger logger,
    List<AppXFMPkgInfo> appXPkgFiles,     // 要安装的 AppX 包列表
    string msPackageRoot,                    // 微软包根目录
    string updateStagingRoot,                // 更新暂存根目录
    CpuId cpuType)                           // 目标 CPU 架构
```

### 3.2 执行流程

```
StageAppXPackages:
  1. 获取适用架构列表 (GetApplicableArchitectures)
     - x86 目标: x86 + neutral
     - amd64 目标: x64 + x86 + neutral
     - arm 目标: arm + neutral
     - arm64 目标: arm64 + arm + neutral

  2. 创建暂存目录: <updateStagingRoot>\appx\
  3. 定位 MakeAppx.exe (与当前程序集同目录)
  4. 创建 Bundle 临时目录: <staging>\Bundle\

  5. 并行处理每个 AppX 包 (Parallel.ForEach):
     a. 检查包文件是否存在
     b. 检查架构是否适用 (不适用则跳过)
     c. 如果是 .appxbundle:
        - MakeAppx.exe unbundle /o /v /p <bundle> /d <temp> /pfn
        - 解压到 Bundle\<GUID>\<BundleName>\
     d. 如果是 .appx:
        - MakeAppx.exe unpack /o /v /p <appx> /d <staging> /pfn
        - 直接解压到 <staging>\<PackageFullName>\

  6. 并行处理 Bundle 中的所有 .appx 子包:
     - 遍历 Bundle 临时目录中的所有 .appx 文件
     - 对每个 .appx 执行 MakeAppx.exe unpack

  7. 删除 Bundle 临时目录
  8. 输出总 MakeAppx 耗时
```

### 3.3 MakeAppx.exe 命令

**解压单个 .appx**:
```
MakeAppx.exe unpack /o /v /p "<appx_file>" /d "<destination>" /pfn
```
- `/o` — 覆盖现有文件
- `/v` — 详细输出
- `/p` — 输入包路径
- `/d` — 输出目录
- `/pfn` — 使用 Package Full Name 作为输出子目录名

**解压 .appxbundle**:
```
MakeAppx.exe unbundle /o /v /p "<bundle_file>" /d "<destination>" /pfn
```

### 3.4 架构过滤

```csharp
// 获取目标架构适用的所有架构
List<APPX_PACKAGE_ARCHITECTURE> applicableArchitectures =
    AppxUtils.GetApplicableArchitectures(cpuType);

// 检查包架构是否适用
APPX_PACKAGE_ARCHITECTURE pkgArch = AppxUtils.GetApplicableArchitecturesId(app.CpuType);
if (!applicableArchitectures.Contains(pkgArch))
{
    logger.LogInfo("Skipping {0}; architecture not applicable", app.Name);
    continue;
}
```

**APPX_PACKAGE_ARCHITECTURE 枚举**:
| 值 | 架构 |
|----|------|
| 0 | X86 |
| 5 | Arm |
| 9 | X64 |
| 12 | Neutral |
| 14 | Arm64 |

---

## 4. 提交大小估算（GetCommitSizeAppX）

### 4.1 方法

```csharp
public uint GetCommitSizeAppX(IULogger logger,
    List<AppXFMPkgInfo> AppXPkgFiles,
    string updateStagingRoot,
    CpuId cpuType,
    List<string> languages,
    List<string> resolutions)
```

### 4.2 流程

```
GetCommitSizeAppX:
  1. 遍历每个 AppX 包
  2. 用 appxpackaging.dll COM API 打开包:
     - 创建 IStream (CreateFileStream)
     - 如果是 bundle: IAppxBundleFactory → GetTotalUncompressedSizeOfBundle
     - 如果是单包: IAppxFactory → GetTotalUncompressedSizeOfPackage
  3. 累加所有包的未压缩大小
  4. 乘以 1.1 (10% 开销估算)
  5. 返回总大小（字节）
```

### 4.3 COM API 调用

```csharp
using AppXComObject<IStream> stream = new AppXComObject<IStream>(
    AppxUtils.CreateFileStream(packagePath, STGM.STGM_READ, 1, create: false));

uint size = AppxUtils.IsAppxBundleFile(path)
    ? AppxUtils.GetTotalUncompressedSizeOfBundle(stream.Ref())
    : AppxUtils.GetTotalUncompressedSizeOfPackage(stream.Ref());
```

---

## 5. Commit 阶段（CommitAppXPackages）

### 5.1 方法签名

```csharp
public void CommitAppXPackages(IULogger logger,
    List<AppXFMPkgInfo> appXPkgFiles,
    string msPackageRoot,
    string updateStagingRoot,
    string preInstalledVolumePath,    // 预安装卷根目录（MainOS 分区挂载点）
    CpuId cpuType,
    List<string> languages,
    List<string> resolutions)
```

### 5.2 执行流程

```
CommitAppXPackages:
  1. 创建 AppxInfoManager (管理包信息、依赖关系、处理顺序)

  2. 扫描暂存目录 <updateStagingRoot>\appx\ 中的所有子目录 (并行):
     对每个目录 (目录名 = PackageFullName):
     a. 提取 PackageFamilyName
     b. 检查是否在 appXPkgFiles 列表中（不在则跳过）
     c. 创建 AppxInfo 对象
     d. 判断是否为 Bundle:
        - 是 Bundle:
          * 用 IAppxBundleFactory 读取 AppxBundleManifest.xml
          * 获取: architecture, version, packageFullName, packageFamilyName
        - 不是 Bundle:
          * 用 IAppxFactory 读取 AppxManifest.xml
          * 获取: architecture, version, packageFullName, packageFamilyName
          * 检查架构适用性
          * 读取 Properties:
            - Framework=true → isFramework
            - ResourcePackage=true → isResource
            - 否则 → isMain (主应用)
          * 如果 isMain: 读取 PackageDependencies → 记录依赖关系
     e. 将 AppxInfo 添加到 AppxInfoManager (线程安全)

  3. AppxInfoManager.ProcessPackages():
     - 按依赖关系排序（框架包先于主应用）
     - 过滤不适用架构的包
     - 分类为: stagedPackages (所有), applicationsPackages (主应用)

  4. 复制许可证文件 (并行):
     对每个 appXPkgFiles:
     a. 检查架构适用性
     b. 如果有 LicenseFilePath:
        - 复制到 <preInstalledVolumePath>\AppData\<PackageFamilyName>.xml
     c. 没有许可证则跳过并记录日志

  5. 复制应用文件 (并行):
     对每个去重后的 stagedPackage (按 packageFullName 去重):
     a. 源: <updateStagingRoot>\appx\<PackageFullName>\
     b. 目标: <preInstalledVolumePath>\WindowsApps\<PackageFullName>\
     c. 递归复制目录 (FileUtils.CopyDirectory)

  6. 写注册表 hive:
     a. 删除旧的 AppxAllUserStore.dat (如果存在)
     b. RegLoadAppKey() 创建/加载离线注册表 hive (offreg.dll)
     c. 对每个 applicationPackage (主应用):
        - AppxUtils.AddApplication() → 写 HKLM\...\AppxAllUserStore\Applications\<PackageFullName>
     d. 对每个 stagedPackage:
        - AppxUtils.AddStaged() → 写 HKLM\...\AppxAllUserStore\Staged\<PackageFamilyName>\<PackageFullName>
     e. 关闭注册表句柄 (自动 flush 到 .dat 文件)

  7. 清理: 删除暂存目录 <updateStagingRoot>\appx\
  8. GC.Collect() × 2 (释放 COM 对象和内存)
```

### 5.3 注册表 hive 结构

```
AppxAllUserStore.dat (离线注册表 hive)
└─ AppxAllUserStore
   ├─ Applications
   │  └─ <PackageFullName>          (主应用注册信息)
   │     ├─ (Default) = ...
   │     ├─ DisplayName = ...
   │     ├─ PackageFamilyName = ...
   │     └─ ...
   └─ Staged
      └─ <PackageFamilyName>
         └─ <PackageFullName>        (暂存包注册信息)
            ├─ (Default) = ...
            ├─ PackageFullName = ...
            └─ ...
```

### 5.4 RegLoadAppKey (offreg.dll)

```csharp
// 创建/加载离线注册表 hive
int hiveHandle = AppxUtils.RegLoadAppKey(hivePath);
// hivePath = <preInstalledVolumePath>\WindowsApps\AppxAllUserStore.dat

using (SafeRegistryHandle handle = new SafeRegistryHandle(new IntPtr(hiveHandle), ownsHandle: true))
{
    RegistryKey regKey = RegistryKey.FromHandle(handle);
    // 写入应用注册信息...
}
// 释放句柄时自动 flush 到 .dat 文件
```

`RegLoadAppKey` 是 offreg.dll 的导出函数，用于在不挂载系统注册表的情况下操作离线注册表 hive 文件。

---

## 6. appxpackaging.dll COM API 使用

### 6.1 核心 COM 接口

AppX 预安装通过 `appxpackaging.dll` 提供的 COM API 读取包清单：

| 接口 | 用途 |
|------|------|
| `IAppxFactory` | 创建单包 manifest reader |
| `IAppxBundleFactory` | 创建 bundle manifest reader |
| `IAppxManifestReader` | 读取单包 AppxManifest.xml |
| `IAppxBundleManifestReader` | 读取 bundle AppxBundleManifest.xml |
| `IAppxManifestPackageId` | 包标识（名称、版本、架构、发布者） |
| `IAppxManifestProperties` | 包属性（Framework、ResourcePackage 等） |
| `IAppxManifestPackageDependenciesEnumerator` | 包依赖枚举器 |
| `IAppxManifestPackageDependency` | 单个包依赖 |
| `IStream` | 文件流（用于打开 .appx/.appxbundle） |

### 6.2 读取单包清单

```csharp
// 1. 创建文件流
IStream stream = AppxUtils.CreateFileStream(manifestPath, STGM.STGM_READ, 1, create: false);

// 2. 创建 IAppxFactory 并读取 manifest
IAppxFactory factory = (IAppxFactory)new AppxFactory();
IAppxManifestReader reader = factory.CreateManifestReader(stream);

// 3. 获取包 ID
IAppxManifestPackageId pkgId = reader.GetPackageId();
APPX_PACKAGE_ARCHITECTURE arch = pkgId.GetArchitecture();
string version = pkgId.GetVersion();
string fullName = pkgId.GetPackageFullName();
string familyName = pkgId.GetPackageFamilyName();

// 4. 获取属性
IAppxManifestProperties props = reader.GetProperties();
bool isFramework = props.GetBoolValue("Framework");
bool isResource = props.GetBoolValue("ResourcePackage");

// 5. 获取依赖
IAppxManifestPackageDependenciesEnumerator deps = reader.GetPackageDependencies();
while (deps.GetHasCurrent())
{
    IAppxManifestPackageDependency dep = deps.GetCurrent();
    string depFamilyName = AppxUtils.GetPackageFaimlyName(dep.GetName(), dep.GetPublisher());
    deps.MoveNext();
}
```

### 6.3 读取 Bundle 清单

```csharp
IAppxBundleFactory bundleFactory = (IAppxBundleFactory)new AppxBundleFactory();
IAppxBundleManifestReader bundleReader = bundleFactory.CreateBundleManifestReader(stream);
IAppxManifestPackageId pkgId = bundleReader.GetPackageId();
// 获取 architecture, version, packageFullName, packageFamilyName
```

### 6.4 AppXComObject<T> 包装器

代码使用 `AppXComObject<T>` 泛型包装器管理 COM 对象的生命周期，确保正确释放：

```csharp
using AppXComObject<IAppxFactory> factory = new AppXComObject<IAppxFactory>((IAppxFactory)new AppxFactory());
using AppXComObject<IStream> stream = new AppXComObject<IStream>(AppxUtils.CreateFileStream(...));
using AppXComObject<IAppxManifestReader> reader = new AppXComObject<IAppxManifestReader>(
    factory.Ref().CreateManifestReader(stream.Ref()));
// using 块结束时自动调用 Marshal.ReleaseComObject
```

---

## 7. AppxInfoManager（包管理器）

### 7.1 作用

`AppxInfoManager` 负责：
1. 收集所有暂存包的信息
2. 管理包依赖关系
3. 按依赖关系排序（框架包先于主应用）
4. 分类包（主应用 / 框架包 / 资源包）
5. 过滤不适用架构的包

### 7.2 AppxInfo 结构

```csharp
public class AppxInfo
{
    public string manifestPath;           // 清单文件路径
    public APPX_PACKAGE_ARCHITECTURE architecture;  // 包架构
    public string version;                // 版本
    public string packageFullName;        // 完整包名
    public string packageFamilyName;      // 包族名
    public bool isBundle;                 // 是否为 Bundle
    public bool isFramework;              // 是否为框架包
    public bool isResource;               // 是否为资源包
    public bool isMain;                   // 是否为主应用
}
```

### 7.3 包分类逻辑

```
读取 AppxManifest.xml Properties:
  ├─ Framework = true        → isFramework = true (框架包，如 .NET Native 运行时)
  ├─ ResourcePackage = true  → isResource = true (资源包，如语言包/主题)
  └─ 否则                     → isMain = true (主应用)
```

### 7.4 依赖处理

```csharp
// 主应用记录其依赖的框架包族名
appxInfoManager.AddDependency(packageFamilyName, dependencyFamilyNames);

// ProcessPackages() 中按依赖排序
// 框架包必须在主应用之前注册到注册表
```

---

## 8. AppXFMPkgInfo（包输入信息）

### 8.1 结构

```csharp
public class AppXFMPkgInfo
{
    public string ID { get; set; }              // PackageFamilyName
    public string Name { get; set; }            // 显示名称
    public string PackagePath { get; set; }     // .appx/.appxbundle 文件路径
    public string LicenseFilePath { get; set; } // 许可证 .xml 路径
    public CpuId CpuType { get; set; }          // 包架构
}
```

### 8.2 来源

`AppXFMPkgInfo` 列表来自：
1. **CompDB 中的 AppX 部分** (`CDBAddOnAppX`) — CompDB XML 中的 `<AppX>` 元素
2. **OEMInput 中的 AppXOptionalPackages** — `<AppXOptionalPackages><AppXID>` 列表
3. **FeatureManifest 中的 AppX 包引用** — FM 文件中定义的 AppX 包

---

## 9. 重建要点

### 9.1 必须保留的机制

1. **两阶段分离**: Stage (MakeAppx 解压) 和 Commit (复制+注册表) 必须分离
2. **架构过滤**: 必须按目标架构过滤包（amd64 包含 x86 模拟，arm64 包含 arm 模拟）
3. **Bundle 处理**: .appxbundle 必须先 unbundle，再对其中的 .appx 子包 unpack
4. **依赖排序**: 框架包必须在主应用之前注册（注册表写入顺序）
5. **许可证复制**: 许可证文件必须复制到 `AppData\<PackageFamilyName>.xml`
6. **注册表 hive**: 必须使用 offreg.dll 的 RegLoadAppKey 创建离线注册表 hive
7. **去重**: 按 PackageFullName 去重（Bundle 中可能有重复架构的包）
8. **MakeAppx /pfn 参数**: 必须使用 /pfn 确保输出目录名为 PackageFullName

### 9.2 可简化的部分

1. **并行处理**: Parallel.ForEach 可简化为顺序处理（功能不变，仅速度差异）
2. **GC.Collect**: 双重 GC 可简化为单次或省略
3. **AppXComObject 包装器**: 可直接使用 try/finally + Marshal.ReleaseComObject
4. **提交大小估算**: 1.1x 倍率可调整或精确计算

### 9.3 关键依赖

| 依赖 | 用途 | 可替代方案 |
|------|------|-----------|
| MakeAppx.exe | .appx/.appxbundle 解压 | 自己实现 ZIP 解压（.appx 本质是 ZIP） |
| appxpackaging.dll | 读取包清单、获取未压缩大小 | 解析 AppxManifest.xml (ZIP 内的 XML) |
| offreg.dll | 离线注册表 hive 操作 | 直接操作 .dat 文件二进制格式（复杂） |
| .NET Framework 4.0 | 运行时 | .NET 6+/Core |

### 9.4 .appx 文件格式

.appx 本质上是一个 ZIP 压缩包，包含：
- `AppxManifest.xml` — 包清单（必选）
- `AppxBundleManifest.xml` — Bundle 清单（仅 .appxbundle）
- `resources.pri` — 资源索引
- 应用文件（.exe, .dll, .xaml, 图片等）
- `[Content_Types].xml` — OPC 内容类型

因此可以用标准 ZIP 库替代 MakeAppx.exe 进行解压，但 MakeAppx.exe 还处理包签名验证和 /pfn 目录命名。

---

## 10. 与 CBS 包的关系

### 10.1 区别

| 特性 | CBS 包 (.cab) | AppX 包 (.appx) |
|------|---------------|-----------------|
| 格式 | Microsoft Cabinet (CAB) | ZIP (OPC) |
| 清单 | update.mum (CBS 清单) | AppxManifest.xml |
| 安装引擎 | UpdateDLL + wcp.dll | appxpackaging.dll + MakeAppx |
| 注册方式 | 离线注册表 + 文件 | 离线注册表 hive + WindowsApps 目录 |
| 暂存位置 | UpdateStagingRoot (各分区 TempSxS) | UpdateStagingRoot\appx |
| 目标位置 | 系统目录 (\Windows\System32 等) | \WindowsApps\<PackageFullName> |
| 许可证 | 嵌入 .cat 签名 | 独立 .xml 文件 → \AppData\ |

### 10.2 协同

AppX 预安装在 CBS 包提交**之后**执行，因为：
1. AppX 依赖的系统组件（如 .NET Native 运行时框架包）可能通过 CBS 包安装
2. WindowsApps 目录和 AppData 目录需要在 CBS 包创建的文件系统结构上建立
3. 注册表 hive 写入需要在 CBS 包的注册表操作完成后进行（避免覆盖）

---

## 11. 源码行号索引

| 功能 | appximaging.cs 行号 |
|------|---------------------|
| AppXImaging 类定义 | 32 |
| 常量定义 | 34-48 |
| StageAppXPackages | 50 |
| UnbundleAppxBundleContents | 108 |
| UnpackAppx | 123 |
| GetCommitSizeAppX | 135 |
| CommitAppXPackages | 155 |
| 扫描暂存目录 + 读取清单 | 162-236 |
| 复制许可证 | 241-266 |
| 复制应用文件 | 267-278 |
| 写注册表 hive | 279-305 |
| 清理 + GC | 306-316 |

---

*文档生成时间: 2026-08-27*
*研究工具: ILSpy 反编译 .NET 程序集 + VS Code 源码分析*
*ADK 版本: 10.0.17704.1000 (Windows 10 RS5)*
