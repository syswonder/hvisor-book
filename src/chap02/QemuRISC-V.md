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
# 安装必要工具
sudo apt-get install autoconf automake autotools-dev curl python3 python3-pip libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev ninja-build git cmake libglib2.0-dev libslirp-dev

git clone https://github.com/riscv/riscv-gnu-toolchain
cd riscv-gnu-toolchain
git rm qemu 
git submodule update --init --recursive
#上述操作会占用超过5GB以上磁盘空间
# 如果git报网络错误，可以执行：
git config --global http.postbuffer 524288000
```
之后开始编译工具链：
```
cd riscv-gnu-toolchain
mkdir build
cd build
../configure --prefix=/opt/riscv64
sudo make linux -j $(nproc)
# 编译完成后，将工具链加入环境变量
echo 'export PATH=/opt/riscv64/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```
这样就得到了 riscv64-unknown-linux-gnu工具链。
# 编译Linux
```
git clone https://github.com/torvalds/linux -b v6.2 --depth=1
cd linux
git checkout v6.2
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- defconfig
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- modules -j$(nproc)
# 开始编译
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- Image -j$(nproc)

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
将做好的根文件系统、Linux内核镜像放在hvisor目录下的指定位置，在hvisor根目录下执行`make run ARCH=riscv64`即可

默认情况下使用 PLIC，执行`make run ARCH=riscv64 IRQ=aia`开启AIA规范

# 启动non-root linux
使用 hvisor-tool 生成hvisor.ko文件，之后在 QEMU 上即可通过 root linux-zone0 启动 zone1-linux。

启动root linux后，/home目录下执行
```bash
sudo insmod hvisor.ko
rm nohup.out
mkdir -p /dev/pts
mount -t devpts devpts /dev/pts
nohup ./hvisor zone start linux2-aia.json && cat nohup.out | grep "char device" && script /dev/null
```
