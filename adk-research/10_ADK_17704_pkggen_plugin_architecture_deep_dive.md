# ADK 10.0.17704.1000 — PkgGen 插件架构与包构建流水线深度研究

> **研究对象**: PkgGen.exe (28160B), spkggen.exe (19968B), ConvertDSM.exe (354816B)
> **源码路径**:
> - `E:\WSK_Tools\ADK_Research\outputsrc\dotnet\PkgGen.cs` (35KB)
> - `E:\WSK_Tools\ADK_Research\outputsrc\dotnet\PkgBldr.Common.cs` (192KB)
> - `E:\WSK_Tools\ADK_Research\outputsrc\dotnet\pkggencommon.cs` (208KB)
> - `E:\WSK_Tools\ADK_Research\outputsrc\dotnet\pkgtoolbox.cs` (253KB)
> **命名空间**: `Microsoft.CompPlat.PkgBldr`
> **程序集版本**: 10.0.10011.16384 (文件), 10.0.0.0 (程序集)
> **.NET 版本**: .NET Framework 4.0

---

## 1. 概述

PkgGen 是 ADK 包构建流水线的**前端编排器**，负责将高层包定义（.pkg.xml）经过多级 XML 转换，最终生成可被 imageapp/UpdateDLL 消费的 CBS 包（.cab）。

PkgGen 本身**不直接生成 .cab**，而是编排三个阶段的工具链：

```
.pkg.xml  ──[PkgGen: PkgToWm]──>  .wm.xml  ──[PkgGen: WmToCsi]──>  CSI manifest  ──[spkggen]──>  .spkg  ──[ConvertDSM]──>  .cab
   (高层)          (MEF插件转换)       (WM格式)       (MEF插件转换)       (asm.v3清单)      (原生工具)     (SPKG)     (原生工具)    (CBS包)
```

### 1.1 三种构建模式

| 模式 | 输入 | 输出 | 用途 |
|------|------|------|------|
| **完整构建** (pkg2cab) | .pkg.xml | .cab | 生产环境，完整流水线 |
| **仅转换** (pkg2wm / wm2csi / csi2wm) | 一种XML | 另一种XML | 调试中间格式 |
| **TestPkg / UniversalBsp** | .wm.xml | .cab | 测试包 / 通用BSP包，跳过pkg2wm阶段 |

---

## 2. 插件架构（MEF）

PkgGen 使用 **MEF (Managed Extensibility Framework)** 实现插件化架构。每个 XML 元素类型对应一个插件类，通过 `[Export(typeof(IPkgPlugin))]` 标记导出。

### 2.1 IPkgPlugin 接口

```csharp
public interface IPkgPlugin
{
    string Name { get; }
    string XmlElementName { get; }
    string XmlElementUniqueXPath { get; }
    string XmlSchemaPath { get; }
    string XmlSchemaNameSpace { get; }

    // 核心转换方法：将 source 元素转换并添加到 parent
    void ConvertEntries(XElement parent, Dictionary<string, IPkgPlugin> plugins,
                        Config enviorn, XElement component);

    bool Pass(BuildPass pass);
    bool SchemaNeedsTrimming(PkgBldrCmd pkgBldrArgs);
    List<TrimElementList> ElementsToTrim(PkgBldrCmd pkgBldrArgs);
    List<TrimAttributeList> AttributesToTrim(PkgBldrCmd pkgBldrArgs);
    XDocument TrimSchema(XDocument xsdDoc, PkgBldrCmd pkgBldrArgs);
}
```

### 2.2 PkgPlugin 抽象基类

```csharp
public abstract class PkgPlugin : IPkgPlugin
{
    internal static string BaseComponentSchemaPath = "BasePlugins.xsd";

    public virtual string Name => XmlElementName;
    public virtual string XmlElementName => GetType().Name;  // 默认用类名
    public virtual string XmlElementUniqueXPath => null;
    public abstract string XmlSchemaPath { get; }
    public virtual bool UseSecurityCompilerPassthrough => false;

    // 默认递归转换子元素
    public virtual void ConvertEntries(XElement parent, Dictionary<string, IPkgPlugin> plugins,
                                        Config enviorn, XElement component)
    {
        foreach (XElement child in component.Elements())
        {
            if (plugins.TryGetValue(child.Name.LocalName, out IPkgPlugin plugin))
            {
                plugin.ConvertEntries(parent, plugins, enviorn, child);
            }
        }
    }
}
```

### 2.3 PluginType 枚举

```csharp
public enum PluginType
{
    WmToCsi,    // WM清单 → CSI清单 (urn:schemas-microsoft-com:asm.v3)
    CsiToWm,    // CSI清单 → WM清单
    CsiToCsi,   // CSI清单后处理（规范化、优化）
    PkgToWm,    // .pkg.xml → WM清单
    CsiToCab,   // CSI清单 → CAB包（TestPkg模式）
    PkgFilter    // Pkg XML过滤器（Wow64 host/guest分离）
}
```

### 2.4 PkgBldrLoader 插件加载器

```csharp
public class PkgBldrLoader
{
    private Dictionary<string, IPkgPlugin> m_plugins;      // 主插件字典（按XmlElementName索引）
    private Dictionary<string, IPkgPlugin> m_wmPlugins;    // WM插件（用于输入验证）
    private PluginType m_pluginType;
    private SchemaSet m_csiSchemaValidator;
    private SchemaSet m_wmSchemaValidator;
    private SchemaSet m_pkgSchemaValidator;

    // 嵌入资源路径
    private string m_csiXsdPath = "PkgBldr.CSI.Xsd";
    private string m_sharedXsdPath = "PkgBldr.Shared.Xsd";
    private string m_pkgXsdPath = "PkgBldr.PKG.Xsd";
    private string[] m_embeddedSchemaNames;  // 从程序集嵌入资源加载

    public PkgBldrLoader(PluginType pluginType, PkgBldrCmd pkgBldrArgs, IDeploymentLogger logger = null)
    {
        // 1. 扫描程序集嵌入资源中的XSD
        m_embeddedSchemaNames = Assembly.GetExecutingAssembly().GetManifestResourceNames()
            .Where(r => r.Contains("Schemas")).ToArray();

        // 2. 加载MEF插件
        m_pluginType = pluginType;
        LoadPlugins();
    }

    private void LoadPlugins()
    {
        m_plugins = LoadPackagePlugins(m_pluginType);  // MEF组合

        // WmToCsi模式同时加载WM插件用于输入验证
        if (m_pluginType != PluginType.PkgFilter)
        {
            m_wmPlugins = (m_pluginType == PluginType.WmToCsi)
                ? m_plugins
                : LoadPackagePlugins(PluginType.WmToCsi);
        }

        // 按插件类型加载对应的XSD SchemaSet
        switch (m_pluginType) { /* 加载CSI/WM/PKG schema */ }
    }

    // 输入/输出验证
    public void ValidateInput(XDocument doc)  => m_wmSchemaValidator.Validate(doc);
    public void ValidateOutput(XDocument doc) => m_csiSchemaValidator.Validate(doc);
}
```

### 2.5 典型插件示例：Directory（WmToCsi）

```csharp
[Export(typeof(IPkgPlugin))]
internal class Directory : PkgPlugin
{
    public override string XmlSchemaPath => "PkgBldr.Shared.Xsd\\Directory.xsd";

    public override void ConvertEntries(XElement ToCsi, Dictionary<string, IPkgPlugin> plugins,
                                         Config enviorn, XElement FromWm)
    {
        XElement dir = new XElement(ToCsi.Name.Namespace + "directory");
        foreach (XAttribute attr in FromWm.Attributes())
        {
            switch (attr.Name.LocalName)
            {
                case "path":
                    dir.Add(new XAttribute("destinationPath",
                        enviorn.Macros.Resolve(attr.Value).TrimEnd('\\')));
                    break;
                case "securityDescriptor":
                    // 转换为子元素 <securityDescriptor name="..."/>
                    XElement sd = new XElement(ToCsi.Name.Namespace + "securityDescriptor");
                    sd.Add(new XAttribute("name", enviorn.Macros.Resolve(attr.Value)));
                    dir.Add(sd);
                    break;
                case "attributes":
                    dir.Add(new XAttribute("attributes", enviorn.Macros.Resolve(attr.Value)));
                    break;
                case "owner":
                    dir.Add(new XAttribute("owner", attr.Value));
                    break;
                case "buildFilter":
                    dir.Add(new XAttribute("buildFilter", attr.Value));
                    break;
            }
        }
        ToCsi.Add(dir);
    }
}
```

### 2.6 已知插件列表（从 PkgBldr.Common.cs 提取）

**WmToCsi 插件**（命名空间 `Microsoft.CompPlat.PkgBldr.Plugins.WmToCsi`）:

| 插件类名 | 对应XML元素 | 功能 |
|----------|------------|------|
| `Memberships` | memberships | 组成员关系 |
| `CategoryMembership` | categoryMembership | 类别成员 |
| `Id` | id | 身份标识（name/publicKeyToken/typeName/version） |
| `COM` | COM | COM组件（含GUID宏解析、Schema裁剪） |
| `Configuration` | configuration | 配置（直接透传+命名空间替换） |
| `Directories` | directories | 目录集合 |
| `Directory` | directory | 单个目录（path→destinationPath, securityDescriptor转换） |
| ... 更多 | ... | 文件、注册表、服务、驱动、能力等 |

---

## 3. 宏系统（MacroResolver）

### 3.1 宏表来源

PkgGen 使用**多层宏表**，按优先级从低到高：

1. **嵌入资源宏表**（编译进程序集）:
   - `Macros_PkgToWm.xml` — Pkg→WM 转换用宏
   - `Macros_WmToCsi.xml` — WM→CSI 转换用宏
   - `Macros_CsiToWm.xml` — CSI→WM 转换用宏
   - `Macros_CsiToCmi.xml` — CSI→CMI（TestPkg模式）用宏
   - `Macros_Policy.xml` — 策略宏

2. **配置文件宏** (`/config:` 参数): 从外部XML导入
3. **命令行变量** (`/variables:` 参数): `NAME=VALUE;NAME=VALUE` 格式
4. **卫星宏**: `langid`（语言）、`resId`（分辨率）

### 3.2 宏解析示例

```csharp
// Directory插件中解析路径宏
string resolvedPath = enviorn.Macros.Resolve(item.Value);
// 输入: "$(runtime.system32)\\drivers"
// 输出: "\\windows\\system32\\drivers"
```

### 3.3 CsiToCmi 宏表中的关键宏

```csharp
macroResolver.Register("build.buildType", "release");
macroResolver.Register("build.arch", enviorn.Arch);           // x86/amd64/arm
macroResolver.Register("build.WindowsPublicKeyToken", "31bf3856ad364e35");
macroResolver.Register("build.version", enviorn.Version);      // 10.0.17704.1000
macroResolver.Register("build.nttree", Environment.ExpandEnvironmentVariables("%build.nttree%"));
```

### 3.4 命令行宏解析

```csharp
// 正则: ^(?<name>[A-Za-z.0-9_{-][A-Za-z.0-9_+{}-]*)=\s*(?<value>.*?)\s*$
// 示例: BUILD_OS_VERSION=10.0.17704.1000;MY_VAR=hello
```

---

## 4. BuildPackage 核心流程

### 4.1 入口

```csharp
public static void BuildPackage(Config config, MacroResolver commandLineMacros)
{
    // 1. 创建临时工作目录
    config.WorkingDirectory = Path.Combine(Path.GetTempPath(), Path.GetRandomFileName());
    Directory.CreateDirectory(config.WorkingDirectory);

    // 2. 按转换类型分发
    switch (config.pkgBldrArgs.convert)
    {
        case ConversionType.csi2wm:  ConvertCsiToWm(config, commandLineMacros); break;
        case ConversionType.pkg2wm:  ConvertPkgToWm(config, commandLineMacros); break;
        case ConversionType.wm2csi:  ConvertWmToCsi(config, commandLineMacros); break;
    }

    // 3. 清理临时目录
    Directory.Delete(config.WorkingDirectory, recursive: true);
}
```

### 4.2 WmToCsi 转换流程（最核心）

```csharp
// 1. 加载WmToCsi插件（单例缓存）
if (m_wmToCsiLoader == null)
    m_wmToCsiLoader = new PkgBldrLoader(PluginType.WmToCsi, config.pkgBldrArgs);

// 2. 加载输入WM XML并验证
XDocument input = PkgBldrHelpers.XDocumentLoadFromLongPath(config.Input);
m_wmToCsiLoader.ValidateInput(input);
config.WmXmlRoot = input.Root;

// 3. 创建空CSI根元素
// <assembly xmlns="urn:schemas-microsoft-com:asm.v3" manifestVersion="1.0"/>
config.CsiXmlRoot = PkgBldrHelpers.CreateEmptyCsiXmlElement();

// 4. 加载宏表
LoadWm2CsiMacroTables(config, commandLineMacros);

// 5. 初始化BuildFilter表达式计算器
InitializeBuildFilterEvaluator(config);
// 设置变量: build.arch, build.product, build.isWow
// 语言/分辨率卫星变量

// 6. 初始化全局安全对象
config.GlobalSecurity = new GlobalSecurity();

// 7. 执行转换（递归插件调度）
config.ExitStatus = ExitStatus.SUCCESS;
m_wmToCsiLoader.Plugins[config.WmXmlRoot.Name.LocalName]
    .ConvertEntries(config.CsiXmlRoot, m_wmToCsiLoader.Plugins, config, config.WmXmlRoot);

// 8. 后处理：CsiToCsi规范化
if (config.ExitStatus != ExitStatus.SKIPPED)
{
    m_wmToCsiLoader.ValidateOutput(new XDocument(config.CsiXmlRoot));

    if (m_csiToCsiLoader == null)
        m_csiToCsiLoader = new PkgBldrLoader(PluginType.CsiToCsi, config.pkgBldrArgs);

    m_csiToCsiLoader.Plugins[config.CsiXmlRoot.Name.LocalName]
        .ConvertEntries(config.CsiXmlRoot, m_csiToCsiLoader.Plugins, config, config.CsiXmlRoot);

    XDocument finalCsi = new XDocument(config.CsiXmlRoot);
    m_csiToCsiLoader.ValidateOutput(finalCsi);

    // 9. TestPkg/UniversalBsp模式：直接生成CAB
    if (config.TestPkg || config.UniversalBsp)
    {
        LoadCsi2CmiMacros(config);
        PkgBldrLoader cabLoader = new PkgBldrLoader(PluginType.CsiToCab, config.pkgBldrArgs);
        cabLoader.Plugins[config.CsiXmlRoot.Name.LocalName]
            .ConvertEntries(config.CsiXmlRoot, cabLoader.Plugins, config, config.CsiXmlRoot);
    }
    else
    {
        // 普通模式：保存CSI manifest供spkggen使用
        PkgBldrHelpers.XDocumentSaveToLongPath(finalCsi, config.Output);
    }
}
```

---

## 5. PkgToCab 完整流水线（pkg2cab 模式）

### 5.1 流程总览

```
PkgGen.Main()
  ├─ 解析命令行参数 (PkgBldrCmd)
  ├─ 构建spkggen参数列表 (GetSpkgGenArguments)
  ├─ PkgToCab(config, spkgGenArgs, testSignOnly)
  │   ├─ [可选] Wow64过滤: BuildWow() → FilterPkgXml()
  │   │   └─ PkgFilter插件分离host/guest文件
  │   ├─ Run.RunSPkgGen(spkgGenArgs)  ← 调用spkggen.exe
  │   │   └─ 生成 .spkg 文件列表
  │   └─ RunConvertDsmInParellel(spkgList)  ← 并行调用ConvertDSM.exe
  │       └─ 每个.spkg → .cab
  └─ 返回0(成功)/-1(失败)
```

### 5.2 spkggen 参数构建

```csharp
private static List<string> GetSpkgGenArguments(PkgBldrCmd args, string versionOverride)
{
    List<string> list = new List<string>();
    if (!string.IsNullOrEmpty(args.project))    list.Add(AddQuotes(args.project));
    if (!string.IsNullOrEmpty(args.config))     list.Add($"/config:{AddQuotes(args.config)}");
    if (!string.IsNullOrEmpty(args.xsd))        list.Add($"/xsd:{AddQuotes(args.xsd)}");
    if (!string.IsNullOrEmpty(args.output))     list.Add($"/output:{AddQuotes(args.output)}");

    list.Add($"/build:{(args.build == BuildType.chk ? "chk" : "fre")}");
    list.Add($"/cpu:{args.cpu}");
    if (!string.IsNullOrEmpty(args.languages))   list.Add($"/languages:{AddQuotes(args.languages)}");
    if (!string.IsNullOrEmpty(args.resolutions)) list.Add($"/resolutions:{AddQuotes(args.resolutions)}");
    if (!string.IsNullOrEmpty(args.variables))   list.Add($"/variables:{AddQuotes(args.variables)}");
    if (!string.IsNullOrEmpty(args.spkgGenToolDirs)) list.Add($"/toolPaths:{AddQuotes(args.spkgGenToolDirs)}");

    if (args.toc)         list.Add("/toc");
    if (args.compress)    list.Add("/compress");
    if (args.diagnostic)  list.Add("/diagnostic");
    if (args.nohives || args.onecore) list.Add("/nohives");
    if (args.isRazzleEnv) list.Add("/isRazzleEnv");

    list.Add($"/version:{versionOverride}");
    return list;
}
```

### 5.3 Wow64 交叉编译支持

当 `.pkg.xml` 中 `BuildWow="yes"` 时，PkgGen 执行 host+guest 双构建：

```csharp
foreach (WowType wowType in Enum.GetValues(typeof(WowType)))
{
    config.IsGuest = (wowType == WowType.guest);

    // 过滤条件：amd64/arm64才支持guest；wowbuild参数控制
    if ((config.Arch != "amd64" && config.Arch != "arm64") || !config.IsGuest) continue;
    if (config.pkgBldrArgs.wowbuild == WowBuildType.HostOnly && config.IsGuest) continue;
    if (config.pkgBldrArgs.wowbuild == WowBuildType.GuestOnly && !config.IsGuest) continue;

    // 1. PkgFilter插件分离host/guest文件
    XElement filtered = FilterPkgXml(unfiltered, config);

    // 2. 保存过滤后的.pkg.xml到临时文件
    string tempFile = Path.GetTempFileName() + (config.IsGuest ? ".guest.pkg.xml" : ".host.pkg.xml");

    // 3. 替换spkggen输入为过滤后的文件
    ChangeSpkgGenInput(spkgGenArgs, tempFile);

    // 4. guest模式重定向输出到临时目录
    if (config.IsGuest) RedirectOutput(spkgGenArgs, tempDirectory);

    // 5. 运行spkggen
    Run.RunSPkgGen(spkgGenArgs, ...);

    // 6. 并行运行ConvertDSM
    RunConvertDsmInParellel(spkgList, testSignOnly, convertDsmThreadCount,
                             isGuest: config.IsGuest, tempDirectory, wowDir);
}
```

### 5.4 并行 ConvertDSM

```csharp
private static void RunConvertDsmInParellel(List<string> spkgList, bool testSignOnly,
                                               int convertDsmThreadCount, bool isGuest = false, ...)
{
    ParallelOptions options = new ParallelOptions { MaxDegreeOfParallelism = convertDsmThreadCount };
    Parallel.ForEach(spkgList, options, spkg =>
    {
        Run.RunDsmConverter(spkg, wow: isGuest, ignoreConvertDsmError: false, testSignOnly);
        // 输出: spkg.Replace(".spkg", ".cab")
    });
}
```

---

## 6. 卫星构建（Satellite Builds）

PkgGen 支持为不同语言和分辨率生成**卫星包**：

### 6.1 SatelliteType 枚举

```csharp
public enum SatelliteType { Neutral, Language, Resolution }
```

### 6.2 构建循环

```csharp
// 对每个卫星类型（Neutral + 每种语言 + 每种分辨率）
foreach (SatelliteId satellite in satelliteList)
{
    config.Satellite = satellite;

    // 语言卫星：注册 langid 宏
    if (satellite.Type == SatelliteType.Language)
        macroResolver.Register("langid", satellite.Culture, allowOverride: true);

    // 对每个Wow类型
    foreach (WowType wowType in Enum.GetValues(typeof(WowType)))
    {
        config.IsGuest = (wowType == WowType.guest);
        BuildPackage(config, macroResolver);
    }
}

// 分辨率卫星（仅host，无guest）
foreach (SatelliteId res in config.AllResolutions)
{
    config.Satellite = res;
    config.IsGuest = false;
    BuildPackage(config, macroResolver);
}
```

### 6.3 文件后缀规则

| 卫星类型 | 文件后缀 | 宏名 |
|----------|---------|------|
| Neutral | (无) | - |
| Language | `lang_<culture>` | `LANGID` / `$(LANGID)` |
| Resolution | `res_<resolution>` | `RESID` / `$(RESID)` |

---

## 7. BuildFilter 表达式计算器

### 7.1 用途

`BuildFilterExpressionEvaluator` 用于根据构建配置条件性地包含/排除 XML 元素。

### 7.2 变量设置

```csharp
private static void InitializeBuildFilterEvaluator(Config env)
{
    env.BuildFilterEvaluator = new BuildFilterExpressionEvaluator();
    env.CultureEvaluator = new BuildFilterExpressionEvaluator();
    env.ResolutionEvaluator = new BuildFilterExpressionEvaluator();

    // 构建变量
    env.BuildFilterEvaluator.SetVariable("build.arch", env.Arch);      // x86/amd64/arm
    env.BuildFilterEvaluator.SetVariable("build.product", env.Product);
    env.BuildFilterEvaluator.SetVariable("build.isWow", env.IsGuest);

    // 语言变量（所有语言默认false，当前卫星语言设为true）
    foreach (SatelliteId lang in env.AllLanguages)
        env.CultureEvaluator.SetVariable(lang.Culture, false);
    if (env.Satellite.Type == SatelliteType.Language)
        env.CultureEvaluator.SetVariable(env.Satellite.Culture, true);

    // 分辨率变量
    foreach (SatelliteId res in env.AllResolutions)
        env.ResolutionEvaluator.SetVariable(res.Resolution, false);
    if (env.Satellite.Type == SatelliteType.Resolution)
        env.ResolutionEvaluator.SetVariable(env.Satellite.Resolution, true);
}
```

### 7.3 使用方式

XML 元素通过 `buildFilter` 属性指定条件：

```xml
<file buildFilter="build.arch=amd64" destinationPath="..." importPath="..."/>
<registryKey buildFilter="langid=en-US" keyName="..."/>
```

---

## 8. XML 命名空间与格式

### 8.1 三种清单格式

| 格式 | 根元素 | 命名空间 | 用途 |
|------|--------|---------|------|
| **PKG** | `<Package>` | 自定义 | 高层包定义（.pkg.xml） |
| **WM** | `<identity>` | `urn:Microsoft.CompPlat/ManifestSchema.v1.00` | Windows Mobile 中间格式（.wm.xml） |
| **CSI** | `<assembly>` | `urn:schemas-microsoft-com:asm.v3` | CBS 组件清单（最终输入spkggen） |

### 8.2 CSI 根元素模板

```csharp
public static XElement CreateEmptyCsiXmlElement()
{
    XElement root = new XElement("assembly");
    XNamespace ns = "urn:schemas-microsoft-com:asm.v3";
    root.Name = ns + root.Name.LocalName;
    root.Add(new XAttribute("xmlns", "urn:schemas-microsoft-com:asm.v3"));
    root.Add(new XAttribute(XNamespace.Xmlns + "xsd", "http://www.w3.org/2001/XMLSchema"));
    root.Add(new XAttribute(XNamespace.Xmlns + "xsi", "http://www.w3.org/2001/XMLSchema-instance"));
    root.Add(new XAttribute("manifestVersion", "1.0"));
    return root;
}
```

### 8.3 WM 根元素模板

```csharp
public static XElement CreateEmptyWmXmlElement()
{
    XElement root = new XElement("identity");
    XNamespace ns = "urn:Microsoft.CompPlat/ManifestSchema.v1.00";
    root.Name = ns + root.Name.LocalName;
    root.Add(new XAttribute(XNamespace.Xmlns + "xsd", "http://www.w3.org/2001/XMLSchema"));
    root.Add(new XAttribute(XNamespace.Xmlns + "xsi", "http://www.w3.org/2001/XMLSchema-instance"));
    return root;
}
```

---

## 9. 长路径支持

PkgGen 完整支持 Windows 长路径（>260字符），通过直接调用 Win32 `CreateFile` API 绕过 .NET Framework 的路径长度限制：

```csharp
[DllImport("kernel32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
private static extern SafeFileHandle CreateFile(string fileName, uint desiredAccess,
    uint shareMode, IntPtr securityAttributes, uint creationDisposition,
    uint flagsAndAttributes, IntPtr templateFile);

public static XDocument XDocumentLoadFromLongPath(string path)
{
    SafeFileHandle handle = CreateFile(path, 1u, 3u, IntPtr.Zero, 3u, 0u, IntPtr.Zero);
    FileStream stream = new FileStream(handle, FileAccess.Read);
    XDocument result = XDocument.Load(stream);
    handle.Close();
    return result;
}
```

---

## 10. Schema 验证与裁剪

### 10.1 SchemaSet 类

每个 `PkgBldrLoader` 实例维护一个或多个 `SchemaSet`，用于输入/输出 XML 的 XSD 验证。

### 10.2 插件级 Schema 裁剪

部分插件（如 `COM`、`Directory`）支持根据构建模式裁剪 XSD schema，移除不需要的元素/属性：

```csharp
public override bool SchemaNeedsTrimming(PkgBldrCmd args)
{
    // testpkg / universalbsp / foroempkg / fortestpkg 模式下需要裁剪
    if (!args.testpkg && !args.universalbsp && !args.foroempkg)
        return args.fortestpkg;
    return true;
}

public override List<TrimElementList> ElementsToTrim(PkgBldrCmd args)
{
    return new List<TrimElementList>
    {
        new TrimElementList("CT_COM", new List<string> { "securityDescriptor", "clients" }),
        new TrimElementList("CT_Interfaces", new List<string> { "typeLib", "securityDescriptor" }),
        // ...
    };
}
```

---

## 11. 命令行参数（PkgBldrCmd）

### 11.1 关键参数

| 参数 | 说明 |
|------|------|
| `/project:<path>` | 输入 .pkg.xml 或 .wm.xml |
| `/output:<path>` | 输出文件路径 |
| `/config:<path>` | 宏配置 XML |
| `/xsd:<path>` | XSD schema 目录 |
| `/convert:<type>` | 转换类型: pkg2wm/wm2csi/csi2wm/pkg2cab |
| `/build:<type>` | fre/chk |
| `/cpu:<arch>` | x86/amd64/arm/arm64 |
| `/version:<ver>` | 版本号（如 10.0.17704.1000） |
| `/languages:<list>` | 语言列表（分号分隔） |
| `/resolutions:<list>` | 分辨率列表 |
| `/variables:<list>` | 宏变量（NAME=VALUE;...） |
| `/compress` | 压缩输出 |
| `/toc` | 生成目录表 |
| `/nohives` | 不生成注册表 hive（OneCore模式） |
| `/onecore` | OneCore 模式 |
| `/testpkg` | 测试包模式（直接从.wm.xml生成.cab） |
| `/universalbsp` | 通用BSP模式 |
| `/fortestpkg` | 为测试包转换（pkg2wm模式） |
| `/foroempkg` | 为OEM包转换（pkg2wm模式） |
| `/wowbuild:<type>` | Wow构建类型: HostOnly/GuestOnly/Both |
| `/wowdir:<path>` | Wow guest输出目录 |
| `/ConvertDsmThreadCount:<n>` | ConvertDSM并行线程数 |
| `/quiet` | 静默模式（仅警告） |
| `/diagnostic` | 诊断模式（调试日志） |
| `/wmxsd:<dir>` | 导出WM/CSI XSD schema到目录 |
| `/useLegacyName` | 使用旧命名规范（仅pkg2wm） |
| `/json:<path>` | JSON depot（SD集成） |
| `/objdir:<path>` | 中间对象目录 |
| `/toolPaths:<path>` | 工具路径（spkggen/ConvertDSM位置） |
| `/isRazzleEnv` | 在Razzle构建环境中运行 |
| `/razzleToolPath:<path>` | Razzle工具路径（用于version.pl） |
| `/usentverp` | 使用NTVERP版本（从version.pl获取） |

### 11.2 参数互斥约束

```csharp
if (args.testpkg && args.universalbsp)
    throw new PkgGenException("/testpkg and /universalbsp switches may not be given together");
if (args.fortestpkg && args.foroempkg)
    throw new PkgGenException("/fortestpkg and /foroempkg switches may not be given together");
if (args.testpkg && args.convert != ConversionType.pkg2cab)
    throw new PkgGenException("/testpkg must not be given with any /convert option");
if (args.fortestpkg && args.convert != ConversionType.pkg2wm)
    throw new PkgGenException("/fortestpkg can only be given with /convert:pkg2wm");
```

---

## 12. 版本处理

### 12.1 版本格式验证

```csharp
private static void CheckVersion(Config config)
{
    // 必须匹配 N.N.N.N 格式
    if (!Regex.Match(args.version, @"^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$").Success)
        throw new PkgGenException("Input version '{0}' is not correctly formatted.", args.version);

    // 版本各段之和不能为0（否则包无法安装）
    if (args.version.Split('.').Sum(x => int.Parse(x)) == 0)
        throw new PkgGenException("Version '{0}' can't be zero or package will fail to install", args.version);
}
```

### 12.2 NTVERP 版本获取

```csharp
// /usentverp 参数：从Razzle环境的version.pl获取版本号
private static string GetNtVerpVersion(Config config)
{
    string workingDir = Environment.ExpandEnvironmentVariables(args.razzleToolPath);
    string perl = Environment.ExpandEnvironmentVariables(args.toolPaths["perl"]);
    string output = Run.RunProcess(workingDir, perl, "version.pl", logger);
    return Regex.Match(output, @"[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+").Value;
}
```

---

## 13. 重建要点

### 13.1 必须保留的机制

1. **MEF插件架构**: 每个XML元素类型对应一个插件，通过 `[Export(typeof(IPkgPlugin))]` 注册
2. **多级转换流水线**: Pkg→WM→CSI→SPKG→CAB，每级独立可调试
3. **宏系统**: 嵌入资源宏表 + 配置文件宏 + 命令行宏 + 卫星宏，支持 `$(name)` 语法
4. **BuildFilter条件编译**: 基于 arch/product/language/resolution 的条件包含
5. **Wow64双构建**: host/guest文件分离，PkgFilter插件过滤
6. **并行ConvertDSM**: 多线程.spkg→.cab转换
7. **长路径支持**: 直接Win32 API绕过MAX_PATH限制
8. **Schema验证**: 输入输出双向XSD验证，支持插件级裁剪
9. **卫星包**: 语言/分辨率独立构建，文件后缀规范

### 13.2 可简化的部分

1. **SD集成**: `SdCommand.Run("edit"/"add")` 仅用于微软内部Source Depot，可移除
2. **Razzle环境**: `/isRazzleEnv`、`version.pl`、NTVERP版本获取，外部构建不需要
3. **JSON Depot**: `/json` 参数用于内部构建追踪
4. **多种构建类型**: fre/chk 区分在现代构建中可简化

### 13.3 关键依赖

| 依赖 | 用途 | 可替代方案 |
|------|------|-----------|
| spkggen.exe | CSI manifest → .spkg | 需逆向或重新实现（原生工具） |
| ConvertDSM.exe | .spkg → .cab | 需逆向或重新实现（原生工具，354KB） |
| .NET Framework 4.0 | 运行时 | .NET 6+/Core（需移植MEF） |
| System.ComponentModel.Composition | MEF框架 | 内置DI容器或手动反射 |

### 13.4 spkggen 和 ConvertDSM 的角色

- **spkggen.exe** (19968B): 小型原生工具，将 CSI manifest (urn:schemas-microsoft-com:asm.v3) 编译为 .spkg（Signed Package）格式，包含文件、注册表、安全描述符等的二进制序列化
- **ConvertDSM.exe** (354816B): 大型原生工具，将 .spkg 转换为标准 CBS .cab 包，生成 update.mum（包清单）、update.cat（安全目录）、并打包所有 payload 文件。**这是实际生成 .cab 的工具**，包含完整的 CBS 包结构生成逻辑

---

## 14. 源码行号索引

| 功能 | 文件 | 行号 |
|------|------|------|
| Program.Main 入口 | PkgGen.cs | 397 |
| BuildPackage | PkgGen.cs | 70 |
| PkgToCab | PkgGen.cs | 287 |
| RunConvertDsmInParellel | PkgGen.cs | 347 |
| GetSpkgGenArguments | PkgGen.cs | 754 |
| FilterPkgXml | PkgGen.cs | 733 |
| InitializeBuildFilterEvaluator | PkgGen.cs | 176 |
| LoadPkg2WmMacroTables | PkgGen.cs | 202 |
| LoadWm2CsiMacroTables | PkgGen.cs | 217 |
| LoadCsi2CmiMacros | PkgGen.cs | 234 |
| PluginType 枚举 | PkgBldr.Common.cs | 3381 |
| PkgBldrLoader 类 | PkgBldr.Common.cs | 3658 |
| PkgBldrLoader.LoadPlugins | PkgBldr.Common.cs | 3714 |
| PkgBldrHelpers.CreateEmptyCsiXmlElement | PkgBldr.Common.cs | 3633 |
| PkgBldrHelpers.CreateEmptyWmXmlElement | PkgBldr.Common.cs | 3645 |
| PkgBldrHelpers.XDocumentLoadFromLongPath | PkgBldr.Common.cs | 3599 |
| IPkgPlugin 接口 | PkgBldr.Common.cs | ~55 |
| PkgPlugin 抽象类 | pkggencommon.cs | 3524 |
| SatelliteId 类 | pkggencommon.cs | 3688 |
| PkgObject 抽象类 | pkggencommon.cs | 3585 |

---

*文档生成时间: 2026-08-27*
*研究工具: ILSpy 反编译 .NET 程序集 + VS Code 源码分析*
*ADK 版本: 10.0.17704.1000 (Windows 10 RS5)*
