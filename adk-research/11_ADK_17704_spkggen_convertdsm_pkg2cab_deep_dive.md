# ADK 10.0.17704.1000 — spkggen.exe + ConvertDSM.exe + pkg2cab 流水线深度逆向分析

> 文档版本: 1.0 | 日期: 2026-08-27 | 研究目标: 为重建 ADK 镜像构建工具链提供完整技术参考

---

## 目录

1. [执行摘要](#1-执行摘要)
2. [pkg2cab 完整流水线总览](#2-pkg2cab-完整流水线总览)
3. [spkggen.exe 深度逆向 (.NET)](#3-spkggenexe-深度逆向-net)
4. [ConvertDSM.exe 深度逆向 (Native C++)](#4-convertdsmexe-深度逆向-native-c)
5. [CBS Manifest 生成机制 (AddFileEntry)](#5-cbs-manifest-生成机制-addfileentry)
6. [安全目录生成机制 (UpdateCatAndSign)](#6-安全目录生成机制-updatecatandsign)
7. [命令行接口完整参考](#7-命令行接口完整参考)
8. [关键数据结构与算法](#8-关键数据结构与算法)
9. [重建项目路线图](#9-重建项目路线图)
10. [源码行号索引](#10-源码行号索引)

---

## 1. 执行摘要

### 核心发现

ADK 17704 的包构建流水线由 **三个层级** 组成，跨越 .NET 与原生 C++ 两种运行时：

| 层级 | 工具 | 运行时 | 输入 | 输出 | 职责 |
|------|------|--------|------|------|------|
| L1 | PkgGen.exe | .NET | .pkg.xml + 插件 | .spkg | 项目解析、宏展开、插件调度、卫星构建 |
| L2 | spkggen.exe | .NET | .pkg.xml | .spkg | 实际包生成（PackageGenerator.Build） |
| L3 | ConvertDSM.exe | Native C++ (i386) | .spkg / .dsm | CBS .cab | SPKG→CBS 转换、manifest 生成、目录签名 |

**关键修正**: spkggen.exe 是 **.NET 程序集**（AssemblyFileVersion 10.0.10011.16384, .NET Framework 4.0），不是原生二进制。之前批量反编译脚本误分类导致其被排除，但 ILSpy 反编译结果 `spkggen.cs` (18.4KB) 已存在于 `outputsrc\dotnet\`。

### 三个最有价值的逆向成果

1. **ConvertDSM::AddFileEntry** — 完整还原了 CBS manifest 中 `<file>` 元素的生成逻辑，包括 SHA256 hash 嵌入、FNV-1a 文件名哈希、boot 文件不压缩规则
2. **ConvertDSM::UpdateCatAndSign** — 揭示了安全目录生成的双工具路径：makecat.exe（.CDF 定义文件）vs updcat.exe（命令行直接添加），以及后续签名流程
3. **spkggen Program.Main** — 完整还原了 .NET 包生成器的命令行接口、MEF 插件加载、XSD 合并、宏解析、权限提升全流程

---

## 2. pkg2cab 完整流水线总览

### 2.1 端到端数据流

```
.pkg.xml (项目描述文件)
    │
    ▼
┌─────────────────────────────────────────┐
│ PkgGen.exe (.NET, 28KB)                │
│  - 解析 .pkg.xml                        │
│  - 加载 MEF 插件 (PkgGen.Plugin.*.dll) │
│  - 5层宏表展开                          │
│  - 调度 spkggen.exe 生成 .spkg          │
│  - 卫星构建 (语言/分辨率)               │
└─────────────────┬───────────────────────┘
                  │ 调用
                  ▼
┌─────────────────────────────────────────┐
│ spkggen.exe (.NET, 19.5KB)             │
│  - PackageGenerator.Build()             │
│  - 解析 .pkg.xml → 内部对象模型         │
│  - 生成 .spkg (Smart Package)           │
│  - 需要 BackupPrivilege + SecurityPriv  │
└─────────────────┬───────────────────────┘
                  │ .spkg
                  ▼
┌─────────────────────────────────────────┐
│ ConvertDSM.exe (Native, 355KB, i386)   │
│  - 读取 .spkg / .dsm                    │
│  - 生成 update.mum (CBS manifest)       │
│  - 生成 update.cat (安全目录)            │
│  - 复制/压缩 payload 文件                │
│  - 打包为 CBS .cab                       │
└─────────────────┬───────────────────────┘
                  │
    ┌─────────────┼─────────────┐
    ▼             ▼             ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│makecat.exe│ │updcat.exe│ │signtool  │
│(.CDF→.cat)│ │(CLI→.cat) │ │(.cat签名)│
└──────────┘ └──────────┘ └──────────┘
                  │
                  ▼
          CBS .cab (update.mum + update.cat + payload)
```

### 2.2 各组件源码位置

| 组件 | 反编译源码 | 大小 | 关键函数 |
|------|-----------|------|---------|
| spkggen.exe | `outputsrc\dotnet\spkggen.cs` | 18.4KB | `Program.Main`, `MergingSchemaValidator` |
| ConvertDSM.exe | `outputsrc\native\ConvertDSM.c` | 1.6MB / 53376行 | `RunConvertDSM`, `AddFileEntry`, `UpdateCatAndSign` |
| ConvertDSMDLL | `outputsrc\native\ConvertDSMDLL.c` | 1.4MB | (DLL 导出函数) |
| PkgGen.exe | `outputsrc\dotnet\PkgGen.cs` | 35KB | (入口) |
| PkgBldr.Common | `outputsrc\dotnet\PkgBldr.Common.cs` | 192KB | (插件框架) |
| pkgcommonmanaged | `outputsrc\dotnet\pkgcommonmanaged.cs` | 429KB | (PkgCommon 核心) |

---

## 3. spkggen.exe 深度逆向 (.NET)

### 3.1 程序集元数据

```csharp
[assembly: AssemblyFileVersion("10.0.10011.16384")]
[assembly: AssemblyVersion("10.0.0.0")]
[assembly: TargetFramework(".NETFramework,Version=v4.0")]
[assembly: ComVisible(false)]
[assembly: CLSCompliant(false)]
namespace Microsoft.WindowsPhone.ImageUpdate.PackageGenerator
```

**注意**: spkggen 的 AssemblyFileVersion 是 10.0.10011.16384（Windows 10 早期版本号），而 ADK 整体版本是 10.0.17704.1000。这说明 spkggen.exe 自 Windows 10 早期（Threshold）以来未更新，是稳定的遗留组件。

### 3.2 类结构

spkggen.exe 仅包含两个内部类：

#### 3.2.1 `Program` — 主入口类

```csharp
internal class Program
{
    private const string c_strProjExtension = ".pkg.xml";
    private static readonly List<string> ErrorBuffer = new List<string>();

    // 日志缓冲：收集原生组件的错误，统一输出
    private static void LogErrorToBuffer(string logString) { ... }
    private static void WriteErrorBuffer() { ... }

    // MEF 插件加载
    private static Dictionary<string, IPkgPlugin> LoadPackagePlugins() { ... }

    // XSD 验证事件
    private static void OnSchemaValidationEvent(object sender, ValidationEventArgs e) { ... }

    // XSD 输出
    private static void WriteXmlSchema(XmlSchema schema, string filePath) { ... }

    // 宏导入
    private static void ImportCommandLineMacros(MacroResolver macroResolver, string variables) { ... }
    private static void ImportGlobalMacros(MacroResolver macroResolver, XmlValidator schemaValidator) { ... }
    private static void ImportGlobalMacros(MacroResolver macroResolver, string file, XmlValidator schemaValidator) { ... }

    // 主入口
    private static int Main(string[] args) { ... }
}
```

#### 3.2.2 `MergingSchemaValidator` — XSD 合并验证器

继承自 `XmlValidator`，核心功能是将 MEF 插件提供的 XSD schema 片段合并到基础 schema 中：

```csharp
internal class MergingSchemaValidator : XmlValidator
{
    public XmlSchema AddSchemaWithPlugins(Stream baseSchemaStream, IEnumerable<IPkgPlugin> plugins)
    public XmlSchema AddSchemaWithPlugins(XmlSchema baseSchema, IEnumerable<IPkgPlugin> plugins)

    private bool SchemaHasElementName(XmlSchema schema, string xmlElementName)
    private XmlSchemaElement GetSchemaComponentsType(XmlSchema schema)
}
```

**合并算法**:
1. 读取基础 schema（嵌入资源 `PkgGenResources.GetProjSchemaStream()`）
2. 找到 `<xs:element name="Components">` 及其内部的 `xs:choice`
3. 对每个插件：
   - 从插件程序集嵌入资源读取 XSD（`plugin.XmlSchemaPath`）
   - 验证 targetNamespace 匹配
   - 将插件 schema 的所有顶层元素添加到基础 schema
   - 在 Components/choice 中添加 `<xs:element ref="ps:XmlElementName" />`
   - 如果插件定义了 `XmlElementUniqueXPath`，添加唯一性约束
4. 调用 `XmlSchemaSet.Reprocess()` 重新编译

### 3.3 Main 函数完整流程

```
Main(args)
│
├─ 1. 日志初始化
│   ├─ LogUtil.IULogTo(LogErrorToBuffer, Warning, Message, Diagnostic)
│   └─ LogUtil.LogCopyright()
│
├─ 2. 命令行解析 (CommandLineParser)
│   ├─ /project    — .pkg.xml 项目文件路径
│   ├─ /config     — 全局变量定义文件
│   ├─ /xsd        — 输出生成的 XSD schema 路径
│   ├─ /output     — 输出目录 (默认 ".")
│   ├─ /version    — 版本号 <major>.<minor>.<qfe>.<build> (默认 "0.0.0.0")
│   ├─ /build      — 构建类型 "fre"|"chk" (默认 "fre")
│   ├─ /cpu        — CPU "X86"|"ARM"|"ARM64"|"AMD64" (默认 "ARM")
│   ├─ /languages  — 语言列表 (分号分隔)
│   ├─ /resolutions— 分辨率列表 (分号分隔)
│   ├─ /variables  — 额外变量 name=value;name=value
│   ├─ /spkgsout   — 输出生成的 SPKG 列表文件
│   ├─ /toolPaths  — 工具搜索路径
│   ├─ /toc        — 仅构建 TOC (bool)
│   ├─ /compress   — 压缩包 (bool)
│   ├─ /diagnostic — 调试输出 (bool)
│   ├─ /nohives    — 无 hive 依赖 (bool)
│   └─ /isRazzleEnv— Razzle 构建环境 (bool)
│
├─ 3. 插件加载
│   └─ LoadPackagePlugins() → Dictionary<string, IPkgPlugin>
│       ├─ AssemblyCatalog(IPkgPlugin.Assembly) — 内置插件
│       └─ DirectoryCatalog(toolDir, "PkgGen.Plugin.*.dll") — 外部插件
│
├─ 4. XSD 合并
│   └─ MergingSchemaValidator.AddSchemaWithPlugins(基础schema, plugins)
│
├─ 5. XSD 输出 (如果 /xsd 指定)
│   └─ WriteXmlSchema(schema, xsdPath) → 仅输出 XSD 后退出
│
├─ 6. 项目文件验证
│   ├─ 必须以 .pkg.xml 结尾
│   └─ LongPath.GetFullPath()
│
├─ 7. 宏解析器初始化
│   ├─ MacroResolver()
│   ├─ 嵌入全局宏 (PkgGenResources.GetGlobalMacroStream())
│   │  或外部宏文件 (/config)
│   ├─ 命令行宏 (/variables)
│   └─ /nohives → 注册 "__nohives" = "true"
│
├─ 8. PackageGenerator 构造
│   └─ new PackageGenerator(
│       plugins, buildPass, cpuId, buildType,
│       versionInfo, macroResolver, schemaValidator,
│       languages, resolutions, toolPaths,
│       isRazzleEnv, compress, spkgsOutPath)
│
├─ 9. 权限提升
│   ├─ ProcessPrivilege.Adjust(BackupPrivilege, enable=true)
│   └─ ProcessPrivilege.Adjust(SecurityPrivilege, enable=true)
│
├─ 10. 构建执行
│   └─ packageGenerator.Build(projectPath, outputDir, compress)
│
├─ 11. 权限恢复
│   ├─ ProcessPrivilege.Adjust(BackupPrivilege, enable=false)
│   └─ ProcessPrivilege.Adjust(SecurityPrivilege, enable=false)
│
└─ 12. 返回 0 (成功)
```

### 3.4 错误处理

三层 catch：
1. `PkgGenException` → 输出 `MessageTrace`，返回 `Marshal.GetHRForException(ex)`
2. `PackageException` → 输出 `MessageTrace`，返回 -1
3. `Exception` → 输出 `ToString()`，返回 -1

所有异常前先调用 `WriteErrorBuffer()` 输出原生组件累积的错误。

### 3.5 命令行宏语法

```regex
^(?<name>[A-Za-z.0-9_{-][A-Za-z.0-9_+{}-]*)=\s*(?<value>.*?)\s*$
```

- 变量名: 字母/数字/点/下划线/大括号/连字符/加号
- 值: 可用双引号包裹
- 分号分隔多个变量
- 重复定义会覆盖（输出 Warning）

---

## 4. ConvertDSM.exe 深度逆向 (Native C++)

### 4.1 基本信息

- **源码路径**: `onecore\base\cbs\mobile\convertdsm\lib\convertdsm.cpp`
- **PE 架构**: i386 (32位原生)
- **大小**: 354,816 字节
- **反编译**: 53,376 行 C 代码
- **依赖**: 标准 C++ 运行时、Win32 API、COM (CoCreateGuid)

### 4.2 核心函数清单

| 函数名 (反编译) | 地址 | 源码行 | 功能 |
|-----------------|------|--------|------|
| `RunConvertDSM` (FUN_00420643) | 0x00420643 | 8388 | wmain — 命令行解析与主入口 |
| `FUN_00418695` | 0x00418695 | 1582 | 打印用法帮助 |
| `ConvertDSM::AddFileEntry` | ~0x0041c4d6 | ~5060 | 生成 CBS manifest `<file>` 元素 |
| `ConvertDSM::UpdateCatAndSign` | ~0x0041b019 | ~4100 | 生成安全目录并签名 |
| `ConvertDSM::RegistriesMap::operator[]` | ~0x00418500 | 1488 | 注册表根键映射 |
| `FUN_0041fd79` | 0x0041fd79 | — | DSM 文件夹模式转换 (3参数) |
| `FUN_00420349` | 0x00420349 | — | SPKG 文件模式转换 (2参数) |
| `GetTempVariablePath` | ~0x004203b4 | 8256 | 获取临时目录路径 (GUID) |
| `TrySetTempPath` | FUN_0042059f | 8307 | 设置临时路径 (TMP 环境变量) |

### 4.3 RunConvertDSM (wmain) 完整分析

**函数签名**: `void __fastcall FUN_00420643(int argc, wchar_t **argv)`

#### 4.3.1 命令行标志解析

所有标志为**单字符**，大小写不敏感，以独立 argv 元素出现（不是 `/c` 或 `-c`，就是 `c`）：

```c
// 标志位映射 (uVar6 / local_74)
case 'c': case 'C': uVar6 |= 0x0001;  // Create cab
case 's': case 'S': uVar6 |= 0x0002;  // Sign output
case 'w': case 'W': uVar6 |= 0x0004;  // Make wow pack
case 'k': case 'K': uVar6 |= 0x0008;  // Skip policy file generation
case 'n': case 'N': uVar6 |= 0x0010;  // Name cab same as SPKG
case 'd': case 'D': uVar6 |= 0x0020;  // Single component (Don't do much)
case 'm': case 'M': uVar6 |= 0x0040;  // Metadata only
case 'r': case 'R': uVar6 |= 0x0100;  // Recall package
case 't': case 'T': uVar6 |= 0x0400;  // ntsign.cmd (requires s)
case 'o': case 'O': uVar6 |= 0x0800;  // Output in same folder as input
case 'f': case 'F': uVar6 |= 0x1000;  // Force test signing
case 'q': case 'Q': local_75 = 1;      // Quiet mode (设置日志回调)
```

**注意**: 位 0x0080 和 0x0200 未使用。

#### 4.3.2 位置参数检测

```c
// 标志解析后，iVar5 = 已解析的标志数量
// 位置参数数量 = argc - iVar5
if (argc != iVar5 + 3 && argc != iVar5 + 2)
    goto print_usage;  // 参数数量不对
```

两种模式：
- **3 位置参数**: `<DSM path> <source path> <output folder>` → DSM 文件夹模式
- **2 位置参数**: `<SPKG path> <output folder>` → SPKG 文件模式

#### 4.3.3 输入类型自动检测

```c
// 检测第一个位置参数是目录还是 .spkg 文件
if (FUN_00427d45(inputPath) == 0) {
    // 是目录 → DSM 模式
    if (FUN_00427cdf(inputPath) != 0) {
        // 也是 .spkg? 不可能
    }
    local_75 = 1;  // DSM 模式标记
} else {
    local_75 = 0;  // SPKG 模式标记
}
```

`FUN_00427d45` = 检查路径是否为目录（GetFileAttributes + FILE_ATTRIBUTE_DIRECTORY）
`FUN_00427cdf` = 检查文件是否以 .spkg 结尾

#### 4.3.4 输出目录处理

```c
if (flags & 0x800) {  // 'o' flag: output in same folder as input
    // 忽略 output folder 参数，使用 input 所在目录
    outputDir = inputDir;
}
```

#### 4.3.5 临时路径设置

```c
TrySetTempPath(&flags);  // FUN_0042059f
// 读取 TMP 环境变量
// 如果失败: Warning "Error getting TMP variable... Falling back to output directory"
// 成功: flags.bit6 = 1, 设置临时路径到 flags+0x38
```

#### 4.3.6 转换执行

```c
if (argc == flags_count + 3) {
    // DSM 模式: <DSM path> <source path> <output folder>
    hr = FUN_0041fd79(dsmPath, sourcePath, &flags);
    // 错误: "Failed to convert packages in '%1' to '%2', error is 0x%3!08X!"
} else {
    // SPKG 模式: <SPKG path> <output folder>
    hr = FUN_00420349(spkgPath, &flags);
    // 错误: 同上
}
```

#### 4.3.7 完成

```c
Log("Conversion complete.");
FUN_00420a15();  // 清理 (释放 ConvertDSM 上下文)
return;
```

### 4.4 用法帮助文本 (完整)

```
ConvertDSM.exe [c] [s] [w] [k] [n] [d] [m] [l] [r] [t] [o] [q] [f] <DSM path> <source path of package> <output folder>
ConvertDSM.exe [c] [s] [w] [k] [n] [d] [m] [l] [r] [t] [o] [q] [f] <SPKG path> <output folder>
     Creates Windows manifest files from the given DSM/SPKG file.
     - c Create cab file.
     - s Sign output.
     - w Make wow pack.
     - k Skip policy file generation.
     - n Name cab same as SPKG.
     - d Put everything in a single component and skip everything else (Don't do much mode).
     - m Only generate metadata (Trust me - it's there mode).
     - l Leave in old SPKG metadata.
     - r Create output as recall package.
     - t Use ntsign.cmd instead of sign.cmd (requires s switch).
     - o Put output in same folder as input. Don't omit <output folder> but it will be ignored.
     - q Quiet mode.
     - f Force test signing.
```

**注意**: 帮助文本中列出了 `l` (Leave in old SPKG metadata) 标志，但在 wmain 的 switch 中**没有找到 `l`/`L` 的 case**。这可能是：
1. 反编译遗漏（switch 跳转表不完整）
2. 该标志在更高层处理（ConvertDSM 构造函数中）
3. 遗留未实现的标志

---

## 5. CBS Manifest 生成机制 (AddFileEntry)

### 5.1 函数概述

`ConvertDSM::AddFileEntry` 是 ConvertDSM 的核心函数之一，负责为每个 payload 文件在 CBS manifest (update.mum) 中生成 `<file>` XML 元素。

**源码位置**: `convertdsm.cpp`, 行 1745-1812+
**反编译行**: ~5060-5305

### 5.2 生成的 XML 结构

```xml
<file name="<destination_path>" destinationPath="<destination_path>" [attributes="archive"] [compress="false"]>
  <asmv2:hash xmlns:asmv2="urn:schemas-microsoft-com:asm.v2">
    <dsig:Transforms xmlns:dsig="http://www.w3.org/2000/09/xmldsig#">
      <dsig:Transform Algorithm="urn:schemas-microsoft-com:HashTransforms.Identity" />
    </dsig:Transforms>
    <dsig:DigestMethod xmlns:dsig="http://www.w3.org/2000/09/xmldsig#"
                       Algorithm="http://www.w3.org/2000/09/xmldsig#sha256" />
    <dsig:DigestValue xmlns:dsig="http://www.w3.org/2000/09/xmldsig#">
      <base64_sha256_hash>
    </dsig:DigestValue>
  </asmv2:hash>
</file>
```

### 5.3 关键算法

#### 5.3.1 SHA256 文件哈希

- **算法**: SHA-256 (DigestMethod = `http://www.w3.org/2000/09/xmldsig#sha256`)
- **变换**: Identity Transform (`urn:schemas-microsoft-com:HashTransforms.Identity`) — 即直接对文件原始字节计算哈希，无预处理
- **编码**: Base64 (DigestValue)
- **计算函数**: `FUN_00419348` (内部封装 CryptHashData / BCryptHash)

#### 5.3.2 FNV-1a 文件名哈希

当目标文件名过长（> 0xFF = 255 字符）时，使用 FNV-1a 哈希生成短文件名：

```c
// FNV-1a 32-bit hash
uint32_t hash = 0x811c9dc5;  // FNV offset basis
for (each byte in wide_char_filename) {
    hash = (byte ^ hash) * 0x01000193;  // FNV prime
}
// 输出: swprintf(buf, 40, L"%u", hash) → 十进制字符串
```

**用途**: 长文件名被截断后，用 FNV-1a 哈希值作为唯一标识，避免冲突。

#### 5.3.3 不压缩文件规则

以下文件**强制不压缩** (`compress="false"`)：

1. **Boot 关键文件** (硬编码列表):
   ```
   bcd
   bootstat.dat
   winload.efi
   winload.efi.mui
   winload.exe
   winload.exe.mui
   winresume.efi
   winresume.efi.mui
   winresume.exe
   winresume.exe.mui
   ```

2. **PlatformManifest 目录下的文件**:
   - 路径前缀: `$(runtime.system32)\PlatformManifest\`

3. **通过 `FUN_00424102` 检测的其他文件**:
   - 该函数检查文件是否已压缩（通过文件头/属性）
   - 如果已压缩，添加 `compress="false"`

#### 5.3.4 文件属性

`attributes` 属性根据 Win32 文件属性生成：

| Win32 属性 | 位掩码 | manifest 属性值 |
|------------|--------|----------------|
| FILE_ATTRIBUTE_ARCHIVE | 0x20 | `archive` |
| FILE_ATTRIBUTE_HIDDEN | 0x02 | `hidden` |
| FILE_ATTRIBUTE_NOT_CONTENT_INDEXED | 0x2000 | `notContentIndexed` |
| FILE_ATTRIBUTE_OFFLINE | 0x1000 | `offline` |

**生成逻辑**:
```c
if (attributes == 0 || attributes == 0x80) {
    // 无属性或仅 NORMAL → 不输出 attributes
} else {
    // 输出 attributes="archive hidden ..."
    // 第一个属性前不加空格，后续属性前加空格
}
```

### 5.4 文件复制/压缩

```c
if (flags.singleComponent || fileType == 2 || fileType == 7 || fileType == 3) {
    // 直接复制 (不压缩)
    CopyFileW(sourcePath, destPath, FALSE);
} else {
    // 压缩复制
    FUN_0042982b(sourcePath, destPath, 0);  // 自定义压缩函数
}
```

`FUN_0042982b` 可能使用 LZX/MSZIP 压缩（CAB 格式标准压缩算法）。

### 5.5 注册表条目

`ConvertDSM::RegistriesMap::operator[]` 将注册表路径映射到根键 HKEY：

```c
// 输入: 注册表路径字符串 (如 "HKLM\Software\...")
// 输出: HKEY 句柄 + 子路径
HKEY rootKey;
if (FUN_00435c25(path, &rootKey, NULL) < 0) {
    // "Failed to GetRootKey for %1"
    return ERROR;
}
// 存储到 map[path] = {rootKey, subPath}
```

`FUN_00435c25` 解析根键前缀（HKLM, HKCU, HKCR, HKU, HKCC）并返回对应 HKEY。

---

## 6. 安全目录生成机制 (UpdateCatAndSign)

### 6.1 函数概述

`ConvertDSM::UpdateCatAndSign` 负责生成 CBS 包的安全目录文件 (update.cat) 并对其签名。

**源码位置**: `convertdsm.cpp`, 行 1587-1695
**反编译行**: ~4100-4543

### 6.2 双工具路径

ConvertDSM 支持两种生成 .cat 的方式，由某个条件标志选择：

#### 路径 A: makecat.exe + .CDF 文件

```
1. 生成 .CDF (Catalog Definition File)
   - 包含: update.mum, deployment manifests, 等
2. 查找 makecat.exe (PATH 搜索)
   - 失败: "Failed to find makecat.exe in PATH."
3. 执行 makecat.exe <cdf_file>
   - 失败: "Failed calling makecat.exe, error is 0x%1!08X!"
4. 输出 update.cat
```

**.CDF 文件格式** (推断):
```
[CatalogHeader]
Name=update.cat
PublicVersion=0x0000001
EncodingType=0x00010001
CATATTR1=0x10010001:OSAttr:2:6.0

[CatalogFiles]
<hash>update.mum=update.mum
<hash>deployment.manifest=deployment.manifest
...
```

#### 路径 B: updcat.exe 命令行

```
1. 查找 updcat.exe (PATH 搜索)
   - 失败: "Failed to find updcat.exe in PATH."

2. 添加 update.mum 到目录:
   updcat.exe "<output>\update.cat" -v 2 -a <hash_alg> "<input>\update.mum"
   - 失败: "Failed to add mum to catalog, error is 0x%1!08X!"

3. 如果有 deployment manifest (非 single component 模式):
   - 生成 deployment manifest (.manifest 文件)
   - 添加到目录:
     updcat.exe "<output>\update.cat" -v 2 -a <hash_alg> "<input>\<id>.manifest"
   - 失败: "Failed to add deployment to catalog, error is 0x%1!08X!"

4. 对每个附加 manifest (循环):
   - 生成 <n>.manifest (n = 0, 1, 2, ...)
   - 如果文件已存在 (FUN_00427cdf != 0):
     updcat.exe "<output>\update.cat" -v 2 -a <hash_alg> "<input>\<n>.manifest"
   - 失败: "Failed to add manifest to catalog, error is 0x%1!08X!"
```

**updcat.exe 命令行参数**:
- `<cat_path>` — 输出 .cat 文件路径
- `-v 2` — 目录版本 2
- `-a <alg>` — 哈希算法 (从 DAT_00404c70 读取，推测为 SHA256)
- `<manifest_path>` — 要添加的 manifest 文件路径

### 6.3 签名流程

```c
// 生成 update.cat 后
if (flags.signCatalog || flags.testSign) {
    signCatalog = (flags & 0x60) != 0;  // 位 0x40/0x20?
    hr = FUN_0041b019(catPath, signCatalog);
    // 失败: "Failed to sign catalog, error is 0x%1!08X!"
}
```

`FUN_0041b019` 是签名函数，可能：
1. 调用 signtool.exe 外部工具
2. 或直接使用 Win32 CryptUI / WinTrust API 内嵌签名
3. 支持测试签名 (`-f` 标志) 和正式签名 (`-s` 标志)
4. `-t` 标志选择 ntsign.cmd 而非 sign.cmd

### 6.4 错误输出格式

所有错误使用统一的日志格式：
```
onecore\base\cbs\mobile\convertdsm\lib\convertdsm.cpp, ConvertDSM::UpdateCatAndSign, line <N>, Error  , <message>
```

行号映射:
| 行号 | 错误 |
|------|------|
| 1587 | Failed to find updcat.exe in PATH |
| 1599 | Failed to add mum to catalog |
| 1601/1602 | Failing updatcat output (mum) |
| 1621 | Failed to add deployment to catalog |
| 1623/1624 | Failing updatcat output (deployment) |
| 1653 | Failed to add manifest to catalog |
| 1655/1656 | Failing updatcat output (manifest) |
| 1667 | Failed to create information for cabbing |
| 1674 | Failed to find makecat.exe in PATH |
| 1683 | Failed calling makecat.exe |
| 1685/1686 | Failing makecat output |
| 1695 | Failed to sign catalog |

---

## 7. 命令行接口完整参考

### 7.1 spkggen.exe

```
spkggen.exe /project:<pkg.xml> [/config:<file>] [/xsd:<path>] [/output:<dir>]
           [/version:<ver>] [/build:<fre|chk>] [/cpu:<X86|ARM|ARM64|AMD64>]
           [/languages:<list>] [/resolutions:<list>] [/variables:<name=value;...>]
           [/spkgsout:<path>] [/toolPaths:<dirs>] [/toc] [/compress]
           [/diagnostic] [/nohives] [/isRazzleEnv]
```

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| /project | string | (必填) | .pkg.xml 项目文件完整路径 |
| /config | string | (嵌入) | 全局变量定义文件 |
| /xsd | string | (无) | 输出生成的 XSD schema 到此路径（仅输出 XSD 后退出） |
| /output | string | "." | 输出目录 |
| /version | string | "0.0.0.0" | 版本号 major.minor.qfe.build |
| /build | enum | "fre" | 构建类型 fre/chk |
| /cpu | enum | "ARM" | CPU 架构 X86/ARM/ARM64/AMD64 |
| /languages | string | "" | 语言列表，分号分隔 |
| /resolutions | string | "" | 分辨率列表，分号分隔 |
| /variables | string | "" | 额外变量 name=value;name=value |
| /spkgsout | string | "" | 输出生成的 SPKG 列表文件 |
| /toolPaths | string | "" | 工具搜索目录 |
| /toc | bool | false | 仅构建 TOC 文件 |
| /compress | bool | false | 压缩生成的包 |
| /diagnostic | bool | false | 启用调试输出 |
| /nohives | bool | false | 无 hive 依赖 |
| /isRazzleEnv | bool | false | Razzle 构建环境 |

### 7.2 ConvertDSM.exe

```
ConvertDSM.exe [c] [s] [w] [k] [n] [d] [m] [l] [r] [t] [o] [q] [f]
               <DSM path> <source path> <output folder>
ConvertDSM.exe [c] [s] [w] [k] [n] [d] [m] [l] [r] [t] [o] [q] [f]
               <SPKG path> <output folder>
```

| 标志 | 位掩码 | 说明 |
|------|--------|------|
| c | 0x0001 | 创建 .cab 文件 |
| s | 0x0002 | 签名输出 |
| w | 0x0004 | 生成 Wow64 包 |
| k | 0x0008 | 跳过策略文件生成 |
| n | 0x0010 | .cab 名称与 .spkg 相同 |
| d | 0x0020 | 单组件模式（不做额外处理） |
| m | 0x0040 | 仅生成元数据 |
| l | (?) | 保留旧 SPKG 元数据（帮助文本列出，wmain 中未确认） |
| r | 0x0100 | 生成 recall 包 |
| t | 0x0400 | 使用 ntsign.cmd 而非 sign.cmd（需 s） |
| o | 0x0800 | 输出到输入所在目录（忽略 output folder） |
| q | (单独) | 安静模式 |
| f | 0x1000 | 强制测试签名 |

---

## 8. 关键数据结构与算法

### 8.1 ConvertDSM 上下文结构 (推断)

从 wmain 和各函数的偏移量引用推断，ConvertDSM 上下文对象（`this` 指针）布局：

| 偏移 | 类型 | 字段 | 说明 |
|------|------|------|------|
| +0x04 | int | flags | 命令行标志位 (uVar6/local_74) |
| +0x06 | byte | tempPathSet | 临时路径已设置标志 |
| +0x08 | wstring | inputPath | 输入路径 |
| +0x20 | wstring | outputPath | 输出路径 |
| +0x38 | wstring | tempPath | 临时目录路径 |
| +0x4c | wstring | packageName | 包名称 |
| +0x64 | wstring | (?) | |
| +0x68 | int | fileType | 文件类型 (2/3/7 等) |
| +0x6c | uint | fileAttributes | Win32 文件属性 |
| +0x7c | int | (?) | |
| +0x9c | HANDLE | logHandle | 日志文件句柄 |
| +0xa4 | wstring | sourcePath | 源路径 (DSM 模式) |
| +0xac | wstring | (?) | |
| +0xc4 | wstring | (?) | |
| +0xdc | wstring | (?) | |
| +0xf4 | wstring | (?) | |
| +0x10c | int | (?) | |
| +0x124 | wstring | (?) | |
| +0x13c | byte | flag_bit0 | (param_7 & 1) |
| +0x13d | byte | signCatalog | 签名目录标志 |
| +0x13e | byte | flag_bit1 | (param_7 >> 1) & 1 |
| +0x13f | byte | flag_bit5 | (param_7 >> 5) & 1 |
| +0x140 | byte | flag_bit8 | (param_7 >> 8) & 1 |
| +0x141 | byte | param_9 | |
| +0x142 | byte | flag_bit9 | (param_7 >> 9) & 1 |
| +0x144 | uint | cabFlags | CAB 创建标志 (0x1fc, 0x3ffc, 0xc000, 0x3f0000) |
| +0x148 | wstring* | (?) | |
| +0x14c | int* | packageInfo | 包信息指针 |
| +0x150 | int | (?) | 循环计数器 |
| +0x154 | HANDLE | manifestFile | manifest 文件句柄 |
| +0x158 | (?) | | |
| +0x1b8 | vector | fileList | 文件列表 |

**注意**: 这是基于反编译偏移量的推断布局，实际结构可能不同。`std::wstring` 在 MSVC 中为 28 字节（SSO 模式），但反编译中显示为 8 字节指针（堆分配模式）。

### 8.2 FNV-1a 哈希实现

```c
uint32_t fnv1a_32(const wchar_t *str, size_t byteLen) {
    uint32_t hash = 0x811c9dc5;  // FNV-1a 32-bit offset basis
    const uint8_t *p = (const uint8_t *)str;
    for (size_t i = 0; i < byteLen; i++) {
        hash ^= p[i];
        hash *= 0x01000193;  // FNV-1a 32-bit prime
    }
    return hash;
}
```

### 8.3 CBS Manifest 文件哈希格式

```
<asmv2:hash xmlns:asmv2="urn:schemas-microsoft-com:asm.v2">
  <dsig:Transforms>
    <dsig:Transform Algorithm="urn:schemas-microsoft-com:HashTransforms.Identity" />
  </dsig:Transforms>
  <dsig:DigestMethod Algorithm="http://www.w3.org/2000/09/xmldsig#sha256" />
  <dsig:DigestValue>[Base64(SHA256(file_contents))]</dsig:DigestValue>
</asmv2:hash>
```

**验证时**:
1. 读取 DigestMethod → 确定算法 (SHA256)
2. 读取 Transform → 确定预处理 (Identity = 无)
3. 对文件原始字节计算 SHA256
4. Base64 编码后与 DigestValue 比较
5. 不匹配 → CBS_E_CORRUPT_FILE (0xc015001b)

---

## 9. 重建项目路线图

### 9.1 已完全逆向的组件

| 组件 | 逆向程度 | 可重建性 |
|------|---------|---------|
| spkggen.exe | 95% (完整 Main + 插件加载) | 高 — .NET 源码清晰 |
| ConvertDSM.exe wmain | 90% (完整命令行解析) | 高 |
| ConvertDSM AddFileEntry | 85% (XML 生成 + hash) | 高 |
| ConvertDSM UpdateCatAndSign | 80% (双工具路径 + 签名) | 中 — 依赖外部 makecat/updcat |
| PkgGen 插件框架 | 已文档化 (见 #09) | 中 |

### 9.2 重建优先级

**Phase 1: 最小可用 pkg2cab 转换器**
1. 实现 spkggen 等价物（.NET 或 Python）：解析 .pkg.xml → 生成 .spkg
2. 实现 ConvertDSM 等价物：.spkg → CBS .cab
   - 生成 update.mum（使用 AddFileEntry 逻辑）
   - 生成 update.cat（使用 makecat.exe 或纯 Python catalog 生成）
   - 打包 .cab（使用 Python cabfile 或 makecab.exe）
3. 集成到 imageapp 构建流水线

**Phase 2: 完整功能**
1. MEF 插件系统（PkgGen.Plugin.*.dll 兼容）
2. 卫星构建（语言/分辨率）
3. Wow64 包
4. 签名支持（测试签名 + 正式签名）
5. 宏解析器（5层宏表）

**Phase 3: 优化与扩展**
1. 并行构建
2. 增量构建
3. 自定义插件 SDK
4. 跨平台支持（Linux/macOS）

### 9.3 关键依赖

重建项目需要以下外部工具（可从 ADK 提取或重新实现）：
- **makecat.exe** — 安全目录生成（可替换为纯 Python 实现）
- **updcat.exe** — 目录更新（可替换为纯 Python 实现）
- **signtool.exe** — 代码签名（可替换为 osslsigncode）
- **makecab.exe** — CAB 打包（Windows 自带，或 Python cabfile）

---

## 10. 源码行号索引

### 10.1 spkggen.cs (`outputsrc\dotnet\spkggen.cs`)

| 行号 | 内容 |
|------|------|
| 1-31 | 程序集元数据 (using, assembly attributes) |
| 32 | namespace 声明 |
| 34-141 | MergingSchemaValidator 类 |
| 41-44 | AddSchemaWithPlugins(Stream) |
| 46-114 | AddSchemaWithPlugins(XmlSchema) — 核心合并算法 |
| 116-126 | SchemaHasElementName |
| 128-140 | GetSchemaComponentsType |
| 142-438 | Program 类 |
| 144 | c_strProjExtension = ".pkg.xml" |
| 148-164 | 错误缓冲 (LogErrorToBuffer, WriteErrorBuffer) |
| 166-207 | LoadPackagePlugins — MEF 插件加载 |
| 209-216 | OnSchemaValidationEvent |
| 218-233 | WriteXmlSchema |
| 235-265 | ImportCommandLineMacros |
| 267-303 | ImportGlobalMacros (两个重载) |
| 305-438 | Main — 完整主入口 |

### 10.2 ConvertDSM.c (`outputsrc\native\ConvertDSM.c`)

| 行号 | 函数/内容 |
|------|----------|
| 1-2 | 文件头 |
| 1480-1497 | ConvertDSM::RegistriesMap::operator[] (行 41) |
| 1582-1612 | FUN_00418695 — 用法帮助打印 |
| 1587-1591 | 两种用法格式 (DSM 3参数 / SPKG 2参数) |
| 1594-1610 | 各标志说明 |
| 4100-4543 | ConvertDSM::UpdateCatAndSign |
| 4233 | 行 1667: Failed to create information for cabbing |
| 4240-4288 | makecat.exe 路径 (生成 .CDF → makecat) |
| 4292-4357 | updcat.exe 路径 (添加 update.mum) |
| 4359-4423 | 添加 deployment manifest |
| 4424-4512 | 循环添加附加 manifest |
| 4514-4533 | 签名 catalog |
| 5060-5305 | ConvertDSM::AddFileEntry |
| 5068 | 行 1745: Failed to get file size |
| 5086-5094 | 长文件名 FNV-1a hash 初始化 |
| 5117-5126 | FNV-1a hash 计算循环 |
| 5131 | 行 1800: File name is too long |
| 5149 | swprintf "%u" — FNV hash 十进制输出 |
| 5233-5252 | Boot 文件不压缩列表 (bcd, winload, winresume) |
| 5258-5271 | PlatformManifest 目录不压缩 |
| 5275-5276 | 文件属性生成 (attributes="...") |
| 5277-5278 | `<file name="..." destinationPath="..."%3%4>` 写入 |
| 5282-5285 | SHA256 hash XML 块写入 (asmv2:hash) |
| 5290 | FUN_0041d105 — 递归添加子文件/注册表 |
| 8250-8283 | GetTempVariablePath — GUID 临时目录 |
| 8307-8347 | TrySetTempPath — TMP 环境变量 |
| 8388-8622 | RunConvertDSM (wmain) — 主入口 |
| 8450-8455 | argc==1 → 打印用法 |
| 8456-8527 | 单字符标志解析循环 (switch) |
| 8529 | 参数数量验证 (3或2位置参数) |
| 8530-8533 | q 标志 → 设置日志回调 |
| 8536-8548 | 输入类型检测 (目录 vs .spkg 文件) |
| 8553-8555 | 清理输入路径 |
| 8560-8568 | DSM 文件夹不存在错误 |
| 8570-8597 | 3参数模式: FUN_0041fd79 (DSM 转换) |
| 8598-8617 | 2参数模式: FUN_00420349 (SPKG 转换) |
| 8618-8621 | "Conversion complete." + 清理 |

---

## 附录 A: 相关文档索引

| 文档 | 内容 |
|------|------|
| `00_INDEX_MASTER.md` | 总索引 (12份文档) |
| `ADK_17704_toolchain_mechanism_deep_dive.md` | 工具链机制总览 |
| `ADK_17704_pkggen_plugin_architecture_deep_dive.md` | PkgGen 插件架构 (MEF, 6种PluginType, 5层宏表) |
| `ADK_17704_cab_hash_cert_deep_dive.md` | CAB hash/证书验证 (wcp.dll VerifyFileHashes) |
| `ADK_17704_updatedll_deep_dive.md` | UpdateDLL.dll (PrepareUpdate/ExecuteUpdate, hash验证调用链) |
| `ADK_17704_imaging_dll_deep_dive.md` | imaging.dll (ProcessImage 全流程) |

## 附录 B: 未覆盖区域

以下区域尚未深入逆向，建议后续研究：
1. `FUN_0041fd79` — DSM 文件夹模式转换核心（3参数路径）
2. `FUN_00420349` — SPKG 文件模式转换核心（2参数路径）
3. `FUN_0041d105` — AddFileEntry 中的递归子文件/注册表添加
4. `FUN_0041b019` — catalog 签名函数内部实现
5. `FUN_0042982b` — 文件压缩函数（LZX/MSZIP?）
6. `FUN_00419348` — SHA256 文件哈希计算内部
7. ConvertDSMDLL.c — DLL 导出函数（1.4MB，未分析）
8. `l` 标志的实际处理位置（帮助文本列出但 wmain switch 中未找到）

---

*文档结束 — 生成于 2026-08-27，基于 ADK 10.0.17704.1000 反编译源码*
