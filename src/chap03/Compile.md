# 如何编译


## Rust环境配置

可以参考[Rust 语言圣经](https://course.rs/first-try/intro.html)配置Rust开发环境，也可参考如下的操作。

### 1. 安装 RustUp 与 Cargo

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | \
    sh -s -- -y --no-modify-path --profile minimal --default-toolchain nightly
```

### 2. 安装工具链

目前项目使用的工具链如下：

 - Rust nightly-2024-05-05
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

## 编译hvisor
首先将 [hvisor 代码仓库](https://github.com/syswonder/hvisor) 拉到本地，并切换到 dev 分支。
```bash
git clone -b dev https://github.com/syswonder/hvisor.git
```

运行下面这条命令进行编译：
```bash
make all
```

