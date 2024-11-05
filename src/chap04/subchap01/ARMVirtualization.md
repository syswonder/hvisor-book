# AArch64下的CPU虚拟化

## CPU启动机制

在AArch64架构下，hvisor利用`psci::cpu_on()`函数唤醒指定的CPU核心，将其从关闭状态带入运行状态。该函数接收CPU的ID、启动地址以及一个不透明参数作为输入。遇到错误时，如CPU已处于唤醒状态，函数会进行适当的错误处理避免重复唤醒。

## CPU虚拟化初始化与运行

`ArchCpu`结构体封装了特定于架构的CPU信息和功能，其`reset()`方法负责将CPU设置为虚拟化模式的初始状态。这包括：

- 设置ELR_EL2寄存器至指定的入口点
- 配置SPSR_EL2寄存器
- 清空通用寄存器
- 重置虚拟机相关寄存器
- `activate_vmm()`，激活虚拟内存管理器（VMM）

`activate_vmm()`方法用于配置VTCR_EL2和HCR_EL2寄存器，启用虚拟化环境。

`ArchCpu`的`run()`和`idle()`方法分别用于启动和闲置CPU。启动时，激活zone的GPM（Guest Page Management），重置到指定的入口点和设备树二进制（DTB）地址，然后通过`vmreturn`宏跳转到EL2入口点。在闲置模式下，CPU被重置到等待状态（WFI），并准备`parking`指令页面以供闲置期间使用。

## EL1与EL2之间的切换

hvisor在AArch64架构中使用EL2作为hypervisor模式，而EL1用于guest OS。`handle_vmexit`宏处理从EL1到EL2的上下文切换（VMEXIT事件），保存用户模式寄存器上下文，调用外部函数处理退出原因，之后返回到hypervisor代码段继续执行。`vmreturn`函数用于从EL2模式回到EL1模式（VMENTRY事件），恢复用户模式寄存器上下文后，通过`eret`指令返回到guest OS的代码段。

## MMU配置与启用

为了支持虚拟化，`enable_mmu()`函数在EL2模式下配置MMU映射，包括设置MAIR_EL2、TCR_EL2和SCTLR_EL2寄存器，允许指令和数据缓存能力，并确保虚拟范围覆盖整个48位地址空间。

通过这些机制，hvisor在AArch64架构上实现了高效的CPU虚拟化，允许多个独立的zones在静态分配的资源下运行，同时保持系统稳定性和性能。