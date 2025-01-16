# UBOOT FIT镜像制作、加载与启动
wheatfox (enkerewpo@hotmail.com)
## ITS源文件
ITS源文件是uboot生成FIT镜像（FIT Image）的源码，即Image Tree Source，其采用Device Tree Source（DTS）的格式，可以通过uboot提供的工具mkimage生成FIT镜像。
在hvisor的ZCU102 port中，使用FIT镜像打包hvisor、root linux、root dtb等文件到一个fitImage中，便于在QEMU和实际硬件上启动。
用于ZCU102平台的ITS文件位于 `scripts/zcu102-aarch64-fit.its`: 

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

其中，`__ROOT_LINUX_IMAGE__`、`__ROOT_LINUX_DTB__`、`__HVISOR_TMP_PATH__`将通过Makefile内使用的sed命令替换为实际的路径。在its源码中，主要分为images和configurations两个部分，images部分定义了要打包的文件，configurations部分定义了如何组合这些文件，在UBOOT启动时，会根据configurations中的default配置自动加载对应的文件到指定的地址，并且可以通过设置多个configurations来支持启动时选择加载不同配置的镜像。

Makefile中mkimage对应的命令：

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
    <p> 不要将已经由UBOOT打包的Image传入its源文件，否则会导致<b>二次打包</b>！因为its中指向的文件应为原始文件（vmlinux等），mkimage在导入its时对逐个文件进行打包处理（vmlinux->"Image"，然后内嵌到fitImage）
</div>

## 在petalinux qemu中通过FIT镜像启动hvisor和root linux

由于fitImage一个文件就包括了所有需要的文件，因此对于qemu来说只需要通过loader把这个文件加载到内存中一个合适的位置即可。

之后qemu启动并进入UBOOT，可以使用下面的命令启动（具体的地址请根据实际情况修改，实际使用时可以把所用行写到一行内copy到UBOOT进行启动，也可以保存到bootcmd中）：

```bash
setenv fit_addr 0x10000000;
setenv root_linux_load 0x200000;
setenv root_rootfs_load 0x4000000;
imxtract ${fit_addr} root_linux ${root_linux_load};
imxtract ${fit_addr} root_rootfs ${root_rootfs_load};
md ${root_linux_load} 20;
bootm ${fit_addr};
```

# 参考文献

[1] Flat Image Tree (FIT). <https://docs.u-boot.org/en/stable/usage/fit/> 