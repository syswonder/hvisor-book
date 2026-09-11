# 基于 RK3588 的 矽慧通 X3300 快速上手

本文档旨在为开发者详细介绍在 矽慧通 X3300 开发板上部署并启动 hvisor 的完整流程。

操作环境基于 Ubuntu 22.04 主机。我们假定已获取好 Rockchip Linux SDK 及 矽慧通 X3300 相应的配置文档 / 配置文件。文中所述的开发板特指基于 RK3588 的 矽慧通 X3300。

## 需要准备的器材

| 类别 | 器材名称 | 规格要求 | 数量 | 用途说明 |
| - | - | - | - | - |
| **核心设备** | 矽慧通 X3300 | | 1 块 | 主要开发平台 |
| **电源设备** | 电源适配器 | DC 9-36V | 1 个 | 供电 |
| **调试工具** | USB 转串口模块 | 支持 1.5 Mbps 波特率 | 1 个 | 串口调试和日志输出 |
| **连接线缆** | Type-C USB 线 | 数据线 | 1 条 | 连接 OTG 口用于烧录 |
| **网络设备** | 以太网线 | 标准网线 | 1 条 | 网络通信 |
| **显示设备** | HDMI 线 | 标准 HDMI 接口 | 1 条 | 连接显示器 |
| **主机环境** | Ubuntu 主机 | Ubuntu 22.04 系统 | 1 台 | 编译和烧录环境 |

## 开发板介绍

在开始之前，请确保开发板已备妥，并熟悉其基本接口布局。

本文档操作所涉及的接口均位于开发板的前后两端。

---

![sysoul_x3300-interface-frontend](./img/sysoul_x3300-interface-frontend.png)

上图所示为开发板前侧接口，需要用到的有：

- OTG 口：Type-C USB 接口，用于烧录。
- BOOT 和 RST 按钮：下层标识 BOOT 和 RST 的按钮，用于进入 MaskROM 模式和复位开发板。

---

![sysoul_x3300-interface-backend](./img/sysoul_x3300-interface-backend.png)

上图所示为开发板后侧接口，需要用到的有：

- 电源：位于右侧，DC 9-36 V 接入，中间为正，两侧为负，可接入任意两个 V+ / V- 接线柱；
- 调试串口：下层右侧标识有 DEBUG 的 RXD、TXD；
- 以太网口：ETH1，设备树配置中为 `ethernet@fe1c0000`；
- 图像输出 HDMI：设备树配置中为 `hdmi@fde80000`；

---

### 电源

请使用规格匹配的电源适配器连接至开发板的 DC 电源接口，以确保供电稳定。该开发板支持 9-36V 宽压输入。正确连接电源后，开发板后侧上下两层的电源指示灯 PWR 均应亮起，如下图所示。

<div align="center">
  <img src="./img/sysoul_x3300-poweron.png" height="300" alt="sysoul_x3300-poweron"/>
</div>

### 串口连接

为进行设备调试与查看启动日志，需要通过 USB 转串口模块连接到开发板的调试串口。

默认串口参数如下：

| 参数项 | 参数值 |
| - | - |
| **波特率 (Baud Rate)** | `1500000` |
| **数据位 (Data Bits)** | `8` |
| **停止位 (Stop Bits)** | `1` |
| **校验位 (Parity)** | `None` |

硬件连接正确后，可使用 `minicom` 等串口工具进行通信。

```sh
# 如果未安装 minicom，先执行下面的命令安装
# sudo apt update && sudo apt install minicom

# 定义串口参数
BAUD_RATE=1500000
SERIAL_DEVICE="/dev/ttyUSB0"

# 启动 minicom
sudo minicom -b "${BAUD_RATE}" -D "${SERIAL_DEVICE}"
```

> **接线提示**
>
> 1.  **标准接法（交叉连接）**：将模块的 `RXD` 连接到开发板的 `TXD`，`TXD` 连接到 `RXD`，`GND` 连接 `GND`。
> 2.  **特殊情况（直连）**：部分厂商（如 [Firefly](https://wiki.t-firefly.com/USB-TO-TTL-Serial/usb-to-ttl-serial.html)）的模块可能已在内部做了交叉处理，此时需要 `RXD` 接 `RXD`，`TXD` 接 `TXD`。
>
> 请务必参考模块说明书或咨询厂商以确认正确的接线方式。

### 进入 Loader / MaskROM 模式

开发者可通过以下方法，使开发板进入指定的烧录模式。

#### 进入 MaskROM 模式

**MaskROM 模式**是芯片内置的底层启动模式，主要用于设备首次烧录或系统固件损坏时进行修复。

1. 保持开发板处于**通电状态**。
2. 同时按住 **`BOOT`** 键与 **`RST`** 键。
3. 先松开 **`RST`** 键。
4. 稍等片刻后，再松开 **`BOOT`** 键，此时开发板即进入 MaskROM 模式。

#### 进入 Loader 模式

**Loader 模式**通过软件命令进入，是常规固件升级时使用的标准模式。

1. 在开发板已启动的系统 Linux / Android 或者 U-Boot 的终端中，执行以下命令：
    ```sh
    reboot loader
    ```
2. 命令执行后，开发板将自动重启并进入 Loader 模式。

## 烧录基础系统

新出厂的开发板可能未预装任何固件。在这种情况下，需要首次烧录基础系统，包括 U-Boot、Linux 内核和根文件系统 (Rootfs)，以便检验开发板是否正常工作。

### 编译固件

下面编译 Rockchip Linux SDK 以生成 `update.img`。

首先切换到 SDK 的目录，查看目录结构如下：

```sh
ROCKCHIP_LINUX_SDK_DIR="~/rockchip_linux_sdk"

cd "${ROCKCHIP_LINUX_SDK_DIR}"
tree -L 1
```

```plain
user@host:~/rockchip_linux_sdk$ tree -L 1
.
├── app
├── buildroot
├── build.sh -> device/rockchip/common/scripts/build.sh
├── debian
├── device
├── docs
├── external
├── kernel
├── Makefile -> device/rockchip/common/Makefile
├── output
├── prebuilts
├── README.md -> device/rockchip/common/README.md
├── rk3588_linux_release.xml
├── rkbin
├── rkflash.sh -> device/rockchip/common/scripts/rkflash.sh
├── rockdev -> output/firmware
├── tools
├── u-boot
├── uefi
└── yocto
```

在编译开始之前，需要修改 U-Boot 的配置，以将自动启动延迟从 0 调整为 10 秒，方便 U-Boot 启动 Debian 之前按下 `Ctrl` + `C` 停止启动。

```sh
sed -i 's/CONFIG_BOOTDELAY=0/CONFIG_BOOTDELAY=10/g' u-boot/configs/rk3588_defconfig
```

下面开始编译。通过环境变量 `RK_ROOTFS_SYSTEM` 指定编译的根文件系统类型为 Debian，然后调用 `build.sh` 即可即可开始编译。

```sh
export RK_ROOTFS_SYSTEM="debian"
./build.sh sysoul_x3300_defconfig && ./build.sh
```

编译结果为 `output/update/Image/update.img`。

### 配置烧录工具

Rockchip 在 Linux 平台下使用 `upgrade_tool` 进行烧录，编译完成之后，`upgrade_tool` 将位于 `tools/linux/Linux_Upgrade_Tool/` 目录下。

当然，也可以直接从 GitHub 仓库中下载。

```sh
# 安装依赖
sudo apt update && sudo apt install libudev-dev libusb-1.0-0-dev
# 克隆包含烧录工具的仓库
git clone https://github.com/vicharak-in/Linux_Upgrade_Tool.git

# 将工具路径添加到 PATH 环境变量，并使其立即生效
echo 'export PATH="$PATH:'$(pwd)/Linux_Upgrade_Tool'"' >> ~/.bashrc
source ~/.bashrc
```

### 连接设备并进入 MaskROM 模式

对于一块全新的开发板，我们需要手动使其进入 **MaskROM 模式**。

1.  请按照前述步骤，使开发板进入 MaskROM 模式。
2.  使用 USB 线缆将开发板的 **USB-OTG** 口连接到主机。
3.  执行以下命令，检查设备是否被正确识别：

```sh
sudo upgrade_tool ld    # ListDevice，查看设备
```

如果看到类似如下的输出，说明设备已准备就绪。

```plain
List of rockusb connected(1)
DevNo=1	Vid=0x2207,Pid=0x350b,LocationID=322	Mode=Maskrom	SerialNo=
```

> **注意**：`Vid=0x2207` 是 Rockchip 公司的 USB 厂商 ID。如果未检测到设备，请检查 USB 线缆连接或更换主机的 USB 端口。

### 烧录固件

接下来，使用 `upgrade_tool` 工具将 `update.img` 固件包烧录至开发板的闪存。

```sh
sudo upgrade_tool uf update.img -noreset    # UpgradeFirmware，更新固件，使用 -noreset 参数防止烧录后自动重启，便于我们手动控制
```

烧录过程将持续数分钟，终端会显示详细的进度日志。

```plain
Loading firmware...
Support Type:3588	FW Ver:<FW-Ver>	FW Time:<YYYY-MM-DD hh:mm:ss>
Loader ver:<Loader-ver>	Loader Time:<YYYY-MM-DD hh:mm:ss>
Start to upgrade firmware...
Download Boot Start
Download Boot Success
Wait For Maskrom Start
Wait For Maskrom Success
Test Device Start
Test Device Success
Check Chip Start
Check Chip Success
Get FlashInfo Start
Get FlashInfo Success
Prepare IDB Start
Prepare IDB Success
Download IDB Start
Download IDB Success
Download Firmware Start
Download Image... (100%)
Download Firmware Success
Upgrade firmware ok.
```

当看到 `Upgrade firmware ok.` 消息时，表示烧录成功。

### 重启并验证

烧录完成后，手动重启开发板以加载新系统。

```sh
sudo upgrade_tool rd    # ResetDevice，复位设备
```

终端应返回：

```plain
Reset Device OK.
```

此时，开发板将启动预装的 Debian 系统。可以将 HDMI 连接到显示器，稍等片刻即可看到图形化界面。

![sysoul_x3300-debian-desktop](./img/sysoul_x3300-debian-desktop.png)

同时，在串口终端中也能观察到完整的启动日志以及控制台。

```plain
root@linaro-alip:/#
```

## 启动 hvisor

后续主机和开发板之间的文件传输均通过 IPv4 网络和 TFTP 协议进行。

### 配置主机网络并搭建 TFTP 服务器

> 参考 [嵌入式平台快速开发-Tftp 服务器搭建与配置](https://foreveryolo.top/posts/17937/)。

首先配置主机的 IPv4 网络。我们将主机与开发板配置在一个点对点的 `/30` 网络中，该网络仅包含两个可用的主机地址，适用于这种直连场景。

```sh
HOST_NET_DEVICE="enp49s0"   # 请根据实际情况修改为主机的网卡名称
HOST_IPV4="192.168.255.1"
NETMASK="255.255.255.252"   # /30 子网掩码

sudo ifconfig "${HOST_NET_DEVICE}" "${HOST_IPV4}" netmask "${NETMASK}"
```

在主机上同时安装 TFTP 服务器软件和 TFTP 客户端软件（用于测试服务器是否正常工作）。

```sh
sudo apt update && sudo apt install tftpd-hpa tftp-hpa
```

创建 TFTP 根目录并设置权限。

```sh
TFTP_DIR="/srv/tftp"

sudo mkdir -p "${TFTP_DIR}"
sudo chown tftp:tftp "${TFTP_DIR}"
sudo chmod -R ugo+rw,a+X "${TFTP_DIR}"
```

编辑 tftpd-hpa 配置文件。

```sh
sudo tee /etc/default/tftpd-hpa > /dev/null <<EOF
# /etc/default/tftpd-hpa

TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/srv/tftp"
TFTP_ADDRESS=":69"
TFTP_OPTIONS="-l -c -s"
EOF
```

启动 TFTP 服务器。

```sh
sudo systemctl start tftpd-hpa.service
```

测试 TFTP 服务器是否正常工作。

```sh
TFTP_DIR="/srv/tftp"
TEST_SRC_FILE_NAME="testfile.txt"
TEST_DST_FILE_NAME="downloaded_testfile.txt"

echo "TFTP Automation Test" > "${TFTP_DIR}/${TEST_SRC_FILE_NAME}" && \
tftp localhost -c get "${TEST_SRC_FILE_NAME}" "${TEST_DST_FILE_NAME}" && \
diff -q "${TFTP_DIR}/${TEST_SRC_FILE_NAME}" "${TEST_DST_FILE_NAME}" && \
echo "TFTP Test PASSED" || echo "TFTP Test FAILED" ; \
rm -f "${TFTP_DIR}/${TEST_SRC_FILE_NAME}" "${TEST_DST_FILE_NAME}"
```

### 准备 root-linux

#### 提取 kernel

可以直接将原有 Rockchip Linux SDK 的编译结果中的 kernel（位于 `kernel/arch/arm64/boot/` 路径下）提取出来，复制到 TFTP 目录。

```sh
TFTP_DIR="/srv/tftp"
ROCKCHIP_LINUX_SDK_DIR="$HOME/rockchip_linux_sdk"

cp "${ROCKCHIP_LINUX_SDK_DIR}/kernel/arch/arm64/boot/Image" "${TFTP_DIR}"
```

#### 编译 device-tree

将开发板的 dts (例如 `sysoul_x3300.dts`) 复制一份，命名为 `zone0.dts`，并进行以下修改以划分资源给 GuestOS：

1.  **为 GuestOS 预留 CPU 核心**：
  hvisor 的设计中，root-linux 和 GuestOS 各使用一部分 CPU 核心。此处我们仅为 root-linux 保留 `cpu0` 和 `cpu1`。在设备树中找到其余的 `cpu` 节点，将其删去。

2.  **为 hvisor 和 GuestOS 预留内存空间**：
  在 `memory` 节点的 `reserved-memory` 区域下，添加两块保留内存。一块用于 hvisor 本身，另一块用于 GuestOS 的物理内存。`no-map` 属性确保 root-linux 内核不会映射和使用这些区域。

  ```diff
  --- a/sysoul_x3300.dts
  +++ b/zone0.dts
  ...
  reserved-memory {
      ...
  +
  +   hvisor@480000 {
  +       no-map;
  +       reg = <0x00 0x480000 0x00 0x400000>;
  +   };
  +
  +   nonroot@50000000 {
  +       no-map;
  +       reg = <0x00 0x50000000 0x00 0x25000000>;
  +   };
  };
  ...
  ```

3.  **添加 hvisor virtio 设备节点**：
  在根节点下添加一个设备节点，用于 hvisor 的 virtio 后端驱动与 root-linux 内核进行通信。中断号需要根据具体的硬件平台进行配置，此处配置为 `0x20`。

  ```diff
  --- a/sysoul_x3300.dts
  +++ b/zone0.dts
  ...
  };

  +	hvisor_virtio_device {
  +		compatible = "hvisor";
  +		interrupt-parent = <0x01>;
  +		interrupts = <0x00 0x20 0x01>;
  +	};
  ```

完成上述修改后，编译设备树文件，并将编译结果复制到 TFTP 目录。

```sh
ROOT_LINUX_DTS="zone0.dts"
TFTP_DIR="/srv/tftp"

dtc -I dts -O dtb -o zone0.dtb "${ROOT_LINUX_DTS}"
cp zone0.dtb "${TFTP_DIR}"
```

#### 配置 rootfs

rootfs 直接采用烧录好的即可，不需要修改。

### 编译 hvisor

配置好 Rust 编译环境之后，拉取 syswonder/hvisor 仓库的 main 分支，完成编译，将编译结果复制到 TFTP 目录。

```sh
TFTP_DIR="/srv/tftp"

git clone git@github.com:syswonder/hvisor.git
cd hvisor
cargo install cargo-binutils

make BID=aarch64/sysoul-x3300
cp target/aarch64-unknown-none/debug/hvisor.bin "${TFTP_DIR}"
```

> 编译 hvisor 之前，还需要根据开发板的具体情况按需修改对应 BID 的 `board.rs` 文件中的配置信息。
>
> 可以通过 Linux 的 `/proc/iomem` 获取内存信息，根据设备信息填写 root zone 的 memory region 信息。
>
> **注意**：中断控制器的地址不要与 root zone 的 memory region 重叠。
>
> `board.rs` 中所需的中断号可以通过设备树文件和下面的命令快速获得。
>
> ```sh
> DTS_FILE="zone0.dts"
>
> cat "${DTS_FILE}" \
> | tr -d '\n' \
> | sed 's/;/\n/g' \
> | grep "interrupts =" \
> | tr -d '=<>' \
> | awk '{for(i=3; i<=NF; i+=3) print $(i)}' \
> | while read hex; do printf "%d\n" "$((hex + 0x20))"; done \
> | sort -n \
> | while read dec; do printf "0x%x\n" "$dec"; done \
> | tr '\n' ','
> ```

### 启动 hvisor + root-linux

连接串口和电源，按下 RST 重启开发板。在串口终端中观察 U-Boot 启动信息，当看到倒计时提示时，立即按下 `Ctrl` + `C` 以中断自动启动流程，进入 U-Boot 命令行。

在 U-Boot 命令行中执行命令。首先配置 IPv4 网络，以访问主机 TFTP 服务器。在确认网络配置无误之后，使用 tftp 命令依次下载 hvisor kernel device-tree，并加载到对应地址，最后通过 `bootm` 指令启动。

```sh
# 配置网络
setenv ipaddr   192.168.255.2
setenv netmask  255.255.255.252
setenv serverip 192.168.255.1

# 为每个文件设置正确的内存地址变量
setenv board_dtb_addr       0x00400000
setenv hvisor_addr          0x00500000
setenv kernel_addr          0x09400000
setenv root_linux_dtb_addr  0x10000000

# 从 TFTP 服务器加载每个文件到对应的内存地址
tftp  ${board_dtb_addr}       sysoul_x3300.dtb
tftp  ${hvisor_addr}          hvisor.bin
tftp  ${kernel_addr}          Image
tftp  ${root_linux_dtb_addr}  zone0.dtb

# 从 hvisor_addr 启动
bootm ${hvisor_addr} - ${board_dtb_addr}
```

> 配置好网络之后，可以通过 ping 命令测试通断。
>
> ```sh
> ping ${serverip}
> ```
>
> 显示结果
>
> ```plain
> => ping ${serverip}
> ethernet@fe1c0000 Waiting for PHY auto negotiation to complete. done
> Using ethernet@fe1c0000 device
> host 192.168.255.1 is alive
> ```

看到 root-linux 正常启动，几乎与正常基础系统无差别。

## 启动 GuestOS

下面以 Ubuntu-22.04 搭配 Linux 5.10 内核 为例，在 zone1 上启动 Linux。配置好 zone1 所需文件后通过 TFTP 传输到开发板上再用 hvisor-tool 启动。

> root-linux 访问 TFTP 服务器需要安装 TFTP 客户端，此时通过主机 Linux 作为跳板，使得 root-linux 可以访问互联网，然后在其上通过包管理器 apt 安装 TFTP 客户端软件。
>
> 具体命令为：
>
> 开发板 root-linux 的 Debian 系统默认使用 NetworkManager 管理网络，因此我们使用 `nmcli` 工具进行配置：
>
> ```sh
> # 定义网络参数
> HOST_IPV4="192.168.255.1"
> BOARD_IPV4="192.168.255.2"
> NETMASK="30"
> DNS_SERVER="8.8.8.8"
> BOARD_NET_DEVICE="eth0"
>
> # 配置开发板的静态 IP 地址、子网掩码、网关和 DNS 服务器
> nmcli connection modify "${BOARD_NET_DEVICE}" 802-3-ethernet.mac-address ""
> nmcli connection modify "${BOARD_NET_DEVICE}" connection.interface-name "${BOARD_NET_DEVICE}"
> nmcli connection modify "${BOARD_NET_DEVICE}" ipv4.method manual
> nmcli connection modify "${BOARD_NET_DEVICE}" ipv4.addresses "${BOARD_IPV4}/${NETMASK}"
> nmcli connection modify "${BOARD_NET_DEVICE}" ipv4.gateway "${HOST_IPV4}"
> nmcli connection modify "${BOARD_NET_DEVICE}" ipv4.dns "${DNS_SERVER}"
> nmcli connection modify "${BOARD_NET_DEVICE}" connection.autoconnect yes
> ```
>
> 在主机上执行：
>
> ```sh
> BOARD_IPV4="192.168.255.2"
> NETMASK="30"
>
> # 开启内核的 IPv4 转发功能
> sudo sysctl -w net.ipv4.ip_forward=1
>
> # 配置防火墙，允许来自开发板的流量进行转发
> sudo iptables -A FORWARD -s "${BOARD_IPV4}" -j ACCEPT
> sudo iptables -A FORWARD -d "${BOARD_IPV4}" -j ACCEPT
>
> # 配置 NAT 规则
> sudo iptables -t nat -A POSTROUTING -s "${BOARD_IPV4}/${NETMASK}" -j MASQUERADE
> ```
>
> 此时开发板 root-linux 即可通过主机访问 Internet，即可执行命令安装 TFTP 客户端软件。
>
> ```sh
> sudo apt update && sudo apt install tftp-hpa
> ```
>
> 最后需要指出，上述所有在主机上执行的命令均为临时配置，在系统重启后会失效。若要实现永久生效，建议将这些配置写入系统的网络管理服务配置文件中或通过启动脚本固化。
>

### 编译 hvisor-tool

首先，安装交叉编译工具链。

```sh
sudo apt update && sudo apt install gcc-aarch64-linux-gnu
```

接下来，克隆并编译 `hvisor-tool`。

```sh
ROCKCHIP_LINUX_SDK_DIR="$HOME/rockchip_linux_sdk"

git clone git@github.com:syswonder/hvisor-tool.git
cd hvisor-tool
export PATH="$PATH:${ROCKCHIP_LINUX_SDK_DIR}/prebuilts/gcc/linux-x86/aarch64/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin"
make all ARCH=arm64 LOG=LOG_INFO KDIR="${ROCKCHIP_LINUX_SDK_DIR}/kernel"
```

编译完成后，将结果放到 TFTP 目录里面。

```sh
HVISOR_TOOL_DIR="$HOME/hvisor-tool"
TFTP_DIR="/srv/tftp"

# 复制 hvisor-tool 的 driver/hvisor.ko 和 tools/hvisor
cp "${HVISOR_TOOL_DIR}/output/hvisor.ko" "${TFTP_DIR}"
cp "${HVISOR_TOOL_DIR}/output/hvisor" "${TFTP_DIR}"
```

### 准备 zone1-linux

为简化，zone1-linux 仅提供 virtio-console（提供控制台）和 virtio-vlk（提供根文件系统）。

#### 编译 kernel

编译完成后，将结果复制到 TFTP 目录。

```sh
ROCKCHIP_LINUX_SDK_DIR="$HOME/rockchip_linux_sdk"
TFTP_DIR="/srv/tftp"

# 下载 linux 5.10 源码
git clone https://github.com/torvalds/linux -b v5.10 --depth=1
cd linux
git checkout v5.10

# 生成默认的编译配置
CROSS_COMPILE_PATH="${ROCKCHIP_LINUX_SDK_DIR}/prebuilts/gcc/linux-x86/aarch64/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin"
CROSS_COMPILE_PREFIX="${CROSS_COMPILE_PATH}/aarch64-none-linux-gnu-"
make ARCH=arm64 CROSS_COMPILE=${CROSS_COMPILE_PREFIX} defconfig

# 启用 CONFIG_BLK_DEV_RAM，以启用 RAM 块设备支持
./scripts/config --enable CONFIG_BLK_DEV_RAM

# 编译
make ARCH=arm64 CROSS_COMPILE=${CROSS_COMPILE_PREFIX} Image -j$(nproc)

# 编译结果复制
cp arch/arm64/boot/Image "${TFTP_DIR}/zone1-linux.kernel"
```

#### 编译 device-tree

zone1-linux 需要一个独立的设备树来描述 hvisor 为其提供的虚拟硬件环境，例如虚拟 CPU、内存布局以及 virtio 设备等。

```sh
HVISOR_DIR="$HOME/hvisor"
TFTP_DIR="/srv/tftp"

cd "${HVISOR_DIR}"
make BID=aarch64/sysoul-x3300 dtb
cp "${HVISOR_DIR}/platform/aarch64/sysoul-x3300/image/dts/zone1-linux.dtb" "${TFTP_DIR}/zone1-linux.dtb"
```

#### 下载 rootfs

使用 `wget` 命令下载预先定义的 Ubuntu 22.04.5 的根文件系统压缩包。这是一个最小化的 Ubuntu 环境，包含了运行一个基本系统所需的核心文件。

```sh
wget https://cdimage.ubuntu.com/ubuntu-base/releases/22.04/release/ubuntu-base-22.04.5-base-arm64.tar.gz
```

创建一个大小为 128 MiB 的空白的磁盘镜像文件，将其格式化为 ext4 文件系统，然后将下载的根文件系统解压进去。

```sh
ZONE1_ROOTFS_IMAGE="zone1-linux-rootfs.ext4"

dd if=/dev/zero of="${ZONE1_ROOTFS_IMAGE}" bs=1M count=128 oflag=direct
mkfs.ext4 "${ZONE1_ROOTFS_IMAGE}"
mkdir -p rootfs/
sudo mount -t ext4 "${ZONE1_ROOTFS_IMAGE}" rootfs/
sudo tar -xzf ubuntu-base-22.04.5-base-arm64.tar.gz -C rootfs/
sudo umount rootfs
rm -r rootfs
```

完成后，将该文件链接（节省空间）到 TFTP 目录。

```sh
TFTP_DIR="/srv/tftp"
ZONE1_ROOTFS_IMAGE="zone1-linux-rootfs.img"
ABSOLUTE_IMAGE_PATH=$(realpath "${ZONE1_ROOTFS_IMAGE}")

ln "${ABSOLUTE_IMAGE_PATH}" "${TFTP_DIR}"
```

#### 准备 JSON 配置文件

`hvisor-tool` 通过 JSON 文件来管理 zone 和 virtio 设备的配置。我们需要创建两个核心的配置文件：

- `zone1-linux.json`：用于定义 zone 本身的资源；
- `zone1-linux-virtio.json`：用于定义其所需的 virtio 设备。

`hvisor` 仓库中存放了本文对应的配置文件，位于 `platform/aarch64/sysoul-x3300/configs/` 目录下，使用下面的命令将其复制到 TFTP 目录下。

```sh
HVISOR_DIR="$HOME/hvisor"
TFTP_DIR="/srv/tftp"

cp "${HVISOR_DIR}/platform/aarch64/sysoul-x3300/configs/*.json" "${TFTP_DIR}"
```

### 启动 zone1-linux

在 root-linux 的 root 用户下，执行命令：

```sh
HOST_IPV4="192.168.255.1"

# 切换到 /root 目录
cd ~

# 从 TFTP 服务器上，下载 hvisor-tool 和 zone1-linux 所需内容
tftp "${HOST_IPV4}" <<EOF
get hvisor.ko
get hvisor
get zone1-linux.kernel
get zone1-linux.dtb
get zone1-linux-rootfs.ext4
get zone1-linux.json
get zone1-linux-virtio.json
quit
EOF

# 加载 hvisor.ko
insmod hvisor.ko

# 给 hvisor 添加可执行权限
chmod +x hvisor

# 启动 zone1-linux 所需的 virtio-console 和 virtio-blk
rm nohup.out
nohup ./hvisor virtio start zone1-linux-virtio.json &

# 启动 zone1-linux
./hvisor zone start zone1-linux.json
```

此时，执行命令

```sh
./hvisor zone list
```

输出为

```plain
|     zone_id     |       cpus        |      name       |     status |
|               0 |              0, 1 |      root-linux |    running |
|               1 |              2, 3 |     zone1-linux |    running |
```

说明 zone1-linux 已经启动成功。

此时输入命令 `cat nohup.out | grep "char device"` 即可查看 zone1-linux virtio-console 对应的 pts（一般是 `/dev/pts/0`），此时执行命令

```sh
screen /dev/pts/0
```

即可连接到 zone1-linux 的 console。

> **提示：如何在 minicom 中分离 (detach) screen 会话**
>
> `minicom` 和 `screen` 的默认转义键都是 `Ctrl` + `A`。因此，当在 `minicom` 窗口中与 `screen` 会话交互时，需要先发送一个转义字符给 `minicom`，再将下一个 `Ctrl` + `A` 传递给 `screen`。
>
> - **分离 `screen` 会话的步骤**：依次按下 `Ctrl` + `A`，`Ctrl` + `A`，`D`。

---

## 外设接口对应的设备树节点

### USB

X3300 的 USB 由 **1 个 USB3 OTG（Type-C）、1 个 USB3 Host、2 组 USB2 Host** 构成，
对应控制器与 PHY 节点如下：

| 功能 | 设备树节点 | 说明 |
|---|---|---|
| USB3 OTG（Type-C，烧录口） | `usb@fc000000`（`snps,dwc3`） | `dr_mode = "otg"`，中断 SPI 0xdc，含 `quirk-skip-phy-init` 等；物理层 = `usb2phy0`（`syscon@fd5d0000` 内 `usb2-phy@0` 的 `otg-port`）提供 USB2 通路 + `phy@fed80000`（USBDP combo PHY0，`usbdp0`）的 `u3-port` 提供 USB3 通路 |
| USB3 Host | `usb@fcd00000`（`snps,dwc3`） | `dr_mode = "host"`，中断 SPI 0xde；USB3 PHY = `phy@fee20000`（`rockchip,rk3588-naneng-combphy`，即 USB3/PCIe combo PHY2，refclk 100 MHz） |
| USB2 Host0（EHCI+OHCI） | `usb@fc800000` / `usb@fc840000` | 中断 SPI 0xd7/0xd8；`clocks` 含 `usbhost/arbiter`（CRU）+ `utmi`（来自 usb2phy）；PHY = `syscon@fd5d8000` 内 `usb2-phy@8000` 的 `host-port` |
| USB2 Host1（EHCI+OHCI） | `usb@fc880000` / `usb@fc8c0000` | 中断 SPI 0xda/0xdb；PHY = `syscon@fd5dc000` 内 `usb2-phy@c000` 的 `host-port` |
| USB2 PHY 寄存器 | `syscon@fd5d0000` / `fd5d8000` / `fd5dc000` | `rockchip,rk3588-usb2phy-grf`，各含一个 `rockchip,rk3588-usb2phy` 子节点 |
| USB3 combo PHY 寄存器 | `syscon@fd5c8000`（usbdpphy-grf） | USBDP PHY0/1 = `phy@fed80000`、`phy@fed90000`（aliases `usbdp0/1`） |

电源域：上述控制器均挂在 USB 电源域（`power-domains = <&power 0x1f>`）下。
USB2 Host 各口的 `utmi` 时钟由对应 usb2phy 以 0-cell 时钟输出提供。

### 串口（UART）

| 功能 | 设备树节点 | 说明 |
|---|---|---|
| 调试串口（板载 DEBUG 口） | `fiq-debugger` 节点 | `rockchip,serial-id = <2>` → 复用 `serial@feb50000`（UART2，`0xfeb50000`）；波特率 1500000；内核 console 为 `ttyFIQ0`，`chosen.bootargs` 同时给出 `earlycon=uart8250,mmio32,0xfeb50000` 早期打印 |
| UART0 | `serial@fd890000` | 中断 SPI 0x14c；alias `serial0` |
| UART1~9 | `serial@feb40000` ~ `serial@febc0000` | alias `serial1`~`serial9`；各节点含 `baudclk/apb_pclk` 时钟、DMA（`dmas`）与对应 pinmux |

> 注意：原厂树中 `serial@feb50000`（UART2 自身）为 `disabled`，调试串口由 FIQ debugger 以 FIQ 方式独占，普通 UART 驱动不再注册该口

### 以太网（GMAC）

| 功能 | 设备树节点 | 说明 |
|---|---|---|
| ETH1（千兆，板载唯一网口） | `ethernet@fe1c0000`（alias `ethernet1`） | `rockchip,rk3588-gmac` + `snps,dwmac-4.20a`；`phy-mode = "rgmii-rxid"`，`clock_in_out = "output"`（MAC 输出 125M 参考时钟）；复位脚 `reset-gpio = <&gpio3 RK_PA7 ...>`（`gpio@fec40000` 第 15 脚）；中断 `macirq` SPI 0xea；电源域 `power-domains = <&power 0x21>` |
| ETH0（未用） | `ethernet@fe1b0000`（alias `ethernet0`） | gmac0，本板未使能 |

时钟：`stmmaceth / clk_mac_ref / pclk_mac / aclk_mac / ptp_ref` 均由 CRU 提供；
PHY 挂在 MAC 节点的 MDIO 子节点下（RGMII，板载 PHY）。

### 音频（3.5mm 口）

| 功能 | 设备树节点 | 说明 |
|---|---|---|
| 音频编解码器 | `i2c3`（`i2c@feab0000`）上的 `es8311@19` | Everest ES8311（地址 0x19，`status = "okay"`）；MCLK = `mclkout_i2s0`（12.288 MHz，CRU clk-out），pinmux `i2s0_mclk`；增益参数 `adc-pga-gain/adc-volume/dac-volume` |
| I2S 数据口 | `i2s@fe470000`（`i2s0_8ch`） | `rockchip,rk3588-i2s-tdm`，与 codec 的 `sound-dai` 对接 |
| 声卡 | `i2s0-sound` | `simple-audio-card`，名 `"rockchip,es8311"`：cpu=`i2s0_8ch`，codec=`es8311` |

功能要点（依据 es8311 驱动 DAPM）：
- **播放**：MONO DAC → DIFFERENTIAL OUT（差分输出，接 3.5mm 输出端）；
- **录音**：MONO ADC，AMIC/DMIC 可选（板上为模拟麦克风输入），PGA 最大 18 dB，`aec-mode = "adc left, adc right"`；
- 支持**同时录放**（全双工），但驱动 DAPM 每方向为单声道；3.5mm 口无插入检测。

### HDMI

| 功能 | 设备树节点 | 说明 |
|---|---|---|
| HDMI0 输出 | `hdmi@fde80000`（alias `hdmi0`） | `rockchip,rk3588-dw-hdmi`；5 路中断（含 HPD）；时钟 `pclk/hpd/earc/hdmitx_ref/aud/dclk_vp0..3/hclk_vo1/link_clk`；`power-domains = <&power 0x1a>`；`status = "okay"`；`enable-gpios` 为 HDMI 5V 使能脚 |
| HDMI0 PHY | `hdmiphy@fed60000` | `rockchip,rk3588-hdptx-phy-hdmi`，经 `phys = <&hdmiphy>` 接入 HDMI0 |
| HDMI1（未用） | `hdmi@fdea0000`（alias `hdmi1`） | 本板未接出 |

视频源：`vop@fdd90000`（VOP2）的 video port 经 `ports/port@0/endpoint@2`（`remote-endpoint`）接到 `hdmi@fde80000`；HDMI 同时是 `#sound-dai-cells = <0>` 的音频输出端点（HDMI 音频走 I2S，配 `hdmi0-sound` 声卡）。

### DSI（MIPI 屏）

| 功能 | 设备树节点 | 说明 |
|---|---|---|
| DSI0 | `dsi@fde20000`（alias `dsi0`） | `rockchip,rk3588-mipi-dsi2`；PHY = `phy@feda0000`（MIPI DCPHY0）；电源域 `power-domains = <&power 0x18>`；`status = "okay"`；`port@0` 接 VOP2，`port@1` 接 panel |
| DSI0 面板 | `dsi@fde20000/panel@0` | `simple-panel-dsi`，4 lane（`dsi,lanes = <4>`），含 `panel-init-sequence`；背光 = `backlight`（PWM，`pwm-backlight`，默认亮度 200/255） |
| DSI1 | `dsi@fde30000`（alias `dsi1`） | 第二路 MIPI DSI（本板为副屏/扩展用途） |
| 显示控制器 | `vop@fdd90000` | VOP2，多 video port 分别路由到 HDMI0/DSI0/DSI1/DP 等 |

> 实践提示：本文书其余章节中，root/zone1/zone2 各自的设备树均由本原厂树裁剪而来
> ——不同 zone 会裁剪掉不属于自己的控制器（例如 zone2 拿走显示/触摸/USB 通路后，
> 原厂树里的 `i2c@feab0000`（音频 codec 所在）与 `i2s0-sound` 在 zone2 侧均被
> `disabled`，音频控制器未划入任何 zone，见"外设划分"相关讨论）。

---

## RKLLM 的 CPU 绑定：RKLLMParam / RKLLMExtendParam

### 背景

RKLLM 的推理主体跑在 NPU 上（`librkllmrt.so` 内嵌 ggml + NPU 算子），但运行时
需要若干 **CPU 线程**做调度、embedding、采样等辅助工作。runtime 内部按
**目标平台 profile** 决定使用多少 CPU、把线程绑到哪些核：

- 报错线索一：`The number of enabled CPUs must be greater than or equal to the number of NPU cores`；
- 报错线索二：`Mismatch between enabled CPUs mask and expected count. Please check the configuration.`；
- 启动日志：`Enabled cpus: [...]` / `Enabled cpus num: N`（N 由模型/平台决定，RK3588 为 4）；
- 绑核失败时：`error: set affinity failed`（线程降级为不绑定，仍可运行，属噪音）。

CPU 数量/掩码可在 **`rkllm_init()` 前**通过参数结构体显式指定。

### 参数结构

```c
typedef struct {
    const char* model_path;      // 模型文件
    int32_t  max_context_len;    // 上下文窗口
    int32_t  max_new_tokens;     // 单次生成上限
    int32_t  top_k, n_keep;
    float    top_p, temperature;
    float    repeat_penalty, frequency_penalty, presence_penalty;
    int32_t  mirostat, ...;
    bool     skip_special_token, ignore_eos_token;
    bool     is_async;
    RKLLMExtendParam extend_param;   // ← CPU 绑定在这里
} RKLLMParam;

typedef struct {
    int32_t  base_domain_id;
    int8_t   embed_flash;
    int8_t   enabled_cpus_num;    // 允许 runtime 使用的 CPU 数量（0 = 自动）
    uint32_t enabled_cpus_mask;   // 允许的 CPU 位图（0 = 自动）
    uint8_t  n_batch;             // >1 开启 batch 推理
    int8_t   use_cross_attn;
    uint8_t  reserved[104];
} RKLLMExtendParam;
```

官方推荐的初始化流程：`rkllm_createDefaultParam()` 取默认值 → 改需要的字段 →
`rkllm_init(&handle, &param, callback)`。

### hvisor 多 zone 场景下的坑

zone 内的 Linux 只看到**从 0 开始的逻辑核**（例如 zone1 拿物理核 4-7，guest 内为
cpu0-3）。而 runtime 的自动枚举与官方 server 的 `0xF0` 都按**物理核号 4-7** 绑核：

```text
rkllm: Enabled cpus: [4, 5, 6, 7]
error: set affinity failed # guest 内不存在 cpu4-7，绑核失败（降级为不绑）
```

绑核失败不影响正确性（线程不绑照跑），但在 zone 环境下属于可消除的噪音与不确定因素。
**正确做法：显式传逻辑核掩码。**
