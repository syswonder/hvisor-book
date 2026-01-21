# VirtIO配置文档

这里说明hvisor-tool如何配置virtio backend，介绍各字段的含义。

以下为配置示例：

```json
{
    "zones": [
        {
            "id": 1,
            "memory_region": [
                {
                    "zone0_ipa": "0x50000000",
                    "zonex_ipa": "0x50000000",
                    "size": "0x30000000"
                }
            ],
            "devices": [
                {
                    "type": "net",
                    "addr": "0xa003600",
                    "len": "0x200",
                    "irq": 75,
                    "tap": "tap0",
                    "mac": [
                        "0x02",
                        "0x00",
                        "0x00",
                        "0x00",
                        "0x01",
                        "0x01"
                    ],
                    "status": "enable"
                },
                {
                    "type": "console",
                    "addr": "0xa003800",
                    "len": "0x200",
                    "irq": 76,
                    "status": "enable"
                },
                {
                    "type": "blk",
                    "addr": "0xa003c00",
                    "len": "0x200",
                    "irq": 78,
                    "img": "rootfs2.ext4",
                    "status": "enable"
                }
            ]
        }
    ]
}
```

由于virtio需要前端和后端通过共享内存交换数据，virtio的virtqueue以及数据用buffer均分配在Guest的内存中，zone0上运行的virtio-backend需要能够访问zonex的内存以交换数据。

注意先启动virtio backend，再启动使用virtio设备的zonex。

相关字段说明：

- `id`字段指定了zonex的ID，用于唯一标识该zonex。
- `memory_region`描述了虚拟机的内存区域，包括中间物理地址（IPA）和大小。
- `zone0_ipa`表示zone0访问zonex内存的起始IPA。
- `zonex_ipa`表示zonex内存区域的起始IPA。
- `size`表示zonex内存区域的大小。
- `devices`描述了zone0提供的virtio设备，包括网络设备、控制台设备和块设备。
- `type`字段指定了设备的类型，包括`net`、`console`和`blk`。
- `addr`字段指定了设备的MMIO起始地址（需要和zonex中的设备树匹配）。
- `len`字段指定了设备的MMIO长度。
- `irq`字段指定了设备使用的中断号。
- `tap`字段指定了虚拟网卡的名称，需要提前在root linux中创建网桥，并将该tap设备添加到网桥中。
- `mac`字段指定了该虚拟网卡的MAC地址。
- `img`字段指定了块设备的镜像文件路径。
- `status`字段指定了设备的状态，包括`enable`和`disable`。

virtio不同设备的实现，参考：

- [virtio-blk文档](../chap04/subchap03/VirtIO/BlockDevice.md)
- [virtio-net文档](../chap04/subchap03/VirtIO/NetworkDevice.md)
- [virtio-console文档](../chap04/subchap03/VirtIO/ConsoleDevice.md)

更多内容请详见：[hvisor-tool-README](https://github.com/syswonder/hvisor-tool?tab=readme-ov-file#readme)