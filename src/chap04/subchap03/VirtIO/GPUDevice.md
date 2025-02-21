# Virtio GPU

要使用hvisor-tool中的Virtio GPU设备，需要首先在host上安装libdrm，并进行一些相关的配置。

## 前置条件

* 安装libdrm

我们需要安装libdrm来编译Virtio-gpu，假设目标平台为**arm64**。

```shell
wget https://dri.freedesktop.org/libdrm/libdrm-2.4.100.tar.gz
tar -xzvf libdrm-2.4.100.tar.gz
cd libdrm-2.4.100
```

tips: 2.4.100以上的libdrm需要使用meson等进行编译，较为麻烦，https://dri.freedesktop.org/libdrm有更多版本。

```shell
# 安装到你的aarch64-linux-gnu编译器
./configure --host=aarch64-linux-gnu --prefix=/usr/aarch64-linux-gnu && make && make install
```

对于 loongarch64 需要使用：

```shell
./configure --host=loongarch64-unknown-linux-gnu --disable-nouveau --disable-intel --prefix=/opt/libdrm-install && make && sudo make install
```

* 配置Linux内核

Linux内核需要支持virtio-gpu和drm相关的驱动，具体来说需要在编译内核时启动以下选项

```
CONFIG_DRM=y
CONFIG_DRM_VIRTIO_GPU=y
```

有可能有其他GPU相关的驱动没有被编译到内核，这里需要根据具体设备编译，可以在编译时使用menuconfig来进行配置，具体在`Device Drivers`->`Graphics support`->` Direct Rendering Infrastructure(DRM)`，`Graphics support`条目下也有支持virtio-gpu相关的驱动，如果需要使用可以开启相关字段的编译，如`Virtio GPU driver`和`DRM Support for bochs dispi vga interface`

`Graphics support`条目的底部还有`Bootup logo`，启用该选项可以在启动时在显示屏幕上看到CPU核数个数的Linux logo

* 为Root Linux探测物理GPU设备

要在`Root Linux`中探测物理GPU设备，你需要编辑`hvisor/src/platform`目录下的文件，以便在PCI总线上探测GPU设备。需要将Virtio-gpu设备的中断号添加到`ROOT_ZONE_IRQS`中。例如：

```
pub const ROOT_PCI_DEVS: [u64; 3] = [0, 1 << 3, 6 << 3];
```

启动`Root Linux`后，你可以通过运行`dmesg | grep drm`或`lspci`来检查你的 GPU 设备是否正常工作。若/dev/dri下出现card0和renderD128等文件，说明成功识别到图形设备，并且该设备可以使用drm操控

* 查看真实GPU设备是否受支持

如果要移植Virtio-GPU到其他平台，需要确保该平台上的物理GPU设备受drm框架支持。要查看 libdrm 支持的设备，可以安装`libdrm-tests`包，使用命令`apt install libdrm-tests`，然后运行`modetest`

* qemu启动参数

如果hvisor运行在qemu aarch64环境下，则需要qemu向root linux提供GPU设备。在qemu启动参数中加入：

```
QEMU_ARGS += -device virtio-gpu,addr=06,iommu_platform=on
QEMU_ARGS += -display sdl
```

同时确保启动参数中包含smmu的配置：

```
-machine virt,secure=on,gic-version=3,virtualization=on,iommu=smmuv3
-global arm-smmuv3.stage=2
```

