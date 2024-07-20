# Zone的配置和管理

hvisor项目作为一款轻量级的hypervisor，它使用了Type-1架构，允许在硬件之上直接运行多个虚拟机（zones）。下面是对zone配置和管理的关键点的详细说明：

## 资源分配

资源如CPU、内存、设备和中断对每个zone都是静态分配的，这意味着一旦分配，这些资源就不会在zones之间动态调度。

## Root Zone配置

根zone的配置是硬编码在hvisor内部的，以Rust语言编写，并表现为一个C风格的结构体HvZoneConfig。这个结构体包含了zone ID、CPU数量、内存区域、中断信息、内核和设备树二进制（DTB）的物理地址与大小等关键信息。

## Non-root Zones配置

非root zones的配置则存储在root linux的文件系统中，通常以JSON格式表示。例如：

```json
    {
        "arch": "arm64",
        "zone_id": 1,
        "cpus": [2, 3],
        "memory_regions": [
            {
                "type": "ram",
                "physical_start": "0x50000000",
                "virtual_start":  "0x50000000",
                "size": "0x30000000"
            },
            {
                "type": "io",
                "physical_start": "0x30a60000",
                "virtual_start":  "0x30a60000",
                "size": "0x1000"
            },
            {
                "type": "virtio",
                "physical_start": "0xa003c00",
                "virtual_start":  "0xa003c00",
                "size": "0x200"
            }
        ],
        "interrupts": [61, 75, 76, 78],
        "kernel_filepath": "./Image",
        "dtb_filepath": "./linux2.dtb",
        "kernel_load_paddr": "0x50400000",
        "dtb_load_paddr":   "0x50000000",
        "entry_point":      "0x50400000"
    }
```

- `arch`字段指定了目标架构（例如arm64）。
- `cpus`是一个列表，指明了分配给该zone的CPU核心ID。
- `memory_regions`描述了不同类型的内存区域及其物理和虚拟起始地址与大小。
- `interrupts`列出了分配给zone的中断号。
- `kernel_filepath`和`dtb_filepath`分别指明了内核和设备树二进制文件的路径。
- `kernel_load_paddr`和`dtb_load_paddr`则是内核和设备树二进制在物理内存中的加载地址。
- `entry_point`指定了内核的入口点地址。

root linux的管理工具负责读取JSON配置文件并将其转换为C风格的结构体，随后传递给hvisor以启动非root zones。