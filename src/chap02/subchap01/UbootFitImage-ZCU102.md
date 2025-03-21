# UBOOT FIT 镜像制作、加载与启动

wheatfox (wheatfox17@icloud.com)

本文介绍 FIT 镜像相关的基本知识，以及如何制作、加载和启动 FIT 镜像。

## ITS 源文件
ITS 是 uboot 生成 FIT 镜像（FIT Image）的源码，即 Image Tree Source，其采用 Device Tree Source（DTS）语法格式，可以通过 uboot 提供的工具 mkimage 生成 FIT 镜像。
在 hvisor 的 ZCU102 移植中，使用 FIT 镜像打包 hvisor、root linux、root dtb 等文件到一个 fitImage 中，便于在 QEMU 和实际硬件上启动。
用于 ZCU102 平台的 ITS 文件位于 `scripts/zcu102-aarch64-fit.its`: 

```c
/dts-v1/;
/ {
    description = "FIT image for HVISOR with Linux kernel, root filesystem, and DTB";
    images {
        root_linux {
            description = "Linux kernel";
            data = /incbin/("__ROOT_LINUX_IMAGE__");
            type = "kernel";
            arch = "arm64";
            os = "linux";
            ...
        };
        ...
        root_dtb {
            description = "Device Tree Blob";
            data = /incbin/("__ROOT_LINUX_DTB__");
            type = "flat_dt";
            ...
        };
        hvisor {
            description = "Hypervisor";
            data = /incbin/("__HVISOR_TMP_PATH__");
            type = "kernel";
            arch = "arm64";
            ...
        };
    };

    configurations {
        default = "config@1";
        config@1 {
            description = "default";
            kernel = "hvisor";
            fdt = "root_dtb";
        };
    };
};
```

其中，`__ROOT_LINUX_IMAGE__`、`__ROOT_LINUX_DTB__`、`__HVISOR_TMP_PATH__`将通过 Makefile 内的 `sed` 命令替换为实际的路径。在 its 源码中，主要分为 images 和 configurations 两个部分，images 部分定义了要打包的文件，configurations 部分定义了如何组合这些文件，在 UBOOT 启动时，会根据 configurations 中的 default 配置自动加载对应的文件到指定的地址，并且可以通过设置多个 configurations 来支持启动时选择加载不同配置的镜像。

Makefile 中 mkimage 对应的命令：

```Makefile
.PHONY: gen-fit
gen-fit: $(hvisor_bin) dtb
	@if [ ! -f scripts/zcu102-aarch64-fit.its ]; then \
		echo "Error: ITS file scripts/zcu102-aarch64-fit.its not found."; \
		exit 1; \
	fi
	$(OBJCOPY) $(hvisor_elf) --strip-all -O binary $(HVISOR_TMP_PATH)
# now we need to create the vmlinux.bin
	$(GCC_OBJCOPY) $(ROOT_LINUX_IMAGE) --strip-all -O binary $(ROOT_LINUX_IMAGE_BIN)
	@sed \
		-e "s|__ROOT_LINUX_IMAGE__|$(ROOT_LINUX_IMAGE_BIN)|g" \
		-e "s|__ROOT_LINUX_ROOTFS__|$(ROOT_LINUX_ROOTFS)|g" \
		-e "s|__ROOT_LINUX_DTB__|$(ROOT_LINUX_DTB)|g" \
		-e "s|__HVISOR_TMP_PATH__|$(HVISOR_TMP_PATH)|g" \
		scripts/zcu102-aarch64-fit.its > temp-fit.its
	@mkimage -f temp-fit.its $(TARGET_FIT_IMAGE)
	@echo "Generated FIT image: $(TARGET_FIT_IMAGE)"
```


<div class="warning">
    <h3>请注意</h3>
    <p> 不要将已经由 UBOOT 打包的 Image 传入 its 源文件，否则会导致 <b>二次打包</b>！因为 its 中指向的文件应为原始文件（vmlinux 等），mkimage 在导入 its 时对逐个文件进行打包处理（vmlinux->"Image"，然后内嵌到 fitImage）
</div>

## 在 petalinux qemu 中通过 FIT 镜像启动 hvisor 和 root linux

由于 fitImage 一个文件就包括了所有需要的文件，因此对于 qemu 来说只需要通过 loader 把这个文件加载到内存中一个合适的位置即可。

之后 qemu 启动并进入 UBOOT，可以使用下面的命令启动（具体的地址请根据实际情况修改，实际使用时可以把所有行写到一行内 copy 到 UBOOT 进行启动，也可以保存到环境变量 `bootcmd` 中，需要UBOOT挂载一个可持久化的 flash 用于环境变量保存）：

```bash
setenv fit_addr 0x10000000; setenv root_linux_load 0x200000;
imxtract ${fit_addr} root_linux ${root_linux_load}; bootm ${fit_addr};
```

# 参考文献

[1] Flat Image Tree (FIT). <https://docs.u-boot.org/en/stable/usage/fit/> 