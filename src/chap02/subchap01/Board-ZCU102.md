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




