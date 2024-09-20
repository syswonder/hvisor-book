# 在QEMU上运行hvisor

## 一、安装交叉编译器 aarch64-none-linux-gnu- 10.3

网址：[https://developer.arm.com/downloads/-/gnu-a](https://developer.arm.com/downloads/-/gnu-a) 

工具选择：AArch64 GNU/Linux target (aarch64-none-linux-gnu) 

下载链接：[https://developer.arm.com/-/media/Files/downloads/gnu-a/10.3-2021.07/binrel/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz?rev=1cb9c51b94f54940bdcccd791451cec3&hash=B380A59EA3DC5FDC0448CA6472BF6B512706F8EC](https://developer.arm.com/-/media/Files/downloads/gnu-a/10.3-2021.07/binrel/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz?rev=1cb9c51b94f54940bdcccd791451cec3&hash=B380A59EA3DC5FDC0448CA6472BF6B512706F8EC) 

```bash
wget https://armkeil.blob.core.windows.net/developer/Files/downloads/gnu-a/10.3-2021.07/binrel/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz
tar xvf gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz
ls gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/
```

安装完成，记住路径，例如在：/home/tools/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-，之后都会使用这个路径。

## 二、编译安装QEMU 7.2.12

```
# 安装编译所需的依赖包
sudo apt install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev \
              gawk build-essential bison flex texinfo gperf libtool patchutils bc \
              zlib1g-dev libexpat-dev pkg-config  libglib2.0-dev libpixman-1-dev libsdl2-dev \
              git tmux python3 python3-pip ninja-build
# 下载源码
wget https://download.qemu.org/qemu-7.2.12.tar.xz 
# 解压
tar xvJf qemu-7.2.12.tar.xz   
cd qemu-7.2.12
#生成设置文件
./configure --enable-kvm --enable-slirp --enable-debug --target-list=aarch64-softmmu,x86_64-softmmu  
#编译
make -j$(nproc)   
```

之后编辑 `~/.bashrc` 文件，在文件的末尾加入几行：

```
# 请注意，qemu-7.2.12 的父目录可以随着你的实际安装位置灵活调整
export PATH=$PATH:/path/to/qemu-7.2.12/build
```

随后即可在当前终端 `source ~/.bashrc` 更新系统路径，或者直接重启一个新的终端。此时可以确认qemu版本：

```
qemu-system-aarch64 --version   #查看版本
```

> 注意，上述依赖包可能不全，例如：
>
> - 出现 `ERROR: pkg-config binary 'pkg-config' not found` 时，可以安装 `pkg-config` 包；
> - 出现 `ERROR: glib-2.48 gthread-2.0 is required to compile QEMU` 时，可以安装 `libglib2.0-dev` 包；
> - 出现 `ERROR: pixman >= 0.21.8 not present` 时，可以安装 `libpixman-1-dev` 包。

## 三、编译Linux Kernel 5.4

在编译root linux的镜像前, 在.config文件中把CONFIG_IPV6和CONFIG_BRIDGE的config都改成y, 以支持在root linux中创建网桥和tap设备。具体操作如下：

```
git clone https://github.com/torvalds/linux -b v5.4 --depth=1
cd linux
git checkout v5.4
make ARCH=arm64 CROSS_COMPILE=/root/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu- defconfig
# 在.config中增加一行
CONFIG_BLK_DEV_RAM=y
# 修改.config的两个CONFIG参数
CONFIG_IPV6=y
CONFIG_BRIDGE=y
# 编译
make ARCH=arm64 CROSS_COMPILE=/root/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu- Image -j$(nproc)
```

> 如果编译linux时报错：
>
> ```
> /usr/bin/ld: scripts/dtc/dtc-parser.tab.o:(.bss+0x20): multiple definition of `yylloc'; scripts/dtc/dtc-lexer.lex.o:(.bss+0x0): first defined here
> ```
>
> 则修改linux文件夹下`scripts/dtc/dtc-lexer.lex.c`，在`YYLTYPE yylloc;`前增加`extern`。再次编译，发现会报错：openssl/bio.h: No such file or directory ，此时执行`sudo apt install libssl-dev`

编译完毕，内核文件位于：arch/arm64/boot/Image。记住整个linux文件夹所在的路径，例如：home/korwylee/lgw/hypervisor/linux。

## 四、基于ubuntu 20.04 arm64 base构建文件系统

我们使用ubuntu 20.04（22.04也可以）来构建根文件系统。

下载：[ubuntu-base-20.04.5-base-arm64.tar.gz](http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.5-base-arm64.tar.gz)  

链接：[http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.5-base-arm64.tar.gz](http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.5-base-arm64.tar.gz)

```bash
wget http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.5-base-arm64.tar.gz

mkdir rootfs
# 创建一个1G大小的ubuntu.img，可以修改count修改img大小
dd if=/dev/zero of=ubuntu-20.04-rootfs_ext4.img bs=1M count=1024 oflag=direct
mkfs.ext4 ubuntu-20.04-rootfs_ext4.img
# 将ubuntu.tar.gz放入已经挂载到rootfs上的ubuntu.img中
sudo mount -t ext4 ubuntu-20.04-rootfs_ext4.img rootfs/
sudo tar -xzf ubuntu-base-20.04.5-base-arm64.tar.gz -C rootfs/

# 让rootfs绑定和获取物理机的一些信息和硬件
# qemu-path为你的qemu路径
sudo cp qemu-path/build/qemu-system-aarch64 rootfs/usr/bin/ 
sudo cp /etc/resolv.conf rootfs/etc/resolv.conf
sudo mount -t proc /proc rootfs/proc
sudo mount -t sysfs /sys rootfs/sys
sudo mount -o bind /dev rootfs/dev
sudo mount -o bind /dev/pts rootfs/dev/pts

# 执行该指令可能会报错，请参考下面的解决办法
sudo chroot rootfs 
sudo apt-get install git sudo vim bash-completion \
		kmod net-tools iputils-ping resolvconf ntpdate
		
adduser arm64
adduser arm64 sudo
echo "kernel-5_4" >/etc/hostname
echo "127.0.0.1 localhost" >/etc/hosts
echo "127.0.0.1 kernel-5_4">>/etc/hosts
dpkg-reconfigure resolvconf
dpkg-reconfigure tzdata
exit

sudo umount rootfs/proc
sudo umount rootfs/sys
sudo umount rootfs/dev/pts
sudo umount rootfs/dev
sudo umount rootfs
```

最后卸载挂载，完成根文件系统的制作。

> 执行`sudo chroot .`时，如果报错`chroot: failed to run command ‘/bin/bash’: Exec format error`，可以执行指令：
>
> ```
> sudo apt-get install qemu-user-static
> sudo update-binfmts --enable qemu-aarch64
> ```

## 五、Rust环境配置

请参考：[Rust语言圣经](https://course.rs/first-try/intro.html)

## 六、编译和运行hvisor

首先将[hvisor代码仓库](https://github.com/KouweiLee/hvisor)拉到本地，并在hvisor/images/aarch64文件夹下，将之前编译好的根文件系统、Linux内核镜像分别放在virtdisk、kernel目录下，并分别重命名为rootfs1.ext4、Image。并在devicetree目录下，执行`make all`。

之后，在hvisor目录下，执行：

```
make ARCH=aarch64 LOG=info FEATURES=platform_qemu run
```

之后会进入uboot启动界面，该界面下执行：

```
bootm 0x40400000 - 0x40000000
```

该启动命令会从物理地址`0x40400000`启动hvisor，设备树的地址为`0x40000000`。hvisor启动时，会自动启动root linux（用于管理的Linux），并进入root linux的shell界面。
