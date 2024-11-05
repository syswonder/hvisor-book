# hvisor的初始化过程

摘要：介绍在qemu上运行hvisor和hvisor初始化过程涉及的相关知识。从qemu启动后开始跟踪整个流程，阅读完本文将对hvisor的初始化过程有一个大概的认识。

## qemu启动流程

qemu模拟的计算机的启动过程：将必要文件加载到内存之后，PC寄存器被初始化为0x1000，从这里开始执行几条指令后就跳转到0x80000000开始执行bootloader（hvsior arm部分使用的是Uboot），执行几条指令后再跳转到uboot可以识别的内核的起始地址执行。

### 生成hvisor的可执行文件

```
rust-objcopy --binary-architecture=aarch64 target/aarch64-unknown-none/debug/hvisor --strip-all -O binary target/aarch64-unknown-none/debug/hvisor.bin.tmp
```

将hvisor的可执行文件转为逻辑二进制，保存为 `hvisor.bin.tmp`。

### 生成uboot可以识别的镜像文件

uboot是一种bootloader，它的主要任务是跳转到hvisor镜像的第一条指令开始执行，所以要保证生成的hvisor镜像是uboot可以识别的，这里需要使用 `mkimage`工具。

```
mkimage -n hvisor_img -A arm64 -O linux -C none -T kernel -a 0x40400000 -e 0x40400000 -d target/aarch64-unknown-none/debug/hvisor.bin.tmp target/aarch64-unknown-none/debug/hvisor.bin
```

* `-n hvisor_img`：指定内核镜像的名称。
* `-A arm64`：指定架构为 ARM64。
* `-O linux`：指定操作系统为 Linux。
* `-C none`：不使用压缩算法。
* `-T kernel`：指定类型为内核。
* `-a 0x40400000`：指定加载地址为 `0x40400000`。
* `-e 0x40400000`：指定入口地址为 `0x40400000`。
* `-d target/aarch64-unknown-none/debug/hvisor.bin.tmp`：指定输入文件为之前生成的临时二进制文件。
* 最后一个参数是生成的输出文件名，即最终的内核镜像文件 `hvisor.bin`。

## 初始化过程

### aarch64.ld链接脚本

要知道hvisor是如何执行的，我们首先查看链接脚本 `aarch64.ld`，这样能对hvisor的执行流程有一个大体认识。

```
ENTRY(arch_entry)
BASE_ADDRESS = 0x40400000;
```

第一行设置了程序入口 `arch_entry` ，这个入口可以在 `arch/aarch64/entry.rs` 中找到，稍后介绍。

```
.text : {
        *(.text.entry)
        *(.text .text.*)
    }
```

我们将 `.text` 段作为最开头的段，且把包含了入口第一条指令的 `.text.entry` 放在 `.text` 段的开头，这样就保证了hvisor确实会从和qemu约定的0x40400000处开始执行。

这里我们还需要记住一个东西叫 `__core_end` , 它是链接脚本的结束位置的地址，等一下启动过程中可以知道它的作用。

### arch_entry

有了上面这些前提，我们可以走进hvisor的第一条指令了，也就是 `arch_entry()` 。

```
// src/arch/aarch64/entry.rs

pub unsafe extern "C" fn arch_entry() -> i32 {
    unsafe {
        core::arch::asm!(
            "
            // x0 = dtbaddr
            mov x1, x0
            mrs x0, mpidr_el1
            and x0, x0, #0xff
            ldr x2, =__core_end          // x2 = &__core_end
            mov x3, {per_cpu_size}      // x3 = per_cpu_size
            madd x4, x0, x3, x3       // x4 = cpuid * per_cpu_size + per_cpu_size
            add x5, x2, x4
            mov sp, x5           // sp = &__core_end + (cpuid + 1) * per_cpu_size
            b {rust_main}             // x0 = cpuid, x1 = dtbaddr
            ",
            options(noreturn),
            per_cpu_size=const PER_CPU_SIZE,
            rust_main = sym crate::rust_main,
        );
    }
}
```

先看内嵌汇编部分。第一句指令 `mov x1,x0` ，将 `x0` 寄存器的值传入 `x1` 寄存器，这里x0中存的是设备树的地址。qemu模拟一台arm架构的计算机，这个计算机中同样有着各种各样的设备，比如鼠标显示屏这种输入输出设备，以及各种存储设备，当我们想要从键盘获取输入、往显示屏输出，都要从某个地方获取输入，或者把输出的数据放到某个地方，在计算机中我们用特定地址来访问。设备树中就保存了这些设备的访问地址，hypervisor作为所有软件的总管，自然要知道设备树的信息，那么Uboot在进入内核之前就会把这些信息放在 `x0`中，这是一种约定。

`mrs x0, mpidr_el1`中，mrs是一个访问系统级别寄存器的指令，也就是把系统寄存器 `mpidr_el1` 的内容送到  `x0` 中，`mpidr_el1` 中存的信息是我们目前在和哪个CPU打交道（计算机支持多核CPU），后续有很多和CPU的配合工作，所以要知道现在在用哪个CPU。这个寄存器包含了很多关于CPU的信息，我们目前要用的是低8位，取出CPU对应的id，也就是 `and x0, x0, #0xff` 这一句在干的事。

`ldr x2, = __core_end` ，在链接脚本的末尾处我们设置了一个符号 `__core_end` ，作为hvisor整个程序空间的结束地址，把这个地址放到 `x2` 中。

`mov x3,{per_cpu_size}` 将每个cpu的栈的大小放进 `x3` 中，这个 `{xxx}` 就是把 `xxx` 这个外部定义的值替换到汇编代码里，可以看到下面的 `per_cpu_size=const PER_CPU_SIZE` 把外部的变量换了个名字作为参数传了进来。而另一个参数中的 `sym` 表示后面跟着的是一个符号，在其他地方被定义。

`per_cpu_size` 在这个大小的空间内，可以进行相关寄存器的保存和恢复，还包含了cpu的栈空间。

`madd x4, x0, x3, x3` 是一个乘加指令，cpu_id * per_cpu_size + per_cpu_size，结果放入 `x4` ，此时 `x4` 存着的是目前的cpu数量需要的空间是多大。（序号从0开始，所以多加一次per_cpu_size）。

`add x5,x2,x4` 的意思是把hvisor的结束地址加上CPU需要的全部空间放到 `x5` 中。

`mov sp,x5` 是找到目前cpu的栈顶。

`b {rust_main}` 代表着跳到 `rust_main` 开始执行，这也说明了这段汇编代码不会返回，与 `option(noreturn)`相对应。

## 进入rust_main()

### fn rust_main(cpuid:usize, host_dtb:usize)

进入 `rust_main`需要两个参数，这两个参数是通过 `x0` 和 `x1` 传递的，还记得前面的entry中，我们的 `x0` 存放的是cpu_id，`x1` 存放的是设备树的相关信息。

### install_trap_vector()

当处理器遇到异常或者中断的时候，就要跳转去相应的位置进行处理，这里就是在设置这些相应的跳转地址（可以视为在设置一张表），用于处理在Hypervisor级别的异常。每个特权级都有自己对应的一张异常向量表，除了EL0，应用程序的特权级，它必须跳转到其他特权级处理异常。`VBAR_ELn` 寄存器用于存储ELn这个特权级下的异常向量表的基地址。

```
extern "C" {
    fn _hyp_trap_vector();
}

pub fn install_trap_vector() {
    // Set the trap vector.
    VBAR_EL2.set(_hyp_trap_vector as _)
}

```

 `VBAR_EL2.set()` 将 `_hyp_trap_vector()` 的地址设置为EL2特权级的异常向量表的基地址。

`_hyp_trap_vector()` 这段汇编代码就是在构建异常向量表。

**异常向量表格式的简单介绍**

根据发生异常的等级和处理异常的等级是否相同分为两类，如果等级不变，则按照是否使用当前等级的SP分为两组，如果异常等级改变，则按照执行模式是64位/32位分为两组，至此异常向量表被划分为4组。在每一组中，每个表项代表一种异常处理情况的入口。

### 主CPU

```
static MASTER_CPU: AtomicI32 = AtomicI32::new(-1);

let mut is_primary = false;
    if MASTER_CPU.load(Ordering::Acquire) == -1 {
        MASTER_CPU.store(cpuid as i32, Ordering::Release);
        is_primary = true;
        println!("Hello, HVISOR!");
        #[cfg(target_arch = "riscv64")]
        clear_bss();
    }
```

`static MASTER_CPU: AtomicI32` 中，`AtomicI32` 表示这是一种原子类型，表示对他的操作要么成功要么失败，不会出现中间状态，可以保证多线程环境下的安全访问，总之它就是一个很安全的 `i32` 类型。

`MASSTER_CPU.load()` 是进行读操作的一个方法，参数 `Ordering::Acquire` 表示，如果在我进行读之前有一些写操作，那么需要等这些写操作按顺序进行完了，我再读。**总之，这个参数保证了数据被正确更改后再进行读取。**

如果读出来是-1，和定义时候的一样，代表主CPU还没有被设置，就把 `cpu_id` 设为主CPU。同样的，`Ordering::Release` 的作用肯定也是指修改之前要保证所有其他的修改都完成了。

### CPU的通用数据结构：PerCpu

hvisor支持不同的架构，合理的系统设计应该让不同的架构使用统一的接口，便于描述各部分的工作。`PerCpu` 就是这样一个通用的CPU描述。

```
pub struct PerCpu {
    pub id: usize,
    pub cpu_on_entry: usize,
    pub arch_cpu: ArchCpu,
    pub zone: Option<Arc<RwLock<Zone>>>,
    pub ctrl_lock: Mutex<()>,
    pub boot_cpu: bool,
    // percpu stack
}
```

对于 `PerCpu` 的各个字段：

- `id` : CPU的序号
- `cpu_on_entry` ：CPU进入EL1也就是guest的时候第一条指令的地址，只有当这个CPU是boot CPU时，才会被置为有效值，初始化的时候我们设置为一个访问不到的地址。
- `arch_cpu` ：与架构相关的CPU描述，行为是由 `PerCpu` 发起，具体的执行者是 `arch_cpu` 。
  - `cpu_id`
  - `psci_on` : cpu是否启动
- `zone` ：zone其实就代表一个guestOS，对于同一个guestOS可能有多个cpu在为他服务
- `ctrl_lock` ：为并发安全性而设置。
- `boot_cpu` ：对于一个guestOS，区分为他服务的CPU的主核/次核，`boot_cpu` 即表示当前CPU是否是某个guest的主核。

### 主核唤醒其他核

```
if is_primary {
        wakeup_secondary_cpus(cpu.id, host_dtb);
}

fn wakeup_secondary_cpus(this_id: usize, host_dtb: usize) {
    for cpu_id in 0..MAX_CPU_NUM {
        if cpu_id == this_id {
            continue;
        }
        cpu_start(cpu_id, arch_entry as _, host_dtb);
    }
}

pub fn cpu_start(cpuid: usize, start_addr: usize, opaque: usize) {
    psci::cpu_on(cpuid as u64 | 0x80000000, start_addr as _, opaque as _).unwrap_or_else(|err| {
        if let psci::error::Error::AlreadyOn = err {
        } else {
            panic!("can't wake up cpu {}", cpuid);
        }
    });
}
```

如果当前CPU是主CPU，就由当前CPU来唤醒其他的次核，次核执行 `cpu_start` ，在 `cpu_start` 中，`cpu_on` 实际上调用了 `call64`中的SMC指令，陷入EL3来执行唤醒CPU的动作。

那么从 `cpu_on` 的声明中我们大概可以猜测它的功能，唤醒一个CPU，这个CPU将要从 `arch_entry` 这个地方开始执行。这是因为多核处理器之间会进行通信协作，那么就必须保证CPU的一致性，所以以相同的入口开始执行，为保持同步，应该保证每个CPU都运行到某个状态，那么可以由接下来的几句代码来验证。

```
    ENTERED_CPUS.fetch_add(1, Ordering::SeqCst);
    wait_for(|| PerCpu::entered_cpus() < MAX_CPU_NUM as _);
    assert_eq!(PerCpu::entered_cpus(), MAX_CPU_NUM as _);
```

其中 `ENTERED_CPUS.fetch_add(1, Ordering::SeqCst)` 代表按照顺序一致性增加 `ENTERED_CPUS` 的值，那么每个CPU执行一次后，这个 `assert_eq` 宏应该可以顺利通过。

### 主核还需要干的事primary_init_early（）

**初始化日志**

1. 全局的日志记录器的创建
2. 日志级别过滤器的设置，设置日志级别过滤器的主要作用是决定哪些日志消息应该被记录和输出。

**初始化堆空间和页表**

1. 在.bss段申请了一段空间作为堆空间，设置好分配器
2. 设置页帧分配器

**解析设备树的信息**

根据 `rust_main` 参数中的设备树地址堆设备树的信息进行解析。

**创建GIC实例**

实例化一个全局的静态变量GIC，是通用中断控制器的一个实例。

**初始化hvisor的页表**

这个页表只是针对hypervisor自身VA转为PA的实现。（以内核和应用的关系来理解）

**为每个VM创建zone**

```
zone_create(zone_id, TENANTS[zone_id] as _, DTB_IPA);

zone_create(vmid: usize, dtb_ptr: *const u8, dtb_ipa: usize) -> Arc<RwLock<Zone>>
```

zone其实就代表一个guestVM，里面包含了某个guestVM可能会用到的各种信息。观察函数的参数，`dtb_ptr`，是hypervisor想让这个guest看到的设备的信息的地址，可以在 `images/aarch64/devicetree` 中看到。而 `dtb_ipa` 的作用是，每个guest都会从cpu的 `x0` 寄存器获取这个地址去寻找设备树的信息，所以需要在构建stage2页表的过程中保证这个ipa会映射到这个guest的dtb地址。以这样的方式告诉guest，它运行在一个什么样的机器上，物理内存起始地址是多少，cpu有几个等等。

```
let guest_fdt = unsafe { fdt::Fdt::from_ptr(dtb_ptr) }.unwrap();
    let guest_entry = guest_fdt
        .memory()
        .regions()
        .next()
        .unwrap()
        .starting_address as usize;
```

上面这段内容，通过解析设备树信息，得到了 `guest_entry`，它的含义是这个guest可以看到的物理地址的起始地址，在qemu的启动参数中，我们也可以看到某个guest镜像被加载到内存的哪个地方，这两个值是相等的。

接下来会根据 `dtb` 的信息，构建该guest的stage-2页表、MMIO映射、IRQ位图。

```
guest_fdt.cpus().for_each(|cpu| {
        let cpu_id = cpu.ids().all().next().unwrap();
        zone.cpu_set.set_bit(cpu_id as usize);
});

pub fn set_bit(&mut self, id: usize) {
    assert!(id <= self.max_cpu_id);
    self.bitmap |= 1 << id;
}
```

上面这段代码是根据dtb中给出的CPU信息，将分配给这个zone的cpu的id记录在位图中。

```
let new_zone_pointer = Arc::new(RwLock::new(zone));
    {
        cpu_set.iter().for_each(|cpuid| {
            let cpu_data = get_cpu_data(cpuid);
            cpu_data.zone = Some(new_zone_pointer.clone());
            //chose boot cpu
            if cpuid == cpu_set.first_cpu().unwrap() {
                cpu_data.boot_cpu = true;
            }
            cpu_data.cpu_on_entry = guest_entry;
        });
    }
  
```

上面这段代码完成的任务是：遍历给这个zone分配的cpu，获取该cpu的 `PerCpu` 可变引用，修改其中的zone成员变量，并且将第一个分配给这个zone的cpu标记为 `boot_cpu`。并且，设置这个zone的主cpu进入guest以后的第一条指令的地址 `guest_entry`。

上面主核CPU要做的事情告一段落，以  `INIT_EARLY_OK.store(1, Ordering::Release)` 作为标记，而其他CPU在主核完成之前，只能进行等待 `wait_for_counter(&INIT_EARLY_OK, 1)`。

### 地址空间初始化

上个部分提到的IPA和PA其实是地址空间的内容，具体的内容将在内存管理的文档中给出，这里做一个简要介绍。

如果不考虑Hypervisor，guestVM作为一个内核，会进行内存管理的工作，也就是应用程序的虚拟地址VA到内核的PA的过程，那么这里的PA，就是真正的内存物理地址。

在考虑Hypervisor的情况下，Hypervisor作为一个内核的角色也同样会做内存管理的工作，只是这时候的应用程序就变成了guestVM，而guestVM是不会意识到Hypervisor的存在的（否则需要更改guestVM的设计，这不符合我们提高性能的初衷）。我们将guestVM中的PA叫做IPA或者GPA，因为它不是最终的物理地址，而是Hypervisor让guestVM看到的中间物理地址，所以整个系统中存在着两套内存管理机制，guestVM管理的VA到IPA的转换，以及Hypervisor管理的从IPA到PA的转换。

### run_vm()

终于到了即将启动guestVM的时刻了。

**将非boot_cpu设置为空闲状态**

对于非boot_cpu，将其设置为空闲状态并等待唤醒。在 `idle` 函数中实现。

```
// arch/aarch64/cpu.rs
// impl ArchCpu > idle

while !self.psci_on {
            _lock = None;
            while !self.psci_on {}
            _lock = Some(cpu_data.ctrl_lock.lock());
        }
```

**启用stage2页表机制**

```
VTTBR_EL2.set_baddr(root_paddr as _);
```

将VTTBR_EL2寄存器的值设置为guestVM对应的satge2页表的根页表起始地址。

**cpu_reset()**

把CPU的状态设置为即将进入guestVM的状态。

```
pub fn reset(&mut self, entry: usize, _cpu_id: usize, dtb: usize) {
        ELR_EL2.set(entry as _);
        SPSR_EL2.set(0x3c5);
        let regs = self.guest_reg();
        regs.clear();
        regs.usr[0] = dtb as _; // dtb addr
        self.reset_vm_regs();
        self.activate_vmm();
    }
```

当cpu从EL1陷入EL2处理结束后，就会返回到 `ELR_EL2` 保存的地址处执行，所以我们将 `ELR_EL2` 的地址设置为对应的guest的第一条指令的地址。

当异常被捕获到EL2中进行处理的时候，原来的进程的状态会被保存在 `SPSR_EL2` 中，这里我们设置为0x3c5。`SPSR_EL2`的低4位记录了返回哪个异常等级，设置为0101，代表要返回EL1h。

还记得之前我们设置了在 `__core_end` 之后设置了各个cpu的空间，在 `guest_reg()` 中，将这段空间用起来了，分配了32个通用寄存器的空间。

并且在 `usr[0]`中保存了 `dtb_ipa` 的值，当eret返回EL1的时候，处理器会将这部分上下文恢复到相应的寄存器中，当guest初启的时候就能够通过 `x0` 获取这个ipa，从而得到设备树的信息，这部分内容与上面初始化stage2页表相对应。

**reset_vm_regs() activate_vmm()**

进行一些寄存器的配置。

**将psci_on设置为true**

标记该cpu已启动进入工作状态。

**vm_return()**

```
vmreturn(self.guest_reg() as *mut _ as usize);

pub unsafe extern "C" fn vmreturn(_gu_regs: usize) -> ! {
    core::arch::asm!(
        "
        /* x0: guest registers */
        mov	sp, x0
        ldp	x1, x0, [sp], #16	/* x1 is the exit_reason */
        ldp	x1, x2, [sp], #16
        ldp	x3, x4, [sp], #16
        ldp	x5, x6, [sp], #16
        ldp	x7, x8, [sp], #16
        ldp	x9, x10, [sp], #16
        ldp	x11, x12, [sp], #16
        ldp	x13, x14, [sp], #16
        ldp	x15, x16, [sp], #16
        ldp	x17, x18, [sp], #16
        ldp	x19, x20, [sp], #16
        ldp	x21, x22, [sp], #16
        ldp	x23, x24, [sp], #16
        ldp	x25, x26, [sp], #16
        ldp	x27, x28, [sp], #16
        ldp	x29, x30, [sp], #16
        /*now el2 sp point to per cpu stack top*/
        eret                            //ret to el2_entry hvc #0 now,depend on ELR_EL2
  
    ",
        options(noreturn),
    );
}
```

可以看到这部分的内容主要是对我们刚才保存的上下文进行恢复，并且返回到虚拟机执行。

将栈顶设置为 `x0`，在调用这个函数的时候通过 `x0` 传入一个参数 `_gu_regs` ，这个参数其实就是寄存器上下文的起始地址。这样我们就可以通过 `sp` 对各个寄存器进行恢复。

`ldp` 是arm架构下的一个加载指令，`ldp x1,x0,[sp]` 代表从 `sp` 这个地址处，加载两个64位的值到 `x1` 和 `x0` 中。并且会自动将 `sp` 的值+16，也就是两个寄存器的大小。这里没有按照 `x0,x1` 的原因是，我们将 `exit` 相关的信息，放在了寄存器上下文的开头，而它的下一个才是 `x0` 。

完成上下文的恢复以后，`sp` 的值就增加了32*8的大小，指向了percpu区域的末尾。

最后我们执行 `eret` 语句，此时cpu从当前特权级EL2的 `ELR_EL2` 中取出返回地址，并且通过 `SPSR_EL2` 知道了他要返回到EL1特权级。还记得我们在设计 `percpu` 的时候，对于boot-cpu，我们将我们在qemu启动参数中写好的内存被放置的地址，设置为cpu返回后执行的第一条指令的地址，所以返回EL1后，cpu就会从内核的第一条指令开始执行。

*至此，读者应该对hvisor的大致启动过程以及设计模块有了大致理解。*
