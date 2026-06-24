# 安装环境说明
树莓派pico在 Windows、macOS 和 Linux 上提供统一的开发体验，本书就基于linux平台完成开发工作。

> [!NOTE]
>如果你平时使用windows开发，你可以遵循这篇文章来配置环境：https://www.cnblogs.com/Koomee/p/1905224

# Rust 环境搭建

本章只覆盖 Rust 开发、烧录和调试需要的工具。C SDK、`arm-none-eabi-gcc`、`PICO_SDK_PATH` 等内容放在 [C SDK 环境](./c_setup.md) 中。

对于本书的 Rust 示例，最常用的流程有两种：

1. 使用 `picotool` 通过 BOOTSEL 模式烧录。
2. 使用 `probe-rs` 通过 Debug Probe 烧录和调试。

如果你暂时没有 Debug Probe，先配置 `picotool` 就足够完成大多数入门示例。

## Picotool

`picotool` 是 Raspberry Pi 官方工具。它可以查看 UF2/ELF 的元数据，也可以在 Pico 2 处于 BOOTSEL 模式时把程序写入开发板。

[Picotool 仓库](https://github.com/raspberrypi/picotool)

> [!TIP]
> 你也可以从 <a href="https://github.com/raspberrypi/pico-sdk-tools">pico-sdk-tools</a> 下载预编译二进制文件。预编译版本更简单；从源码构建的好处是可以和本地 Pico SDK 版本保持一致。

### 安装构建依赖

Debian / Ubuntu:

```sh
sudo apt install build-essential pkg-config libusb-1.0-0-dev cmake git
```

Arch Linux:

```sh
sudo pacman -S --needed base-devel pkgconf libusb cmake git
```

### 克隆 Pico SDK 和 picotool

这里假设把工具放在 `~/embedded` 下，你可以换成自己的路径。

```sh
mkdir -p ~/embedded
cd ~/embedded

git clone https://github.com/raspberrypi/pico-sdk
git clone https://github.com/raspberrypi/picotool

cd pico-sdk
git submodule update --init
```

### 构建并安装 picotool

`PICO_SDK_PATH` 必须指向你的 Pico SDK 路径。注意这里要使用 `export`，否则子进程中的 CMake 读不到这个变量。

```sh
export PICO_SDK_PATH="$HOME/embedded/pico-sdk"

cd ~/embedded/picotool
cmake -S . -B build -DPICO_SDK_PATH="$PICO_SDK_PATH"
cmake --build build -j8
sudo cmake --install build
```

安装完成后检查：

```sh
picotool version
```

> [!NOTE]
> 只把 `picotool` 二进制放进 `PATH` 能让命令行找到它，但 Pico SDK 的 CMake 还可能找不到 `picotoolConfig.cmake`，然后尝试联网下载 picotool。执行 `sudo cmake --install build` 会同时安装二进制和 CMake package，这是更稳妥的方式。

### 配置 picotool 的 udev 规则

Linux 默认可能不允许普通用户访问 BOOTSEL 模式下的 Pico。安装 udev 规则后，`cargo run` 使用 `picotool` 时就不需要 `sudo`。

```sh
cd ~/embedded/picotool
sudo install -m 0644 udev/60-picotool.rules /etc/udev/rules.d/60-picotool.rules
sudo udevadm control --reload-rules
```

然后拔下并重新插入 Pico。烧录时需要按住 BOOTSEL 再插入 USB。

如果规则文件中引用了 `plugdev`，但你的发行版没有这个组，可以创建它并把当前用户加入进去：

```sh
getent group plugdev || sudo groupadd -r plugdev
sudo usermod -aG plugdev "$USER"
```

修改用户组后需要重新登录，或者重启系统，当前终端会话才一定能拿到新组权限。

## Rust 目标平台

Pico 2 使用 RP2350 芯片。RP2350 同时包含 ARM Cortex-M33 和 Hazard3 RISC-V 内核。本书主要使用 ARM 目标：

```sh
rustup target add thumbv8m.main-none-eabihf
```

如果你要尝试 RISC-V 模式，也可以添加：

```sh
rustup target add riscv32imac-unknown-none-elf
```

检查已安装目标：

```sh
rustup target list --installed
```

## probe-rs - 烧录与调试工具

`probe-rs` 是 Rust 嵌入式生态常用的烧录和调试工具。使用 Raspberry Pi Debug Probe、CMSIS-DAP、J-Link、ST-Link 等调试探针时，可以用它来下载程序、查看 RTT 日志和调试。

使用官方安装脚本：

```bash
curl -LsSf https://github.com/probe-rs/probe-rs/releases/latest/download/probe-rs-tools-installer.sh | sh
```

安装后检查：

```sh
probe-rs --version
cargo flash --version
cargo embed --version
```

最新说明以 [probe-rs 官方文档](https://probe.rs/) 为准。

### 配置 probe-rs 的 udev 规则

默认情况下，Linux 上的调试探针通常只能由 root 访问。安装 udev 规则后，普通用户也可以使用 `probe-rs`。

```sh
curl -Lsf https://probe.rs/files/69-probe-rs.rules -o /tmp/69-probe-rs.rules
sudo install -m 0644 /tmp/69-probe-rs.rules /etc/udev/rules.d/69-probe-rs.rules
sudo udevadm control --reload-rules
```

如果系统没有 `plugdev` 组，同样需要创建并加入：

```sh
getent group plugdev || sudo groupadd -r plugdev
sudo usermod -aG plugdev "$USER"
```

完成后拔下并重新插入 Debug Probe。修改用户组后需要重新登录。

检查是否能枚举调试探针：

```sh
probe-rs list
```

没有连接 Debug Probe 时，看到 `No debug probes were found.` 是正常的；如果出现权限错误，优先检查 udev 规则是否已重载、当前用户是否已经重新登录。

## Cargo runner

本书示例通常在 `.cargo/config.toml` 中配置 runner：

```toml
[target.'cfg(all(target_arch = "arm", target_os = "none"))']
runner = "picotool load -u -v -x -t elf"

[build]
target = "thumbv8m.main-none-eabihf"
```

这样执行 `cargo run` 时，Cargo 会先编译 ELF，然后调用：

```sh
picotool load -u -v -x -t elf <your-program.elf>
```

如果你已经安装了 udev 规则，runner 中不应该再写 `sudo picotool ...`。把 `sudo` 写进 runner 会让自动化构建和 IDE 集成变复杂。

使用 Debug Probe 时，可以改用 `probe-rs`：

```toml
runner = "probe-rs run --chip RP2350"
```

也可以直接使用：

```sh
cargo flash --chip RP2350 --release
cargo embed --chip RP2350 --release
```

## 验证环境

可以用一个 Pico 2 Rust 示例验证环境：

```sh
git clone https://github.com/ImplFerris/pico2-quick
cd pico2-quick
cargo build
```

如果 Pico 2 已经以 BOOTSEL 模式连接：

```sh
cargo run
```

也可以单独检查 `picotool` 是否能访问 USB：

```sh
picotool info
```

没有连接 BOOTSEL 设备时，`No accessible RP-series devices in BOOTSEL mode were found.` 是正常结果。若在容器或沙箱中运行，可能会看到 `Failed to initialise libUSB`，这通常是运行环境没有 USB 访问权限，不一定是系统安装错误。
