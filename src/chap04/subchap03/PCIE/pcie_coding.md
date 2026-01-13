# pcie的具体实现

```
┌─────────────────────────────────────────────────────────┐
│                    hvisor PCIe 虚拟化                    │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │  ECAM        │  │  DWC PCIe    │  │ LoongArch64  │   │
│  │  Accessor    │  │  Accessor    │  │  Accessor    │   │
│  └──────────────┘  └──────────────┘  └──────────────┘   │
│         │                  │                  │         │
│         └──────────────────┴──────────────────┘         │
│                         │                               │
│             ┌───────────▼───────────┐                   │
│             │  PciConfigAccessor    │                   │
│             │     (Trait)           │                   │
│             └───────────┬───────────┘                   │
│                         │                               │
│  ┌──────────────────────┼──────────────────────┐        │
│  │                      │                      │        │
│  ┌────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ Enumerate  │  │ ConfigSpace  │  │ MMIO Handler │     │
│  └────────────┘  └──────────────┘  └──────────────┘     │
│                                                         │
│  ┌─────────────────────────────────────────────────────┐│
│  │         GLOBAL_PCIE_LIST (global dev list)          ││
│  │  BTreeMap<Bdf, ArcRwLockVirtualPciConfigSpace>      ││
│  └─────────────────────────────────────────────────────┘│
│                                                         │
│  ┌─────────────────────────────────────────────────────┐│
│  │         Zone.vpci_bus (Zone virt bus)               ││
│  │      BTreeMap<Bdf, VirtualPciConfigSpace>           ││
│  └─────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
```
hvisor的pcie系统通过PciConfigAccessor为媒介实现3种不同访问方式的兼容，这里以ecam为例介绍通用的实现。

### bdf重映射
因为linux访问pcie总线是按顺序访问的，如果分配的多个设备之间产生空隙，那么linux有时会认为该位置后没有已经没有其他设备不再继续访问，所以hvisor需要对分给同一zone的设备bdf进行重整。
重整规则如下:
1.如果设备的bus号增加，则vbus号自增1；
2.如果设备的function号不为0且function/vfunction为0的设备不属于zone，则vfunction为0；

### 设备枚举
在hvisor初始化时，会对pcie总线进行遍历，采用和linux类似的方式，确认总线上的设备类型及需要使用的mem资源（主要是bar），在完成设备资源的分配后，hvisor会将该设备添加到全局设备列表中供各个zone使用

### 设备分配
在设备分配之前，hvisor已经将所有的设备信息存储在全局的设备列表中。为了能够正常的让所有zone都能够访问所有可能的设备，hvisor不会将bridge类型的设备从全局设备列表中移除，以便其他zone建立pcie拓扑结构；对于其他类型的设备，当设备被分配给一个zone时，设备会从全局设备列表移除以保证设备功能的正常。

### zone如何访问设备（mmio handler）
当zone访问pcie总线时，对于ecam来说，因为pcie总线的地址直接在cpu域展开，hvisor可以直接通过地址确认zone需要访问的设备bdf以确认设备的唯一身份，以此访问对应设备的配置空间。对于direct模式的pcie虚拟化来说，为了让root分配资源但不驱动设备，对于非root的设备，hvisor替换配置空间中的id和class等字段，让zone无法识别设备类型，使zone无法找到正确的设备驱动，但是设备的其他资源均能够正常访问，以此完成设备资源的分配。

### BAR空间的管理
为了能够尽可能的进行设备直通，提高设备运行效率，hvisor采用二阶段地址映射的方式将bar等资源直通给zone。hvisor会在设备遍历阶段完成对资源的分配，写入设备的物理寄存器，当zone访问bar等资源时，不再修改物理寄存器，而是修改对应zone的页表，将地址映射到正确的资源位置。

### Designware Core PCIE
本节描述了hvisor对于dwc pcie的适配

#### atu基础
对于dwc来说，cpu域与pcie域不直接映射，而是通过atu这一硬件进行访问转换。
单个atu的长度为0x200，其中0-0x100为outbound，0x100-0x200为inbound，分别代表了cpu域到pci域的访问与pci域到cpu域的访问。hvisor只管理outbound访问（从cpu侧访问）。
除了atu0会被复用以外，其他atu都是固定不变的，只在初始化时配置一次。atu0会被cfg0和cfg1复用，如果atu数量不足（在linux中表现为io_shared设置为1，则atu0还会被io类型的mem访问复用）。

#### atu的虚拟化实现
hvisor实际只对atu0进行较为复杂的管理，其他atu基本只会进行一次配置（禁止其他zone的重复配置并在软件查询atu数量时做出响应即可）。
hvisor在创建虚拟总线时为每条总线创建了虚拟atu0，zone对与atu0的访问会被hvisor截取，只有当hvisor发现对对应地址的访问即将进行时，atu0才会被真正写入。
为了避免多个zone交叉访问atu0产生异常，hvisor设置了一个自旋锁以保证访问顺序。

#### 配置空间的访问
因为dwc从cpu侧观察，当产生mmio访问配置空间时，不管访问的是哪个设备，设备地址总是一致的，该地址实际由atu0的pci_addr决定映射到哪一设备。在每次zone访问对应的pci配置空间时，总是会先设置atu0再进行访问，所以hvisor会采用上一次写入atu0的pci_addr来作为需要访问的设备的唯一地址（实际上该地址和bdf可以直接相互转化），并在mmio访问时才真实写入atu0。

#### pcie其他空间的管理
dwc方式访问pcie时，每个bridge会从设备树允许的空间中申请自身及其下设备可以使用的bar空间，并记录为bar6-bar9。hvisor借助root linux对这些空间进行配置以简化自身的逻辑。当root linux配置相关bar寄存器时，hvisor会允许其修改对应的物理寄存器，同时将所有的bar空间写入root linux的页表中；non-root zone对这些空间的访问机制同ecam pcie，通过二阶段地址翻译进行。

### Loongarch64 PCIE
本节描述了hvisor对于loongarch64 pcie的适配

#### 地址访问
loongarch64 pcie与ecam pcie的主要不同在于，loongarch64采用了窗⼝映射的方式，mmio handler的地址必须加上⼀个前缀。hvisor使用`HV_ADDR_PREFIX`和`LOONG_HT_PREFIX`描述该地址前缀。

因为loongarch64部分驱动采用了写死的方式分配资源，所以hvisor使用dircet模式以支持相关资源的正确分配

#### 配置空间地址组成
当hvisor解析zone传入的mmio handler地址时，loongarch64的pcie地址有自己独特的规则，规则如下
```
Type 0 format (Root Bus):
Bits 31-28: Offset[11:8]
Bits 27-16: Reserved (0)
Bits 15-11: Device Number
Bits 10-8:  Function Number
Bits 7-0:   Offset[7:0]

Type 1 format (Other Bus):
Bits 31-28: Offset[11:8]
Bits 27-16: Bus Number
Bits 15-11: Device Number
Bits 10-8:  Function Number
Bits 7-0:   Offset[7:0]
```
