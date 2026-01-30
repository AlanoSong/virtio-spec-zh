# 1 介绍

## 1.1 规范性引用
- *省略*

## 1.2 非规范引用
- *省略*

## 1.3 术语
- 本文件中的关键词“必须”（MUST）、“禁止”（MUST NOT）、“必需”（REQUIRED）、“应”（SHALL）、“禁止”（SHALL NOT）、“应该”（SHOULD）、“不应该”（SHOULD NOT）、“建议”（RECOMMENDED）、“不建议”（NOT RECOMMENDED）、“可以”（MAY）以及“可选”（OPTIONAL），只有在以全大写字母形式出现时，才应按照 [RFC2119] 和 [RFC8174] 中所述的含义来理解。

### 1.3.1 旧版接口：术语
- 在本规范 1.0 版本之前的规范草案（例如，参见[虚拟机 PCI 草案]）定义了驱动与设备之间类似但不同的接口。由于这些接口已被广泛采用，因此本规范包含了***可选***功能，以简化从这些旧版草案接口的过渡。
- 具体而言，设备和驱动***可以***支持：
- ***旧版接口*** 是由本规范早期草案（在 1.0 版本之前）指定的接口
- ***旧版设备*** 是在本规范发布之前实现的，并在主机侧实现的旧版接口
- ***旧版驱动*** 是在本规范发布之前实现的，并在客户机侧实现的旧接口
- 旧版设备和驱动不符合本规范的要求。
- 为了简化从这些早期草案接口的过渡，设备***可以***实现以下功能：
- ***过渡设备*** 一种既能支持符合本规范的驱动，又能兼容旧版驱动的设备。
- 同样，驱动***可以***实现以下功能：
- ***过渡驱动*** 一种既能支持符合本规范的设备，又能兼容旧版设备的驱动。
- ***注意***：并非必须要提供旧版接口；也就是说，除非有向后兼容性的需求，否则不需要实现它们！
- 没有旧版接口兼容性的设备或驱动，分别被称为非过渡设备和非过渡驱动。

### 1.3.2 从早期规范草案过渡
- 对于已经采用旧版接口的设备和驱动而言，需要对它们进行一些修改以支持本规范。
- 在这种情况下，读者或许会发现，重点关注章节标题中带有“旧版口”标签的部分会更有帮助。这些部分突出了自早期草案以来所做出的改动。

## 1.4 结构体规范
- 许多设备和驱动的结构体内存布局，使用 C 语言的结构体语法。所有的结构体都假定没有额外的 padding [为了内存对齐而额外添加的字节，能够提高内存访问效率]。开发者可以使用 GNU C __attribute__((packed)) 语法，告知 C 语言编译器在结构体内部插入一些额外的 padding。
- 对于结构体中使用的一些实数数据类型，有以下约定：
- 1. ***u8, u16, u32, u64*** 无符号实数，规定了位长度。
- 2. ***le16, le32, le64*** 无符号实数，规定了位长度，采用小端字节序 [小端就是低位字节，存储在低地址，例如对于 0x1234，内存从低位开始，先存 0x34，再存 0x12]
- 3. ***be16, be32, be64*** 无符号实数，规定了位长度，采用大段字节序 [大端与上文小端相反]
- 本规范中定义的一些域，并不是从字节边界开始或者结束的。这样的域称为 bit-fields。一个 bit-fields 通常是对实数类型的拆分 [byte 中的 bit 位，可以单独使用]。
- 实数中的 bit-fields 通常是按照顺序列出来的，从最低有效位到最高有效位。bit-fields 被视为具有指定宽度的无符号整数，且各位之间的有效性顺序保持不变。
- 例如：
```c
struct S {
    be16 {
        A : 15;
        B : 1;
    } x;
    be16 y;
};
```
- A 存储在 x 的低 15 bit 中，而 B 存储在 x 的最高 1 bit 中，16 bit 的实数 x 使用大端字节序存储，并且位于结构体 S 的头部，随后存储的是一个无符号实数 y，它使用大端字节序存储，位于结构体 S 的 2 字节（16 bits）偏移处。
- 要注意，这种表示方式在某种程度上类似于 C 语言中的位域语法，但在编写可移植代码时不应盲目地将其转换为位域表示法：它符合 C 编译器在小端架构下对位域的打包方式，但不符合 C 编译器在大端架构下对位域的打包方式。
- 假定以下宏定义 CPU_TO_BE16 将 16 bit 实数从 CPU 原生的字节序转换成大端字节序，下面是生成要存储到 x 的值的等效可移植 C 代码：
```c
CPU_TO_BE16(B << 15 | A)
```

## 1.5 常量规范
- 在很多情况下，设备与驱动之间的接口中，所使用的数值是通过 C 语言的 #define 和 /* */ 注释语法来记录的。多个相关数值会以一个共同的前缀作为开头，并使用“_”作为分隔符进行组合。以“_XXX”作为后缀则表示一组中的所有数值。例如：
```c
/* Field Fld value A description */
#define VIRTIO_FLD_A (1 << 0)
/* Field Fld value B description */
#define VIRTIO_FLD_B (1 << 1)
```
- 为字段 Fld 记录了 2 个数值，其中 Fld 的值为 1 代表 A，值为 2 代表 B。请注意，<< 表示左移操作。
- 此外，在这种情况下，VIRTIO_FLD_A 和 VIRTIO_FLD_B 分别代表 Fld 的值 1 和 2。此外，VIRTIO_FLD_XXX 指的是 VIRTIO_FLD_A 或者 VIRTIO_FLD_B 中的一个。

# 2 virtio 设备的基本设施
- virtio 设备是通过总线特定的方法被发现并识别的（请参阅总线章节：4.1 virtio 在 PCI 总线上的实现、4.2 virtio 在 MMIO 上的实现以及 4.3 Virtio 在通道 I/O 上的实现）。每个设备都包含以下部分：
- 1. 设备状态字段
- 2. 特性位
- 3. 通知
- 4. 设备配置空间
- 5. 1 个或多个 virtqueue

## 2.1 设备状态字段
- 在设备被驱动初始化期间，驱动遵循 3.1 章节中规定的步骤。
- `设备状态字段`能以一种简单直观的方式，对这一序列中，已完成的步骤进行标示。最直观的理解方式，是将其想象成与控制台上的信号灯相连，从而能显示每个设备的状态。定义了以下标志位（按照它们通常出现的顺序列出）：
- 1. `ACKNOWLEDGE` (1) 表示客户 OS 已找到该设备，并将其识别为有效的 virtio 设备。
- 2. `DRIVER` (2) 表示客机 OS 知晓如何驱动该设备。注意：设置此位可能会有较长（或无限）的延迟时间。例如，在 Linux 系统下，驱动可以是可加载模块。
- 3. `FAILED` (128) 表示在客户机中出现了错误，并且它已放弃对该设备的使用。这可能是内部错误，或者驱动由于某种原因不接受该设备，甚至在设备运行过程中出现致命错误。
- 4. `FEATURES_OK` (8) 表示驱动已确认其能够理解的所有功能，并且功能协商已完成。
- 5. `DRIVER_OK` (4) 表示驱动已设置好并准备好驱动该设备。
- 6. `DEVICE_NEEDS_RESET` (64) 表示设备已出现错误且无法自行恢复。
- `设备状态字段`初始值为 0，在设备重置时会重新初始化为 0。

### 2.7.9 按序消耗descriptor

- 一些device通常按照descriptor生成的顺序消耗他们。这些device能够声明VIRTIO_F_IN_ORDER特性。如果该特性（alano：在driver和device之间）协商成功，那么device在通知driver批量buffer的使用情况时，只用填写一个single used ring entry，并把descriptor链表的head entry对应的id填在其中。
- 然后device根据批量buffer的长度直接跳过。相应的，device也会根据批量buffer的大小，更改used idx的大小。
- driver需要查看used id，并且计算出batch buffer的大小，以便能够抵达下一个device将要填写的used ring位置。

### 2.7.11 操作 Virtqueues 的辅助工具

- Linux内核源码中，在 include/uapi/linux/virtio_ring.h 中包含了上述定义，并且提供了更易于使用的辅助例程。这部分代码由 IBM 和 Red Hat 在 BSD 许可证下授权，因此可以在其他项目中自由使用，并且这些代码在 virtio_queue.h 中也重新实现了一份。

### 2.7.12 Virtqueue 如何操作

- virtqueue的操作分为两块：（driver）为 device提供新的 available buffer，和 device 处理 used buffer

- 注意：举例来说，最简单的 virtio 网络设备包含了2个 virtqueue：发送端 virtqueue 和接收端 virtqueue。driver 向发送端 virtqueue 发送数据包，这些数据 device 可读，并且在这些数据包用完后清理掉。同样的，接收到的数据包会被添加到接收端的 vritqueue 中，并且等待处理。

- 接下来详细描述，这2块操作如何使用 split virtqueue。

### 2.7.13 （driver）向 device 提供 buffer

- driver按照以下步骤，向 device 的 virtqueue 提供 buffer：
- 1. driver 将数据 buffer 放置在 descriptor table 的空闲描述符中，并按照需要的方式组织成一个链表（详见 2.7.5 节，virtqueue descriptor table） 。
- 2. driver 找到 available ring 中的下一个 ring entry，并将描述符链表的头部索引填写进去。
- 3. 如果要填写的是批量的 buffer，上述步骤 1 和 2 可能需要重复操作。
- 4. driver 执行适当的内存屏障（Alano：常见的内存保护手段，保证 driver 和 device 端的读写一致），来保证 device 在下一步操作之前，能看到更新过的 descriptor table 和 available ring。
- 5. available idx 会随着添加到 available ring 中的描述符链表头的数量增加而增加。
- 6. driver 执行适当的内存屏障，来确保它自己能够在检查 notification suppression 之前，成功更新 idx 域。
- 7. 如果（driver 和 device之间）未禁用 available buffer notification，那么 driver 会向 device 发送一个这样的 notification 。
- 需要注意的是：上述步骤没有对 available ring buffer 的溢出覆盖异常采取预防手段：这种情况应该不会发生，因为 ring buffer 的大小和 descriptor table 的大小一直，所以上述步骤（1）就能够防止这种异常发生。
- 另外：最大的 queue size 是 32768（16 bit 数据里面，最大的 2 的 n 次幂）（Alano：2^15）。
- 接下来详细说说，上述的每一个步骤有什么要求。

#### 2.7.13.1 将 buffer 添加到 descriptor table 中

#### 2.7.13.2 更新 available ring

#### 2.7.13.3 更新 idx

##### 2.7.13.3.1 更新 idx 对 driver 有什么要求

#### 2.7.13.4 （driver）通知 device

##### 2.7.13.4.1 （driver）通知 device，对 driver 有什么要求

### 2.7.14 （driver）从 device 接收 used buffer

- 一旦 device 处理完 descriptor 描述的 buffer 数据后，device 会按照章节 2.7.7 中描述的方式，发送一个 used buffer notification。
- 注意：出于性能考虑，处理 used buffer ring时，driver *也许* 会禁用 used buffer notification。但同时我们也要注意到，在清空 ring 和重新使能 notification 的间隙，可能会出现 norification 丢失的问题。这个问题可以通过，在重新启用 notification 后，重新检查 used buffer 的方式来解决：

```c
virtq_disable_used_buffer_notifications(vq);
for (;;) {
    if (vq->last_seen_used != le16_to_cpu(virtq->used.idx)) {
        virtq_enable_used_buffer_notifications(vq);
        mb();

        if (vq->last_seen_used != le16_to_cpu(virtq->used.idx))
            break;

        virtq_disable_used_buffer_notifications(vq);
    }

    struct virtq_used_elem *e = virtq.used->ring[vq->last_seen_used%vsz];
    process_buffer(e);
    vq->last_seen_used++;
}
```

## 2.8 packed virtqueue

- packed virtqueue 是一个可选的布局紧凑型 virtqueue，并且使用可读可写内存（即内存可以同时被 host 和 guest 读和写）。
- 要想使用 packed virtqueue 功能，需要（Alano：driver 和 device）协商 VIRTIO_F_RING_PACKED 特性标志位。
- 每个 packed virtqueue 支持高达 2^15 个 entires。
- 根据当前（Alano：协议规定的）传输方式，virtqueue 被放置在 driver 分配的 guest memory 中。每个 packed virtqueue 包含 3 部分：
- 1. descriptor ring - 位于 descriptor area 中
- 2. driver event suppression - 位于 driver area 中
- 3. device event suppression - 位于 device area 中

- descriptor ring 包含多个描述符，并且每个描述符可能包含以下部分：
- 1. Buffer ID
- 2. Element Address
- 3. Element Length
- 4. Flags

- buffer 包含 0 个或者更多的 device 可读的，物理连续（Alano：就是物理地址连续）的元素，并且这些元素后面紧跟着 0 个或者更多的物理连续的 device 可写的元素。（每个 buffer 至少包含一个元素）
- 当 driver 想把这样一个 buffer 发送给 device 的时候，driver 会向 descriptor ring 中，写入至少 1 个 available descriptor，这个 descriptor 会填写该 buffer 的描述元素。这个 descriptor 会通过自己保存 Buffer ID 的方式，将自己和对应的 buffer 绑定在一起。
- 然后 driver 会通知 device。当处理完这个 buffer 之后，device 会向 descriptor ring 中写入 1 个 used device descriptor，并把 Buffer ID 填进这个 descriptor 中，然后再发送一个 used event notification。
- descriptor ring 通过回环的方式工作：driver 按次序写入 descriptor。在写到 ring 的尾部之后，下一个 descriptor 又被写到 ring 的头部。一旦 ring 被写满了，driver 就停止发送新的请求，然后等待 device 处理 descriptor，并把 descriptor 标记成 used 状态，driver 就又能获得新的 descriptor 使用。
- 同样的，device 按顺序从 ring 中读取 descriptor，同时侦测 descriptor 是否被标记成 available 状态。等处理完 descriptor 之后，device 又会将 used descriptor 写回到 ring 中。
- 注意：在 device 读取到 driver 提供的 descriptor时，device 可能不会按原来的顺序处理这些 descriptor。当 device 写 used descriptor 时，它会严格按照自己完成的顺序去写。

### 2.8.1 driver & device ring 计数器
- driver 和 device 都会在内部维护一个 single-bit 计数器，该计数器初始值会被设置成 1。
- 其中 driver 维护的计数器叫做 driver ring wrap counter。每次 driver 将 ring 中的 descriptor 标记为 available 状态时，都会去更新这个计数器的值。
- device 维护的计数器叫做 device ring wrap counter。每次 device 用完 ring 中的 descriptor 时，都会去更新这个计数器的值。
- 当 driver 和 device 在处理同一个 descriptor 时，或者所有的 available descriptor 都被用完时，我们能看到 driver 和 device 各自的 ring wrap counter 能够匹配上。
- 如果想标记 descriptor 为 available 或者 used 状态，driver 和 device 同时需要使用以下标志位：
```c
#define VIRTQ_DESC_F_AVAIL (1 << 7)
#define VIRTQ_DESC_F_USED (1 << 15)
```
- 要把 descriptor 标记为 available 状态，driver 需要根据内部维护的 driver ring wrap counter，设置 Flags 中的 VIRTQ_DESC_F_AVAIL 位。
- 要把 descriptor 标记为 used 状态，device 需要根据内部的 device ring wrap counter，设置 Flags 中的 VIRTQ_DESC_F_USED 位。
- 因此 VIRTQ_DESC_F_AVAIL 和 VIRTQ_DESC_F_USED 位，对于 available descriptor 来说值是不同的，对于 used descriptor 来说值是相同的。
- 要注意，以上情形通常用于合理性检查，因为这些条件是必要的，但是不是充分条件 - 例如，所有的 descriptor 初始化时没有清零。为了侦测 used 和 available descriptor，driver 和 device 持续跟踪最后一个已发现的 VIRTQ_DESC_F_USED / VIRTQ_DESC_F_AVAIL 的值，是很有必要的。也可以用其他的方法去探测 VIRTQ_DESC_F_USED / VIRTQ_DESC_F_AVAIL 位。

### 2.8.2 轮询 available 和 used descriptor

# 3 总体初始化和设备操作
- 我们首先对设备初始化进行概述，然后详细阐述设备的细节，以及每个步骤如何执行。这一部分最好与总线相关的章节一起阅读，后者会介绍如何与该特定设备进行通信。

## 3.1 设备初始化

### 3.1.1 驱动要求：设备初始化
- 驱动***必须***按照以下顺序去初始化一个设备：
- 1. 重置设备。
- 2. 设置 ACKNOWLEDGE 状态位：表示 guest OS 已经识别到设备。
- 3. 设置 DRIVER 状态位：表示 guest OS 知道如何驱动该设备。
- 4. 读取设备的特性标志位，并将 OS 和驱动能够理解的特性位子集写给设备。在此过程中，驱动***可以***读取（但***不能***写入）设备相关的配置字段，以此来检查驱动能否支持该设备。
- 5. 设置 FEATURES_OK 状态位。在此步骤后，驱动就***不能***再接受新的特性位。
- 6. 重新读取设备状态，并确认 FEATURES_OK 位依然有效：否则就表示，设备不支持驱动的特性子集并且设备不可用。
- 7. 执行设备特定的初始化，包括侦测该设备的 virtqueue，可选的逐总线设置，读取（可能还包括写入）设备的 virtio 配置空间，以及填充 virtqueue。
- 8. 设置 DRIVER_OK 状态位。至此，设备“上线”。
- 如果以上任一步骤出错，驱动***需要***设置 FAILED 状态位来表示：驱动已经放弃该设备（驱动可以重置该设备来重启它）。这种情况下，驱动***不能***继续进行初始化操作。
- 在设置好 DRIVER_OK 前，驱动***不能***向设备发送任何 buffer available notification。

### 3.1.2 旧版接口：设备初始化
- 旧版设备不支持 FEATURES_OK 状态位，因此没有一种系统的方式，让设备表明不支持的特性组合。旧版设备也没有提供一种明确的机制来结束特性协商，这意味着设备在首次使用时就确定了特性，并且无法引入会从根本上改变设备初始操作的特性。
- 旧版驱动常常在设置 DRIVER_OK 位之前就使用设备，有时甚至在向设备写入特性位之前就开始使用了。
- 结果是步骤 5 和 6 被省略了，而步骤 4、7 和 8 被混为一谈。
- 因此，在使用旧版接口时：
- 1. 过渡型驱动***必须***按照 [3.1](#3.1-设备初始化) 所述执行初始化序列，但需省略步骤 5 和 6。
- 2. 过渡型设备***必须***支持驱动在步骤 [4](#3.1-设备初始化) 之前写入设备配置字段。
- 3. 过渡型设备***必须***支持驱动在步骤 [8](#3.1-设备初始化) 之前使用该设备。

## 3.2 设备操作
- 在操作设备时，设备配置空间中的每个字段都可以由驱动或设备进行更改。
- 每当设备触发此类配置更改时，驱动都会收到通知。这使得驱动能够缓存设备配置，避免在未收到通知的情况下进行耗时的配置读取操作。

### 3.2.1 设备配置更改的通知
- 对于那些可以更改设备相关配置信息的设备，当设备相关配置发生更改时，会发送配置更改通知。
- 此外，此通知还会由设备设置 DEVICE_NEEDS_RESET（见 2.1.2）触发。

## 3.3 设备清理
- 一旦驱动设置了 DRIVER_OK 状态位，设备的所有配置 virtqueue 都被视为处于活动状态。一旦设备被重置，设备的任何 virtqueue 都不再处于活动状态。

### 3.3.1 驱动要求：设备清理
- 驱动***不能***更改已公开 buffer（即已提供给设备，但尚未被设备使用的设备活跃 virtqueue 中的 buffer）的 virtqueue 条目。因此，驱动在移除已暴露的 buffer 之前，***必须***确保 virtqueue 已处于关闭状态（通过设备复位操作实现）。

# 4 virtio 传输选项

- virtio 支持多种不同的总线，因此该标准被分为通用部分和总线相关部分。

## 4.1 PCI 总线上的 virtio
- virtio 设备通常用 PCI 设备实现。
- 一个 virtio 设备可以实现成任意类型的 PCI 设备：例如一个通用 PCI 设备，或者一个 PCIe 设备。为了确保符合最新的（PCI 协议）要求，可以参阅 PCI-SIG 官方网站 http://www.pcisig.com。

### 4.1.1 设备要求：PCI 总线上的 virtio
- 一个使用 PCI 总线实现的 virtio 设备，***必须***向客户机提供接口，该接口符合 PCI 协议规范：PCI 或者 PCIe 均可。

### 4.1.2 PCI 设备识别
- 任何具有 PCI Vendor ID 0x1AF4 以及 PCI Device ID 0x1000 至 0x107F（包括这 2 个 ID）的 PCI 设备都是 virtio 设备。此范围内的具体 ID 表明该设备支持哪款 virtio 设备。PCI Device ID 是通过将 virtio Device ID 加上 0x1040 得到的，具体操作如第 5 节所述。此外，这些设备***可以***使用一个过渡性的 PCI Device ID 范围，即 0x1000 到 0x103F，具体取决于具体设备类型。

#### 4.1.2.1 设备要求：PCI 设备识别
- 设备***必须***具有 PCI Vendor ID 0x1AF4。设备***必须***：通过将 virtio Device ID 加上 0x1040 来计算出 PCI 设备 ID（如第 5 节所述），或者根据设备类型采用过渡性 PCI Device ID（具体如下）：

| 过渡性 PCI Device ID | virtio 设备 |
| ---- | ----- |
| 0x1000 | 网络设备 |
| 0x1001 | 块设备 |
| 0x1002 | 内存 ballooning（传统）|
| 0x1003 | 控制台 |
| 0x1004 | SCSI 主机 |
| 0x1005 | entropy source |
| 0x1009 | 9P 传输 |

- 举例：网络设备如果用 virtio Device ID 表示的话，就是 PCI Device ID 0x1041（0x1040 + 1），如果用过渡 PCI Device ID 就是 0x1000。
- PCI 子系统 Vendor ID 和 PCI 子系统 Device ID ***可以***反应环境中的 PCI 供应商和设备 ID（供驱动识别环境使用）。
- 非过渡设备***应该***在 0x1040 到 0x107f 之间取用一个 PCI Device ID。非过渡设备***应该***有一个 1 或者更高的 PCI 修订 ID。非过渡设备***应该***有一个 0x40 或者更高的 PCI 子系统设备 ID。
- 这些是为了降低旧版驱动试图控制该设备的可能性。

#### 4.1.2.2 驱动要求：PCI 设备识别
- 驱动***必须***与 PCI Vendor ID 为 0x1AF4 且 PCI Device ID 在 0x1040 到 0x107f 范围内的设备相匹配（该范围是通过将 virtio Device ID 加上 0x1040 得到的），如第 5 节所述。对于第 4.1.2 节中列出的设备类型，其驱动***必须***与具有 PCI Vendor ID 0x1AF4 和第 4.1.2 节中所指示的过渡 PCI Device ID 的设备相匹配。
- 驱动***必须***匹配任何 PCI 修订 ID。驱动***必须***匹配任何 PCI 子系统 Vendor ID 和任何 PCI 子系统 Device ID。

#### 4.1.2.3 旧版接口：关于 PCI 设备识别的说明
- 过渡设备***必须***具有 PCI 版本标识为 0 的属性。过渡设备***必须***使 PCI 子系统 Device ID 与 virtio 设备 ID 相匹配，如第 5 节所述。过渡设备***必须***将过渡 PCI Device ID 设置在 0x1000 到 0x103f 的范围内。
- 这是为了与旧版驱动兼容。

### 4.1.3 PCI 设备布局
- 设备是通过 I/O 和/或内存区域进行配置的（不过请参阅 4.1.4.9 中关于通过 PCI 配置空间进行访问的内容），其配置方式由 virtio 结构的 PCI 属性所规定。
- 设备配置区域中存在不同大小的字段。所有 64 位，32 位和 16 位字段均为小端格式。64 位字段应被视为两个 32 位字段，低 32 位部分紧随高 32 位部分之后。

#### 4.1.3.1 驱动要求：PCI 设备布局
- 对于设备配置访问，驱动***必须***对 8 位宽字段使用 8 位宽的访问方式，对 16 位宽字段使用 16 位宽且对齐的访问方式，对 32 位宽和 64 位宽字段使用 32 位宽且对齐的访问方式。对于 64 位字段，驱动***可以***分别独立访问该字段的高 32 位部分和低 32 位部分。

#### 4.1.3.2 设备要求：PCI 设备布局
- 对于 64 位设备配置字段，设备***必须***允许驱动独立访问高 32 位和低 32 位字段。

### 4.1.4 virtio 结构 PCI 功能
- virtio 设备配置布局包含几个结构体：
- 1. 通用配置
- 2. 通知
- 3. ISR 状态
- 4. 设备特定配置（可选）
- 5. PCI 配置访问
- 每个结构都可以通过属于该功能的基地址寄存器（BAR）进行映射，或者通过 PCI 配置空间中的特殊 VIRTIO_PCI_CAP_PCI_CFG 字段进行访问。

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