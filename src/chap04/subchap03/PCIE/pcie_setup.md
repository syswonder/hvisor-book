# pcie配置

### 总线配置
不论是哪种访问pcie的方式，都需要配置ROOT_PCI_CONFIG
对于arm64/loongarch64/riscv来说，其值可以从设备树中直接获取
对于x86来说，通常从acpi mcfg表中获取信息
这里以从dts中获取信息为例
```
pub const ROOT_PCI_CONFIG: &[HvPciConfig] = &[
    HvPciConfig {
        ecam_base: 0x4010000000,
        ecam_size: 0x10000000,
        io_base: 0x3eff0000,
        io_size: 0x10000,
        pci_io_base: 0x0,
        mem32_base: 0x10000000,
        mem32_size: 0x2eff0000,
        pci_mem32_base: 0x10000000,
        mem64_base: 0x8000000000,
        mem64_size: 0x8000000000,
        pci_mem64_base: 0x8000000000,
        bus_range_begin: 0,
        bus_range_end: 0xff,
        domain: 0x0,
    }
];

// 这里只截取了dts中相关的配置
pcie@10000000 {
    ranges = <0x1000000 0x00 0x00 0x00 0x3eff0000 0x00 0x10000 
    0x2000000 0x00 0x10000000 0x00 0x10000000 0x00 0x2eff0000 
    0x3000000 0x80 0x00 0x80 0x00 0x80 0x00>;
    reg = <0x40 0x10000000 0x00 0x10000000>;
    bus-range = <0x00 0xff>;
    linux,pci-domain = <0x00>;
};
```

`HvPciConfig` 中的各个字段值通过解析设备树节点 `pcie@10000000` 中的属性来填写，对应关系如下：

- `ecam_base: 0x4010000000` <= `reg = <0x40 0x10000000 ...>` 
- `ecam_size: 0x10000000` <= `reg` 中的size `0x10000000`
- `io_base: 0x3eff0000` <= `ranges` 的io部分的cpu域的地址 `0x00 0x3eff0000`
- `io_size: 0x10000` <= `ranges` 的io部分的cpu域的size `0x00 0x10000`
- `pci_io_base: 0x0` <= `ranges` 的io部分的pci域的地址 `0x00 0x00 0x00`
- `mem32`与`mem64`实际与`io`相同，对于判断range中的字段类型可以参考[相关解释](https://elinux.org/Device_Tree_Usage#PCI_Address_Translation)
- `bus_range_begin: 0` <= `bus-range = <0x00 0xff>` 的起始值
- `bus_range_end: 0xff` <= `bus-range = <0x00 0xff>` 的结束值
- `domain: 0x0` <= `linux,pci-domain = <0x00>`，如果没有该字段可以启动系统后查看，如果只有一个pcie节点则domain一般为0

特别的
- 对于direct模式的pcie系统来说`mem32`、`mem64`和`io`没有实际意义，可填0
- 对于loongarch64来说，不区分mem32和mem64，但是因为是direct模式，所以可不填

### 设备配置
`ROOT_PCI_DEVS` 定义了需要由 hvisor 进行虚拟化管理的 PCI 设备列表。
每个设备通过 `pci_dev!` 宏配置，包含以下字段：

- `domain`: PCI 域号，与 `ROOT_PCI_CONFIG` 中的 `domain` 字段对应
- `bus`: PCI 总线号
- `device`: PCI 设备号
- `function`: PCI 功能号
- `dev_type`: 设备类型，`VpciDevType::Physical` 表示物理直通设备，其他类型定义参考`VpciDevType`

对于 x86 ，需要配置 `ROOT_PCI_MAX_BUS` 来指定最大总线号。

### dwc的特殊配置
对于使用 DWC PCIe 控制器的平台，除了需要配置 `ROOT_PCI_CONFIG` 外，还需要额外配置 `ROOT_DWC_ATU_CONFIG` 来指定 ATU 相关的参数。

`HvDwcAtuConfig` 中的各个字段值也通过解析设备树节点中的属性来填写，这里以 rk3568 为例，对应关系如下：
```
pub const ROOT_DWC_ATU_CONFIG: &[HvDwcAtuConfig] = &[
    HvDwcAtuConfig {
        ecam_base: 0x3c0400000,
        dbi_base: 0x3c0400000,
        dbi_size: 0x10000,
        apb_base: 0xfe270000,
        apb_size: 0x10000,
        cfg_base: 0xf2000000,
        cfg_size: 0x80000*2,
        io_cfg_atu_shared: 0,
    },
];

pcie@fe270000 {
    reg = <0x03 0xc0400000 0x00 0x400000 0x00 0xfe270000 0x00 0x10000>;
    reg-names = "pcie-dbi", "pcie-apb";
    ranges = <0x800 0x00 0xf2000000 0x00 0xf2000000 0x00 0x100000 ...>;
    num-viewport = <0x08>;
    ...
};
```

- `ecam_base: 0x3c0400000` <= `reg = <0x03 0xc0400000 ...>` 用于指示和 ROOT_PCI_CONFIG 中对应总线的联系，需要与其一一对应
- `dbi_base: 0x3c0400000` <= 从reg-names中发现该板卡的dbi为reg中的第一部分，所以从对应的区域获取地址，dbi一般用于root bus的访问
- `dbi_size: 0x10000` <= DBI 寄存器size
- `apb_base: 0xfe270000` <= 从reg-names中发现该板卡存在apb，所以根据对应信息填写，部分板卡该位置填充的值为cfg，需要注意甄别
- `apb_size: 0x10000` <= APB 寄存器size
- `cfg_base: 0xf2000000` <= 在该板卡中，cfg为`ranges` 中 `0x800` 类型的地址；部分板卡中cfg地址也是存储在reg中，用reg-names标识，需要注意甄别
- `cfg_size: 0x80000*2` <= cfg size
- `io_cfg_atu_shared: 0` <= 该值用于标识atu0是否被io访问共享，一般来说 `num-viewport >= 4` 时为 0，应与linux中的同名变量保持一致
