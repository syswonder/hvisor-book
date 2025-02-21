PCI设备主要有三个空间：配置空间（Configuration Space）、内存空间（Memory Space）和I/O空间（I/O Space）。

### 1. 配置空间（Configuration Space）
- **用途**：用于设备初始化和配置。
- **大小**：每个PCI设备都有256字节的配置空间。
- **访问方式**：通过总线编号、设备编号和功能编号进行访问。
- **内容**：
  - 设备标识信息（如供应商ID、设备ID）。
  - 状态和命令寄存器。
  - 基地址寄存器（BARs），用于映射设备的内存空间和I/O空间。
  - 中断线和中断引脚等信息。

### 2. 内存空间（Memory Space）
- **用途**：用于访问设备的寄存器和内存，适用于高带宽访问。
- **大小**：由设备制造商定义，映射到系统内存地址空间中。
- **访问方式**：通过内存读写指令进行访问。
- **内容**：
  - 设备寄存器：用于控制和状态读取。
  - 设备专用内存：如帧缓冲区、DMA缓冲区等。

### 3. I/O空间（I/O Space）
- **用途**：用于访问设备的控制寄存器，适用于低带宽访问。
- **大小**：由设备制造商定义，映射到系统的I/O地址空间中。
- **访问方式**：通过特殊的I/O指令（如`in`和`out`）进行访问。
- **内容**：
  - 设备控制寄存器：用于执行特定的I/O操作。

### 总结
- **配置空间**主要用于设备初始化和配置。
- **内存空间**用于高速访问设备的寄存器和内存。
- **I/O空间**用于低速访问设备的控制寄存器。

pci的虚拟化主要是对上述的三个空间做管理。考虑到多数设备中并不存在多条pci总线，且该pci总线的所有权一般归属于zone0，为了保证zone0中pci设备的访问速度，当不存在需要将该总线上的设备分配给其他zone时，hvisor并不会对zone0的pci总线及pci设备做任何处理。

在将PCI设备分配给zone时，我们需要确保zone0中的Linux不再使用它们。只要设备被分配给其他zone，那么zone0就不应该访问这些设备。很遗憾，我们不能仅仅使用PCI热插拔来在运行时移除/重新添加设备，因为Linux可能会重新编程BARs并定位资源到我们不期望或者不允许的位置。因此，需要一个存在于zone0内核中的驱动拦截对这些PCI设备的访问，我们将目光投向了hvisor tool。

hvisor tool会将自己注册为一个PCI虚拟驱动程序，并在其他zone使用这些设备时声明对这些设备的管理权。在创建zone之前，hvisor会让这些设备从他们自己的驱动程序中解绑并绑定到hvisor tool。当一个zone被销毁时，这些设备实际上已经没有zone使用，但是在zone0看来hvisor tool仍然是一个有效的虚拟驱动程序，所以设备的释放需要手动完成。hvisor tool会释放绑定到这些zone的设备，从zone0 linux的角度来看，这些设备不被绑定到任何驱动程序，那么如果需要使用这些设备linux会自动的重新绑定正确的驱动程序。

现在我们需要让zone能够正确的访问到pci设备，为了尽可能简洁的完成这一目标，我们直接复用了pci总线的结构，也就是说，关于pci总线的内容会同时出现在需要使用该总线上设备的设备树中，但是除了真正拥有这条总线的zone以外，其他zone只能通过mmio由hvisor代理访问该设备，当zone试图访问一个PCI设备时，hvisor会检查是否拥有该设备的所有权，所有权在zone创建时一同被声明。如果zone访问一个属于自己的设备的配置空间，hvisor会正确的返回信息。

目前，I/O空间和内存空间的处理方案与配置空间相同。因为bars资源的唯一性，配置空间不可能被直接分配给zone，且对bar空间的访问频率很低，并不会过多的影响效率。但是I/O空间和内存空间的直接分配是理论上可行，进一步会将I/O空间和内存空间直接分配给对应的zone以提高访问速度。

为了方便在QEMU中测试PCI虚拟化，我们编写了一个PCI设备。

# PCIe资源分配与隔离

## 资源分配方式

在每个zone的配置文件中，通过 `num_pci_devs`指定分配给该zone的PCIe设备的数量，通过 `alloc_pci_devs`指定这些设备的BDF。注意，必须包括0。

例如：

```json
{
    "arch": "riscv",
    "name": "linux2",
    "zone_id": 1,
    ///
    "num_pci_devs": 2,
    "alloc_pci_devs": [0, 16]
}
```

## virt PCI

```rust
pub struct PciRoot {
    endpoints: Vec<EndpointConfig>,
    bridges: Vec<BridgeConfig>,
    alloc_devs: Vec<usize>, // include host bridge
    phantom_devs: Vec<PhantomCfg>,
    bar_regions: Vec<BarRegion>,
}
```

需要说明的是，`phantom_devs`是不属于这个虚拟机的设备；`bar_regions`是属于该虚拟机的设备的BAR空间。

### phantom_dev

这部分代码在`src/pci/phantom_cfg.rs`中可以找到，当虚拟机第一次访问到不属于自己的设备时，创建`phantom_dev`。

处理函数在`src/pci/pci.rs`中的`mmio_pci_handler`可以找到，这是我们处理虚拟机对配置空间的访问的函数。

#### header

**hvisor让每个虚拟机看到同样的PCIe拓扑，这样能够避免BAR和bus号分配不同带来的复杂处理，尤其是对于桥设备中的涉及TLB转发的配置，能够节省很多功夫。**

但对于不是分配给该虚拟机的Endpoint，将其虚拟为`phantom_dev`，访问header时应该返回特定的`vendor-id`和`device-id`，例如0x77777777，以及返回reserved `class-code`，对于这类存在但无法找到对应驱动的设备，虚拟机只会在枚举阶段进行一些基础的配置，如BAR的预留。

#### capabilities

capabilities部分涉及到MSI的配置等，当虚拟机访问`capabilities-pointer`时返回0，代表该设备无capabilities，防止对设备所属虚拟机的配置(例如BAR空间中的MSI-TABLE的配置内容)进行覆盖。

#### command

另外对于`COMMAND`寄存器，虚拟机检测到没有`MSI capabilities`，则会将传统中断打开，这涉及到`COMMAND`寄存器中的`DisINTx`字段的设置，硬件要求MSI和legacy只能选择其一，避免虚拟机之间设置的矛盾(本来非所属虚拟机也不应该设置)，故我们需要一个虚拟的`COMMAND`寄存器。

### 关于BAR

这部分代码在`src/pci/pcibar.rs`中可以找到。

```rust
pub struct PciBar {
    val: u32,
    bar_type: BarType,
    size: usize,
}

pub struct BarRegion{
    pub start: usize,
    pub size: usize,
    pub bar_type: BarType
}

pub enum BarType {
    Mem32,
    Mem64,
    IO,
    #[default]
    Unknown,
}
```

每个虚拟机看到同样的拓扑，则BAR空间的分配完全相同。

那么在非root虚拟机启动时，直接读取root配置好的BAR，就能得知每个虚拟机应该访问的BAR空间是哪些(由分配给它的设备决定)。

如果当虚拟机访问BAR的时候再陷入hypervisor进行代理，效率就低了，我们应该让硬件做这个事情，直接将这段空间写入虚拟机的stage-2页表中，注意`pci_bars_register`函数中，填入页表时，要根据`BarRegion`的`BarType`，找到该类型的PCI地址与CPU地址的映射关系(写在了设备树中，同时同步于配置文件的`pci_config`)，将BAR配置中的PCI地址转为对应的CPU地址再写入页表。

**上述从root配置好的BAR中获取BAR分配结果的方法主要是，区分Endpoint和Bridge(这是因为二者的BAR数量不同)，根据BDF访问配置空间，首先读取root的配置结果，再写入全1获得大小，再写回配置结果。具体代码可结合`endpoint.rs`，`bridge.rs`以及`pcibar.rs`查看，其中涉及到64位内存地址的需要特别注意。**