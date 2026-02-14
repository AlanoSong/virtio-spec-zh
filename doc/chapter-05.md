# 5 设备类型
- 除了队列、配置空间和内置在 virtio 中的特性协商功能之外，[本协议]还定义了若干设备。
- 以下设备标识符用于识别不同类型的 virtio 设备。某些设备标识符是为那些尚未在本标准中定义的设备保留的。
- 确定哪些设备可用及其类型是与总线相关的。

| 设备 ID | Virtio 设备 |
| ---- | ---- |
| 0 | 保留（无效） |
| 1 | 网络设备 |
| 2 | 块设备 |
| 3 | 控制台 |
| 4 | 熵源 |
| 5 | 内存 ballooning（传统） |
| 6 | I/O 内存 |
| 7 | rpmsg |
| 8 | SCSI 主机 |
| 9 | 9P 传输 |
| 10 | mac80211 wlan |
| 11 | rproc 串口 |
| 12 | virtio CAIF |
| 13 | 内存 ballon |
| 16 | GPU 设备 |
| 17 | 定时器/时钟设备 |
| 18 | 输入设备 |
| 19 | Socket 设备 |
| 20 | 加密设备 |
| 21 | 信号分配模块 |
| 22 | pstore 设备 |
| 23 | IOMMU 设备 |
| 24 | 内存设备 |
| 25 | 声卡设备 |
| 26 | 文件系统设备 |
| 27 | PMEM 设备 |
| 28 | RPMB 设备 |
| 29 | mac80211 hwsim 无线仿真设备 |
| 30 | 视频编码设备 |
| 31 | 视频解码设备 |
| 32 | SCMI 设备 |
| 33 | NitroSecureModule |
| 34 | I2C 适配器 |
| 35 | 看门狗 |
| 36 | CAN 设备 |
| 38 | 参数服务器 |
| 39 | 音频控制设备 |
| 40 | 蓝牙设备 |
| 41 | GPIO 设备 |
| 42 | RDMA 设备 |
| 43 | 摄像头设备 |
| 44 | ISM 设备 |
| 45 | SPI 主机 |
- 上述某些设备在本文件中未作具体说明，因为它们被认为尚不成熟或属于特定领域范畴。请注意，其中一些仅由现有的唯一实现方式所规定；它们可能会成为未来规范的一部分，也可能完全被摒弃，或者在本规范之外继续存在。我们不再对此做进一步阐述。

## 5.7 GPU 设备
- virtio-gpu 是一种基于 virtio 的图形适配器。它可以在 2D 和 3D 模式下运行。在 3D 模式下，它会将渲染操作转交给主机 GPU 来处理，因此需要主机配备支持 3D 的 GPU。
- 在 2D 模式下，virtio-gpu 设备支持 ARGB 硬件光标和多扫描输出（也称为头部）。

### 5.7.1 设备 ID
- 16

### 5.7.2 虚拟队列
- **0** controlq - 发送控制指令的队列
- **1** cursorq - 发送光标更新信息的队列
- 这 2 个队列有相同的格式。每个请求和响应，都有一个固定格式的头部，紧跟着一些指令相关的数据字段。单独的光标队列，是用于光标命令（VIRTIO_GPU_CMD_UPDATE_CURSOR 和 VIRTIO_GPU_CMD_MOVE_CURSOR）的“快速通道”，因此，它们不会被控制队列中的耗时命令影响[光标队列和控制队列互不影响，且控制队列由于具备各种渲染和资源指令，比较耗时]。

### 特性位
- VIRTIO_GPU_F_VIRGL (0) - 支持 virgl 3D 模式。
- VIRTIO_GPU_F_EDID (1) - 支持 EDID。
- VIRTIO_GPU_F_RESOURCE_UUID (2) - 支持向其他 virtio 设备传输资源时，指定资源UUID。
- VIRTIO_GPU_F_RESOURCE_BLOB (3) - 支持创建和使用 blob 资源。
- VIRTIO_GPU_F_CONTEXT_INIT (4) - 支持多上下文类型和同步时间线。需要先支持 VIRTIO_GPU_F_VIRGL。

### 5.7.4 设备配置布局
- GPU 设备配置过程中，使用以下结构体和定义：
```c
#define VIRTIO_GPU_EVENT_DISPLAY (1 << 0)

struct virtio_gpu_config {
    le32 events_read;
    le32 events_clear;
    le32 num_scanouts;
    le32 num_capsets;
};
```

#### 5.7.4.1 设备配置字段
- ***events_read*** - 向驱动发送阻塞事件的信号。驱动**禁止**写该字段。
- ***events_clear*** - 清除设备的阻塞事件。将 '1' 写入一个位中，将会清除 *events_read* 中对应的那个位，这类似于写清除的操作方式。
- ***num_scanouts*** - 指定了该设备所能支持的最大扫描次数。最小值为 1，最大值为 16。
- ***num_capsets*** - 指定了该设备所能支持的最大能力集合数量。最小值为 0。

#### 5.7.4.2 事件
- **VIRTIO_GPU_EVENT_DISPLAY** 显示配置已发生更改。驱动**应该**使用 VIRTIO_GPU_CMD_GET_DISPLAY_INFO 命令从设备中获取相关信息。如果 EDID 支持已经协商过（通过 VIRTIO_GPU_F_EDID 特性标志），设备还**应该**使用 VIRTIO_GPU_CMD_GET_EDID 命令来获取更新后的 EDID 数据块。

### 5.7.5 设备要求：设备初始化
- 驱动**应该**使用 VIRTIO_GPU_CMD_GET_DISPLAY_INFO 命令从设备获取显示信息，并且利用这些信息进行初始的扫描输出设置。如果已协商支持 EDID（VIRTIO_GPU_F_EDID 特性标志），则设备还**应该**使用 VIRTIO_GPU_CMD_GET_EDID 命令获取 EDID 信息。如果没有可用信息，或者所有显示器均处于禁用状态，驱动**可以**选择使用备用方案，例如，在显示器 0 上使用 1024x768 的分辨率。
- 驱动**应该**查询所有设备支持的共享内存区域。如果设备支持共享内存，该共享内存区域的 shmid **必须**是以下类型中的一种：
```c
enum virtio_gpu_shm_id {
    VIRTIO_GPU_SHM_ID_UNDEFINED = 0,
    VIRTIO_GPU_SHM_ID_HOST_VISIBLE = 1,
};
```
- 具有 VIRTIO_GPU_SHM_ID_HOST_VISIBLE 标识的共享内存区域，被称为“主机可见内存区域”。若存在这种主机可见内存区域，则设备**必须**支持 VIRTIO_GPU_CMD_RESOURCE_MAP_BLOB 和 VIRTIO_GPU_CMD_RESOURCE_UNMAP_BLOB 指令。

### 5.7.6 设备操作
- virtio-gpu 是基于主机专属资源这一概念而设计的。除非支持共享内存区域，否则，客户机**必须**通过直接内存访问（DMA）将数据传输到这些资源中。这是为了能够与未来的 3D 渲染进行兼容，而设定的设计要求。在未加速的 2D 模式下，没有从资源进行 DMA 传输的支持，只有向资源传输数据的支持。
- 资源最初是简单的 2D 资源，包括宽度、高度、格式以及标识符。然后，客户机必须将后端存储附加到这些资源上，以便 DMA 传输能够正常进行。这类似于真实 GPU 中的 GART。

#### 5.7.6.1 设备操作：创建帧缓冲区和配置扫描输出
- 使用 VIRTIO_GPU_CMD_RESOURCE_CREATE_2D 指令，创建主机资源。
- 从客户机内存中分配一个帧缓冲区，并使用 VIRTIO_GPU_CMD_RESOURCE_ATTACH_BACKING 命令将其作为后端存储，附加到刚刚创建的资源上。支持散列列表，因此帧缓冲区在客户机物理内存中无需连续。
- 使用 VIRTIO_GPU_CMD_SET_SCANOUT 指令，将帧缓冲区与显示扫描输出关联。
- 向帧缓冲区进行渲染。
- 使用 VIRTIO_GPU_CMD_TRANSFER_TO_HOST_2D 指令，将客户机内存中的数据，更新到主机资源中。
- 使用 VIRTIO_GPU_CMD_RESOURCE_FLUSH 指令，将更新过的资源，刷新到显示。

#### 5.7.6.3 设备操作：使用页翻转
- 可以创建多个帧缓冲区，通过使用 VIRTIO_GPU_CMD_SET_SCANOUT 和 VIRTIO_GPU_CMD_RESOURCE_FLUSH 来在它们之间切换，并使用 VIRTIO_GPU_CMD_TRANSFER_TO_HOST_2D 来更新不可见的帧缓冲区。

> 例如：DirectX 框架中，有 Swap Chain 的概念，用户可以创建多个帧缓冲区，正在被显示器调用的叫 Front Buffer，正在被渲染框架调用的叫 Back Buffer，然后 Front/Back Buffer 进行角色交替，这样可以提升渲染效率。这里的概念也是类似。

#### 5.7.6.4 设备操作：多线程设置
- 如果存在 2 个或更多的显示器，那么有多种配置方式可供选择：
- 1. 创建 1 个单一的帧缓冲区，并将其连接到所有显示器（镜像）。
- 2. 为每个显示器创建一个帧缓冲区。
- 3. 创建一个大的帧缓冲区，并配置扫描输出以分别显示该缓冲区的不同区域。

#### 5.7.6.5 设备要求：设备操作：指令生命周期和屏障
- 设备**可以**异步地处理这些 controlq 命令，并在命令处理完成之前将其返回给驱动。如果驱动需要知道处理何时完成，它可以将 VIRTIO_GPU_FLAG_FENCE 标志设置到请求中。[这种情况下，]设备**必须**在返回命令之前完成处理。
- 注意：当前的 QEMU 实现仅在 3D 模式下进行异步处理，即将处理任务卸载到主机 GPU 上。

#### 5.7.6.6 设备操作：配置鼠标光标
- 鼠标光标图像属于常规资源，但其尺寸必须为 64x64。驱动**必须**创建并填充该资源（使用常规的 VIRTIO_GPU_CMD_RESOURCE_CREATE_2D，VIRTIO_GPU_CMD_RESOURCE_ATTACH_BACKING 和 VIRTIO_GPU_CMD_TRANSFER_TO_HOST_2D controlq 指令），并确保其完成（使用 VIRTIO_GPU_FLAG_FENCE 标志）。
- 然后，[驱动]可以向 cursorq 发送 VIRTIO_GPU_CMD_UPDATE_CURSOR 指令，来设置指针的形状和位置。若要不更新形状而移动指针，请使用 VIRTIO_GPU_CMD_MOVE_CURSOR 替代。

#### 5.7.6.7 设备操作：请求头部
- 所有虚拟队列上的请求和响应，都有一个固定的数据头部，使用以下结构体布局和定义：
```c
enum virtio_gpu_ctrl_type {

    /* 2d commands */
    VIRTIO_GPU_CMD_GET_DISPLAY_INFO = 0x0100,
    VIRTIO_GPU_CMD_RESOURCE_CREATE_2D,
    VIRTIO_GPU_CMD_RESOURCE_UNREF,
    VIRTIO_GPU_CMD_SET_SCANOUT,
    VIRTIO_GPU_CMD_RESOURCE_FLUSH,
    VIRTIO_GPU_CMD_TRANSFER_TO_HOST_2D,
    VIRTIO_GPU_CMD_RESOURCE_ATTACH_BACKING,
    VIRTIO_GPU_CMD_RESOURCE_DETACH_BACKING,
    VIRTIO_GPU_CMD_GET_CAPSET_INFO,
    VIRTIO_GPU_CMD_GET_CAPSET,
    VIRTIO_GPU_CMD_GET_EDID,
    VIRTIO_GPU_CMD_RESOURCE_ASSIGN_UUID,
    VIRTIO_GPU_CMD_RESOURCE_CREATE_BLOB,
    VIRTIO_GPU_CMD_SET_SCANOUT_BLOB,

    /* 3d commands */
    VIRTIO_GPU_CMD_CTX_CREATE = 0x0200,
    VIRTIO_GPU_CMD_CTX_DESTROY,
    VIRTIO_GPU_CMD_CTX_ATTACH_RESOURCE,
    VIRTIO_GPU_CMD_CTX_DETACH_RESOURCE,
    VIRTIO_GPU_CMD_RESOURCE_CREATE_3D,
    VIRTIO_GPU_CMD_TRANSFER_TO_HOST_3D,
    VIRTIO_GPU_CMD_TRANSFER_FROM_HOST_3D,
    VIRTIO_GPU_CMD_SUBMIT_3D,
    VIRTIO_GPU_CMD_RESOURCE_MAP_BLOB,
    VIRTIO_GPU_CMD_RESOURCE_UNMAP_BLOB,

    /* cursor commands */
    VIRTIO_GPU_CMD_UPDATE_CURSOR = 0x0300,
    VIRTIO_GPU_CMD_MOVE_CURSOR,

    /* success responses */
    VIRTIO_GPU_RESP_OK_NODATA = 0x1100,
    VIRTIO_GPU_RESP_OK_DISPLAY_INFO,
    VIRTIO_GPU_RESP_OK_CAPSET_INFO,
    VIRTIO_GPU_RESP_OK_CAPSET,
    VIRTIO_GPU_RESP_OK_EDID,
    VIRTIO_GPU_RESP_OK_RESOURCE_UUID,
    VIRTIO_GPU_RESP_OK_MAP_INFO,

    /* error responses */
    VIRTIO_GPU_RESP_ERR_UNSPEC = 0x1200,
    VIRTIO_GPU_RESP_ERR_OUT_OF_MEMORY,
    VIRTIO_GPU_RESP_ERR_INVALID_SCANOUT_ID,
    VIRTIO_GPU_RESP_ERR_INVALID_RESOURCE_ID,
    VIRTIO_GPU_RESP_ERR_INVALID_CONTEXT_ID,
    VIRTIO_GPU_RESP_ERR_INVALID_PARAMETER,
};

#define VIRTIO_GPU_FLAG_FENCE (1 << 0)
#define VIRTIO_GPU_FLAG_INFO_RING_IDX (1 << 1)

struct virtio_gpu_ctrl_hdr {
    le32 type;
    le32 flags;
    le64 fence_id;
    le32 ctx_id;
    u8 ring_idx;
    u8 padding[3];
};
```
- 每条请求中的固定的头部结构体 virtio_gpu_ctrl_hdr 包含以下字段：
- 1. ***types*** - 定义驱动请求（VIRTIO_GPU_CMD_*）的类型，或者设备响应的类型（VIRTIO_GPU_RESP_*）。
- 2. ***flags*** - 请求/响应标志。
- 3. ***fence_id*** - 如果驱动在请求 *flags* 字段中设置了 VIRTIO_GPU_FLAG_FENCE 位，设备**必须**：1. 在响应中设置 VIRTIO_GPU_FLAG_FENCE 位；2. 将请求中的 *fence_id* 字段，拷贝到响应中，并且；3. 只能在指令处理完成后，发送响应。
- 4. ***ctx_id*** - 渲染上下文（只在 3D 模式使用）。
- 5. ***ring_idx*** 如果支持 VIRTIO_GPU_F_CONTEXT_INIT 特性，那么驱动**可以**在请求 *flags* 中设置 VIRTIO_GPU_FLAG_INFO_RING_IDX 位。这种情况下：1. *ring_idx* 表示上下文特定的环形缓冲区索引值。其最小值为 0，最大值为 63（包含）。2. 如果设置了 VIRTIO_GPU_FLAG_FENCE 标志，那么 *fence_id* 在由 *ctx_idx* 和环形缓冲区索引所定义的同步时间线上，充当序列号的作用。3. 如果设置了 VIRTIO_GPU_FLAG_FENCE 标志，并且与 *fence_id* 关联的命令完成时，设备**必须**在相同的同步时间线上，针对序号小于等于 *fence_id* 的所有未完成命令发送响应。
- 如果没有负载情况下，[指令处理]成功时，设备会返回 VIRTIO_GPU_RESP_OK_NODATA。否则 *type* 字段会指示负载的类型。
- [指令处理]失败时，设备会返回错误码 VIRTIO_GPU_RESP_ERR_* 中的某 1 个。

#### 5.7.6.8 设备操作：controlq
- 对于任一坐标，规定 (0,0) 位于左上，x 坐标向右增长，y 坐标向下增长。
- **VIRTIO_GPU_CMD_GET_DISPLAY_INFO** [该指令]获取当前的输出配置。没有请求数据（只是简单的结构体 *virtio_gpu_ctrl_hdr*）。响应类型是 VIRTIO_GPU_RESP_OK_DISPLAY_INFO，响应数据是结构体 *virtio_gpu_resp_display_info*。
```c
#define VIRTIO_GPU_MAX_SCANOUTS 16
struct virtio_gpu_rect {
    le32 x;
    le32 y;
    le32 width;
    le32 height;
};

struct virtio_gpu_resp_display_info {
    struct virtio_gpu_ctrl_hdr hdr;
    struct virtio_gpu_display_one {
    struct virtio_gpu_rect r;
        le32 enabled;
        le32 flags;
    } pmodes[VIRTIO_GPU_MAX_SCANOUTS];
};
```
- 该响应包含一个列表，该列表包含每个扫描输出信息。该信息中包含扫描输出是否使能，和倾向的位置和尺寸是多少。
- 该尺寸（字段 *width* 和 *height*）与 EDID 显示信息中的原生面板分辨率相似，但虚拟机的情况有所不同，即当代表虚拟机内显示的主机窗口被调整大小时，尺寸可能会发生变化。
- 该位置（字段 *x* 和 *y*）描述了显示是如何被组织的。
- 当用户使能过显示器时，*enabled* 字段就被置上。就和物理显示连接器的联通状态一个道理。

- **VIRTIO_GPU_CMD_GET_EDID** [该指令]获取给定的扫描输出的 EDID 数据。请求数据是结构体 virtio_gpu_get_edid。响应类型是 VIRTIO_GPU_RESP_OK_EDID，响应数据是结构体 virtio_gpu_resp_edid。该[特性的]支持是可选的，并且通过使用 VIRTIO_GPU_F_EDID 特性标志来协商。
```c
struct virtio_gpu_get_edid {
    struct virtio_gpu_ctrl_hdr hdr;
    le32 scanout;
    le32 padding;
};

struct virtio_gpu_resp_edid {
    struct virtio_gpu_ctrl_hdr hdr;
    le32 size;
    le32 padding;
    u8 edid[1024];
};
```
- 该响应包含扫描输出的 EDID 显示数据 blob（如 VESA 所描述）。

- **VIRTIO_GPU_CMD_RESOURCE_CREATE_2D** [该指令]创建一个主机上的 2D 资源。请求数据是结构体 *virtio_gpu_resource_create_2d*。响应类型是 VIRTIO_GPU_RESP_OK_NODATA。
```c
enum virtio_gpu_formats {
    VIRTIO_GPU_FORMAT_B8G8R8A8_UNORM = 1,
    VIRTIO_GPU_FORMAT_B8G8R8X8_UNORM = 2,
    VIRTIO_GPU_FORMAT_A8R8G8B8_UNORM = 3,
    VIRTIO_GPU_FORMAT_X8R8G8B8_UNORM = 4,
    VIRTIO_GPU_FORMAT_R8G8B8A8_UNORM = 67,
    VIRTIO_GPU_FORMAT_X8B8G8R8_UNORM = 68,
    VIRTIO_GPU_FORMAT_A8B8G8R8_UNORM = 121,
    VIRTIO_GPU_FORMAT_R8G8B8X8_UNORM = 134,
};

struct virtio_gpu_resource_create_2d {
    struct virtio_gpu_ctrl_hdr hdr;
    le32 resource_id;
    le32 format;
    le32 width;
    le32 height;
};
```
- 该指令在主机上创建一个 2D 资源，指定了宽度，高度和格式。该资源 id 由客户机生成。

- **VIRTIO_GPU_CMD_RESOURCE_UNREF** 销毁一个资源。请求数据是结构体 *virtio_gpu_resource_unref*。响应类型是 VIRTIO_GPU_RESP_OK_NODATA。
```c
struct virtio_gpu_resource_unref {
    struct virtio_gpu_ctrl_hdr hdr;
    le32 resource_id;
    le32 padding;
};
```
- 该指令通知主机：客户机不再需要该资源。

- **VIRTIO_GPU_CMD_SET_SCANOUT** [该指令]为某个[显示]输出设置扫描输出参数。请求数据是结构体 *virtio_gpu_set_scanout*。响应类型是 VIRTIO_GPU_RESP_OK_NODATA。
```c
struct virtio_gpu_set_scanout {
    struct virtio_gpu_ctrl_hdr hdr;
    struct virtio_gpu_rect r;
    le32 scanout_id;
    le32 resource_id;
};
```
- 该指令为某个扫描输出设置扫描输出参数。其中的 *resource_id* 是扫描输出所取资源[的 ID]，伴随着一个矩阵。
- 扫描矩阵必须完全被底层资源所覆盖[矩阵范围不能超过资源尺寸]。允许存在重叠（或完全相同的）扫描区域，典型的应用场景是屏幕镜像。
- 驱动可以使用 resource_id = 0 [这种赋值方式]，去取消一个扫描输出。

- **VIRTIO_GPU_CMD_RESOURCE_FLUSH** [该指令]刷新一个扫描输出资源。请求数据是结构体 *virtio_gpu_resource_flush*。响应类型是 VIRTIO_GPU_RESP_OK_NODATA。
```c
struct virtio_gpu_resource_flush {
    struct virtio_gpu_ctrl_hdr hdr;
    struct virtio_gpu_rect r;
    le32 resource_id;
    le32 padding;
};
```
- 该指令将资源刷新到屏幕。它包含一个矩形区域和一个资源 ID，并清除该资源正在使用的任何扫描输出内容。