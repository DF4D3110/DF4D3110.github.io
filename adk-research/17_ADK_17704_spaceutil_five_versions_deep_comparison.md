# ADK 10.0.17704.1000 — spaceutil.exe 五版本深度对比分析

> 文档版本: 1.0 | 日期: 2026-08-27
> 范围: rs1/rs2/rs3/rs4/rs5 五个版本 spaceutil.exe 的完整逆向对比
> 反编译源: `outputsrc\native\rs{1-5}_spaceutil.c` (总计 ~1.96MB, 73354行)

---

## 目录

1. [概述](#1-概述)
2. [五版本基本信息对比](#2-五版本基本信息对比)
3. [命令体系完整还原](#3-命令体系完整还原)
4. [IOCTL 接口深度分析](#4-ioctl-接口深度分析)
5. [设备交互原理](#5-设备交互原理)
6. [注册表交互](#6-注册表交互)
7. [参数系统](#7-参数系统)
8. [版本演化分析](#8-版本演化分析)
9. [工作原理完整流程](#9-工作原理完整流程)
10. [重建项目建议](#10-重建项目建议)

---

## 1. 概述

### 1.1 spaceutil.exe 是什么

**spaceutil.exe = Storage Spaces Utility**，是 Microsoft Storage Spaces（存储空间）技术的命令行管理工具。它通过内核驱动 `spaceport.sys`（Storage Spaces 端口驱动）和 `volmgr.sys`（卷管理器）的 IOCTL 接口，实现对存储池（Pool）、虚拟磁盘（Space）、物理磁盘（Drive）、机箱（Enclosure）、存储层（Tier）、任务（Task）等对象的全生命周期管理。

### 1.2 在 ADK 中的角色

spaceutil.exe 在 Windows 10 ADK 中用于 **离线镜像构建阶段的存储配置**：
- 在 FFU 镜像中预配置 Storage Spaces 池和虚拟磁盘
- 设置存储池参数（列数、条带大小、冗余类型）
- 配置存储层（Tier）用于分层存储
- 初始化存储子系统

### 1.3 为什么有5个版本

ADK 同时打包了 rs1~rs5 五个版本的 spaceutil.exe，每个版本对应不同 Windows 10 版本的 `spaceport.sys` 驱动接口：

| 版本 | Windows 10 | 代号 | 发布年份 |
|------|-----------|------|---------|
| rs1 | 1507 | Threshold 1 | 2015 |
| rs2 | 1607 | Anniversary Update | 2016 |
| rs3 | 1703 | Creators Update | 2017 |
| rs4 | 1803 | April 2018 Update | 2018 |
| rs5 | 1809 | October 2018 Update | 2018 |

镜像构建时根据目标系统版本选择对应的 spaceutil.exe，确保 IOCTL 数据结构与目标驱动匹配。

---

## 2. 五版本基本信息对比

### 2.1 总体对比表

| 属性 | rs1 | rs2 | rs3 | rs4 | rs5 |
|------|-----|-----|-----|-----|-----|
| 原始大小 | 124,928B | 123,904B | 104,960B | 107,520B | 113,664B |
| 反编译大小 | 388KB | 395KB | 374KB | 396KB | 411KB |
| 反编译行数 | 13,576 | 13,823 | 14,456 | 15,462 | 16,037 |
| 函数数量 | 216 | 221 | 270 | 295 | 303 |
| IOCTL调用点 | 56 | 58 | 57 | 58 | 58 |
| 唯一IOCTL码 | 53 | 53 | 53 | 53 | 53 |
| 命令数 | 38 | 38 | 38 | 38 | 38 |
| 命令名可读 | ✅ 是 | ✅ 是 | ❌ 否 | ❌ 否 | ❌ 否 |
| 设备路径数 | 3 | 3 | 3 | 3 | 3 |

### 2.2 关键发现

1. **IOCTL 接口完全稳定**：53个唯一IOCTL码在所有5个版本中**100%相同**，没有任何增删。这意味着 spaceport.sys 驱动的 IOCTL 接口从 rs1 到 rs5 保持向后兼容。

2. **函数数量持续增长**：从 rs1 的 216 个函数增长到 rs5 的 303 个函数（+40%），但 IOCTL 码不变。新增函数主要是：
   - 内部数据结构处理函数
   - 参数解析和验证函数
   - 错误处理和日志函数
   - 新的对象属性处理

3. **命令名在 rs3 开始被混淆**：rs1/rs2 的命令名以明文宽字符串存储（`SuAddDrives`、`SuCreatePool` 等），rs3/rs4/rs5 的命令名不再以明文出现，只有基础字符串（`Auto`、`clusdisk`、`False`、`Number`、`spaceport`、`True`）。这可能是：
   - 命令名被加密/编码存储
   - 命令表移到了资源段或其他段
   - 使用了运行时解密

4. **原始大小先降后升**：rs3 最小（105KB），rs5 最大（114KB）。rs3 的精简可能与命令名混淆有关（减少了字符串存储），后续版本增加了新功能。

---

## 3. 命令体系完整还原

### 3.1 38个命令完整列表（基于 rs1/rs2 明文）

所有命令以 `Su`（Storage Utility）前缀开头，按功能分类：

#### 3.1.1 初始化与控制（2个）

| 命令 | 功能 | 对应IOCTL |
|------|------|----------|
| `SuInitialize` | 初始化存储子系统 | 0xE70004, 0xE70008 |
| `SuSetControl` | 设置全局控制参数 | 0xE70404, 0xE70408 |

#### 3.1.2 存储池管理（Pool）（7个）

| 命令 | 功能 | 对应IOCTL |
|------|------|----------|
| `SuCreatePool` | 创建存储池 | 0xE7C00C~0xE7C020 |
| `SuDeletePool` | 删除存储池 | 0xE7C010, 0xE7C018 |
| `SuSetPool` | 设置池属性 | 0xE7C014, 0xE7C01C |
| `SuSetPoolAttributes` | 设置池扩展属性 | 0xE7C020 |
| `SuGetPoolTemplate` | 获取池模板 | 0xE7C00C, 0xE7C010 |
| `SuRefreshPool` | 刷新池状态 | 0xE7C018 |
| `SuUpgradePool` | 升级池版本 | 0xE7C01C, 0xE7C020 |

#### 3.1.3 虚拟磁盘管理（Space）（10个）

| 命令 | 功能 | 对应IOCTL |
|------|------|----------|
| `SuCreateSpace` | 创建虚拟磁盘 | 0xE7C40C~0xE7C430 |
| `SuCreateSpaceEstimate` | 估算创建空间需求 | 0xE7C40C, 0xE7C410 |
| `SuDeleteSpace` | 删除虚拟磁盘 | 0xE7C418, 0xE7C424 |
| `SuSetSpace` | 设置空间属性 | 0xE7C410, 0xE7C418 |
| `SuSetSpaceAttributes` | 设置空间扩展属性 | 0xE7C424, 0xE7C428 |
| `SuSetSpacePriority` | 设置空间IO优先级 | 0xE7C42C, 0xE7C430 |
| `SuGetSpaceTemplate` | 获取空间模板 | 0xE7C40C, 0xE7C410 |
| `SuAttachSpace` | 附加空间到系统 | 0xE7C418, 0xE7C424 |
| `SuDetachSpace` | 分离空间 | 0xE7C428, 0xE7C42C |
| `SuRepairSpace` | 修复空间 | 0xE7C430 |

#### 3.1.4 物理磁盘管理（Drive）（8个）

| 命令 | 功能 | 对应IOCTL |
|------|------|----------|
| `SuAddDrives` | 添加物理磁盘到池 | 0xE7C80C~0xE7C830 |
| `SuRemoveDrives` | 从池移除物理磁盘 | 0xE7C818, 0xE7C820 |
| `SuRemoveDrivesEstimate` | 估算移除磁盘影响 | 0xE7C80C, 0xE7C810 |
| `SuReallocateDrives` | 重新分配磁盘数据 | 0xE7C814, 0xE7C81C |
| `SuRetireDrives` | 退役磁盘（标记为待移除） | 0xE7C824, 0xE7C828 |
| `SuResetDrive` | 重置磁盘状态 | 0xE7C82C, 0xE7C830 |
| `SuSetDrive` | 设置磁盘属性 | 0xE7C810, 0xE7C818 |
| `SuRefreshDrive` | 刷新磁盘状态 | 0xE7C814 |

#### 3.1.5 存储层管理（Tier）（4个）

| 命令 | 功能 | 对应IOCTL |
|------|------|----------|
| `SuCreateTier` | 创建存储层 | 0xE7CC0C |
| `SuDeleteTier` | 删除存储层 | 0xE7CC0C |
| `SuSetTier` | 设置层属性 | 0xE7CC0C |
| `SuGetTierTemplate` | 获取层模板 | 0xE7CC0C |
| `SuGetTierFiltered` | 获取过滤后的层列表 | 0xE7CC0C |

#### 3.1.6 机箱管理（Enclosure）（2个）

| 命令 | 功能 | 对应IOCTL |
|------|------|----------|
| `SuSetEnclosure` | 设置机箱属性 | 0xE7D00C |
| `SuRefreshEnclosure` | 刷新机箱状态 | 0xE7D00C |

#### 3.1.7 任务管理（Task）（2个）

| 命令 | 功能 | 对应IOCTL |
|------|------|----------|
| `SuStopTask` | 停止后台任务 | 0xE7D40C~0xE7D414 |
| `SuRebalancePool` | 重新平衡池数据 | 0xE7D40C, 0xE7D410 |
| `SuRebalanceMetadata` | 重新平衡元数据 | 0xE7D414 |

#### 3.1.8 其他（1个）

| 命令 | 功能 | 对应IOCTL |
|------|------|----------|
| `SetParameter` | 设置注册表参数 | 注册表API |

### 3.2 对象查询模式

除了38个 Su* 命令，spaceutil 还支持一种通用的对象查询模式，通过以下对象类型标识符：

| 对象类型 | 标识符 | 说明 |
|---------|--------|------|
| 控制对象 | `Control.Find` | 查询全局控制参数 |
| 磁盘对象 | `Drive.Find` | 查询物理磁盘 |
| 机箱对象 | `Enclosure.Find` | 查询机箱/机柜 |
| 池对象 | `Pool.Find` | 查询存储池 |
| 空间对象 | `Space.Find` | 查询虚拟磁盘 |
| 空间新建 | `Space.New` | 查询新建空间参数 |
| 任务对象 | `Task.Find` | 查询后台任务 |
| 层对象 | `Tier.Find` | 查询存储层 |

这些 `.Find` 标识符用于查询操作，配合过滤参数返回匹配的对象列表。

### 3.3 命令表结构

命令表是一个静态数组，每个条目包含：
```c
typedef struct _SPACEUTIL_COMMAND {
    LPCWSTR  pszCommandName;    // 命令名 (如 L"SuCreatePool")
    UINT     uDescriptionId;     // 描述资源ID
    UINT     uReserved1;
    UINT     uReserved2;
    PFN_CMD  pfnHandler;         // 命令处理函数指针
} SPACEUTIL_COMMAND, *PSPACEUTIL_COMMAND;

SPACEUTIL_COMMAND g_CommandTable[38] = { ... };
```

wmain 通过遍历命令表匹配命令名，然后调用对应的处理函数。

---

## 4. IOCTL 接口深度分析

### 4.1 IOCTL 码格式

Windows IOCTL 码使用 `CTL_CODE` 宏定义：
```c
#define CTL_CODE(DeviceType, Function, Method, Access) (
    ((DeviceType) << 16) | ((Access) << 14) | ((Function) << 2) | (Method)
)
```

| 字段 | 位宽 | 说明 |
|------|------|------|
| DeviceType | 16位 (bit 16-31) | 设备类型 |
| Access | 2位 (bit 14-15) | 所需访问权限 |
| Function | 12位 (bit 2-13) | 功能码 |
| Method | 2位 (bit 0-1) | 数据传输方式 |

### 4.2 53个IOCTL码完整分类

#### 4.2.1 Spaceport 私有IOCTL（DeviceType = 0xE7 = 231）— 46个

**基础控制类（Function 0x001-0x002）**：
| IOCTL | Function | Method | Access | 功能推断 |
|-------|----------|--------|--------|---------|
| 0xE70004 | 0x001 | BUFFERED | ANY | 初始化/查询驱动版本 |
| 0xE70008 | 0x002 | BUFFERED | ANY | 设置/获取全局参数 |

**控制参数类（Function 0x101-0x107）**：
| IOCTL | Function | Method | Access | 功能推断 |
|-------|----------|--------|--------|---------|
| 0xE70404 | 0x101 | BUFFERED | ANY | 获取控制参数 |
| 0xE70408 | 0x102 | BUFFERED | ANY | 设置控制参数 |
| 0xE7041C | 0x107 | OUT_DIRECT | ANY | 枚举控制对象 |

**池操作类（Function 0x201-0x202）**：
| IOCTL | Function | Method | Access | 功能推断 |
|-------|----------|--------|--------|---------|
| 0xE70804 | 0x201 | BUFFERED | ANY | 池操作请求 |
| 0xE70808 | 0x202 | BUFFERED | ANY | 池状态查询 |

**空间操作类（Function 0x301-0x302）**：
| IOCTL | Function | Method | Access | 功能推断 |
|-------|----------|--------|--------|---------|
| 0xE70C04 | 0x301 | BUFFERED | ANY | 空间操作请求 |
| 0xE70C08 | 0x302 | BUFFERED | ANY | 空间状态查询 |

**磁盘操作类（Function 0x401-0x404）**：
| IOCTL | Function | Method | Access | 功能推断 |
|-------|----------|--------|--------|---------|
| 0xE71004 | 0x401 | BUFFERED | ANY | 磁盘操作请求 |
| 0xE71008 | 0x402 | BUFFERED | ANY | 磁盘状态查询 |
| 0xE71010 | 0x404 | BUFFERED | ANY | 磁盘枚举 |

**层操作类（Function 0x501-0x502）**：
| IOCTL | Function | Method | Access | 功能推断 |
|-------|----------|--------|--------|---------|
| 0xE71404 | 0x501 | BUFFERED | ANY | 层操作请求 |
| 0xE71408 | 0x502 | BUFFERED | ANY | 层状态查询 |

**任务操作类（Function 0x601）**：
| IOCTL | Function | Method | Access | 功能推断 |
|-------|----------|--------|--------|---------|
| 0xE71804 | 0x601 | BUFFERED | ANY | 任务操作/查询 |

**池详细操作类（Function 0xF003-0xF008，Access=READ）**：
| IOCTL | Function | Method | Access | 功能推断 |
|-------|----------|--------|--------|---------|
| 0xE7C00C | 0xF003 | NEITHER | READ | 池属性查询 |
| 0xE7C010 | 0xF004 | BUFFERED | READ | 池配置获取 |
| 0xE7C014 | 0xF005 | OUT_DIRECT | READ | 池数据读取 |
| 0xE7C018 | 0xF006 | BUFFERED | READ | 池状态设置 |
| 0xE7C01C | 0xF007 | OUT_DIRECT | READ | 池数据写入 |
| 0xE7C020 | 0xF008 | BUFFERED | READ | 池扩展属性 |

**空间详细操作类（Function 0xF103-0xF10C，Access=READ）**：
| IOCTL | Function | Method | Access | 功能推断 |
|-------|----------|--------|--------|---------|
| 0xE7C40C | 0xF103 | NEITHER | READ | 空间属性查询 |
| 0xE7C410 | 0xF104 | BUFFERED | READ | 空间配置获取 |
| 0xE7C418 | 0xF106 | BUFFERED | READ | 空间状态设置 |
| 0xE7C424 | 0xF109 | BUFFERED | READ | 空间扩展属性 |
| 0xE7C428 | 0xF10A | OUT_DIRECT | READ | 空间数据读取 |
| 0xE7C42C | 0xF10B | OUT_DIRECT | READ | 空间数据写入 |
| 0xE7C430 | 0xF10C | BUFFERED | READ | 空间修复操作 |

**磁盘详细操作类（Function 0xF203-0xF20C，Access=READ）**：
| IOCTL | Function | Method | Access | 功能推断 |
|-------|----------|--------|--------|---------|
| 0xE7C80C | 0xF203 | NEITHER | READ | 磁盘属性查询 |
| 0xE7C810 | 0xF204 | BUFFERED | READ | 磁盘配置获取 |
| 0xE7C814 | 0xF205 | OUT_DIRECT | READ | 磁盘数据读取 |
| 0xE7C818 | 0xF206 | BUFFERED | READ | 磁盘状态设置 |
| 0xE7C81C | 0xF207 | OUT_DIRECT | READ | 磁盘数据写入 |
| 0xE7C820 | 0xF208 | BUFFERED | READ | 磁盘扩展属性 |
| 0xE7C824 | 0xF209 | BUFFERED | READ | 磁盘退役操作 |
| 0xE7C828 | 0xF20A | OUT_DIRECT | READ | 磁盘退役确认 |
| 0xE7C82C | 0xF20B | OUT_DIRECT | READ | 磁盘重置 |
| 0xE7C830 | 0xF20C | BUFFERED | READ | 磁盘重置确认 |

**层详细操作类（Function 0xF303，Access=READ）**：
| IOCTL | Function | Method | Access | 功能推断 |
|-------|----------|--------|--------|---------|
| 0xE7CC0C | 0xF303 | NEITHER | READ | 层操作（增删改查） |

**机箱详细操作类（Function 0xF403，Access=READ）**：
| IOCTL | Function | Method | Access | 功能推断 |
|-------|----------|--------|--------|---------|
| 0xE7D00C | 0xF403 | NEITHER | READ | 机箱操作（设置/刷新） |

**任务详细操作类（Function 0xF503-0xF505，Access=READ）**：
| IOCTL | Function | Method | Access | 功能推断 |
|-------|----------|--------|--------|---------|
| 0xE7D40C | 0xF503 | NEITHER | READ | 任务查询 |
| 0xE7D410 | 0xF504 | BUFFERED | READ | 任务控制 |
| 0xE7D414 | 0xF505 | OUT_DIRECT | READ | 任务数据 |

**其他操作类（Function 0xF602，Access=READ）**：
| IOCTL | Function | Method | Access | 功能推断 |
|-------|----------|--------|--------|---------|
| 0xE7D808 | 0xF602 | BUFFERED | READ | 枚举/刷新操作 |

#### 4.2.2 VolMgr IOCTL（DeviceType = 0x70 = 112）— 5个

| IOCTL | Function | Method | Access | 功能推断 |
|-------|----------|--------|--------|---------|
| 0x70050 | 0x014 | BUFFERED | ANY | 卷管理器查询 |
| 0x70140 | 0x050 | BUFFERED | ANY | 卷管理器配置 |
| 0x70C020 | 0xF008 | BUFFERED | READ | 卷扩展属性 |
| 0x70C024 | 0xF009 | BUFFERED | READ | 卷属性设置 |
| 0x7C0F4 | 0xF03D | OUT_DIRECT | READ | 卷数据操作 |

#### 4.2.3 其他设备IOCTL — 4个

| IOCTL | DeviceType | 功能推断 |
|-------|-----------|---------|
| 0x2D1104 | 0xB4 (180) | 存储类驱动查询 |
| 0x2DDC9C | 0xB7 (183) | 存储类驱动操作 |
| 0x76433C | 0x1D9 (473) | 第三方存储驱动 |
| 0x76C3B0 | 0x1DB (475) | 第三方存储驱动 |

### 4.3 IOCTL 数据传输方式

| Method | 值 | 说明 | spaceutil中的使用 |
|--------|-----|------|------------------|
| METHOD_BUFFERED | 0 | I/O管理器分配系统缓冲区，复制数据 | 最常用（~35个），用于配置/状态 |
| METHOD_IN_DIRECT | 1 | 输入缓冲区直接映射 | 未使用 |
| METHOD_OUT_DIRECT | 2 | 输出缓冲区直接映射 | ~10个，用于大数据读取 |
| METHOD_NEITHER | 3 | 不使用缓冲区，驱动直接处理用户指针 | ~5个，用于高性能查询 |

---

## 5. 设备交互原理

### 5.1 三个设备路径

spaceutil.exe 与三个内核设备交互：

#### 5.1.1 `\\.\Spaceport` — 主设备

```c
// 全局句柄
HANDLE g_hSpaceport = INVALID_HANDLE_VALUE;

// 打开设备
g_hSpaceport = CreateFileW(
    L"\\\\.\\Spaceport",
    GENERIC_READ | GENERIC_WRITE,   // 0xC0000000
    FILE_SHARE_READ | FILE_SHARE_WRITE,  // 0x3
    NULL,
    OPEN_EXISTING,                    // 0x3
    0,
    NULL);
```

这是 spaceport.sys 驱动创建的控制设备对象，所有 Storage Spaces 管理操作通过此设备的 IOCTL 完成。

#### 5.1.2 `\\.\VolMgrControl` — 卷管理器设备

```c
HANDLE hVolMgr = CreateFileW(
    L"\\\\.\\VolMgrControl",
    GENERIC_READ | GENERIC_WRITE,
    FILE_SHARE_READ | FILE_SHARE_WRITE,
    NULL,
    OPEN_EXISTING,
    0,
    NULL);
```

volmgr.sys 驱动的控制设备，用于卷管理器级别的操作（如卷扩展、卷属性查询）。

#### 5.1.3 `\\.\PhysicalDrive%d` — 物理磁盘设备

```c
WCHAR szPath[32];
swprintf_s(szPath, L"\\\\.\\PhysicalDrive%d", dwDriveNumber);

HANDLE hDrive = CreateFileW(
    szPath,
    GENERIC_READ | GENERIC_WRITE,
    FILE_SHARE_READ | FILE_SHARE_WRITE,
    NULL,
    OPEN_EXISTING,
    0,
    NULL);
```

直接打开物理磁盘设备，用于磁盘级别的操作（如磁盘重置、磁盘健康查询）。

### 5.2 驱动类型选择

spaceutil 支持两种存储驱动后端，通过命令参数选择：

| 驱动名 | 说明 | 使用场景 |
|--------|------|---------|
| `spaceport` | Storage Spaces 端口驱动 | 标准 Storage Spaces 配置 |
| `clusdisk` | 集群磁盘驱动 | 故障转移集群场景 |

选择逻辑：
```c
if (_wcsicmp(pszDriverType, L"clusdisk") == 0) {
    // 使用集群磁盘驱动路径
} else if (_wcsicmp(pszDriverType, L"spaceport") == 0) {
    // 使用 spaceport 驱动路径
}
```

### 5.3 IOCTL 调用模式

典型的 IOCTL 调用流程：
```c
BOOL SpaceUtilIoctl(
    HANDLE  hDevice,
    DWORD   dwIoControlCode,
    LPVOID  lpInBuffer,
    DWORD   nInBufferSize,
    LPVOID  lpOutBuffer,
    DWORD   nOutBufferSize,
    LPDWORD lpBytesReturned)
{
    BOOL bResult = DeviceIoControl(
        hDevice,
        dwIoControlCode,
        lpInBuffer,
        nInBufferSize,
        lpOutBuffer,
        nOutBufferSize,
        lpBytesReturned,
        NULL);  // 同步调用 (lpOverlapped = NULL)

    if (!bResult) {
        DWORD dwError = GetLastError();
        // 错误处理和日志
    }
    return bResult;
}
```

所有 IOCTL 调用都是**同步**的（lpOverlapped = NULL），spaceutil 不使用异步 IO。

---

## 6. 注册表交互

### 6.1 注册表路径

```
HKLM\SYSTEM\CurrentControlSet\Services\spaceport\Parameters
```

### 6.2 操作流程

```c
// 1. 打开注册表键
HKEY hKey;
LONG lResult = RegOpenKeyExW(
    HKEY_LOCAL_MACHINE,           // 0x80000002
    L"SYSTEM\\CurrentControlSet\\Services\\spaceport\\Parameters",
    0,
    KEY_SET_VALUE,                 // 只需要写权限
    &hKey);

// 2. 设置 DWORD 值
DWORD dwValue = ...;
lResult = RegSetValueExW(
    hKey,
    pszValueName,                  // 参数名
    0,
    REG_DWORD,
    (LPBYTE)&dwValue,
    sizeof(DWORD));

// 3. 关闭键
RegCloseKey(hKey);
```

### 6.3 SetParameter 命令

`SetParameter` 命令用于设置 spaceport 驱动的注册表参数。这些参数在驱动加载时读取，控制驱动的运行时行为：
- 调试标志
- 性能参数
- 功能开关
- 日志级别

---

## 7. 参数系统

### 7.1 命令行格式

```
spaceutil <Command> [Option1 Value1] [Option2 Value2] ...
```

- 选项名前可加 `-` 或 `/` 前缀
- `-?` 显示帮助
- 选项和值成对出现

### 7.2 参数类型

| 类型 | 格式说明 | 示例 | 解析方式 |
|------|---------|------|---------|
| Boolean | `{True, False}` | `True`, `False` | `_wcsicmp` 比较 |
| Guid | `{Guid}` | 32字符十六进制（无连字符） | `swscanf` 或手动解析 |
| String | `{String}` | 任意字符串 | 直接使用 |
| Number | `Number` | 十进制整数 | `_wtoi` / `wcstoul` |
| Size | `Size with optional unit suffix: K, M...` | `1024`, `4K`, `2M` | 解析数字+单位后缀 |
| Auto | `Auto` | `Auto` | 特殊标记值 |

### 7.3 Size 单位后缀

```c
// 伪代码
DWORD ParseSize(LPCWSTR pszSize) {
    DWORD dwValue = _wtoi(pszSize);
    WCHAR chSuffix = ...; // 最后一个字符

    switch (chSuffix) {
        case L'K': case L'k': dwValue *= 1024; break;
        case L'M': case L'm': dwValue *= 1024 * 1024; break;
        case L'G': case L'g': dwValue *= 1024 * 1024 * 1024; break;
        // 无后缀: 字节
    }
    return dwValue;
}
```

### 7.4 配置对象

spaceutil 内部维护 28 个（0x1C）配置对象，每个对象对应一个可设置的参数。参数通过 `FUN_004047d5`（rs5）设置到全局配置数组，命令处理函数从配置数组读取参数值。

---

## 8. 版本演化分析

### 8.1 函数数量增长趋势

```
rs1: 216 ──┐
rs2: 221 ──┤ +5 (2.3%)
rs3: 270 ──┤ +49 (22.2%)  ← 大幅增长
rs4: 295 ──┤ +25 (9.3%)
rs5: 303 ──┘ +8 (2.7%)
```

### 8.2 各版本新增功能推断

#### rs1 → rs2 (+5函数)
- 小幅改进，可能是 bug 修复和小功能增强
- IOCTL 接口不变

#### rs2 → rs3 (+49函数，最大增幅)
- **命令名混淆**：rs3 开始命令名不再以明文存储
- 可能引入了新的对象类型或属性
- 内部数据结构重构
- 新增错误处理和日志函数

#### rs3 → rs4 (+25函数)
- 持续功能增强
- 可能新增存储层（Tier）相关功能
- 改进参数验证

#### rs4 → rs5 (+8函数)
- 小幅改进
- rs5 是当前 ADK 版本，功能趋于稳定

### 8.3 命令名混淆分析

rs1/rs2 的命令名以明文宽字符串存储在 `.rdata` 段：
```
L"SuAddDrives", L"SuAttachSpace", L"SuCreatePool", ...
```

rs3/rs4/rs5 中这些字符串消失，只有：
```
L"Auto", L"clusdisk", L"False", L"Number", L"spaceport", L"True"
```

**可能的混淆方式**：
1. **加密存储**：命令名加密存储在数据段，运行时解密后使用
2. **资源段存储**：命令名移到资源段，通过资源ID加载
3. **动态生成**：命令名通过代码逻辑动态生成
4. **字符串表编码**：使用自定义编码的字符串表

**对重建项目的影响**：
- 重建时应使用明文命令名（参考 rs1/rs2）
- 混淆不影响功能，只是增加了逆向难度
- IOCTL 接口不变，因此功能完全兼容

### 8.4 IOCTL 接口稳定性

53个IOCTL码在所有5个版本中**完全相同**，这表明：
1. spaceport.sys 驱动的 IOCTL 接口设计非常稳定
2. 新增功能通过现有IOCTL的新子命令/新数据结构实现，而非新增IOCTL码
3. 用户态工具与内核驱动之间有严格的版本协商机制（可能通过 `SuInitialize` 或 0xE70004 IOCTL 交换版本信息）

---

## 9. 工作原理完整流程

### 9.1 启动流程

```
1. CRT 启动 (_mainCRTStartup)
   └→ 调用 __wgetmainargs 获取 argc/argv/envp

2. wmain 入口
   ├→ 从 argv[0] 提取程序名 (最后一个 \ 之后)
   ├→ 检查 argc > 1 (是否有命令参数)
   │   ├→ 无命令 → 调用 Usage() 打印38个命令用法
   │   └→ 有命令 → 去掉前导 - 或 /
   │
   ├→ 遍历命令表 (38个条目)
   │   └→ _wcsicmp 匹配命令名
   │       ├→ 未找到 → Usage()
   │       └→ 找到 → 记录命令索引
   │
   ├→ 初始化 (FUN_0040964d)
   │   ├→ CreateFileW(L"\\.\Spaceport") 打开主设备
   │   ├→ 检查驱动版本 (IOCTL 0xE70004)
   │   └→ 初始化全局配置对象 (28个)
   │
   ├→ 解析剩余参数 (argv+2, argc-2)
   │   └→ 逐个解析 key-value 对，设置到配置对象
   │
   └→ 调用命令处理函数 (handlerTable[index])
       ├→ 从配置对象读取参数
       ├→ 构造 IOCTL 输入缓冲区
       ├→ DeviceIoControl 调用
       ├→ 处理输出缓冲区
       └→ 格式化输出结果
```

### 9.2 典型操作流程（以 SuCreatePool 为例）

```
spaceutil SuCreatePool PoolId={Guid} DriveIds={Guid},{Guid} ...
                          ProvisionType=Fixed Resiliency=Mirror ...

1. 解析参数
   ├→ PoolId → GUID (存储池唯一标识)
   ├→ DriveIds → GUID 数组 (物理磁盘列表)
   ├→ ProvisionType → Thin/Fixed (精简/固定配置)
   ├→ Resiliency → Simple/Mirror/Parity (冗余类型)
   └→ 其他属性 (列数、条带大小等)

2. 构造 IOCTL 输入缓冲区
   ├→ SPACEPORT_CREATE_POOL_INPUT 结构体
   │   ├→ PoolId (GUID)
   │   ├→ DriveCount (DWORD)
   │   ├→ DriveIds[DriveCount] (GUID 数组)
   │   ├→ ProvisioningType (ENUM)
   │   ├→ ResiliencyType (ENUM)
   │   ├─ NumberOfColumns (DWORD)
   │   └→ InterleaveSize (DWORD)

3. DeviceIoControl(hSpaceport, 0xE7C00C/0xE7C010, ...)
   └→ spaceport.sys 内核处理
       ├→ 验证磁盘可用性
       ├→ 分配池元数据区域
       ├→ 在每块磁盘上创建池元数据
       └→ 返回池句柄/状态

4. 处理输出
   ├→ 检查返回状态
   ├→ 输出池ID和状态
   └→ 错误处理 (磁盘不可用、空间不足等)
```

### 9.3 对象查询流程（以 Pool.Find 为例）

```
spaceutil Pool.Find Filter=...

1. 构造查询输入缓冲区
   ├→ SPACEPORT_QUERY_INPUT
   │   ├→ ObjectType = Pool
   │   ├→ FilterCount
   │   └→ Filters[] (属性过滤条件)

2. DeviceIoControl(hSpaceport, 0xE7C00C, ...)
   └→ 驱动返回匹配的池列表

3. 枚举输出
   ├→ 遍历返回的池数组
   ├→ 格式化每个池的属性
   └→ 输出到控制台
```

### 9.4 安全描述符处理

spaceutil 使用 `ConvertSecurityDescriptorToStringSecurityDescriptor` API 将安全描述符转换为字符串格式（SDDL），用于：
- 查询池/空间的安全描述符
- 设置池/空间的访问控制
- 输出安全信息

---

## 10. 重建项目建议

### 10.1 必须重建的组件

| 组件 | 优先级 | 重建难度 | 说明 |
|------|--------|---------|------|
| 命令行解析框架 | 高 | 低 | 38命令表 + 参数解析 |
| 设备打开/关闭 | 高 | 低 | 3个设备路径的 CreateFile/CloseHandle |
| IOCTL 封装层 | 高 | 中 | 53个IOCTL的类型安全封装 |
| 38个命令处理函数 | 高 | 高 | 每个命令的输入/输出处理 |
| 参数类型解析 | 中 | 低 | Boolean/Guid/String/Number/Size/Auto |
| 注册表参数设置 | 中 | 低 | SetParameter 命令 |
| 用法输出 | 低 | 低 | 自动对齐的命令列表 |

### 10.2 IOCTL 数据结构重建

由于 IOCTL 码稳定，但数据结构未在反编译中完整还原（需要 IDA 深度分析结构体），建议：

1. **参考 Microsoft 公开文档**：Storage Spaces 的部分 IOCTL 在 Windows Driver Kit 文档中有定义
2. **参考 rs1/rs2 明文**：rs1/rs2 的命令名和字符串更完整，有助于推断数据结构
3. **逐步逆向**：从简单命令（如 SuInitialize、SuSetControl）开始，逐步还原复杂命令
4. **结构体对齐**：注意 x86/x64 结构体对齐差异（ADK 是 i386 版本）

### 10.3 版本兼容策略

由于5个版本的 IOCTL 码相同，但数据结构可能有差异：
- 使用版本协商机制（SuInitialize / 0xE70004）检测驱动版本
- 根据版本选择对应的数据结构定义
- 或只支持 rs5（最新版本），简化实现

### 10.4 替代方案

如果不需要完整重建 spaceutil.exe，可以考虑：
1. **PowerShell Storage Spaces cmdlets**：`New-StoragePool`、`New-VirtualDisk` 等，底层也是调用相同的 IOCTL
2. **WMI/CIM**：`ROOT\Microsoft\Windows\Storage` 命名空间提供 Storage Spaces 管理
3. **直接调用 IOCTL**：在自定义程序中直接使用 DeviceIoControl，参考本文档的 IOCTL 列表

### 10.5 关键数据结构推断

基于命令名和 IOCTL 分类，以下是关键数据结构的推断：

```c
// 存储池属性
typedef struct _SPACEPORT_POOL_INFO {
    GUID    PoolId;
    WCHAR   PoolName[64];
    DWORD   PoolStatus;
    DWORD   ProvisioningType;   // Thin=1, Fixed=2
    DWORD   ResiliencyType;     // Simple=1, Mirror=2, Parity=3
    DWORD   NumberOfColumns;
    DWORDLONG InterleaveSize;
    DWORDLONG TotalManagedSpace;
    DWORDLONG RemainingManagedSpace;
    DWORD   DriveCount;
    // ... 更多字段
} SPACEPORT_POOL_INFO, *PSPACEPORT_POOL_INFO;

// 虚拟磁盘属性
typedef struct _SPACEPORT_SPACE_INFO {
    GUID    SpaceId;
    GUID    PoolId;
    WCHAR   SpaceName[64];
    DWORD   SpaceStatus;
    DWORD   ProvisioningType;
    DWORD   ResiliencyType;
    DWORD   NumberOfColumns;
    DWORDLONG InterleaveSize;
    DWORDLONG Size;
    DWORDLONG AllocatedSize;
    DWORD   IOPriority;
    // ... 更多字段
} SPACEPORT_SPACE_INFO, *PSPACEPORT_SPACE_INFO;

// 物理磁盘属性
typedef struct _SPACEPORT_DRIVE_INFO {
    GUID    DriveId;
    GUID    PoolId;
    WCHAR   DriveFriendlyName[64];
    DWORD   DriveStatus;
    DWORD   HealthStatus;
    DWORD   UsageType;
    DWORDLONG Size;
    DWORDLONG AllocatedSize;
    DWORD   SlotNumber;
    // ... 更多字段
} SPACEPORT_DRIVE_INFO, *PSPACEPORT_DRIVE_INFO;
```

---

## 附录A: 完整文件清单

| 文件名 | 原始大小 | 反编译大小 | 函数数 |
|--------|---------|-----------|--------|
| rs1_spaceutil.exe | 124,928B | 388KB | 216 |
| rs2_spaceutil.exe | 123,904B | 395KB | 221 |
| rs3_spaceutil.exe | 104,960B | 374KB | 270 |
| rs4_spaceutil.exe | 107,520B | 396KB | 295 |
| rs5_spaceutil.exe | 113,664B | 411KB | 303 |

## 附录B: IOCTL 码速查表

```
基础控制:   0xE70004, 0xE70008
控制参数:   0xE70404, 0xE70408, 0xE7041C
池操作:     0xE70804, 0xE70808
空间操作:   0xE70C04, 0xE70C08
磁盘操作:   0xE71004, 0xE71008, 0xE71010
层操作:     0xE71404, 0xE71408
任务操作:   0xE71804
池详细:     0xE7C00C, 0xE7C010, 0xE7C014, 0xE7C018, 0xE7C01C, 0xE7C020
空间详细:   0xE7C40C, 0xE7C410, 0xE7C418, 0xE7C424, 0xE7C428, 0xE7C42C, 0xE7C430
磁盘详细:   0xE7C80C~0xE7C830 (10个)
层详细:     0xE7CC0C
机箱详细:   0xE7D00C
任务详细:   0xE7D40C, 0xE7D410, 0xE7D414
其他:       0xE7D808
VolMgr:     0x70050, 0x70140, 0x70C020, 0x70C024, 0x7C0F4
其他设备:   0x2D1104, 0x2DDC9C, 0x76433C, 0x76C3B0
```

---

*文档结束 — 生成于 2026-08-27*
*spaceutil.exe 五版本深度对比分析完成，覆盖38命令、53 IOCTL、3设备路径、完整工作原理和重建建议*
