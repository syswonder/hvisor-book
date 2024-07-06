# Virtio Block

Virtio磁盘设备的实现遵循Virtio规范约定，采用MMIO的设备访问方式供其他虚拟机发现和使用。目前支持`VIRTIO_BLK_F_SEG_MAX`、`VIRTIO_BLK_F_SIZE_MAX`、`VIRTIO_F_VERSION_1`、`VIRTIO_RING_F_INDIRECT_DESC`和`VIRTIO_RING_F_EVENT_IDX`五种特性。

## Virtio设备的顶层描述——VirtIODevice

一个Virtio设备由VirtIODevice结构体表示，该结构体包含设备ID、Virtqueue的个数vqs_len、所属的虚拟机ID、设备中断号irq_id、MMIO区域的起始地址base_addr、MMIO区域长度len、设备类型type、部分由设备保存的MMIO寄存器regs、Virtqueue数组vqs、指向描述特定设备信息的指针dev。通过这些信息，可以完整描述一个Virtio设备。

```c
// The highest representations of virtio device
struct VirtIODevice
{
    uint32_t id;
    uint32_t vqs_len;
    uint32_t zone_id;
    uint32_t irq_id;
    uint64_t base_addr; // the virtio device's base addr in non root zone's memory
    uint64_t len;       // mmio region's length
    VirtioDeviceType type;
    VirtMmioRegs regs;
    VirtQueue *vqs;
    // according to device type, blk is BlkDev, net is NetDev, console is ConsoleDev.
    void *dev;          
    bool activated;
};

typedef struct VirtMmioRegs {
    uint32_t device_id;
    uint32_t dev_feature_sel;
    uint32_t drv_feature_sel;
    uint32_t queue_sel;
    uint32_t interrupt_status;
    uint32_t interrupt_ack;
    uint32_t status;
    uint32_t generation;
    uint64_t dev_feature;
    uint64_t drv_feature;
} VirtMmioRegs;
```

## Virtio Block设备的描述信息

对于Virtio磁盘设备，VirtIODevice中type字段为VirtioTBlock，vqs_len为1，表示只有一个Virtqueue，dev指针指向描述磁盘设备具体信息的virtio_blk_dev结构体。virtio_blk_dev中config用来表示设备的数据容量和一次数据传输中最大的数据量，img_fd为该设备打开的磁盘镜像的文件描述符，tid、mtx、cond用于工作线程，procq为工作队列，closing用来指示工作线程何时关闭。virtio_blk_dev和blkp_req结构体的定义见图4.6。

```c
typedef struct virtio_blk_dev {
    BlkConfig config;
    int img_fd;
	// describe the worker thread that executes read, write and ioctl.
	pthread_t tid;
	pthread_mutex_t mtx;
	pthread_cond_t cond;
	TAILQ_HEAD(, blkp_req) procq;
	int close;
} BlkDev;

// A request needed to process by blk thread.
struct blkp_req {
	TAILQ_ENTRY(blkp_req) link;
    struct iovec *iov;
	int iovcnt;
	uint64_t offset;
	uint32_t type;
	uint16_t idx;
};
```

## Virtio Block设备工作线程

每个Virtio磁盘设备，都拥有一个工作线程和工作队列。工作线程的线程ID保存在virtio_blk_dev中的tid字段，工作队列则是procq。工作线程负责进行数据IO操作及调用中断注入系统接口。它在Virtio磁盘设备启动后被创建，并不断查询工作队列中是否有新的任务，如果队列为空则等待条件变量cond，否则处理任务。

当驱动写磁盘设备MMIO区域的QueueNotify寄存器时，表示可用环中有新的IO请求。Virtio磁盘设备（位于主线程的执行流）收到该请求后，首先会读取可用环得到描述符链的第一个描述符，第一个描述符指向的内存缓冲区包含了IO请求的类型（读/写）、要读写的扇区编号，之后的描述符指向的内存缓冲区均为数据缓冲区，对于读操作会将读到的数据存入这些数据缓冲区，对于写操作则会从数据缓冲区获取要写入的数据，最后一个描述符对应的内存缓冲区（结果缓冲区）用于设备描述IO请求的完成结果，可选项有成功（OK）、失败（IOERR）、不支持的操作（UNSUPP）。 据此解析整个描述符链即可获得有关该IO请求的所有信息，并将其保存在blkp_req结构体中，该结构体中的字段iov表示所有数据缓冲区，offset表示IO操作的数据偏移量，type表示IO操作的类型（读/写），idx为描述符链的首描述符索引，用于更新已用环。随后设备会将blkp_req加入到工作队列procq中，并通过signal函数唤醒阻塞在条件变量cond上的工作线程。工作线程即可对任务进行处理。

工作线程获取到任务后，会根据blkp_req指示的IO操作信息通过preadv和pwritev函数读写img_fd所对应的磁盘镜像。完成读写操作后，会首先更新描述符链的最后一个描述符，该描述符用于描述IO请求的完成结果，例如成功、失败、不支持该操作等。然后更新已用环，将该描述符链的首描述符写到新的表项中。随后进行中断注入，通知其他虚拟机。

工作线程的设立，可以有效地将耗时操作分散到其他CPU核上，提高主线程分发请求的效率和吞吐量，提升设备性能。