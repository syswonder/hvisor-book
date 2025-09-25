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

要知道hvisor是如何执行的，我们首先查看链接脚本 `platform/aarch64/qemu-gicv3/linker.ld`。链接脚本（Linker Script）是连接编译器与硬件的关键桥梁，由 GNU `ld` 工具使用，用于控制目标文件的段（section）如何被映射到输出可执行文件的内存布局中，决定了程序在内存中如何布局。它定义了程序入口点、各段的加载地址与运行地址、内存区域的排列顺序
特殊符号的定义（如 hvisor中的 `__core_end` ）。

```
ENTRY(arch_entry)
BASE_ADDRESS = 0x40400000;
```

第一行设置了程序入口函数为 `arch_entry` ，这个入口可以在 `arch/aarch64/entry.rs` 中找到，稍后介绍。第二行定义了程序的基地址为0x40400000。

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

```rust
// src/arch/aarch64/entry.rs

#[naked]
#[no_mangle]
#[link_section = ".text.entry"]
pub unsafe extern "C" fn arch_entry() -> ! {
    unsafe {
        core::arch::asm!(
            "
            // x0 = dtbaddr
            mov x18, x0

            /* Insert nop instruction to ensure byte at offset 10 in hvisor binary is non-zero.
            * Rockchip U-Boot (arch_preboot_os@arch/arm/mach-rockchip/board.c:670) performs 
            * forced relocation if this byte is zero, causing boot failure. This padding
            * prevents unintended relocation by maintaining non-zero value at this critical
            * offset in the binary layout. */

            nop
            nop
            bl {boot_cpuid_get}        // x17 = cpuid

            adrp x2, __core_end        // x2 = &__core_end
            mov x3, {per_cpu_size}     // x3 = per_cpu_size
            madd x4, x17, x3, x3       // x4 = cpuid * per_cpu_size
            add x5, x2, x4
            mov sp, x5                 // sp = &__core_end + (cpuid + 1) * per_cpu_size

            // disable cache and MMU
            mrs x1, sctlr_el2
            bic x1, x1, #0xf
            msr sctlr_el2, x1

            // cache_invalidate(0): clear dl1$
            mov x0, #0
            bl  {cache_invalidate}

            ic  iallu

            cmp x17, 0
            b.ne 1f

            // if (cpu_id == 0) cache_invalidate(2): clear l2$
            mov x0, #2
            bl  {cache_invalidate}

            // ic  iallu

            bl {clear_bss}
            bl {boot_pt_init}
        1:
            bl {mmu_enable}

            mov x1, x18
            mov x0, x17
            mov x18, #0
            mov x17, #0
            bl {rust_main}            // x0 = cpuid, x1 = dtbaddr
            ",
            options(noreturn),
            boot_cpuid_get = sym boot_cpuid_get,
            cache_invalidate = sym cache_invalidate,
            per_cpu_size = const PER_CPU_SIZE,
            rust_main = sym crate::rust_main,
            clear_bss = sym crate::clear_bss,
            boot_pt_init = sym super::mmu::boot_pt_init,
            mmu_enable = sym super::mmu::mmu_enable,
        );
    }
}
```

`arch_entry`函数有三条属性（Attributes），
- `#[naked]`告诉编译器生成一个“裸函数”，编译器不会自动插入函数开始时的“序言”（prologue，如保存寄存器、调整栈帧）。
- `#[no_mangle]`表示禁用符号名称修饰，使得链接器（ld）能通过这个名称找到它。
- `#[link_section = ".text.entry"]` 指定该函数应被放入名为 .text.entry 的代码段（section），正好与之前`linker.ld`中对应。

再看内嵌汇编部分。第一句指令 `mov x18, x0` ，将 `x0` 寄存器的值传入 `x18` 寄存器，这里x0中存的是设备树的地址。qemu模拟一台arm架构的计算机，这个计算机中同样有着各种各样的设备，比如鼠标显示屏这种输入输出设备，以及各种存储设备，当我们想要从键盘获取输入、往显示屏输出，都要从某个地方获取输入，或者把输出的数据放到某个地方，在计算机中我们用特定地址来访问。设备树中就保存了这些设备的访问地址，hypervisor作为所有软件的总管，自然要知道设备树的信息，那么Uboot在进入内核之前就会把这些信息放在 `x0`中，这是一种约定。

第二、三条指令为`nop`，因为某些 Rockchip 平台的 U-Boot 在加载镜像前会检查二进制文件第10个字节，如果该字节为 0，U-Boot 会认为这是一个“未重定位”的镜像，强行进行重定位（relocation），导致 Hypervisor 被错误地移动到错误地址，启动失败。通过插入 nop 指令（机器码非零），确保第10个字节不是 0，从而绕过 U-Boot 的误判。


`bl {boot_cpuid_get}` 指令调用 boot_cpuid_get 函数获取当前 CPU 核心的 ID（cpuid），后续用于初始化 per-CPU 数据、判断是否为主核（cpuid == 0）。

```
adrp x2, __core_end        // x2 = &__core_end
mov x3, {per_cpu_size}     // x3 = per_cpu_size
madd x4, x17, x3, x3       // x4 = (cpuid + 1) * per_cpu_size
add x5, x2, x4
mov sp, x5                 // sp = __core_end + (cpuid + 1) * per_cpu_size
```

这些指令负责在多核处理器上为每个 CPU 核心（CPU core）设置独立的栈空间，每个 CPU 核心都需要自己的 运行时栈（stack），用于保存函数调用的返回地址、局部变量、寄存器备份等。
如果多个核心共享同一个栈，会造成数据冲突和崩溃。`__core_end`来自链接脚本`linker.ld`，作为hvisor整个程序空间的结束地址。`adrp x2, __core_end`将符号 `__core_end` 所在的 4KB 页面基地址 加载到 x2 寄存器中，`mov x3, {per_cpu_size}` 将常量 PER_CPU_SIZE 加载到 x3 寄存器中。`madd x4, x17, x3, x3` 是一个乘加指令，将 `(cpuid + 1) * per_cpu_size` 存入 `x4` 寄存器， `+1` 可以避免 CPU 0 的栈从 `__core_end` 正好开始，防止与前面的数据段（.bss）紧贴，留出安全间隙。`add x5, x2, x4` 将“基地址”和“偏移量”相加，得到最终的栈顶地址，将计算出的栈顶地址写入栈指针寄存器sp。

```
mrs x1, sctlr_el2
bic x1, x1, #0xf
msr sctlr_el2, x1
```

这些指令负责关闭缓存与 MMU ，确保在重新设置页表和缓存策略前，当前的 MMU、数据缓存（D-Cache）、指令缓存（I-Cache）全部关闭，避免旧状态干扰新配置。mrs是一个访问系统级别寄存器的指令，也就是把系统寄存器 `mpidr_el1` 的内容送到  `x1` 中。`mrs x1, sctlr_el2`读取 EL2 系统控制寄存器 `SCTLR_EL2` 的值到通用寄存器 `x1`。`bic x1, x1, #0xf` 相当于`x1 = x1 & ~0xf`，即清除 x1 的低 4 位，也就相当于下面这段代码：

```
sctlr_el2.M = 0;  // 关闭 MMU
sctlr_el2.A = 0;  // 关闭对齐检查（避免未对齐访问异常）
sctlr_el2.C = 0;  // 关闭数据缓存
sctlr_el2.I = 0;  // 关闭指令缓存
```

`msr sctlr_el2, x1`将修改后的值写回 SCTLR_EL2，立即生效。

```
mov x0, #0
bl  {cache_invalidate}

ic  iallu
```

这段代码负责清除当前 CPU 核心的 L1 数据缓存（L1 D-Cache） 和 L1 指令缓存（L1 I-Cache）。首先`mov x0, #0`设置参数 `x0 = 0`，表示目标是 L1 缓存，再通过 `bl  {cache_invalidate}` 调用汇编编写的 `cache_invalidate(cache_level: usize)` 函数。`ic iallu`指令使当前核心的整个 L1 指令缓存无效。之前的`SCTLR_EL2.I=0` 只是“禁用”了 I-Cache，但缓存中可能仍有旧指令，必须显式 invalidate 才能清除。

```
cmp x17, 0
b.ne 1f

// if (cpu_id == 0) cache_invalidate(2): clear l2$
mov x0, #2
bl  {cache_invalidate}
```

这段代码用于主核清理 L2 缓存，也就是尽管有多个 CPU 但是总共只会执行一次，L2 缓存是多个 CPU 核心共享的，如果每个核心都清理一次 L2，会造成重复和潜在竞争。

```asm
bl {clear_bss}
bl {boot_pt_init}
1:
bl {mmu_enable}
```

这段代码首先调用 `clear_bss` 函数，将 `.bss` 段清零，`.bss` 段存放未初始化的全局变量，再调用`boot_pt_init`函数初始化启动页表（Boot Page Table），为即将开启的 MMU 准备虚拟内存映射。`boot_pt_init`函数会读取平台定义的物理内存区域列表 `BOARD_PHYSMEM_LIST`，检查其合法性（对齐、排序），根据这个列表创建多级页表（L0 和 L1，因为只是“启动页表”，不需要精细控制，所以用大块映射即可），实现从虚拟地址到物理地址的映射，目前的映射是恒等映射，也就是虚拟地址完全等于物理地址。

接着调用`mmu_enable`函数，通过`MAIR`配置内存属性、设置页表基址（TTBR0）、设置页表控制（TCR）、启用 MMU、数据缓存（D-Cache）、指令缓存（I-Cache） ，让 CPU 开始使用虚拟地址，地址翻译开始生效，此刻起，所有地址都变为虚拟地址。

```asm
mov x1, x18
mov x0, x17
mov x18, #0
mov x17, #0
bl {rust_main}            // x0 = cpuid, x1 = dtbaddr
```

最后一步跳转到 Rust 主函数`fn rust_main(cpuid: usize, host_dtb: usize)`开始执行，这也说明了这段汇编代码不会返回，与 `option(noreturn)`相对应。

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

`_hyp_trap_vector()` 这段汇编代码就是在构建异常向量表。汇编代码的具体定义位于`src/arch/aarch64/trap.S`。

**异常向量表格式的简单介绍**

根据发生异常的等级和处理异常的等级是否相同分为两类，如果等级不变，则按照是否使用当前等级的SP分为两组，如果异常等级改变，则按照执行模式是64位/32位分为两组，至此异常向量表被划分为4组。在每一组中，每个表项代表一种异常处理情况的入口。

### 主CPU

```
static MASTER_CPU: AtomicI32 = AtomicI32::new(-1);

let mut is_primary = false;
if MASTER_CPU.load(Ordering::Acquire) == -1 {
    MASTER_CPU.store(cpuid as i32, Ordering::Release);
    is_primary = true;
    memory::heap::init();
    memory::heap::test();
}
```

`static MASTER_CPU: AtomicI32` 中，`AtomicI32` 表示这是一种原子类型，表示对他的操作要么成功要么失败，不会出现中间状态，可以保证多线程环境下的安全访问，总之它就是一个很安全的 `i32` 类型。

`MASSTER_CPU.load()` 是进行读操作的一个方法，参数 `Ordering::Acquire` 表示，如果在我进行读之前有一些写操作，那么需要等这些写操作按顺序进行完了，我再读。**总之，这个参数保证了数据被正确更改后再进行读取。**

如果读出来是-1，和定义时候的一样，代表主CPU还没有被设置，就把 `cpu_id` 设为主CPU。同样的，`Ordering::Release` 的作用肯定也是指修改之前要保证所有其他的修改都完成了。

也就是说，这段代码实现的功能是，第一个进入此函数的 CPU（即 cpuid == 0 的主核）会发现 MASTER_CPU == -1，于是会设置自己为 `MASTER_CPU`，标记 `is_primary = true`，初始化全局堆内存（heap）。堆的作用是支持动态内存分配如Box, Vec, Arc 等，用于创建虚拟机、页表、设备模型等对象。

### CPU的通用数据结构：PerCpu

hvisor支持不同的架构，合理的系统设计应该让不同的架构使用统一的接口，便于描述各部分的工作。`PerCpu` 就是这样一个通用的CPU描述，它为系统中的每一个 CPU 核心维护一份独立的私有数据。在多核系统中，每个 CPU 都有自己的栈、状态、当前运行的虚拟机等，不能共享这些数据，否则会引发竞态，所以需要一个机制，也就是给每个 CPU 分配一块专属内存区域。

```rust
#[repr(C)]
pub struct PerCpu {
    pub id: usize,
    pub cpu_on_entry: usize,
    pub dtb_ipa: usize,
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
- `dtb_ipa`：Guest VM 使用的设备树在客户机视角下的物理地址（IPA，中间物理地址）
- `arch_cpu` ：与架构相关的CPU描述，行为是由 `PerCpu` 发起，具体的执行者是 `arch_cpu` 。
  - `cpu_id`
  - `psci_on` : cpu是否启动
- `zone` ：zone其实就代表一个guestOS，对于同一个guestOS可能有多个cpu在为他服务
- `ctrl_lock` ：为并发安全性而设置。
- `boot_cpu` ：对于一个guestOS，区分为他服务的CPU的主核/次核，`boot_cpu` 即表示当前CPU是否是某个guest的主核。

### 主核唤醒其他核

```rust
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
    let new_cpuid = {
        if cpuid >= MAX_CPU_NUM {
            panic!("Invalid cpuid: {}", cpuid);
        }
        BOARD_MPIDR_MAPPINGS[cpuid]
    };
    psci::cpu_on(new_cpuid, start_addr as _, opaque as _).unwrap_or_else(|err| {
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

zone其实就代表一个guestVM，是虚拟机（Virtual Machine）的运行时管理结构体，里面包含了某个guestVM可能会用到的各种信息。

```rust
pub struct Zone {
    pub name: [u8; CONFIG_NAME_MAXLEN],     // 虚拟机名称
    pub id: usize,                          // 虚拟机 ID
    pub mmio: Vec<MMIOConfig>,              // MMIO 区域及其处理函数
    pub cpu_num: usize,                     // 分配给该 Zone 的 CPU 数量
    pub cpu_set: CpuSet,                    // 分配给该 Zone 的 CPU 集合
    pub irq_bitmap: [u32; 1024 / 32],       // 中断位图（支持最多 1024 个中断）
    pub gpm: MemorySet<Stage2PageTable>,    // 客户机物理内存映射（S2PT）
    pub pciroot: PciRoot,                   // PCI 总线根设备
    pub is_err: bool,                       // 是否发生错误
}
```

```rust
let zone = zone_create(root_zone_config()).unwrap();
add_zone(zone);
```

`zone_create` 会接收一个`HvZoneConfig`类型的参数，这是一个只读的、编译时/启动时确定的虚拟机配置结构体，用于在创建 Zone（虚拟机）时传递所有必要的初始化参数,其核心作用是将静态的`HvZoneConfig`配置转换为动态的Zone运行时结构，完成虚拟机资源的分配和初始化。HvZoneConfig描述了虚拟机所需的CPU、内存、中断、PCI设备等资源，而Zone则是这些资源在运行时的具体实现。

```rust
pub struct HvZoneConfig {
    pub zone_id: u32,                         // 虚拟机唯一 ID（0 通常是 root VM）
    cpus: u64,                                // CPU 位图（bitmask），表示分配哪些 CPU 核心（如 0b101 = CPU0 和 CPU2）
    num_memory_regions: u32,                  // 内存区域数量
    memory_regions: [HvConfigMemoryRegion; CONFIG_MAX_MEMORY_REGIONS], // 客户机物理内存布局（GPA 映射）
    num_interrupts: u32,                      // 分配的中断数量
    interrupts: [u32; CONFIG_MAX_INTERRUPTS], // 属于该 VM 的中断 ID 列表（如 GIC IRQ）
    num_ivc_configs: u32,                     // IVC（Inter-VM Communication）通道数量
    ivc_configs: [HvIvcConfig; CONFIG_MAX_IVC_CONFIGS], // IVC 通道配置（跨虚拟机通信）
    pub entry_point: u64,                     // 客户机启动入口地址（GPA）
    pub kernel_load_paddr: u64,               // 内核镜像加载的客户机物理地址
    pub kernel_size: u64,                     // 内核大小（若为 INVALID_ADDRESS 表示不指定）
    pub dtb_load_paddr: u64,                  // 设备树（DTB）加载的客户机物理地址
    pub dtb_size: u64,                        // DTB 大小（若为 INVALID_ADDRESS 表示不指定）
    pub name: [u8; CONFIG_NAME_MAXLEN],       // 虚拟机名称
    pub arch_config: HvArchZoneConfig,        // 架构特定配置
    pub pci_config: HvPciConfig,              // PCI 总线配置
    pub num_pci_devs: u64,                    // 分配的虚拟 PCI 设备数量
    pub alloc_pci_devs: [u64; CONFIG_MAX_PCI_DEV], // 分配的 PCI 设备 BDF（Bus:Device:Function）列表
}

pub struct HvConfigMemoryRegion {
    pub mem_type: u32,           // RAM / IO / VIRTIO
    pub physical_start: u64,     // GPA 起始地址
    pub virtual_start: u64,      // IPA（客户机视角的虚拟地址）
    pub size: u64,
}

pub struct HvIvcConfig {
    pub ivc_id: u32,              // 通信通道 ID
    pub peer_id: u32,             // 对端 Zone ID
    pub control_table_ipa: u64,   // 控制表的客户机虚拟地址
    pub shared_mem_ipa: u64,      // 共享内存的客户机虚拟地址
    pub rw_sec_size: u32,         // 读写段大小
    pub out_sec_size: u32,        // 输出段大小
    pub interrupt_num: u32,       // 通知中断号
    pub max_peers: u32,           // 最大对端数
}

pub struct HvPciConfig {
    pub ecam_base: u64,          // ECAM 基地址（GPA）
    pub ecam_size: u64,          // ECAM 大小
    pub num_buses: u32,          // PCI 总线数量
}

pub struct Zone {
    pub name: [u8; CONFIG_NAME_MAXLEN],
    pub id: usize,
    pub mmio: Vec<MMIOConfig>,           // 管理 Zone 的内存映射 I/O
    pub cpu_num: usize,                  // 记录分配给该 Zone 的 CPU 核心数量
    pub cpu_set: CpuSet,                 // 位图，记录哪些 CPU 核心被分配给了该 Zone
    pub irq_bitmap: [u32; 1024 / 32],    // 中断位图，用于记录该 Zone 拥有的中断号（IRQ）
    pub gpm: MemorySet<Stage2PageTable>, // 用于虚拟化内存管理的二级页表
    pub pciroot: PciRoot,                // 管理该 Zone 的 PCI（Peripheral Component Interconnect）总线和设备。
    pub is_err: bool,                    // 表示该 Zone 是否处于错误状态
}
```

`zone_create`函数在hisor Hypervisor中具有多种应用场景：

- 创建Root Zone：当zone_id为0时，创建特权虚拟机(通常为宿主机操作系统)，拥有完整的硬件访问权限。
- 创建普通虚拟机：当zone_id大于0时，创建非特权虚拟机，其资源访问受到Hypervisor的严格限制。

下面详细讲解一下该函数的运行流程：

```rust
let mut zone = Zone::new(zone_id, &config.name);
zone.pt_init(config.memory_regions()).unwrap();
zone.mmio_init(&config.arch_config);
zone.arch_zone_configuration(config)?;
zone.pci_init(
    &config.pci_config,
    config.num_pci_devs as _,
    &config.alloc_pci_devs,
);
```

首先基于配置创建新的`Zone`结构体，调用`pt_init`方法初始化Stage2页表，将`HvConfigMemoryRegion`数组转换为虚拟机的内存布局，建立GPA到 HPA 的映射关系。然后调用`mmio_init`初始化架构相关的 MMIO 设备。再调用`arch_zone_configuration`进行架构的特定配置。再调用`pci_init`初始化虚拟 PCI 总线。

```rust
let mut cpu_num = 0;
for cpu_id in config.cpus().iter() {
    if let Some(zone) = get_cpu_data(*cpu_id as _).zone.clone() {
        return hv_result_err!(
            EBUSY,
            format!(
                "Failed to create zone: cpu {} already belongs to zone {}",
                cpu_id,
                zone.read().id
            )
        );
    }
    zone.cpu_set.set_bit(*cpu_id as _);
    cpu_num += 1;
}
zone.cpu_num = cpu_num;
```

这段代码遍历配置中指定的 CPU 列表（cpus()）、检查每个 CPU 是否已被其他 Zone 占用，若已绑定，则返回 EBUSY 错误，成功则将其加入当前 Zone 的 `cpu_set`，最终设置 `cpu_num` 为分配的 CPU 数量

```rust
zone.virqc_init(config);

zone.irq_bitmap_init(config.interrupts());

let mut dtb_ipa = INVALID_ADDRESS as u64;
for region in config.memory_regions() {
    // region contains config.dtb_load_paddr?
    if region.physical_start <= config.dtb_load_paddr
        && region.physical_start + region.size > config.dtb_load_paddr
    {
        dtb_ipa = region.virtual_start + config.dtb_load_paddr - region.physical_start;
    }
}
```
`virqc_init`函数初始化虚拟中断控制器（Virtual IRQ Controller），`irq_bitmap_init`根据配置中的中断列表（如串口、网卡使用的 IRQ 号），更新 irq_bitmap，设置哪些中断号是这个 Zone “拥有”的。

接着计算 DTB 的 IPA 地址（设备树位置），客户机操作系统启动时需要加载设备树（Device Tree Blob, DTB），dtb_load_paddr 是 DTB 在客户机视角下的物理地址（GPA），但由于 GPA 和 IPA 可能有偏移（因为映射到了不同的 HPA），我们需要知道“当客户机访问这个 GPA 时，它实际上对应哪个 IPA？”，所以这里通过遍历内存区域，找到包含 dtb_load_paddr 的 region， 然后根据 virtual_start - physical_start 的偏移量，算出对应的 IPA。最终结果存入 dtb_ipa，后面会传递给每个 CPU 上下文。

```rust

info!("zone cpu_set: {:#b}", zone.cpu_set.bitmap);
let cpu_set = zone.cpu_set;

let new_zone_pointer = Arc::new(RwLock::new(zone));
{
    cpu_set.iter().for_each(|cpuid| {
        let cpu_data = get_cpu_data(cpuid);
        cpu_data.zone = Some(new_zone_pointer.clone());
        //chose boot cpu
        if cpuid == cpu_set.first_cpu().unwrap() {
            cpu_data.boot_cpu = true;
        }
        cpu_data.cpu_on_entry = config.entry_point as _;
        cpu_data.dtb_ipa = dtb_ipa as _;
        #[cfg(target_arch = "aarch64")]
        {
            cpu_data.arch_cpu.is_aarch32 = config.arch_config.is_aarch32 != 0;
        }
    });
}

Ok(new_zone_pointer)
```

这段代码是最关键的一步，让每个物理 CPU 知道自己要运行哪个 Zone。`cpu_data.zone` 指向当前 CPU 要运行的 Zone，`boot_cpu`标记是否为主引导 CPU，`cpu_on_entry`表示当 CPU 被唤醒时跳转的入口地址，`dtb_ipa`表示设备树在 IPA 空间的位置，供客户机读取硬件信息，`is_aarch32`为架构标志，决定运行 32 位还是 64 位模式。

最后代码返回一个引用计数的可读写锁包裹的 Zone 实例，后续可以通过这个句柄进行`vcpu_run`、查询状态、动态添加设备、销毁 Zone。

上面主核CPU要做的事情告一段落，以  `INIT_EARLY_OK.store(1, Ordering::Release)` 作为标记，而其他CPU在主核完成之前，只能进行等待 `wait_for_counter(&INIT_EARLY_OK, 1)`。

### 地址空间初始化

上个部分提到的IPA和PA其实是地址空间的内容，具体的内容将在内存管理的文档中给出，这里做一个简要介绍。

如果不考虑Hypervisor，guestVM作为一个内核，会进行内存管理的工作，也就是应用程序的虚拟地址VA到内核的PA的过程，那么这里的PA，就是真正的内存物理地址。

在考虑Hypervisor的情况下，Hypervisor作为一个内核的角色也同样会做内存管理的工作，只是这时候的应用程序就变成了guestVM，而guestVM是不会意识到Hypervisor的存在的（否则需要更改guestVM的设计，这不符合我们提高性能的初衷）。我们将guestVM中的PA叫做IPA或者GPA，因为它不是最终的物理地址，而是Hypervisor让guestVM看到的中间物理地址，所以整个系统中存在着两套内存管理机制，guestVM管理的VA到IPA的转换，以及Hypervisor管理的从IPA到PA的转换。

### run_vm()

在 `zone_create` 中，我们完成了虚拟机的资源分配和配置，但还没有真正运行它。现在终于到了即将启动guestVM的时刻了。

```rust
// percpu.rs
// impl PerCpu > run_vm

pub fn run_vm(&mut self) {
    if !self.boot_cpu {
        info!("CPU{}: Idling the CPU before starting VM...", self.id);
        self.arch_cpu.idle();
    }
    info!("CPU{}: Running the VM...", self.id);
    self.arch_cpu.run();
}
```

真正的启动发生在 `run_vm` 函数中，而它最终会调用到 `ArchCpu` 的两个核心方法：

`run()`：主函数，用于启动或恢复虚拟机执行
`idle()`：备用函数，用于非启动 CPU 的“待机”状态

**将非boot_cpu设置为空闲状态**

对于非boot_cpu，将其设置为空闲状态并等待唤醒。在 `idle` 函数中实现。

**核心CPU的启动**
```rust
pub fn run(&mut self) -> ! {
    assert!(this_cpu_id() == self.cpuid);
    this_cpu_data().activate_gpm();
    self.reset(this_cpu_data().cpu_on_entry, this_cpu_data().dtb_ipa);
    if self.is_aarch32 {
        info!("cpu {} is aarch32", self.cpuid);
        HCR_EL2.write(...); // 切换为 AArch32 模式
        SPSR_EL2.set(0x1D3); // 返回 Supervisor 模式
    }
    self.power_on = true;
    info!("cpu {} started at {:#x?}", self.cpuid, this_cpu_data().cpu_on_entry);
    unsafe {
        vmreturn(self.guest_reg() as *mut _ as usize);
    }
}
```

首先确保当前运行的物理 CPU 与 ArchCpu 实例匹配，防止跨 CPU 错误调用。接着调用`activate_gpm`激活当前 CPU 的 Stage 2 页表，实际上是调用 gpm.activate()，它会将 `VTTBR_EL2` 寄存器设置为当前 Zone 的 Stage 2 页表基地址，从而启用虚拟机的内存视图（即 GPA → HPA 映射）。

`self.reset(entry, dtb)`函数会重置 vCPU 的寄存器状态，为进入客户机做准备，具体来说，它会设置`ELR_EL2`（Exception Link Register at EL2）寄存器为客户机的入口点，当从 EL2 返回时（执行 eret），CPU 会跳转到 ELR_EL2 指向的地址，另外还会设置 `SPSR_EL2`（Saved Program Status Register） 寄存器，定义了返回后处理器的状态为禁止调试、异步中止、IRQ、FIQ，并且返回到 EL1（内核态），使用 SP_EL1 堆栈。接着初始化客户机的通用寄存器，`x0` 被设为设备树（DTB）的物理地址，这是 Linux 内核启动所需参数。然后回调用`activate_vmm`函数设置 Stage 2 页表控制寄存器。

`vmreturn`是最关键的一步，这是一个汇编函数：
```rust
#[naked]
#[no_mangle]
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
    )
}
```

可以看到`vmreturn`这部分的内容主要是对我们刚才保存的上下文进行恢复，并且返回到虚拟机执行。

将栈顶设置为 `x0`，在调用这个函数的时候通过 `x0` 传入一个参数 `_gu_regs` ，这个参数其实就是寄存器上下文的起始地址。这样我们就可以通过 `sp` 对各个寄存器进行恢复。

`ldp` 是arm架构下的一个加载指令，`ldp x1,x0,[sp]` 代表从 `sp` 这个地址处，加载两个64位的值到 `x1` 和 `x0` 中。并且会自动将 `sp` 的值+16，也就是两个寄存器的大小。这里没有按照 `x0,x1` 的原因是，我们将 `exit` 相关的信息，放在了寄存器上下文的开头，而它的下一个才是 `x0` 。

完成上下文的恢复以后，`sp` 的值就增加了32*8的大小，指向了percpu区域的末尾。

最后我们执行 `eret` 语句，此时cpu从当前特权级EL2的 `ELR_EL2` 中取出返回地址，并且通过 `SPSR_EL2` 知道了他要返回到EL1特权级。还记得我们在设计 `percpu` 的时候，对于boot-cpu，我们将我们在qemu启动参数中写好的内存被放置的地址，设置为cpu返回后执行的第一条指令的地址，所以返回EL1后，cpu就会从内核的第一条指令开始执行。

*至此，读者应该对hvisor的大致启动过程以及设计模块有了大致理解。*
