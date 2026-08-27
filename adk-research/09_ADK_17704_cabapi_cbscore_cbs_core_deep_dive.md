# ADK 10.0.17704.1000 — cabapi.dll + cbscore.dll CBS 核心引擎深度分析

> 文档版本: 1.0 | 日期: 2026-08-27
> 反编译源: `ida_decompiled\native\cabapi.c` (357KB, 415函数), `ida_decompiled\native\cbscore.c` (9.7MB, 230+核心类函数)

---

## 1. 执行摘要

cabapi.dll 和 cbscore.dll 构成了 Windows CBS (Component Based Servicing) 的核心引擎，负责包的安装、卸载、配置和状态管理。这两个 DLL 在镜像构建期间被 imageapp/imaging 间接调用，在运行时被 UpdateDLL 和 TrustedInstaller 调用。

| DLL | 大小 | 函数数 | 职责 |
|-----|------|--------|------|
| cabapi.dll | 357KB | 415 | CBS API 层：WIL 封装、IULogger 日志、staging/commit/reset 子阶段映射、CBS 公共接口 |
| cbscore.dll | 9.7MB | 230+核心类 | CBS 核心引擎：CCbsSession/CCbsTransaction/CCbsPackage/CCbsInstaller/CCbsFeature/CCbsVuln 等完整类体系 |

---

## 2. cabapi.dll 架构

### 2.1 技术栈

- **WIL (Windows Implementation Library)**: 完整的 WIL 错误处理框架（ResultMacros, FailFast, Exception Handling）
- **IULogger**: 统一日志系统（IU = Image Update），支持文件日志和 ETW 日志
- **IUProgress**: 进度报告系统
- **STRSAFE**: 安全字符串操作（StringCopyWorkerW, StringCchVPrintfW）

### 2.2 子阶段映射系统

cabapi.dll 定义了完整的 CBS 操作子阶段映射：

```cpp
// Staging 子阶段
STAGING_SUBPHASES_MAP → vector<pair<IUSubphase, DWORD>>
  从 STAGING_SUBPHASES_PAIRS 到 RESET_FINALIZE_SUBPHASES_PAIRS

// Commit 子阶段
COMMIT_SUBPHASES_MAP → vector<pair<IUSubphase, DWORD>>
  从 COMMIT_SUBPHASES_PAIRS 到 RESET_SUBPHASES_PAIRS

// Reset 子阶段
RESET_SUBPHASES_MAP → vector<pair<IUSubphase, DWORD>>
  从 RESET_SUBPHASES_PAIRS 到 STAGING_SUBPHASES_PAIRS

// Reset+Commit 子阶段
RESET_COMMIT_SUBPHASES_MAP

// Reset+Staging 子阶段
RESET_STAGING_SUBPHASES_MAP

// Reset+Finalize 子阶段
RESET_FINALIZE_SUBPHASES_MAP

// 主映射: IUPhase → vector<pair<IUSubphase, DWORD>>
PHASE_SUBPHASE_MAP → map<IUPhase, vector<pair<IUSubphase, DWORD>>>
```

**IUPhase 枚举**（推断）:
- Staging = 暂存阶段
- Commit = 提交阶段
- Reset = 重置阶段
- Finalize = 完成阶段

**IUSubphase 枚举**（推断）:
- 各种子操作：文件提取、注册表写入、服务配置、驱动安装等

### 2.3 关键全局对象

| 对象 | 说明 |
|------|------|
| `IULogger::s_Instance` | 全局日志单例 |
| `IULogger::s_wsLogPath` | 日志文件路径 |
| `IULogger::s_wsEtwLogPath` | ETW 日志路径 |
| `s_LoggerCriticalSection` | 日志临界区 |
| `IULogAdapter::s_Instance` | 日志适配器单例 |
| `IUProgress::s_Instance` | 进度报告单例 |
| `STAGING_SUBPHASES_MAP` | Staging 子阶段映射 |
| `COMMIT_SUBPHASES_MAP` | Commit 子阶段映射 |
| `RESET_SUBPHASES_MAP` | Reset 子阶段映射 |
| `PHASE_SUBPHASE_MAP` | 主阶段→子阶段映射 |
| `pszCabPath` | CAB 路径（WIL 错误日志中引用） |

---

## 3. cbscore.dll 架构

### 3.1 核心类体系

cbscore.dll 实现了完整的 CBS 核心类层次，基于 COM-style IUnknown：

```
CCbsIUnknownImpl<IInterface,>  (COM 基类: QueryInterface/AddRef/Release)
│
├─ CCbsSession          (CBS 会话: 包操作的上下文)
├─ CCbsTransaction      (CBS 事务: 原子性操作)
├─ CCbsPackage          (CBS 包: .cab 包的抽象)
├─ CCbsInstaller        (CBS 安装器: 执行实际安装)
├─ CCbsFeature          (CBS 功能: Feature On Demand)
├─ CCbsVuln             (CBS 漏洞: 安全更新)
├─ CCbsStack<T>         (CBS 栈: 模板化容器)
├─ CCbsSessionManager   (会话管理器: 全局单例)
├─ CCbsStoreParameters  (存储参数)
├─ CCbsStoreObject      (存储对象: 注册表/文件抽象)
├─ CCbsLockMonitor      (锁监视器: 调试/死锁检测)
├─ CCbsArrayBase<T>     (数组基类)
├─ CCbsArray<T>         (动态数组)
├─ CCbsInterfaceArray<T> (接口数组)
└─ PackageStore          (包存储: 会话 ID 管理)
```

### 3.2 CCbsSession 类

**关键方法**:

| 方法 | 偏移/说明 |
|------|----------|
| `IsComplete()` | 行47338: 检查会话是否完成。完成状态: state==208/224/240 且 pendingCount==totalCount |
| `IsFODRetryCandidate()` | 行33003: FOD (Feature on Demand) 重试候选检查。条件: offset+728!=0 且 state!=17 |

**状态机**（推断，从 IsComplete 反推）:

| 状态值 | 含义 |
|--------|------|
| 17 | 进行中 (In Progress) |
| 208 | 暂存完成 (Staging Complete) |
| 224 | 提交完成 (Commit Complete) |
| 240 | 最终完成 (Finalize Complete) |

**CCbsSession 布局**（推断）:

| 偏移 | 类型 | 字段 |
|------|------|------|
| +0 | CCbsIUnknownImpl | COM 基类 |
| +48 | DWORD | 会话状态 (state) |
| +75 | DWORD | 已完成操作计数 |
| +77 | DWORD | 总操作计数 |
| +78 | HRESULT | 最后错误码 |
| +183 | DWORD | FOD 相关状态 |
| +728 | BYTE | FOD 重试标志 |

### 3.3 CCbsSessionManager 类

**构造函数**（行47387, 大小 0x120 = 288字节）:

```cpp
CCbsSessionManager::CCbsSessionManager() {
    // 4x CCbsInterfaceArray<CCbsFeaturePackage*>
    CCbsInterfaceArray<CCbsFeaturePackage*>(this);           // +0
    CCbsInterfaceArray<CCbsFeaturePackage*>(this+8);         // +8
    CCbsInterfaceArray<CCbsFeaturePackage*>(this+16);        // +16
    CCbsInterfaceArray<CCbsFeaturePackage*>(this+24);        // +24

    // 临界区
    InitializeCriticalSection(this+128);  // "CCbsSessionManager"
    CCbsLockMonitor::AddNewLock("CCbsSessionManager", 0x0B, this+128);

    InitializeCriticalSection(this+152);  // "CSIInventoryCriticalSection"
    CCbsLockMonitor::AddNewLock("CSIInventoryCriticalSection", 0x40, this+152);

    // 更多数组
    CCbsInterfaceArray<CCbsFeaturePackage*>(this+54);
    CCbsInterfaceArray<ICbsUpdate*>(this+62);
}
```

**全局单例**:
- `pSessionManager` — 全局会话管理器指针
- `SessionManagerInitialize()` — 初始化函数（分配 0x120 字节）

### 3.4 PackageStore 会话 ID 管理

**`PackageStoreGetNextSessionID()`**（行33010, 源码 `packagestore.cpp:273`）:

```cpp
HRESULT PackageStoreGetNextSessionID(
    BOOL    bOffline,       // a1: 0=在线存储, 1=离线(进程内)
    FILETIME* pSessionId)   // a2: 输出会话 ID
{
    CBS_EnterCriticalSection(&vPackageStoreLock);

    if (bOffline) {
        // 离线模式: 用 ProcessID + TickCount 生成 ID
        *pSessionId = MAKE_FILETIME(GetCurrentProcessId(), GetTickCount());
    } else {
        // 在线模式: 从持久化存储读取/递增
        CCbsStoreParameters::GetOnlineStoreObject(1, &storeObj);
        GetSystemTimeAsFileTime(&now);

        // 读取 "SessionIdHigh" 和 "SessionIdLow"
        storeObj.GetValue("SessionIdHigh", &high);
        storeObj.GetValue("SessionIdLow", &low);

        // 组合并递增
        current = (high << 32) | low;
        if (now <= current) now = current + 1;

        // 写回
        storeObj.SetValue("SessionIdHigh", now.dwHighDateTime);
        storeObj.SetValue("SessionIdLow", now.dwLowDateTime);

        *pSessionId = now;
    }
}
```

**关键设计**:
- 会话 ID 使用 FILETIME 结构（64位时间戳）
- 在线模式持久化到存储（注册表/配置文件），保证跨进程唯一
- 离线模式用 ProcessID+TickCount，保证进程内唯一
- 使用 `vPackageStoreLock` 全局临界区保护

### 3.5 CCbsLockMonitor 锁监视器

cbscore.dll 实现了完整的锁监视器系统，用于死锁检测和调试：

```cpp
CCbsLockMonitor::AddNewLock(
    LPCWSTR name,           // 锁名称
    DWORD   line,           // 代码行号
    CRITICAL_SECTION* pcs)  // 临界区指针
```

已注册的锁:
- `"CCbsSessionManager"` (行 0x0B)
- `"CSIInventoryCriticalSection"` (行 0x40)
- `vPackageStoreLock` (PackageStore 全局锁)

### 3.6 CCbsStack<_XML_TOKEN>

XML 解析栈，用于 CBS manifest (update.mum) 的 XML 解析：

```cpp
class CCbsStack<_XML_TOKEN> {
    CCbsArrayBase<PACKAGE_STORE_CURRENT_AND_PENDING_STATE, CCbsArray<...>> array;
    CCbsIUnknownImpl<ICbsStack,> unknown;
};
```

---

## 4. CBS 操作流程

### 4.1 包安装流程（推断）

```
1. CCbsSessionManager::SessionManagerInitialize()
   → 创建全局会话管理器

2. CCbsSession::Create()
   → 创建新的 CBS 会话
   → PackageStoreGetNextSessionID() 生成会话 ID

3. CCbsPackage::Open(cabPath)
   → 打开 .cab 包
   → 解析 update.mum manifest
   → CCbsStack<_XML_TOKEN> 解析 XML

4. CCbsTransaction::Create(session)
   → 创建事务

5. Staging 阶段 (IUPhase=Staging)
   → 遍历 STAGING_SUBPHASES_MAP
   → 提取文件到临时位置
   → 计算/验证文件 hash (调用 wcp.dll)
   → 验证 catalog 签名 (调用 wcp.dll CCatalog)

6. Commit 阶段 (IUPhase=Commit)
   → 遍历 COMMIT_SUBPHASES_MAP
   → 原子性提交文件到最终位置
   → 写入注册表
   → 配置服务/驱动

7. Finalize 阶段
   → 清理临时文件
   → 更新包状态

8. CCbsSession::IsComplete()
   → 检查 state==240 且 pending==total
```

### 4.2 与其他 DLL 的交互

```
cbscore.dll
    │
    ├─→ wcp.dll (CCatalog, hash验证, 证书验证)
    │    - CCatalog::Create/VerifySigner/VerifyCertChainRoot
    │    - CCSDirectTransaction::VerifyFileHashes
    │
    ├─→ cabapi.dll (CBS API 层, 子阶段映射, 日志)
    │    - IULogger 日志
    │    - PHASE_SUBPHASE_MAP 子阶段管理
    │
    ├─→ UpdateDLL.dll (两阶段更新引擎)
    │    - PrepareUpdate / ExecuteUpdate
    │    - 调用 CBS 会话执行更新
    │
    └─→ TrustedInstaller.exe (CBS 服务宿主)
         - 进程内托管 cbscore
         - 处理远程 CBS 请求
```

---

## 5. 关键源码文件

| 源码路径 | 内容 |
|---------|------|
| `onecore\base\cbs\core\packagestore.cpp` | PackageStore 实现，会话 ID 管理 (行273) |
| `onecore\base\cbs\core\session.cpp` (推断) | CCbsSession 实现 |
| `onecore\base\cbs\core\sessionmanager.cpp` (推断) | CCbsSessionManager 实现 |
| `onecore\base\cbs\core\package.cpp` (推断) | CCbsPackage 实现 |
| `onecore\base\cbs\core\transaction.cpp` (推断) | CCbsTransaction 实现 |
| `onecore\base\cbs\core\installer.cpp` (推断) | CCbsInstaller 实现 |
| `onecore\base\cbs\core\lockmonitor.cpp` (推断) | CCbsLockMonitor 实现 |
| `onecore\base\cbs\core\storeobject.cpp` (推断) | CCbsStoreObject 实现 |

---

## 6. 重建项目 Implications

### 6.1 cabapi.dll 可复用部分

| 组件 | 复用难度 | 说明 |
|------|---------|------|
| WIL 错误处理 | 低 | 开源库，可直接使用 |
| IULogger 日志系统 | 中 | 需要重新实现日志格式 |
| 子阶段映射系统 | 中 | 需要理解 IUPhase/IUSubphase 枚举值 |
| STRSAFE 字符串 | 低 | Windows 自带 |

### 6.2 cbscore.dll 重建难度

cbscore.dll 是整个 CBS 系统中最复杂的组件，9.7MB 反编译代码，230+ 核心类函数。完整重建需要：

1. **COM 类体系**: 所有 CCbs* 类都基于 CCbsIUnknownImpl，需要完整的 COM 接口定义
2. **状态机**: CCbsSession 状态机（17/208/224/240 等状态）
3. **包存储**: PackageStore 持久化层（注册表/文件）
4. **锁监视器**: 死锁检测系统
5. **XML 解析**: CCbsStack<_XML_TOKEN> manifest 解析
6. **子阶段执行**: 与 cabapi.dll 子阶段映射配合的执行引擎

### 6.3 替代方案

对于镜像构建项目，不需要完整重建 cbscore.dll：
- **离线镜像构建**: imageapp/imaging 直接操作文件系统和注册表，不需要完整的 CBS 事务
- **包注入**: 可以直接将 .cab 包的文件提取到镜像中，跳过 CBS 会话管理
- **最小 CBS**: 只需要实现包打开、manifest 解析、文件提取、hash 验证（wcp.dll 已覆盖）

---

## 7. 反编译行号索引

### cabapi.c

| 行号 | 内容 |
|------|------|
| 6-170 | 动态初始化器 (WIL, IULogger, 子阶段映射) |
| 113 | STAGING_SUBPHASES_MAP 初始化 |
| 123 | COMMIT_SUBPHASES_MAP 初始化 |
| 133 | RESET_SUBPHASES_MAP 初始化 |
| 143 | RESET_COMMIT_SUBPHASES_MAP 初始化 |
| 153 | RESET_STAGING_SUBPHASES_MAP 初始化 |
| 163 | RESET_FINALIZE_SUBPHASES_MAP 初始化 |
| 173 | PHASE_SUBPHASE_MAP 初始化 |
| 6472 | StringCopyWorkerW (STRSAFE) |
| 6510 | wil_details_GetNtDllModuleHandle |
| 6525 | wil::details::LogStringPrintf |
| 6540 | wil::GetFailureLogString |

### cbscore.c

| 行号 | 函数 | 说明 |
|------|------|------|
| 32992 | CCbsStack<_XML_TOKEN>::~vector deleting destructor | XML 栈析构 |
| 33003 | CCbsSession::IsFODRetryCandidate | FOD 重试检查 |
| 33010 | PackageStoreGetNextSessionID | 会话 ID 生成 (packagestore.cpp:273) |
| 47321 | CCbsIUnknownImpl<ICbsStack,>::QueryInterface | COM QueryInterface |
| 47331 | CCbsIUnknownImpl<ICbsStack,>::Release | COM Release |
| 47338 | CCbsSession::IsComplete | 会话完成检查 |
| 47354 | SessionManagerInitialize | 会话管理器初始化 |
| 47387 | CCbsSessionManager::CCbsSessionManager | 会话管理器构造 (0x120字节) |

---

*文档结束 — 生成于 2026-08-27，基于 IDA Pro 9.4 反编译 cabapi.dll (357KB) + cbscore.dll (9.7MB)*

*注意: cbscore.dll 是 CBS 核心引擎，代码量巨大（9.7MB），本文档聚焦架构和关键类，完整函数级逆向需要更多轮次分析。*
