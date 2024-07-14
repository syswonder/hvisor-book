# LoongArch处理器虚拟化简介

LoongArch指令集是中国龙芯中科公司于2020年发布的自主RISC指令集，包括基础指令集、二进制翻译拓展（LBT）、向量拓展（LSX）、高级向量扩展（LASX）和虚拟化拓展（LVZ）五个模块。

本文主要对LoongArch指令集的CPU虚拟化相关设计进行简要介绍。

## LoongArch寄存器简介

## CPU模式

Host/Guest

### GCSR寄存器组


### CSR中涉及虚拟化的寄存器


### 进入Guest模式的流程（KVM）

https://github.com/torvalds/linux/blob/master/arch/loongarch/kvm/switch.S


1. 【switch_to_guest】：

2. 清空CSR.ECFG.VS字段（设置为0，即所有异常共用一个入口地址）
3. 读取hypervisor中保存的guest eentry（客户OS中断向量地址）-> GEENTRY
   1. 然后将GEENTRY写入CSR.EENTRY
4. 读取hypervisor中保存的guest era（客户OS异常返回地址）-> GPC
   1. 然后将GPC写入CSR.ERA
5. 读取CSR.PGDL全局页表地址，存到hypervisor中
6. 从hypervisor中加载guest pgdl到CSR.PGDL
7. 读出CSR.GSTAT.GID和CSR.GTLBC.TGID，将这两个位域混合到一个32位的数中，写入CSR.GTLBC
8. 将CSR.PRMD.PIE置1，打开hypervisor级的全局中断
   1. PRMD - 当触发例外时，如果例外类型不是 TLB 重填例外和机器错误例外，硬件会将此时处理器核的特权等级、全局中断使能和监视点使能位保存至例外前模式信息寄存器（PRMD）中，用于例外返回时恢复处理器核的现场。
9. 将CSR.GSTAT.PVM置1，其目的是使ertn指令进入guest mode
10. hypervisor将自己保存的该guest的通用寄存器（GPRS）恢复到硬件寄存器上（恢复现场）。
11. **执行ertn指令，进入guest。**


## 虚拟化相关的异常

22 - GSPR **客户机特权级别错误** id=22, Guest Privileged Error 

23 - HVC **调用hypervisor的异常** id=23, Hypercall 

24 - GCM **客户机CSR修改异常** id=24, Guest CSR modified 进入guest mode后，普通的寄存器操作在物理CPU上调度执行，客户OS若要修改CSR，则应该发出这个异常？然后进入后文提到的客户异常中断处理向量（区别hypervisor的异常中断向量）。

GCM_SUBCODE

0 - GCSC code=0, Software caused 判断客户机CSR修改是软件发出的

1 - GCHC code=1, Hardware caused 判断客户机CSR修改是硬件发出的

### 处理Guest模式下异常的流程（KVM）

https://github.com/torvalds/linux/blob/master/arch/loongarch/kvm/switch.S

1. 【kvm_exc_entry】：（guest出现exception时应配置为跳转到这里！）

2. hypervisor首先保存好guest的通用寄存器（GPRS），保护现场。
3. hypervisor保存CSR.ESTAT -> host ESTAT
4. hypervisor保存CSR.ERA -> GPC
5. hypervisor保存CSR.BADV -> host BADV，即触发地址错误例外时，记录出错的Virtual Address
6. hypervisor保存CSR.BADI -> host BADI，该寄存器用于记录触发同步类例外的指令的指令码，所谓同步类例外是指除了中断（INT）、客户机CSR硬件修改例外（**GCHC**）、机器错误例外（MERR）之外的所有例外。
7. 读取hypervisor保存好的host ECFG，写入CSR.ECFG（即切换到host下的异常配置）
8. 读取hypervisor保存好的host EENTRY，写入CSR.EENTRY
9. 读取hypervisor保存好的host PGD，写入CSR.PGDL（恢复host页表全局目录基址，低半空间）
10. 设置CSR.GSTAT.PVM/PGM关闭，其作用是“使下次ertn默认进入的是root mode”
11. 清空GTLBC.TGID域
12. 恢复kvm per cpu寄存器
    1. kvm汇编里涉及到KVM_ARCH_HTP, KVM_ARCH_HSP, KVM_ARCH_HPERCPU
13. **跳转到KVM_ARCH_HANDLE_EXIT位置处理异常**（采用的jirl指令，说明这是一个函数调用，执行完之后会返回到kvm_exc_entry里jirl指令的下一条继续执行。
14. 判断刚才的函数ret是否<=0
    1. 若<=0，则继续运行host
    2. 否则继续运行guest，保存percpu寄存器，“因为可能会切换到不同的CPU继续运行guest”
       1. 保存host percpu寄存器到CSR.KSave寄存器里

15. 跳转到switch_to_guest

## vCPU上下文需要保存的寄存器

LoongArch函数调用规范

s0-s9
sp
ra