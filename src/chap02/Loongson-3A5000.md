# 在龙芯 3A5000主板（7A2000）上启动 hvisor

韩喻泷 <wheatfox17@icloud.com>

更新时间：2025.3.24

## 第一步：获取 hvisor 源码并编译

首先需要安装 `loongarch64-unknown-linux-gnu-` 工具链，请从 <https://github.com/sunhaiyong1978/CLFS-for-LoongArch/releases/download/8.0/loongarch64-clfs-8.0-cross-tools-gcc-full.tar.xz> 下载并解压到本地，然后请将 `cross-tools/bin` 目录添加到你的 `PATH` 环境变量中，保证 `loongarch64-unknown-linux-gnu-gcc` 等工具可以被 shell 直接调用。

然后克隆代码到本地：

```bash
git clone -b dev https://github.com/syswonder/hvisor
make BID=loongarch64/ls3a5000
```
编译完成后在 `target` 目录下可以找到 strip 之后的 `hvisor.bin` 文件。

## 第二步（不自己编译 buildroot/linux 等）：获取 rootfs/内核镜像

请从 <https://github.com/enkerewpo/linux-hvisor-loongarch64/releases> 下载最新发布的 hvisor 默认龙芯 linux 镜像（包括 root linux kernel+root linux dtb+root linux rootfs，其中 root linux rootfs 中包括 non root linux+nonroot linux dtb+nonroot linux rootfs）。rootfs 中已打包好 nonroot 的启动 json 以及 hvisor-tool、内核模块等。


## 第二步（自己编译 buildroot/linux 等）：完整编译 rootfs/内核镜像

如果你需要自己编译，这个流程将会较为复杂，接下来将介绍相关细节：

### 1. 准备好环境

创建一个工作目录（可选）：

```bash
mkdir workspace && cd workspace

git clone -b dev https://github.com/syswonder/hvisor
git clone https://github.com/enkerewpo/buildroot-loongarch64
git clone https://github.com/enkerewpo/linux-hvisor-loongarch64 hvisor-la64-linux
git clone https://github.com/enkerewpo/hvisor-tool
git clone https://github.com/enkerewpo/hvisor_uefi_packer
```
### 2. 准备 buildroot 环境

因为 buildroot 在找不到需要编译的 package 时会从各个地方下载源码压缩包，这里我准备好了一个预下载的镜像：

<https://pan.baidu.com/s/1sVPRt0JiExUxFm2QiCL_nA?pwd=la64>

下载后将 `dl` 目录放在 buildroot-loongarch64 根目录即可，或者你也可以不下载，让 buildroot 自动下载（可能会非常慢）。如果你在解压了 `dl` 目录后发现编译时仍然有软件包需要下载，也是正常现象。

### 3. 编译 buildroot

```bash
cd buildroot-loongarch64
make loongson3a5000_hvisor_defconfig

make menuconfig # 请将 Toolchain/Toolchain path prefix 设置为你本地的 loongarch64 工具链路径和前缀
# 然后选择右下角 save 保存到 .config 文件

make -j$(nproc)
```

<div class="warning">
    <h3>请注意</h3>
    <p> 这个过程可能持续数小时，取决于你的机器性能和网络环境。</p>
</div>


### 4. 第一次编译 linux（为后续 make world 做准备）

```bash
cd hvisor-la64-linux # 目前默认使用 linux 6.13.7
./build def # 生成默认 root linux 的 defconfig
# ./build nonroot_def # 生成默认 nonroot linux 的 defconfig

# ./build menuconfig # 如果你想自定义内核配置，可以使用这个命令
# （其会修改当前目录下的 .config 文件，请注意你正在修改的是 root linux 还是 nonroot linux 的配置，
# 可以查看根目录 .flag 文件内容是 ROOT 还是 NONROOT）

./build kernel # 编译当前 .config 对应的内核（可能是 root linux
# 或 nonroot linux，取决于 ./build def 和 ./build nonroot_def）
```

<div class="warning">
    <h3>请注意</h3>
    <p> 这个过程可能持续几十分钟，取决于你的机器性能。</p>
</div>

### 5. 通过 hvisor uefi packer 执行 make world 流程

首先需要修改 `hvisor_uefi_packer` 目录下的 `Makefile.1` 文件，将 `HVISOR_LA64_LINUX_DIR` 等变量修改为实际的路径：

```Makefile
HVISOR_LA64_LINUX_DIR = ../hvisor-la64-linux
BUILDROOT_DIR = ../buildroot-loongarch64
HVISOR_TOOL_DIR = ../hvisor-tool
```

然后运行：

```bash
cd hvisor_uefi_packer
./make_world
```

简单介绍一下 `make_world` 脚本的流程，具体命令请参见 `Makefile.1` 文件：
1. 编译 hvisor-tool，由于 hvisor-tool 的内核模块需要和 root linux 的内核版本一致，所以第一次需要先手动编译一次 root linux，然后 make world 才能成功编译 hvisor-tool。
2. 复制 hvisor-tool 的相关文件到 buildroot 的 rootfs overlay 中，位于 `$(BUILDROOT_DIR)/board/loongson/ls3a5000/rootfs_ramdisk_overlay`。
3. 编译 nonroot linux（nonroot 目前没有用到 buildroot，而是一个简单的 busybox rootfs），需要注意，其中生成的 `vmlinux` 内部包括 nonroot 的 dtb 和 busybox rootfs(initramfs)（全部内嵌在内核中），并将 `vmlinux.bin` 移动到 buildroot 的 rootfs overlay 中。请记住这个 nonroot linux 的 `vmlinux` 的 entry 地址，之后你可以修改 buildroot 的 overlay 中的 `linux2.json` 文件，将这个 entry 地址写入。
4. 编译 buildroot rootfs，此时 rootfs 内包括前面编译的 nonroot linux 的 vmlinux，以及 hvisor-tool 的相关文件。
5. 编译 root linux，生成的 `vmlinux` 内部包括 root linux 的 dtb 和 buildroot rootfs（initramfs），请记录这个 root linux 的 `vmlinux` 的 entry 地址，和文件路径，后续需要在 hvisor 和 hvisor uefi packer 中使用。
6. 结束，我们最终需要的就是这个 root linux 的 `vmlinux.bin`。

### 6. 编译 UEFI 镜像

由于 3A5000 以及之后的 3 系 CPU 的主板均采用 UEFI 启动，所以只能通过 efi 镜像的方法启动 hvisor。

接着上一步，在 hvisor uefi packer 目录中，首先修改 `./make_image` 脚本中的 `HVISOR_SRC_DIR` 为你实际保存 hvisor 源码的路径，然后运行编译脚本：

```bash
make menuconfig # 配置为你本地的loongarch64 gcc工具链前缀、hvisor.bin路径、vmlinux.bin路径

# 1. 修改make_image中的 HVISOR_SRC_DIR=../hvisor为你实际保存hvisor源码的路径，之后再运行脚本
# 2. 修改 BOARD=ls3a5000/ls3a6000（根据你的实际板子型号选择），后文的 env 中的 BOARD 同理

# ./make_world # 见上一步的说明，此步在不需要重新编译buildroot/linux 的情况下可跳过

ARCH=loongarch64 BOARD=ls3a5000 ./make_image
# make_image 仅编译 hvisor 和 BOOTLOONGARCH64.EFI
```

此时会在 `hvisor_uefi_packer` 目录下生成 `BOOTLOONGARCH64.EFI`，将其放在 U 盘的第一个 FAT32 分区的 `/EFI/BOOT/BOOTLOONGARCH64.EFI` 位置。

<div class="warning">
    <h3>请注意</h3>
    <p> 当你自己编译 root 和 nonroot linux 时，请手动 readelf 得到两个 vmlinux 的 entry 地址，并在 board.rs 以及 linux2.json 中对应写好，否则一定会启动失败。
</div>


## 上板启动

主板上电开机，按 **F12** 进入UEFI Boot Menu，选择你插入的 U 盘，在 UEFI Boot Menu 中选择 U 盘启动，即可进入 hvisor，然后自动进入 root linux。

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