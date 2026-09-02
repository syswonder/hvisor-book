
# PerCPU 定义

hvisor 为每个物理 CPU 维护一份独立的 per-CPU 数据（PerCpu），存放该 CPU 的虚拟化上下文、所运行的虚拟机（zone）信息与同步状态。per-CPU 数据由一个核心问题驱动设计：**CPU 如何快速、安全地访问"属于自己"的那一份数据**。本章描述当前的设计。

## 内存布局

所有 per-CPU 数据区（slot）从静态链接区结束处（`__core_end`）开始连续排布，每个 slot 大小固定为 `PER_CPU_SIZE`（512 KiB），共 `MAX_CPU_NUM`（即 `BOARD_NCPUS`）个：

```
slot 基址 = PER_CPU_ARRAY_PTR + cpuid × PER_CPU_SIZE
           = __core_end        + cpuid × 512 KiB
```

每个 slot 的低地址存放 `PerCpu` 结构体，其上方是该 CPU 的栈：各架构在启动早期（进入 `rust_main` 前）按 `slot 基址 + PER_CPU_SIZE` 设置栈指针。per-CPU 区域之后是 hvisor 的内存池。

## PerCpu 结构体

```rust
#[repr(C)]
pub struct PerCpu {
    pub id: usize,            // 逻辑 CPU id（0..BOARD_NCPUS），刻意放在偏移 0
    pub cpu_on_entry: usize,  // 唤醒本 CPU 时写入的入口地址，初始为 INVALID_ADDRESS
    pub dtb_ipa: usize,       // 启动时传给本 CPU 的设备树地址
    pub vcpu_state: VcpuStateCell, // vCPU 生命周期状态（原子包装）
    pub arch_cpu: ArchCpu,    // 架构相关上下文（guest 寄存器、VMCS、trap 上下文等）
    pub zone: Option<Arc<Zone>>,   // 本 CPU 所属的 zone
    pub ctrl_lock: Mutex<()>,      // 跨 CPU 访问本 slot 时的同步锁
    pub boot_cpu: bool,            // 是否为引导 CPU
    pub self_ptr: usize,           // slot 基址的内存副本（x86_64 使用，见下文）
}
```

- `vcpu_state` 用原子单元（`VcpuStateCell`）包装，取值 `Stopped` / `Ready` / `Running` / `Blocked`，表示本物理 CPU 上 vCPU 的调度状态，供其它 CPU 远程修改（如通过事件唤醒）。
- 结构体以 `#[repr(C)]` 声明，且 `id` 是**第一个字段**：四种架构都约定"slot 偏移 0 处就是逻辑 CPU id"，本 CPU 的 id 可以直接从缓存指针处读回（x86_64 上就是一条 `gs:[0]` 载入）。

`PerCpu::new(cpu_id)` 由每个物理 CPU 在启动早期调用一次：它计算出自己的 slot 地址并就地写入上述结构，然后把 slot 基址缓存到架构寄存器中（见下节）。`run_vm` / `entered_cpus` / `activate_gpm` 等方法围绕 slot 提供"运行虚拟机 / 统计已进入的 CPU 数 / 激活 zone 页表"等操作。

## 本 CPU 数据的快速访问：架构寄存器缓存

每个物理 CPU 的 slot 基址在启动后不再变化，因此不必在每次访问时重新推导。hvisor 把 slot 基址缓存在各架构的寄存器中，写入点只有一个——该 CPU 自身的 `PerCpu::new`：

| 架构 | 缓存寄存器 | 说明 |
|---|---|---|
| aarch64 | `TPIDR_EL2` | EL2 私有寄存器，guest 运行在 EL1，无法读写；VM 进出不改变其值 |
| riscv64 | `CSR_SSCRATCH` | HS 级 CSR；缓存的是 `&PerCpu.arch_cpu`（见下文 trap.S 约定） |
| x86_64 | `IA32_GS_BASE` | 由 VMCS host-state 在每次 VM exit 时自动重载，host 态 gs 相对寻址恒指向本 CPU slot |
| loongarch64 | 根 CSR `SAVE0` | 根特权态寄存器；SAVE3/SAVE4 保留给 trap 交接，guest 的 SAVE 值在 GCSR 文件中，互不影响 |

各架构通过统一的 `set_this_cpu_pointer(slot_base)` / `this_cpu_pointer()` 两个接口读写缓存，具体语义因架构而异：

- **aarch64**：直接缓存 slot 基址。读取时一条 `mrs` 取回基址，再按需访问结构体字段。
- **riscv64**：入口汇编 `trap.S` 通过 `csrrw` 交换 `x31` 与 `sscratch` 来访问 guest 寄存器，因此 `sscratch` 必须恒指向 `&PerCpu.arch_cpu`（guest 寄存器位于 `ArchCpu` 偏移 0）。`set_this_cpu_pointer` 在写入时自动加上 `arch_cpu` 字段偏移，调用方只需给出 slot 基址；读回时用 `container_of`（指针减同一偏移）还原 slot 基址。vCPU 复位时 `reset_regs` 按同一值重写。
- **x86_64**：slot 基址写入 `IA32_GS_BASE` MSR。`setup_vmcs_host` 在每次 run/idle 时把当前 MSR 值快照进 VMCS host-state 字段，硬件在每次 VM exit 自动重载，因此 host 态代码的 gs 相对寻址总是落在本 CPU 的 slot 上；guest 的 GS_BASE 是 VMCS guest-state 中的独立字段，语义不变。由于 `rdmsr` 较慢，`PerCpu::new` 同时把基址存一份在结构体的 `self_ptr` 字段，读回时用一条 gs 相对载入（`gs:[offset_of!(self_ptr)]`）即可，避免读 MSR。
- **loongarch64**：根 CSR `SAVE0`（0x30）当前空闲，直接缓存 slot 基址；`SAVE3`/`SAVE4` 留给异常交接（运行/空闲时写入 trap 上下文与栈指针），guest 的 `SAVE0` 属于独立的 GCSR 文件，由 trap 代码经 `gcsrrd`/`gcsrwr` 维护。

缓存就绪后，本 CPU 的 per-CPU 访问退化为"一次寄存器读 + 一次轻量访存"，不再涉及 CPU id 查询、查表或扫描。

## 访问接口

hvisor 的 per-CPU 访问分为两类：

| 接口 | 用途 | 开销 |
|---|---|---|
| `this_cpu_data()` / `this_cpu_id()` | 访问**当前 CPU** 自己的 slot / 逻辑 id | 寄存器缓存直接读回，常量级 |
| `get_cpu_data(cpu_id)` | 按逻辑 id 访问**任意 CPU** 的 slot（发送事件、修改远端 vCPU 状态、唤醒等） | `基址 + id × PER_CPU_SIZE` 直接寻址 |

- `this_cpu_data()` 返回 `&'static mut PerCpu`，等价于把 `this_cpu_pointer()` 当作结构体指针解引用。
- `this_cpu_id()` 在 aarch64/x86_64/loongarch64 上就是"读回 slot 偏移 0 的 `id` 字段"；riscv64 读 `ArchCpu` 中缓存的 cpuid，二者含义相同（同一 CPU 的逻辑 id）。
- `get_cpu_data()` 不经过任何缓存寄存器，可用于任意 CPU 上下文（包括中断处理中对其它 CPU 的定向操作）。

**使用前提**：`this_cpu_*` 系列只在当前 CPU 完成 `PerCpu::new`（即缓存寄存器写入）之后有效。引导流程保证了这一点，详见下一节。

## 初始化时序与安全不变量

"寄存器缓存先写后读"由以下引导时序保证，是全局不变式：

1. **日志路径**：日志系统在 `primary_init_early` 中安装，而所有 CPU 只有先通过 `PerCpu::new` 并越过 `ENTERED_CPUS` 关卡后才会进入该函数——因此任何经日志器输出的记录（每条记录都会读取 `this_cpu_data().id`）都发生在缓存就绪之后。`println!` 直接输出到串口，不经过日志器，可用于更早的阶段。
2. **中断路径**：各架构使能中断（GIC CPU 接口、LAPIC、全局中断使能）都发生在 irqchip 初始化阶段，晚于所有 CPU 的 `PerCpu::new`，因此中断处理函数执行时缓存必然已写入。
3. **riscv64**：`sscratch` 必须保持指向 `&arch_cpu` 的 trap.S 约定由 `set_this_cpu_pointer` 的内部偏移换算保证，不依赖调用方。
4. **x86_64**：VMCS host-state 的 GS_BASE 快照在每次 run/idle 时由该 VMCS 所属的 pCPU 重新取样，VMCS 与 pCPU 一一对应。
5. **loongarch64**：`SAVE0` 不与异常交接（专用 `SAVE3`/`SAVE4`）和 guest GCSR 状态冲突。

## 按目标 CPU 索引的 per-CPU 结构

部分 per-CPU 结构只按**目标 CPU 的逻辑 id**被访问（发往某 CPU 的事件队列、guest 软件中断位图等），无需借助 per-CPU 寄存器或链接段机制。这类结构直接定义为以 `MAX_CPU_NUM` 为长度的静态数组，每个元素以 `repr(align(64))` 对齐，避免相邻 CPU 的条目共享缓存行：

- `src/event.rs` 的 `PERCPU_EVENTS: [EventQueue; MAX_CPU_NUM]`：各 CPU 的 IPI/事件队列（`VecDeque<usize>` + 互斥锁）。
- `src/device/irqchip/ls7a2000/mod.rs` 的 `GUEST_HWI_ASSERTED`：各 pCPU 的 guest 软件中断（HWI）位图。

访问时以目标 CPU 的逻辑 id 为下标（如 `&PERCPU_EVENTS[cpu].0`）取对应条目再加锁操作；本 CPU 取自己的队列时同样按自己的 id 索引。此类结构与 `PerCpu` slot 一起构成完整的 per-CPU 数据视图。
