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

## RsResource 实例存储位置

### CPU-RM vs GSP-RM 架构

NVIDIA GPU 驱动采用了分离式架构：

1. **CPU-RM (Kernel RM / Client RM)**:
   - 运行在主机 CPU 的内核模式驱动 (KMD)
   - 负责操作系统交互、用户态通信
   - 管理资源的客户端视图
   - 所有 RsResource 实例**存储在系统内存 (host memory)** 中

2. **GSP-RM (GPU System Processor RM / Physical RM)**:
   - 运行在 GPU 内部的 GSP (GPU System Processor) 固件中
   - 从 Turing 架构开始引入
   - 负责 GPU 硬件直接操作、电源管理、显示控制等
   - GSP-RM 有自己独立的资源实例，存储在 **GPU 固件内存空间**

### 资源实例的内存分配

查看源码可以确认：

```c
// src/nvidia/src/libraries/resserv/src/rs_server.c:324
PORT_MEM_ALLOCATOR *pAllocator = portMemAllocatorCreateNonPaged();

// src/nvidia/src/libraries/resserv/src/rs_server.c:4076
status = objCreateDynamicWithFlags(&pDynamic, ...);
```

RsServer 使用非分页系统内存分配器创建资源对象，这意味着：

- **RsResource 实例存储在 CPU 侧的系统内存 (RAM) 中**
- 不存储在 GPU 固件内部
- 使用内核非分页内存池，确保资源对象始终驻留在内存中

### CPU-RM 和 GSP-RM 的通信

两者通过 RPC (Remote Procedure Call) 机制通信：

```c
// src/nvidia/src/kernel/vgpu/rpc.c
// 定义了 GSP-RM 相关的 RPC 调用
NV_VGPU_MSG_FUNCTION_GSP_RM_CONTROL    // Control 调用
NV_VGPU_MSG_FUNCTION_GSP_RM_ALLOC      // 资源分配
```

**工作流程**：
1. CPU-RM 在系统内存中维护 RsResource 实例
2. 当需要 GPU 硬件操作时，通过 RPC 与 GSP-RM 通信
3. GSP-RM 在 GPU 内部维护自己的资源状态
4. 两者保持状态同步

### 为什么这样设计？

1. **性能优化**: GPU 固件内存有限，不适合存储大量元数据
2. **安全隔离**: CPU-RM 管理策略，GSP-RM 控制硬件，职责分离
3. **向后兼容**: 老旧 GPU 无 GSP，仍可使用相同的 ResServ 架构
4. **调试便利**: CPU 侧资源易于检查和调试

### 代码证据

```c
// src/nvidia/src/kernel/gpu/gsp/kernel_gsp.c
// GSP-RM 是作为固件加载到 GPU 的
kgspInitRm_IMPL(struct OBJGPU *pGpu, struct KernelGsp *pKernelGsp, 
                GSP_FIRMWARE *pGspFw)

// CPU-RM 中的资源分配
// src/nvidia/src/libraries/resserv/src/rs_server.c
serverAllocResource(...) {
    // 在系统内存中分配资源对象
    status = objCreateDynamicWithFlags(&pDynamic, ...);
}
```

## 内存分配流程详解

### 系统内存 (SYSMEM) 分配流程

系统内存分配用于需要 CPU 访问或 CPU-GPU 共享的数据。

#### 1. 入口和初始化

```c
// 用户调用 NvRmAlloc 或类似 API
sysmemConstruct_IMPL() 
  -> memConstruct_IMPL()      // 通用内存对象构造
```

**主要代码路径**: `src/nvidia/src/kernel/mem_mgr/system_mem.c`

#### 2. 资源分配核心流程

```c
sysmemConstruct_IMPL()
  |
  ├─> memUtilsAllocMemDesc()    // 分配内存描述符
  |     └─> memdescCreate()     // 创建 MEMORY_DESCRIPTOR
  |
  ├─> sysmemAllocResources()    // 分配系统内存资源
  |     |
  |     ├─> memUtilsAllocMemDesc()  // 准备内存描述符
  |     |     └─> memdescSetFlag(ADDR_SYSMEM)  // 标记为系统内存
  |     |
  |     ├─> memdescAlloc()      // 实际分配内存
  |     |     └─> osAllocPages()  // OS 层分配物理页面
  |     |           └─> osAllocPagesInternal()  // Unix/Linux 实现
  |     |                 └─> os_alloc_pages()   // 调用内核内存分配
  |     |
  |     └─> 设置内存属性 (contiguity, page size, etc.)
  |
  └─> memConstructCommon()      // 通用构造完成
        └─> 注册到 GPU 映射系统
```

#### 3. OS 层物理页分配

**代码位置**: `src/nvidia/arch/nvalloc/unix/src/os.c`

```c
osAllocPagesInternal(MEMORY_DESCRIPTOR *pMemDesc)
  |
  ├─> 检查连续性要求 (contiguous/non-contiguous)
  ├─> 确定页面大小 (4KB, 64KB, 2MB, etc.)
  |
  └─> os_alloc_pages()        // 内核接口
        |
        ├─ 连续内存: alloc_pages() 或 __get_free_pages()
        └─ 非连续内存: vmalloc() 或逐页分配
```

#### 4. 关键特性

- **页面大小**: 支持 4KB、64KB、2MB、512MB、256GB
- **连续性**: 可以分配物理连续或非连续内存
- **NUMA 支持**: `osAllocPagesNode()` 支持 NUMA 节点指定
- **缓存属性**: 可配置 cached/uncached/write-combined
- **DMA 映射**: 自动设置 IOMMU/SMMU 映射 (如果需要)

### 显存 (VIDMEM) 分配流程

显存分配用于 GPU 密集访问的数据，存储在 GPU 板载 VRAM 中。

#### 1. 入口和初始化

```c
// 用户指定 NVOS32_ATTR_LOCATION_VIDMEM
vidmemConstruct_IMPL()
  -> memConstruct_IMPL()
```

**主要代码路径**: `src/nvidia/src/kernel/mem_mgr/video_mem.c`

#### 2. 资源分配核心流程

```c
vidmemConstruct_IMPL()
  |
  ├─> memmgrAllocResources()      // 内存管理器分配
  |
  ├─> vidmemAllocResources()      // 显存专用分配
  |     |
  |     ├─> _vidmemQueryAlignment() // 查询对齐需求
  |     |     └─> memmgrDeterminePageSize()  // 确定页面大小
  |     |
  |     ├─> 选择分配器:
  |     |     |
  |     |     ├─ PMA (Physical Memory Allocator) - 现代 GPU
  |     |     |   └─> _vidmemPmaAllocate()
  |     |     |         └─> pmaAllocatePages()  // PMA 分配页面
  |     |     |               |
  |     |     |               ├─ 检查 NUMA 配置
  |     |     |               ├─ 设置分配选项 (连续性、对齐等)
  |     |     |               └─ 从 GPU 帧缓冲区分配
  |     |     |
  |     |     └─ Heap (传统方式) - 老旧 GPU
  |     |         └─> heapAlloc()            // 堆分配
  |     |               └─> 从 FB heap 分配内存块
  |     |
  |     └─> memdescSetFlag(ADDR_FBMEM)     // 标记为帧缓冲内存
  |
  ├─> GSP-RM 场景:
  |     └─> NV_RM_RPC_ALLOC_VIDMEM()       // RPC 到 GSP-RM
  |           └─> GSP-RM 在 GPU 端执行实际硬件操作
  |
  └─> memConstructCommon()
```

#### 3. PMA (Physical Memory Allocator)

**现代 GPU 使用 PMA 管理显存**:

```c
pmaAllocatePages(PMA *pPma, ...)
  |
  ├─> 检查可用内存
  ├─> 应用分配策略
  |     ├─ 优先连续分配 (如果请求)
  |     ├─ NUMA 感知分配
  |     └─ 碎片整理优化
  |
  ├─> 从空闲列表分配页面
  ├─> 更新内存统计
  └─> 返回物理帧地址
```

**代码位置**: `src/nvidia/src/kernel/gpu/mem_mgr/phys_mem_allocator/`

#### 4. Heap 分配器 (传统)

**老旧 GPU 使用堆管理**:

```c
heapAlloc(Heap *pHeap, ...)
  |
  ├─> 在堆中查找合适的空闲块
  ├─> 应用对齐要求
  ├─> 分割或合并块
  └─> 标记块为已使用
```

#### 5. 关键特性

- **分配器**: PMA (现代) vs Heap (传统)
- **页面大小**: 支持多种页面大小
- **压缩**: 支持内存压缩 (如果硬件支持)
- **保护内存**: 支持受保护/未受保护内存 (Confidential Computing)
- **持久化**: 支持持久化 VIDMEM (跨重启保留)
- **GSP-RM 集成**: 通过 RPC 与 GSP-RM 同步

### 内存分配对比

| 特性 | 系统内存 (SYSMEM) | 显存 (VIDMEM) |
|------|------------------|---------------|
| **物理位置** | 主机 RAM | GPU 板载 VRAM |
| **分配器** | OS 页分配器 | PMA 或 Heap |
| **主要用途** | CPU 访问、共享数据 | GPU 计算、纹理 |
| **带宽** | PCIe 带宽限制 | 高速 GPU 内存总线 |
| **延迟** | 较高 (PCIe) | 极低 (本地) |
| **容量** | 取决于系统 RAM | GPU VRAM 容量 |
| **代码路径** | `system_mem.c` | `video_mem.c` |
| **地址空间标记** | `ADDR_SYSMEM` | `ADDR_FBMEM` |

### 内存描述符 (MEMORY_DESCRIPTOR)

两种分配都使用 MEMORY_DESCRIPTOR 来跟踪内存:

```c
struct MEMORY_DESCRIPTOR {
    NvU64  Size;              // 内存大小
    NvU64  Alignment;         // 对齐要求
    NvU32  _flags;            // 标志 (连续性、缓存属性等)
    NvU64  _pageSize;         // 页面大小
    NV_ADDRESS_SPACE addressSpace;  // ADDR_SYSMEM 或 ADDR_FBMEM
    Heap  *pHeap;             // 关联的堆 (如果使用堆分配)
    PMA_ALLOC_INFO *pPmaAllocInfo;  // PMA 分配信息
    // ... 更多字段
};
```

**代码位置**: `src/nvidia/src/kernel/gpu/mem_mgr/mem_desc.c`

### 分配参数和属性

**NVOS32 分配参数**:

```c
NV_MEMORY_ALLOCATION_PARAMS {
    NvU32  owner;             // 所有者 (client handle)
    NvU32  type;              // 内存类型
    NvU32  flags;             // 分配标志
    NvU32  attr;              // 属性 (位置、页面大小、连续性等)
    NvU32  attr2;             // 扩展属性
    NvU64  size;              // 请求大小
    NvU64  alignment;         // 对齐要求
    NvU64  offset;            // 返回的偏移量
    NvU64  limit;             // 返回的限制
    // ...
}
```

**关键属性标志**:

- `NVOS32_ATTR_LOCATION_VIDMEM` - 显存分配
- `NVOS32_ATTR_LOCATION_PCI` - 系统内存分配  
- `NVOS32_ATTR_PHYSICALITY_CONTIGUOUS` - 物理连续
- `NVOS32_ATTR_PHYSICALITY_NONCONTIGUOUS` - 物理非连续
- `NVOS32_ATTR_PAGE_SIZE_*` - 页面大小选择

## 总结

NVIDIA 开源 GPU 内核模块的 Resource Manager 是一个设计精良的资源管理系统，具有以下特点：

1. **分层架构**: 清晰的 Server-Client-Resource 三层结构
2. **细粒度锁**: 支持高并发的多层锁机制
3. **面向对象**: 使用 NVOC 实现的资源继承体系
4. **访问控制**: 完善的权限管理系统
5. **可扩展性**: 通过资源描述符和类继承支持新资源类型
6. **跨平台**: 抽象层支持不同操作系统
7. **CPU-GPU 分离**: CPU-RM 管理资源对象 (存储在系统内存)，GSP-RM 控制硬件 (运行在 GPU 固件)
8. **双内存系统**: 支持系统内存和显存的统一管理接口

**关键结论**: 
- RsResource 实例存储在 **KMD (内核模式驱动)** 的系统内存中
- **系统内存分配**通过 OS 页分配器 (`osAllocPages`)
- **显存分配**通过 PMA 或 Heap 从 GPU VRAM 分配
- GSP-RM 是独立的固件程序，通过 RPC 与 CPU-RM 通信

该架构为 GPU 硬件资源的安全、高效管理提供了坚实的基础，同时支持灵活的内存分配策略以满足不同应用场景的需求。
