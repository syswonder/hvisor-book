# 如何启动 Root Linux

## QEMU

### 安装依赖

#### 1. 安装依赖

```bash
apt-get install -y jq wget build-essential \
 libglib2.0-0 libfdt1 libpixman-1-0 zlib1g \
 libfdt-dev libpixman-1-dev libglib2.0-dev \
 zlib1g-dev ninja-build
```

#### 1. 下载并解压QEMU

```bash
wget https://download.qemu.org/qemu-7.0.0.tar.xz
tar -xvf qemu-${QEMU_VERSION}.tar.xz
```

#### 2. 条件编译并安装QEMU

这里我们只编译用于仿真 aarch64 的 QEMU，如果需要其他架构的 QEMU，可以参考[QEMU官方文档](https://wiki.qemu.org/Hosts/Linux)。
```bash
cd qemu-7.0.0 && \
./configure --target-list=aarch64-softmmu,aarch64-linux-user && \
make -j$(nproc) && \
make install
```

#### 3. 测试QEMU是否安装成功

```bash
qemu-system-aarch64 --version
```

### 启动Root Linux

#### 1. 准备Root文件系统和内核镜像

将镜像文件放置于```hvisor/images/aarch64/kernel/```,命名为```Image```。

将Root文件系统放置于```hvisor/images/aarch64/virtdisk/```,命名为```rootfs1.ext4```。

#### 2. 启动QEMU
在hviosr目录下执行以下命令：
```bash
make run
```

#### 3. 进入QEMU

将自动加载uboot，等待uboot加载完成后，输入```bootm 0x40400000 - 0x40000000```，即可进入Root Linux。

