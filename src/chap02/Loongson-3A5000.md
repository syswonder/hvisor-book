# 在龙芯3A5000主板（7A2000）上启动 hvisor

韩喻泷 <wheatfox17@icloud.com>

更新时间：2025.3.3

## 第一步：获取 hvisor 源码并编译

首先需要安装 `loongarch64-unknown-linux-gnu-` 工具链，请从 <https://github.com/sunhaiyong1978/CLFS-for-LoongArch/releases/download/8.0/loongarch64-clfs-8.0-cross-tools-gcc-full.tar.xz> 下载并解压到本地，然后请将 `cross-tools/bin` 目录添加到你的 `PATH` 环境变量中，保证 `loongarch64-unknown-linux-gnu-gcc` 等工具可以被 shell 直接调用。

然后克隆代码到本地：

```bash
git clone -b dev https://github.com/syswonder/hvisor
make ARCH=loongarch64 FEATURES=platform_3a5000
```
编译完成后在 target 目录下可以找到 strip 之后的 `hvisor.bin`（编译输出的最后一行会显示文件路径）。

## 获取 `vmlinux.bin` 等文件

请从<https://github.com/enkerewpo/linux-hvisor-loongarch64/releases>下载最新发布的 hvisor 默认龙芯 linux 镜像（包括 root linux kernel+root linux dtb+root linux rootfs，其中 root linux rootfs 中包括 non root linux+nonroot linux dtb+nonroot linux rootfs）。如果你需要自行编译 linux kernel 以及 rootfs，可参考该仓库的 `arch/loongarch` 目录中 hvisor 相关设备树以及我为 3A5000 移植的 buildroot 环境（<https://github.com/enkerewpo/buildroot-loongarch64>）。如果你需要手动编译 hvisor-tool，请参考<https://github.com/enkerewpo/hvisor-tool>，关于所有环境的编译顺序和脚本调用流程请参考 `Makefile.1` 文件中 `world` 目标内的代码（<https://github.com/enkerewpo/hvisor_uefi_packer/blob/main/Makefile.1>），并通过运行 `./make_world` 脚本编译所有东西，如果你需要手动编译这些，则需要在 `Makefile.1` 内修改对应的代码路径变量，包括：

```
HVISOR_LA64_LINUX_DIR = ../hvisor-la64-linux
BUILDROOT_DIR = ../buildroot-loongarch64
HVISOR_TOOL_DIR = ../hvisor-tool
```

然后运行 `./make_world`，请注意，第一次编译 linux 和 buildroot 的时间可能相当长（可能长达几十分钟，取决于你的机器性能）。

## 获取 hvisor UEFI Image Packer

由于 3A5000 以及之后的 3 系 CPU 的主板均采用 UEFI 启动，所以只能通过 efi 镜像的方法启动 hvisor，克隆仓库<https://github.com/enkerewpo/hvisor_uefi_packer>到本地：

```bash
make menuconfig # 配置为你本地的loongarch64 gcc工具链前缀、hvisor.bin路径、vmlinux.bin路径
# 1. 修改make_image中的HVISOR_SRC_DIR=../hvisor为你实际保存hvisor源码的路径，之后再运行脚本
# 2. 修改 FEATURES=platform_3a5000/3a6000yy0121，取决于你的主板型号，其中不带后缀的默认是龙芯自己的官方主板，带后缀的为第三方设计的主板
./make_image
# 得到 BOOTLOONGARCH64.EFI 文件
```

得到的`BOOTLOONGARCH64.EFI`必须放在U盘的第一个FAT32分区的`/EFI/BOOT/BOOTLOONGARCH64.EFI`位置。然后插入U盘启动即可进入 hvisor 并自动启动 root linux。

由于 root linux 相关的元信息（加载地址，内存区域等）硬编码在 hvisor 源码中（`src/platform/ls3a5000_loongarch64.rs`），如果你是手动编译 linux 内核，则需要修改这里的配置再重新编译hvisor。

## 上板启动

主板上电开机，按 **F12** 进入UEFI Boot Menu，选择你插入的 U 盘后回车，会自动启动 hvisor，并进入 root linux 的 bash 环境。

## 启动 nonroot

如果你使用的是 release 中提供的相关镜像，启动后在 root linux 的 bash 内输入：

```bash
./daemon.sh
./start.sh # 启动 nonroot，之后请手动运行 screen /dev/pts/0
./start.sh -s # 启动 nonroot 并自动进入 screen
```

之后会自动启动 nonroot（一些相关配置文件位于 root linux 的 `/tool` 目录内，包括提供给 hvisor-tool 的 nonroot zone 配置 json 以及 virtio 配置 json 文件），之后回自动打开一个screen 进程连接 nonroot linux 的 virtio-console，你会看到一个打印了 nonroot 字样的 bash 出现，你可以在使用 screen 时按 `CTRL+A D` 快捷键 detach（请记住显示的 screen session 名称 / ID），此时会返回 root linux，如果希望返回 nonroot linux，则运行

```bash
screen -r {刚才的session全名或者只输入最前面的 ID}
```

之后会返回 nonroot linux 的 bash。