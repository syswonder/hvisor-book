# 如何编译

## 使用 Docker 编译
### 1. 安装 Docker
```bash
sudo snap install docker
```
也可参考 [Docker 官方文档](https://docs.docker.com/install/) 安装 Docker。

### 2. 构建镜像

```bash
make build_docker
```

此步骤构建一个 Docker 镜像，自动构建编译所需的全部依赖。

### 3. 运行容器

```bash
make docker
```
此步骤会启动一个容器，将当前目录挂载到容器中，并进入容器的 shell。

### 4. 编译

在容器中执行以下命令编译即可编译。
```bash
make all
```

## 使用本地环境编译

### 1. 安装 RustUp 与 Cargo

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | \
    sh -s -- -y --no-modify-path --profile minimal --default-toolchain nightly
```

### 2. 安装工具链

目前项目使用的工具链如下：

 - Rust nightly 2023-07-12
 - [rustfmt](https://crates.io/crates/rustfmt)
 - [clippy](https://crates.io/crates/clippy)
 - [cargo-binutils](https://crates.io/crates/cargo-binutils/0.3.6)
 - rust-src
 - llvm-tools-preview
 - taget: aarch64-unknown-none

可以自行检查是否安装了这些工具，也可以使用以下命令安装：

#### (1) 安装 toml-cli 和 cargo-binutils

```bash
cargo install toml-cli cargo-binutils
```

#### (2) 安装目标平台交叉编译工具链

```bash
rustup target add aarch64-unknown-none
```

#### (3) 解析 rust-toolchain.toml 安装 Rust 工具链

```bash
RUST_VERSION=$(toml get -r rust-toolchain.toml toolchain.channel) && \
Components=$(toml get -r rust-toolchain.toml toolchain.components | jq -r 'join(" ")') && \
rustup install $RUST_VERSION && \
rustup component add --toolchain $RUST_VERSION $Components
```
#### (4) 编译

```bash
make all
```
