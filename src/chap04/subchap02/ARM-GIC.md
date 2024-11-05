# ARM GICv3模块

## 1. GICv3模块

### GICv3初始化流程

hvisor中的GICv3初始化流程涉及了GIC分布控制器（GICD）和GIC重新分布控制器（GICR）的初始化，以及中断处理和虚拟中断注入的机制。这一过程的关键步骤：

- SDEI版本检查：通过smc_arg1!(0xc4000020)获取Secure Debug Extensions Interface (SDEI)的版本信息。
- ICCs配置：设置icc_ctlr_el1以仅提供优先级下降功能，设置icc_pmr_el1以定义中断优先级掩码，使能Group 1 IRQs。
- 清除待处理中断：调用gicv3_clear_pending_irqs函数，清除所有待处理的中断，确保系统处于干净状态。
- VMCR和HCR配置：设置ich_vmcr_el2和ich_hcr_el2寄存器，使能虚拟化CPU接口，准备虚拟中断处理。

### 待处理中断处理

- `pending_irq`函数读取`icc_iar1_el1`寄存器，返回当前正在处理的中断ID，若值大于等于0x3fe则视为无效中断。
- `deactivate_irq`函数通过写入`icc_eoir1_el1`和`icc_dir_el1`寄存器来清除中断标志，使能中断。

### 虚拟中断注入

- `inject_irq`函数检查是否有可用的`List Register (LR)`，并将虚拟中断信息写入其中。此函数区分硬件中断和软件生成中断，适当设置LR中的字段。

### GIC数据结构初始化

- GIC是一个全局的Once容器，用于延迟初始化Gic结构体，其中包含了GICD和GICR的基地址及其大小。
- primary_init_early和primary_init_late函数分别在早期和后期初始化阶段配置GIC，使能中断。

### 区域（Zone）级别的初始化

在Zone结构体中，`arch_irqchip_reset`方法负责重置分配给特定zone的所有中断，通过直接写入GICD的ICENABLER和ICACTIVER寄存器来实现。

## 2. vGICv3模块

hvisor的VGICv3（Virtual Generic Interrupt Controller version 3）模块提供了对ARMv8-A架构中GICv3的虚拟化支持。它通过MMIO（Memory Mapped I/O）访问和中断比特图管理来控制和协调不同zone（虚拟机实例）间的中断请求。

### MMIO区域注册

在初始化阶段，`Zone`结构体的`vgicv3_mmio_init`方法注册了GIC分布控制器（GICD）和每个CPU的GIC重新分布控制器（GICR）的MMIO区域。MMIO区域注册是通过`mmio_region_register`函数完成的，该函数关联了特定的处理器或中断控制器地址，以及相应的处理函数`vgicv3_dist_handler`和`vgicv3_redist_handler`。

### 中断比特图初始化

`Zone`结构体的`irq_bitmap_init`方法用于初始化中断比特图，这是为了跟踪哪些中断属于当前`zone`。通过遍历提供的中断列表，每个中断都会被插入到比特图中。`insert_irq_to_bitmap`函数负责将特定的中断号映射到比特图中的相应位置。
MMIO访问限制

`restrict_bitmask_access`函数用于限制对`GICD`寄存器的`MMIO`访问，确保只有属于当前`zone`的中断才能被修改。该函数检查访问是否针对当前zone的中断，如果是，则更新访问掩码，以允许或限制特定的读写操作。

### VGICv3 MMIO处理

`vgicv3_redist_handler`和`vgicv3_dist_handler`函数分别处理GICR和GICD的MMIO访问。`vgicv3_redist_handler`函数处理GICR的读写操作，检查是否访问的是当前`zone`的GICR，如果是，则允许访问；否则，忽略该访问。`vgicv3_dist_handler`函数根据不同的GICD寄存器类型，调用`vgicv3_handle_irq_ops`或`restrict_bitmask_access`函数，以适当地处理中断路由和配置寄存器的访问。

通过上述机制，hvisor能够有效地管理跨zone的中断，确保每个zone只能够访问和控制分配给它的中断资源，同时提供必要的隔离性。这使得在多zone环境中，VGICv3能够高效、安全地工作，支持复杂的虚拟化场景。
