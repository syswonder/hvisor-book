# hvisor Book 结构

[简介](./Introduction.md)

# 关于 hvisor

- [hvisor 概述](./chap01/Overview.md)

- [hvisor 支持的指令集和处理器](./chap01/ISA.md)

- [hvisor 支持的硬件平台](./chap01/Board.md)

- [hvisor 硬件适配开发手册 🧑🏻‍💻](./chap01/PlatformDev.md)

# hvisor 快速上手指南

- [Qemu AArch64 快速上手](./chap02/QemuAArch64.md)

- [Qemu RISC-V 快速上手](./chap02/QemuRISC-V.md)

- [NXP i.MX 8 快速上手](./chap02/NXPIMX8.md)

- [FPGA-Rockechip 快速上手](./chap02/FPGA-Rockechip.md)

- [龙芯 3A5000 hvisor 快速上手](./chap02/Loongson-3A5000.md)

- [Xilinx ZCU102 hvisor 快速上手](./chap02/subchap01/Xilinx-ZCU102.md)

	- [Qemu ZCU102 hvisor 启动](./chap02/subchap01/Qemu-ZCU102.md)
	
	- [Board ZCU102 hvisor 多模式启动](./chap02/subchap01/Board-ZCU102.md)
	
	- [ZCU102 Nonroot 启动](./chap02/subchap01/Nonroot-ZCU102.md)
	
    - [UBOOT FIT 镜像制作、加载与启动](./chap02/subchap01/UbootFitImage-ZCU102.md)

- [RK3588 hvisor 快速上手](./chap02/RK3588.md)

- [FPGA 香山昆明湖快速上手]()


# hvisor 使用手册

- [如何编译](./chap03/Compile.md)

- [启动管理 Linux VM](./chap03/BootRootLinux.md)

- [启动两个VM：Linux1 和 Linux2](./chap03/BootNonRootLinux.md)

- [启动两个VM：Linux 和 RTOS](./chap03/BootNonRootRTOS.md)

- [ZONE的配置与管理](./chap03/ZoneConfig.md)

- [命令行工具](./chap03/CMDTools.md)

- [VirtIO 的使用](./chap03/VirtIOUseage.md)

# hvisor架构与实现

- [hvisor 架构](./chap04/Structure.md)
- [hvisor 启动与运行](./chap04/BootAndRun.md)
- [CPU 虚拟化](./chap04/subchap01/CPUVirtualization.md)

    - [PerCPU 定义](./chap04/subchap01/PerCPU.md)

    - [ARM 处理器虚拟化](./chap04/subchap01/ARMVirtualization.md)

    - [RISC-V 处理器虚拟化](./chap04/subchap01/RISCVirtualization.md)
    
    - [LoongArch 处理器虚拟化](./chap04/subchap01/LoongArchVirtualization.md)
- [内存虚拟化](./chap04/MemVirtualization.md)
- [中断虚拟化](./chap04/subchap02/InterruptVirtualization.md)

    - [ARM 中断控制 GIC](./chap04/subchap02/ARM-GIC.md)

    - [RISC-V 中断控制 PLIC](./chap04/subchap02/RISC-PLIC.md) 

    - [RISC-V 中断控制 AIA](./chap04/subchap02/RISC-AIA.md)

    - [LoongArch 中断控制](./chap04/subchap02/LoongArch-Controller.md)
- [I/O 虚拟化](./chap04/subchap03/IO-Virtualization.md)

    - [IOMMU](./chap04/subchap03/IOMMU/IOMMU-Define.md)

        - [ARM SMMU 的实现](./chap04/subchap03/IOMMU/ARM-SMMU.md)

        - [RISC-V IOMMU 标准的实现](./chap04/subchap03/IOMMU/RISC-IOMMU.md)
- [VirtIO](./chap04/subchap03/VirtIO/VirtIO-Define.md)
  
    - [Block](./chap04/subchap03/VirtIO/BlockDevice.md)
    - [Net](./chap04/subchap03/VirtIO/NetDevice.md)
    - [Console](./chap04/subchap03/VirtIO/ConsoleDevice.md)
    - [GPU](./chap04/subchap03/VirtIO/GPUDevice.md)
- [PCI 虚拟化](./chap04/subchap03/PCI-Virtualization.md)
- [Hvisor 管理工具](./chap04/subchap04/ManageTools.md)
  
    - [Hypercall](./chap04/subchap04/HyperCall.md)

# hvisor 的规划

- [TODO]()

