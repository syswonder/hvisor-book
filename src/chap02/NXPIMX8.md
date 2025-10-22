# 在NXP-IMX8MP上启动hvisor


## 1. 下载厂商提供的linux源码

[https://pan.baidu.com/s/1XimrhPBQIG5edY4tPN9_pw?pwd=kdtk](https://pan.baidu.com/s/1XimrhPBQIG5edY4tPN9_pw?pwd=kdtk)

提取码：kdtk

进入`Linux/源码/`目录下，下载`OK8MP-linux-sdk.tar.bz2.0*`3个压缩包，下载完成后，执行：

```
cd Linux/sources

# 合并分卷压缩包
cat OK8MP-linux-sdk.tar.bz2.0* > OK8MP-linux-sdk.tar.bz2

# 解压合并的压缩包
tar -xvjf OK8MP-linux-sdk.tar.bz2

```

解压后，`OK8MP-linux-kernel`目录就是linux源码目录。

## 2. linux源码编译

### 安装交叉编译工具

1. 下载交叉编译工具链：

   ```bash
   wget https://armkeil.blob.core.windows.net/developer/Files/downloads/gnu-a/10.3-2021.07/binrel/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz
   ```

2. 解压工具链：

   ```bash
   tar xvf gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz
   ```

3. 添加路径，使 `aarch64-none-linux-gnu-*` 可以直接使用，修改 `~/.bashrc` 文件：

   ```bash
   echo 'export PATH=$PWD/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin:$PATH' >> ~/.bashrc
   source ~/.bashrc
   ```

### 编译linux

1. 切换到 Linux 内核源码目录：

   ```bash
   cd Linux/sources/OK8MP-linux-sdk
   ```

2. 执行编译命令：

   ```makefile
   # 设置 Linux 内核配置
   make OK8MP-C_defconfig ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu-
   
   # 编译 Linux 内核
   make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- Image -j$(nproc)
   
   # 复制编译后的镜像到 tftp 目录
   cp arch/arm64/boot/Image ~/tftp/
   ```

这里建立一个tftp目录，方便之后对镜像整理，也方便附录中使用tftp传输镜像。

## 3. 制作sd卡

1. 将SD卡插入读卡器，并连接至主机。

2. 切换至Linux/Images目录。

3. 执行以下命令，进行分区：

   ```bash
   fdisk <$DRIVE>
   d  # 删除所有分区
   n  # 创建新分区
   p  # 选择主分区
   1  # 分区编号为1
   16384  # 起始扇区
   t  # 更改分区类型
   83  # 选择Linux文件系统（ext4）
   w  # 保存并退出
   ```

4. 将启动文件写入SD卡启动盘：

   ```bash
   dd if=imx-boot_4G.bin of=<$DRIVE> bs=1K seek=32 conv=fsync
   ```

5. 格式化SD卡启动盘的第一个分区为ext4格式：

   ```bash
   mkfs.ext4 <$DRIVE>1
   ```

6. 将SD卡读卡器拔出，重新连接。将根文件系统rootfs.tar解压到SD卡1号分区，rootfs.tar可以自行参考[qemu-aarch64](https://hvisor.syswonder.org/chap02/QemuAArch64.html)制作，也可以使用下面的镜像。

   ```bash
   tar -xvf rootfs.tar -C <path/to/mounted/SD/card/partition>
   ```

rootfs.tar下载地址：

```
https://disk.pku.edu.cn/link/AADFFFE8F568DE4E73BE24F5AED54B00EB
文件名：rootfs.tar
```

7. 完成后，弹出SD卡。

## 4. 编译hvisor

1. 整理配置文件

将配置文件放到该放的地方，配置文件样例可以参考[这里](https://github.com/syswonder/hvisor-tool/tree/main/examples/nxp-aarch64/gpu_on_root)。

2. 编译hvisor

进入hvisor目录，切换到main分支或dev分支，执行编译命令：

```makefile
make ARCH=aarch64 FEATURES=platform_imx8mp,gicv3 LOG=info all

# 将编译后的hvisor镜像放入tftp
make cp
```

## 5. 启动hvisor和root linux

启动NXP板子之前，需要将tftp目录下的文件放到sd卡，比如放到sd卡的/home/arm64目录下，tftp目录下的文件包括：

* Image：root linux镜像，也可以用作non root linux镜像
* linux1.dtb, linux2.dtb：root linux和non root linux的设备树
* hvisor.bin：hvisor镜像
* OK8MP-C.dtb：这个用于uboot启动时做一些检查，本质没什么用，可以从这里获取[OK8MP-C.dts](https://github.com/KouweiLee/tftp/blob/82545a7c83460747056ca35022de94c2ea365d29/OK8MP-C.dts)

启动NXP板子：

1. 调整拨码开关以启用SD卡启动模式：(1,2,3,4) = (ON,ON,OFF,OFF)。
2. 将SD卡插入SD插槽。
3. 使用串口线将开发板与主机相连。
4. 通过终端软件打开串口

启动NXP板子后，串口应该有输出，重启开发板，立刻按下空格保持不懂，使uboot进入命令行终端，执行如下命令：

```
setenv loadaddr 0x40400000; setenv fdt_addr 0x40000000; setenv zone0_kernel_addr 0xa0400000; setenv zone0_fdt_addr 0xa0000000; ext4load mmc 1:1 ${loadaddr} /home/arm64/hvisor.bin; ext4load mmc 1:1 ${fdt_addr} /home/arm64/OK8MP-C.dtb; ext4load mmc 1:1 ${zone0_kernel_addr} /home/arm64/Image; ext4load mmc 1:1 ${zone0_fdt_addr} /home/arm64/linux1.dtb; bootm ${loadaddr} - ${fdt_addr};
```

执行后，hvisor应该就启动并自动进入root linux了。

## 6. 启动non root linux

启动non root linux需要用到hvisor-tool。具体请参考[hvisor-tool](https://github.com/syswonder/hvisor-tool)的 README。

## 附. 使用tftp便捷传输镜像

tftp方便开发板与主机间的数据传输，不需要每次插拔sd卡。具体步骤如下：

### 对于ubuntu系统

如果你使用的是ubuntu系统，则依次执行：

1. 安装 TFTP 服务器软件包

   ```bash
   sudo apt-get update
   sudo apt-get install tftpd-hpa tftp-hpa
   ```

2. 配置 TFTP 服务器

   创建 TFTP 根目录并设置权限：

   ```bash
   mkdir -p ~/tftp
   sudo chown -R $USER:$USER ~/tftp
   sudo chmod -R 755 ~/tftp
   ```

   编辑 tftpd-hpa 配置文件：

   ```bash
   sudo nano /etc/default/tftpd-hpa
   ```

   修改如下：

   ```plaintext
   # /etc/default/tftpd-hpa
   
   TFTP_USERNAME="tftp"
   TFTP_DIRECTORY="/home/<your-username>/tftp"
   TFTP_ADDRESS=":69"
   TFTP_OPTIONS="-l -c -s"
   ```

   将 `<your-username>` 替换为实际用户名。

3. 启动/重启 TFTP 服务

   ```bash
   sudo systemctl restart tftpd-hpa
   ```

4. 验证 TFTP 服务器

   ```bash
   echo "TFTP Server Test" > ~/tftp/testfile.txt
   ```

   ```bash
   tftp localhost
   tftp> get testfile.txt
   tftp> quit
   cat testfile.txt
   ```

   若显示 "TFTP Server Test"，则 TFTP 服务器工作正常。

5. 配置开机启动：

   ```
   sudo systemctl enable tftpd-hpa
   ```

6. 使用网线将开发板的网口（共有两个，请选择下方的一个）与主机连接。并配置主机有线网卡，ip：192.169.137.2, netmask: 255.255.255.0。

之后启动开发板，进入uboot命令行后，执行命令变为：

```
setenv serverip 192.169.137.2; setenv ipaddr 192.169.137.3; setenv loadaddr 0x40400000; setenv fdt_addr 0x40000000; setenv zone0_kernel_addr 0xa0400000; setenv zone0_fdt_addr 0xa0000000; tftp ${loadaddr} ${serverip}:hvisor.bin; tftp ${fdt_addr} ${serverip}:OK8MP-C.dtb; tftp ${zone0_kernel_addr} ${serverip}:Image; tftp ${zone0_fdt_addr} ${serverip}:linux1.dtb; bootm ${loadaddr} - ${fdt_addr};
```

解释:

- `setenv serverip 192.169.137.2`：设置tftp服务器的IP地址。
- `setenv ipaddr 192.169.137.3`：设置开发板的IP地址。
- `setenv loadaddr 0x40400000`：设置hvisor镜像的加载地址。
- `setenv fdt_addr 0x40000000`：设置设备树文件的加载地址。
- `setenv zone0_kernel_addr 0xa0400000`：设置guest Linux镜像的加载地址。
- `setenv zone0_fdt_addr 0xa0000000`：设置root Linux的设备树文件的加载地址。
- `tftp ${loadaddr} ${serverip}:hvisor.bin`：从tftp服务器下载hvisor镜像到hvisor的加载地址。
- `tftp ${fdt_addr} ${serverip}:OK8MP-C.dtb`：从tftp服务器下载设备树文件到设备树文件的加载地址。
- `tftp ${zone0_kernel_addr} ${serverip}:Image`：从tftp服务器下载guest Linux镜像到guest Linux镜像的加载地址。
- `tftp ${zone0_fdt_addr} ${serverip}:linux1.dtb`：从tftp服务器下载root Linux的设备树文件到root Linux的设备树文件的加载地址。
- `bootm ${loadaddr} - ${fdt_addr}`：启动hvisor，加载hvisor镜像和设备树文件。

### 对于windows系统

可以参考这篇文章：
[https://blog.csdn.net/qq_52192220/article/details/142693036](https://blog.csdn.net/qq_52192220/article/details/142693036)
