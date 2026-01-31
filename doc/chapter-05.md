# 5 设备类型

## 5.7 GPU 设备
- virtio-gpu 是一种基于 virtio 的图形适配器。它可以在 2D 模式和 3D 模式下运行。在 3D 模式下，它会将渲染操作转交给主机 GPU 来处理，因此需要主机配备支持 3D 的 GPU。
- 在 2D 模式下，virtio-gpu 设备支持 ARGB 硬件光标和多扫描输出（也称为头部）。

### 5.7.1 设备 ID
- 16

### 5.7.2 虚拟队列
- 0 controlq - 发送控制指令的队列
- 1 cursorq - 发送光标更新信息的队列
- 这 2 个队列有相同的格式。每个请求和响应都有一个固定格式的头部，紧跟着一些指令相关的数据字段。单独的光标队列是用于光标命令（VIRTIO_GPU_CMD_UPDATE_CURSOR 和 VIRTIO_GPU_CMD_MOVE_CURSOR）的“快速通道”，因此它们不会被控制队列中的耗时命令影响。

### 特性位
- VIRTIO_GPU_F_VIRGL (0) 支持 virgl 3D 模式
- VIRTIO_GPU_F_EDID (1) 支持 EDID
- VIRTIO_GPU_F_RESOURCE_UUID (2) 支持向其他 virtio 设备传输 resource 时，指定 resource UUID
- VIRTIO_GPU_F_RESOURCE_BLOB (3) 支持创建和使用 blob resource
- VIRTIO_GPU_F_CONTEXT_INIT (4) 支持多上下文类型和同步时间线。需要先支持 VIRTIO_GPU_F_VIRGL。

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
- events_read 向驱动发送阻塞事件的信号。驱动***不能***写该字段。
- events_clear 清除设备的阻塞事件。将 '1' 写入一个位中，将会清除 events_read 中对应的那个位，这类似于写清除的操作方式。
- num_scanouts 指定了该设备所能支持的最大扫描次数。最小值为 1，最大值为 16。
- num_capsets 指定了该设备所能支持的最大能力集合数量。最小值为 0。

#### 5.7.4.2 事件
- VIRTIO_GPU_EVENT_DISPLAY 显示配置已发生更改。驱动***应当***使用 VIRTIO_GPU_CMD_GET_DISPLAY_INFO 命令从设备中获取相关信息。如果 EDID 支持已经协商过（通过 VIRTIO_GPU_F_EDID 特性标志），设备还***应该***使用 VIRTIO_GPU_CMD_GET_EDID 命令来获取更新后的 EDID 数据块。

### 5.7.5 设备要求：设备初始化
- 驱动***应该***使用 VIRTIO_GPU_CMD_GET_DISPLAY_INFO 命令从设备获取显示信息，并利用这些信息进行初始的扫描输出设置。如果已协商支持 EDID（VIRTIO_GPU_F_EDID 特性标志），则设备还***应该***使用 VIRTIO_GPU_CMD_GET_EDID 命令获取 EDID 信息。如果没有可用信息，或者所有显示器均处于禁用状态，驱动***可以***选择使用备用方案，例如，在显示器 0 上使用 1024x768 的分辨率。
- 驱动***应该***查询所有设备支持的共享内存区域。如果设备支持共享内存，该共享内存区域的 shmid ***必须***是以下类型中的一种：
```c
enum virtio_gpu_shm_id {
    VIRTIO_GPU_SHM_ID_UNDEFINED = 0,
    VIRTIO_GPU_SHM_ID_HOST_VISIBLE = 1,
};
```
- 具有 VIRTIO_GPU_SHM_ID_HOST_VISIBLE 标识的共享内存区域，被称为“主机可见内存区域”。若存在这种主机可见内存区域，则设备***必须***支持 VIRTIO_GPU_CMD_RESOURCE_MAP_BLOB 和 VIRTIO_GPU_CMD_RESOURCE_UNMAP_BLOB 指令。

### 5.7.6 设备操作
- virtio-gpu 是基于主机专属资源这一概念而设计的。客户机***必须***通过直接内存访问（DMA）将数据传输到这些资源中，除非支持共享内存区域。这是为了能够与未来的 3D 渲染进行兼容，而设定的设计要求。在未加速的 2D 模式下，没有从资源进行 DMA 传输的支持，只有向资源传输数据的支持。
- 资源最初是简单的 2D 资源，包括宽度、高度、格式以及标识符。然后，客户机必须将后端存储附加到这些资源上，以便 DMA 传输能够正常进行。这类似于真实 GPU 中的 GART。

#### 5.7.6.1 设备操作：创建帧缓冲区和配置扫描输出
- 使用 VIRTIO_GPU_CMD_RESOURCE_CREATE_2D 指令，创建主机资源。
- 从客户机内存中分配一个帧缓冲区，并使用 VIRTIO_GPU_CMD_RESOURCE_ATTACH_BACKING 命令将其作为后端存储，附加到刚刚创建的资源上。支持散列列表，因此帧缓冲区在客户机物理内存中无需连续。
- 使用 VIRTIO_GPU_CMD_SET_SCANOUT 指令，将帧缓冲区与显示扫描输出关联。
- 向你的帧缓冲区进行渲染。
- 使用 VIRTIO_GPU_CMD_TRANSFER_TO_HOST_2D 指令，将客户机内存中的数据，更新到主机资源中。
- 使用 VIRTIO_GPU_CMD_RESOURCE_FLUSH 指令，将更新过的资源，刷新到显示。

#### 5.7.6.3 设备操作：使用页翻转
- 可以创建多个帧缓冲区，通过使用 VIRTIO_GPU_CMD_SET_SCANOUT 和 VIRTIO_GPU_CMD_RESOURCE_FLUSH 来在它们之间切换，并使用 VIRTIO_GPU_CMD_TRANSFER_TO_HOST_2D 来更新不可见的帧缓冲区。

#### 5.7.6.4 设备操作：多线程设置
- 如果存在 2 个或更多的显示器，那么有多种配置方式可供选择：
- 1. 创建 1 个单一的帧缓冲区，并将其连接到所有显示器（镜像）。
- 2. 为每个显示器创建一个帧缓冲区。
- 3. 创建一个大的帧缓冲区，并配置扫描输出以分别显示该缓冲区的不同区域。

#### 5.7.6.5 设备要求：设备操作：指令声明周期和 fencing
- 设备***可以***异步地处理这些 controlq 命令，并在命令处理完成之前将其返回给驱动。如果驱动需要知道处理何时完成，它可以将 VIRTIO_GPU_FLAG_FENCE 标志设置到请求中。设备***必须***在返回命令之前完成处理。
- 注意：当前的 QEMU 实现仅在 3D 模式下进行异步处理，即将处理任务卸载到主机 GPU 上。

#### 5.7.6.6 设备操作：配置鼠标光标
- 鼠标光标图像属于常规资源，但其尺寸必须为 64x64。驱动***必须***创建并填充该资源（使用常规的 VIRTIO_GPU_CMD_RESOURCE_CREATE_2D，VIRTIO_GPU_CMD_RESOURCE_ATTACH_BACKING 和 VIRTIO_GPU_CMD_TRANSFER_TO_HOST_2D controlq 指令），并确保其完成（使用 VIRTIO_GPU_FLAG_FENCE 标志）。
- 然后可以向 cursorq 发送 VIRTIO_GPU_CMD_UPDATE_CURSOR 指令，来设置指针的形状和位置。若要不更新形状而移动指针，请使用 VIRTIO_GPU_CMD_MOVE_CURSOR 替代。

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
- 1. types 定义驱动请求（VIRTIO_GPU_CMD_*）的类型，或者设备响应的类型（VIRTIO_GPU_RESP_*）。
- 2. flags 请求/响应标志。
- 3. fence_id 如果驱动在请求标志字段中设置了 VIRTIO_GPU_FLAG_FENCE 位，设备***必须***：
-     3.1 在响应中设置 VIRTIO_GPU_FLAG_FENCE 位，
-     3.2 将请求中的 fence_id 内容，拷贝到响应中，并且
-     3.3 只能在指令处理完成后，发送响应。
- 4. ctx_id 渲染上下文（只在 3D 模式使用）。
- 5. ring_idx 如果支持 VIRTIO_GPU_F_CONTEXT_INIT 特性，那么驱动***可以***在请求标志位中设置 VIRTIO_GPU_FLAG_INFO_RING_IDX 位。这种情况下：
-     5.1 ring_idx 表示上下文特定的环形缓冲区索引值。其最小值为 0，最大值为 63。
-     5.2 如果设置了 VIRTIO_GPU_FLAG_FENCE 标志，那么 fence_id 在由 ctx_idx 和 ring_idx 定义的同步时间线上充当序列号的作用。
-     5.3 如果设置了 VIRTIO_GPU_FLAG_FENCE 标志，并且与 fence_id 关联的命令完成时，设备***必须***在相同的同步时间线上，针对序号小于等于 fence_id 的所有未完成命令发送响应。
- 如果没有负载情况下，[指令处理]成功时，设备会返回 VIRTIO_GPU_RESP_OK_NODATA。否则 type 字段会指示负载的类型。
- [指令处理]失败时，设备会返回错误码 VIRTIO_GPU_RESP_ERR_* 中的某 1 个。

#### 5.7.6.8 设备操作：controlq
- 对于任一坐标，规定 (0,0) 位于左上，x 坐标向右增长，y 坐标向下增长。