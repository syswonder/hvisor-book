# 文件树解释
hvsior 项目的文件树如下所示

```bash
├── driver # hvsior 实现的驱动
├── images # 存放各个平台的镜像
│   ├── aarch64
│   │    ├── bootloader # 存放启动引导程序
│   │    ├── devicetree # 存放设备树
│   │    ├── kernel # 存放内核
│   │    └── virtdisk # 存放根文件系统
│   │       └── rootfs
│   └── riscv64 # 同上
│        └── rCore-Tutorial-v3
├── scripts # 存放编译脚本
├── src # hvsior 源码
│   ├── arch # 架构相关代码
│   │   ├── aarch64
│   │   └── riscv64
│   ├── device # 设备相关代码
│   │   ├── irqchip
│   │   │   ├── gicv3
│   │   │   └── plic
│   │   └── uart
│   ├── hypercall # 超级调用相关代码
│   ├── memory # 内存管理相关代码
│   └── platform # 平台相关代码
├── tools # 工具
│   └── includes # 头文件
└── vendor # 第三方代码
    └── fdt
        ├── dtb
        ├── dts
        ├── examples
        └── src

```