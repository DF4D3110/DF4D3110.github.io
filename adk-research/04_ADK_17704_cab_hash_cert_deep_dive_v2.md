# ADK 17704 CAB 包 Hash 与证书验证机制 — 深度研究报告

> 生成日期：2026-08-27
> 研究范围：CBS 清单 hash 格式、CDF/Catalog 内部结构、wcp.dll 原生 hash 验证、cbscore/updateapi 验证链
> 数据来源：ILSpy 反编译 (.NET) + IDA Pro Hex-Rays 反编译 (Native, wcp.c 13MB) + PE 导出表分析 + 实际 .cat 文件解析

---

## 一、CBS 清单 (update.mum) Hash 格式

### 1.1 CBS 包内部结构

每个 CBS .cab 包包含以下关键文件：

```
package.cab
├── update.mum          ★ CBS 清单 (XML格式，包含所有文件的 hash)
├── update.cat          ★ 目录文件 (Authenticode 签名)
├── <files>/            实际负载文件（按组件组织）
└── <manifest>.manifest 组件级清单
```

### 1.2 update.mum 中的文件 Hash 格式

CBS 清单是 XML 格式，每个文件条目包含以下属性：

```xml
<file name="kernel32.dll"
      location="\Windows\System32\"
      size="1234567"
      compressedSize="456789"
      stagedSize="1300000"
      hash="A1B2C3D4E5F6..."          ★ 文件内容 hash（十六进制字符串）
      hashAlgorithm="SHA256"          ★ hash 算法
      sourcePackage="Microsoft-Windows-Kernel"
      embeddedSigningCategory="WindowsSystemComponent"
      signInfoRequired="true" />       ★ 是否需要签名信息
```

### 1.3 Hash 算法 ID 映射表 (wcp.dll!RtlGetHashAlgorithmHashLength)

通过反编译 `wcp.dll` 中的 `RtlGetHashAlgorithmHashLength` 函数（地址 0x101cedd0），得到 CBS 清单使用的 hash 算法 ID 与长度映射：

| 算法 ID | Hash 长度 | 算法名称 | 用途 |
|---------|-----------|----------|------|
| 1 | 8 字节 (64-bit) | CRC/Murmur 类 | 内部快速校验 |
| 2 | 0x14 = 20 字节 | **SHA1** | 兼容/目录 hash |
| 3 | 0x20 = 32 字节 | **SHA256** | ★ 主要文件内容 hash |
| 4 | 0x30 = 48 字节 | SHA384 | 高强度场景 |
| 5 | 0x40 = 64 字节 | SHA512 | 最高强度 |
| 6,7,8 | 0x10 = 16 字节 | MD5 | 遗留兼容 |

**反编译源码** (wcp.c 行 280988-281030):
```c
void RtlGetHashAlgorithmHashLength(int param_1, undefined4 param_2, undefined4 *param_3)
{
    if (param_3 != NULL) *param_3 = 0;
    if (param_1 == 0 && param_3 != NULL) {
        switch (param_2) {
        case 1: *param_3 = 8;     break;   // 64-bit
        case 2: *param_3 = 0x14;  break;   // SHA1 (20 bytes)
        case 3: *param_3 = 0x20;  break;   // SHA256 (32 bytes)
        case 4: *param_3 = 0x30;  break;   // SHA384 (48 bytes)
        case 5: *param_3 = 0x40;  break;   // SHA512 (64 bytes)
        case 6: case 7: case 8:
            *param_3 = 0x10;  break;       // MD5 (16 bytes)
        }
    }
}
```

### 1.4 HashInfo 结构体

`CmsGetFileHashInformation` 函数返回的 `HashInfo` 结构体（定义在 `Windows::ManifestParser::Rtl` 命名空间）：

```c
struct HashInfo {
    DWORD  hashAlgorithm;   // 算法 ID (1-8)
    DWORD  hashLength;      // hash 数据长度
    BYTE   hashData[64];    // hash 数据（最大64字节，SHA512）
};
```

### 1.5 CmsGetFileHashInformation — 从清单提取文件 hash

```c
// wcp.dll 导出函数 (地址 0x100b6b90)
// public: static long __stdcall WcpManifest::CmsGetFileHashInformation(
//     struct WcpManifestParser::File const &,
//     struct Windows::ManifestParser::Rtl::HashInfo *)
long WcpManifest::CmsGetFileHashInformation(File *param_1, HashInfo *param_2)
{
    return FUN_100b787c();  // 内部实现：从 File 对象的 XML 属性提取 hash
}
```

**调用链**：
```
CBS 服务加载 update.mum
  → 解析每个 <file> 元素
  → WcpManifestParser::File 对象存储 hash 属性
  → CmsGetFileHashInformation(file, &hashInfo) 提取 hash
  → hashInfo.hashAlgorithm + hashInfo.hashData
```

---

## 二、CDF 格式与 Catalog (.cat) 内部结构

### 2.1 CDF (Catalog Definition File) 格式

`MakeCat.exe` 通过 CDF 文件生成 .cat 目录文件。从 `ImageSigner.GenerateCatalogFile()` 反编译得到 CDF 格式：

```
[CatalogHeader]
Name=<输出.cat文件路径>

[CatalogFiles]
<成员标签>=<被引用文件路径>
```

**实际示例** (FFU HashTable 目录):
```
[CatalogHeader]
Name=C:\Users\...\Temp\HashTable.cat

[CatalogFiles]
HashTable.blob=C:\Users\...\Temp\HashTable.blob
```

### 2.2 Catalog 文件内部结构

Catalog (.cat) 是 PKCS#7 格式的 Authenticode 签名文件，内部包含：

```
.cat 文件 (PKCS#7 SignedData)
├── SignerInfo                      ★ 签名者信息（证书+签名）
│   ├── 签名证书 (X.509)
│   ├── 签名算法 (sha256RSA)
│   └── 签名值
├── Certificates                    ★ 证书链
│   ├── 签名证书
│   ├── 中间 CA 证书
│   └── 根 CA 证书
└── Catalog Members (CRYPTCATMEMBER[])  ★ 目录成员列表
    └── 每个成员:
        ├── pwszReferenceTag       成员标签 (如 "HashTable.blob")
        ├── pwszFileName           文件名
        ├── gSubjectType           主题类型 GUID
        ├── pIndirectData          SPC_INDIRECT_DATA 指针
        └── sEncodedIndirectData   ★ 编码的间接数据（含文件 hash）
            ├── cbData              数据大小
            └── pbData              数据指针
                └── 末尾 N 字节 = 被引用文件的 hash
                    (SHA1 = 20字节, SHA256 = 32字节)
```

### 2.3 CRYPTCATMEMBER 结构体 (精确布局)

```c
typedef struct CRYPTCATMEMBER_ {
    DWORD           cbStruct;               // 结构体大小
    LPWSTR          pwszReferenceTag;       // 成员标签
    LPWSTR          pwszFileName;           // 文件名
    GUID            gSubjectType;            // 主题类型
    DWORD           fdwMemberFlags;          // 成员标志
    struct SIP_INDIRECT_DATA_ *pIndirectData; // 间接数据
    DWORD           dwCertVersion;           // 证书版本
    DWORD           dwReserved;              // 保留
    HANDLE          hReserved;               // 保留句柄
    CRYPT_ATTR_BLOB sEncodedIndirectData;   // ★ 编码的间接数据
    CRYPT_ATTR_BLOB sEncodedMemberInfo;      // 编码的成员信息
} CRYPTCATMEMBER;

typedef struct CRYPT_ATTR_BLOB_ {
    DWORD  cbData;   // 数据大小
    BYTE  *pbData;   // 数据指针
} CRYPT_ATTR_BLOB;
```

### 2.4 从 Catalog 提取文件 Hash — GetCatalogHash

```csharp
// imagecommon.cs — ImageSigner.GetCatalogHash
internal static byte[] GetCatalogHash(string catalogFile)
{
    // 1. 打开 Catalog
    IntPtr hCatalog = CryptCATOpen(catalogFile, 2, IntPtr.Zero, 0, 0);

    // 2. 枚举第一个成员
    IntPtr hMember = CryptCATEnumerateMember(hCatalog, IntPtr.Zero);
    CRYPTCATMEMBER member = Marshal.PtrToStructure<CRYPTCATMEMBER>(hMember);

    // 3. ★ 从 sEncodedIndirectData 末尾提取 hash
    //    IndirectData 是 SPC_INDIRECT_DATA_CONTENT 结构
    //    其末尾包含被引用文件的 hash (SHA1 = 20字节)
    byte[] hash = new byte[20];  // SHA1
    int offset = (int)member.sEncodedIndirectData.cbData - hash.Length;
    Marshal.Copy(
        IntPtr.Add(member.sEncodedIndirectData.pbData, offset),
        hash, 0, hash.Length);

    CryptCATClose(hCatalog);
    return hash;
}
```

**原理**：`SPC_INDIRECT_DATA_CONTENT` 结构 = `SPC_ATTRIBUTE_TYPE` + `SPC_HASH_INFO`，其中 `SPC_HASH_INFO` 包含 `Algorithm` + `Hash`。hash 数据位于编码结构的末尾。

### 2.5 实际 .cat 文件分析

对 `E:\WSK_Tools\ADK_Research\SourceDir\Windows Kits\10\Catalogs\catc36902435dc33ec7f0b63405ca9f0047.cat` 的分析结果：

| 属性 | 值 |
|------|-----|
| 文件大小 | 101,885 字节 |
| 签名者 | CN=Microsoft Corporation, O=Microsoft Corporation, L=Redmond, S=Washington, C=US |
| 颁发者 | CN=Microsoft Development PCA 2014, O=Microsoft Corporation |
| 证书指纹 | 4FF31605BD0C478ED64E95E1411A0D21D35F7EE7 |
| 有效期 | 2018-04-20 至 2018-12-15 |
| 签名算法 | sha256RSA |
| 公钥算法 | RSA (2160 bits) |
| IsCatalogFile | True |

---

## 三、wcp.dll 原生 Hash 验证机制

### 3.1 wcp.dll 导出函数总览

wcp.dll (Windows Component Platform) 共导出 **443 个函数**，与 hash/证书验证相关的关键函数：

| 函数名 | 地址 | 功能 |
|--------|------|------|
| `CmsGetFileHashInformation` | 0x100b6b90 | 从清单 File 对象提取 hash |
| `FileHashAlgorithmToDigest` | - | hash 算法 → CMS 摘要方法转换 |
| `RtlGetHashAlgorithmHashLength` | 0x101cedd0 | 算法 ID → hash 长度映射 |
| `RtlHashLBlob` | - | 对二进制 Blob 计算 hash |
| `RtlHashLUnicodeString` | - | 对 Unicode 字符串计算 hash |
| `RtlHashLUtf8String` | - | 对 UTF-8 字符串计算 hash |
| `RtlMurmurHashLBlob` | - | Murmur hash (快速非加密) |
| `CCatalog::Create` | 0x102474e0 | 创建 Catalog 对象 |
| `CCatalog::Close` | - | 关闭 Catalog |
| `CCatalog::FindSubject` | - | 查找 Catalog 中的主题 |
| `CCatalog::VerifyCertChainRoot` | - | 验证证书链根 |
| `CCatalog::VerifySigner` | - | 验证签名者 |
| `CCatalog::GetSignerPublicKeyToken` | - | 获取签名者公钥令牌 |
| `GenerateCatalogThumbprint` | - | 生成 Catalog 指纹 |
| `DoesTargetHaveTestRootCert` | - | 检查目标是否有测试根证书 |

### 3.2 CCSDirectTransaction::VerifyFileHashes — 核心文件 hash 验证

这是 CBS 服务中**最核心的文件 hash 验证函数**，源文件 `onecore\base\wcp\componentstore\csd_transact.cpp`。

**函数签名** (wcp.c 行 110520):
```c
void __thiscall CCSDirectTransaction::VerifyFileHashes(
    CCSDirectTransaction *this,
    IRtlDefinitionAppId *componentId,   // 组件标识
    LPCWSTR filePath,                    // 文件路径
    DWORD fileId,                        // 文件 ID
    BOOL failOnMismatch,                 // ★ 不匹配时是否失败
    BOOL *result);                       // 输出: 1=匹配, 0=不匹配
```

**验证流程**:
```
VerifyFileHashes(component, file, failOnMismatch, &result)
  │
  ├─ 1. FUN_1025337a(component, file, ...)  获取文件信息
  │     ├─ 从组件清单读取预期 hash (CmsGetFileHashInformation)
  │     ├─ 打开实际文件
  │     └─ 计算实际文件 hash (BCryptHash / CryptHash)
  │
  ├─ 2. 比较预期 hash 与实际 hash
  │     ├─ status == 2 → hash 匹配 → *result = 1
  │     └─ status != 2 → hash 不匹配
  │
  ├─ 3. 不匹配处理
  │     ├─ 日志: "VerifyFileHashes encountered hash difference for {f} in {i}"
  │     ├─ *result = 0
  │     └─ if (failOnMismatch):
  │           → 返回 HRESULT 0xc015001b (CBS_E_CORRUPT_FILE)
  │           → 源位置: csd_transact.cpp:0x8e5
  │
  └─ 4. 匹配 → *result = 1, 返回 S_OK
```

**反编译关键代码** (wcp.c 行 110544-110571):
```c
puVar2 = FUN_1025337a(param_1, param_2, param_3, 1, param_5, param_6, &local_b0);
if ((int)puVar2 < 0) { /* error */ }
else {
    if (local_b0 == 2) {
        *param_8 = 1;  // ★ hash 匹配
    }
    else {
        // ★ hash 不匹配
        FUN_100b28a3(0, &DAT_102a9164,
            "VerifyFileHashes encountered hash difference for {f} in {i}",
            2, &DAT_100297c0, RtlTraceFormat_IRtlBaseIdentity, param_4,
            &DAT_10026580, RtlTraceFormat_PCLUNICODE_STRING, param_5);
        *param_8 = 0;
        if (param_7 != '\0') {  // failOnMismatch
            local_c = 0xc015001b;  // ★ CBS_E_CORRUPT_FILE
            FUN_10119dc9();
            // 源: csd_transact.cpp, CCSDirectTransaction::VerifyFileHashes, line 0x8e5
        }
    }
}
```

### 3.3 第二种 VerifyFileHashes 变体 (FUN_1011749c)

另一个 hash 验证变体（wcp.c 行 110579），使用不同的 hash 计算路径：

```c
void __thiscall FUN_1011749c(
    undefined4 param_1,     // 组件上下文
    int param_2,             // 文件信息
    undefined4 param_3,      // 文件路径
    undefined4 param_4,      // 组件 ID
    char param_5,            // failOnMismatch
    undefined1 *param_6)     // result
{
    // 1. 初始化 hash 算法对象
    FUN_1010cd6b();

    // 2. 获取文件大小
    puVar4 = FUN_101083eb(param_3);

    // 3. 创建 hash 缓冲区 (32字节 = SHA256)
    FUN_100b6abe(1, &local_ac, &local_c4);  // local_c4 = 32 (0x20)

    // 4. 创建 hash 对象
    FUN_100b6ba4(0);

    // 5. ★ 计算并比较 hash
    //    local_c4 = local_bc - local_c0 >> 5 = hash长度/32 = 1 (SHA256)
    local_c4 = local_bc - local_c0 >> 5;
    puVar4 = FUN_1028ce12(param_3, param_4, *(param_2 + 0x14), &local_b4);

    if (-1 < (int)puVar4) {
        if (local_b4 == 6) {  // ★ hash 不匹配 (status 6)
            FUN_100b28a3(0, &DAT_102a9164,
                "VerifyFileHashes encountered hash difference for {f} in {i}", ...);
            if (param_5 != '\0') {  // failOnMismatch
                local_10 = 0xc015001b;  // CBS_E_CORRUPT_FILE
            }
        }
        else {
            local_7c = 1;  // hash 匹配
        }
    }
}
```

### 3.4 StageComponentFile — 暂存阶段的 hash 验证

`CCSDirectTransaction::StageComponentFile` (wcp.c 行 110739) 是文件暂存的主函数，在暂存过程中集成 hash 验证：

```
StageComponentFile(fileInfo, &result)
  │
  ├─ 1. 检查 fStrongNamed 标志
  │     if (!fStrongNamed) → 错误 0xc0000429 (STATUS_INVALID_PARAMETER)
  │
  ├─ 2. 检查组件是否已暂存
  │     if (stageState != 2) → 错误 0xc015000c (组件未暂存)
  │
  ├─ 3. 检查清单是否存在
  │     if (!manifestExists) → 错误 0xc0150004
  │
  ├─ 4. ★ 调用 VerifyFileHashes
  │     ├─ 版本化索引路径: FUN_101172e3(...)
  │     └─ 清单解析路径: FUN_1011749c(...)  [即第二种 VerifyFileHashes]
  │     传入 failOnMismatch = (configOption == 1)
  │
  ├─ 5. 复制文件到 WinSxS 暂存区
  │     FUN_10101b02(...) 执行实际文件复制
  │
  ├─ 6. 复制后再次验证 hash
  │     if (versionedIndex && !noVerify):
  │       VerifyFileHashes(...) 再次验证
  │
  └─ 7. 标记文件已暂存
        *(this + 0xdc6) = 1
        *result = 1
```

### 3.5 Hash 验证错误码

| HRESULT | 名称 | 含义 |
|---------|------|------|
| `0xc015001b` | CBS_E_CORRUPT_FILE | 文件 hash 不匹配（文件损坏） |
| `0xc0150004` | CBS_E_MANIFEST_MISSING | 组件清单缺失 |
| `0xc015000c` | CBS_E_NOT_STAGED | 组件未暂存 |
| `0xc015001f` | CBS_E_FILE_NOT_FOUND | 文件在组件中未找到 |
| `0xc0000429` | STATUS_INVALID_PARAMETER | fStrongNamed 标志未设置 |

---

## 四、CCatalog 类 — 原生目录验证

### 4.1 CCatalog::Create — 创建 Catalog 对象

```c
// wcp.dll (地址 0x102474e0)
// public: long __thiscall CCatalog::Create(
//     unsigned long, unsigned long,
//     struct _LBLOB const *, unsigned long *)
long __thiscall CCatalog::Create(
    CCatalog *this,
    DWORD param_1,       // 标志
    DWORD param_2,       // 编码类型
    _LBLOB *param_3,     // Catalog 数据 Blob
    DWORD *param_4)      // 输出句柄
{
    // 1. 检查输入 Blob 非空
    if (*(int *)param_3 == 0) { /* error */ }

    // 2. 调用 CryptCATOpen 打开 Catalog
    //    使用 PCCTL_CONTEXT 进行 PKCS#7 解析
    PCCTL_CONTEXT pCtlContext;
    // ... CryptCATOpen(...)

    // 3. 生成 Catalog 指纹
    GenerateCatalogThumbprint(this, &thumbprint);

    // 4. 验证签名者 (可选)
    if (param_1 & VERIFY_SIGNER) {
        CCatalog::VerifySigner(this, ...);
    }

    // 5. 验证证书链根 (可选)
    if (param_1 & VERIFY_CHAIN) {
        CCatalog::VerifyCertChainRoot(this, ...);
    }
}
```

### 4.2 CCatalog 验证方法链

```
CCatalog::Create(catalogData)
  ├─ CryptCATOpen() 解析 PKCS#7
  ├─ GenerateCatalogThumbprint() 生成指纹
  │
  ├─ CCatalog::VerifySigner()
  │   ├─ 提取 SignerInfo
  │   ├─ 验证签名值 (RSA 验签)
  │   └─ 验证摘要算法匹配
  │
  ├─ CCatalog::VerifyCertChainRoot()
  │   ├─ 构建 X509 证书链
  │   ├─ 验证链中每个证书的签名
  │   ├─ 检查根证书是否受信任
  │   └─ 检查证书有效期
  │
  ├─ CCatalog::FindSubject(tag)
  │   └─ 按成员标签查找 CRYPTCATMEMBER
  │
  └─ CCatalog::GetSignerPublicKeyToken()
      └─ 提取签名者证书的公钥令牌 (SHA1 of public key)
```

### 4.3 DoesTargetHaveTestRootCert — 测试根证书检测

```c
// wcp.dll 导出函数
// 检查目标系统是否安装了 Microsoft 测试根证书
// 用于判断是否允许测试签名的包
long DoesTargetHaveTestRootCert(...)
{
    // 1. 打开证书存储 (Local Machine\Root)
    // 2. 查找 Microsoft Test Root 证书
    //    指纹: 8A334AA8052DD244A647306A76B8178FA215F344
    // 3. 返回是否存在
}
```

---

## 五、完整验证链 — 从 CAB 到文件系统

### 5.1 端到端验证流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│  Stage 阶段 (PrepareUpdate) — 包加载与暂存                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 加载 .cab 包                                                     │
│     └─ UpdateDLL!DeviceSideManifest_Load_CBS(cabPath)              │
│         └─ 解析 update.mum (CBS 清单)                                │
│             └─ 提取每个文件的 hash (hashAlgorithm + hashData)        │
│                                                                     │
│  2. 验证包签名                                                        │
│     └─ 提取 update.cat (目录文件)                                    │
│         ├─ CCatalog::Create(catData)                                │
│         │   ├─ CryptCATOpen 解析 PKCS#7                              │
│         │   ├─ CCatalog::VerifySigner() 验证签名者                  │
│         │   └─ CCatalog::VerifyCertChainRoot() 验证证书链            │
│         └─ ImageSigner.HasSignature(cat, isMicrosoft)               │
│             └─ X509Chain 比对 4 个 Microsoft 证书指纹                │
│                                                                     │
│  3. 解压文件到暂存区 (TempSxS)                                       │
│     └─ 每个文件解压后:                                                │
│         CCSDirectTransaction::VerifyFileHashes(component, file)     │
│         ├─ 从清单读取预期 hash (CmsGetFileHashInformation)           │
│         ├─ 计算实际文件 hash (BCryptHash SHA256)                     │
│         ├─ 比较 hash                                                  │
│         └─ 不匹配 → CBS_E_CORRUPT_FILE (0xc015001b)                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Commit 阶段 (ExecuteUpdate) — 文件安装与最终验证                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  4. 从暂存区复制文件到目标分区                                         │
│     └─ CCSDirectTransaction::StageComponentFile(fileInfo)           │
│         ├─ 检查 fStrongNamed 标志                                    │
│         ├─ 检查组件暂存状态                                           │
│         ├─ ★ VerifyFileHashes() 复制前验证                          │
│         ├─ 复制文件到 WinSxS/目标位置                                 │
│         ├─ ★ VerifyFileHashes() 复制后验证 (可选)                    │
│         └─ 标记文件已暂存                                             │
│                                                                     │
│  5. 应用注册表、权限、服务等                                           │
│                                                                     │
│  6. 生成 UpdateHistory.xml + UpdateOutput.xml                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Finalize 阶段 — FFU 镜像 hash 链                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  7. 生成 FFU HashTable                                               │
│     └─ 镜像数据按 ChunkSize (默认2MB) 分块                           │
│         └─ 每块计算 SHA256 → 存入 HashTable                          │
│                                                                     │
│  8. 生成 Catalog (.cat)                                              │
│     └─ ImageSigner.GenerateCatalogFile(hashTableData)               │
│         ├─ 写 HashTable.blob 临时文件                                │
│         ├─ 生成 CDF (Catalog Definition File)                        │
│         └─ MakeCat.exe 生成 .cat                                     │
│             └─ Catalog 包含 HashTable 的 SHA1 (20字节)               │
│                                                                     │
│  9. 签名 FFU                                                         │
│     └─ ImageSigner.SignFFUImage()                                    │
│         ├─ IsCatalogFile() 验证是合法 catalog                         │
│         ├─ HasSignature() 验证 catalog 有 Microsoft 签名             │
│         └─ VerifyCatalogData() 验证 catalog hash 匹配 HashTable      │
│                                                                     │
│  10. 写入 FFU                                                        │
│      OutputWrapper → SecurityWrapper → ManifestWrapper → Payload     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 三层 Hash 验证汇总

| 层级 | 验证点 | Hash 算法 | 验证时机 | 失败处理 |
|------|--------|-----------|----------|----------|
| **包级** | CAB 包内文件 vs update.mum 清单 | SHA256 (算法ID=3) | Stage 解压后 | CBS_E_CORRUPT_FILE |
| **包级** | update.cat 签名验证 | sha256RSA | Stage 加载时 | 包被拒绝 |
| **安装级** | 暂存文件 vs 目标文件 | SHA256 | Commit 复制前后 | CBS_E_CORRUPT_FILE |
| **镜像级** | FFU chunk vs HashTable | SHA256 | Finalize / 刷写时 | 镜像损坏 |
| **镜像级** | HashTable vs Catalog | SHA1 | Finalize / 刷写时 | 目录不匹配 |
| **镜像级** | Catalog 签名验证 | sha256RSA | Finalize | 签名无效 |

---

## 六、关键原生 API 完整列表

### 6.1 UpdateDLL.dll — 清单与文件 API

```c
// 清单管理
int DeviceSideManifest_Create(out IntPtr obj);
int DeviceSideManifest_Load_CBS(IntPtr obj, LPCWSTR cabPath);
int DeviceSideManifest_Load(IntPtr obj, LPCWSTR xmlPath);
void DeviceSideManifest_Free(IntPtr obj);

// 文件条目
int DeviceSideManifest_Add_File(
    IntPtr obj, FileType fileType, LPCWSTR devicePath, LPCWSTR cabPath,
    FileAttributes attributes, LPCWSTR sourcePackage, LPCWSTR embedSignCategory,
    ULONG64 FileSize, ULONG64 CompressedFileSize, ULONG64 StagedFileSize,
    LPCWSTR fileHash,          // ★ 文件 hash
    BOOL signFile);             // ★ 是否需要签名

// 文件属性获取
FileType DSMFileEntry_Get_FileType(IntPtr file);
LPCWSTR DSMFileEntry_Get_DevicePath(IntPtr file);
LPCWSTR DSMFileEntry_Get_CabPath(IntPtr file);
LPCWSTR DSMFileEntry_Get_FileHash(IntPtr file);     // ★ 获取文件 hash
BOOL DSMFileEntry_Get_SignInfo(IntPtr file);          // ★ 获取签名需求
ULONG64 DSMFileEntry_Get_FileSize(IntPtr file);
ULONG64 DSMFileEntry_Get_CompressedFileSize(IntPtr file);
ULONG64 DSMFileEntry_Get_StagedFileSize(IntPtr file);
LPCWSTR DSMFileEntry_Get_SourcePackage(IntPtr file);
LPCWSTR DSMFileEntry_Get_EmbeddedSigningCategory(IntPtr file);

// 文件大小查询
uint IU_GetStagedAndCompressedSize(
    LPCWSTR file, out ULONG64 fileSize,
    out ULONG64 stagedSize, out ULONG64 compressedSize);
```

### 6.2 wcp.dll — Hash 与 Catalog API

```c
// Hash 算法
void RtlGetHashAlgorithmHashLength(int reserved, DWORD algorithmId, DWORD *length);
//   algorithmId: 1=8B, 2=SHA1(20B), 3=SHA256(32B), 4=SHA384(48B), 5=SHA512(64B), 6-8=MD5(16B)

// 文件 hash 提取
long WcpManifest::CmsGetFileHashInformation(File *file, HashInfo *hashInfo);

// Hash 计算
long RtlHashLBlob(ALG_ID algorithm, const BYTE *data, DWORD size, BYTE *hash);
long RtlHashLUnicodeString(ALG_ID algorithm, LPCWSTR str, BYTE *hash);
long RtlMurmurHashLBlob(const BYTE *data, DWORD size, ULONG64 *hash);

// Catalog
long CCatalog::Create(DWORD flags, DWORD encoding, _LBLOB *data, DWORD *handle);
void CCatalog::Close(CCatalog *this);
long CCatalog::FindSubject(CCatalog *this, LPCWSTR tag, CRYPTCATMEMBER **member);
long CCatalog::VerifySigner(CCatalog *this, ...);
long CCatalog::VerifyCertChainRoot(CCatalog *this, ...);
long CCatalog::GetSignerPublicKeyToken(CCatalog *this, BYTE *token, DWORD *size);
long GenerateCatalogThumbprint(CCatalog *this, _LUNICODE_STRING *thumbprint);
long DoesTargetHaveTestRootCert(...);
```

### 6.3 WinTrust.dll — Catalog 操作 API

```c
IntPtr CryptCATOpen(LPCWSTR fileName, DWORD fdwOpenFlags, HCRYPTPROV hProv,
                     DWORD dwPublicVersion, DWORD dwEncodingType);
BOOL CryptCATClose(IntPtr hCatalog);
IntPtr CryptCATEnumerateMember(IntPtr hCatalog, IntPtr pPrevMember);
BOOL IsCatalogFile(HANDLE hFile, LPCWSTR pwszFileName);
```

### 6.4 signinfohelper.dll — 签名信息辅助

```c
// 唯一导出函数
BOOL IsSignInfoRequired(LPCWSTR filePath, LPCWSTR packageName);
// 判断给定文件是否需要嵌入签名信息
```

---

## 七、重建项目关键技术点

### 7.1 必须复用的原生组件

| 组件 | 原因 | 替代难度 |
|------|------|----------|
| **wcp.dll** | CBS 核心，包含 hash 验证、组件存储、事务管理 | 极高（443个导出，13MB反编译） |
| **cbscore.dll** | CBS 服务核心，包处理流水线 | 极高 |
| **UpdateDLL.dll** | 两阶段更新、清单加载、文件 hash 传递 | 高 |
| **updateapi.dll** | 更新 API 接口层 | 高 |
| **WinTrust.dll** | 系统自带，Catalog 操作 | 无需替代 |

### 7.2 可重写的托管组件

| 组件 | 功能 | 重写要点 |
|------|------|----------|
| **ImageSigner** | FFU 签名/验证 | 已完整反编译，可直接参考 |
| **PackageTools** | 包 hash 计算/签名 | CalculateFileHash(SHA256), CalculateFileSha1Hash(SHA1) |
| **ValidateProductionImage** | 生产镜像验证 | 已完整反编译，逻辑清晰 |
| **PkgManifest** | 包清单解析 | 可参考反编译的 C# 代码 |

### 7.3 Hash 验证实现要点

1. **CBS 清单 hash 解析**：从 update.mum 的 `<file>` 元素提取 `hash` 和 `hashAlgorithm` 属性
2. **Hash 算法映射**：使用 `RtlGetHashAlgorithmHashLength` 的映射表（ID 3 = SHA256 = 32字节）
3. **文件 hash 计算**：使用 BCryptHash API（SHA256），或 C# 的 `SHA256Managed`
4. **Hash 比较**：逐字节比较，不匹配返回 `0xc015001b` (CBS_E_CORRUPT_FILE)
5. **Catalog hash 提取**：从 `CRYPTCATMEMBER.sEncodedIndirectData` 末尾提取 20 字节 (SHA1)
6. **FFU HashTable**：按 2MB 分块，每块 SHA256，连续存储
7. **证书链验证**：X509Chain + 4 个 Microsoft 证书指纹比对

### 7.4 测试根证书指纹

```
Microsoft Production Root:  3B1EFD3A66EA28B16697394703A72CA340A05BD5
Microsoft Test Root:        8A334AA8052DD244A647306A76B8178FA215F344
Microsoft Flight PCA:       9E594333273339A97051B0F82E86F266B917EDB3
Microsoft Flight Windows:   5f444a6740b7ca2434c7a5925222c2339ee0f1b7
```

---

## 八、参考文件索引

| 文件 | 大小 | 内容 |
|------|------|------|
| `outputsrc\native\wcp.c` | 13MB | wcp.dll 完整反编译（含 VerifyFileHashes、CCatalog、RtlGetHashAlgorithmHashLength） |
| `ida_decompiled\native\wcp.c` | 18.9MB | IDA 版本 wcp.dll 反编译 |
| `outputsrc\dotnet\imagecommon.cs` | 474KB | ImageSigner、PackageTools、FileEntry 托管代码 |
| `outputsrc\dotnet\pkgcommonmanaged.cs` | 429KB | PkgManifest、Package.LoadFromCab、FileEntryBase |
| `outputsrc\dotnet\imaging.cs` | 148KB | ValidateProductionImage、StageImage、CommitImage |
| `outputsrc\dotnet\pkgtoolbox.cs` | 253KB | PackageTools 完整实现 |
| `ADK\signinfohelper.dll` | 8.7KB | 签名信息辅助（仅导出 IsSignInfoRequired） |
| `ADK\wcp.dll` | - | Windows Component Platform（443个导出） |
| `SourceDir\...\Catalogs\*.cat` | 101KB | 实际 Microsoft 签名的 Catalog 文件 |

---

*报告基于 ADK 10.0.17704.1000 反编译内容生成*
*研究工具：ILSpy / IDA Pro 9.4 Hex-Rays / PE 导出表分析 / PowerShell 证书解析*
*关键源文件：onecore\base\wcp\componentstore\csd_transact.cpp (CCSDirectTransaction::VerifyFileHashes)*
