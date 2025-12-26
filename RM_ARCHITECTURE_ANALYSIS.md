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

## CUDA 计算的显存分配机制

### CUDA 显存分配的执行位置

基于对代码的分析，**CUDA 计算相关的显存申请主要在 KMD (内核模式驱动) 中进行分配和管理**。

#### 分配流程概述

1. **用户态调用**:
   ```
   CUDA Runtime (libcuda.so) 
     └─> NvRmAlloc() / cuMemAlloc()
         └─> ioctl() 系统调用到 KMD
   ```

2. **KMD 处理** (`vidmemConstruct_IMPL`):
   ```c
   vidmemConstruct_IMPL()
     └─> vidmemAllocResources()
         └─> PMA (Physical Memory Allocator)
             └─> pmaAllocatePages()  // 在 KMD 中执行
   ```

3. **GSP-RM 同步** (如果启用):
   ```c
   NV_RM_RPC_ALLOC_VIDMEM(pGpu, ...)
     └─> RPC 调用通知 GSP-RM
         └─> GSP-RM 更新硬件状态
   ```

#### 关键点

- **分配决策**: 在 **KMD** 中进行
- **内存管理**: **PMA (KMD 组件)** 管理显存分配
- **元数据**: 存储在 **KMD 的系统内存**中
- **硬件操作**: GSP-RM (如果启用) 负责硬件配置
- **同步机制**: KMD 通过 RPC 与 GSP-RM 保持状态同步

**结论**: CUDA 显存分配的**核心逻辑和资源管理在 KMD 中完成**，GSP-RM 仅在启用时负责底层硬件控制和配置。

## GPFIFO 工作提交机制详解

### GPFIFO 是什么？

GPFIFO (Graphics Processing FIFO) 是 GPU 命令队列，用于将用户态准备的工作 (pushbuffer) 提交给 GPU 执行。

### 提交机制演进

#### 1. Pre-Volta 架构 (传统方式)

**流程**: 用户态 → KMD → 固件

```
用户态:
  1. 准备 pushbuffer (GPU 命令)
  2. 填充 GPFIFO entry
  3. ioctl(UPDATE_GPPUT) 到 KMD

KMD:
  4. 验证请求
  5. 更新 GPU_PUT 寄存器
  6. 通知 GPU HOST 引擎

GPU HOST:
  7. 从 GPFIFO 读取条目
  8. 执行 pushbuffer 命令
```

**代码位置**: `src/nvidia/src/kernel/gpu/mem_mgr/channel_utils.c:454`

#### 2. Volta+ 架构 (Usermode Submission)

**流程**: 用户态直接 trigger → 固件 (绕过 KMD)

**核心机制 - Doorbell Register**:

```c
// 用户态可以直接写入映射的 doorbell 寄存器
// src/nvidia/src/kernel/rmapi/nv_gpu_ops.c:5597
// "In Volta+, a channel can submit work by 'ringing a doorbell' 
//  on the gpu after updating the GP_PUT."
```

**详细流程**:

```
初始化阶段 (通过 KMD):
  1. 分配 Channel
  2. 创建 GPFIFO
  3. 映射 Usermode Region (doorbell)
  4. 获取 Work Submit Token

运行时提交 (用户态直接操作):
  5. 用户态准备 pushbuffer
  6. 填充 GPFIFO entry
  7. 更新 GP_PUT (用户态写 USERD)
  8. 写 doorbell 寄存器 (携带 token)
     └─> 直接触发 GPU HOST 引擎
  
GPU 侧:
  9. HOST 引擎检测 doorbell 中断
  10. 读取 GPFIFO + pushbuffer
  11. 调度执行
```

**关键代码**:

```c
// Usermode 区域映射
// src/nvidia/src/kernel/rmapi/nv_gpu_ops.c:5653
channel->workSubmissionOffset = 
    (NvU32*)((NvU8*)rmSubDevice->clientRegionMapping + 
             NVC361_NOTIFY_CHANNEL_PENDING);

channel->workSubmissionToken = params.workSubmitToken;

// 用户态直接写 doorbell
// (用户空间库代码，不在内核源码中)
*workSubmissionOffset = workSubmissionToken;
```

**GSP-RM 场景的特殊处理**:

```c
// src/nvidia/src/kernel/gpu/fifo/arch/ampere/kernel_fifo_ga100.c:975
// "Updating the usermode doorbell is different for CPU vs. GSP."

if (!RMCFG_FEATURE_PLATFORM_GSP) {
    // CPU-RM: 直接更新寄存器
    kfifoUpdateUsermodeDoorbell_HAL(...);
} else {
    // GSP-RM: 通过内部机制触发
    kfifoUpdateInternalDoorbellForUsermode_HAL(...);
}
```

### 两种机制对比

| 特性 | Pre-Volta | Volta+ (Usermode) |
|------|-----------|-------------------|
| **提交路径** | 用户态 → KMD → GPU | 用户态 → GPU (直接) |
| **KMD 参与** | 每次都需要 | 仅初始化 |
| **延迟** | 较高 (系统调用) | 极低 (直接写寄存器) |
| **吞吐量** | 受 ioctl 限制 | 极高 |
| **安全性** | KMD 验证 | Token 验证 |
| **适用场景** | 老旧 GPU | Volta/Turing/Ampere/Hopper+ |
| **CUDA 使用** | CUDA < 9.0 | CUDA 9.0+ |

### CUDA 与 GPFIFO

**CUDA 应用的工作提交流程**:

```
CUDA Kernel Launch:
  1. CUDA Runtime 准备kernel参数
  2. 生成 GPU 命令到 pushbuffer
  3. 填充 GPFIFO entry
  4. Volta+: 直接写 doorbell (零系统调用)
     Pre-Volta: ioctl 到 KMD
  5. GPU 从 GPFIFO 读取并执行
```

**性能优势**:

- **Volta+ 的 usermode submission** 消除了内核态切换开销
- 适合高频率的小 kernel 启动场景
- 显著降低 CPU-GPU 同步延迟

### 代码路径总结

**用户态映射**:
- `src/nvidia/src/kernel/rmapi/nv_gpu_ops.c` - UVM/CUDA GPU Ops
- `src/nvidia/src/kernel/gpu/fifo/usermode_api.c` - Usermode API

**GPFIFO 管理**:
- `src/nvidia/src/kernel/gpu/mem_mgr/channel_utils.c` - GPFIFO 填充
- `src/nvidia/src/kernel/gpu/fifo/kernel_channel.c` - Channel 管理

**Doorbell 实现**:
- `src/nvidia/src/kernel/gpu/fifo/arch/volta/kernel_fifo_gv100.c` - Volta
- `src/nvidia/src/kernel/gpu/fifo/arch/ampere/kernel_fifo_ga100.c` - Ampere
- `src/nvidia/src/kernel/gpu/fifo/arch/hopper/kernel_fifo_gh100.c` - Hopper

## CUDA 显存管理的具体算法（HBM/GDDR 上的 Buffer 管理）

### 算法概述

NVIDIA 开源 GPU 内核模块使用 **PMA (Physical Memory Allocator)** 作为 CUDA 显存管理的核心算法，这是一个专为 GPU VRAM (HBM/GDDR) 设计的高性能内存分配器。PMA 采用 **多层位图 (Multi-layer Bitmap)** 机制，**既不是 Buddy System 也不是 Slab**，而是一种针对 GPU 内存特性优化的自定义算法。

### 核心算法：多层位图（Multi-layer Bitmap）

#### 1. 位图结构 (Regmap)

PMA 使用 **8 层独立位图**跟踪每个 64KB 页帧的状态和属性：

```c
// src/nvidia/inc/kernel/gpu/mem_mgr/phys_mem_allocator/regmap.h
typedef struct pma_regmap {
    NvU64 totalFrames;                /* 总帧数 (每帧 64KB) */
    NvU64 mapLength;                  /* 位图长度 */
    NvU64 *map[PMA_BITS_PER_PAGE];    /* 8 层位图数组 */
    NvU64 frameEvictionsInProcess;    /* 正在驱逐的帧数 */
    PMA_STATS *pPmaStats;             /* 统计信息 */
    NvBool bProtected;                /* 是否保护内存 (VPR/CPR) */
} PMA_REGMAP;
```

**8 层位图定义**:
```c
// src/nvidia/inc/kernel/gpu/mem_mgr/phys_mem_allocator/map_defines.h

// 状态位图 (2 层)
#define MAP_IDX_ALLOC_UNPIN 0  // 已分配-未锁定 (可驱逐)
#define MAP_IDX_ALLOC_PIN   1  // 已分配-已锁定 (不可驱逐)

// 属性位图 (6 层)
#define MAP_IDX_EVICTING    2  // 正在驱逐
#define MAP_IDX_SCRUBBING   3  // 正在清零
#define MAP_IDX_PERSISTENT  4  // 持久化内存
#define MAP_IDX_NUMA_REUSE  5  // NUMA 重用
#define MAP_IDX_BLACKLIST   6  // 黑名单页面
#define MAP_IDX_LOCALIZED   7  // 本地化内存 (uGPU)

// 页面状态
#define STATE_FREE      0x00              // 空闲
#define STATE_UNPIN     NVBIT(0)          // 已分配-未锁定
#define STATE_PIN       NVBIT(1)          // 已分配-已锁定
```

#### 2. 分配粒度

```c
#define PMA_GRANULARITY 0x10000  // 64KB (基础分配单位)
#define PMA_PAGE_SHIFT  16       // 64KB = 2^16

// 支持的页面大小
#define _PMA_64KB    (64ULL  * 1024)      // 基础页
#define _PMA_128KB   (128ULL * 1024)      // 2 个基础页
#define _PMA_2MB     (2ULL * 1024 * 1024) // 32 个基础页 (大页优化)
#define _PMA_512MB   (512ULL * 1024 * 1024)
```

### 分配算法详解

#### 1. 连续分配算法 (`pmaRegmapScanContiguous`)

**用于**: 连续大块内存 (纹理、framebuffer、大型 CUDA 数组)

**核心思想**: 使用位操作快速扫描位图，寻找连续的空闲帧

**算法步骤**:
```c
// src/nvidia/src/kernel/gpu/mem_mgr/phys_mem_allocator/regmap.c

1. 计算所需帧数:
   numFrames = actualSize >> PMA_PAGE_SHIFT  // 例: 2MB = 32 帧

2. 对齐处理:
   frameAlignment = alignment >> PMA_PAGE_SHIFT
   alignedAddrBase = NV_ALIGN_UP(addrBase, alignment)

3. 快速位图扫描:
   for (frameNum = frameStart; frameNum <= frameLimit - numFramesLimit; ) {
       // 读取起始和结束帧状态
       startFrameAllocState = pmaRegmapRead(pRegmap, frameNum);
       endFrameAllocState = pmaRegmapRead(pRegmap, frameNum + numFramesLimit);
       
       // 检查是否空闲
       if ((endFrameAllocState & STATE_MASK) != STATE_FREE) {
           frameNum += numFrames;  // 跳过整块
           continue;
       }
       
       if ((startFrameAllocState & STATE_MASK) != STATE_FREE) {
           frameNum += frameAlignment;  // 跳到下一个对齐位置
           continue;
       }
       
       // 使用 _checkOne() 验证中间所有帧都空闲
       if (_checkOne(bits, start, end) == -1) {
           // 找到连续空闲块!
           *freeList = frameNum;
           return NV_OK;
       }
   }
```

**优化技术**:

**A. 位操作加速** (`_checkOne`):
```c
// 快速验证连续空闲块
static NvS64 _checkOne(NvU64 *bits, NvU64 start, NvU64 end) {
    startMapIdx = PAGE_MAPIDX(start);   // start / 64
    startBitIdx = PAGE_BITIDX(start);   // start % 64
    
    // 检查中间的 64 位字
    for (mapIdx = startMapIdx + 1; mapIdx <= (endMapIdx - 1); mapIdx++) {
        if (bits[mapIdx] != 0) {
            // 使用 portUtilCountTrailingZeros64() 找到第一个非零位
            firstSetBit = portUtilCountTrailingZeros64(bits[mapIdx]);
            return (mapIdx << 6) + firstSetBit;  // 返回第一个已分配帧
        }
    }
    return -1;  // 全部空闲
}
```

**B. 最长零序列查找** (`maxZerosGet`):
```c
// 找到 64 位中最长的连续零序列
static NvU32 maxZerosGet(NvU64 bits, NvU32* pStartPos) {
    if (bits == 0) {
        return 64;  // 全零
    }
    
    // 1. 计算前导零
    leadingZeros = portUtilCountLeadingZeros64(bits);
    
    // 2. 扫描内部零序列
    while (currentPos < 64) {
        // 跳过 1
        ones = portUtilCountTrailingZeros64(~remainingBits);
        currentPos += ones;
        remainingBits >>= ones;
        
        // 计数 0
        zeros = portUtilCountTrailingZeros64(remainingBits);
        if (zeros > maxZeros) {
            maxZeros = zeros;
            bestStartPos = currentPos;
        }
        currentPos += zeros;
        remainingBits >>= zeros;
    }
    
    return maxZeros;
}
```

#### 2. 非连续分配算法 (`pmaRegmapScanDiscontiguous`)

**用于**: 小块内存、碎片化场景

**核心思想**: 尽可能分配连续块，但允许碎片

**算法步骤**:
```c
1. 初始化搜索状态:
   NvU64 latestFree[PMA_BITS_PER_PAGE];  // 8 层位图的最新空闲位置
   
2. 遍历所有帧:
   for (frameNum = 0; frameNum < totalFrames && pagesAllocated < numPages; ) {
       // 计算当前位图索引
       mapIdx = frameNum >> 6;  // frameNum / 64
       bitIdx = frameNum & 0x3F;  // frameNum % 64
       
       // 检查所有 8 层位图
       NvU64 combined = 0;
       for (i = 0; i < PMA_BITS_PER_PAGE; i++) {
           combined |= pRegmap->map[i][mapIdx];
       }
       
       // 找到空闲位
       if (!(combined & (1ULL << bitIdx))) {
           freeList[pagesAllocated++] = frameNum;
       }
       
       frameNum++;
   }
```

#### 3. 2MB 大页优化

**触发条件**: 
- 分配大小 >= 2MB
- 对齐要求 >= 2MB
- `PMA_ALLOCATE_CONTIGUOUS` 标志

**优化效果**:
```c
// 统计信息
PMA_STATS {
    NvU64 num2mbPages;           // 总 2MB 页数
    NvU64 numFree2mbPages;       // 空闲 2MB 页数
    
    // 快速查找空闲 2MB 块
    num2mbPages = totalFrames / (_PMA_2MB >> PMA_PAGE_SHIFT);
    // = totalFrames / 32
}
```

**分配策略**:
1. 优先从 2MB 对齐的空闲块分配
2. 更新 `num2mbPages` 统计
3. 减少页表条目 (32 个 64KB → 1 个 2MB)
4. 提高 TLB 命中率

### 驱逐算法 (Eviction)

#### NUMA 驱逐 (`pmaRegMapScanContiguousNumaEviction`)

**场景**: ATS/HMM 系统中内存不足时

**算法**:
```c
1. 扫描可驱逐范围:
   // 只有 ALLOC_UNPIN 状态可驱逐
   for (frameNum = frameStart; frameNum <= frameLimit; ) {
       // 检查起始和结束帧
       if ((endFrameAllocState & STATE_MASK) != STATE_UNPIN) {
           frameNum += numFrames;  // 跳过
           continue;
       }
       
       // 使用 _pmaRegmapScanNumaUnevictable() 验证整块可驱逐
       firstUnevictableFrame = _pmaRegmapScanNumaUnevictable(
           pRegmap, frameNum, frameNum + numFramesLimit);
       
       if (firstUnevictableFrame == -1) {
           // 找到可驱逐块
           *evictStart = addrBase + (frameNum << PMA_PAGE_SHIFT);
           *evictEnd = *evictStart + actualSize - 1;
           return NV_OK;
       }
       
       // 跳到不可驱逐帧之后
       frameNum = alignUpToMod(firstUnevictableFrame + 1, 
                               frameAlignment, frameAlignmentPadding);
   }
```

**驱逐状态转换**:
```
STATE_UNPIN → ATTRIB_EVICTING (设置驱逐位)
            → 回调 UVM 驱逐 (pmaEvictPagesCb_t)
            → STATE_FREE (驱逐完成)
```

### 内存清零 (Scrubbing)

**安全要求**: 防止数据泄漏

**算法**:
```c
// src/nvidia/src/kernel/gpu/mem_mgr/phys_mem_allocator/phys_mem_allocator.c

if (pPma->bScrubOnFree) {
    // 设置 scrubbing 位
    pmaRegmapChangePageStateAttrib(pMap, frameNum, pageSize, 
                                   ATTRIB_SCRUBBING, ATTRIB_SCRUBBING);
    
    // 异步清零 (通过 SEC2 引擎或 CE)
    status = pPma->pScrubObj->scrubSubmitPages(
        pPma->pScrubObj, numPages, pPages, pageSize);
    
    // 清零完成后清除 scrubbing 位
}
```

### 黑名单管理 (Blacklisting)

**用途**: ECC 错误页面管理

**数据结构**:
```c
typedef struct {
    NvU64  physOffset;  // 物理偏移 (64KB 对齐)
    NvBool bIsDynamic;  // 动态黑名单
    NvBool bIsValid;    // 是否仍由 RM 管理
} PMA_BLACKLIST_CHUNK;
```

**算法**:
```c
1. 标记黑名单页:
   pmaRegmapChangePageStateAttrib(pMap, frameNum, pageSize,
                                  ATTRIB_BLACKLIST, ATTRIB_BLACKLIST);

2. 分配时跳过:
   if (frameState & ATTRIB_BLACKLIST) {
       frameNum++;  // 跳过黑名单帧
       continue;
   }
```

### 性能特性对比

| 特性 | PMA Multi-Bitmap | Buddy System | Slab Allocator |
|------|------------------|--------------|----------------|
| **分配粒度** | 64KB (固定) | 可变 (2^n) | 固定小对象 |
| **连续分配** | O(n/64) 位扫描 | O(log n) 分裂 | 不支持 |
| **碎片化处理** | 位图紧凑 | 需要合并 | 内部碎片高 |
| **大块优化** | 2MB 大页 | 高阶块 | 不适用 |
| **驱逐支持** | 原生支持 | 需额外逻辑 | 不支持 |
| **NUMA 感知** | 集成 | 需扩展 | 需扩展 |
| **并发性能** | 分层锁 | 全局锁 | 每 CPU 缓存 |
| **内存开销** | 8 bits/64KB<br>= 0.00012% | 指针开销 | 元数据开销高 |

### 算法复杂度分析

**连续分配**:
- **最优情况**: O(1) - 第一个位置就找到
- **平均情况**: O(n/64) - 位图扫描，每次检查 64 帧
- **最坏情况**: O(n) - 扫描整个内存空间

**非连续分配**:
- **时间复杂度**: O(n) - 线性扫描
- **空间复杂度**: O(1) - 原地操作

**位图查找优化**:
```c
// 使用硬件加速的位操作
portUtilCountTrailingZeros64(bits)  // CLZ 指令 - O(1)
portUtilCountLeadingZeros64(bits)   // CTZ 指令 - O(1)
```

### PMA vs Buddy System vs Slab 的选择理由

**为什么 NVIDIA 选择 PMA 而非 Buddy System?**

1. **固定粒度优势**:
   - GPU 内存访问以 64KB 页为单位（硬件特性）
   - 避免 Buddy System 的分裂/合并开销
   - 位图操作比树操作更快

2. **碎片化控制**:
   - Buddy System 的外部碎片问题严重
   - PMA 的位图紧凑表示，碎片可见性高
   - 支持主动碎片整理（驱逐机制）

3. **驱逐集成**:
   - GPU 内存常需驱逐到系统内存
   - PMA 原生支持 UNPIN/PIN 状态
   - Buddy System 需额外元数据跟踪

4. **NUMA/HMM 支持**:
   - PMA 设计时就考虑 ATS/HMM
   - 与 OS 内存管理器无缝集成
   - Buddy System 是自包含的，难以集成

**为什么不用 Slab?**

1. **分配大小**:
   - Slab 适合小对象（< 4KB）
   - CUDA 显存常是 MB/GB 级别
   - Slab 的内部碎片不可接受

2. **不需要对象池**:
   - Slab 的优势是对象重用
   - GPU 内存是匿名页面，无对象语义
   - 不需要构造/析构函数

### 实际应用示例

#### CUDA 内存分配路径

```
用户态:
cudaMalloc(&ptr, 128 * 1024 * 1024)  // 128MB

↓ ioctl

KMD:
vidmemConstruct_IMPL()
  ↓
vidmemAllocResources()
  ↓
pmaAllocatePages(pPma, 
                 pageCount = 128MB / 64KB = 2048,
                 pageSize = 64KB,
                 flags = PMA_ALLOCATE_CONTIGUOUS)
  ↓
PMA 位图扫描:
  - 扫描 2048 个连续空闲帧
  - 使用 _checkOne() 快速验证
  - 找到位置: 帧 #5000 ~ #7047
  ↓
标记位图:
  - map[MAP_IDX_ALLOC_PIN][78] |= 0xFFFFFFFFF8000000
  - map[MAP_IDX_ALLOC_PIN][79-109] = 0xFFFFFFFFFFFFFFFF
  - map[MAP_IDX_ALLOC_PIN][110] |= 0x000000000000007F
  ↓
更新统计:
  - pPmaStats->numFreeFrames -= 2048
  - pPmaStats->numFree2mbPages -= 64
  ↓
返回物理地址:
  fbOffset = 5000 * 64KB = 320MB
```

### 调试和监控

**PMA 统计信息**:
```c
typedef struct _PMA_STATS {
    NvU64 num2mbPages;               // 总 2MB 页数
    NvU64 numFreeFrames;             // 空闲 64KB 帧数
    NvU64 numFree2mbPages;           // 空闲 2MB 页数
    NvU64 numFreeFramesProtected;    // 保护内存空闲帧
    NvU64 numFreeFramesLocalizable[2]; // 每 uGPU 空闲帧
} PMA_STATS;
```

**位图可视化**:
```c
// 调试输出
void pmaRegmapPrint(PMA_REGMAP *pMap) {
    for (j = 0; j < PMA_BITS_PER_PAGE; j++) {
        NV_PRINTF("*** %d-th MAP ***\n", j);
        for (i = 0; i < pMap->mapLength; i+=4) {
            NV_PRINTF("map[%d]: %llx\n", i, pMap->map[j][i]);
        }
    }
}
```

### 总结

NVIDIA 的 PMA (Physical Memory Allocator) 是一个**自定义的多层位图分配器**，专为 GPU VRAM 的特性优化：

**核心特点**:
1. **8 层位图**: 2 层状态 + 6 层属性，精确跟踪每个 64KB 帧
2. **位操作优化**: 使用硬件 CLZ/CTZ 指令加速扫描
3. **固定粒度**: 64KB 基础单位，2MB 大页优化
4. **驱逐集成**: 原生支持 UNPIN/PIN 和 NUMA 驱逐
5. **低开销**: 位图仅占 0.00012% 内存

**优于传统算法**:
- **vs Buddy System**: 避免分裂/合并，碎片可见，驱逐友好
- **vs Slab**: 支持大块分配，无内部碎片，适合 GPU 场景

PMA 是针对 **HBM/GDDR 高带宽内存**和 **CUDA 计算负载**专门设计的高性能内存管理算法。

## Ada Lovelace 架构显存分配策略详解

### Ada 架构概述

Ada Lovelace (AD10X) 是 NVIDIA 的最新数据中心 GPU 架构（本代码库 v590.48.01），继承了 Ampere/Hopper 的先进内存管理特性，同时引入了架构特定的优化策略。

### Ada 架构显存分配核心策略

#### 1. PMA (Physical Memory Allocator) 主导

Ada 架构**完全使用 PMA** 作为显存分配器，已废弃传统 Heap 分配器。

**核心特点**:
```c
// src/nvidia/src/kernel/gpu/mem_mgr/arch/ada/mem_mgr_ad102.c
// src/nvidia/src/kernel/gpu/mem_mgr/arch/ada/mem_mgr_ad104.c

// Ada 继承 Ampere/Hopper PMA 机制
- 64KB 页面粒度 (PMA_GRANULARITY = 64KB)
- 支持 2MB 大页优化
- NUMA 感知分配 (ATS/HMM 场景)
- 动态碎片整理
- 零拷贝 eviction 支持
```

#### 2. 上下文保留内存策略

**AD102** (高端 SKU - RTX 6000 Ada):
```c
// mem_mgr_ad102.c:34
NvU64 memmgrGetMaxContextSize_AD102(OBJGPU *pGpu, MemoryManager *pMemoryManager)
{
    NvU64 size = memmgrGetMaxContextSize_GA100(pGpu, pMemoryManager);
    
    if (RMCFG_FEATURE_PLATFORM_MODS) {
        size += 64 * 1024 * 1024;  // +64MB for MODS 测试平台
    }
    return size;
}
```

**AD104** (中端 SKU - RTX 4000 Ada):
```c
// mem_mgr_ad104.c:35
NvU64 memmgrGetMaxContextSize_AD104(OBJGPU *pGpu, MemoryManager *pMemoryManager)
{
    NvU64 fbSize = 0;
    NvU64 size = memmgrGetMaxContextSize_GA100(pGpu, pMemoryManager);
    
    // 获取可用 FB 大小
    kmemsysGetUsableFbSize_HAL(pGpu, pKernelMemorySystem, &fbSize);
    const NvU32 fbSizeGB = (NvU32)(NV_ALIGN_UP64(fbSize, 1 << 30) >> 30);
    
    // 小显存 GPU 优化 (< 12GB)
    if (fbSizeGB < 12) {
        size += 10 * 1024 * 1024;  // +10MB 缓冲 (Bug: 4455873)
    }
    
    if (RMCFG_FEATURE_PLATFORM_MODS) {
        size += 64 * 1024 * 1024;
    }
    return size;
}
```

**策略说明**:
- **高端 GPU (AD102)**: 统一保留策略，不考虑显存大小
- **中端 GPU (AD104)**: 根据 FB 大小动态调整保留内存
  - < 12GB: 额外 +10MB 保护缓冲
  - ≥ 12GB: 标准保留
- **MODS 平台**: 所有 SKU 额外 +64MB 用于测试和调试

#### 3. 内存分配流程 (Ada 架构)

**CUDA 显存分配完整路径**:

```
用户态 CUDA Runtime:
  cuMemAlloc(size) / cudaMalloc(size)
    ↓
  ioctl(NV_ESC_RM_ALLOC_MEMORY)
    ↓
  
KMD (内核模式驱动):
  RmAllocMemory()
    ↓
  RMAPI: rmapiAlloc()
    ↓
  vidmemConstruct_IMPL()
    ↓
  vidmemAllocResources()
    ↓
  memdescAlloc()
    ↓
  
PMA 分配器 (Ada 使用):
  pmaAllocatePages(pPma, pageCount, pageSize, allocFlags)
    ↓
  // Ada 特定优化
  - 64KB 对齐检查
  - NUMA 节点选择 (如果启用)
  - 压缩支持检查 (Ada 支持 GMK 压缩)
  - 内存区域选择 (考虑 ECC/Protected 属性)
    ↓
  
物理页面分配:
  - 更新 PMA 位图 (regmap)
  - 标记页面状态: FREE → ALLOC_PIN
  - 记录分配元数据
    ↓
  
GSP-RM 同步 (如果启用):
  NV_RM_RPC_ALLOC_VIDMEM(pGpu, ...)
    ↓
  GSP 固件更新硬件页表和状态
    ↓
  
返回给用户态:
  - 物理地址 (FB 偏移)
  - 虚拟地址 (如果已映射)
  - 内存描述符
```

#### 4. Ada 架构特定优化

**4.1 压缩支持**

Ada 支持 **GMK (Generic Memory Kind) 压缩**:
```c
// src/nvidia/src/kernel/gpu/mem_mgr/arch/turing/mem_mgr_tu102.c
// Ada 继承 Turing+ 压缩机制

- 支持 4KB/64KB 压缩页面
- compressionPageSize: 动态选择
- 压缩率: 2:1 到 8:1 (取决于数据)
- 适用场景: 纹理、framebuffer、CUDA 大数组
```

**4.2 NUMA 感知分配**

在 ATS/HMM 启用的系统中:
```c
// src/nvidia/src/kernel/gpu/mem_mgr/phys_mem_allocator/numa.c

Ada NUMA 策略:
- numaNodeId: GPU 对应的 NUMA 节点
- 优先从本地节点分配
- numaReclaimSkipThreshold: 回收阈值 (默认 90%)
- 支持自动 online (PMA_INIT_NUMA_AUTO_ONLINE)
```

**4.3 ECC 内存管理**

Ada 支持 ECC 保护内存:
```c
分配标志:
- NVOS32_ALLOC_FLAGS_PROTECTED_MEM: ECC 保护
- 自动 scrubbing (初始化清零)
- 错误检测和纠正 (SECDED)
```

#### 5. Ada 显存分配参数和标志

**常用分配标志** (`NVOS32_ALLOC_FLAGS_*`):
```c
- ALIGNMENT_FORCE: 强制对齐 (64KB/2MB)
- PERSISTENT_VIDMEM: 持久化内存 (跨进程)
- FIXED_ADDRESS_ALLOCATE: 固定地址分配
- SKIP_SCRUB: 跳过内存清零 (性能优化)
- PROTECTED_MEM: ECC 保护内存
- TURBO_CIPHER_ENCRYPTED: 加密内存 (安全计算)
- NUMA: NUMA 感知分配
```

**PMA 分配选项** (`PMA_ALLOCATION_OPTIONS`):
```c
struct PMA_ALLOCATION_OPTIONS {
    NvU32 flags;              // PMA_ALLOCATE_CONTIGUOUS 等
    NvU32 numaCpuNodeId;      // NUMA CPU 节点
    NvU64 alignment;          // 对齐要求
    NvU32 resultContiguity;   // 连续性要求
};
```

#### 6. Ada 架构性能优化建议

**6.1 大块分配优化**
```c
// 推荐: 使用 2MB 大页
cudaMalloc(&ptr, 2 * 1024 * 1024);  // 2MB 对齐

优势:
- 减少页表条目
- 提高 TLB 命中率
- PMA 分配效率更高
```

**6.2 批量分配**
```c
// 推荐: 批量分配而非多次小分配
cudaMalloc(&ptr, total_size);  // 一次大块

而非:
for (i = 0; i < N; i++)
    cudaMalloc(&ptrs[i], small_size);  // 多次小块 (碎片化)
```

**6.3 持久化内存复用**
```c
// 对于长生命周期数据，使用持久化分配
NV_MEMORY_ALLOCATION_PARAMS params = {0};
params.flags = NVOS32_ALLOC_FLAGS_PERSISTENT_VIDMEM;

优势:
- 跨进程共享
- 避免频繁分配/释放
```

#### 7. Ada vs Ampere vs Hopper 对比

| 特性 | Ampere (GA10X) | Ada (AD10X) | Hopper (GH10X) |
|------|----------------|-------------|----------------|
| **PMA 粒度** | 64KB | 64KB | 64KB |
| **大页支持** | 2MB | 2MB | 2MB |
| **压缩** | 是 (GMK) | 是 (GMK) | 是 (GMK+) |
| **ECC** | 可选 | 可选 | 标准 (数据中心) |
| **NUMA** | 支持 | 支持 | 增强 |
| **最大 FB** | 48GB | 48GB | 80GB+ |
| **GSP-RM** | 支持 | 支持 | 必需 |
| **Scrubbing** | 软件 | 软件 | 硬件加速 |
| **上下文保留** | 基础 | 分层 (AD102/AD104) | 统一 |

#### 8. Ada 架构调试和监控

**PMA 统计信息**:
```c
// 查询 PMA 状态
NV_STATUS pmaQueryConfigs(PMA *pPma, NvU32 *pConfig);

统计项:
- pmaStats.numFreeFrames: 空闲帧数
- pmaStats.num2mbPages: 2MB 页数
- pmaStats.numAllocations: 分配次数
- evictionInProgress: 是否正在 eviction
```

**调试寄存器键**:
```
NV_REG_STR_RM_ENABLE_PMA=1              # 启用 PMA
NV_REG_STR_RM_ENABLE_PMA_MANAGED_PTABLES # PMA 管理页表
```

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
9. **用户态加速**: Volta+ 支持用户态直接提交工作，绕过内核提升性能
10. **Ada 优化**: 分层内存策略、GMK 压缩、NUMA 感知、ECC 支持

**关键结论**: 
- **RsResource 实例**存储在 **KMD (内核模式驱动)** 的系统内存中
- **系统内存分配**通过 OS 页分配器 (`osAllocPages`)
- **显存分配**通过 PMA 或 Heap 从 GPU VRAM 分配，**核心逻辑在 KMD**
- **CUDA 显存管理**主要在 **KMD** 中进行，GSP-RM 负责硬件控制
- **GPFIFO 提交**:
  - **Pre-Volta**: 用户态 → KMD → GPU (传统路径)
  - **Volta+**: 用户态 → GPU (直接 doorbell，零系统调用)
- **Ada 显存分配**:
  - **AD102**: 统一保留策略 (+64MB MODS)
  - **AD104**: 动态保留 (< 12GB +10MB)
  - **PMA 主导**: 64KB 粒度，2MB 大页，GMK 压缩
  - **NUMA 感知**: 本地节点优先，自动 online
  - **ECC 保护**: 可选 SECDED，自动 scrubbing
- GSP-RM 是独立的固件程序，通过 RPC 与 CPU-RM 通信

**Ada 架构特色**:
Ada Lovelace 通过分层内存保留策略、增强的 PMA 分配器、GMK 压缩支持、NUMA 感知和 ECC 保护，为现代 CUDA 计算和 AI/ML 工作负载提供了高效、可靠的显存管理机制。其设计充分考虑了不同 SKU 的硬件特性，通过动态策略优化内存利用率和性能。
