# 在 QEMU x86_64 上运行 Hvisor

刘天弘（lzoi_lth@163.com）

## 一、环境准备

### 硬件

- 具有 Intel CPU 的计算机
- 支持 VT-x，并已在 BIOS 启用

### 软件

- 使用 Ubuntu 等 Linux 操作系统，以下示例基于 WSL2 Ubuntu 24.04 LTS


## 二、安装 gcc 编译器

```bash
sudo apt update
sudo apt install gcc
```

## 三、安装 QEMU

推荐自行编译 QEMU，便于日后修改 QEMU 源码进行调试。此处以 QEMU v9.2.3 为例，也可以安装更新的版本。

```bash
# 安装依赖
sudo apt install git libglib2.0-dev libfdt-dev libpixman-1-dev zlib1g-dev \
ninja-build python3-venv bzip2 make

# 获取 QEMU 源码
wget https://download.qemu.org/qemu-9.2.3.tar.xz

# 解压
tar xvJf qemu-9.2.3.tar.xz

# 进入源码目录
cd qemu-9.2.3/

# 生成配置并编译
./configure --enable-kvm --enable-slirp --target-list=x86_64-softmmu

# 编译 qemu
make -j$(nproc)
```

编辑 `~/.bashrc` 文件，在末尾加入：

```bash
export PATH="/path/to/qemu-9.2.3/build:$PATH"
```

随后在终端执行 `source ~/.bashrc` 更新环境变量，或者重启一个新终端。使用 `qemu-system-x86_64 --version` 确认当前 QEMU 版本，若为 9.2.3 则安装成功。

## 四、编译 Linux Kernel

以 Linux v5.19 为例。

```bash
# 安装依赖
sudo apt install flex bison libelf-dev libssl-dev

# 下载 linux 5.19 源码
git clone https://github.com/torvalds/linux -b v5.19 --depth=1
cd linux
git checkout v5.19

# 生成默认的编译配置
make ARCH=x86_64 defconfig

# 启用 X2APIC 及其依赖项
./scripts/config --enable CONFIG_X86_X2APIC
./scripts/config --enable CONFIG_ACRN_GUEST
# 启用 RAM 块设备支持
./scripts/config --enable CONFIG_BLK_DEV_RAM
# 启用 IPV6、BRIDGE 和 TUN，以支持在 root linux 中创建网桥和 tap 设备
./scripts/config --enable CONFIG_IPV6
./scripts/config --enable CONFIG_BRIDGE
./scripts/config --enable CONFIG_TUN
# 启用 VIRTIO MMIO，以支持 non-root linux 使用 virtio 驱动
./scripts/config --enable CONFIG_VIRTIO_MMIO
./scripts/config --enable CONFIG_VIRTIO_MMIO_CMDLINE_DEVICES
# 关闭编译期间将警告视为报错
./scripts/config --disable CONFIG_WERROR

# 编译，遇到选项一直 Enter 即可
make ARCH=x86_64 -j$(nproc)
```

## 五、基于 Ubuntu 22.04 构建根文件系统

```bash
# 下载 Ubuntu 镜像
wget http://cdimage.ubuntu.com/ubuntu-base/releases/22.04/release/ubuntu-base-22.04.5-base-amd64.tar.gz

# 创建 rootfs，用于挂载 rootfs1.img
mkdir -p rootfs

# 创建一个 2G 大小的 ubuntu.img，可以修改 count 修改 img 大小
dd if=/dev/zero of=rootfs1.img bs=1M count=2048 oflag=direct

# 格式化为 ext4 文件系统
mkfs.ext4 rootfs1.img

# 挂载 rootfs1.img
sudo mount -t ext4 rootfs1.img rootfs/

# 将 ubuntu.tar.gz 的内容解压到 rootfs
sudo tar -xzf ubuntu-base-22.04.5-base-amd64.tar.gz -C rootfs/

# 让 rootfs 绑定和获取物理机的一些信息和硬件
sudo cp /etc/resolv.conf rootfs/etc/resolv.conf
sudo mount -t proc /proc rootfs/proc
sudo mount -t sysfs /sys rootfs/sys
sudo mount -o bind /dev rootfs/dev
sudo mount -o bind /dev/pts rootfs/dev/pts

# 将文件系统切换到 rootfs
sudo chroot rootfs

# 在 rootfs 中安装必要的软件包
apt update
apt install git sudo vim bash-completion \
    kmod net-tools iputils-ping resolvconf ntpdate screen \
    pciutils iproute2 isc-dhcp-client systemd bridge-utils

# 创建进入根文件系统时执行的 init 脚本，赋予执行权限
touch init
chmod 777 init

# 修改 init 脚本，具体内容如下：
# ======================= init =======================
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
mkdir -p /dev/pts
mount -t devpts none /dev/pts
echo
echo "Hello Zone 0!"
echo "This boot took $(cut -d' ' -f1 /proc/uptime) seconds"
echo
script /dev/null -c "hostname zone0 && su"
# ====================================================

# 退出 rootfs
exit

# 卸载 rootfs
sudo umount rootfs/proc
sudo umount rootfs/sys
sudo umount rootfs/dev/pts
sudo umount rootfs/dev
sudo umount rootfs
```


## 六、Rust 环境配置

请参考 [Rust 语言圣经](https://course.rs/first-try/intro.html)。


## 七、编译并运行 Hvisor，进入 zone0 Linux

```bash
# 下载到本地
git clone https://github.com/syswonder/hvisor.git
cd ./hvisor

# 安装依赖
sudo apt install grub-common xorriso grub-efi-amd64 mtools ovmf

# 新建 kernel 文件夹，用于放置 Linux kernel
mkdir -p ./platform/x86_64/qemu/image/kernel

# 将第四步编译的 Linux kernel 复制到 kernel 文件夹
LINUX_PATH="<路径>/linux"
cp "${LINUX_PATH}/arch/x86/boot/setup.bin" ./platform/x86_64/qemu/image/kernel/
cp "${LINUX_PATH}/arch/x86/boot/vmlinux.bin" ./platform/x86_64/qemu/image/kernel/

# 新建 virtdisk 文件夹，用于放置根文件系统
mkdir -p ./platform/x86_64/qemu/image/virtdisk

# 将第五步制作的根文件系统 rootfs1.img 复制到 virtdisk 文件夹
ROOTFS1_PATH="<路径>/rootfs1.img"
cp "${ROOTFS1_PATH}" ./platform/x86_64/qemu/image/virtdisk/

# 运行 Hvisor
make ARCH=x86_64 BOARD=qemu run
```

<div class="warning">
    <h3>请注意</h3>
    <p> 
若执行 `make BOARD=qemu run` 遇到如下报错：

```
Could not access KVM kernel module: Permission denied
qemu-system-x86_64: -accel kvm: failed to initialize kvm: Permission denied
```

执行 `sudo usermod -a -G kvm 你的用户名` 加入 kvm 的用户组，重启终端后再次尝试。
    </p>
</div>

## 八、使用 hvisor-tool 运行 zone1 Linux

完成最新版本的 hvisor-tool 的编译，具体请参考 [hvisor-tool](https://github.com/syswonder/hvisor-tool) 的 README。

```bash
git clone https://github.com/syswonder/hvisor-tool.git
cd hvisor-tool

# KDIR 必须设为第四步编译的 Linux kernel 根目录 
make all ARCH=x86_64 LOG=LOG_WARN KDIR=linux根目录
```

编译完成后，需要将 hvisor-tool 的可执行文件 `tools/hvisor` 和内核模块 `driver/hvisor.ko` 复制到 zone0 的根文件系统中的指定位置，例如 `/`，将 zone1 的根文件系统、内核镜像以及配置文件放在同一目录。具体的文件名需要与 hvisor-tool 配置文件（来自 hvisor-tool 的 `examples/qemu-x86_64/virtio_cfg.json` 和 `examples/qemu-x86_64/zone1_linux.json`）的内容保持一致。

```bash
LINUX_PATH="<路径>/linux"
HVISOR_PATH="<路径>/hvisor"
HVISOR_TOOL_PATH="<路径>/hvisor-tool"
ROOTFS2_PATH="<路径>/rootfs2.img"

# 回到创建根文件系统时的目录，挂载
sudo mount -t ext4 rootfs1.img rootfs/

# 复制 hvisor-tool 的 driver/hvisor.ko 和 tools/hvisor
sudo cp "${HVISOR_TOOL_PATH}/driver/hvisor.ko" rootfs/hvisor.ko
sudo cp "${HVISOR_TOOL_PATH}/tools/hvisor" rootfs/hvisor

# 复制 zone1 的配置文件到 root 路径下
sudo cp "${HVISOR_TOOL_PATH}/examples/qemu-x86_64/virtio_cfg.json" \
    rootfs/virtio_cfg.json
sudo cp "${HVISOR_TOOL_PATH}/examples/qemu-x86_64/zone1_linux.json" \
    rootfs/zone1_linux.json

# 复制 zone1 的根文件系统和内核镜像，文件系统的制作见下文，
# 内核镜像可以直接沿用第四步所得
sudo cp "${ROOTFS2_PATH}" \
    rootfs/rootfs2.img
sudo cp "${LINUX_PATH}/arch/x86/boot/setup.bin" \
    rootfs/setup.bin
sudo cp "${LINUX_PATH}/arch/x86/boot/vmlinux.bin" \
    rootfs/vmlinux.bin

# 复制内核跳板，需要 Hvisor 已经编译过一次
sudo cp "${HVISOR_PATH}/platform/x86_64/qemu/image/bootloader/out/boot.bin" \
    rootfs/boot.bin

# 修改 init，新增内容如下：
# ======================= init =======================
echo "This boot took $(cut -d' ' -f1 /proc/uptime) seconds"
echo
...
ifconfig eth0 up
dhclient eth0
brctl addbr br0
brctl addif br0 eth0
ifconfig eth0 0
dhclient br0
ip tuntap add dev tap0 mode tap
brctl addif br0 tap0
ip link set dev tap0 up
insmod hvisor.ko
...
script /dev/null -c "hostname zone0 && su"
# ====================================================

# 卸载
sudo umount rootfs

# 需要将新的 rootfs1.img 移动至 Hvisor 中
ROOTFS1_PATH="<路径>/rootfs1.img"
cp "${ROOTFS1_PATH}" ./platform/x86_64/qemu/image/virtdisk/
```

zone1 根文件系统的制作可以参考第五步，不用安装额外的依赖包。但需要适当缩减容量（例如 1G），使之能够放入 zone0 的根文件系统中。zone1 的 init 脚本内容如下：

```bash
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
mkdir -p /dev/pts
mount -t devpts none /dev/pts
echo
echo "Hello Zone 1!"
echo "This boot took $(cut -d' ' -f1 /proc/uptime) seconds"
echo
ip link set eth0 up
dhclient eth0
script /dev/null -c "hostname zone1 && su"
```

回到 Hvisor 的根目录，执行下述指令，即可进入 zone1 Linux

```bash
# 运行 Hvisor，进行 zone0
make ARCH=x86_64 BOARD=qemu run

# 启动 hvisor-tool virtio 设备
nohup ./hvisor virtio start virtio_cfg.json &

# 启动 zone1
./hvisor zone start ./zone1_linux.json

# 切换到 zone1 的终端，xxx 一般为 1
screen /dev/pts/xxx
```