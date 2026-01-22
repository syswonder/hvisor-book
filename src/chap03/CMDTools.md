# 命令行工具

命令行工具是附属于hvisor的管理工具，用于在管理虚拟机Root Linux上创建和关闭其他虚拟机，并负责启动Virtio守护进程，提供Virtio设备模拟。

仓库地址位于[hvisor-tool](https://github.com/syswonder/hvisor-tool)。更详细的内容请见对应仓库README。

## 如何配置

Non-root Zones的配置可以参考：[Non-root Zones 配置文档](../chap03/ZoneConfig.md)。

virtio的配置可以参考：[virtio 配置文档](../chap03/VirtIOUseage.md)。

## 如何使用

hvisor-tool的使用可以参考：[管理工具使用文档](../chap04/subchap04/ManageTools.md)。

hvisor-tool在qemu-aarch64下的使用示例可以参考：[qemu-aarch64下的使用示例](https://github.com/syswonder/hvisor-tool/tree/main/examples/qemu-aarch64/with_virtio_blk_console)。

如果使用virtio设备，请先启动virtio守护进程，然后再启动Non-root Zones。
