# 如何启动NonRoot Linux

hvisor对NonRoot的启动做了妥善处理，使得启动较为简单，方式如下：

1. 准备好用于 NonRoot Linux 的内核镜像，设备树，以及文件系统。将内核和设备树放置在 Root Linux 的文件系统中。

2. 在给 NonRoot Linux 的设备树文件中指定好此 NonRoot Linux所使用的串口和需要挂载的文件系统，示例如下：

```
	chosen {
		bootargs = "clk_ignore_unused console=ttymxc3,115200 earlycon=ec_imx6q3,0x30a60000,115200 root=/dev/mmcblk3p2 rootwait rw";
		stdout-path = "/soc@0/bus@30800000/serial@30a60000";
	};
```

3. 编译用于 Hvisor 的[内核模块和命令行工具](https://github.com/syswonder/hvisor-tool?tab=readme-ov-file)，将其放置在 Root Linux 的文件系统中。

4. 启动 Hvisor 的 Root Linux，注入刚才编译好的内核模块：

```
insmod hvisor.ko
```

5. 使用命令行工具，这里假定其名字为```hvisor```，启动 NonRoot Linux。

```
./hvisor zone start --kernel 内核镜像,addr=0x70000000 --dtb 设备树文件,addr=0x91000000 --id 虚拟机编号（从1开始指定）
```

6. NonRoot Linux 启动完毕，打开刚才指定的串口即可使用。