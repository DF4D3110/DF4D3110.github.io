# ADK 10.0.17704.1000 — wcp.dll 完整深度逆向分析 (IDA 18MB 版)

> 文档版本: 2.0 (基于 IDA Pro 9.4 完整反编译，18,912,503 字节 / ~540,000 行)
> 日期: 2026-08-27
> 替代: 本文档 supersedes 之前基于 12.5MB 版的部分分析，补充了 CCatalog 类完整实现、证书链验证、hash 验证完整调用链
> 反编译源: `E:\WSK_Tools\ADK_Research\ida_decompiled\native\wcp.c`

---

## 目录

1. [执行摘要](#1-执行摘要)
2. [wcp.dll 架构总览](#2-wcpdll-架构总览)
3. [CCatalog 类完整实现](#3-ccatalog-类完整实现)
4. [文件 Hash 验证完整调用链](#4-文件-hash-验证完整调用链)
5. [组件签名验证完整调用链](#5-组件签名验证完整调用链)
6. [Manifest vs Catalog 验证](#6-manifest-vs-catalog-验证)
7. [关键常量与 OID](#7-关键常量与-oid)
8. [错误码体系](#8-错误码体系)
9. [源码行号索引](#9-源码行号索引)
10. [重建项目 implications](#10-重建项目-implications)

---

## 1. 执行摘要

### 核心发现

wcp.dll (Windows Component Platform) 是 ADK 镜像构建与 CBS 包安装的核心引擎，IDA 完整反编译达 **18MB / ~540,000 行**。本文档基于 IDA 9.4 完整版重新深入分析，揭示了之前 12.5MB 版中缺失的关键实现：

1. **CCatalog 类** — 安全目录操作的完整 Win32 CryptoAPI 封装，5 个核心方法全部逆向
2. **证书链验证** — `CCatalog::VerifyCertChainRoot` 的完整实现，使用 `CertGetCertificateChain` + `CertVerifyCertificateChainPolicy` (Microsoft Root Policy)
3. **Hash 验证调用链** — 从 `CCSDirectTransaction::VerifyFileHashes` 到底层 `Windows::Cms::Rtl::ValidateFileHash` 的 4 层调用链
4. **PublicKeyToken** — 8 字节 SHA1(公钥)，用于组件身份与 catalog 签名者匹配
5. **Catalog Subject Algorithm** — 通过 OID 识别 SHA1 vs SHA256

### 与之前分析的关键差异

| 方面 | 之前 (12.5MB 版) | 现在 (18MB IDA 版) |
|------|-------------------|---------------------|
| CCatalog::Create | 仅知道调用 CertCreateCTLContext | 完整实现 + OID 识别 + 错误处理 |
| CCatalog::VerifySigner | 未知 | CertOpenStore + CryptMsgGetAndVerifySigner |
| CCatalog::VerifyCertChainRoot | 未知 | 完整证书链构建 + Microsoft Root Policy 验证 |
| CCatalog::FindSubject | 仅知道函数名 | CertFindSubjectInCTL 完整封装 |
| CCatalog::GetSignerPublicKeyToken | 未知 | CryptImportKey + SHA1 公钥哈希 |
| CmsVerifyFileHash | 未知 | 完整 flags 解码 + 返回值映射 |
| CmsVerifyFileHashes | 未知 | manifest 多 hash 验证 + 备用 hash 回退 |
| 源码文件 | 部分 | 完整: catalog.cpp, hashverify.cpp, csd_transact.cpp, csd_winners.cpp, corruptionrepair.cpp |

---

## 2. wcp.dll 架构总览

### 2.1 模块分层

```
┌─────────────────────────────────────────────────────────┐
│ CCSDirectTransaction (CBS 直接事务层)                    │
│  - VerifyFileHashes (2个重载)                            │
│  - ValidateComponentSignature                            │
│  - AddImplicationsToCatalogsAndVerifyComponentHashes     │
│  - AddCatalog                                             │
│  源码: csd_transact.cpp, csd_winners.cpp                │
├─────────────────────────────────────────────────────────┤
│ ManifestParser (Manifest 解析层)                          │
│  - CmsVerifyFileHash                                      │
│  - CmsVerifyFileHashes                                    │
│  - CmsFindMetadata                                        │
│  - CmsGetManifestIdentity                                 │
│  源码: hashverify.cpp                                     │
├─────────────────────────────────────────────────────────┤
│ Cms (CMS 加密消息层)                                      │
│  - Windows::Cms::Rtl::ValidateFileHash                   │
│  - FileHashAlgorithmToDigest                              │
│  源码: (内联/静态库)                                      │
├─────────────────────────────────────────────────────────┤
│ CCatalog (安全目录层)                                     │
│  - Create / Close                                         │
│  - VerifySigner                                           │
│  - VerifyCertChainRoot                                    │
│  - FindSubject                                            │
│  - GetSignerPublicKeyToken                                │
│  - GetTimeStamperInfo                                     │
│  源码: rtllib\win32lib\catalog.cpp                       │
├─────────────────────────────────────────────────────────┤
│ Win32 CryptoAPI (系统层)                                  │
│  CertCreateCTLContext / CertFindSubjectInCTL             │
│  CryptMsgGetAndVerifySigner                               │
│  CertOpenStore / CertAddStoreToCollection                │
│  CertGetCertificateChain / CertVerifyCertificateChainPolicy│
│  CryptAcquireContextW / CryptImportKey / CryptHashData   │
└─────────────────────────────────────────────────────────┘
```

### 2.2 核心类清单

| 类 | 职责 | 关键方法 |
|----|------|---------|
| `CCatalog` | 安全目录 (.cat) 操作封装 | Create, Close, VerifySigner, VerifyCertChainRoot, FindSubject, GetSignerPublicKeyToken |
| `CCSDirectTransaction` | CBS 直接事务 | VerifyFileHashes, ValidateComponentSignature, AddCatalog, AddImplicationsToCatalogsAndVerifyComponentHashes |
| `ComponentStore::CRawStoreLayout` | 组件存储原始布局 | OpenCatalogFile, AddCatalogFile, FetchManifest, FetchManifestContent |
| `ComponentStore::VersionedIndexHelper` | 版本索引辅助 | OpenComponentDerivedData, InitComponentData |
| `WcpManifest` | WCP Manifest | IsFusion, FindFile, GetFileHashes |
| `Windows::ManifestParser::Rtl` | Manifest 解析 | CmsVerifyFileHash, CmsVerifyFileHashes, CmsFindMetadata |

---

## 3. CCatalog 类完整实现

### 3.1 类布局 (推断)

```cpp
class CCatalog {
    PCCTL_CONTEXT  m_pCtlContext;      // +0x00  CTL 上下文 (CertCreateCTLContext 返回)
    HCERTSTORE     m_hMsgCertStore;    // +0x04  消息证书存储 (CertOpenStore)
    PCCERT_CONTEXT m_pSignerCert;      // +0x08  签名者证书上下文
    FILETIME       m_signerTime;       // +0x0C  签名时间
    DWORD          m_dwSignerFlags;    // +0x10  签名者标志
    DWORD          m_subjectAlgorithm; // +0x28  主题算法: 2=SHA1, 4=SHA256
    BYTE           m_opaqueData[?];    // +0x36+ 其他不透明数据
};
```

### 3.2 CCatalog::Create

**地址**: 0x102474e0 | **反编译行**: 464730 | **源码**: `onecore\base\wcp\rtllib\win32lib\catalog.cpp`

```cpp
HRESULT CCatalog::Create(
    DWORD  dwFlags,         // a2: 0=普通, 1=允许损坏检测
    DWORD  dwEncodingType,  // a3: 0x10001 = PKCS_7_ASN_ENCODING | X509_ASN_ENCODING
    LBLOB* pCtlEncodedBlob, // a4: 编码的 CTL 数据 (即 .cat 文件内容)
    DWORD* pIsCorrupt)      // a5: 输出: 1=catalog 数据损坏
```

**完整流程**:

```
1. 参数验证
   - pCtlEncodedBlob->Buffer != NULL
   - 如果 dwFlags&1, pIsCorrupt != NULL

2. CertCreateCTLContext(0x10001, pCtlEncodedBlob->Buffer, pCtlEncodedBlob->Length)
   - 创建 CTL (Certificate Trust List) 上下文
   - 失败处理:
     * GetLastError() == ERROR_INVALID_DATA (13) 且 dwFlags&1:
       → *pIsCorrupt = 1, 返回 S_OK (不报错，标记为损坏)
     * 其他错误: ConvertHResultToNtStatus() 转换后返回

3. 验证 CTL INFO
   - pCtlContext->pCtlInfo != NULL
   - 失败: "CTL INFO in CTL Context was NULL" → 返回 STATUS_INVALID_PARAMETER

4. 识别 Subject Algorithm OID
   - 读取 pCtlInfo->SubjectAlgorithm.pszObjId
   - 转换为 Unicode 字符串
   - 比较:
     * "1.3.6.1.4.1.311.12.1.2" → m_subjectAlgorithm = 2 (SHA1)
     * "1.3.6.1.4.1.311.12.1.3" → m_subjectAlgorithm = 4 (SHA256)
     * 其他 → "Catalog is using an unsupported subject algorithm" → 返回 STATUS_INVALID_PARAMETER

5. 返回 S_OK
```

**关键 OID**:
- `1.3.6.1.4.1.311.12.1.2` = szOID_CATALOG_LIST_MEMBER (SHA1 catalog)
- `1.3.6.1.4.1.311.12.1.3` = szOID_CATALOG_LIST_MEMBER_V2 (SHA256 catalog)

### 3.3 CCatalog::VerifySigner

**地址**: 0x102477e0 | **反编译行**: 464942 | **源码**: `catalog.cpp`

```cpp
HRESULT CCatalog::VerifySigner(IRtlSystemIsolationLayer* pSystem)
```

**完整流程**:

```
1. 验证内部状态: m_pCtlContext != NULL, m_hMsgCertStore == NULL, m_pSignerCert == NULL

2. CertOpenStore(
       CERT_STORE_PROV_MSG,      // 1 = 从加密消息打开
       0x10001,                   // 编码类型
       0, 0,
       m_pCtlContext->hCryptMsg)  // CTL 上下文中的加密消息句柄
   → m_hMsgCertStore

3. CryptMsgGetAndVerifySigner(
       m_pCtlContext->hCryptMsg,
       1,                          // 第一个签名者
       &m_hMsgCertStore,
       0,
       &m_pSignerCert,            // 输出: 签名者证书
       &m_dwSignerFlags)          // 输出: 签名者标志
   - 失败: 检查错误码是否为已知的 catalog 损坏错误
     * CRYPT_E_SECURITY_SETTINGS (-2146881278)
     * TRUST_E_BASIC_CONSTRAINTS (-2146881277)
     * CERT_E_CHAINING (-2146881269)
     * CERT_E_WRONG_USAGE (-2146881268)
     * CERT_E_UNTRUSTEDROOT (-2146881266)
     * CERT_E_CN_NO_MATCH (-2146881229)
     * CERT_E_EXPIRED (-2146881022)
     → 输出 trace: "The catalog is corrupt."

4. VerifyOpusInfo(pSystem, m_pSignerCert, m_dwSignerFlags)
   - 验证 OPUS (Owner Publisher Update Service) 信息
   - 这是 Microsoft 特定的发布者验证

5. 返回结果
```

### 3.4 CCatalog::VerifyCertChainRoot

**地址**: 0x10247960 | **反编译行**: 465030 | **源码**: `catalog.cpp`

```cpp
HRESULT CCatalog::VerifyCertChainRoot(
    DWORD  dwFlags,     // a2: 0=正式, 1=允许测试根
    DWORD* pResult)     // a3: 输出: 0=验证通过, 1=测试根, 2=其他
```

**完整流程**:

```
1. 参数验证: dwFlags 只能是 0 或 1

2. EKU 设置: 代码签名 OID "1.3.6.1.5.5.7.3.3" (szOID_PKIX_KP_CODE_SIGNING)

3. IsLifetimeSigningCert(m_pSignerCert)
   - 检查签名者证书是否为终身签名证书 (Lifetime Signing)

4. CCatalog::GetTimeStamperInfo(this, &hasTimestamp, &timestampTime, &pTimestampChain, &timestampFlags)
   - 获取时间戳信息 (RFC 3161 或 Authenticode 时间戳)
   - 如果有时间戳，使用时间戳时间进行证书有效期验证

5. CertOpenStore(CERT_STORE_PROV_COLLECTION, 0, 0, 0, NULL)
   → hCollectionStore (集合证书存储)

6. CertAddStoreToCollection(hCollectionStore, m_hMsgCertStore, 0, 0)
   - 将消息证书存储添加到集合中 (包含中间证书)

7. 构建证书链参数 (CERT_CHAIN_PARA):
   - RequestedUsage = 代码签名 EKU
   - 可能包含时间戳证书链

8. CertGetCertificateChain(
       hChainEngine = NULL,  // 默认链引擎 (HCCE_CURRENT_USER)
       pCertContext = m_pSignerCert,
       pTime = &timestampTime (如果有时间戳) 或 NULL (当前时间),
       hAdditionalStore = hCollectionStore,
       pChainPara,
       dwFlags,
       pvReserved = NULL,
       &pChainContext)

9. CertVerifyCertificateChainPolicy(
       pszPolicyOID = MICROSOFT_ROOT_CERT_CHAIN_POLICY_OID,
       pChainContext,
       pPolicyPara,
       &pPolicyStatus)
   - 使用 Microsoft 根证书策略验证
   - 检查链是否终止于受信任的 Microsoft 根

10. 结果判断:
    - 如果链终止于 Microsoft 生产根 → *pResult = 0
    - 如果链终止于 Microsoft 测试根 (且 dwFlags&1) → *pResult = 1
    - 其他情况 → *pResult = 2

11. 清理: CertFreeCertificateChain, CertCloseStore
```

**关键证书策略 OID**:
- `1.3.6.1.4.1.311.10.1` = MICROSOFT_ROOT_CERT_CHAIN_POLICY_OID (Microsoft 根策略)
- `1.3.6.1.5.5.7.3.3` = szOID_PKIX_KP_CODE_SIGNING (代码签名 EKU)

### 3.5 CCatalog::FindSubject

**地址**: 0x10248230 | **反编译行**: 465625 | **源码**: `catalog.cpp`

```cpp
HRESULT CCatalog::FindSubject(
    DWORD   dwFlags,     // a2: 必须为 0
    LBLOB*  pSubject,    // a3: 要查找的主题 (即文件 hash)
    BOOL*   pbFound,     // a4: 输出: TRUE=找到
    LONG*   pLastError)  // a5: 输出: 最后错误 (NTSTATUS)
```

**完整流程**:

```
1. 参数验证: m_pCtlContext != NULL, dwFlags == 0

2. 初始化输出: *pbFound = FALSE, *pLastError = 0

3. CertFindSubjectInCTL(
       0x10001,           // 编码类型
       1,                  // 主题类型 (CTL_ANY_SUBJECT_TYPE)
       pSubject->Buffer,   // 主题数据 (hash 字节)
       m_pCtlContext,      // CTL 上下文
       0)                  // 索引 (从第一个开始)
   → PCTL_ENTRY (找到的 CTL 条目) 或 NULL

4. *pbFound = (pCtlEntry != NULL)

5. 如果 pLastError != NULL:
   *pLastError = ConvertHResultToNtStatus(GetLastError())

6. 返回 S_OK
```

**注意**: `CertFindSubjectInCTL` 是 Win32 API，用于在 CTL 中查找指定的主题。对于安全目录，主题就是文件的 SHA1/SHA256 hash。

### 3.6 CCatalog::GetSignerPublicKeyToken

**地址**: 0x102482f0 | **反编译行**: 465669 | **源码**: `catalog.cpp`

```cpp
HRESULT CCatalog::GetSignerPublicKeyToken(
    IRtlSystemIsolationLayer* pSystem,
    LBLOB* pPublicKeyToken)  // 输出: 8字节公钥 token
```

**完整流程**:

```
1. 参数验证: pPublicKeyToken->MaximumLength >= 8 (PUBLIC_KEY_TOKEN_LENGTH)

2. 如果已缓存 (m_opaqueData[36] != 0):
   → 直接返回缓存的 PublicKeyToken

3. CryptAcquireContextW(
       &hCryptProv,
       NULL,           // 容器名
       NULL,           // 提供者名 (默认)
       PROV_RSA_FULL,  // 1
       CRYPT_VERIFYCONTEXT | CRYPT_NEWKEYSET | CRYPT_MACHINE_KEYSET)  // 0xF0000000
   → hCryptProv

4. CryptImportKey(
       hCryptProv,
       m_pSignerCert->pCertInfo->SubjectPublicKeyInfo.PublicKey.pbData,
       m_pSignerCert->pCertInfo->SubjectPublicKeyInfo.PublicKey.cbData,
       0,  // hPubKey (无)
       0,  // dwFlags
       &hPublicKey)

5. 计算公钥 SHA1 hash:
   CryptCreateHash(hCryptProv, CALG_SHA1, 0, 0, &hHash)
   CryptHashPublicKeyInfo(hHash, 0, X509_ASN_ENCODING, &SubjectPublicKeyInfo)
   // 或: CryptGetHashParam(hHash, HP_HASHVAL, pbHash, &cbHash, 0)

6. 取 SHA1 hash 的前 8 字节作为 PublicKeyToken
   memcpy(pPublicKeyToken->Buffer, sha1Hash, 8)
   pPublicKeyToken->Length = 8

7. 缓存结果到 m_opaqueData

8. 清理: CryptDestroyKey, CryptDestroyHash, CryptReleaseContext

9. 返回 S_OK
```

**PublicKeyToken 定义**: 8 字节 = SHA1(证书公钥) 的前 8 字节。这与 .NET 强名称程序集的 PublicKeyToken 相同算法。

### 3.7 CCatalog 方法调用关系

```
CCatalog::Create (加载 .cat 文件)
    │
    ├─→ CertCreateCTLContext (创建 CTL 上下文)
    └─→ OID 识别 (SHA1=2 / SHA256=4)

CCatalog::VerifySigner (验证签名者)
    │
    ├─→ CertOpenStore (从消息打开证书存储)
    ├─→ CryptMsgGetAndVerifySigner (获取并验证签名者)
    └─→ VerifyOpusInfo (Microsoft 发布者验证)

CCatalog::VerifyCertChainRoot (验证证书链根)
    │
    ├─→ IsLifetimeSigningCert (终身签名检查)
    ├─→ CCatalog::GetTimeStamperInfo (时间戳)
    ├─→ CertOpenStore(CERT_STORE_PROV_COLLECTION)
    ├─→ CertAddStoreToCollection
    ├─→ CertGetCertificateChain (构建证书链)
    └─→ CertVerifyCertificateChainPolicy (Microsoft Root Policy)

CCatalog::FindSubject (查找文件 hash)
    └─→ CertFindSubjectInCTL

CCatalog::GetSignerPublicKeyToken (获取公钥 token)
    ├─→ CryptAcquireContextW
    ├─→ CryptImportKey
    └─→ SHA1(公钥) 前 8 字节
```

---

## 4. 文件 Hash 验证完整调用链

### 4.1 调用链总览

```
CCSDirectTransaction::VerifyFileHashes (重载1: 直接参数)
  行138709, 源码 csd_transact.cpp:2277
    │
    └─→ Windows::Cms::Rtl::ValidateFileHash
          (底层 hash 计算与比较)

CCSDirectTransaction::VerifyFileHashes (重载2: WcpManifest)
  行138804, 源码 csd_transact.cpp
    │
    ├─→ WcpManifest::IsFusion
    ├─→ WcpManifest::FindFile (在 manifest 中查找文件)
    ├─→ WcpManifest::GetFileHashes (获取文件 hash 列表)
    └─→ Windows::ManifestParser::Rtl::CmsVerifyFileHashes
          行540485, 源码 hashverify.cpp
            │
            ├─→ CdfBindValue<Cms::_CMS_FILE> (绑定文件值)
            ├─→ Windows::Cms::Rtl::FileHashAlgorithmToDigest
            └─→ CmsVerifyFileHash
                  行540424, 源码 hashverify.cpp:91
                    │
                    └─→ Windows::Cms::Rtl::ValidateFileHash
```

### 4.2 CCSDirectTransaction::VerifyFileHashes (重载1)

**地址**: 0x101172e3 | **反编译行**: 138709 | **源码**: `csd_transact.cpp:2277`

```cpp
HRESULT CCSDirectTransaction::VerifyFileHashes(
    // 参数通过寄存器传递，反编译显示为 a1-a10
    BYTE   a9,    // 布尔: 是否在 hash 不匹配时报错
    BYTE*  a10)   // 输出: 1=匹配, 0=不匹配
```

**流程**:

```
1. 跟踪日志 (如果 Facility_ComponentStore 跟踪标志 & 0xE)

2. Windows::Cms::Rtl::ValidateFileHash(
       a9 ^ 1,    // 取反: a9=1(报错)→传0, a9=0(不报错)→传1
       2,         // hash 类型?
       a3, a4, a5, // 文件路径/hash 数据
       1,         // flags
       a7, a8,    // 文件对象
       &v12)      // 输出结果

3. 如果 ValidateFileHash 失败 → 返回错误

4. 结果判断:
   - v12 == 2 → *a10 = 1 (匹配)
   - v12 != 2 → *a10 = 0 (不匹配)
     * 输出 trace: "VerifyFileHashes encountered hash difference for {f} in {i}"
     * 如果 a9 != 0 (要求报错) → 返回 CBS_E_CORRUPT_FILE (0xC015001B)

5. 返回 S_OK
```

### 4.3 CCSDirectTransaction::VerifyFileHashes (重载2)

**地址**: 0x1011749c | **反编译行**: 138804 | **源码**: `csd_transact.cpp`

```cpp
HRESULT CCSDirectTransaction::VerifyFileHashes(
    WcpManifest*      pManifest,    // manifest 对象
    LUNICODE_STRING*  pFilePath,    // 文件路径
    IRtlFile*         pFile,        // 文件对象
    BOOL              bStrict,      // 严格模式
    BOOL*             pbMatch)      // 输出: TRUE=匹配
```

**流程**:

```
1. *pbMatch = TRUE (默认匹配)

2. WcpManifest::IsFusion(pManifest)
   - 检查是否为 Fusion manifest (SxS 风格)

3. 将文件路径转换为 UTF8:
   Windows::Rtl::AutoString<LUTF8_STRING>::Assign(&utf8Path, pFilePath)

4. WcpManifest::FindFile(pManifest, 1, &utf8Path, &fileIndex)
   - 在 manifest 中查找文件
   - 失败 → 返回错误

5. WcpManifest::GetFileHashes(fileIndex, &hashCount, NULL)
   - 获取该文件的 hash 数量

6. Windows::ManifestParser::Rtl::CmsVerifyFileHashes(
       pManifest,
       manifestIdentity,
       pCdf,
       pFilePath,
       pFile,
       &result)
   - 实际执行 hash 验证

7. 结果判断:
   - result == 6 → 匹配 (CmsVerifyFileHash 返回 6)
   - 其他 → 不匹配，输出 trace

8. 返回 S_OK
```

### 4.4 CmsVerifyFileHash

**地址**: 0x1028ca3b | **反编译行**: 540424 | **源码**: `onecore\base\wcp\manifestparser\hashverify.cpp:91`

```cpp
HRESULT CmsVerifyFileHash(
    BYTE    flags,      // a1: hash 验证标志位
    int     a2,         // digest 算法
    SIZE_T  a3, a4, a5, // hash 数据
    int     a6,         // flags
    int     a7,         // manifest 解析器
    int     a8,         // 文件路径
    DWORD*  pResult)    // a9: 输出结果
```

**flags 解码** (a1):

| 位 | 值 | 含义 |
|----|-----|------|
| 3 | 0x08 | 备用 hash 回退 |
| 4 | 0x10 | 主 hash |
| 5 | 0x20 | 备用 hash |

```cpp
v10 = 0;
if (flags & 0x10) v10 |= 1;   // 主 hash
if (flags & 0x20) v10 |= 2;   // 备用 hash
if (flags & 0x08) v10 |= 4;   // 备用回退
if (flags & 0x04) v10 |= 8;   // 其他
```

**流程**:

```
1. 解码 flags → v10 (传递给 ValidateFileHash)

2. Windows::Cms::Rtl::ValidateFileHash(
       v10, a2, a3, a4, a5, a6, a7, a8, &v14)

3. 结果映射:
   - v14 == 1 → *pResult = 2 (主 hash 匹配)
   - v14 == 2 → *pResult = 5 (备用 hash 匹配)
   - v14 == 3 → *pResult = 6 (hash 不匹配)
     * 如果 (flags & 0x10) == 0 (非主 hash 模式):
       → 返回 CBS_E_CORRUPT_FILE (0xC015001B)
       → 源码位置: hashverify.cpp:91
   - v14 == 0 → *pResult = 0 (无 hash)

4. 返回 S_OK
```

### 4.5 CmsVerifyFileHashes

**地址**: 0x1028cb19 | **反编译行**: 540485 | **源码**: `hashverify.cpp`

```cpp
HRESULT Windows::ManifestParser::Rtl::CmsVerifyFileHashes(
    DWORD*                pResult,        // 输出结果
    Windows::ManifestParser::Rtl* this,
    unsigned int          manifestIdentity,
    IRtlCdf*              pCdf,
    LUNICODE_STRING*      pFilePath,
    IRtlFile*             pFile,
    unsigned int*         pFinalResult)
```

**流程**:

```
1. 从 manifest 获取文件 identity:
   pManifest->GetFileIdentity(1, &identityType, &identityCount, &identityVersion)
   - 失败 → *pCdf = 1, 返回

2. 验证 identity 格式:
   - identityCount == 1 且 identityVersion == 1
   - 不满足 → *pCdf = 1, 返回

3. CdfBindValue<Cms::_CMS_FILE>(pManifest, &cmsFile, identityValue)
   - 绑定 CMS 文件结构

4. 如果 cmsFile.hashAlgorithm != -1:
   a. Windows::Cms::Rtl::FileHashAlgorithmToDigest(cmsFile.hashAlgorithm)
      → digestAlg (如果返回 -1，跳到备用 hash 处理)
   b. 从 manifest 获取 hash 数据:
      pManifest->GetHashData(cmsFile.hashIndex, &hashBlob)
   c. CmsVerifyFileHash(
          62,                    // flags = 0x3E (bit1+2+3+4+5)
          digestAlg,
          hashBlob,
          3,                     // flags
          pManifestParser,
          pFilePath,
          &result)
   d. 如果 result != 5 (非备用 hash 匹配) → 返回结果
   e. result == 5 → 主 hash 不匹配，尝试备用 hash

5. 备用 hash 处理:
   - 如果 cmsFile.alternateHashAlgorithm == -1 → *pFinalResult = 4 (无备用 hash)
   - 否则:
     a. FileHashAlgorithmToDigest(alternateHashAlgorithm)
     b. GetHashData(alternateHashIndex, &altHashBlob)
     c. CmsVerifyFileHash(flags, altDigestAlg, altHashBlob, ...)
     d. 返回结果

6. 安全检查:
   - 如果 hash 算法不兼容，输出 trace:
     "!! Security issue - manifest found with incompatible file xp hash type {d} - identity {id}"
```

**返回值含义**:

| 值 | 含义 |
|----|------|
| 0 | 无 hash |
| 1 | identity 格式错误 |
| 2 | 主 hash 匹配 |
| 4 | 无备用 hash |
| 5 | 主 hash 不匹配，尝试备用 |
| 6 | hash 不匹配 (最终) |

---

## 5. 组件签名验证完整调用链

### 5.1 调用链总览

```
CCSDirectTransaction::ValidateComponentSignature
  行140886, 源码 csd_transact.cpp:3825
    │
    ├─→ CRawStoreLayout::OpenCatalogFile (打开 .cat 文件)
    │
    ├─→ CCatalog::Create (创建 CTL 上下文)
    │
    ├─→ CCatalog::GetSignerPublicKeyToken (获取签名者公钥 token)
    │
    ├─→ 组件 Identity 中的 PublicKeyToken 比较
    │   ├─ 匹配 → 验证通过
    │   └─ 不匹配 → 继续证书链验证
    │
    ├─→ DoesTargetHaveTestRootCert (检查测试根证书)
    │
    └─→ CCatalog::VerifyCertChainRoot (验证证书链根)
          ├─→ CertGetCertificateChain
          └─→ CertVerifyCertificateChainPolicy (Microsoft Root)
```

### 5.2 CCSDirectTransaction::ValidateComponentSignature

**地址**: 0x1011961e | **反编译行**: 140886 | **源码**: `csd_transact.cpp:3825`

```cpp
HRESULT CCSDirectTransaction::ValidateComponentSignature(
    IRtlDefinitionIdentity* pComponentIdentity,  // 组件身份
    LUNICODE_STRING*          pCatalogPath)       // catalog 文件路径
```

**完整流程**:

```
1. CRawStoreLayout::OpenCatalogFile(
       this+2048,           // CRawStoreLayout 指针
       pCatalogPath,
       &catalogBlob,        // 输出: catalog 文件内容
       &bHasCatalog,        // 输出: 是否有 catalog
       &catalogInfo)
   - 打开并读取 .cat 文件
   - 失败 → 返回错误

2. 如果 !bHasCatalog → 直接返回 S_OK (无 catalog 不验证)

3. CCatalog::Create(
       &catalog,
       1,                   // dwFlags: 允许损坏检测
       0x10001,             // 编码类型
       &catalogBlob,
       &isCorrupt)
   - 创建 CTL 上下文
   - 如果 isCorrupt == 1:
     → MarkStoreCorrupt (标记存储损坏)
     → ReportStoreFileCorruption (报告)
     → 返回 STATUS_CATALOG_CORRUPT (0xC00F011A?)

4. CCatalog::GetSignerPublicKeyToken(
       m_pSystemIsolation,
       &signerPublicKeyToken)
   - 获取签名者的 8 字节 PublicKeyToken

5. 从组件 Identity 获取 PublicKeyToken:
   pComponentIdentity->GetAttribute(0, &identityPublicKeyToken)
   - identityPublicKeyToken 格式: {Length, MaximumLength, Buffer}
   - 验证 Length >= 8

6. 比较 PublicKeyToken:
   if (signerPublicKeyToken.Buffer == identityPublicKeyToken.Buffer ||
       memcmp(signerPublicKeyToken.Buffer, identityPublicKeyToken.Buffer, 8) == 0)
   → 匹配! 跳过证书链验证，返回 S_OK

7. 不匹配 → 检查测试根:
   DoesTargetHaveTestRootCert(m_pSystemIsolation, &hasTestRoot)
   - 检查目标系统是否安装了 Microsoft 测试根证书

8. 确定验证标志:
   verifyFlags = hasTestRoot;
   if (identityFlags & 0x100 && identityBuild == 1)
       verifyFlags |= 2;  // 组件标记为测试签名

9. CCatalog::VerifyCertChainRoot(&catalog, verifyFlags, &chainResult)
   - 验证证书链是否终止于 Microsoft 根

10. 返回结果
```

### 5.3 关键比较逻辑

```cpp
// 行140966-140982
if ( (identityFlags & 2) == 0 )
    __debugbreak();  // identity 必须有 PublicKeyToken 属性

identityPublicKeyToken = identityBlob.Buffer;
identityTokenLength = identityBlob.Length;

if ( signerToken == identityPublicKeyToken ||
     memcmp(signerTokenData, identityTokenData, 8) != 0 )
{
    // PublicKeyToken 不匹配 → 需要证书链验证
    DoesTargetHaveTestRootCert(system, &hasTestRoot);

    verifyFlags = hasTestRoot;
    if ( (identityFlags & 0x100) != 0 && identityBuild == 1 )
        verifyFlags |= 2;

    CCatalog::VerifyCertChainRoot(catalog, verifyFlags, &chainResult);
}
// else: PublicKeyToken 匹配 → 验证通过
```

---

## 6. Manifest vs Catalog 验证

### 6.1 VerifyManifestAgainstCatalog

**地址**: 0x10109852 | **反编译行**: 124987 | **源码**: `onecore\base\wcp\componentstore\corruptionrepair.cpp`

```cpp
const char* anonymous_namespace::VerifyManifestAgainstCatalog(
    ComponentStoreImpl* pStore,
    DWORD*              pResult,        // 输出结果
    const char*         pCatalogBlob,   // catalog 文件内容
    int                 catalogSize,
    int                 a5, a6, a7, a8,
    const char*         pManifestBlob,  // manifest 文件内容
    const char*         pManifestPath,
    int                 a11)
```

**流程**:

```
1. 参数验证: pDisp != NULL

2. CCatalog::Create(
       &catalog,
       1,              // 允许损坏检测
       0x10001,
       &catalogBlob,
       &isCorrupt)
   - 如果 isCorrupt == 1:
     → MarkStoreCorrupt
     → ReportStoreFileCorruption
     → *pResult = 3, 返回

3. 根据 catalog subject algorithm 计算 manifest hash:
   - algorithm == 2 (SHA1):
     → RtlAllocateLBlob(0x14)  (20字节)
     → RtlHashLBlob(0, 2, pManifestBlob)  → SHA1
     → FormatBytesIntoString → 十六进制字符串
   - algorithm == 4 (SHA256):
     → RtlAllocateLBlob(0x20)  (32字节)
     → RtlHashLBlob(0, 3, pManifestBlob)  → SHA256
   - 其他:
     → "Can't understand the catalog subject algorithm"
     → 返回 STATUS_INVALID_PARAMETER

4. CCatalog::FindSubject(
       &catalog,
       0,
       &manifestHash,
       &bFound,
       &findResult)

5. 结果:
   - findResult >= 0 (找到) → *pResult = 1
   - findResult < 0 (未找到) → *pResult = 2 * bFound + 2
     * bFound=TRUE → *pResult = 4
     * bFound=FALSE → *pResult = 2

6. CCatalog::Close(&catalog)

7. 返回
```

### 6.2 AddImplicationsToCatalogsAndVerifyComponentHashes

**地址**: ~0x10109bf0 | **反编译行**: 160291 | **源码**: `onecore\base\wcp\componentstore\csd_winners.cpp:1625`

这是更高级别的函数，在组件安装/更新时验证所有组件的 hash 是否在 catalog 中：

```
1. 遍历所有受影响的组件 (通过 CBlobTable / CGuidIdTable)

2. 对每个组件:
   a. 打开组件的 catalog 文件
   b. CCatalog::Create
   c. 根据 catalog subject algorithm 计算组件 manifest hash:
      - algorithm == 2 (SHA1):
        → CRawStoreLayout::FetchManifestContent (获取 manifest 内容)
        → RtlHashLBlob(0, 2, manifestContent) → SHA1 (20字节)
      - algorithm == 4 (SHA256):
        → VersionedIndexHelper::OpenComponentDerivedData
        → 从派生数据中获取 "S256H" 属性 (预计算的 SHA256 hash)
        → 如果没有 S256H:
          → FetchManifestContent + RtlHashLBlob(0, 3) → SHA256 (32字节)
   d. CCatalog::FindSubject(catalog, 0, componentHash, &bFound, &result)
   e. 如果未找到:
      → "Couldn't find the hash of component: {id} in the catalog {cat}."
      → 返回错误 (csd_winners.cpp:1755)

3. 所有组件验证通过后:
   → CRawStoreLayout::AddCatalogMarkFromTlc
   → CRawStoreLayout::AddVerifiedCatalogMarkInVersionedIndex
```

**关键优化**: SHA256 hash 优先从组件派生数据中的 "S256H" 属性获取，避免重复计算。

---

## 7. 关键常量与 OID

### 7.1 OID 列表

| OID | 名称 | 用途 |
|-----|------|------|
| `1.3.6.1.4.1.311.12.1.2` | szOID_CATALOG_LIST_MEMBER | SHA1 catalog 主题算法 |
| `1.3.6.1.4.1.311.12.1.3` | szOID_CATALOG_LIST_MEMBER_V2 | SHA256 catalog 主题算法 |
| `1.3.6.1.5.5.7.3.3` | szOID_PKIX_KP_CODE_SIGNING | 代码签名 EKU |
| `1.3.6.1.4.1.311.10.1` | MICROSOFT_ROOT_CERT_CHAIN_POLICY_OID | Microsoft 根证书策略 |
| `1.3.6.1.4.1.311.2.1.14` | szOID_RFC3161_counterSign | RFC 3161 时间戳 |
| `1.2.840.113549.1.1.11` | szOID_RSA_SHA256RSA | SHA256 RSA 签名 |
| `1.2.840.113549.2.1` | szOID_OIWSEC_sha1 | SHA1 |
| `2.16.840.1.101.3.4.2.1` | szOID_NIST_sha256 | SHA256 |

### 7.2 Catalog Subject Algorithm

| 值 | 算法 | Hash 长度 | OID |
|----|------|----------|-----|
| 2 | SHA1 | 20 字节 | 1.3.6.1.4.1.311.12.1.2 |
| 4 | SHA256 | 32 字节 | 1.3.6.1.4.1.311.12.1.3 |

### 7.3 Hash 验证返回值

#### ValidateFileHash → CmsVerifyFileHash

| ValidateFileHash | CmsVerifyFileHash | 含义 |
|-------------------|-------------------|------|
| 1 | 2 | 主 hash 匹配 |
| 2 | 5 | 备用 hash 匹配 |
| 3 | 6 | hash 不匹配 |
| 0 | 0 | 无 hash |

#### CmsVerifyFileHashes

| 值 | 含义 |
|----|------|
| 0 | 无 hash |
| 1 | identity 格式错误 |
| 2 | 主 hash 匹配 |
| 4 | 无备用 hash |
| 5 | 主 hash 不匹配，尝试备用 |
| 6 | hash 不匹配 (最终) |

#### VerifyManifestAgainstCatalog

| 值 | 含义 |
|----|------|
| 1 | 找到 (匹配) |
| 2 | 未找到 (bFound=FALSE) |
| 3 | catalog 损坏 |
| 4 | 未找到 (bFound=TRUE) |

### 7.4 编码类型常量

| 值 | 名称 |
|----|------|
| 0x00001 | X509_ASN_ENCODING |
| 0x10000 | PKCS_7_ASN_ENCODING |
| 0x10001 | X509_ASN_ENCODING \| PKCS_7_ASN_ENCODING (默认) |

### 7.5 PublicKeyToken

- **长度**: 8 字节
- **算法**: SHA1(证书 SubjectPublicKeyInfo) 的前 8 字节
- **用途**: 组件身份与 catalog 签名者的快速匹配
- **与 .NET 关系**: 与 .NET 强名称程序集的 PublicKeyToken 算法完全相同

---

## 8. 错误码体系

### 8.1 CBS 相关错误码

| 错误码 | 名称 | 含义 |
|--------|------|------|
| 0xC015001B | CBS_E_CORRUPT_FILE | 文件 hash 不匹配 (CmsVerifyFileHash:91) |
| 0xC00F011A | STATUS_CATALOG_CORRUPT? | catalog 数据损坏 |
| 0x8007000D | ERROR_INVALID_DATA | CertCreateCTLContext 失败 (catalog 格式错误) |

### 8.2 CryptoAPI 错误码 (VerifySigner)

| 错误码 | 名称 | 含义 |
|--------|------|------|
| 0x8009200D | CRYPT_E_SECURITY_SETTINGS | 安全设置限制 |
| 0x80092004 | CRYPT_E_NOT_FOUND | 未找到 |
| 0x800B010F | CERT_E_CHAINING | 证书链构建失败 |
| 0x800B0110 | CERT_E_WRONG_USAGE | 证书 EKU 不匹配 |
| 0x800B0109 | CERT_E_UNTRUSTEDROOT | 根证书不受信任 |
| 0x800B010A | CERT_E_CN_NO_MATCH | CN 不匹配 |
| 0x800B0101 | CERT_E_EXPIRED | 证书过期 |
| 0x800B010B | TRUST_E_BASIC_CONSTRAINTS | 基本约束验证失败 |

### 8.3 错误转换

wcp.dll 使用 `ConvertHResultToNtStatus()` 将 Win32/HRESULT 错误码转换为 NTSTATUS：

```
Win32 错误 (0-65535) → (error & 0xFFFF) | 0x80070000 → NTSTATUS
HRESULT 错误 (0x800xxxxxx) → 直接转换
```

---

## 9. 源码行号索引

### 9.1 wcp.c (IDA 18MB 版)

| 反编译行 | 函数 | 源码文件:行 |
|---------|------|------------|
| 464730 | CCatalog::Create | rtllib\win32lib\catalog.cpp |
| 464942 | CCatalog::VerifySigner | catalog.cpp |
| 465030 | CCatalog::VerifyCertChainRoot | catalog.cpp |
| 465625 | CCatalog::FindSubject | catalog.cpp |
| 465669 | CCatalog::GetSignerPublicKeyToken | catalog.cpp:1059 |
| 138709 | CCSDirectTransaction::VerifyFileHashes (重载1) | csd_transact.cpp:2277 |
| 138804 | CCSDirectTransaction::VerifyFileHashes (重载2) | csd_transact.cpp |
| 140821 | CCSDirectTransaction::AddCatalog | csd_transact.cpp |
| 140886 | CCSDirectTransaction::ValidateComponentSignature | csd_transact.cpp:3825 |
| 124987 | anonymous_namespace::VerifyManifestAgainstCatalog | corruptionrepair.cpp |
| 160291 | CCSDirectTransaction::AddImplicationsToCatalogsAndVerifyComponentHashes | csd_winners.cpp:1625 |
| 540424 | CmsVerifyFileHash | manifestparser\hashverify.cpp:91 |
| 540485 | Windows::ManifestParser::Rtl::CmsVerifyFileHashes | hashverify.cpp |
| 60546 | AttributeValidation::ValidateHashAlg | (manifest 属性验证) |

### 9.2 源码文件清单

| 源码文件 | 内容 |
|---------|------|
| `onecore\base\wcp\rtllib\win32lib\catalog.cpp` | CCatalog 类完整实现 |
| `onecore\base\wcp\rtllib\inc\catalog.h` | CCatalog 类头文件 |
| `onecore\base\wcp\componentstore\csd_transact.cpp` | CCSDirectTransaction 事务实现 |
| `onecore\base\wcp\componentstore\csd_winners.cpp` | 组件赢家选择与 catalog 验证 |
| `onecore\base\wcp\componentstore\corruptionrepair.cpp` | 存储损坏修复与 manifest-catalog 验证 |
| `onecore\base\wcp\manifestparser\hashverify.cpp` | CMS hash 验证实现 |

---

## 10. 重建项目 Implications

### 10.1 可以直接复用的组件

| 组件 | 复用方式 | 说明 |
|------|---------|------|
| CCatalog 类 | 可直接用 Win32 CryptoAPI 重写 | 所有方法都有明确的 Win32 API 对应 |
| CertCreateCTLContext | 系统 API | 无需重写，Windows 自带 |
| CertFindSubjectInCTL | 系统 API | 无需重写 |
| CryptMsgGetAndVerifySigner | 系统 API | 无需重写 |
| CertGetCertificateChain | 系统 API | 无需重写 |
| CertVerifyCertificateChainPolicy | 系统 API | 无需重写 |

### 10.2 需要重新实现的逻辑

| 逻辑 | 难度 | 说明 |
|------|------|------|
| Catalog Subject Algorithm OID 识别 | 低 | 两个 OID 字符串比较 |
| PublicKeyToken 计算 | 低 | CryptImportKey + SHA1 前8字节 |
| CmsVerifyFileHash flags 解码 | 中 | 4个标志位的组合逻辑 |
| CmsVerifyFileHashes 备用 hash 回退 | 中 | 主 hash 失败后尝试备用 |
| ValidateComponentSignature 匹配逻辑 | 中 | PublicKeyToken 匹配 + 证书链回退 |
| AddImplicationsToCatalogsAndVerifyComponentHashes | 高 | 多组件遍历 + S256H 优化 + catalog 标记 |

### 10.3 关键依赖

重建 wcp.dll 的 hash/cert 验证功能需要：
1. **Win32 CryptoAPI** — 所有 catalog 操作的基础
2. **Microsoft 根证书** — 证书链验证需要系统安装 Microsoft 生产/测试根证书
3. **Catalog 文件格式** — .cat 文件是 PKCS#7 签名的 CTL，需要正确生成
4. **SHA1/SHA256 hash 计算** — 文件内容 hash 与 catalog subject 匹配

### 10.4 测试策略

1. **单元测试**: 用已知的 Microsoft 签名 .cat 文件测试 CCatalog 类
2. **集成测试**: 用真实的 CBS .cab 包测试完整的 hash 验证链
3. **证书链测试**: 分别测试生产签名和测试签名的 catalog
4. **损坏检测测试**: 测试篡改文件后 hash 验证是否正确失败

---

*文档结束 — 生成于 2026-08-27，基于 IDA Pro 9.4 完整反编译 wcp.dll (18,912,503 字节)*

*本文档为 v2.0 完整版，替代之前基于 12.5MB 反编译的部分分析。所有 CCatalog 类方法、hash 验证调用链、证书链验证均已完整逆向。*
