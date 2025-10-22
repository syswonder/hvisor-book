# hvisor 硬件适配 


## 设计原则

1. **代码与板子配置分离**：hvisor 本身的 `src` 内部不出现任何 `platform_xxx` 相关的 `cfg`。
2. **平台独立性**：引入之前的 hvisor-deploy 架构，在 `platform` 目录下有序存放各个体系结构和板子的相关信息。
3. **板卡目录索引**：
   - 统一采用 `platform/$ARCH/$BOARD` 作为板卡专用目录。
   - 每个板卡的唯一 **BID (Board ID)** 采用 `ARCH/BOARD` 格式，例如 `aarch64/qemu-gicv3`。
4. **编译简化**：支持使用 `BID=xxx/xxx` 直接指定板卡，同时兼容 `ARCH=xxx BOARD=xxx` 风格。
5. **结构化配置**：每个板卡目录包含如下文件：
   - `linker.ld` - 链接脚本
   - `platform.mk` - QEMU 启动 Makefile 及 `hvisor.bin` 处理
   - `board.rs` - 板卡定义 Rust 代码
   - `configs/` - hvisor-tool 启动 zone 的 JSON 配置
   - `cargo/`
     - `features` - 板卡对应的具体cargo features，包括驱动、功能等
     - `config.template.toml` -  `.cargo/config` 的模板，由每个板子自己维护
   - `test/` - (可选) QEMU 相关测试代码，包括单元测试、系统测试等
   - `image/` - 启动文件目录，包含多个子目录：
     - `bootloader/` - (可选) 用于 QEMU 本地运行和 unittest/systemtest 测试
     - `dts/` - (可选) zone 0, 1, 2, … 的设备树源文件
     - `its/` - (可选) 用于 U-Boot FIT image 生成（hvisor aarch64 zcu102）
     - `acpi/` - (可选) x86 平台的 ACPI 设备树源码（hvisor x86_64）
     - `kernel/` - (可选) 适用于目标平台的 kernel Image
     - `virtdisk/` - (可选) 虚拟磁盘文件，如 rootfs 等

## 代码实现细节

### 自动生成 `.cargo/config.toml`
- 通过 `tools/gen_cargo_config.sh` 生成，确保 `linker.ld` 配置动态更新。
- `config.template.toml` 采用 `__ARCH__`、`__BOARD__` 等占位符，由 `gen_cargo_config.sh` 替换，生成 `.cargo/config.toml`。

### `build.rs` 自动软链接 `board.rs`
- `build.rs` 负责将 `platform/$ARCH/$BOARD/board.rs` 软链接到 `src/platform/__board.rs`。
- 避免 Makefile 处理，每次构建仅在 `env` 变量变更时触发，减少不必要的全量编译。

### 通过 Cargo features 选择驱动
- **避免 `platform_xxx` 直接出现在 `src/`**，改为基于 `features` 进行配置。
- `cargo/features` 统一存储板卡驱动、功能等配置。

## 各板卡对应 `features` 一览

| BOARD ID               | FEATURES                                                     |
| ---------------------- | ------------------------------------------------------------ |
| `aarch64/qemu-gicv3`   | `gicv3` `pl011` `iommu` `pci` `pt_layout_qemu`               |
| `aarch64/qemu-gicv2`   | `gicv2` `pl011` `iommu` `pci` `pt_layout_qemu`               |
| `aarch64/imx8mp`       | `gicv3` `imx_uart`                                           |
| `aarch64/zcu102`       | `gicv2` `xuartps`                                            |
| `riscv64/qemu-plic`    | `plic`                                                       |
| `riscv64/qemu-aia`     | `aia`                                                        |
| `loongarch64/ls3a5000` | `loongson_chip_7a2000` `loongson_uart` `loongson_cpu_3a5000` |
| `loongarch64/ls3a6000` | `loongson_chip_7a2000` `loongson_uart` `loongson_cpu_3a6000` |
| `aarch64/rk3588`       | `gicv3` `uart_16550` `uart_addr_rk3588` `pt_layout_rk`       |
| `aarch64/rk3568`       | `gicv3` `uart_16550` `uart_addr_rk3568` `pt_layout_rk`       |
| `x86_64/qemu         ` |                                                              |

## 开发与编译指南

### 编译不同板卡

```bash
make ARCH=aarch64 BOARD=qemu-gicv3
make BID=aarch64/qemu-gicv3  # 使用 BID 简写
make BID=aarch64/imx8mp
make BID=loongarch64/ls3a5000
make BID=x86_64/qemu
```

### 适配新板卡

1. **确定 `features`**：对照已有 `features` 归类，添加所需驱动和配置。
2. **创建 `platform/$ARCH/$BOARD` 目录**：
   - 添加 `linker.ld`, `board.rs`, `features` 等文件。
3. **编译测试**：

```bash
make BID=xxx/new_board
```

### `features` 设计原则

- **最小化层次**：
  - 例如 `cpu-a72` 而不是 `board_xxx`，以便多个板卡复用。
- **明确驱动/功能分类**：
  - `irqchip` (`gicv3`, `plic`, ...)
  - `uart` (`pl011`, `imx_uart`, ...)
  - `iommu`, `pci`, `pt_layout_xxx`, ...

