# Resource Manager (RM) 架构分析文档

## 概述

NVIDIA 开源 GPU 内核模块中的 Resource Manager (RM) 是一个复杂的资源管理系统，负责管理 GPU 硬件资源、客户端会话、内存分配、命令提交等核心功能。本文档详细分析 RM 的模块架构和资源管理机制。

## 核心架构组件

### 1. Resource Server (RsServer)

**位置**: `src/nvidia/src/libraries/resserv/src/rs_server.c`

Resource Server 是 RM 的核心，负责：
- 管理所有客户端 (RsClient)
- 维护客户端锁机制
- 处理资源分配和释放
- 提供资源查找和访问控制

**关键数据结构**:
```c
struct RsServer {
    PORT_RWLOCK    *pClientListLock;    // 全局客户端列表锁
    RsClientList    clientList;          // 客户端列表
    NvHandle        clientHandleBase;    // 客户端句柄基址
    NvHandle        internalHandleBase;  // 内部句柄基址
    RS_PRIV_LEVEL   privilegeLevel;      // 权限级别
    PORT_MEM_ALLOCATOR *pAllocator;      // 内存分配器
};
```

**主要功能**:
- `serverConstruct()`: 初始化资源服务器
- `serverDestruct()`: 销毁资源服务器
- `serverFreeDomain()`: 释放域中的所有资源
- `serverAllocClient()`: 分配新客户端
- `serverFreeClient()`: 释放客户端

### 2. RMAPI (Resource Manager API)

**位置**: `src/nvidia/src/kernel/rmapi/rmapi.c`

RMAPI 是 RM 对外的主要接口层，提供了多种 API 类型：

**API 类型** (`RMAPI_TYPE`):
```c
- RMAPI_EXTERNAL: 外部用户空间 API
- RMAPI_EXTERNAL_KERNEL: 外部内核 API
- RMAPI_MODS_LOCK_BYPASS: MODS 测试绕过锁
- RMAPI_API_LOCK_INTERNAL: 内部 API 锁
- RMAPI_GPU_LOCK_INTERNAL: 内部 GPU 锁
- RMAPI_STUBS: 存根实现
```

**核心功能**:
```c
NV_STATUS rmapiInitialize(void);        // 初始化 RMAPI
void rmapiShutdown(void);               // 关闭 RMAPI
NV_STATUS rmapiLockAcquire(flags, mod); // 获取 API 锁
void rmapiLockRelease(void);            // 释放 API 锁
```

**全局对象**:
- `g_resServ`: 全局资源服务器实例
- `g_RmApiList[]`: RMAPI 接口数组
- `g_RmApiLock`: API 锁结构

### 3. 客户端管理 (RsClient)

**位置**: `src/nvidia/src/libraries/resserv/src/rs_client.c`

每个客户端代表一个独立的 GPU 使用者（进程或内核模块）。

**CLIENT_ENTRY 结构**:
```c
struct CLIENT_ENTRY {
    PORT_RWLOCK   *pLock;           // 客户端锁
    RsClient      *pClient;         // 客户端对象
    NvHandle       hClient;         // 客户端句柄
    NvU64          lockOwnerTid;    // 锁持有者线程 ID
    NvU32          lockReadOwnerCnt;// 读锁计数
    NvU32          refCount;        // 引用计数
    NvBool         bPendingFree;    // 待释放标志
};
```

**客户端功能**:
- 资源命名空间隔离
- 独立的锁保护
- 引用计数管理
- 资源树维护

### 4. 资源对象 (RsResource)

**位置**: `src/nvidia/inc/libraries/resserv/rs_resource.h`

所有 GPU 资源的基类，使用面向对象的继承机制。

**资源层次结构**:
```
RsResource (基类)
  ├── RmResource (RM 特定资源)
  │   ├── GpuResource (GPU 相关资源)
  │   │   ├── Memory (内存对象)
  │   │   ├── Channel (通道对象)
  │   │   ├── ContextDma (DMA 上下文)
  │   │   └── ...
  │   ├── Device (设备对象)
  │   ├── Subdevice (子设备)
  │   └── ...
  └── ...
```

**资源操作参数**:
```c
// 分配参数
struct RS_RES_ALLOC_PARAMS_INTERNAL {
    NvHandle hClient;           // 客户端句柄
    NvHandle hParent;           // 父资源句柄
    NvHandle hResource;         // 资源句柄
    NvU32    externalClassId;   // 外部类 ID
    void    *pAllocParams;      // 分配参数
    NvU32    paramsSize;        // 参数大小
    RS_LOCK_INFO *pLockInfo;    // 锁信息
    // ... 其他字段
};

// 释放参数
struct RS_RES_FREE_PARAMS_INTERNAL {
    NvHandle hClient;           // 客户端句柄
    NvHandle hResource;         // 资源句柄
    NvBool   bInvalidateOnly;   // 仅失效不释放句柄
    RS_LOCK_INFO *pLockInfo;    // 锁信息
    // ... 其他字段
};
```

### 5. 锁机制

**多层锁结构**:

1. **API Lock** (`g_RmApiLock`):
   - 保护 RMAPI 调用序列化
   - 读写锁，支持多读单写
   - 用于客户端级别的同步

2. **Client Lock**:
   - 每个客户端独立的读写锁
   - 保护客户端资源树
   - 支持细粒度并发

3. **GPU Lock**:
   - 保护 GPU 硬件状态
   - 用于 DPC 和 ISR 上下文
   - 与 API Lock 配合使用

**锁获取标志**:
```c
#define RMAPI_LOCK_FLAGS_NONE           0x00000000
#define RMAPI_LOCK_FLAGS_COND_ACQUIRE   NVBIT(0)  // 条件获取
#define RMAPI_LOCK_FLAGS_READ           NVBIT(1)  // 读锁
#define RMAPI_LOCK_FLAGS_WRITE          0x00000000
#define RMAPI_LOCK_FLAGS_LOW_PRIORITY   NVBIT(2)  // 低优先级
#define RMAPI_LOCK_FLAGS_READ_FORCE     NVBIT(3)  // 强制读锁
```

## 资源管理流程

### 资源分配流程

1. **客户端调用 RMAPI**:
   ```
   rmapiAlloc() -> serverAllocResource()
   ```

2. **锁获取**:
   ```
   rmapiLockAcquire(WRITE) -> 客户端锁获取
   ```

3. **资源构造**:
   ```
   resConstruct() -> 调用资源类的构造函数
   ```

4. **资源插入**:
   ```
   clientAddResource() -> 插入客户端资源树
   ```

5. **锁释放**:
   ```
   rmapiLockRelease()
   ```

### 资源释放流程

1. **客户端调用 RMAPI**:
   ```
   rmapiFree() -> serverFreeResource()
   ```

2. **锁获取**:
   ```
   rmapiLockAcquire(WRITE)
   ```

3. **资源查找**:
   ```
   clientGetResource() -> 从客户端资源树查找
   ```

4. **资源析构**:
   ```
   resDestruct() -> 调用资源类的析构函数
   ```

5. **资源移除**:
   ```
   clientRemoveResource() -> 从资源树移除
   ```

6. **释放句柄**:
   ```
   clientFreeHandle() -> 释放资源句柄
   ```

### Control Call 流程

Control Call 用于对已分配资源进行控制操作：

1. **RMAPI 入口**:
   ```c
   rmapiControl() -> serverControl()
   ```

2. **参数拷贝**:
   ```
   rmapiParamsCopy() -> 从用户空间拷贝参数
   ```

3. **权限检查**:
   ```
   resAccessCheckRights() -> 检查访问权限
   ```

4. **Control 分发**:
   ```
   resControl() -> 调用资源的 control 方法
   ```

5. **结果拷贝**:
   ```
   rmapiParamsCopy() -> 拷贝结果到用户空间
   ```

## 关键特性

### 1. 资源共享 (RsShared)

允许资源在多个客户端之间共享：

```c
class RsShared : Object {
    NvS32 refCount;      // 引用计数
    MapNode node;        // 映射节点
};
```

**使用场景**:
- 跨进程内存共享
- 多客户端访问同一 GPU 资源

### 2. 会话管理 (RsSession)

用于管理跨客户端句柄空间的对象：

```c
class RsSession : RsShared {
    PORT_RWLOCK *pLock;  // 会话锁
    // 依赖关系管理
};
```

### 3. 访问权限控制

基于位掩码的细粒度访问控制：

```c
typedef struct RS_ACCESS_MASK {
    NvU8 limbs[RS_ACCESS_MAX_LIMBS];
} RS_ACCESS_MASK;
```

**权限类型**:
- 读权限
- 写权限
- 分配权限
- 控制权限

### 4. 资源描述符 (RS_RESOURCE_DESC)

每个资源类都有对应的描述符：

```c
struct RS_RESOURCE_DESC {
    NvU32 externalClassId;      // 外部类 ID
    NvU32 internalClassId;      // 内部类 ID
    NvU32 flags;                // 标志位
    RS_ACCESS_MASK *pRightsRequired; // 所需权限
    // ... 其他字段
};
```

## 主要代码路径

### 核心 RMAPI 代码:
- `src/nvidia/src/kernel/rmapi/rmapi.c` - RMAPI 主实现
- `src/nvidia/src/kernel/rmapi/alloc_free.c` - 资源分配/释放
- `src/nvidia/src/kernel/rmapi/control.c` - Control call 处理
- `src/nvidia/src/kernel/rmapi/client.c` - 客户端管理
- `src/nvidia/src/kernel/rmapi/resource.c` - 资源管理

### Resource Server 库:
- `src/nvidia/src/libraries/resserv/src/rs_server.c` - 服务器实现
- `src/nvidia/src/libraries/resserv/src/rs_client.c` - 客户端实现
- `src/nvidia/src/libraries/resserv/src/rs_resource.c` - 资源基类

### 锁管理:
- `src/nvidia/src/kernel/core/locks.c` - 核心锁实现
- `src/nvidia/src/kernel/core/locks_common.c` - 通用锁功能

### GPU 操作接口:
- `src/nvidia/src/kernel/rmapi/nv_gpu_ops.c` - GPU 操作 API
- `src/nvidia/arch/nvalloc/unix/src/rm-gpu-ops.c` - Unix GPU 操作

## 内存管理集成

RM 通过以下模块管理 GPU 内存：

- `src/nvidia/src/kernel/mem_mgr/mem_mgr.c` - 内存管理器
- `src/nvidia/src/kernel/mem_mgr/mem.c` - 内存对象
- `src/nvidia/src/kernel/mem_mgr/vaspace.c` - 虚拟地址空间
- `src/nvidia/src/kernel/mem_mgr/heap.c` - 堆管理
- `src/nvidia/src/kernel/mem_mgr/mem_desc.c` - 内存描述符

## RPC 机制

对于 vGPU 和 GSP 场景，RM 使用 RPC 机制：

- `src/nvidia/src/kernel/vgpu/rpc.c` - vGPU RPC
- `src/nvidia/src/kernel/gpu/gsp/kernel_gsp.c` - GSP 通信

## 总结

NVIDIA 开源 GPU 内核模块的 Resource Manager 是一个设计精良的资源管理系统，具有以下特点：

1. **分层架构**: 清晰的 Server-Client-Resource 三层结构
2. **细粒度锁**: 支持高并发的多层锁机制
3. **面向对象**: 使用 NVOC 实现的资源继承体系
4. **访问控制**: 完善的权限管理系统
5. **可扩展性**: 通过资源描述符和类继承支持新资源类型
6. **跨平台**: 抽象层支持不同操作系统

该架构为 GPU 硬件资源的安全、高效管理提供了坚实的基础。
