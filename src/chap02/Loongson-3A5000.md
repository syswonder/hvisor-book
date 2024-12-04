# 在龙芯3A5000主板（7A2000）上启动hvisor

韩喻泷 <enkerewpo@hotmail.com>

更新时间：2024.12.4

## 第一步：获取hvisor源码并编译

克隆代码到本地：

```bash
git clone -b dev-loongarch https://github.com/syswonder/hvisor # dev-loongarch分支
make ARCH=loongarch64
```
编译完成后在target目录下可以找到strip之后的hvisor.bin（编译输出的最后一行会显示文件路径）。

## 获取vmlinux.bin镜像

请从<https://github.com/enkerewpo/linux-hvisor-loongarch64/releases>下载最新发布的hvisor默认龙芯linux镜像（包括root linux kernel+root linux dtb+root linux rootfs，其中root linux rootfs中包括non root linux+nonroot linux dtb+nonroot linux rootfs）。如果你需要自行编译linux kernel以及rootfs，可参考该仓库的`arch/loongarch`目录中hvisor相关设备树以及我为3A5000移植的buildroot环境（<https://github.com/enkerewpo/buildroot-loongarch64>）。如果你需要手动编译hvisor-tool，请参考<https://github.com/enkerewpo/hvisor-tool>，关于所有环境的编译顺序和脚本调用流程请参考`Makefile.1`文件中`world`目标内的代码（<https://github.com/enkerewpo/hvisor_uefi_packer/blob/main/Makefile.1>），并通过运行`./make_world`脚本编译所有东西，如果你需要手动编译这些，则需要在Makefile.1内修改对应的代码路径变量，包括：

```
HVISOR_LA64_LINUX_DIR = ../hvisor-la64-linux
BUILDROOT_DIR = ../buildroot-loongarch64
HVISOR_TOOL_DIR = ../hvisor-tool
```

然后运行 `./make_world`，请注意，第一次编译linux和buildroot的时间可能相当长（可能长达几十分钟，取决于你的机器性能）。

## 获取hvisor UEFI Image Packer

由于3A5000以及之后的3系CPU的主板均采用UEFI启动，所以只能通过efi镜像的方法启动hvisor，克隆仓库<https://github.com/enkerewpo/hvisor_uefi_packer>到本地：

```bash
make menuconfig # 配置为你本地的loongarch64 gcc工具链前缀、hvisor.bin路径、vmlinux.bin路径
# 修改make_image中的HVISOR_SRC_DIR=../hvisor为你实际保存hvisor源码的路径，之后再运行脚本
./make_image
# 得到 BOOTLOONGARCH64.EFI 文件
```

得到的`BOOTLOONGARCH64.EFI`必须放在U盘的第一个FAT32分区的`/EFI/BOOT/BOOTLOONGARCH64.EFI`位置。然后插入U盘启动即可进入hvisor并自动启动root linux。

由于root linux相关的元信息（加载地址，内存区域等）硬编码在hvisor源码中（`src/platform/ls3a5000_loongarch64.rs`），如果你是手动编译linux内核，则需要修改这里的配置再重新编译hvisor。

## 上板启动

主板上电开机，按 **F12** 进入UEFI Boot Menu，选择你插入的U盘后回车，会自动启动hvisor，并进入root linux的bash环境。

## 启动nonroot

如果你使用的是release中提供的相关镜像，启动后在root linux的bash内输入：

```bash
./daemon.sh
./linux2_virtio.sh
```

之后会自动启动nonroot（一些相关配置文件位于root linux的`/tool`目录内，包括提供给hvisor-tool的nonroot zone配置json以及virtio配置json文件），之后回自动打开一个screen进程连接nonroot linux的virtio-console，你会看到一个打印了nonroot字样的bash出现，你可以在使用screen时按CTRL+A D快捷键detach（请记住显示的screen session名称），此时会返回root linux，如果希望返回nonroot linux，则运行

```bash
screen -r {刚才的session全名 或者 只输入最前面的数字}
```

之后会返回nonroot linux的bash。