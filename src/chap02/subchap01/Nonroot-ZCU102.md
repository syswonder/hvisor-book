## ZCU102 NonRoot 启动
任航麒(2572131118@qq.com)
1. 使用启动 Root 时所用的 Linux 内核源码编译 [hvisor-tool](https://github.com/syswonder/hvisor-tool)，详细编译流程可以参考 [Readme](https://github.com/syswonder/hvisor-tool/blob/main/README-zh.md).
2. 准备启动 NonRoot 所需要的 ```virtio_cfg.json``` 和 ```zone1_linux.json```，这里可以直接使用 hvisor-tool 目录下的 ```example/zcu102-aarch64```，里面的内容已经经过验证，确保可以启动。
3. 准备 NonRoot 所需要的 linux 内核 Image，文件系统 rootfs，以及设备树 linux1.dtb。其中的内核和文件系统可以和 Root 一样，Linux1.dtb 则是按需配置，也可以使用 hvisor 目录下的 ```images/aarch64/devicetree/zcu102-nonroot-aarch64.dts```.
4. 将 ```hvisor.ko, hvisor, virtio_cfg, zone1_linux.json, linux1.dtb, Image, rootfs.ext4``` 拷贝到 Root Linux 所用的文件系统中。
5. 在 RootLinux 输入下述命令启动 NonRoot:
```
# 加载内核模块
insmod hvisor.ko
# 创建 virtio 设备
nohup ./hvisor virtio start virtio_cfg.json &
# 根据 json 配置文件启动 NonRoot
./hvisor zone start zone1_linux.json 
# 查看 NonRoot 的输出，并交互。
screen /dev/pts/0
```
更多操作细节参考 [hvisor-tool Readme](https://github.com/syswonder/hvisor-tool/blob/main/README-zh.md)