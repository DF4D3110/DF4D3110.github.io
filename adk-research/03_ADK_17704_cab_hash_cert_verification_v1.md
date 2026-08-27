# ADK 17704 CAB 包文件 Hash 与证书验证机制深度研究

> 生成日期：2026-08-27
> 研究范围：CAB 包加载、文件 hash 存储与验证、证书链验证、生产镜像校验、FFU hash 链
> 数据来源：ILSpy 反编译 (.NET) + PE 导出表分析 (Native) + IDA Hex-Rays 反编译

---

## 一、CAB 包加载与解析机制

### 1.1 包加载调用链

```
Package.LoadFromCab(cabPath)                          [公共入口]
  → WPCanonicalPackage.LoadFromCab(cabPath)
    → WPCabPackage.Load(cabPath)
      → PkgManifest.Load(cabPath, "update.mum" 或 "dsm.xml")
        ├─ 检测 CAB 内是否包含 "update.mum"
        │   ├─ 包含 → PkgManifest.Load_CBS(cabPath)   [CBS 包]
        │   └─ 不包含 → 提取 DSM → PkgManifest.Load(dsmPath)  [SPKG 包]
        └─ ImportFromNativeObject() → 从原生对象导入所有字段
```

### 1.2 CBS 包 vs SPKG 包

| 特性 | CBS 包 (Component Based Servicing) | SPKG 包 (Signed Package) |
|------|-------------------------------------|--------------------------|
| 标识文件 | `update.mum` (CBS 清单) | `dsm.xml` (Device Side Manifest) |
| 加载方式 | `DeviceSideManifest_Load_CBS()` 原生 API | 提取 DSM → `DeviceSideManifest_Load()` |
| 原生 DLL | UpdateDLL.dll | UpdateDLL.dll |
| 包样式枚举 | `PackageStyle.CBS` | `PackageStyle.SPKG` |
| 典型来源 | ConvertDSM.exe 输出 (.cab) | spkggen.exe 输出 (.spkg) |

### 1.3 PkgManifest.Load_CBS 原生调用

```csharp
// pkgcommonmanaged.cs
public static PkgManifest Load_CBS(string pkgPath)
{
    IntPtr dsmObj = NativeMethods.DeviceSideManifest_Create();
    try
    {
        // ★ 核心：直接从 CAB 文件加载 CBS 清单
        int hr = NativeMethods.DeviceSideManifest_Load_CBS(
            dsmObj, LongPath.GetFullPathUNC(pkgPath));
        NativeMethods.CheckHResultWithAdditionalInfo(hr,
            "DeviceSideManifest_Load_CBS", "Package File: " + pkgPath);

        PkgManifest manifest = new PkgManifest();
        manifest.PackageStyle = PackageStyle.CBS;
        manifest.ImportFromNativeObject(dsmObj);  // 导入所有字段+文件列表
        return manifest;
    }
    finally { NativeMethods.DeviceSideManifest_Free(dsmObj); }
}
```

**原生 API 签名** (UpdateDLL.dll, cdecl 调用约定):
```c
int DeviceSideManifest_Load_CBS(IntPtr objPtr, LPCWSTR cabPath);
```

---

## 二、文件 Hash 存储机制

### 2.1 FileEntryBase — 文件条目基类

```csharp
// pkgcommonmanaged.cs
public class FileEntryBase
{
    private string _fileHash = string.Empty;   // ★ 文件内容 hash (字符串)
    private ulong _size;                         // 原始文件大小
    private ulong _compressedSize;               // 压缩后大小
    private ulong _stagedSize;                   // 暂存大小

    public FileType FileType { get; set; }      // 文件类型枚举
    public string SourcePath { get; set; }
    public string DevicePath { get; set; }      // 设备上的目标路径
    public string CabPath { get; set; }         // CAB 内的路径
    public string FileArch { get; set; }
    public bool SignInfoRequired { get; set; }   // ★ 是否需要签名信息

    public string FileHash
    {
        get { return string.IsNullOrEmpty(_fileHash) ? "" : _fileHash; }
        set { _fileHash = string.IsNullOrEmpty(value) ? "" : value; }
    }
    // Size/CompressedSize/StagedSize 通过 IU_GetStagedAndCompressedSize 惰性计算
}
```

### 2.2 FileEntry — 完整文件条目

```csharp
public sealed class FileEntry : FileEntryBase, IFileEntry
{
    public FileAttributes Attributes { get; set; }
    public string SourcePackage { get; private set; }       // 源包名
    public string EmbeddedSigningCategory { get; private set; } // 嵌入签名类别

    // 从原生对象构造
    public FileEntry(IntPtr filePtr) : base(filePtr)
    {
        Attributes = NativeMethods.DSMFileEntry_Get_Attributes(filePtr);
        SourcePackage = NativeMethods.DSMFileEntry_Get_SourcePackage(filePtr);
        EmbeddedSigningCategory = NativeMethods.DSMFileEntry_Get_EmbeddedSigningCategory(filePtr);
        base.Size = NativeMethods.DSMFileEntry_Get_FileSize(filePtr);
        base.CompressedSize = NativeMethods.DSMFileEntry_Get_CompressedFileSize(filePtr);
        base.StagedSize = NativeMethods.DSMFileEntry_Get_StagedFileSize(filePtr);
    }
}
```

### 2.3 FileType 枚举 — 文件类型

```csharp
public enum FileType
{
    Invalid,
    Regular,                    // 普通文件
    Registry,                   // 注册表文件
    SecurityPolicy,             // 安全策略
    Reserved,
    BinaryPartition,            // 二进制分区
    Manifest,                   // ★ 清单文件 (DSM/MUM)
    RegistryMultiStringAppend,
    Certificates,               // ★ 证书文件
    Catalog,                    // ★ 目录文件 (.cat)
    DirectoryBackupMetadata,
    FileBackupMetadata
}
```

### 2.4 原生文件添加 API — hash 传递

```csharp
// UpdateDLL.dll 原生 API
[DllImport("UpdateDLL.dll", CallingConvention = CallingConvention.Cdecl, CharSet = CharSet.Unicode)]
public static extern int DeviceSideManifest_Add_File(
    IntPtr objPtr,
    FileType fileType,
    string devicePath,
    string cabPath,
    FileAttributes attributes,
    string sourcePackage,
    string embedSignCategory,
    ulong FileSize,           // 原始大小
    ulong CompressedFileSize, // 压缩大小
    ulong StagedFileSize,     // 暂存大小
    string fileHash,          // ★ 文件 hash (字符串)
    [MarshalAs(UnmanagedType.U1)] bool signFile);  // ★ 是否需要签名
```

### 2.5 包级 Hash 计算

```csharp
// imagecommon.cs — CompDBPackageInfo
public static string GetPackageHash(string packageFile)
{
    byte[] hash = PackageTools.CalculateFileHash(packageFile);  // SHA256
    return Convert.ToBase64String(hash);
}

public static string GetPackageSha1Hash(string packageFile)
{
    byte[] hash = PackageTools.CalculateFileSha1Hash(packageFile);  // SHA1
    return Convert.ToBase64String(hash);
}
```

**Hash 用途**：
- **SHA256** (`CalculateFileHash`)：CompDB 中包的主要 hash，用于包去重和版本比较
- **SHA1** (`CalculateFileSha1Hash`)：CompDB 中包的辅助 hash，兼容旧版系统
- 两者均以 Base64 字符串存储在 CompDB XML 中

---

## 三、证书验证机制

### 3.1 ImageSigner 类 — 核心证书验证

```csharp
// imagecommon.cs
public class ImageSigner
{
    private SHA256 _sha256;
    private static Dictionary<string, bool> certPublicKeys = new Dictionary<string, bool>();

    // ★ 硬编码的 Microsoft 证书指纹
    public const string ProdCertRootThumbprint    = "3B1EFD3A66EA28B16697394703A72CA340A05BD5";
    public const string TestCertRootThumbprint    = "8A334AA8052DD244A647306A76B8178FA215F344";
    public const string FlightCertPCAThumbprint   = "9E594333273339A97051B0F82E86F266B917EDB3";
    public const string FlightCertWindowsThumbprint = "5f444a6740b7ca2434c7a5925222c2339ee0f1b7";
}
```

### 3.2 HasSignature — 签名验证核心方法

```csharp
public static bool HasSignature(string filename, bool EnsureMicrosoftIssuer)
{
    X509Certificate2 cert = null;
    bool result = false;
    try
    {
        // 从文件加载证书 (支持 PE/CAB/CAT 等带 Authenticode 签名的文件)
        cert = new X509Certificate2(FileToolBox.ReadAllBytes(filename));

        if (EnsureMicrosoftIssuer)
        {
            // 检查缓存
            if (!certPublicKeys.TryGetValue(cert.Thumbprint, out result))
            {
                // 构建证书链 (使用机器上下文)
                X509Chain chain = new X509Chain(useMachineContext: true);
                chain.Build(cert);

                // 遍历证书链，检查是否包含 Microsoft 根/中间证书
                foreach (X509ChainElement element in chain.ChainElements)
                {
                    string tp = element.Certificate.Thumbprint;
                    if (tp.Equals(ProdCertRootThumbprint,    OrdinalIgnoreCase) ||
                        tp.Equals(FlightCertPCAThumbprint,   OrdinalIgnoreCase) ||
                        tp.Equals(FlightCertWindowsThumbprint, OrdinalIgnoreCase) ||
                        tp.Equals(TestCertRootThumbprint,    OrdinalIgnoreCase))
                    {
                        result = true;
                        break;
                    }
                }
                // 缓存链中所有证书的验证结果
                foreach (X509ChainElement element in chain.ChainElements)
                    certPublicKeys[element.Certificate.Thumbprint] = result;
            }
        }
        else
        {
            // 不要求 Microsoft 颁发者，只要有签名即可
            result = (cert != null && !string.IsNullOrEmpty(cert.Subject));
        }
    }
    catch { result = false; }
    return result;
}
```

**验证逻辑**：
1. 用 `X509Certificate2` 从文件字节加载 Authenticode 签名证书
2. 构建 X509 证书链（机器上下文）
3. 遍历链中每个证书，比对指纹是否匹配 4 个 Microsoft 已知证书之一
4. 结果缓存到 `certPublicKeys` 字典（按证书指纹）

### 3.3 HasValidSignature — 通用有效签名验证

```csharp
public static bool HasValidSignature(string filename, List<string> validRootThumbprints)
{
    X509Certificate2 cert = new X509Certificate2(filename);
    X509Chain chain = new X509Chain(useMachineContext: true);
    chain.Build(cert);
    // 检查链中是否有任何证书的指纹在有效根列表中
    return chain.ChainElements.Cast<X509ChainElement>()
        .Any(e => validRootThumbprints.Any(tp =>
            tp.Equals(e.Certificate.Thumbprint, StringComparison.OrdinalIgnoreCase)));
}
```

### 3.4 Microsoft 证书指纹详解

| 常量名 | 指纹 | 证书类型 |
|--------|------|----------|
| `ProdCertRootThumbprint` | `3B1EFD3A66EA28B16697394703A72CA340A05BD5` | Microsoft Production Root CA（生产根证书） |
| `TestCertRootThumbprint` | `8A334AA8052DD244A647306A76B8178FA215F344` | Microsoft Test Root CA（测试根证书） |
| `FlightCertPCAThumbprint` | `9E594333273339A97051B0F82E86F266B917EDB3` | Microsoft Flight PCA（飞行签名中间证书） |
| `FlightCertWindowsThumbprint` | `5f444a6740b7ca2434c7a5925222c2339ee0f1b7` | Microsoft Flight Windows（飞行签名 Windows 证书） |

---

## 四、生产镜像验证 (ValidateProductionImage)

### 4.1 验证流程

```csharp
// imaging.cs
private void ValidateProductionImage()
{
    // 1. 仅在新建 + Production 模式下执行
    if (_bDoingUpdate || _releaseType != ReleaseType.Production) return;

    // 2. BuildType 必须为 fre (Retail)
    if (!_oemInput.BuildType.Equals("fre", OrdinalIgnoreCase))
        throw new ImageCommonException("BuildType must be 'fre' for Retail images");

    // 3. 生产镜像不允许 OEMInput 中有 PackageFiles
    if (_oemInput.PackageFiles != null && _oemInput.PackageFiles.Count() > 0)
        throw new ImageCommonException("Retail images cannot use PackageFiles");

    // 4. 遍历所有包
    foreach (IPkgInfo pkg in _packageInfoList.Values)
    {
        // 4a. 检查包的 ReleaseType 和 BuildType
        if (pkg.ReleaseType != ReleaseType.Production ||
            pkg.BuildType != BuildType.Retail)
        {
            _logger.LogInfo("Non-production package: {0}", pkg.Name);
            nonProdPackages.Append("\t'" + pkg.Name + "'");
        }

        // 4b. 提取包中的 Catalog 文件并验证签名
        var catalogFiles = pkg.Files.Where(f => f.FileType == FileType.Catalog);
        if (catalogFiles.Count() == 0)
        {
            _logger.LogWarning("Package has no queryable catalog: " + pkg.Name);
            continue;
        }
        IFileEntry catalogEntry = catalogFiles.First();
        pkg.ExtractFile(catalogEntry.DevicePath, tempCatPath, overwrite: true);

        // ★ 核心：Microsoft 包必须有 Microsoft 颁发的签名
        bool isMicrosoft = (pkg.OwnerType == OwnerType.Microsoft);
        if (!ImageSigner.HasSignature(tempCatPath, EnsureMicrosoftIssuer: isMicrosoft))
        {
            improperlySigned.Append("\t'" + pkg.Name + "': (" + pkg.OwnerType + ")");
        }
        FileToolBox.Delete(tempCatPath);
    }

    // 5. FFU 模式：错误致命；VHD 模式：仅记录
    if (_bDoingFFU && (nonProdPackages.Length > 0 || improperlySigned.Length > 0))
        throw new ImageCommonException("Production image validation failed.");
}
```

### 4.2 验证规则矩阵

| 检查项 | FFU 模式 | VHD 模式 |
|--------|----------|----------|
| BuildType 必须为 fre | 致命错误 | 致命错误 |
| 禁止 PackageFiles | 致命错误 | 致命错误 |
| 包 ReleaseType=Production | 致命错误 | 仅日志 |
| 包 BuildType=Retail | 致命错误 | 仅日志 |
| Microsoft 包有 Microsoft 签名 | 致命错误 | 仅日志 |
| OEM 包有任意签名 | 致命错误 | 仅日志 |

### 4.3 包所有者类型 (OwnerType)

```csharp
public enum OwnerType
{
    Invalid,
    Microsoft,       // 微软包 — 必须有 Microsoft 颁发的签名
    OEM,             // OEM 包 — 必须有任意有效签名
    SiliconVendor,   // 芯片厂商包
    MobileOperator   // 移动运营商包
}
```

---

## 五、FFU 镜像 Hash 链验证

### 5.1 三层 Hash 链结构

```
┌─────────────────────────────────────────────────────────┐
│  第一层：Catalog (.cat)                                   │
│  包含 HashTable 的 SHA1 hash (20字节)                     │
│  由 MakeCat.exe 生成，Authenticode 签名                   │
├─────────────────────────────────────────────────────────┤
│  第二层：HashTable                                        │
│  每个 Chunk 的 SHA256 hash (32字节/个)                   │
│  连续存储，总大小 = Chunk数 × 32                          │
├─────────────────────────────────────────────────────────┤
│  第三层：Payload (镜像数据)                                │
│  按 ChunkSize (默认2MB) 分块                              │
│  每块独立计算 SHA256                                       │
└─────────────────────────────────────────────────────────┘
```

### 5.2 VerifyCatalog — 完整验证入口

```csharp
public void VerifyCatalog()
{
    // 1. 验证 Catalog 中的 hash 与 HashTable 匹配
    if (!VerifyCatalogData(_ffuImage.CatalogData, _ffuImage.HashTableData))
        throw new ImageCommonException("Catalog does not match Hash Table");

    // 2. 验证 HashTable 中每个 chunk hash 与实际 payload 匹配
    if (!VerifyHashTable())
        throw new ImageCommonException("Hash Table does not match payload");
}
```

### 5.3 VerifyCatalogData — Catalog vs HashTable

```csharp
public static bool VerifyCatalogData(string catalogFile, byte[] hashTableData)
{
    SHA1Managed sha1 = new SHA1Managed();
    byte[] catalogHash = GetCatalogHash(catalogFile);  // 从 Catalog 提取 SHA1
    byte[] computedHash = sha1.ComputeHash(hashTableData);  // 计算 HashTable 的 SHA1

    // 逐字节比较
    if (catalogHash.Length != computedHash.Length) return false;
    for (int i = 0; i < computedHash.Length; i++)
        if (catalogHash[i] != computedHash[i]) return false;
    return true;
}
```

### 5.4 GetCatalogHash — 从 Catalog 提取 Hash

```csharp
internal static byte[] GetCatalogHash(string catalogFile)
{
    IntPtr hCatalog = CryptCATOpen(catalogFile, 2, IntPtr.Zero, 0, 0);
    IntPtr hMember = CryptCATEnumerateMember(hCatalog, IntPtr.Zero);
    CRYPTCATMEMBER member = (CRYPTCATMEMBER)Marshal.PtrToStructure(hMember, typeof(CRYPTCATMEMBER));

    // ★ 关键：从 IndirectData 的末尾提取 20 字节 SHA1
    byte[] hash = new byte[20];
    int offset = (int)member.sEncodedIndirectData.cbData - hash.Length;
    Marshal.Copy(IntPtr.Add(member.sEncodedIndirectData.pbData, offset), hash, 0, hash.Length);

    CryptCATClose(hCatalog);
    return hash;
}
```

**原理**：Catalog 成员的 `sEncodedIndirectData` 是一个 `SPC_INDIRECT_DATA_CONTENT` 结构，其末尾包含被引用文件的 hash。对于 HashTable，这个 hash 是 SHA1（20字节）。

### 5.5 VerifyHashTable — HashTable vs Payload

```csharp
internal bool VerifyHashTable()
{
    int hashOffset = 0;
    byte[] chunkHash = null;
    int chunkNum = 0;

    byte[] hashTableData = _ffuImage.HashTableData;
    using (FileStream stream = _ffuImage.GetImageStream())
    {
        stream.Position = _ffuImage.StartOfImageHeader;
        chunkHash = GetFirstChunkHash(stream);  // 跳过安全数据，读第一个 chunk
        chunkNum++;

        while (chunkHash != null)
        {
            // 逐字节比较 chunk 的 SHA256 与 HashTable 中的对应条目
            for (int i = 0; i < chunkHash.Length; i++)
            {
                if (hashOffset > hashTableData.Length)
                    throw new ImageCommonException("Hash Table too small");
                if (chunkHash[i] != hashTableData[hashOffset])
                    throw new ImageCommonException(
                        $"Chunk {chunkNum} hash mismatch at byte {i}: " +
                        $"{chunkHash[i]:X2} vs {hashTableData[hashOffset]:X2}");
                hashOffset++;
            }
            chunkHash = GetNextChunkHash(stream);
            chunkNum++;
        }
    }
    return true;
}

private byte[] GetNextChunkHash(Stream stream)
{
    byte[] chunk = new byte[_ffuImage.ChunkSizeInBytes];  // 默认 2MB
    if (stream.Position == stream.Length) return null;
    stream.Read(chunk, 0, chunk.Length);
    return _sha256.ComputeHash(chunk);  // SHA256
}
```

### 5.6 GenerateCatalogFile — Catalog 生成

```csharp
public static byte[] GenerateCatalogFile(byte[] hashData)
{
    // 1. 写 HashTable.blob 临时文件
    FileToolBox.WriteAllBytes(tempBlobPath, hashData);

    // 2. 生成 CDF (Catalog Definition File)
    using (StreamWriter sw = new StreamWriter(tempCdfPath))
    {
        sw.WriteLine("[CatalogHeader]");
        sw.WriteLine("Name={0}", tempCatPath);
        sw.WriteLine("[CatalogFiles]");
        sw.WriteLine("{0}={1}", "HashTable.blob", tempBlobPath);
    }

    // 3. 调用 MakeCat.exe 生成 .cat 文件
    Process.Start("MakeCat.exe", $"\"{tempCdfPath}\"");

    // 4. 读取生成的 .cat 文件
    return FileToolBox.ReadAllBytes(tempCatPath);
}
```

---

## 六、signinfohelper.dll — 签名信息辅助

### 6.1 导出函数分析

通过 PE 导出表分析，`signinfohelper.dll` (8704字节) **仅导出一个函数**：

```
Export: IsSignInfoRequired
```

### 6.2 用途推断

- `FileEntryBase.SignInfoRequired` 字段指示某个文件是否需要签名信息
- `IsSignInfoRequired` 原生函数用于判断给定文件是否需要嵌入签名信息
- 在 `DeviceSideManifest_Add_File` 原生 API 中，`signFile` 参数与此相关
- 包构建时 (PkgGen/spkggen)，根据文件类型和内容决定是否需要签名

---

## 七、包签名机制

### 7.1 PackageTools.SignFile — 包签名入口

```csharp
// pkgtoolbox.cs
public static void SignFile(string filePath)
{
    SignFile(filePath, SigningHashAlgorithm.SHA2);  // 默认 SHA256
}

public static void SignFiles(List<string> filesToSign, bool testOnly, SigningHashAlgorithm algo)
{
    if (IsOEMSigningScenario())
        DoSigningOemScenario(filesToSign);   // OEM 场景：sign.cmd
    else
        DoSigningBuildScenario(filesToSign, testOnly, algo);  // 构建场景：signer.cmd
}
```

### 7.2 两种签名场景

| 场景 | 检测条件 | 签名工具 | 命令 |
|------|----------|----------|------|
| **Build 场景** | `signer.cmd` 存在于 PATH | signer.cmd | `signer.cmd authenticode -s <scenario> -i -f <filelist>` |
| **OEM 场景** | `signer.cmd` 不存在 | sign.cmd | `sign.cmd <file>` (并行) |

### 7.3 签名 Hash 算法

```csharp
public enum SigningHashAlgorithm
{
    SHA1,      // WindowsSystemComponentSha1
    SHA2,      // WindowsSystemComponent (默认)
    SHA1AND2   // WindowsSystemComponentDualSigned (双签名)
}
```

### 7.4 签名场景字符串

| 算法 | signer.cmd 场景参数 |
|------|---------------------|
| SHA1 | `WindowsSystemComponentSha1` |
| SHA2 | `WindowsSystemComponent` |
| SHA1AND2 | `WindowsSystemComponentDualSigned` |

---

## 八、CBS 原生 Hash 验证 (UpdateDLL / wcp / cbscore)

### 8.1 两阶段更新中的 Hash 验证

```
Stage 阶段 (PrepareUpdate):
  UpdateDLL.PrepareUpdateWithFlags()
    → 解析 UpdateInput.xml 中的包列表
    → 解压每个 .cab 到 UpdateStagingRoot\<Partition>\TempSxS\
    → wcp.dll 解析 CBS 清单 (update.mum)
    → 提取每个文件的 hash (嵌入在清单中)
    → 验证包签名 (Authenticode)
    → 计算暂存空间需求

Commit 阶段 (ExecuteUpdate):
  UpdateDLL.ExecuteUpdate()
    → 挂载离线注册表配置单元
    → wcp.dll / cbscore.dll 执行组件服务
    → 对每个文件：
      ├─ 从暂存区复制到目标分区
      ├─ 计算文件实际 hash
      ├─ 与清单中的 hash 比对
      └─ 不匹配则报错 (CBS_E_CORRUPT_FILE 等)
    → 应用注册表、权限、服务等
    → 生成 UpdateHistory.xml
```

### 8.2 关键原生 API

```c
// UpdateDLL.dll — 文件大小/暂存大小查询
uint IU_GetStagedAndCompressedSize(
    LPCWSTR file,
    out ULONG64 fileSize,
    out ULONG64 stagedSize,
    out ULONG64 compressedSize);

// UpdateDLL.dll — 清单加载
int DeviceSideManifest_Load_CBS(IntPtr obj, LPCWSTR cabPath);
int DeviceSideManifest_Load(IntPtr obj, LPCWSTR xmlPath);

// UpdateDLL.dll — 文件条目属性
FileType FileEntryBase_Get_FileType(IntPtr file);
LPCWSTR FileEntryBase_Get_DevicePath(IntPtr file);
LPCWSTR FileEntryBase_Get_CabPath(IntPtr file);
LPCWSTR FileEntryBase_Get_FileHash(IntPtr file);    // ★ 文件 hash
bool FileEntryBase_Get_SignInfo(IntPtr file);         // ★ 签名信息需求
```

### 8.3 CBS 清单中的 Hash 格式

CBS 包的 `update.mum` 清单中，每个文件条目包含：
- **`hash`** 属性：文件内容的 SHA256 hash（十六进制字符串）
- **`size`** 属性：文件原始大小
- **`compressedSize`** 属性：压缩后大小
- **`stagedSize`** 属性：暂存大小（含对齐开销）

这些值在包构建时由 spkggen/ConvertDSM 计算并嵌入，在 CBS 服务时由 wcp.dll 验证。

---

## 九、CompDB 中的 Hash 存储

### 9.1 CompDBPayloadInfo — 包负载 Hash

```csharp
public class CompDBPayloadInfo
{
    public string PayloadHash { get; set; }      // SHA256 (Base64)
    public string PayloadSha1Hash { get; set; }  // SHA1 (Base64)
    public long PayloadSize { get; set; }

    public void SetPayloadHash(string payloadFile)
    {
        PayloadHash = GetPayloadHash(payloadFile);        // SHA256 Base64
        PayloadSha1Hash = GetPayloadSha1Hash(payloadFile); // SHA1 Base64
        PayloadSize = GetPayloadSize(payloadFile);
    }

    public static string GetPayloadHash(string payloadFile)
    {
        byte[] hash = PackageTools.CalculateFileHash(payloadFile);  // SHA256
        return Convert.ToBase64String(hash);
    }
}
```

### 9.2 Hash 比较模式

```csharp
public enum CompDBPackageInfoComparison
{
    Standard,               // 完整比较 (含 hash)
    IgnorePayloadHashes,    // 忽略负载 hash
    IgnorePayloadPath,      // 忽略负载路径
    OnlyUniqueID,           // 仅唯一 ID
    OnlyUniqueIDAndFeatureID
}
```

---

## 十、完整验证链总结

### 10.1 构建时验证 (imageapp)

```
OEMInput.xml + FM
    │
    ▼
Package.LoadFromCab()  [加载每个 .cab]
    ├─ DeviceSideManifest_Load_CBS()  [解析 CBS 清单]
    ├─ 提取文件列表 + 每个文件的 FileHash
    └─ 提取 Catalog 文件 (FileType.Catalog)
    │
    ▼
ValidateProductionImage()  [仅 Production 模式]
    ├─ 检查 ReleaseType=Production, BuildType=Retail
    └─ ImageSigner.HasSignature(catalog, isMicrosoft)
        ├─ X509Certificate2 加载证书
        ├─ X509Chain 构建证书链
        └─ 比对链中证书指纹 vs Microsoft 4个已知指纹
    │
    ▼
StageImage() → UpdateDLL.PrepareUpdate()
    └─ CBS 栈解压包，验证包签名，提取文件 hash
    │
    ▼
CommitImage() → UpdateDLL.ExecuteUpdate()
    └─ CBS 栈复制文件到分区，验证每个文件的 hash 与清单匹配
    │
    ▼
FinalizeImage()
    ├─ 生成 HashTable (每个 chunk 的 SHA256)
    ├─ ImageSigner.GenerateCatalogFile() → MakeCat.exe 生成 .cat
    ├─ ImageSigner.SignFFUImage()
    │   ├─ IsCatalogFile() 验证是合法 catalog
    │   ├─ HasSignature() 验证 catalog 有 Microsoft 签名
    │   └─ VerifyCatalogData() 验证 catalog hash 匹配 HashTable
    └─ 写入 FFU: OutputWrapper → SecurityWrapper → ManifestWrapper → Payload
```

### 10.2 刷写时验证 (设备端)

```
FFU 镜像
    │
    ▼
VerifyCatalog()
    ├─ VerifyCatalogData()  [Catalog SHA1 vs HashTable SHA1]
    └─ VerifyHashTable()    [每个 chunk SHA256 vs HashTable 条目]
    │
    ▼
应用到设备存储
```

---

## 十一、关键数据结构

### 11.1 CRYPTCATMEMBER (Catalog 成员)

```csharp
[StructLayout(LayoutKind.Sequential)]
public struct CRYPTCATMEMBER
{
    private uint cbStruct;
    [MarshalAs(UnmanagedType.LPWStr)] private string pwszReferenceTag;
    [MarshalAs(UnmanagedType.LPWStr)] private string pwszFileName;
    private Guid gSubjectType;
    private uint fdwMemberFlags;
    private IntPtr pIndirectData;       // ★ SPC_INDIRECT_DATA 指针
    private uint dwCertVersion;
    private uint dwReserved;
    private IntPtr hReserved;
    public CRYPT_ATTR_BLOB sEncodedIndirectData;  // ★ 编码的间接数据 (含 hash)
    private CRYPT_ATTR_BLOB sEncodedMemberInfo;
}
```

### 11.2 CRYPT_ATTR_BLOB

```csharp
public struct CRYPT_ATTR_BLOB
{
    public uint cbData;     // 数据大小
    public IntPtr pbData;   // 数据指针
}
```

### 11.3 WinTrust P/Invoke

```csharp
[DllImport("WinTrust.dll", CharSet = CharSet.Auto, SetLastError = true)]
private static extern IntPtr CryptCATOpen(string pwszFileName, uint fdwOpenFlags,
    IntPtr hProv, uint dwPublicVersion, uint dwEncodingType);

[DllImport("WinTrust.dll", CharSet = CharSet.Auto, SetLastError = true)]
private static extern bool CryptCATClose(IntPtr hCatalog);

[DllImport("WinTrust.dll", CharSet = CharSet.Auto, SetLastError = true)]
private static extern IntPtr CryptCATEnumerateMember(IntPtr hCatalog, IntPtr pPrevMember);

[DllImport("WinTrust.dll", CharSet = CharSet.Auto, SetLastError = true)]
public static extern bool IsCatalogFile(IntPtr hFile, string pwszFileName);
```

---

## 十二、重建项目关键要点

### 12.1 必须保留的组件

| 组件 | 类型 | 功能 | 可替代性 |
|------|------|------|----------|
| `UpdateDLL.dll` | Native | CBS 包加载、两阶段更新、文件 hash 验证 | 不可替代 |
| `wcp.dll` | Native | Windows Component Platform，CBS 服务核心 | 不可替代 |
| `cbscore.dll` | Native | CBS 核心，组件服务 | 不可替代 |
| `signinfohelper.dll` | Native | `IsSignInfoRequired` 函数 | 可替代（逻辑简单） |
| `WinTrust.dll` | System | Catalog 操作 (CryptCAT*) | 系统自带 |
| `MakeCat.exe` | Tool | Catalog 文件生成 | 可替代（需实现 CDF 解析） |
| `signer.cmd` / `sign.cmd` | Script | Authenticode 签名 | 可替代（用 signtool.exe） |

### 12.2 Hash 算法使用场景

| 算法 | 用途 | 位置 |
|------|------|------|
| **SHA256** | 文件内容 hash | CBS 清单 FileEntry.FileHash |
| **SHA256** | FFU chunk hash | FFU HashTable |
| **SHA256** | 包文件 hash | CompDB PayloadHash (Base64) |
| **SHA1** | HashTable 的 hash | FFU Catalog (IndirectData) |
| **SHA1** | 包文件辅助 hash | CompDB PayloadSha1Hash (Base64) |
| **SHA1/SHA2/Dual** | Authenticode 签名 | 包/Catalog 签名 |

### 12.3 证书验证关键逻辑

1. **Microsoft 包**：证书链必须包含 4 个已知 Microsoft 证书指纹之一
2. **OEM 包**：只需有任意有效 Authenticode 签名
3. **生产镜像**：所有包必须 ReleaseType=Production + BuildType=Retail
4. **Catalog 验证**：Catalog 的 SHA1 必须匹配 HashTable 的 SHA1
5. **HashTable 验证**：每个 chunk 的 SHA256 必须匹配 HashTable 对应条目

---

*报告基于 ADK 10.0.17704.1000 反编译内容生成*
*研究工具：ILSpy / PE 导出表分析 / IDA Pro 9.4 Hex-Rays*
