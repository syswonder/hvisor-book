# 如何启动 Root Linux

## 安装QEMU

### 1. 安装依赖

```bash
sudo apt install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev \
    gawk build-essential bison flex texinfo gperf libtool patchutils bc \
    zlib1g-dev libexpat-dev pkg-config libglib2.0-dev libpixman-1-dev libsdl2-dev \
    git tmux python3 python3-pip ninja-build
```

> 注意，上述依赖包可能不全，例如：
>
> - 出现 `ERROR: pkg-config binary 'pkg-config' not found` 时，可以安装 `pkg-config` 包；
> - 出现 `ERROR: glib-2.48 gthread-2.0 is required to compile QEMU` 时，可以安装 `libglib2.0-dev` 包；
> - 出现 `ERROR: pixman >= 0.21.8 not present` 时，可以安装 `libpixman-1-dev` 包。

> 若生成设置文件时遇到报错 `ERROR: Dependency "slirp" not found, tried pkgconfig`：
>
> 下载 [https://gitlab.freedesktop.org/slirp/libslirp](https://gitlab.freedesktop.org/slirp/libslirp) 包，并按 README 安装即可。

### 2. 条件编译并安装QEMU

注意，QEMU 需要使用 9.0.1 或更高版本，以正常使用 PCI 虚拟化。
```bash
wget https://download.qemu.org/qemu-9.0.1.tar.xz
tar -xvf qemu-9.0.1.tar.xz
cd qemu-9.0.1
# 生成配置并编译
./configure --enable-kvm --enable-slirp --enable-debug --target-list=aarch64-softmmu,x86_64-softmmu
make -j$(nproc)
```

- `--enable-kvm`表示启用 ​Kernel-based Virtual Machine​ 加速支持；
- `--enable-slirp`表示启用用户态网络协议栈（SLIRP），允许虚拟机无需主机 root 权限即可访问网络。
- `--enable-debug`表示编译带调试信息的版本
- `--target-list=aarch64-softmmu,x86_64-softmmu`表示只编译 ​ARM64​ 和 ​x86_64​ 的系统模拟器（softmmu表示全系统模拟），如果需要其他架构的 QEMU，可以参考[QEMU官方文档](https://wiki.qemu.org/Hosts/Linux)。

### 3. 配置qemu的环境变量

编辑 `~/.bashrc` 文件（如果启用其他shell则编辑对应文件，如使用zsh则编辑`~/.zshrc`文件），在文件的末尾加入如下代码：
```bash
# 请注意，qemu-9.0.1 的父目录可以随着你的实际安装位置灵活调整。另外需要把其放在 $PATH 变量开头。
export PATH="/path/to/qemu-9.0.1/build:$PATH"
```

随后即可在当前终端 `source ~/.bashrc` 更新系统路径，或者直接重启一个新的终端。

#### 4. 测试QEMU是否安装成功

```bash
qemu-system-aarch64 --version
```
可以将`aarch64`替换为其他架构以检查对应版本的qemu是否正确安装。

## 启动Root Linux

### 1. 准备内核镜像

这里以 Linux Kernel 5.4 为例展示内核镜像的编译方法。

首先需要安装对应架构的交叉编译器，如果是`aarch64`架构可安装`aarch64-none-linux-gnu-10.3`；如果是`riscv64`架构可安装`riscv-gnu-toolchain`；如果是`loongarch64`架构可安装`loongarch64-unknown-linux-gnu-`工具链。下面以`aarch64`架构为例。

```bash
# CROSS_COMPILE 路径需要根据安装交叉编译器的路径进行更改
CROSS_COMPILE_PATH="<路径>/bin"

# 下载 linux 5.4 源码
git clone https://github.com/torvalds/linux -b v5.4 --depth=1
cd linux
git checkout v5.4

# 生成默认的编译配置
CROSS_COMPILE_PREFIX=${CROSS_COMPILE_PATH}/aarch64-none-linux-gnu-
make ARCH=arm64 CROSS_COMPILE=${CROSS_COMPILE_PREFIX} defconfig

# 启用 CONFIG_BLK_DEV_RAM，以启用 RAM 块设备支持
./scripts/config --enable CONFIG_BLK_DEV_RAM
# 启用 CONFIG_IPV6 和 CONFIG_BRIDGE，以支持在 root linux 中创建网桥和 tap 设备
./scripts/config --enable CONFIG_IPV6
./scripts/config --enable CONFIG_BRIDGE

# 编译
make ARCH=arm64 CROSS_COMPILE=${CROSS_COMPILE_PREFIX} Image -j$(nproc)
```

> 如果编译 linux 时报错：
>
> ```
> /usr/bin/ld: scripts/dtc/dtc-parser.tab.o:(.bss+0x20): multiple definition of `yylloc'; scripts/dtc/dtc-lexer.lex.o:(.bss+0x0): first defined here
> ```
>
> 则修改 linux 文件夹下 `scripts/dtc/dtc-lexer.lex.c`，在 `YYLTYPE yylloc;` 前增加 `extern`。再次编译，发现会报错 `openssl/bio.h: No such file or directory`，此时执行 `sudo apt install libssl-dev`。

> 编译过程中会出现：
> ```
> RAM block device support (BLK_DEV_RAM) [Y/n/m/?] y
>   Default number of RAM disks (BLK_DEV_RAM_COUNT) [16] (NEW)
>   Default RAM disk size (kbytes) (BLK_DEV_RAM_SIZE) [4096] (NEW)
> ```
> 即配置具体参数，直接回车采用默认值即可。

编译完毕，内核镜像位于 `arch/arm64/boot/Image`。

将镜像文件放置于`platform/<架构>/<平台名>/image/kernel/`目录下并命名为`Image`，例如`platform/aarch64/qemu-gicv3/image/kernel`。

### 2. 准备Root文件系统

可以参考如下方法使用 Ubuntu 22.04 自制根文件系统。以`aarch64`架构为例，如果编译的是其他架构的 qemu 需换成对应架构。

```bash
# QEMU 路径，需要根据之前安装时的路径进行更改
QEMU_PATH="<路径>/build/qemu-system-aarch64" # 这里换成对应架构

# 下载 ubuntu base
wget http://cdimage.ubuntu.com/ubuntu-base/releases/22.04/release/ubuntu-base-22.04.5-base-arm64.tar.gz

# 创建 rootfs，用于挂载后续的 rootfs1.img
mkdir -p rootfs

# 创建一个 1 GiB 大小的 rootfs1.img，可以通过修改 count 修改 img 大小
dd if=/dev/zero of=rootfs1.img bs=1M count=1024 oflag=direct
# 格式化为 ext4 文件系统
mkfs.ext4 rootfs1.img

# 挂载 rootfs1.img
sudo mount -t ext4 rootfs1.img rootfs/
# 将 ubuntu.tar.gz 的内容解压到 rootfs
sudo tar -xzf ubuntu-base-22.04.5-base-arm64.tar.gz -C rootfs/

# 让 rootfs 绑定和获取物理机的一些信息和硬件
sudo cp "${QEMU_PATH}" rootfs/usr/bin/
sudo cp /etc/resolv.conf rootfs/etc/resolv.conf
sudo mount -t proc /proc rootfs/proc
sudo mount -t sysfs /sys rootfs/sys
sudo mount -o bind /dev rootfs/dev
sudo mount -o bind /dev/pts rootfs/dev/pts

# 将文件系统切换到 rootfs
sudo chroot rootfs  # 执行该指令可能会报错，请参考下面的解决办法
# 在 rootfs 中安装必要的软件包
apt-get update
apt-get install git sudo vim bash-completion \
    kmod net-tools iputils-ping resolvconf ntpdate screen
apt-get clean

# 退出 rootfs
exit

# 卸载 rootfs
sudo umount rootfs/proc
sudo umount rootfs/sys
sudo umount rootfs/dev/pts
sudo umount rootfs/dev
sudo umount rootfs
```

> 此时可以顺便创建后续要用到的 `rootfs2.img`作为 NonRoot Linux 的根文件系统，其大小应适当减少，以便放入 `rootfs1.img` 中。
> ```bash
> QEMU_PATH="<路径>/build/qemu-system-aarch64" # 这里换成对应架构
>
> # 创建 rootfs2.img，其大小适当减少到 256 MiB
> dd if=/dev/zero of=rootfs2.img bs=1M count=256 oflag=direct
> mkfs.ext4 rootfs2.img
> sudo mount -t ext4 rootfs2.img rootfs/
> sudo tar -xzf ubuntu-base-22.04.5-base-arm64.tar.gz -C rootfs/
> sudo cp "${QEMU_PATH}" rootfs/usr/bin/
> sudo cp /etc/resolv.conf rootfs/etc/resolv.conf
> sudo mount -t proc /proc rootfs/proc
> sudo mount -t sysfs /sys rootfs/sys
> sudo mount -o bind /dev rootfs/dev
> sudo mount -o bind /dev/pts rootfs/dev/pts
> sudo umount rootfs/proc
> sudo umount rootfs/sys
> sudo umount rootfs/dev/pts
> sudo umount rootfs/dev
> sudo umount rootfs
> ```

最后卸载挂载，完成根文件系统的制作。

> 执行 `sudo chroot rootfs` 时，如果报错 `chroot: failed to run command '/bin/bash': Exec format error`，可以执行指令：
>
> ```bash
> sudo apt-get install qemu-user-static
> sudo update-binfmts --enable qemu-aarch64 # 这里换成对应的架构名称
> ```

将 Root 文件系统放置于`platform/<架构>/<平台名>/image/virtdisk/`目录下并命名为`rootfs1.ext4`，例如`platform/aarch64/qemu-gicv3/image/virtdisk/rootfs1.ext4`。

### 3. 准备设备树

切换到`platform/<架构>/<平台名>/image/dts/`目录下，下面以`platform/aarch64/qemu-gicv3/image/dts/`为例，执行`make all`命令，该命令会​自动编译当前目录下所有`.dts`文件为`.dtb`文件，确保设备树源文件被转换为目标硬件可用的二进制格式。
```bash
cd platform/aarch64/qemu-gicv3/image/dts/
make all
cd -    # 回到 hvisor 目录
```

### 4. 启动QEMU
在hviosr目录下执行执行`make run`命令启动 hvisor，可通过指定`ARCH=<架构>`调整架构、`BOARD=<平台名>`调整平台名、`LOG=<日志等级>`调整日志输出等级，例如
```bash
make ARCH=aarch64 LOG=info BOARD=qemu-gicv3 run
```

### 5. 进入 uboot 启动界面

启动 hvisor 后将自动加载uboot，等待uboot加载完成后，该界面下执行
```
bootm 0x40400000 - 0x40000000
```
该启动命令会从物理地址 0x40400000 启动 hvisor，0x40000000 本质上已无用，但因历史原因仍然保留。hvisor 启动时，会自动启动 root linux（用于管理的 Linux），并进入 root linux 的 shell 界面，root linux 即为 zone0，承担管理工作。
