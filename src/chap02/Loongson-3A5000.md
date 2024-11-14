# 在龙芯3A5000主板（7A2000）上启动hvisor

韩喻泷 <enkerewpo@hotmail.com>

更新时间：2024.11.6

## 第一步：获取hvisor源码并编译

克隆代码到本地：

```bash
git clone -b dev-loongarch https://github.com/syswonder/hvisor # dev-loongarch分支
make ARCH=loongarch64 -j$(nproc)
```
编译完成后在target目录下可以找到strip之后的hvisor.bin（编译输出的最后一行会显示文件路径）。

## 获取vmlinux.bin镜像

请从<https://github.com/enkerewpo/linux-hvisor-loongarch64/releases>下载最新发布的hvisor默认龙芯linux镜像（包括root linux kernel+root linux dtb+root linux rootfs，其中root linux rootfs中包括non root linux+nonroot linux dtb+nonroot linux rootfs）。如果你需要自行编译linux kernel以及rootfs，可参考该仓库的arch/loongarch目录中hvisor相关设备树以及其他相关代码。

## 获取hvisor UEFI Image Packer

由于3A5000采用UEFI启动，所以只能通过efi镜像的方法启动hvisor，克隆仓库<https://github.com/enkerewpo/hvisor_uefi_packer>到本地：

```bash
make menuconfig # 配置为你本地的loongarch64 gcc工具链前缀、hvisor.bin路径、vmlinux.bin路径
./make_image
# 得到 BOOTLOONGARCH64.EFI 文件
```

得到的`BOOTLOONGARCH64.EFI`必须放在U盘的第一个FAT32分区的`/EFI/BOOT/BOOTLOONGARCH64.EFI`位置。然后插入U盘启动即可进入hvisor并自动启动root linux。

由于root linux相关的元信息（加载地址，内存区域等）硬编码在hvisor源码中（`src/platform/ls3a5000_loongarch64.rs`），如果你是手动编译linux内核，则需要修改这里的配置再重新编译hvisor。

## 上板启动

主板上电开机，按 **F12** 进入UEFI Boot Menu，选择你插入的U盘后回车，会自动启动hvisor。