# ADK 10.0.17704.1000 — UpdateDLL.dll 两阶段更新机制深度研究

> **研究对象**: `updatedll.dll` (1.5MB, 32位, 116个导出函数)
> **源码路径**: `E:\WSK_Tools\ADK_Research\outputsrc\native\updatedll.c` (6.5MB)
> **托管封装**: `E:\WSK_Tools\ADK_Research\outputsrc\dotnet\imaging.cs` (UpdateMain 类, 行3305-3541)
> **ADK 版本**: Windows 10 RS5 (17704.1000)
> **源码路径标识**: `onecore\base\cbs\mobile\updatedll\lib\`

---

## 1. 概述

UpdateDLL.dll 是 ADK 镜像构建流水线的**核心执行引擎**，负责将 CBS 包（.cab/.msu）实际应用到离线镜像（VHD/物理磁盘/存储空间）。它实现了经典的**两阶段更新模型（Two-Phase Update）**：

| 阶段 | 函数 | 作用 | 可中断性 |
|------|------|------|----------|
| **Phase 1: Staging（暂存）** | `PrepareUpdate` / `PrepareUpdateWithFlags` | 解压包、验证签名/hash、将文件暂存到目标分区的临时空间、估算空间需求 | 可安全中断，不影响原系统 |
| **Phase 2: Commit（提交）** | `ExecuteUpdate` | 将暂存文件原子性地替换到最终位置、更新注册表、写 BCD、交换 A/B 分区 | 到达"Point of No Return"后不可中断 |

这是 Windows Mobile/OneCore 时代的标准更新架构，与桌面 Windows 的 `TrustedInstaller` + `CSI` 在线更新不同——UpdateDLL 是**纯离线**的，直接操作文件系统和注册表 hive，不需要运行中的 Windows 服务。

---

## 2. 116 个导出函数完整分类

### 2.1 更新上下文管理（8个）

| 函数 | 说明 |
|------|------|
| `CreateUpdateContext` | 创建更新上下文对象（CUpdateContext），返回句柄 |
| `ReleaseUpdateContext` | 释放上下文 |
| `Initialize` | 初始化上下文，加载 wcp.dll、设置日志回调、分配内部结构 |
| `Deinitialize` | 反初始化，清理资源 |
| `SetPoolId` | 设置存储池 ID（用于存储空间模式） |
| `GetStageFilesTickCount` | 获取暂存阶段耗时（毫秒） |
| `GetExecuteUpdateOSTickCount` | 获取提交阶段 OS 操作耗时 |
| `IU_LogTo` | 设置日志输出回调 |

### 2.2 两阶段更新核心（4个）

| 函数 | 说明 |
|------|------|
| `PrepareUpdate` | 基础暂存（无 flags） |
| `PrepareUpdateWithFlags` | 带 flags 的暂存（支持 ImperfectUpdateAllowed / CreatingStagingVHD） |
| `ExecuteUpdate` | 执行提交 |
| `SetIsolationIMalloc` | 设置隔离内存分配器（与 wcp.dll 协同） |

### 2.3 Device-Side Manifest 管理（12个）

`DeviceSideManifest_*` 系列函数操作设备端清单（DSM），这是描述目标设备当前状态的 XML 清单：

- `DeviceSideManifest_Create` / `DeviceSideManifest_Destroy`
- `DeviceSideManifest_Load_CBS` / `DeviceSideManifest_Load_DSM`
- `DeviceSideManifest_Save`
- `DeviceSideManifest_GetPackageCount` / `DeviceSideManifest_GetPackage`
- `DeviceSideManifest_AddPackage` / `DeviceSideManifest_RemovePackage`
- `DeviceSideManifest_GetFileCount` / `DeviceSideManifest_GetFile`

### 2.4 Diff Manifest 管理（18个）

`DiffManifest_*` 系列操作差异清单，描述源版本→目标版本的增量变更：

- `DiffManifest_Create` / `DiffManifest_Destroy`
- `DiffManifest_Load` / `DiffManifest_Save`
- `DiffManifest_GetAddedPackages` / `DiffManifest_GetRemovedPackages`
- `DiffManifest_GetUpdatedPackages`
- `DiffManifest_GetAddedFiles` / `DiffManifest_GetRemovedFiles` / `DiffManifest_GetUpdatedFiles`
- `DiffManifest_GetRegistryDeltas`
- ... 等

### 2.5 文件条目访问（16个）

`DSMFileEntry_Get_*` 和 `FileEntryBase_Get_*` 系列：

- `FileEntryBase_Get_SourcePath` / `FileEntryBase_Get_DestinationPath`
- `FileEntryBase_Get_FileHash` / `FileEntryBase_Get_HashAlgorithm`
- `FileEntryBase_Get_FileSize` / `FileEntryBase_Get_Attributes`
- `FileEntryBase_Get_SignInfoRequired`
- `DSMFileEntry_Get_PartitionId` / `DSMFileEntry_Get_StagingPath`
- ... 等

### 2.6 包描述符（40个）

`PackageDescriptor_Get_*` / `Set_*` 系列是最大的一组，操作 CBS 包的元数据：

- 身份: `Get_Name` / `Get_Version` / `Get_Architecture` / `Get_PublicKeyToken` / `Get_Culture`
- 类型: `Get_PackageType` (Feature Pack / Driver / Language Pack / ...)
- 依赖: `Get_DependencyCount` / `Get_Dependency`
- 文件: `Get_FileCount` / `Get_File`
- 注册表: `Get_RegistryKeyCount` / `Get_RegistryKey`
- 证书: `Get_CatalogFile` / `Get_Signature`
- 状态: `Get_InstallState` / `Set_InstallState`
- ... 等

### 2.7 工具函数（20个）

`IU_*` 前缀的工具函数：

| 函数 | 说明 |
|------|------|
| `IU_GetDirectorySize` | 递归计算目录大小（用于空间估算） |
| `IU_GetClusterSize` | 获取卷簇大小 |
| `IU_MountWim` / `IU_DismountWim` | 挂载/卸载 WIM（WinPE 恢复分区用） |
| `IU_CopyAllFiles` | 批量复制文件 |
| `IU_InitializeDefaultProgressReporting` | 初始化默认进度报告 |
| `IU_GetCompressedFilesize` / `IU_GetUncompressedFilesize` | 获取压缩/未压缩文件大小 |
| `IU_CaptureUVMState` | 捕获 UVM（Utility VM）状态（用于基于虚拟化的更新） |
| ... 等 |

### 2.8 注册表操作（4个）

| 函数 | 说明 |
|------|------|
| `RegHiveManipulator_BuildHives` | 从包的注册表 delta 构建离线注册表 hive |
| `RegHiveManipulator_ValidateRegistryHive` | 验证 hive 完整性 |
| `RegHiveManipulator_ExportRegistryHiveDeltas` | 导出 hive 增量 |
| `RegHiveManipulator_MergeHives` | 合并 hive |

### 2.9 WDS / 其他（4个）

- `SetupWdscore` — 配置 Windows Deployment Services 核心
- `CreateNewWindows` — 创建新的 Windows 离线存储（wcp.dll 转发）
- `WcpInitialize` / `WcpShutdown` — wcp.dll 初始化/关闭

---

## 3. 核心数据结构

### 3.1 CUpdateContext（更新上下文）

从反编译代码中的偏移量推断，上下文对象大小约 **0x280+ 字节**，关键字段布局：

| 偏移 | 字段 | 类型 | 说明 |
|------|------|------|------|
| +0x00 | vtable | ptr | 虚函数表 |
| +0x04 | state | uint32 | 上下文状态（0=未初始化, 1=已初始化, 2=暂存中, 3=暂存完成, 4=提交中） |
| +0x05 | isStaging | byte | 是否在暂存阶段 |
| +0x0C | imagePath | wstring | 镜像路径（VHD文件或物理磁盘路径） |
| +0x20 | imagePath_len | uint32 | 字符串长度（>7 时为堆分配指针） |
| +0x46 | stagingPath | wstring | 暂存根路径 |
| +0x4C | dataRootPath | wstring | 数据根路径（DataRootPath） |
| +0x60 | deviceSideManifest | ptr | 设备端清单对象 |
| +0x6B | workingDirectory | wstring | 工作目录 |
| +0x89 | logDirectory | wstring | 日志目录 |
| +0x94 | poolId | ptr | 存储池 ID（存储空间模式） |
| +0x9A | useSpaces | byte | 是否使用存储空间模式 |
| +0x118 | partitionList | ptr | 分区列表（std::vector） |
| +0x124 | useABSwap | uint32 | 是否使用 A/B 分区交换 |
| +0x128 | partitionIter | ptr | 分区迭代器 |
| +0x130 | mountPath | wstring | 挂载路径 |
| +0x144 | mountPath_len | uint32 | 挂载路径长度 |
| +0x160 | badDiffsPath | wstring | bad diffs 路径（Point of No Return 标记） |
| +0x178 | partitionTable | uint32[5] | 分区表备份（用于 A/B 交换） |
| +0x1AC | dataRootPath2 | wstring | 数据根路径（备用） |
| +0x1C0 | dataRootPath2_len | uint32 | 长度 |
| +0x224 | lastFailureLogPath | wstring | 上次失败日志路径 |
| +0x238 | lastFailureLogPath_len | uint32 | 长度 |
| +0x242 | imperfectAllowed | byte | 是否允许不完美更新（ImperfectUpdateAllowed） |
| +0x260 | stageStartTick | uint32 | 暂存开始时间（GetTickCount） |
| +0x264 | stageElapsed | uint32 | 暂存耗时 |
| +0x268 | noCleanup | byte | 是否跳过清理（NO_CLEANUP 环境变量） |
| +0x269 | isStagingFlag | byte | 暂存标志 |
| +0x26C | reserveSpace[4] | uint32[4] | 预留空间（4个分区的固定预留） |
| +0x270~0x27C | reserveParams | uint32[4] | 预留空间参数 |

> **注**: 字符串字段使用 `std::wstring` 的 SSO（Small String Optimization）布局——长度 ≤7 时内联存储，>7 时第一个字段变为堆指针。代码中 `if (7 < *(uint*)(base+offset))` 即判断是否使用堆指针。

### 3.2 PARTITION_INFO（分区信息）

每个分区的描述结构，从 `FUN_100b418e`（获取分区信息）和相关函数推断：

| 偏移 | 字段 | 说明 |
|------|------|------|
| +0x00 | vtable | 虚表 |
| +0x34 | stagingRoot | 暂存根路径 |
| +0x48 | stagingRoot_len | 长度 |
| +0x64 | mountPath | 挂载路径 |
| +0x78 | mountPath_len | 长度 |
| +0x100 | partitionId | 分区 ID（GUID 或序号） |
| +0x118 | fileList | 文件列表 |
| +0x124 | isMainOS | 是否 MainOS 分区 |
| +0x1AC | dataRootPath | 数据根路径 |

### 3.3 OFFLINE_STORE_CREATION_PARAMETERS

从 imaging.cs 的 P/Invoke 声明（行3485附近）：

```csharp
[StructLayout(LayoutKind.Sequential, CharSet = CharSet.Unicode)]
public struct OFFLINE_STORE_CREATION_PARAMETERS
{
    public uint Size;                    // 结构大小
    public uint Flags;                   // 创建标志
    [MarshalAs(UnmanagedType.LPWStr)] public string ImagePath;
    [MarshalAs(UnmanagedType.LPWStr)] public string WindowsDirectory;
    [MarshalAs(UnmanagedType.LPWStr)] public string SystemRoot;
    // ... 更多字段
}
```

---

## 4. PrepareUpdate（暂存阶段）完整流程

### 4.1 函数签名

```c
HRESULT __fastcall PrepareUpdateWithFlags(
    CUpdateContext* this,   // ECX: 上下文指针
    DWORD flags,             // EDX: PrepareUpdateFlags
    ULARGE_INTEGER* outRequiredSpace  // 输出: 所需空间
);
```

**PrepareUpdateFlags**:
- `None = 0`
- `ImperfectUpdateAllowed = 1` — 允许不完美更新（缺少可选依赖时继续）
- `CreatingStagingVHD = 2` — 正在创建暂存 VHD（特殊模式）

### 4.2 执行步骤（按源码行号）

**入口检查** (行91407-91423):
1. 记录 `GetTickCount()` 到 `stageStartTick`
2. 设置 `isStagingFlag = flags & 1`（ImperfectUpdateAllowed）
3. 设置 `imperfectAllowed = flags & 1`
4. 检查 `state` 字段（偏移+5），如果已在暂存中则报错返回
5. 检查是否已调用 `Initialize`，未初始化则返回 `0x80070016`（ERROR_NOT_READY）

**日志保存** (行91425-91457):
6. 拼接 `lastFailureLogPath + mountPath`，保存上次失败日志
7. 失败时仅警告（`Warning`），不中断

**清理旧暂存空间** (行91458-91468):
8. 调用 `FUN_100b93df(partitionList)` 删除旧的暂存空间
9. 失败时警告并返回

**检测旧存储空间** (行91469-91529):
10. 获取分区暂存路径 `FUN_100b41fe(partitionList)`
11. 调用 `FUN_100c4c1b(path)` 检测是否存在旧的暂存空间
12. 如果存在旧空间:
    - 调用 `FUN_100d6ae2()` 执行清理
    - 如果需要估算空间（`local_31 != 0`）:
      - `FUN_100b62d0(partitionList, &estimated)` 估算暂存空间
      - `GetDiskFreeSpaceExW(dataRootPath, &free, ...)` 获取可用空间
      - 比较 `free < estimated`，输出 "Staging is NOT expected to fit" 或 "Staging is expected to fit"
      - **注意**: 日志明确说 "The following message is ***NOT an error***, simply an estimate"
13. 如果旧空间清理失败，警告返回

**空间估算输出** (行91516-91522):
14. `*outRequiredSpace = estimated`（64位）
15. 日志: "UpdateDLL: Staging completed successfully"
16. 调用 `FUN_100d6482()` 进入暂存文件子流程

### 4.3 暂存文件子流程（FUN_100d6482 → FUN_100d64cf）

**包验证** (行91871-91880):
1. `FUN_100d6f25(context)` — **包验证**：遍历所有包，验证签名和 catalog hash
   - 失败返回 "Package verification failed, error is 0x%08X!"
   - 这是 hash/证书验证的关键调用点（内部调用 wcp.dll!CCSDirectTransaction::VerifyFileHashes）

**空间预留** (行91881-91884):
2. 设置 `useSpaces = 1`（标记使用了空间估算）
3. `FUN_100b9832(&reservedSpace)` — 预留空间
4. `FUN_100b82ac(partitionList+0x46)` — 初始化分区暂存目录

**逐分区文件准备** (行91885-91893, 调用 FUN_100d6608):
5. 遍历分区列表（`partitionList` at +0x118）:
   - 对每个分区:
     a. `FUN_100c2a65(partition+0x28)` — **准备文件**（解压、hash验证、复制到暂存位置）
        - 失败: "Failed to prepare files for partition: %1, error is 0x%08X!"
     b. `FUN_100c3a4e(partition+0x28)` — **预预留暂存空间**
        - 失败: "Failed to pre-reserve staging space for partition: %1, error is 0x%08X!"

**提交到已知 Store ID 列表** (行91895-91913):
6. 如果 `isStagingFlag == 0`（非 Imperfect 模式）:
   - `FUN_100d59ab(context)` — 额外验证
7. 如果 `poolId != NULL`:
   - 设置 `context+0x48 = 1`
   - `FUN_100d22c1(context)` — **提交更新到已知 Store ID 列表**
     - 失败: "Failed to commit update to list of known store IDs, error is 0x%08X!"

**暂存文件大小统计** (行91924-91940):
8. 如果无错误（`iVar3 == 0`）:
   - `FUN_100f6ddc(mountPath, 1, NULL, &totalStagedSize)` — 统计暂存文件总大小
   - 日志: "Total size of staged files = %I64u! bytes"

**写历史文件** (行92684-92698):
9. `FUN_100fa01f()` — 检查是否需要写历史
10. 如果需要:
    - `FUN_100b418e(partitionList)` — 获取分区信息
    - `FUN_100d52a1(context, context+0x60)` — 写更新历史文件
    - 失败仅警告

**完成** (行92700-92713):
11. `FUN_100b2cbe(context+400, mountPath)` — 检查是否需要清理
12. `FUN_100fcac9(&DAT_1016fe70, result, needCleanup)` — 记录结果
13. 计算耗时: `elapsed = GetTickCount() - stageStartTick`
14. 日志: "Staging files completed in %u! ms"

---

## 5. ExecuteUpdate（提交阶段）完整流程

### 5.1 函数签名

```c
HRESULT __fastcall ExecuteUpdate(
    CUpdateContext* this,   // ECX
    DWORD flags              // EDX: ExecuteUpdateFlags
);
```

**ExecuteUpdateFlags**:
- `None = 0`
- `UsingStagingVHD = 1` — 使用暂存 VHD 模式
- `ResetCommit = 2` — 重置提交状态（用于重试）

### 5.2 执行步骤

**前置检查** (行88529-88538):
1. 日志: "UpdateDLL: Committing update."
2. 检查是否已初始化，未初始化返回 `0x80070016`，错误: "ExecuteUpdate was called before initialize"

**加载 UpdateOutput.xml** (行88641-88703):
3. 构建路径: `SharedData\DuShared\UpdateOutput.xml`
4. `FUN_100f5405(path)` — 检查文件是否存在
5. 如果不存在，尝试从 `mountPath + \SharedData\DuShared\UpdateOutput.xml` 加载
6. 如果仍不存在，返回错误: "ExecuteUpdate was called before completion of PrepareUpdate"
7. `FUN_100bd4b7(deviceSideManifest, path)` — 加载更新输出文件（包含暂存结果）

**快速路径: state==4** (行88690-88728):
8. 如果 `state == 4`（已暂存完成，直接提交）:
   - 调用 `FUN_100bdc88(0, 4, deviceSideManifest, partitionList, mountPath)` — 快速提交
   - 检查 `NO_CLEANUP` 环境变量，未设置则清理
   - 日志: "UpdateDLL: Update completed successfully"
   - 返回

**验证暂存结果** (行88729-88737):
9. `FUN_100bdb83(deviceSideManifest)` — 验证暂存是否成功
   - 失败: "Staging didn't previously succeed, error is 0x%08X!"

**Pagefile 设置** (行88738-88751):
10. 如果非 StagingVHD 模式:
    - `FUN_100d0ffb(dataRootPath)` — 设置 pagefile
      - 失败仅警告: "Pagefile setup failed with DataRootPath %1, result was 0x%08X!"

**读取更新输入 + 设置压缩状态** (行88752-88787):
11. `FUN_100d6d72(context)` — 读取更新输入文件
    - 失败: "Failed to read update input file, error is 0x%08X!"
12. `FUN_100b6f54(partitionList)` — 设置压缩状态
    - 失败: "Failed to set the compression state, error is 0x%08X!"

**获取分区参数** (行88788-88836):
13. `FUN_100c7371(0x1016e19c)` — 获取分区参数数量
14. `FUN_100b7bd8(...)` — 获取6个分区参数（stagingRoot, mountPath, 等）
15. `FUN_100c7397(&DAT_1016e19c, ...)` — 设置分区参数到内部状态

**SBCP 检查** (行88837-88864):
16. 如果非 StagingVHD 模式:
    - `FUN_100f52d6()` — 检查是否需要 SBCP（Single Binary Compatibility Platform）验证
    - 如果需要:
      - 获取分区 `stagingRoot` 和 `mountPath`
      - `FUN_100d1637(mountPath, stagingRoot)` — **SBCP 检查**（验证暂存文件与目标系统兼容性）
        - 失败: "Failed SBCP check with staging root %1 and mount path %2, error was 0x%08X!"

**核心提交** (行88867-88875):
17. `FUN_100d22c1(context)` — **核心提交**（将暂存文件替换到最终位置）
    - 失败: "Commit failed, error is 0x%08x!"

> **存储空间模式**: 如果使用存储空间（`useSpaces`），则调用 `FUN_100d3376(context)` 替代:
> - 失败: "Failed to commit spaces-based image, error is 0x%08X!"

**复制 BCD 到物理 ESP** (行88879-88887):
18. `FUN_100d3688(partitionList)` — 复制 BCD（Boot Configuration Data）到物理 ESP 分区
    - 失败: "Failed to copy BCD to physical ESP, error is 0x%08X!"

**A/B 分区交换** (行88889-88896):
19. 如果 `useABSwap != 0`:
    - 日志: "Swapping backup and primary partitions in GPT"
    - `FUN_100d4bc5(...)` — 执行 GPT 分区表交换
      - `FUN_101433ca(0, backupTable, partitionTable)` — `RepairPartitionTableWithId`（用备份表修复）
      - `FUN_101434c1(1, highPart, partitionTable)` — 复制备份表到主表
      - 失败: "RepairPartitionTableWithId failed" / "Failed to copy backup table to the primary table"
    - 返回

**获取 CPU 架构 + 创建固定预留空间** (行88898-88936):
20. `FUN_100b41fe(partitionList)` — 获取分区信息
21. `FUN_100bfdc0(partitionInfo, &cpuArch)` — 获取 CPU 架构
    - 失败: "Failed to get cpu arch, error is 0x%08x!"
22. `FUN_100fa01f()` — 检查是否需要创建预留空间
23. 如果需要，遍历4个预留空间条目（`reserveSpace[4]` at +0x26C）:
    - 如果与当前值不同且 `cpuArch != 2`:
      - `FUN_101413e1(poolId, offset, size, type, flags)` — 创建固定预留空间
        - 失败: "Failed to create fixed reserve space, error is 0x%08x!"

**Point of No Return** (行88937-88971):
24. `FUN_100b2e1c(context+400, mountPath)` — 删除 Point of No Return 标记文件
    - 失败仅警告: "Failed to delete point of no return marker, error is 0x%08x!"
25. 获取 `badDiffsPath`（at +0x160）
26. `FUN_100f749b(badDiffsPath)` — 删除 bad diffs 目录
    - 如果删除失败且存在错误回调:
      - 记录错误: "Failed to delete baddiffs from path {0}"
      - `FUN_100b3650(errorCallback, ...)` — 通知错误
      - 返回
27. 日志: "Reached point of no return end"
28. `FUN_100d4e8e()` — 完成提交

### 5.3 完成后处理（FUN_100d4e8e → FUN_100d5029）

**写输出文件** (行90331-90339):
1. 检查 `result < 0`，如果写输出文件失败:
   - 警告: "UpdateDLL: Failed to write output file, hr = 0x%08x!, retrying..."
   - `FUN_100d50b7()` — 重试写输出文件
     - `FUN_100bdc88(result, state, deviceSideManifest, partitionList, mountPath)` — 重试
     - 再次失败: "UpdateDLL: Failed to write output file, hr = 0x%08x!"

**写更新历史** (行90340-90345):
2. 如果非 StagingVHD 模式:
   - 日志: "Writing update history."
   - `FUN_100d5165()` — 写历史
     - `FUN_100d52a1(context, deviceSideManifest)` — 实际写历史文件
     - 失败仅警告

**报告最终进度** (行90346-90349):
3. 日志: "Reporting final progress."
4. `FUN_100d51d8()` — 报告最终进度

---

## 6. Hash 与证书验证在 UpdateDLL 中的调用链

UpdateDLL 本身**不直接实现** hash 验证，而是通过 wcp.dll 的 CBS 引擎完成。调用链如下：

```
PrepareUpdateWithFlags
  └─ FUN_100d6f25(context)          [包验证入口]
       └─ 遍历 PackageDescriptor 列表
            └─ wcp.dll!CCSDirectTransaction::StagePackage
                 ├─ 验证 .cat 证书签名（WinVerifyTrust）
                 ├─ 解析 CBS 清单（manifest.xml）
                 └─ CCSDirectTransaction::VerifyFileHashes
                      ├─ 对每个 FileEntry:
                      │   ├─ 读取清单中的 FileHash（SHA256, 32字节）
                      │   ├─ 计算实际文件的 SHA256
                      │   └─ 比较 → 不匹配返回 0xc015001b (CBS_E_CORRUPT_FILE)
                      └─ 验证通过后将文件复制到暂存位置
```

### 6.1 关键验证点

| 验证类型 | 发生时机 | 失败行为 |
|----------|----------|----------|
| **CAB 签名验证** | PrepareUpdate 开始时 | 硬失败，返回错误 |
| **Catalog (.cat) 证书链** | 加载包时 | 硬失败（除非 ImperfectUpdateAllowed） |
| **文件 hash 验证** | 暂存每个文件时 | 硬失败，`CBS_E_CORRUPT_FILE` |
| **SBCP 兼容性检查** | ExecuteUpdate 提交前 | 硬失败（仅非 StagingVHD 模式） |
| **UpdateOutput.xml 存在性** | ExecuteUpdate 开始时 | 硬失败（"called before completion of PrepareUpdate"） |

### 6.2 ImperfectUpdateAllowed 的影响

当 `PrepareUpdateFlags.ImperfectUpdateAllowed = 1` 时:
- `context+0x242 = 1`（标记允许不完美更新）
- `context+0x269 = 1`（暂存标志）
- 跳过 `FUN_100d59ab(context)` 的额外验证
- 可选依赖缺失时继续执行（而非硬失败）
- **但文件 hash 验证仍然强制执行**——Imperfect 只放宽依赖检查，不放宽完整性检查

---

## 7. 存储空间（Storage Spaces）模式

当使用存储空间模式时（`SetPoolId` 设置了 poolId），UpdateDLL 的行为发生变化：

### 7.1 PrepareUpdate 中的差异

- `useSpaces = 1`（行91882）
- 调用 `FUN_100b82ac(partitionList+0x46)` 初始化存储空间分区
- 空间估算使用 `FUN_100b62d0(partitionList, &estimated)` 而非简单的目录大小
- 提交时调用 `FUN_100d3376(context)` 而非 `FUN_100d22c1(context)`

### 7.2 关键 IOCTL

存储空间操作通过 `DeviceIoControl` 完成，关键控制码：
- `0x2d1080` — 创建存储池（IOCTL_STORAGE_POOL_CREATE）
- `0x2d5928` — 获取 VHD 文件名（IOCTL_VHD_GET_PHYSICAL_PATH）

---

## 8. A/B 分区交换机制

### 8.1 触发条件

`context+0x124 (useABSwap) != 0` 时启用。这是双分区系统（如 Windows Mobile / HoloLens）的标准更新方式：

- **Primary 分区**: 当前运行的系统
- **Backup 分区**: 更新写入的目标
- 更新完成后交换 GPT 分区表中的 Type GUID，使 Backup 变为新的 Primary

### 8.2 执行流程

```
FUN_100d4bc5 (A/B 交换)
  ├─ 读取 partitionTable[5] (context+0x178)
  ├─ FUN_101433ca(0, highPart, table)  → RepairPartitionTableWithId
  │    └─ 用备份分区表修复主分区表（设置 id=0）
  └─ FUN_101434c1(1, highPart, table)  → CopyBackupToPrimary
       └─ 将备份表复制到主表（id=1）
```

### 8.3 分区表结构

`partitionTable` 是 5 个 `uint32` 的数组（context+0x178 ~ +0x18C），每个条目对应一个分区的 Type GUID 低32位或分区属性。

---

## 9. Point of No Return（不可返回点）

### 9.1 机制

UpdateDLL 使用一个标记文件来跟踪"不可返回点"：

- **标记路径**: `context+0x160 (badDiffsPath)` — 一个目录路径
- **设置时机**: 暂存阶段开始时创建此目录（包含 bad diffs 信息）
- **清除时机**: ExecuteUpdate 成功完成核心提交后删除此目录
- **意义**: 如果系统在删除标记之前断电/崩溃，下次启动时检测到标记 → 知道更新未完成 → 触发回滚或修复

### 9.2 代码中的处理

```c
// ExecuteUpdate 行88937-88971
FUN_100b2e1c(context+400, mountPath);  // 删除 PNR 标记文件
badDiffsPath = context+0x160;
if (RemoveDirectoryW(badDiffsPath) == 0) {
    // 删除失败 → 通知错误回调
    Log("Failed to delete baddiffs from path {0}");
    errorCallback(error);
    return;
}
Log("Reached point of no return end");
```

---

## 10. 日志与错误报告

### 10.1 日志函数

UpdateDLL 使用统一的日志函数族（`FUN_100fc3xx`）：

| 函数 | 级别 | 格式 |
|------|------|------|
| `FUN_100fc3b0` | Info | `L"UpdateDLL: ..."` |
| `FUN_100fc370` | Error | 包含源码路径+行号+错误描述 |
| `FUN_100fc390` | Warning | 包含源码路径+行号+警告描述 |
| `FUN_100fcac9` | Result | 记录最终结果（成功/失败+是否需清理） |

### 10.2 错误日志格式

所有错误日志都包含完整的源码溯源信息：

```
onecore\base\cbs\mobile\updatedll\lib\prepareupdate.cpp,
UpdateMain::PrepareUpdate, line 125, Error,
Package verification failed, error is 0x%08X!
```

这使得从日志直接定位到源码行号成为可能。

### 10.3 进度报告

- `IU_InitializeDefaultProgressReporting` — 初始化默认进度报告
- 暂存完成: "Staging files completed in %u ms"
- 提交完成: "UpdateDLL: Update completed successfully"
- 空间估算: "Staging is%1 expected to fit: estimated to take %2 bytes, %3 are free"

---

## 11. 托管封装层（imaging.cs UpdateMain 类）

### 11.1 P/Invoke 声明

imaging.cs 中的 `UpdateMain` 类（行3305-3541）封装了 UpdateDLL.dll 的核心函数：

```csharp
internal static class UpdateMain
{
    // === 更新上下文管理 ===
    [DllImport("updatedll.dll", CallingConvention = CallingConvention.Cdecl)]
    internal static extern int CreateUpdateContext(out IntPtr context);

    [DllImport("updatedll.dll", CallingConvention = CallingConvention.Cdecl)]
    internal static extern int ReleaseUpdateContext(IntPtr context);

    [DllImport("updatedll.dll", CallingConvention = CallingConvention.Cdecl)]
    internal static extern int Initialize(IntPtr context,
        [MarshalAs(UnmanagedType.LPWStr)] string imagePath,
        [MarshalAs(UnmanagedType.LPWStr)] string windowsDir,
        IntPtr parameters);

    [DllImport("updatedll.dll", CallingConvention = CallingConvention.Cdecl)]
    internal static extern int Deinitialize(IntPtr context);

    // === 两阶段更新 ===
    [DllImport("updatedll.dll", CallingConvention = CallingConvention.Cdecl)]
    internal static extern int PrepareUpdate(IntPtr context, out ulong requiredSpace);

    [DllImport("updatedll.dll", CallingConvention = CallingConvention.Cdecl)]
    internal static extern int PrepareUpdateWithFlags(IntPtr context,
        uint flags, out ulong requiredSpace);

    [DllImport("updatedll.dll", CallingConvention = CallingConvention.Cdecl)]
    internal static extern int ExecuteUpdate(IntPtr context, uint flags);

    [DllImport("updatedll.dll", CallingConvention = CallingConvention.Cdecl)]
    internal static extern int SetPoolId(IntPtr context,
        [MarshalAs(UnmanagedType.LPWStr)] string poolId);

    // === 性能统计 ===
    [DllImport("updatedll.dll", CallingConvention = CallingConvention.Cdecl)]
    internal static extern uint GetStageFilesTickCount(IntPtr context);

    [DllImport("updatedll.dll", CallingConvention = CallingConvention.Cdecl)]
    internal static extern uint GetExecuteUpdateOSTickCount(IntPtr context);

    [DllImport("updatedll.dll", CallingConvention = CallingConvention.Cdecl)]
    internal static extern ulong GetCompressedFilesize(IntPtr context);

    [DllImport("updatedll.dll", CallingConvention = CallingConvention.Cdecl)]
    internal static extern ulong GetUncompressedFilesize(IntPtr context);

    // === 工具函数 ===
    [DllImport("updatedll.dll", CallingConvention = CallingConvention.Cdecl)]
    internal static extern int IU_GetDirectorySize(
        [MarshalAs(UnmanagedType.LPWStr)] string path, out ulong size);

    [DllImport("updatedll.dll", CallingConvention = CallingConvention.Cdecl)]
    internal static extern int IU_GetClusterSize(
        [MarshalAs(UnmanagedType.LPWStr)] string path, out uint clusterSize);

    [DllImport("updatedll.dll", CallingConvention = CallingConvention.Cdecl)]
    internal static extern int CopyAllFiles(
        [MarshalAs(UnmanagedType.LPWStr)] string src,
        [MarshalAs(UnmanagedType.LPWStr)] string dst);

    [DllImport("updatedll.dll", CallingConvention = CallingConvention.Cdecl)]
    internal static extern int IU_MountWim(
        [MarshalAs(UnmanagedType.LPWStr)] string wimPath,
        [MarshalAs(UnmanagedType.LPWStr)] string mountPath);

    [DllImport("updatedll.dll", CallingConvention = CallingConvention.Cdecl)]
    internal static extern int IU_DismountWim(
        [MarshalAs(UnmanagedType.LPWStr)] string mountPath);

    [DllImport("updatedll.dll", CallingConvention = CallingConvention.Cdecl)]
    internal static extern void IU_LogTo(IntPtr logCallback);

    [DllImport("updatedll.dll", CallingConvention = CallingConvention.Cdecl)]
    internal static extern void IU_InitializeDefaultProgressReporting();

    // === wcp.dll 转发 ===
    [DllImport("wcp.dll", CallingConvention = CallingConvention.StdCall)]
    internal static extern int WcpInitialize();

    [DllImport("wcp.dll", CallingConvention = CallingConvention.StdCall)]
    internal static extern int WcpShutdown();

    [DllImport("wcp.dll", CallingConvention = CallingConvention.StdCall)]
    internal static extern int SetIsolationIMalloc(IntPtr malloc);

    [DllImport("wcp.dll", CallingConvention = CallingConvention.StdCall)]
    internal static extern int CreateNewWindows(IntPtr parameters);

    // === Ole32 ===
    [DllImport("Ole32.dll")]
    internal static extern int CoGetMalloc(uint context, out IntPtr malloc);
}
```

### 11.2 调用约定

- **UpdateDLL.dll 函数**: `CallingConvention.Cdecl`（C 调用约定，调用者清理栈）
- **wcp.dll 函数**: `CallingConvention.StdCall`（标准调用，被调用者清理栈）
- 这种差异源于两个 DLL 的编译选项不同

---

## 12. 重建要点

### 12.1 必须保留的机制

1. **两阶段分离**: PrepareUpdate 和 ExecuteUpdate 必须严格分离，中间通过 `UpdateOutput.xml` 传递状态
2. **Point of No Return**: 必须实现标记文件机制，支持崩溃恢复
3. **Hash 验证**: 文件 hash 验证不可跳过，即使 ImperfectUpdateAllowed 模式
4. **空间估算**: PrepareUpdate 必须返回准确的空间需求，使用 `IU_GetDirectorySize` + 簇大小对齐
5. **A/B 交换**: 如果目标设备支持双分区，必须实现 GPT 分区表交换
6. **存储空间模式**: 必须同时支持普通分区和 Storage Spaces 两种模式

### 12.2 可简化的部分

1. **UVM 支持**: `IU_CaptureUVMState` 是基于虚拟化的更新，普通镜像构建不需要
2. **WDS 配置**: `SetupWdscore` 仅用于网络部署场景
3. **Diff Manifest**: 增量更新相关的 18 个函数，全新镜像构建不需要
4. **进度报告回调**: 可以简化为简单的控制台输出

### 12.3 关键依赖

| 依赖 | 用途 | 可替代方案 |
|------|------|-----------|
| wcp.dll | CBS 包解析、hash 验证、注册表处理 | 自己实现 CBS 清单解析 + SHA256 + offreg.dll |
| imagestorageservice.dll | VHD/存储空间操作 | 自己实现 virtio API + VDS COM |
| offreg.dll | 离线注册表 hive 操作 | 直接使用（微软官方 redistributable） |
| appxpackaging.dll | AppX 包处理 | 自己实现或使用 MakeAppx |
| signinfohelper.dll | SignInfoRequired 判断 | 仅一个导出函数，可直接硬编码逻辑 |

---

## 13. 关键常量与错误码

### 13.1 错误码

| 错误码 | 值 | 含义 |
|--------|-----|------|
| `ERROR_NOT_READY` | 0x80070016 | 未调用 Initialize / PrepareUpdate 未完成 |
| `CBS_E_CORRUPT_FILE` | 0xc015001b | 文件 hash 不匹配 |
| `E_FAIL` | 0x8000ffff | 通用失败（提交后非 S_OK） |

### 13.2 环境变量

| 变量 | 作用 |
|------|------|
| `NO_CLEANUP` | 设置后跳过暂存文件清理（用于调试） |
| `FORCE_CLEANUP` | 强制清理旧暂存空间 |

### 13.3 状态值

| 值 | 状态 |
|----|------|
| 0 | 未初始化 |
| 1 | 已初始化 |
| 2 | 暂存中 |
| 3 | 暂存完成 |
| 4 | 已暂存（快速提交路径） |

---

## 14. 源码行号索引

| 函数/功能 | updatedll.c 行号 |
|-----------|-------------------|
| PrepareUpdateWithFlags 入口 | 91400 |
| PrepareUpdate 包验证 (FUN_100d6f25) | 91871 |
| PrepareUpdate 逐分区文件准备 (FUN_100d6608) | 92788 |
| PrepareUpdate 完成 (FUN_100d64cf) | 92654 |
| ExecuteUpdate 入口 | 88520 |
| ExecuteUpdate 加载 UpdateOutput.xml | 88641 |
| ExecuteUpdate SBCP 检查 | 88837 |
| ExecuteUpdate 核心提交 | 88867 |
| ExecuteUpdate A/B 交换 (FUN_100d4bc5) | 89878 |
| ExecuteUpdate Point of No Return | 88937 |
| ExecuteUpdate 完成后处理 (FUN_100d4e8e) | 90172 |
| DeviceSideManifest_Load_CBS | ~3317 |
| CreateUpdateContext / Initialize | ~7365-7377 |

---

*文档生成时间: 2026-08-27*
*研究工具: IDA Pro 反编译 + VS Code 源码分析 + PowerShell PE 导出表解析*
*ADK 版本: 10.0.17704.1000 (Windows 10 RS5)*
