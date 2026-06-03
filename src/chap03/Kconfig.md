# hvisor Kconfig 配置体系

## 1. 为什么需要 Kconfig

hvisor 支持多种 CPU 架构（aarch64、riscv64、loongarch64、x86_64）和多种开发板。不同板级需要不同的驱动与功能组合，例如：

- GICv2 / GICv3、PLIC、AIA 等中断控制器
- PL011 / UART16550 等串口
- PCI/PCIe 及 ECAM、DesignWare 等访问方式
- IOMMU、Virtio-PCI 等可选子系统

若把这些选项全部写死在 `Cargo.toml` 的 `[features]` 里，每增加一块板子就要维护一份 feature 列表，容易出错。Kconfig 的做法与 Linux 内核类似：

1. 在统一的 `Kconfig` 文件里**声明所有可选项**及其依赖关系；
2. 每块板子有一份 **`defconfig`**，记录默认开启哪些选项；
3. 构建时生成 **`.config`**，再转成 Rust 编译器能识别的 **`rustc-cfg`**。

## 2. 快速编译

**BID**（Board ID）格式为 `<arch>/<board>`，例如 `aarch64/qemu-gicv3`、`riscv64/qemu-plic`。

Makefile 通过 `ARCH` 和 `BOARD`（或 `BID=aarch64/qemu-gicv3`）选择平台目录：

```bash
make defconfig ARCH=aarch64 BOARD=qemu-gicv3
make all ARCH=aarch64 BOARD=qemu-gicv3
```

## 3. 核心文件一览

| 路径 | 作用 |
|------|------|
| `kconfig/Kconfig` | 全局 Kconfig 菜单：架构、中断、UART、PCIe、IOMMU 等 |
| `kconfig/cfg_map.toml` | `CONFIG_*` 符号 → Rust `cfg` 名称的映射表 |
| `platform/<arch>/<board>/kconfig/defconfig` | 该板级的默认 `CONFIG_*` 列表 |
| `.config` | 构建时生成的完整配置（**不要手动编辑后长期提交**） |
| `build.rs` | 根据 `.config` 发出 `cargo:rustc-cfg=...`，并链接 `board.rs` |
| `tools/kconfig/kconfig_cli.py` | `defconfig` / `menuconfig` 入口 |
| `tools/kconfig/host_config.sh` | 根据 `.config` 生成 `.cargo/config.toml` 等 |

## 4. 常用 Make command

| 命令 | 说明 |
|------|------|
| `make defconfig ARCH=... BOARD=...` | 从板级 defconfig 生成 `.config` |
| `make menuconfig ARCH=... BOARD=...` | 交互式修改配置（需 Python venv + curses） |
| `make savedefconfig ARCH=... BOARD=...` | 将当前 `.config` 中的 `CONFIG_*` 写回 defconfig |
| `make all ARCH=... BOARD=...` | defconfig → 生成 Cargo 配置 → 编译 → 内存重叠检查 |
| `make elf ARCH=... BOARD=...` | 仅编译，不打包 bin |
| `make clean` | 清理构建产物 |

首次使用需创建 Kconfig 虚拟环境（`make defconfig` 会自动触发）：

```bash
./tools/kconfig/bootstrap_venv.sh
```

## 5. 如何新增一个 feature（完整流程）

以新增可选功能 **`MY_FEATURE`** 为例（Rust cfg 名为 `my_feature`）。

### 步骤 1：在 `kconfig/Kconfig` 中声明

在合适菜单下增加 bool 选项，并写好 `depends on`：

```kconfig
config MY_FEATURE
	bool "My optional subsystem"
	depends on PCI          # 按实际需求填写
	default n
```

若与其它选项有联动，可使用 `select` / `depends on`。

### 步骤 2：在 `kconfig/cfg_map.toml` 中注册映射

```toml
CONFIG_MY_FEATURE = "my_feature"
```

键名必须与 `.config` 中一致（带 `CONFIG_` 前缀）。

### 步骤 3：在 Rust 源码中使用

```rust
#[cfg(my_feature)]
mod my_feature_impl;

#[cfg(my_feature)]
pub fn init_my_feature() { ... }
```

复杂条件可组合平台 cfg 与架构：

```rust
#[cfg(all(my_feature, target_arch = "riscv64"))]
fn riscv_specific() { ... }
```

**不要**再使用已废弃的 `#[cfg(feature = "my_feature")]` 或 `Cargo.toml [features]`。

### 步骤 4：在需要该功能的板级 defconfig 中启用

编辑 `platform/<arch>/<board>/kconfig/defconfig`，增加：

```text
CONFIG_MY_FEATURE=y
```

示例（`platform/aarch64/qemu-gicv3/kconfig/defconfig`）：

```text
CONFIG_ARCH_AARCH64=y
CONFIG_PCI=y
CONFIG_VIRTIO_PCI=y
...
```
