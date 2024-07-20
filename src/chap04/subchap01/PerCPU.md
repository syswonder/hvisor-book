
# PerCPU结构体

在hvisor的架构中，PerCpu结构体扮演着核心角色，用于实现每个CPU核心的本地状态管理以及支持CPU虚拟化。下面是对PerCpu结构体及相关函数的详细介绍：

## PerCpu结构体定义

PerCpu结构体被设计为每个CPU核心存储其特定数据和状态的容器。它的布局如下：

```
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

各字段定义如下：

```
    id: CPU核心的标识符。
    cpu_on_entry: 一个用于追踪CPU进入状态的地址，初始化为INVALID_ADDRESS，表示无效地址。
    dtb_ipa: 设备树二进制的物理地址，同样初始化为INVALID_ADDRESS。
    arch_cpu: 一个指向ArchCpu类型的引用，ArchCpu包含特定于架构的CPU信息和功能。
    zone: 一个可选的Arc<RwLock<Zone>>类型，表示当前CPU核心正在运行的虚拟机（zone）。
    ctrl_lock: 一个互斥锁，用于控制访问和同步PerCpu的数据。
    boot_cpu: 一个布尔值，指示是否为引导CPU。
```

## PerCpu的构造和操作

```
    PerCpu::new: 此函数创建并初始化PerCpu结构体。它首先计算结构体的虚拟地址，然后安全地写入初始化数据。对于RISC-V架构，还会更新CSR_SSCRATCH寄存器来存储ArchCpu的指针。
    run_vm: 当调用此方法时，如果当前CPU不是引导CPU，则会先将其置于空闲状态，然后再运行虚拟机。
    entered_cpus: 返回已进入虚拟机运行状态的CPU核心数。
    activate_gpm: 激活所关联zone的GPM（Guest Page Management）。
```

## 获取PerCpu实例

```
    get_cpu_data: 提供基于CPU ID获取PerCpu实例的方法。
    this_cpu_data: 返回当前执行CPU的PerCpu实例。
```

