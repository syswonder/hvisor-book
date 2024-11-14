#  qemu bosc-kmh hvisor
## hvisor 更改为单核
qemu参数smp改为1

const.rs 中的 MAX_CPU_NUM改为1

qemu_riscv64.rs 中的 
```pub const ROOT_ZONE_CPUS: u64 = (1 << 0);```

## 修改： hvisor的二阶段映射
```rs
//src/arch/riscv64/s2pt.rs
        attr |= Self::VALID | Self::USER | Self::ACCESSED | Self::DIRTY;//stage 2 page table must user
```

## 编译linux内核

```bash
git clone git@gitlab.bosoc.cc:openxiangshan/riscv-linux.git	# 下载源码仓库
git checkout -b devel origin/devel	# 切换分支到devel
# 注意: 发人员请基于上述主开发分支devel最新的HEAD进行开始工作
export CROSS_COMPILE=riscv64-unknown-linux-gnu-	# 指定编译器文件前缀
export ARCH=riscv  # 指定架构

export PATH=$PATH:/home/ran/toolchain/gcc12/riscv-toolchain-20230425/bin # 工具链路径  
make distclean	# 清理旧的编译产物, 此命令会导致重新编译所有文件, 请酌情使用
make defconfig xiangshan.config # 编译生成.config
gedit .config
CONFIG_BLK_DEV_INITRD=y
CONFIG_INITRAMFS_SOURCE="~/fdisk/kvm/rootfs_kvm_riscv64.cpio"
make
```
## 包含文件系统的 Image 打包到 hvisor.bin 中​
打包完成后的fw_payload.bin有250M，更改内存布局缩小到78M
```rs
// config
#[link_section = ".img1"]
#[used]
pub static GUEST1_KERNEL: [u8; include_bytes!("../images/riscv64/kernel/Image").len()] =
    *include_bytes!("../images/riscv64/kernel/Image");
#[link_section = ".dtb1"]
#[used]
pub static GUEST1_DTB: [u8; include_bytes!("../images/riscv64/devicetree/linux.dtb").len()] =
    *include_bytes!("../images/riscv64/devicetree/linux.dtb");
```
```ld
    . = . + 0x2000000;
    gdtb = .;
    . = 0x82000000;
    .dtb1 : {
        KEEP(*(.dtb1))
    }
    . = ALIGN(4K);
    . = 0x83000000;
    .img1 : {
        KEEP(*(.img1))
    }
```
按照linker脚本和kmh_v2_1core.dtb的内容，修改内存布局，PLIC，串口配置
```rs
//src/platform/qemu_riscv64.rs
pub const ROOT_ZONE_DTB_ADDR: u64 = 0x82000000;
pub const ROOT_ZONE_KERNEL_ADDR: u64 = 0x83000000;
pub const ROOT_ZONE_ENTRY: u64 = 0x83000000;
pub const ROOT_ZONE_CPUS: u64 = 1 << 0;
...
pub const ROOT_ZONE_MEMORY_REGIONS: [HvConfigMemoryRegion; 2] = [
    HvConfigMemoryRegion {
        mem_type: MEM_TYPE_RAM,
        physical_start: 0x81000000,
        virtual_start: 0x81000000,
        size: 0x08000000,
    }, // ram
    HvConfigMemoryRegion {
        mem_type: MEM_TYPE_IO,
        physical_start: 0x310b0000,
        virtual_start: 0x310b0000,
        size: 0x10000,
    }, // serial
];
```


## 将hvisor.bin 嵌入opensbi fw_payload
FPGA上运行时，使用打包好的软件镜像opensbi fw_payload运行
```bash
	cd ~/source/opensbi && \
	make PLATFORM=generic \
    	FW_PAYLOAD_PATH=/home/dorso/work/hvisor/target/riscv64gc-unknown-none-elf/debug/hvisor.bin \
		FW_FDT_PATH=/home/dorso/source/opensbi/kmh-v2-1core.dtb
```

## qemu bosc-kmh运行
```bash
QEMU := ~/source/qemu-devel/build/qemu-system-riscv64
# bosc的cpu不手动指定
QEMU_ARGS := -machine bosc-kmh
QEMU_ARGS += -nographic
QEMU_ARGS += -smp 1
QEMU_ARGS += -m 2G
# 指定制作好的fw_payload作为bios
QEMU_ARGS += -bios ~/source/opensbi/build/platform/generic/firmware/fw_payload.elf
```
## FPGA 运行 hvisor
在opensbi目录下
```bash
./bin2fpgadata -i build/platform/generic/firmware/fw_payload.bin
```
执行成功后生成的软件镜像文件data.txt位于当前源码根目录

```bash
source /home/tools/Xilinx/Vivado/2020.2/settings64.sh
# 确认与FPGA相连的Linux 服务器已经通过上述source命令并执行了hw_server命令以启动相关服务, 然后本x86_64 Linux 电脑将使用下述命令中tcl脚本与后台服务器建立通信
vivado -mode batch -source ../onboard-ai1-fpga81-remote.tcl -tclargs <path to bitstream files>/  ./data.txt
```
其中<path to bitstream files>为使用的硬件镜像路径
