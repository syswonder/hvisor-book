# RISCV下的CPU虚拟化

摘要：围绕ArchCpu结构，介绍RISCV架构下的CPU虚拟化工作。

## 涉及的两个数据结构

hvisor支持多种架构，每个架构的CPU虚拟化需要做的工作不同，但在一个系统中又应该提供统一的接口，故我们将CPU拆分成 `PerCpu` 和 `ArchCpu` 两个数据结构。

### PerCpu

这是一个通用的CPU的描述，在 `PerCpu` 的文档中已给出介绍。

### ArchCpu

`ArchCpu` 是针对具体架构（**本文中介绍RISCV架构**）的CPU结构。由这个结构承担CPU具体的行为。

在ARM架构下，也有对应的 `ArchCpu` ，与本节介绍的 `ArchCpu` 具体结构略有不同，但他们具有相同的接口（也就是都具有初始化等行为）。

包含的字段如下：

```
pub struct ArchCpu {
    pub x: [usize; 32], //x0~x31
    pub hstatus: usize,
    pub sstatus: usize,
    pub sepc: usize,
    pub stack_top: usize,
    pub cpuid: usize,
    // pub first_cpu: usize,
    pub power_on: bool,
    pub init: bool,
    pub sstc: bool,
}
```

各个字段的解释如下：

- `x` ：通用寄存器的值
- `hstatus` ：存储Hypervisor状态寄存器的值
- `sstatus` ：存储Supervisor状态寄存器的值，管理S模式的状态信息，如中断使能标志等
- `sepc` ：异常处理结束的返回地址
- `stack_top` ：对应的cpu栈的栈顶
- `power_on` ：该cpu是否被开启
- `init` ：该cpu是否已初始化
- `sstc` ：是否配置了定时器中断

## 相关方法

这一部分讲解涉及的方法。

### ArchCpu::init

这个方法主要是对cpu进行初始化工作，设置初次进入vm的时候的上下文，以及一些CSR的初始化。

```
pub fn init(&mut self, entry: usize, cpu_id: usize, dtb: usize) {
        write_csr!(CSR_SSCRATCH, self as *const _ as usize); //arch cpu pointer
        self.sepc = entry;
        self.hstatus = 1 << 7 | 2 << 32; //HSTATUS_SPV | HSTATUS_VSXL_64
        self.sstatus = 1 << 8 | 1 << 63 | 3 << 13 | 3 << 15; //SPP
        self.stack_top = self.stack_top() as usize;
        self.x[10] = cpu_id; //cpu id
        self.x[11] = dtb; //dtb addr

        set_csr!(CSR_HIDELEG, 1 << 2 | 1 << 6 | 1 << 10); //HIDELEG_VSSI | HIDELEG_VSTI | HIDELEG_VSEI
        set_csr!(CSR_HEDELEG, 1 << 8 | 1 << 12 | 1 << 13 | 1 << 15); //HEDELEG_ECU | HEDELEG_IPF | HEDELEG_LPF | HEDELEG_SPF
        set_csr!(CSR_HCOUNTEREN, 1 << 1); //HCOUNTEREN_TM
                                          //In VU-mode, a counter is not readable unless the applicable bits are set in both hcounteren and scounteren.
        set_csr!(CSR_SCOUNTEREN, 1 << 1);
        write_csr!(CSR_HTIMEDELTA, 0);
        set_csr!(CSR_HENVCFG, 1 << 63);
        //write_csr!(CSR_VSSTATUS, 1 << 63 | 3 << 13 | 3 << 15); //SSTATUS_SD | SSTATUS_FS_DIRTY | SSTATUS_XS_DIRTY

        // enable all interupts
        set_csr!(CSR_SIE, 1 << 9 | 1 << 5 | 1 << 1); //SEIE STIE SSIE
                                                     // write_csr!(CSR_HIE, 1 << 12 | 1 << 10 | 1 << 6 | 1 << 2); //SGEIE VSEIE VSTIE VSSIE
        write_csr!(CSR_HIE, 0);
        write_csr!(CSR_VSTVEC, 0);
        write_csr!(CSR_VSSCRATCH, 0);
        write_csr!(CSR_VSEPC, 0);
        write_csr!(CSR_VSCAUSE, 0);
        write_csr!(CSR_VSTVAL, 0);
        write_csr!(CSR_HVIP, 0);
        write_csr!(CSR_VSATP, 0);
    }
```

`write_csr!(CSR_SSCRATCH, self as *const _ as usize)` 延续了上一个方法的内容，将 `ArchCpu` 的地址写入 `sscratch` 。将返回地址设置为入口，将 `hstatus` 的 `SPV` 字段设置为1，代表返回vm的时候vm是运行在VS态下的（或理解为异常发生前的vm是在VS态运行的）；`VSXL` 字段设置了VS模式下寄存器的长度。`sstatus` 的  `SPP` 等字段给出 Trap 发生之前 CPU 处在哪个特权级等信息。`SPP` 与 `SPV` 字段，结合起来使用，能够确定在HS模式下执行 `sret` 指令应该返回哪个特权级，返回的地址由 `spec` 设置。

`HIDELEG` 和 `CSR_HEDELEG` 这两个寄存器的设置是将某些中断委托给VS态处理。`HCOUNTEREN` 和 `SCOUNTEREN` 用于限制vm能够访问的性能计数器，此处是使能了 `TM` 字段，代表允许访问 `time` 寄存器。`HTIMEDELTA` 用于调整vm读取 `time` 寄存器的值，在VS或者VU模式下读取 `time` 将返回 `HTIMEDELTA` 和 `time` 的和。`SIE` 的作用的中断使能，我们开启了所有中断。

在代码中注意 `write_csr!` 和 `set_csr!` 的区别，`write_csr!` 采用的是直接写入，也就是覆盖的方法，而 `set_csr!` 则是采用“or”的做法，设置某些位。

### ArchCpu::idle

通过执行wfi指令，将非主cpu设置为低功耗的idle状态。

设置一个特殊的内存页，包含了使得CPU进入低功耗等待状态的指令，从而在系统中未分配任何任务给某些CPU时，可以将它们置于低功耗等待状态，直到发生中断。

```
pub fn idle(&mut self) -> ! {
        extern "C" {
            fn vcpu_arch_entry() -> !;
        }
        assert!(this_cpu_id() == self.cpuid);
        self.init(0, this_cpu_data().id, this_cpu_data().opaque);
        // reset current cpu -> pc = 0x0 (wfi)
        PARKING_MEMORY_SET.call_once(|| {
            let parking_code: [u8; 4] = [0x73, 0x00, 0x50, 0x10]; // 1: wfi; b 1b
            unsafe {
                PARKING_INST_PAGE[..4].copy_from_slice(&parking_code);
            }

            let mut gpm = MemorySet::<Stage2PageTable>::new();
            gpm.insert(MemoryRegion::new_with_offset_mapper(
                0 as GuestPhysAddr,
                unsafe { &PARKING_INST_PAGE as *const _ as HostPhysAddr - PHYS_VIRT_OFFSET },
                PAGE_SIZE,
                MemFlags::READ | MemFlags::WRITE | MemFlags::EXECUTE,
            ))
            .unwrap();
            gpm
        });
        unsafe {
            PARKING_MEMORY_SET.get().unwrap().activate();
            vcpu_arch_entry();
        }
}
```

将该cpu的入口地址设置为0，而0地址将会被映射到 `parking page` ，`parking page` 中设置了一些wfi指令的编码。`wfi`指令使得CPU进入等待状态，直到有中断发生。

然后进入 `vcpu_arch_entry` ，`vcpu_arch_entry` 指向一段汇编代码，功能是根据 `sscratch` 找到 `ArchCpu` 进行上下文的恢复，然后执行 `sret` ，返回到 `spec` 的地址处执行，即执行刚刚设置的wfi指令（**而不是内核代码**），进入低功耗模式。

虽然在这里也进行了一些初始化的工作，但是并没有将cpu的初始化标志 `init` 设置为 `true` ，所以后续真正要唤醒并运行该cpu时，会重新进行初始化（在run方法中体现）。

### ArchCpu::run

该方法主要内容是进行了一些初始化，设置了正确的cpu执行入口，以及修改cpu已初始化的标志。

```
pub fn run(&mut self) -> ! {
        extern "C" {
            fn vcpu_arch_entry() -> !;
        }

        assert!(this_cpu_id() == self.cpuid);
        //change power_on
        this_cpu_data().activate_gpm();
        if !self.init {
            self.init(
                this_cpu_data().cpu_on_entry,
                this_cpu_data().id,
                this_cpu_data().opaque, //dtb_ipa
            );
            self.init = true;
        }

        self.power_on = true;
        info!("CPU{} run@{:#x}", self.cpuid, self.sepc);
        info!("CPU{:#x?}", self);
        unsafe {
            vcpu_arch_entry();
        }
    }
```

可以看到这次的初始化，正确设置了cpu的入口地址，该入口地址指向内核的代码。然后进入 `vcpu_arch_entry` 执行，恢复上下文，**再返回到内核的代码执行**。

### vcpu_arch_entry / VM_ENTRY

这是一段汇编代码，描述的是从hvisor进入vm的时候需要处理的工作。首先就是通过 `sscratch` 寄存器得到原本的 `ArchCpu` 中的上下文信息，再将 `hstatus` 、`sstatus` 、`sepc` 设置成之前我们保存的值，保证返回到vm的时候是VS态、并且从正确位置开始执行。最后恢复通用寄存器的值，并使用 `sret` 返回vm。

```
.macro VM_ENTRY

    csrr   x31, sscratch

    ld   x1, 32*8(x31)
    csrw   hstatus, x1
    ld   x1, 33*8(x31)
    csrw   sstatus, x1
    ld   x1, 34*8(x31)
    csrw   sepc, x1

    ld   x0, 0*8(x31)
    ...
    ld   x31, 31*8(x31)

    sret
    j   .
.endm
```

### VM_EXIT

从vm退出进入hvisor时，也需要保存vm退出时候的相关状态。

首先还是通过 `sscratch` 寄存器得到 `ArchCpu` 的地址，但在这里我们会交换 `sscratch` 和 `x31` 的信息，而不是直接覆盖 `x31` 。然后将除 `x31` 外的通用寄存器的值进行保存，那么 `x31` 的信息现在在 `sscratch` 中，所以我们先把 `x31` 的值保存到 `sp` ，再交换 `x31` 和 `sscratch` ，将 `x31` 的信息通过 `sp` 存到 `ArchCpu` 的对应位置。

再将 `hstatus` 、`sstatus` 、`sepc` 进行保存，当我们在hvisor处理完工作，将要返回vm的时候，需要用到VM_ENTRY的代码，将这三个寄存器的值进行恢复到vm进入hvisor之前的状态，所以在这里我们应该进行保存。

`ld sp, 35*8(sp)` 将 `ArchCpu` 所保存的内核栈的栈顶放到 `sp` 中进行使用，便于我们在hvisor中去使用内核栈。

`csrr a0, sscratch` 这一句，将 `sscratch` 的值放到 `a0` 寄存器中，当我们保存完上下文，跳转到异常处理函数的时候，参数将会通过 `a0` 传递，那么在异常处理的时候就能够访问保存起来的上下文，比如退出码等等。

```
.macro VM_EXIT

    csrrw   x31, sscratch, x31

    sd   x0, 0*8(x31)
    ...
    sd   x30, 30*8(x31)
  
    mv      sp, x31
    csrrw   x31, sscratch, x31
    sd   x31, 31*8(sp)

    csrr    t0, hstatus
    sd   t0, 32*8(sp)
    csrr    t0, sstatus
    sd   t0, 33*8(sp)
    csrr    t0, sepc
    sd   t0, 34*8(sp)

    ld   sp, 35*8(sp)
    csrr      a0, sscratch

.endm
```
