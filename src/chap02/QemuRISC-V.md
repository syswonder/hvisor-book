# 安装qemu
安装QEMU 9.0.2：
```
wget https://download.qemu.org/qemu-9.0.2.tar.xz
# 解压
tar xvJf qemu-9.0.2.tar.xz
cd qemu-9.0.2
# 配置Riscv支持
./configure --target-list=riscv64-softmmu,riscv64-linux-user 
make -j$(nproc)
#加入环境变量
export PATH=$PATH:/path/to/qemu-9.0.2/build
#测试是否安装成功
qemu-system-riscv64 --version
```
# 安装交叉编译器
riscv的交叉编译器需从riscv-gnu-toolchain获取并编译。
```
git clone https://github.com/riscv/riscv-gnu-toolchain
cd riscv-gnu-toolchain
#ubuntu
sudo apt-get install autoconf automake autotools-dev curl python3 python3-pip python3-tomli libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev ninja-build git cmake libglib2.0-dev libslirp-dev gdb

./configure --prefix=/path/to/riscv #你自己的存储路径
make linux
echo 'export PATH=/opt/riscv64/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```
这样就得到了 riscv64-unknown-linux-gnu工具链。
# 编译Linux
qemu-plic:
```
git clone https://github.com/torvalds/linux -b v6.2 --depth=1
cd linux
git checkout v6.2
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- defconfig
# 开始编译
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- Image -j$(nproc)
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- modules -j$(nproc)

```
qemu-aia用的是6.10-rc1:
```
git clone https://github.com/torvalds/linux -b v6.10-rc1 --depth=1
cd linux
git checkout v6.10-rc1
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- defconfig
# 开始编译
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- Image -j$(nproc)
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- modules -j$(nproc)
```
# 制作ubuntu根文件系统
```
wget http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.2-base-riscv64.tar.gz
mkdir rootfs
dd if=/dev/zero of=riscv_rootfs.img bs=1M count=1024 oflag=direct
mkfs.ext4 riscv_rootfs.img
sudo mount -t ext4 riscv_rootfs.img rootfs/
sudo tar -xzf ubuntu-base-20.04.2-base-riscv64.tar.gz -C rootfs/

sudo cp /path-to-qemu/build/qemu-system-riscv64 rootfs/usr/bin/
sudo cp /etc/resolv.conf rootfs/etc/resolv.conf
sudo mount -t proc /proc rootfs/proc
sudo mount -t sysfs /sys rootfs/sys
sudo mount -o bind /dev rootfs/dev
sudo mount -o bind /dev/pts rootfs/dev/pts
sudo chroot rootfs 
# 进入chroot后，安装必要的软件包：
apt-get update
apt-get install git sudo vim bash-completion \
    kmod net-tools iputils-ping resolvconf ntpdate
exit

sudo umount rootfs/proc
sudo umount rootfs/sys
sudo umount rootfs/dev/pts
sudo umount rootfs/dev
sudo umount rootfs
```

# 运行hvisor
首先将[hvisor 代码仓库](https://github.com/syswonder/hvisor)拉到本地，并在 hvisor/platform/riscv64/BOARD/image 文件夹下（其中BOARD为要启动的类型），将之前编译好的根文件系统、Linux 内核镜像分别放在 virtdisk、kernel 目录下，并分别重命名为 rootfs1.ext4、Image。

> riscv版本还需要busybox文件，可以使用`qemu-img convert -f raw -O qcow2 rootfs1.ext4 rootfs-busybox.qcow2`生成

第二步，编译设备树文件，进入hvisor/platform/riscv64/BOARD/image/dts 目录，执行`make all`编译设备树

之后，在 hvisor 目录下，执行：`make run ARCH=riscv64 BOARD=qemu-plic`

或执行`make run ARCH=riscv64 BOARD=aia`开启AIA规范

如在 Qemu 入口处遇见 SIGTRAP 断点，请修改 `hvisor/platform/riscv64/BOARD/platform.mk` 文件，移除 `QEMU_ARGS` 中的 -S 选项

# 启动non-root linux
首先完成最新版本的 hvisor-tool 的编译。具体请参考[hvisor-tool](https://github.com/syswonder/hvisor-tool)的 README（是riscv64-linux-gnu-gcc工具链，通过ubuntu的apt直接安装即可）。例如，若要编译面向 riscv 的命令行工具，且 Hvisor 环境中的 Linux 镜像编译来源的源码位于 `~/linux`，则可执行

```
make all ARCH=riscv LOG=LOG_WARN KDIR=~/linux
```

> 请务必保证 Hvisor 中的Root Linux 镜像是由编译 hvisor-tool 时参数选项中的 Linux 源码目录编译产生。

编译完成后，将 output/hvisor.ko、output/hvisor复制到 hvisor/platform/riscv64/BOARD/image/virtdisk/rootfs1.ext4 根文件系统中启动 zone1 linux 的目录（例如/same_path/）；再将 zone1 的内核镜像（如果是与 zone0 相同的 Linux，复制一份 BOARD/image/kernel/Image 即可）、设备树（BOARD/image/dts/linux2.dtb）、配置文件（BOARD/configs/zone1-linux.json等）放在相同目录（/same_path/），并重命名为 Image、linux2.dtb、linux2.json等(需要根据.json里面内容进行命名文件名)。

之后需要为Zone1 linux制作一个根文件系统。可以将 BOARD/image/virtdisk 中的 rootfs1.ext4 复制一份，也可以重新制作根文件系统（最好改小镜像大小），并改名为 riscv_rootfs2.img(需要根据.json里面内容进行命名文件名)。之后将riscv_rootfs2.img放入rootfs1.ext4根文件系统中的相同目录（/same_path/）。


对于qemu-plic,启动root linux后，/home目录下执行
```bash
sudo insmod hvisor.ko
rm nohup.out
mkdir -p /dev/pts
mount -t devpts devpts /dev/pts
nohup ./hvisor zone start zone1-linux.json && cat nohup.out | grep "char device" && script /dev/null
```
对于qemu-aia,需启动virtio的后端
```
insmod hvisor.ko
mount -t proc proc /proc
mount -t sysfs sysfs /sys
rm nohup.out
mkdir -p /dev/pts
mount -t devpts devpts /dev/pts
nohup ./hvisor virtio start zone1-linux-virtio.json &
./hvisor zone start zone1_linux.json && \
cat nohup.out | grep "char device" && \
script /dev/null
```