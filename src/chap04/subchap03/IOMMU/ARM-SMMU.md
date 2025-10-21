# ARM-SMMU技术文档

摘要：介绍ARM-SMMU的开发过程。

## 背景知识

简单介绍SMMMU的原理与作用。

### DMA是什么？为什么需要IOMMU？

运行在hvisor之上的虚拟机，需要和设备进行交互，但如果每次都等待CPU来主持这类工作，会使处理效率降低，那么就出现了DMA机制。**DMA 是一种允许设备直接与内存交换数据而不需要 CPU 参与的机制。**

那么我们可以大致得出虚拟机通过DMA和设备交互的过程，首先虚拟机发出DMA请求，告诉目标设备把数据写到哪个地方，然后设备根据地址写入内存。

但上述过程需要考虑一些问题：

- hvisor对每个虚拟机都做了内存虚拟化，所以虚拟机发出的DMA请求的目标内存地址是GPA，在这里也叫IOVA，需要将这个地址转为真正的PA，才能写入到物理内存的正确位置。
- 再者，如果不对IOVA的范围加以限制，那么代表着可以通过DMA机制访问任何一个内存地址，从而造成无法预见的严重后果。

所以我们需要一个既能帮我们做地址转换，又能保证操作地址的合法性的机构，就像MMU内存管理单元一样，这个机构就叫**IOMMU**，在Arm架构中它有另一个名字叫**SMMU**（后续都称SMMU）**。**

现在你知道了，SMMU可以将虚拟地址转为物理地址，从而保证设备直接访问内存的合法性。

### SMMU具体要做的工作

上面说到SMMU的功能类似MMU，MMU的作用对象是虚拟机或者应用程序，而SMMU的作用对象是每个设备，每个设备以sid作为标识，对应的表叫做**stream table**。该表以设备的sid作为索引，PCI设备的sid可以从BDF号获取：sid = (B << 5) | (D << 3) | F。

## 开发工作

目前我们在Qemu中实现了对SMMUv3的stage-2地址转换支持，创建简单的线性表，并且使用PCI设备进行了简单的验证。

IOMMU的工作还未并入主线，可以切换到IOMMU分支查看。

### 整体思路

我们将PCI HOST直通给zone0，即在提供给zone0的设备树中加上PCI节点，将对应的内存地址在zone0的第二阶段页表中做好映射，并确保中断注入正常。那么zone0就会自己去探测PCI设备并进行配置，而我们在hvisor中只需要做好SMMU的配置工作就好。

### Qemu参数

在 `machine` 中添加 `iommu=smmuv3` 以开启SMMUv3支持，并在 `global` 中添加 `arm-smmuv3.stage=2` 开启第二阶段地址翻译。

注意在Qemu中尚不支持嵌套翻译，如果不指出 `stage=2` ，则默认只支持第一阶段地址翻译，请使用Qemu-8.1以上版本，低版本中不支持开启第二阶段地址翻译。

添加PCI设备时请注意开启 `iommu_platform=on` 。

addr可以指定该设备的bdf号。

**在Qemu模拟的PCI总线中，除了PCI HOST，还有一个默认的网卡设备，所以其余添加的设备的addr参数必须从2.0开始。**

```makefile
// scripts/qemu-aarch64.mk

QEMU_ARGS := -machine virt,secure=on,gic-version=3,virtualization=on,iommu=smmuv3
QEMU_ARGS += -global arm-smmuv3.stage=2

QEMU_ARGS += -device virtio-blk-pci,drive=Xa003e000,disable-legacy=on,disable-modern=off,iommu_platform=on,addr=2.0
```

### 在hvisor的页表中映射SMMU相关的内存

查阅Qemu的源码可知VIRT_SMMU对应的内存区域起始地址为0x09050000，大小为0x20000，我们需要访问这个区域，所以在hvisor的页表中必须进行映射。

```rust
// src/arch/aarch64/mm.rs

pub fn init_hv_page_table(fdt: &fdt::Fdt) -> HvResult {
    hv_pt.insert(MemoryRegion::new_with_offset_mapper(
        smmuv3_base(),
        smmuv3_base(),
        smmuv3_size(),
        MemFlags::READ | MemFlags::WRITE,
    ))?;
}
```

### SMMUv3数据结构

该结构包含了将会访问的SMMUv3的内存区域的引用，是否支持二级表，sid的最大位数，以及stream table的基地址和分配的页帧。

其中的rp是所定义的 `RegisterPage` 的引用，`RegisterPage` 根据SMMUv3手册中第六章的偏移量进行设置，读者可自行查阅。

```rust
// src/arch/aarch64/iommu.rs

pub struct Smmuv3{
    rp:&'static RegisterPage,

    strtab_2lvl:bool,
    sid_max_bits:usize,

    frames:Vec<Frame>,

    // strtab
    strtab_base:usize,

    // about queues...
}
```

### new()

在完成映射工作后，我们便可以引用对应的这段寄存器区域。

```rust
impl Smmuv3{
    fn new() -> Self{
        let rp = unsafe {
            &*(SMMU_BASE_ADDR as *const RegisterPage)
        };

        let mut r = Self{
            ...
        };

        r.check_env();

        r.init_structures();

        r.device_reset();

        r
    }
}
```

### check_env()

检查当前环境支持哪个阶段的地址转换、支持什么类型的流表、支持多少位的sid等信息。

以其中检查环境支持哪种表格式为例，支持的表的类型在 `IDR0` 寄存器中，通过 `self.rp.IDR0.get() as usize` 获取 `IDR0` 的数值，通过 `extract_bit` 进行截取，获取 `ST_LEVEL` 字段的值，根据手册可知，0b00代表支持线性表，0b01代表支持线性表和二级表，0b1x为保留位，我们可以根据该信息选择创建什么类型的流表。

```rust
impl Smmuv3{
    fn check_env(&mut self){
        let idr0 = self.rp.IDR0.get() as usize;

        info!("Smmuv3 IDR0:{:b}",idr0);

        // supported types of stream tables.
        let stb_support = extract_bits(idr0, IDR0_ST_LEVEL_OFF, IDR0_ST_LEVEL_LEN);
        match stb_support{
            0 => info!("Smmuv3 Linear Stream Table Supported."),
            1 => {info!("Smmuv3 2-level Stream Table Supoorted.");
                self.strtab_2lvl = true;
            }
            _ => info!("Smmuv3 don't support any stream table."),
        }

	...
    }
}
```

### init_linear_strtab()

我们需要支持第二阶段地址转换，且系统中的设备并不多，所以我们选择用线性表。

在申请线性表需要的空间时，我们应该根据当前的sid的最多位数得到表项的个数，乘上每个表项需要的空间 `STRTAB_STE_SIZE` ，进而知道需要申请多少个页帧。但SMMUv3对stream table的起始地址有着严格的要求，起始地址的低 (5+sid_max_bits) 位必须为0。

由于当前的hvisor中暂不支持这样申请空间，我们在确保安全的情况下，申请一段空间，并在这段空间里面选定一个符合条件的地址作为表基址，虽然这样会造成一些空间的浪费。

申请了空间后，我们可以将这个表的基地址填入 `STRTAB_BASE` 这个寄存器：

```rust
	let mut base = extract_bits(self.strtab_base, STRTAB_BASE_OFF, STRTAB_BASE_LEN);
	base = base << STRTAB_BASE_OFF;
	base |= STRTAB_BASE_RA;
	self.rp.STRTAB_BASE.set(base as _);
```

接着我们还要设置 `STRTAB_BASE_CFG` 寄存器，来表明我们使用的表的格式是线性表或者二级表，以及表项的个数（使用以LOG2的形式表示，即SID的最大位数）：

```rust
        // format : linear table
        cfg |= STRTAB_BASE_CFG_FMT_LINEAR << STRTAB_BASE_CFG_FMT_OFF;

        // table size : log2(entries)
        // entry_num = 2^(sid_bits)
        // log2(size) = sid_bits
        cfg |= self.sid_max_bits << STRTAB_BASE_CFG_LOG2SIZE_OFF;

        // linear table -> ignore SPLIT field
        self.rp.STRTAB_BASE_CFG.set(cfg as _);
```

### init_bypass_ste(sid:usize)

当前我们还未配置任何相关信息，需要先将所有表项置为默认状态。

对于每个sid，根据表基址找到表项的地址，即合法位为0，地址翻译设置为 `BYPASS`。

```rust
	let base = self.strtab_base + sid * STRTAB_STE_SIZE;
	let tab = unsafe{&mut *(base as *mut [u64;STRTAB_STE_DWORDS])};

	let mut val:usize = 0;
	val |= STRTAB_STE_0_V;
	val |= STRTAB_STE_0_CFG_BYPASS << STRTAB_STE_0_CFG_OFF;
```

### device_reset()

上面我们做了一些准备工作，但还需要一些额外的配置，比如使能SMMU，否则会导致SMMU处于disabled状态。

```rust
	let cr0 = CR0_SMMUEN;
	self.rp.CR0.set(cr0 as _);
```

### write_ste(sid:usize,vmid:usize,root_pt:usize)

该方法用于配置具体设备的信息。

首先我们同样要根据sid找到对应的表项地址。

```rust
	let base = self.strtab_base + sid * STRTAB_STE_SIZE;
        let tab = unsafe{&mut *(base as *mut [u64;STRTAB_STE_DWORDS])};
```

第二步我们要指明，这个设备的相关信息，我们是要用来进行第二阶段地址翻译的，且这个表项是合法的了。

```rust
        let mut val0:usize = 0;
        val0 |= STRTAB_STE_0_V;
        val0 |= STRTAB_STE_0_CFG_S2_TRANS << STRTAB_STE_0_CFG_OFF;
```

第三步我们要说明当前这个设备分配给哪个虚拟机来用，并且开启第二阶段页表遍历，`S2AA64` 代表第二阶段翻译表是基于aarch64的，`S2R` 代表启用错误记录。

```rust
        let mut val2:usize = 0;
        val2 |= vmid << STRTAB_STE_2_S2VMID_OFF;
        val2 |= STRTAB_STE_2_S2PTW;
        val2 |= STRTAB_STE_2_S2AA64;
        val2 |= STRTAB_STE_2_S2R;
```

最后一步就是要指出第二阶段翻译的依据，就是在hvisor中的对应虚拟机的页表，只需要将页表基地址填入对应位置即可，即 `S2TTB` 这个字段。

这里我们也需要说明这个页表的配置信息，这样SMMU才知道这个页表的格式等信息，才能使用这个页表，即 `VTCR` 这个字段。

```rust
	let vtcr = 20 + (2<<6) + (1<<8) + (1<<10) + (3<<12) + (0<<14) + (4<<16);
        let v = extract_bits(vtcr as _, 0, STRTAB_STE_2_VTCR_LEN);
        val2 |= v << STRTAB_STE_2_VTCR_OFF;

        let vttbr = extract_bits(root_pt, STRTAB_STE_3_S2TTB_OFF, STRTAB_STE_3_S2TTB_LEN);
```

## 初始化以及设备分配

在 `src/main.rs` 中，进行了hvisor的页表初始化以后（映射了SMMU相关区域），可以进行SMMU的初始化。

```rust
fn primary_init_early(dtb: usize) {
    ...

    crate::arch::mm::init_hv_page_table(&host_fdt).unwrap();

    info!("Primary CPU init hv page table OK.");

    iommu_init();

    zone_create(0,ROOT_ENTRY,ROOT_ZONE_DTB_ADDR as _, DTB_IPA).unwrap();
    INIT_EARLY_OK.store(1, Ordering::Release);
}
```

接着需要分配设备，这一步我们在创建虚拟机的时候同步完成，目前我们只将设备分配给zone0使用。

```rust
// src/zone.rs

pub fn zone_create(
    zone_id: usize,
    guest_entry: usize,
    dtb_ptr: *const u8,
    dtb_ipa: usize,
) -> HvResult<Arc<RwLock<Zone>>> {
    ...

    if zone_id==0{
        // add_device(0, 0x8, zone.gpm.root_paddr());
        iommu_add_device(zone_id, BLK_PCI_ID, zone.gpm.root_paddr());
    }
  
    ...
}
```

## 简单验证

在qemu启动参数中开启 `-trace smmuv3_*` ，即可看到相关输出：

```bash
smmuv3_config_cache_hit Config cache HIT for sid=0x10 (hits=1469, misses=1, hit rate=99)
smmuv3_translate_success smmuv3-iommu-memory-region-16-2 sid=0x10 iova=0x8e043242 translated=0x8e043242 perm=0x3
```

### 注意事项

在QEMU模拟的aarch64机器中，默认存在 `virtio-net-pci` 设备，但你必须手动添加参数，使其经过IOMMU，就像下面这样：

```makefile
QEMU_ARGS += -netdev type=user,id=net1
QEMU_ARGS += -device virtio-net-pci,netdev=net1,disable-legacy=on,disable-modern=off,iommu_platform=on
```

当然你也可以在GPA和HPA非恒等映射的情况下，测试IOMMU是否能够正常工作。但你需要把root根文件系统挂载的 `virtio-blk-device` 换为 `virtio-blk-pci`，因为普通的MMIO设备在QEMU中不经过IOMMU，这会导致DMA失败，进而导致root虚拟机无法启动。

一个参考的设置方式如下：

```makefile
QEMU_ARGS += -drive if=none,file=$(FSIMG1),id=Xa003e000,format=raw
# QEMU_ARGS += -device virtio-blk-device,drive=Xa003e000,bus=virtio-mmio-bus.31
QEMU_ARGS +=-device virtio-blk-pci,drive=Xa003e000,disable-legacy=on,disable-modern=off,iommu_platform=on
```