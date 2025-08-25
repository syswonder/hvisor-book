# 如何启动 NonRoot Linux（Zone1 Linux）

使用 hvisor-tool 启动 NonRoot Linux。

```bash
# Linux 源代码路径
LINUX_PATH="<路径>/linux"

git clone https://github.com/syswonder/hvisor-tool.git
cd hvisor-tool
make all ARCH=arm64 LOG=LOG_WARN KDIR="${LINUX_PATH}"
```

> 请务必保证 hvisor 中的 root linux 镜像也是由编译 hvisor-tool 时参数选项中的 Linux 源代码目录编译产生。

> 请务必保证 hvisor-tool 编译时采用的 linux header 版本与 root linux 的 linux header 版本一致，否则 hvisor-tool 的 driver 可能会无法加载。 可以通过使用 root linux 相同的交叉编译工具链进行编译。

编译完成后，需要将 hvisor-tool 的可执行文件 `tools/hvisor` 和内核模块 `driver/hvisor.ko` 复制到 root linux 的根文件系统中启动 zone1 linux 的目录，例如 `/root`，再同时将 zone1 的根文件系统、内核镜像、以及编译后的设备树放在同一目录。

还需要将 hvisor-tool 的配置文件放入文件系统中，文件名需要保持一致，配置文件位于`platform/<架构>/<平台名>/configs/`目录下。

以架构为`aarch64`、平台名为`qemu-gicv3`为例，则配置文件为`platform/aarch64/qemu-gicv3/configs/zone1-linux-virtio.json` 和 `platform/aarch64/qemu-gicv3/configs/zone1-linux.json`。

下面命令以`aarch64`架构为例，若为其他架构请更改命令对应的部分。

```bash
# 回到创建的 root linux 根文件系统时的目录
LINUX_PATH="<路径>/linux"
HVISOR_PATH="<路径>/hvisor"
HVISOR_TOOL_PATH="<路径>/hvisor-tool"

# 挂载
sudo mount -t ext4 rootfs1.img rootfs/

# 复制 hvisor-tool 的 driver/hvisor.ko 和 tools/hvisor
sudo cp "${HVISOR_TOOL_PATH}/driver/hvisor.ko" rootfs/root/
sudo cp "${HVISOR_TOOL_PATH}/tools/hvisor" rootfs/root/

# 复制`platform/<架构>/<平台名>/configs/`目录下的 hvisor-tool 的配置文件到 root 路径下
sudo cp "${HVISOR_PATH}/platform/aarch64/qemu-gicv3/configs/zone1-linux-virtio.json" \
    rootfs/root/zone1-linux-virtio.json
sudo cp "${HVISOR_PATH}/platform/aarch64/qemu-gicv3/configs/zone1-linux.json" \
    rootfs/root/zone1-linux.json

# 复制 zone1 linux 的根文件系统、内核镜像、以及编译后的设备树
sudo cp rootfs2.img \
    rootfs/root/rootfs2.ext4
sudo cp "${LINUX_PATH}/arch/arm64/boot/Image" \
    rootfs/root/Image
sudo cp "${HVISOR_PATH}/platform/aarch64/qemu-gicv3/image/dts/zone1-linux.dtb" \
    rootfs/root/zone1-linux.dtb

# 卸载
sudo umount rootfs
```

之后在 QEMU 上即可通过 root linux 启动 zone1-linux。具体命令如下。

在 hvisor 目录下执行
```bash
make ARCH=<架构> LOG=<日志等级> BOARD=<平台名> run
```

例如
```bash
make ARCH=aarch64 LOG=info BOARD=qemu-gicv3 run
```
接下来启动 root linux
```bash
bootm 0x40400000 - 0x40000000

cd root
insmod hvisor.ko
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mkdir -p /dev/pts
mount -t devpts devpts /dev/pts

rm nohup.out

# 启动 zone1-linux 的 virtio
nohup ./hvisor virtio start zone1-linux-virtio.json &
# 启动 zone1-linux
./hvisor zone start zone1-linux.json && \
cat nohup.out | grep "char device" && \
script /dev/null
```

如果显示 virtio 出现 WARNING 或者 ERROR，可以查看 nohup.out 查看详细信息，或者使用 dmesg 命令查看内核日志。