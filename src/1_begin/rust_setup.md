# 安装环境说明
树莓派pico在 Windows、macOS 和 Linux 上提供统一的开发体验，本书就基于linux平台完成开发工作。

# Rust 环境搭建
本章只覆盖 Rust 开发、烧录和调试需要的工具。C SDK、`arm-none-eabi-gcc`、`PICO_SDK_PATH` 等内容放在 [C SDK 环境](./c_setup.md) 中。

对于本书的 Rust 示例，最常用的流程有两种：

1. 使用 `picotool` 通过 BOOTSEL 模式烧录。
2. 使用 `probe-rs` 通过 Debug Probe 烧录和调试。

由于使用 Rust 做 Pico 嵌入式开发的人目前仍然比 C/C++ 少，虽然 VS Code 的 Raspberry Pi Pico 插件可以直接新建 Rust 项目，但它生成的模板走的是 [rp-hal](https://github.com/rp-rs/rp-hal) 生态，而不是本书主讲的 [Embassy](https://github.com/embassy-rs/embassy) 生态，所以本书不直接沿用 Pico VS Code 插件生成的 Rust 模板，自己手动配置项目更有利于学习。

## 安装rust

本书的 Rust 示例项目以 Embassy 为主线，示例工程位于：

```text
/home/ruoshui/Documents/MyProject/raspberrypi/rp2350/pico-learn/pico2-quick
```

该项目使用 `embassy-rp` 和 `embassy-executor`，默认构建 Pico 2；如果要构建 Pico 2 W，需要启用 `pico2w` feature。安装环境时重点准备三类工具：

1. Rust 主机工具链：`rustc`、`cargo`、`rustup`。
2. RP2350 ARM 编译目标：`thumbv8m.main-none-eabihf`。
3. 烧录/调试工具：BOOTSEL 路线使用 `picotool`，Debug Probe 路线使用 `probe-rs`、`cargo flash`、`cargo embed`。

### 安装 Rust 工具链

推荐使用 `rustup` 管理 Rust。Linux 下可以直接使用官方安装脚本：

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"
```

安装后使用稳定版工具链，并更新到当前版本：

```bash
rustup default stable
rustup update
rustc --version
cargo --version
```

本书主线使用 Pico 2 的 ARM Cortex-M33 内核，所以需要添加：

```bash
rustup target add thumbv8m.main-none-eabihf
```

如果后续要尝试 RP2350 的 RISC-V 内核，再额外添加：

```bash
rustup target add riscv32imac-unknown-none-elf
```

检查目标是否已经安装：

```bash
rustup target list --installed | grep -E 'thumbv8m.main-none-eabihf|riscv32imac-unknown-none-elf'
```

### 安装烧录和调试工具

如果使用 BOOTSEL 模式烧录，示例项目的 `.cargo/config.toml` 已经把 `cargo run` 配成调用：

```text
picotool load -u -v -x -t elf
```

因此先确认 `picotool` 可用：

```bash
picotool version
```

如果命令不存在，可以先按 C SDK 环境章节安装 `picotool`，或者使用 Raspberry Pi Pico VS Code 插件安装在 `~/.pico-sdk/picotool/` 下的预编译版本。

如果使用 Raspberry Pi Debug Probe 调试或烧录，安装 `probe-rs` 工具链：

```bash
curl -LsSf https://github.com/probe-rs/probe-rs/releases/latest/download/probe-rs-tools-installer.sh | sh
source "$HOME/.cargo/env"

probe-rs --version
cargo flash --version
cargo embed --version
```

Linux 下还需要配置 udev 规则，否则普通用户可能无法访问 BOOTSEL 设备或 Debug Probe。`picotool` 和 `probe-rs` 的 udev 配置分别见后续小节。

### 验证示例项目

进入本书使用的 Embassy 示例项目：

```bash
cd /home/ruoshui/Documents/MyProject/raspberrypi/rp2350/pico-learn/pico2-quick
```

编译 Pico 2 默认版本：

```bash
cargo build-pico2
```

编译 Pico 2 W 版本：

```bash
cargo build-pico2w
```

如果 Pico 2 以 BOOTSEL 模式连接，可以用 `picotool` 路线运行：

```bash
cargo run-pico2
```

Pico 2 W 对应：

```bash
cargo run-pico2w
```

如果使用 Debug Probe，可以用 `probe-rs` 路线烧录或调试：

```bash
cargo flash-pico2
cargo embed-pico2
```

Pico 2 W 对应：

```bash
cargo flash-pico2w
cargo embed-pico2w
```

## Rust 目标平台

Pico 2 使用 RP2350 芯片。RP2350 同时包含 ARM Cortex-M33 和 Hazard3 RISC-V 内核。本书主要使用 ARM target：

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

可以用一个 Pico 2 / Pico 2W 的 Rust 示例验证环境：

```sh
git clone https://github.com/Zhruoshui/pico2-quick
cd pico2-quick
cargo build
```

如果 Pico 2 / Pico 2W 已经以 BOOTSEL 模式连接：

```sh
cargo run --release #　for Pico 2
cargo run --release --no-default-features --features pico2w # for Pico 2W

```

也可以单独检查 `picotool` 是否能访问 USB：

```sh
picotool info
```

没有连接 BOOTSEL 设备时，`No accessible RP-series devices in BOOTSEL mode were found.` 是正常结果。若在容器或沙箱中运行，可能会看到 `Failed to initialise libUSB`，这通常是运行环境没有 USB 访问权限，不一定是系统安装错误。
