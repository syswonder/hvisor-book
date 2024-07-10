# FPGA zcu102

Author: 杨竣轶(Jerry) github.com/comet959

```shell
# Before, Install vivado 2022.2 software
# Ubuntu 20.04 can work fine
sudo apt update

git clone https://github.com/U-interrupt/uintr-rocket-chip.git
cd uintr-rocket-chip
git submodule update --init --recursive
export RISCV=/opt/riscv64
git checkout 98e9e41
vim digilent-vivado-script/config.ini # Env Config

make checkout
make clean
make build

# Use vivado to open the vivado project, then change the top file, run synthesis, run implementation, generate bitstream.
# Connect the zcu102 - Jtag and Uart on your PC.
# Use dd command to flash the image include boot and rootfs part.
# Change the boot button mode to (On Off Off Off)
# Boot the power.

sudo screen /dev/ttyUSB0 115200 # Aarch64 Core Uart
sudo screen /dev/ttyUSB2 115200 # Riscv Core Uart

# On /dev/ttyUSB0
cd uintr-rocket-chip
./load-and-reset.sh

# Focus on ttyUSB2, then you will see the Riscv Linux Boot Msg.

```



## 在RocketChip中开启H扩展

```shell
vim path/to/repo/common/src/main/scala/Configs.scala
```

```scala
// change
class UintrConfig extends Config(
  new WithNBigCores(4) ++
    new WithNExtTopInterrupts(6) ++
    new WithTimebase((BigInt(10000000))) ++ // 10 MHz
    new WithDTS("freechips.rocketchip-unknown", Nil) ++
    new WithUIPI ++
    new WithCustomBootROM(0x10000, "../common/boot/bootrom/bootrom.img") ++
    new WithDefaultMemPort ++
    new WithDefaultMMIOPort ++
    new WithDefaultSlavePort ++
    new WithoutTLMonitors ++
    new WithCoherentBusTopology ++
    new BaseSubsystemConfig
)

// to

class UintrConfig extends Config(
  new WithHypervisor ++
  new WithNBigCores(4) ++
    new WithNExtTopInterrupts(6) ++
    new WithTimebase((BigInt(10000000))) ++ // 10 MHz
    new WithDTS("freechips.rocketchip-unknown", Nil) ++
    new WithUIPI ++
    new WithCustomBootROM(0x10000, "../common/boot/bootrom/bootrom.img") ++
    new WithDefaultMemPort ++
    new WithDefaultMMIOPort ++
    new WithDefaultSlavePort ++
    new WithoutTLMonitors ++
    new WithCoherentBusTopology ++
    new BaseSubsystemConfig
)

```
