# 4 虚拟 I/O 传输选项

- 虚拟 I/O 支持多种不同的总线，因此本规范分为虚拟 I/O 通用部分和总线特定部分。

## 4.1 基于 PCI 总线的虚拟 I/O
- 虚拟 I/O 设备通常用 PCI 设备实现。
- 虚拟 I/O 设备，可以实现成任意类型的 PCI 设备：例如一个通用 PCI 设备，或者一个 PCIe 设备。为了确保满足最新的（PCI 协议）要求，可以参阅 PCI-SIG 官方网站 http://www.pcisig.com。

### 4.1.1 设备要求：PCI 总线上的虚拟 I/O
- 使用 PCI 总线实现的虚拟 I/O 设备，***必须***向客户机提供接口，该接口符合恰当的 PCI 协议规范：PCI 或者 PCIe 均可。

### 4.1.2 PCI 设备识别
- 任何具有 PCI 厂商 ID 0x1AF4 以及 PCI 设备 ID 0x1000 至 0x107F（包括这 2 个 ID）的 PCI 设备都是虚拟 I/O 设备。此范围内的具体 ID，表明该设备支持哪款虚拟 I/O 设备。PCI 设备 ID 是通过将虚拟 I/O 设备 ID 加上 0x1040 得到的，如第 5 节所述。此外，这些设备***可以***使用一个过渡 PCI 设备 ID 范围，即 0x1000 到 0x103F，具体取决于具体设备类型。

#### 4.1.2.1 设备要求：PCI 设备识别
- 设备***必须***具有 PCI 厂商 ID 0x1AF4。设备***必须***：通过将虚拟 I/O 设备 ID 加上 0x1040 来计算出 PCI 设备 ID（如第 5 节所述），或者根据设备类型采用过渡性 PCI 设备 ID（具体如下）：

| 过渡 PCI 设备 ID | 虚拟 I/O 设备 |
| ---- | ----- |
| 0x1000 | 网络设备 |
| 0x1001 | 块设备 |
| 0x1002 | 内存 ballooning（传统）|
| 0x1003 | 控制台 |
| 0x1004 | SCSI 主机 |
| 0x1005 | entropy source |
| 0x1009 | 9P 传输 |

- 例如，虚拟 I/O 设备 ID 为 1 的网络设备，它的 PCI 设备 ID 为 0x1041（0x1040 + 1），或者用过渡 PCI 设备 ID 表示就是 0x1000。
- PCI 子系统厂商 ID 和 PCI 子系统设备 ID，***可以***反应环境中的 PCI 厂商和设备 ID（供驱动识别环境使用）。
- 非过渡设备，***应该***在 0x1040 到 0x107f 之间取用一个 PCI 设备 ID。非过渡设备，***应该***有一个值为 1 或更高的 PCI 修订 ID。非过渡设备，***应该***有一个值为 0x40 或更高的 PCI 子系统设备 ID。
- 这些是为了降低传统驱动试图控制该设备的可能性。

#### 4.1.2.2 驱动要求：PCI 设备识别
- 驱动***必须***与 PCI 厂商 ID 为 0x1AF4，且 PCI 设备 ID 在 0x1040 到 0x107f 范围内的设备相匹配（该范围是通过将虚拟 I/O 设备 ID 加上 0x1040 得到的），如第 5 节所述。对于第 4.1.2 节中列出的设备类型，其驱动***必须***与具有 PCI 厂商 ID 0x1AF4 和第 4.1.2 节中所指示的过渡 PCI 设备 ID 的设备相匹配。
- 驱动***必须***匹配任何 PCI 修订 ID。驱动***可以***匹配任何 PCI 子系统厂商 ID，和任何 PCI 子系统 Device ID。

#### 4.1.2.3 传统接口：关于 PCI 设备识别的说明
- 过渡设备***必须***具有一个值为 0 的 PCI 修改 ID。过渡设备***必须***使 PCI 子系统设备 ID 与虚拟 I/O 设备 ID 相匹配，如第 5 节所述。过渡设备***必须***将过渡 PCI 设备 ID 限定在 0x1000 到 0x103f 范围内。
- 这是为了与传统驱动兼容。

### 4.1.3 PCI 设备布局
- 设备是通过 I/O ***和/或***内存区域进行配置的（不过请参阅 4.1.4.9 中关于通过 PCI 配置空间进行访问的内容），其配置方式由虚拟 I/O 结构的 PCI 属性所规定。
- 设备配置区域中存在不同大小的字段。所有 64 位，32 位和 16 位字段均为小端格式。64 位字段应被视为两个 32 位字段，低 32 位部分在前，高 32 位部分在后。

#### 4.1.3.1 驱动要求：PCI 设备布局
- 对于设备配置访问，驱动***必须***对 8 位宽字段使用 8 位宽的访问方式，对 16 位宽字段使用 16 位宽且对齐的访问方式，对 32 位宽和 64 位宽字段使用 32 位宽且对齐的访问方式。对于 64 位字段，驱动***可以***分别独立访问该字段的高 32 位部分和低 32 位部分。

#### 4.1.3.2 设备要求：PCI 设备布局
- 对于 64 位设备配置字段，设备***必须***允许驱动独立访问高 32 位和低 32 位字段。

### 4.1.4 虚拟 I/O 结构 PCI 功能
- 虚拟 I/O 设备配置布局，包含几个结构体：
- 1. 通用配置
- 2. 通知
- 3. ISR 状态
- 4. 设备特定配置（可选）
- 5. PCI 配置访问
- 每个结构体，都可以通过属于该功能的基地址寄存器（BAR）进行映射，或者通过 PCI 配置空间中特殊的 VIRTIO_PCI_CAP_PCI_CFG 字段进行访问。
- 每个结构体的位置，是通过设备的 PCI 配置空间中的功能列表中，特定于厂商的 PCI 功能来指定的。这个虚拟 I/O 结构体采用小端格式；除非另有说明，否则所有字段对于驱动都是只读的：
```c
struct virtio_pci_cap {
    u8 cap_vndr; /* Generic PCI field: PCI_CAP_ID_VNDR */
    u8 cap_next; /* Generic PCI field: next ptr. */
    u8 cap_len; /* Generic PCI field: capability length */
    u8 cfg_type; /* Identifies the structure. */
    u8 bar; /* Where to find it. */
    u8 id; /* Multiple capabilities of the same type */
    u8 padding[2]; /* Pad to full dword. */
    le32 offset; /* Offset within bar. */
    le32 length; /* Length of the structure, in bytes. */
};
```
- 该结构体后面可以跟额外的数据（取决于 cfg_type 的值），具体看下文。
- 这些字段的解释如下：
- 1. ***cap_vndr*** 0x09；表明厂商特定的功能。
- 2. ***cap_next*** 链接到 PCI 配置空间中，功能链表的下一个功能。
- 3. ***cap_len*** 该功能结构体的长度，包含 virtio_pci_cap 结构体的整个部分，如果有额外数据的话，也包含额外数据的长度。该长度***可以***包含填充字节，或者驱动未使用的字段。
- 4. ***cfg_type*** 表示结构体类型，依据以下表格：
```c
/* Common configuration */
#define VIRTIO_PCI_CAP_COMMON_CFG 1
/* Notifications */
#define VIRTIO_PCI_CAP_NOTIFY_CFG 2
/* ISR Status */
#define VIRTIO_PCI_CAP_ISR_CFG 3
/* Device specific configuration */
#define VIRTIO_PCI_CAP_DEVICE_CFG 4
/* PCI configuration access */
#define VIRTIO_PCI_CAP_PCI_CFG 5
/* Shared memory region */
#define VIRTIO_PCI_CAP_SHARED_MEMORY_CFG 8
/* Vendor-specific data */
#define VIRTIO_PCI_CAP_VENDOR_CFG 9
```
> 任何其他值保留给未来使用。

> 每种结构体后面会分别详细描述。

> 对于任何种类型，设备***可以***提供多个结构体[来描述] - 这使得设备可以暴露多个接口给驱动。能力列表中各项能力的排列顺序，反映了该设备所建议的优先级顺序。设备可以规定，若要使用 id 字段，则应优先采用该机制来执行排序操作。

> ***注意***：例如，在某些虚拟机管理程序中，使用 I/O 访问进行的通知，比使用内存访问进行的通知速度更快。在这种情况下，该设备会以 cfg_type 设置为 VIRTIO_PCI_CAP_NOTIFY_CFG 的方式公开两种功能：第一种功能针对 I/O BAR，第二种功能针对内存 BAR。在这个例子中，如果存在 I/O 资源，驱动程序将使用 I/O BAR；而当 I/O 资源不可用时，则会回退使用内存 BAR。

- 5. ***id*** 某些设备，会使用该字段来唯一标识特定类型的各种功能。若设备类型未明确此字段的含义，则其内容无效。
- 6. ***offset*** 指明了该结构相对于基地址（与 BAR 相关联的基地址）的起始位置。各结构部分中均列出了 offset 的对齐要求。
- 7. ***length*** 指明了结构体的长度。
> length ***可以***包含填充字节，或者驱动未使用的字段，或者未来的拓展字段。
> ***注意***：例如，未来的设备，可能会拥有数兆字节的大型结构数据。而当前的设备从未使用过超过 4K 字节大小的结构，因此驱动可能将映射的结构大小限制为例如 4K 字节（从而忽略结构中超过 4K 字节的部分），以便在不丧失功能且不浪费资源的情况下，与这类设备实现向前兼容性。
- 该类型的变体，结构体 virtio_pci_cap64，正是为了偏移或者长度超过 4GiB 的场景而定义的：
```c
struct virtio_pci_cap64 {
    struct virtio_pci_cap cap;
    u32 offset_hi;
    u32 length_hi;
};
```
- 已知 cap.length 和 cap.offset 字段仅仅只有 32 位，额外的 offset_hi 和 length_hi 字段提供了高 32 位，组合在一起就是 64 位，能够满足 offset 和 length 为 64 为位的场景。

#### 4.1.4.1 驱动要求：虚拟 I/O 结构 PCI 功能
- 驱动***必须***忽略任何具有保留的 cfg_type 值的特定厂商功能结构。
- 驱动***应该***使用它们能够支持的每个虚拟 I/O 结构类型的首个实例。
- 驱动***必须***接受大于此处指定的 cap_len 值的值。
- 驱动***必须***忽略任何具有保留 bar 值的特定厂商功能结构。
- 驱动***应该***仅映射足够大的配置结构部分以供设备操作使用。驱动***必须***处理意外过长的长度，但可以检查该长度是否足够用于设备操作。
- 驱动***禁止***向能功能结构的任何字段写入数据，但 4.1.4.9.2 中详细说明的那些具有 cap_type VIRTIO_PCI_CAP_PCI_CFG 的字段除外。

#### 4.1.4.2 设备要求：虚拟 I/O 结构 PCI 功能
- 设备***必须***将任何额外数据（从 cap_vndr 字段开头，一直到额外数据字段的末尾（如果有））包含在 cap_len 中。设备***可以***为超出此范围的任何结构附加额外数据或填充字节。
- 如果该设备呈现相同类型的多个结构，它***应当***按照最优（首先）到最差（最后）的顺序对它们进行排序。

#### 4.1.4.3 通用配置结构布局
- 可以通过 VIRTIO_PCI_CAP_COMMON_CFG 中的 bar 和 offset 值，来定位通用配置结构体的位置；它的布局如下所示。
```c
struct virtio_pci_common_cfg {
    /* About the whole device. */
    le32 device_feature_select; /* read-write */
    le32 device_feature; /* read-only for driver */
    le32 driver_feature_select; /* read-write */
    le32 driver_feature; /* read-write */
    le16 config_msix_vector; /* read-write */
    le16 num_queues; /* read-only for driver */
    u8 device_status; /* read-write */
    u8 config_generation; /* read-only for driver */
    /* About a specific virtqueue. */
    le16 queue_select; /* read-write */
    le16 queue_size; /* read-write */
    le16 queue_msix_vector; /* read-write */
    le16 queue_enable; /* read-write */
    le16 queue_notify_off; /* read-only for driver */
    le64 queue_desc; /* read-write */
    le64 queue_driver; /* read-write */
    le64 queue_device; /* read-write */
    le16 queue_notif_config_data; /* read-only for driver */
    le16 queue_reset; /* read-write */
    /* About the administration virtqueue. */
    le16 admin_queue_index; /* read-only for driver */
    le16 admin_queue_num; /* read-only for driver */
};
```
- 1. device_feature_select 驱动使用该值，来选择显示 device_feature 的哪些特性位。值 0x0 选择 0 到 31 位的特征位，0x1 选择 32 到 63 位的特征位，依此类推。
- 2. device_feature 设备使用该值，向驱动报告它提供的特征位：驱动通过写入 device_feature_select 来选择要呈现的特征位。
- 3. driver_feature_select 驱动使用该值，来选择显示 driver_feature 的哪些特征位。值 0x0 选择 0 到 31 位的特征位，0x1 选择 32 到 63 位的特征位，依此类推。
- 4. driver_feature 驱动将此值写入，以接受设备提供的特征位。由 driver_feature_select 选择的驱动特征位。
- 5. config_msix_vector 由驱动设置为配置更改通知的 MSI-X 向量。
- 6. num_queues 设备在此处，指定支持的最大虚拟队列数。如果支持任何管理虚拟队列，则将排除这些队列。
- 7. device_status 驱动在此处写入设备状态（请参阅 2.1）。将此字段写入 0 可以重置设备。
- 8. config_generation 配置原子性值。每次配置发生明显变化时，设备都会更改此值。
- 9. queue_select 队列选择。驱动选择下一个字段[queue_size]指的是哪一个虚拟队列。
- 10. queue_size 队列大小。在重置时，该值指定设备支持的最大队列大小。该值可由驱动修改以减少内存需求。值为 0 表示该队列不可用。
- 11. queue_msix_vector 由驱动设置为虚拟队列通知的 MSI-X 向量。
- 12. queue_enable 驱动使用此值来选择性地阻止设备执行来自此虚拟队列的请求。1 - 启用；0 - 禁用。
- 13. queue_notify_off 驱动读取此值，以计算此虚拟队列在通知结构起始位置的偏移量。
> 注意：这不是字节偏移量。请参阅下文 4.1.4.4。
- 14. queue_desc 驱动在此处写入描述符区域的物理地址。请参阅第 2.6 节。
- 15. queue_driver 驱动在此处写入驱动区域的物理地址。请参阅第 2.6 节。
- 16. queue_device 驱动在此处写入设备区域的物理地址。请参阅第 2.6 节。
- 17. queue_notif_config_data 仅当[驱动和设备]协商过 VIRTIO_F_NOTIF_CONFIG_DATA 时才存在。驱动在向设备发送可用缓冲通知时将使用此值。请参阅第 4.1.5.2 节。
> 注意：此字段为设备提供了灵活性，以便其在可用缓冲通知中确定如何引用虚拟队列。在简单的情况下，设备可以将 queue_notif_config_data 设置为虚拟队列索引。某些设备可能会受益于提供其他值，例如，内部虚拟队列标识符，或与虚拟队列索引相关的内部偏移量。
> 注意：此字段之前被称为 queue_notify_data。
- 18. queue_reset 驱动使用此功能来有选择地重置队列。只有在已协商 VIRTIO_F_RING_RESET 时，此字段才会存在。（请参阅 2.6.1）。
- 19. admin_queue_index 设备使用此功能来报告第一个管理虚拟队列的索引。这个字段，只有在 VIRTIO_F_ADMIN_VQ 已达成协商的情况下才有效。
- 20. admin_queue_num 设备利用此值来报告所支持的管理虚拟队列的数量。具有索引在 dmin_queue_index 和 （admin_queue_index + admin_queue_num - 1）之间（包括这两个值）的虚拟队列，将作为管理虚拟队列使用。值为 0 表示没有支持的管理虚拟队列。只有在 IRTIO_F_ADMIN_VQ 已达成协商的情况下，此字段才有效。

##### 4.1.4.3.1 设备要求：通用配置结构布局
- *offset* ***必须***是 4 字节对齐的。
- 设备**必须**具备至少一种通用的配置功能。
- 该设备**必须**在 *device_feature* 中展示其所能提供的功能位，从 *device_feature_select* * 32 的位置开始（对于由驱动编写的任何 *device_feature_select* 值）
- **注意**：这意味着对于除 0 或 1 以外的任何设备功能选择选项，它都将显示为 0，因为此处定义的所有功能的数值都不超过 63。
- 设备**必须**按照以下顺序显示驱动在 *driver_feature* 中所写入的任何有效特性位：从 *driver_feature_select* * 32 开始的位（对于驱动所写的 *driver_feature_select*）。有效的特性位，是那些是对应 *device_feature* 的子集。设备**可以**显示驱动所写入的无效位。
- **注意**：这意味着设备可以忽略那些它从未提供过的功能位的写入操作，并在读取时直接返回 0 。或者，它也可以直接复制驱动所写入的内容（但在驱动程序设置 FEATURES_OK 标志时，它仍需进行检查）。
- **注意**：无论如何，驱动不应该写入无效位，如 3.1.1 所述。
- 在驱动读取到一个设备特定配置值（该值自设备特定配置的任何部分上次被读取以来已发生更改）之后，设备**必须**显示 *config_generation* 有所变化。
- **注意**：由于 config_generation 是一个 8 位值，每次配置更改时，简单地对其进行递增操作可能会因循环覆盖而违反此要求。更好的做法是在其发生变化时设置一个内部标志，如果在驱动从设备特定配置中读取时该标志已设置，则对 config_generation 进行递增操作并清除该标志。
- 当 0 被写入 *device_status* 时，设备**必须**重置，并且一旦重置完成，就在 *device_status* 中写入 0。
- 当重置时，设备必须在 *queue_enable* 中写入 0。
- 如果已经协商过 VIRTIO_F_RING_RESET，设备**必须**在重置时，向 *queue_reset* 中写入 0。
- 如果已经协商过 VIRTIO_F_RING_RESET，在虚拟队列已经用 *queue_enable* 使能之后，设备**必须**向 *queue_reset* 写入 0。
- 当 1 被写入 *queue_reset* 时，设备**必须**重置队列。只要队列重置还在进行，设备**必须**继续向 *queue_reset* 写入 1。当队列重置完成时，设备**必须**同时向 *queue_reset* 和 *queue_enable* 写入 0。
- 如果与当前队列选择对应的虚拟队列不可用，则设备的 *queue_size* 字段**必须**为 0 。
- 如果 VIRTIO_F_RING_PACKED 未被协商，设备**必须**向 *queue_size* 写入 0 或者 2^n。
- 如果已经协商过 VIRTIO_F_ADMIN_VQ，则值 admin_queue_index **必须**等于或大于 num_queues；同时，admin_queue_num **必须**小于或等于 0x10000 - admin_queue_index，以确保有效管理队列的索引能够处于一个 16 位范围内，且该范围要大于所有其他虚拟队列的范围。

##### 4.1.4.3.2 驱动要求：通用配置结构布局
- 驱动**禁止**写入 device_feature、num_queues、config_generation、queue_notify_off 或 queue_notif_config_data。
- 如果已经协商过 VIRTIO_F_RING_PACKED，驱动**禁止**将 queue_size 的值设为 0。如果未协商 VIRTIO_F_RING_PACKED，驱动**禁止**将 queue_size 的值设为非 2^n 的值。
- 驱动**必须**在启用 virtqueue 之前先配置其他 virtqueue 字段，然后使用 queue_enable 来启用它。
- 在将 device_status 的值设为 0 之后，驱动**必须**等待对 device_status 的读取返回 0 之后再重新初始化设备。
- 驱动**禁止**将 0 写入 *queue_enable* 字段。
- 如果已经协商过 VIRTIO_F_RING_RESET，那么在驱动将 1 写入 *queue_reset* 字段以重置队列之后，驱动**必须**在读取回 0 的 *queue_reset* 值之后，才认为队列重置已完成。驱动**可以**在确保其他虚拟队列字段已正确设置后，通过将 1 写入 *queue_enable* 字段来重新启用队列。驱动还**可以**将驱动可写入的队列配置值，设置为与队列重置之前使用的值不同的值（请参阅 2.6.1）。
- 如果已经协商过 VIRTIO_F_ADMIN_VQ，并且驱动配置了任何管理虚拟队列，驱动**必须**使用范围在 admin_queue_index 到 admin_queue_index + admin_queue_num - 1（包括这两个值）内的索引来配置管理虚拟队列。驱动可以配置的管理虚拟队列数量**可以**少于设备支持的数量。

#### 4.1.4.4 通知结构布局
- 通知的位置，是通过 VIRTIO_PCI_CAP_NOTIFY_CFG 字段找到的。该字段后面紧跟着一个附加字段，像这样：
```c
    struct virtio_pci_notify_cap {
    struct virtio_pci_cap cap;
    le32 notify_off_multiplier; /* Multiplier for queue_notify_off. */
};
```
- *notify_off_multiplier* 和 *queue_notify_off* 结合使用，能够得到 BAR 内给一个虚拟队列使用的队列通知地址：
```c
    cap.offset + queue_notify_off * notify_off_multiplier
```
- [上述公式中的] *cap.offset* 和 *notify_off_multiplier* 是从前文的通知功能结构中[virtio_pci_notify_cap]取得的，以及 *queue_notify_off* 是从通用配置结构中获得的。
- **注意**：例如，如果 *notifier_off_multiplier* 值是 0，对于所有队列，设备使用相同的队列通知地址。

##### 4.1.4.4.1 设备要求：通知功能
- 设备**必须**具备至少一种通知功能。
- 对于没有提供 VIRTIO_F_NOTIFICATION_DATA [能力]的设备：
- 1. *cap.offset* 必须是 2 字节对齐。
- 2. 设备**必须**：要么将 *notify_off_multiplier* 设为 2 的偶数次幂；要么将 *notify_off_mutiplier* 设为 0。
- 3. 设备给定的 *cap.length* **必须**至少为 2，并且必须足够大，以能够支持所有可能配置下所有支持的队列的队列通知偏移量。
- 对于所有的队列，设备给定的 *cap.length* 值必须满足：
```c
    cap.length >= queue_notify_off * notify_off_multiplier + 2
```
- 对于提供 VIRTIO_F_NOTIFICATION_DATA [能力]的设备：
- 1. 设备**必须**：要么将 notify_off_multiplier 给定为一个 2 的幂次方，且也是 4 的倍数的数值；要么将其表示为 0。
- 2. *cap.offset* **必须**是 4 字节对齐的。
- 3. 设备给定的 *cap.length* 值**必须**至少为 4，并且**必须**足够大，以支持所有支持的队列在所有可能配置下的队列通知偏移量。
- 对于所有队列，设备给定的 *cap.length* 的值**必须**满足：
```c
    cap.length >= queue_notify_off * notify_off_multiplier + 4
```

#### 4.1.4.5 ISR 状态功能
- VIRTIO_PCI_CAP_ISR_CFG 功能指向至少 1 个单字节，其中包含了为 INT#x 中断处理所使用的 8 位 ISR 状态。
- ISR 状态 中的 *offset* 没有对齐要求。
- ISR 中的 bit 位，允许驱动去区分设备专有配置变动中断，和正常虚拟队列中断：

| 位 | 0 | 1 | 2 - 31 |
| ---- | ---- | ---- | ---- |
| 作用 | 队列中断 | 设备配置中断 | 保留 |
- 为了避免不必要的访问，[驱动]只需要读取这个寄存器，将它重置为 0，这触发设备取消该中断。
- 在这种方法中，驱动通过读取 ISR 状态，来触发设备取消中断。
- 详见章节 4.1.5.3 和 4.1.5.4 来了解具体细节。

##### 4.1.4.5.1 设备要求：ISR 状态功能
- 设备**必须**至少具备一个 VIRTIO_PCI_CAP_ISR_CFG 功能。
- 在向驱动发送设备配置更改通知之前，设备**必须**在 ISR 状态中设置**设备配置中断**位。
- 如果禁用了 MSI-X 功能，在向驱动发送虚拟队列通知之前，设备**必须**在*中断状态*中设置**队列中断**位。
- 如果禁用了 MSI-X 功能，设备**必须**将设备的**PCI 状态寄存器**中的**中断状态**位，设置为设备*中断状态*中所有位的逻辑或值。然后，设备会使能/取消 INT#x 中断，除非根据标准的 PCI 规则进行屏蔽。
- 设备在驱动读取后，必须将中断状态重置为 0。

##### 4.1.4.5.2 驱动要求：ISR 状态功能
- 如果使能了 MSI-X 功能，驱动在探测**队列中断**时，**不应该**访问*中断状态*。

#### 4.1.4.6 设备特定配置
- 对于任何具有设备特定配置的设备类型，设备**必须**至少具备一个 VIRTIO_PCI_CAP_DEVICE_CFG 功能。

##### 4.1.4.6.1 设备要求：设备特定配置
- 设备特定配置相关的*offset*，**必须**是 4 字节对齐。

#### 4.1.4.7 共享内存功能
- 共享内存区域（章节 2.10）在 PCI 传输中，被作为一系列的 VIRTIO_PCI_CAP_SHARED_MEMORY_CFG 功能进行枚举，每个区域对应一个功能。
- 该功能由结构体 virtio_pci_cap64 定义，并利用 cap.id，来允许每个设备拥有多个共享内存区域。cap.id 中的标识符并不表示特定的优先顺序；它只是用于唯一标识一个区域。

##### 4.1.4.7.1 设备要求：共享内存功能
- 由 *cap.offset*、*offset_hi*、*cap.length* 和 *length_hi* 字段组合所定义的区域，**必须**完全包含在由 cap.bar 指定的 BAR 内部。
- *cap.id* 对于任何一个设备实例而言，都**必须**是唯一的。

#### 4.1.4.8 厂商数据功能
- 可选的厂商数据功能，使设备能够向驱动提供厂商特定的数据，且不会产生冲突，用于调试和/或报告目的，并且不会与标准功能发生冲突。
- 此功能是对标准子系统 ID 和子系统厂商 ID 字段（PCI 配置空间头中的偏移量 0x2C 和 0x2E）的补充，而非替代，这些字段由 [PCI] 规定。
- 厂商数据功能在 PCI 传输中被列为 IRTIO_PCI_CAP_VENDOR_CFG 功能。
- 该功能具有以下结构体：
```c
struct virtio_pci_vndr_data {
    u8 cap_vndr; /* Generic PCI field: PCI_CAP_ID_VNDR */
    u8 cap_next; /* Generic PCI field: next ptr. */
    u8 cap_len; /* Generic PCI field: capability length */
    u8 cfg_type; /* Identifies the structure. */
    u16 vendor_id; /* Identifies the vendor-specific format. */
    /* For Vendor Definition */
    /* Pads structure to a multiple of 4 bytes */
    /* Reads must not have side effects */
};
```
- 其中 *vendor_id* 标识 PCI 协议中 PCI-SIG 指定的**厂商 ID**
- 要注意，功能长度要求是 4 的倍数。
- 为了使通用驱动安全地访问该功能，读取该功能**禁止**产生任何副作用。

##### 4.1.4.8.1 设备要求：厂商数据功能
- 设备**可以**提供与 PCI 厂商 ID 或 PCI 子系统厂商 ID 不匹配的 *vendor_id*。
- 设备**可以**提供具有不同或相同 *vendor_id* 值的多个厂商数据功能。
- 厂商 ID 值**禁止**等于 0x1AF4。
- 厂商数据功能的大小，**必须**是 4 字节的倍数。
- 驱动对厂商数据功能的读取操作，**禁止**产生任何副作用。

##### 4.1.4.8.2 驱动要求：厂商数据功能
- 除了调试和上报目的外，驱动**不应该**使用厂商数据功能。
- 在写入厂商数据功能前，驱动**必须**验证 *vendor_id*。

#### 4.1.4.9 PCI 配置访问功能
- VIRTIO_PCI_CAP_PCI_CFG 特性，创建了一种替代性的（并且可能并非最优的）访问方式，用于访问常见的配置、通知、中断服务请求以及设备特定的配置区域。
- 该功能字段，后面紧跟着一个这样的附加字段：
```c
struct virtio_pci_cfg_cap {
    struct virtio_pci_cap cap;
    u8 pci_cfg_data[4]; /* Data for BAR access. */
};
```
- 字段 *cap.bar*，*cap.length*，*cap.offset* 和 *pci_cfg_data* 对于驱动来说是可读可写的。
- 为了访问设备区域，驱动往功能结构像这样写入：
- 1. 驱动通过写入 *cap.bar* 来访问 BAR。
- 2. 驱动通过往 *cap.length* 写入 1，2 或 4，来设置访问的长度。
- 3. 驱动通过写入 *cap.offset* 来设置 BAR 内的偏移。
- 等设置完成，*pci_cfg_data* 会提供一个[数据]窗口，它的长度是 *cap.length*，在给定的 *cap.bar* 的 *cap.offset* 偏移处。

##### 4.1.4.9.1 设备要求：PCI 配置访问功能
- 当检测到驱动对 *pci_cfg_data* 的写访问时，设备**必须**在由 *cap.bar* 选定的 BAR 处执行一个偏移为 *cap.offset* 的写访问操作，并使用 *pci_cfg_data* 中的前 *cap.length* 个字节来进行此操作。
- 当检测到驱动对 *pci_cfg_data* 的读访问时，设备**必须**在由 *cap.bar* 选定的 BAR 处执行一个长度为 *cap.length* 的读访问操作，并将前 *cap.length* 个字节存储到 *pci_cfg_data* 中。

##### 4.1.4.9.2 驱动要求：PCI 配置访问功能
- 驱动**禁止**编写一个非 *cap.length* 的倍数的 *cap.offset*（即所有访问操作都必须对齐[到*cap.length*]）。
- 驱动**禁止**读取或写入 *pci_cfg_data*，除非 *cap.bar*、*cap.length* 和 *cap.offset* 指向由其他某种类型（非 VIRTIO_PCI_CAP_PCI_CFG）的虚拟队列结构定义的 PCI 能力所指定的 BAR 范围内的 *cap.length* 个字节。

#### 4.1.4.10 传统接口：PCI 设备布局注意事项

#### 4.1.4.11 配备传统驱动的非过渡设备：PCI 设备布局注意事项

##### 4.1.4.11.1 设备要求：配备传统驱动的非过渡设备
- 对于在平台上已知存在具有相同 ID（包括 PCI 版本、设备和厂商 ID）的传统设备的旧驱动的情况，非过渡设备**应该**采取以下步骤，以使传统驱动在尝试驱动它们[设备]时能够顺利地报错：
- 1. 在 BAR0 中提供一个 I/O 控制区域，并且
- 2. 对 BAR0 中偏移量为 18 的位置（对应于传统布局中的设备状态寄存器）的单字节零写入进行响应，方法是向每个 BAR 依次呈现零值，并忽略写入操作。

### 4.1.5 PCI 特定初始化和设备操作

#### 4.1.5.1 设备初始化
- 本文档中 PCI 特定的步骤，在设备初始化期间执行。

##### 4.1.5.1.1 虚拟 I/O 设备配置布局检测
- 在设备初始化之前，驱动会扫描 PCI 功能列表，并使用 4.1.4 中所详述的虚拟 I/O 结构 PCI 能力来检测虚拟 I/O 配置布局。

###### 4.1.5.1.1.1 传统接口：设备布局检测注意事项

##### 4.1.5.1.2 MSI-X 向量配置
- 当设备上具备并且使能了 MSI-X 功能（通过标准 PCI 配置空间）时，会使用 *config_msix_vector* 和 *queue_msix_vector* 来将配置更改和队列中断映射到 MSI-X 向量上。在这种情况下，ISR 状态将不会被使用。
- 向 *config_msix_vector/queue_msix_vector* 中写入有效的 MSI-X 表条目编号（0 到 0x7FF），可将由配置更改/选定队列事件触发的中断分别映射到相应的 MSI-X 向量。若要禁用某一事件类型的中断，驱动可通过写入特殊的 NO_VECTOR 值来取消对该事件的映射：
```c
/* Vector value used to disable MSI for queue */
#define VIRTIO_MSI_NO_VECTOR 0xffff
```
- 要注意，映射一个事件到向量，可能需要设备去分配内部的设备资源，并且这可能会失败。

###### 4.1.5.1.2.1 设备要求：MSI-X 向量配置

###### 4.1.5.1.2.2 驱动要求：MSI-X 向量配置

##### 4.1.5.1.3 虚拟队列配置
- 由于一个设备可以有零个或多个用于批量数据传输的虚拟队列，因此驱动需要将其作为设备特定配置的一部分进行设置。
- 驱动通常会按照以下方式，为设备所拥有的每个虚拟队列进行配置：
- 1. 将虚拟队列索引写入 *queue_select*。
- 2. 从 *queue_size* 字段读取虚拟队列的大小。这决定了虚拟队列的大小（请参阅 2.6 节 虚拟队列）。
如果此字段值为 0，则表示不存在该虚拟队列。
- 3. 可选的情况下，选择一个较小的虚拟队列大小，并将其写入 *queue_size* 字段。
- 4. 在连续的物理内存中，为虚拟队列分配并初始化描述符表、可用和已用环形缓冲区。
- 5. 可选地，如果设备上存在并启用了 MSI-X 功能，则选择一个向量来用于请求由虚拟队列事件触发的中断。将与该向量相对应的 MSI-X 表条目编号写入 *queue_msix_vector* 中。读取 *queue_msix_vector*：如果成功，将之前写入的值返回；如果失败，则返回 NO_VECTOR 值。

###### 4.1.5.1.3.1 传统接口：虚拟队列配置注意事项

#### 4.1.5.2 可用缓冲区通知
- 当 VIRTIO_F_NOTIFICATION_DATA 未被协商时，驱动会通过向虚拟队列的 队列通知地址写入仅包含 16 位通知值的方式，向设备发送可用缓冲区通知。
- 通知值依赖于 VIRTIO_F_NOTIF_CONFIG_DATA 的协商结果。
- 如果 VIRTIO_F_NOTIFICATION_DATA 已经协商过，驱动会通过向 队列通知地址写入以下 32 位值来向设备发送可用缓冲区通知：
```c
le32 {
    union {
        vq_index: 16; /* Used if VIRTIO_F_NOTIF_CONFIG_DATA not negotiated */
        vq_notif_config_data: 16; /* Used if VIRTIO_F_NOTIF_CONFIG_DATA negotiated */
    };
    next_off : 15;
    next_wrap : 1;
};
```
- 1. 当 VIRTIO_F_NOTIF_CONFIG_DATA 未被协商，*vq_index* 被设置成虚拟队列索引。
- 2. 当 VIRTIO_F_NOTIF_CONFIG_DATA 被协商过，*vq_notif_config_data* 被设置成 *queue_notif_config_data*。
- 查阅『2.9 驱动通知』获取成员定义。
- 查阅『4.1.4.4』了解如何计算『队列通知』地址。

##### 4.1.5.2.1 驱动要求：可用缓冲区通知
- 如果未协商 VIRTIO_F_NOTIFICATION_DATA，驱动通知**必须**是 16 位的通知。
- 如果协商过 VIRTIO_F_NOTIFICATION_DATA，驱动通知**必须**是 32 位的通知。
- 如果未协商 VIRTIO_F_NOTIF_CONFIG_DATA：
- 1. 如果未协商 VIRTIO_F_NOTIFICATION_DATA，驱动**必须**将通知值写到虚拟队列索引。
- 2. 如果协商过 VIRTIO_F_NOTIFICATION_DATA，驱动**必须**将 *vq_index* 写到虚拟队列索引。
- 如果协商过 VIRTIO_F_NOTIF_CONFIG_DATA ：
- 1. 如果未协商 VIRTIO_F_NOTIFICATION_DATA，驱动**必须**将通知值写到 *queue_notif_config_data*。
- 2. 如果协商过 VIRTIO_F_NOTIFICATION_DATA，驱动**必须**将 *vq_notify_config_data* 写到 *queue_notif_config_data*。

#### 4.1.5.3 已使用缓冲区通知
- 如果虚拟队列需要使用，已使用缓冲区通知，则设备通常会按照以下方式进行操作：
- 如果未启用 MSI-X 功能：
- 1. 为该设备设置 ISR 状态字段的最低位。
- 2. 为该设备发送相应的 PCI 中断信号。
- 如果启用了 MSI-X 功能：
- 1. 如果 *queue_msix_vector* 不等于 NO_VECTOR，则为该设备请求相应的 MSI-X 中断消息，此时 *queue_msix_vector* 会设置 MSI-X 表的条目编号。

##### 4.1.5.3.1 设备要求：已使用缓冲区通知
- 如果 MSI-X 功能被使能，并且 *queue_msix_vector* 不是虚拟队列的 NO_VECTOR，设备**禁止**传递该虚拟队列的中断。

#### 4.1.5.4 设备配置更改通知
- 某些虚拟 I/O PCI 设备可以更改设备配置状态，作为设备它特定配置区域[变更]的反馈。这种情况下：
- 如果 MSI-X 功能未开启：
- 1. 设置设备的 ISR 状态字段的第 2 低的比特位。
- 2. 发送对应的 PCI 中断。
- 如果 MSI-X 功能开启：
- 1. 如果 *config_msix_vector* 不等于 NO_VECTOR，为设备请求对应的 MSI-X *中断信息，config_msix_vector* 设置成 MSI-X 表条目编号。
- 一个单一的中断**可以**指示一个或者多个虚拟队列已经被使用，并且配置空间已经被修改。

##### 4.1.5.4.1 设备要求：设备配置更改通知
- 如果 MSI-X 功能开启，并且 *config_msix_vector* 不等于 NO_VECTOR，设备**禁止**传递设备配置空间更改相关的中断。

##### 4.1.5.4.2 驱动要求：设备配置更改通知
- 驱动必须处理以下情况：同一个中断被用来指示设备配置空间变更，和一个或者多个虚拟队列被使用。

#### 4.1.5.5 驱动如何处理中断
- 驱动中断处理程序通常会：
- 如果 MSI-X 功能被禁用：
– 1. 读取 ISR 状态字段，该字段会将其重置为零。
– 2. 如果最低位被设置：遍历所有虚拟队列以查找该设备，以查看该设备是否已完成任何需要处理的操作。
– 3. 如果第 2 个最低位被设置：重新检查配置空间以查看 *config_msix_vector* 有何变化。
- 如果 MSI-X 功能启用：
– 1. 遍历与该 MSI-X 向量映射到的虚拟队列对应的设备，以查看该设备是否已完成任何需要处理的操作。
– 2. 如果 MSI-X 向量等于 *config_msix_vector*，则重新检查配置空间以查看有何变化。