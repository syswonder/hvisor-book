# Qemu ZCU102 hvisor 启动
任航麒(2572131118@qq.com)
## 安装 Petalinux
1. 安装 [Petalinux 2024.1](https://china.xilinx.com/support/download/index.html/content/xilinx/zh/downloadNav/embedded-design-tools/2024-1.html)
    请注意，本文以 2024.1 为例进行介绍，并不意味着其他版本不可以，只是其他版本未经验证，且测试中发现 Petalinux 对于操作系统有较强的依赖，请安装适合于自己操作系统的对应版本的 Petalinux.
2. 将下载好的 ```petalinux.run``` 文件放置到想要安装到的目录下，为其添加执行权限，之后直接 ```./petalinux.run``` 运行安装程序。
3. 安装程序会自动检测所需要的环境，如果不符合则会将缺失的环境提示出来，只需要对其一个个 ```apt insntall``` 即可。
4. 安装完成后每次使用 Petalinux 前需要进入安装目录，手动 ```source settings.sh``` 来添加环境变量，嫌麻烦将可以将该命令加入到 ```~/.bashrc``` 中
## 安装 ZCU102 BSP
1. 下载对应于 Petalinux 版本的 BSP，例子中是 [ZCU102 BSP 2024.1](https://china.xilinx.com/support/download/index.html/content/xilinx/zh/downloadNav/embedded-design-tools/2024-1.html)
2. 激活 Petalinux 环境，即在 Petalinux 安装目录中 ```source settings.sh```。
3. 基于 BSP 创建 Petalinux Project: ```petalinux-create -t project -s xilinx-zcu102-v2024.1-05230256.bsp```
4. 此时会创建一个 ```xilinx-zcu102-2024.1``` 文件夹，其中就有了 QEMU 模拟 ZCU102 所需的参数（设备树），以及预先编译好可以直接上板的 Linux 镜像、设备树、Uboot等。
## 编译 Hvisor
参照《在 Qemu 上运行 Hvisor》对编译 Hvisor 所需的环境进行配置，之后在 hvisor 目录下，执行：
```
make ARCH=aarch64 LOG=info BOARD=zcu102 cp
```
进行编译工作，目录下```/target/aarch64-unknown-none（可能不同）/debug/hvisor```，即为所需求的 hvisor 镜像。
## 准备设备树
### 使用现有设备树
在 Hvisor 的 image/devicetree 目录下，有 zcu102-root-aarch64.dts，其为已经经过测试用来启动RootLinux的设备树文件，对其进行编译即可。
```
dtc -I dts -O dtb -o zcu102-root-aarch64.dtb zcu102-root-aarch64.dts
```
如果 dtc 命令无效，则安装 device-tree-compiler。
```
sudo apt-get install device-tree-compiler
```
### 自行准备设备树
如果对设备有定制需求，则建议自行准备设备树，可以反编译 ZCU102 BSP 中的 ```pre-built/linux/images/system.dtb``` 获取完整设备树，基于 ```zcu102-root-aarch64.dts``` 进行增减。
## 准备镜像
### 使用现有镜像
建议直接使用 ZCU102 BSP 中的 ```pre-built/linux/images/Image``` 作为 Linux 内核在 ZCU102 上启动，其驱动配置完整。
### 自行编译
经过测试，linux 源码中 5.15 之前对于 ZYNQMP 的支持不全面，不建议自行编译时使用这之前的版本进行编译，在之后的版本进行编译时可以直接按照一般编译流程进行编译，因为源码对于 ZYNQMP 的基本支持默认开启。具体编译操作如下：
1. 访问 [linux-xlnx](https://github.com/Xilinx/linux-xlnx/tags?after=xilinx-v2023.1) 官网下载 Linux 源码，下载时最好下载 ```zynqmp-soc-for-v6.3```。
2. ```tar -xvf zynqmp-soc-for-v6.3``` 解压源码
3. 进入解压好的目录，执行下述命令使用默认配置，```make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig``` 
4. 进行编译：```make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image -j$(nproc)```
5. 编译完成后，目录中 ```arch/arm64/boot/Image``` 即为所需镜像。
## 启用 QEMU 仿真
1. 激活 Petalinux 环境，即在Petalinux 安装目录中 ```source settings.sh```。
2. 进入 ```xilinx-zcu102-2024.1``` 文件夹，使用下述命令即可在 QEMU仿真的 ZCU102上启动 hvisor，其中的文件路径需要按照自己的实际情况进行修改。
```
# QEMU 参数传递
petalinux-boot --qemu --prebuilt 2 --qemu-args '-device loader,file=hvisor,addr=0x40400000,force-raw=on -device loader,
file=zcu102-root-aarch64.dtb,addr=0x40000000,force-raw=on -device loader,file=zcu102-root-aarch64.dtb,addr=0x04000000,
force-raw=on -device loader,file=/home/hangqi-ren/Image,addr=0x00200000,force-raw=on -drive if=sd,format=raw,index=1,
file=rootfs.ext4' 
# 启动 hvisor
bootm 0x40400000 - 0x40000000
```
