# Board ZCU102 hvisor 多模式启动
## 在 ZCU102 开发板 SD mode 下启动 Hvisor 
### 准备 SD 卡
1. 准备一块标准 SD 卡，对其进行分区，一块为 Boot 分区（FAT32），其余为文件系统分区（EXT4），windows 分区可以使用 [DiskGenius](https://www.diskgenius.cn/download.php)，Linux 分区可以使用 [fdisk](https://www.cnblogs.com/renshengdezheli/p/13941563.html)、[mkfs](https://blog.csdn.net/linkedin_35878439/article/details/82020925)
2. 准备一个文件系统，将其内容拷贝到任一文件系统分区中，可以参考 《NXPIMX8》 制作 Ubuntu 文件系统、也可以直接使用 ZCU102 BSP 中的文件系统。
3. 将 ```zcu102-root-aarch64.dtb、Image、hvisor``` 拷贝到 Boot 分区中。
4. 在 SD mode 下，需要提供从 SD卡中提供 ATF、Uboot，因此将 ZCU102 BSP 中```pre-built/linux/images/boot.scr 和 BOOT.BIN``` 拷贝到 BOOT 分区中。
#### 启动 ZCU102
1. 将 ZCU102 设置为 SD mode，插入 SD 卡，连接串口，上电
2. 输入任意按键打断 Uboot 自动脚本执行，运行下述命令启动 hvisor 及 root linux: 
```
fatload mmc 0:1 0x40400000 hvisor;fatload mmc 0:1 0x40000000 zcu102-root-aarch64.dtb
fatload mmc 0:1 0x04000000 zcu102-root-aarch64.dtb;fatload mmc 0:1 0x00200000 Image;bootm 0x40400000 - 0x40000000
```
3. 如果成功启动，将可以在串口看到 hvisor 信息以及 linux 信息，最终进入文件系统。

## 在 ZCU102 开发板 Jtag mode 下启动 Hvisor


首先将板子附带的两个线缆连接到板子的 JTAG 和 UART 接口上，另一端通过 USB 连接到 PC。

然后在命令行打开一个 petalinux 工程，确保工程已经编译过并生成了对应的启动文件（vmlinux、BOOT.BIN等），之后进入工程根目录运行 [1]：

```bash
petalinux-boot --jtag --prebuilt 3
```

其中prebuilt代表启动的层次：

- **Level 1**: 只下载FPGA bitstream，启动 FSBL 和 PMUFW
- **Level 2**: 下载FPGA bitstream 并启动 UBOOT，并启动 FSBL、PMUFW 和 TF-A（Trusted Firmware-A [2]）
- **Level 3**: 下载并启动 linux，并加载或启动FPGA bitstream、FSBL、PMUFW、TF-A、UBOOT

之后 JTAG 会通过 JTAG 线把对应的文件下载到板子上（保存到指的内存地址），并启动对应的 bootloader，具体官方的 UBOOT 默认脚本参见工程镜像目录的 boot.scr 文件。

请通过另一个 UART 线观察 ZCU102 板子的输出（包括FSBL、UBOOT、Linux等输出），可以通过 screen/gtkterm/termius/minicom 等串口工具查看。

<div class="warning">
    <h3>请注意</h3>
    <p> 由于 petalinux 规定了一些固定内存地址，如 linux kernel、fitImage、DTB 的默认加载地址（可在 petalinux 编译工程时配置），由于我们需要加载启动自制的 fitImage，目前发现的问题是如果 root linux dtb 在 its 中所写的加载地址和 petalinux 编译时的加载地址一致，会导致该 dtb 被覆盖为默认的 petalinux dtb，从而导致root linux接受到错误的 dtb 而无法启动。因此需要在编译时指定和 petalinux 默认 dtb/fitImage 加载地址不同的地址，以防止出现其他问题。
</div>

# 参考文献

[1] PetaLinux Tools Documentation: Reference Guide (UG1144).<https://docs.amd.com/r/2023.1-English/ug1144-petalinux-tools-reference-guide/Booting-a-PetaLinux-Image-on-Hardware-with-JTAG>
[2] Trusted Firmware-A Documentation.<https://trustedfirmware-a.readthedocs.io/en/latest/>