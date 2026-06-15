# RP2350 / Raspberry Pi Pico 2 环境搭建方案

本文面向 AI agent 或维护者，用于快速复现 Raspberry Pi Pico 2 / RP2350 的 Rust 与 C SDK 开发环境。它不是面向读者的叙事章节，而是一份可执行的检查清单、安装步骤和排错记录。

## 目标

把一台 Linux 机器配置到以下状态：

- 可以编译 RP2350 Rust 固件。
- 可以编译 Pico C SDK 示例。
- `picotool` 可以被命令行和 Pico SDK CMake 同时找到。
- `probe-rs`、`cargo flash`、`cargo embed` 可用。
- 普通用户可以通过 udev 规则访问 BOOTSEL 模式下的 Pico 和 Debug Probe。

## 本仓库当前路径约定

当前项目根目录：

```sh
/home/ruoshui/Documents/MyProject/raspberrypi/rp2350
```

当前本地 Pico SDK：

```sh
/home/ruoshui/Documents/MyProject/raspberrypi/rp2350/pico-sdk
```

当前本地 picotool 源码：

```sh
/home/ruoshui/Documents/MyProject/raspberrypi/rp2350/picotool
```

项目环境脚本：

```sh
/home/ruoshui/Documents/MyProject/raspberrypi/rp2350/pico-env.sh
```

在本项目中工作时，优先执行：

```sh
cd /home/ruoshui/Documents/MyProject/raspberrypi/rp2350
. ./pico-env.sh
```

## 前置检查

先确认系统、工具和已有状态：

```sh
cat /etc/os-release

for c in cmake make gcc g++ pkg-config rustup rustc cargo picotool probe-rs arm-none-eabi-gcc; do
  printf '%-20s' "$c"
  command -v "$c" || true
done

rustup target list --installed
git -C pico-sdk submodule status
```

对 Arch Linux，尤其注意：

- `apt` 命令不可用，使用 `pacman`。
- `plugdev` 组可能不存在，需要手动创建。

## 安装系统依赖

Arch Linux:

```sh
sudo pacman -S --needed base-devel cmake git python pkgconf libusb
sudo pacman -S --needed arm-none-eabi-gcc arm-none-eabi-binutils arm-none-eabi-newlib
```

Debian / Ubuntu:

```sh
sudo apt install build-essential cmake git python3 pkg-config libusb-1.0-0-dev
sudo apt install gcc-arm-none-eabi binutils-arm-none-eabi libnewlib-arm-none-eabi
```

验证：

```sh
arm-none-eabi-gcc --version
arm-none-eabi-objcopy --version
```

若 CMake 报：

```text
Compiler 'arm-none-eabi-gcc' not found
```

说明 C SDK 交叉编译器缺失，或者工具链不在 `PATH` 中。

## Rust target

RP2350 ARM 模式需要：

```sh
rustup target add thumbv8m.main-none-eabihf
```

RISC-V 模式可选：

```sh
rustup target add riscv32imac-unknown-none-elf
```

验证：

```sh
rustup target list --installed | sort
```

期望至少包含：

```text
thumbv8m.main-none-eabihf
riscv32imac-unknown-none-elf
```

## Pico SDK

如果 SDK 不存在：

```sh
git clone https://github.com/raspberrypi/pico-sdk
```

初始化子模块：

```sh
git -C pico-sdk submodule update --init
```

只初始化 `lib/mbedtls` 可以构建部分工具，但会导致 TinyUSB、lwIP、cyw43、BTstack 相关示例被跳过。完整开发环境建议初始化全部一级子模块。

验证：

```sh
git -C pico-sdk submodule status
```

每行前面是空格表示已检出；前面是 `-` 表示尚未初始化。

## PICO_SDK_PATH

设置当前 shell：

```sh
export PICO_SDK_PATH="/home/ruoshui/Documents/MyProject/raspberrypi/rp2350/pico-sdk"
```

本项目已提供：

```sh
. ./pico-env.sh
```

注意：只写 `PICO_SDK_PATH=/path/to/pico-sdk` 不够，必须 `export`，否则 CMake 子进程可能读不到。

## picotool

`picotool` 有两个层面的可发现性：

- 命令行能执行 `picotool`。
- Pico SDK CMake 能找到 `picotoolConfig.cmake`。

只把二进制放进 `PATH` 只能解决第一点。若第二点缺失，Pico SDK CMake 可能尝试联网克隆 picotool。

从源码构建并安装：

```sh
cd /home/ruoshui/Documents/MyProject/raspberrypi/rp2350/picotool
cmake -S . -B build -DPICO_SDK_PATH="$PICO_SDK_PATH"
cmake --build build -j8
sudo cmake --install build
```

验证：

```sh
picotool version
test -f /usr/local/lib/cmake/picotool/picotoolConfig.cmake
```

成功后 Pico SDK CMake 输出应包含：

```text
Using picotool from /usr/local/bin/picotool
```

## udev 规则

安装 picotool 规则：

```sh
sudo install -m 0644 picotool/udev/60-picotool.rules /etc/udev/rules.d/60-picotool.rules
```

安装 probe-rs 规则：

```sh
curl -Lsf https://probe.rs/files/69-probe-rs.rules -o /tmp/69-probe-rs.rules
sudo install -m 0644 /tmp/69-probe-rs.rules /etc/udev/rules.d/69-probe-rs.rules
```

Arch Linux 可能没有 `plugdev` 组：

```sh
getent group plugdev || sudo groupadd -r plugdev
sudo usermod -aG plugdev "$USER"
```

重载规则：

```sh
sudo udevadm control --reload-rules
```

然后重新插拔 Pico / Debug Probe。修改用户组后，重新登录或重启终端会话。

确认真实系统权限：

```sh
stat -c '%U:%G %a %n' /etc/udev/rules.d/60-picotool.rules /etc/udev/rules.d/69-probe-rs.rules
```

期望：

```text
root:root 644 /etc/udev/rules.d/60-picotool.rules
root:root 644 /etc/udev/rules.d/69-probe-rs.rules
```

## probe-rs

安装：

```sh
curl -LsSf https://github.com/probe-rs/probe-rs/releases/latest/download/probe-rs-tools-installer.sh | sh
```

验证：

```sh
probe-rs --version
cargo flash --version
cargo embed --version
probe-rs list
```

没有连接 Debug Probe 时，`No debug probes were found.` 是正常结果。若出现权限错误，检查 udev、`plugdev` 和重新登录。

## Rust 示例验证

当前仓库中可用：

```sh
cd /home/ruoshui/Documents/MyProject/raspberrypi/rp2350/pico_learn/pico2-quick
cargo build
```

期望结果：

```text
Finished `dev` profile ... target(s) ...
```

如果 Pico 2 处于 BOOTSEL 模式，可执行：

```sh
cargo run
```

示例 `.cargo/config.toml` 中的 runner 应使用普通用户命令：

```toml
runner = "picotool load -u -v -x -t elf"
```

不要写成：

```toml
runner = "sudo picotool load -u -v -x -t elf"
```

因为这会干扰 IDE、自动化和非交互式执行。

## C SDK 示例验证

配置 Pico 2：

```sh
cd /home/ruoshui/Documents/MyProject/raspberrypi/rp2350
env PICO_SDK_PATH="$PWD/pico-sdk" cmake -S pico_learn/pico-examples -B /tmp/pico-examples-build-full -DPICO_BOARD=pico2
```

构建 blink：

```sh
cmake --build /tmp/pico-examples-build-full --target blink -j8
```

期望产物：

```text
/tmp/pico-examples-build-full/blink/blink.elf
/tmp/pico-examples-build-full/blink/blink.uf2
/tmp/pico-examples-build-full/blink/blink.bin
```

烧录 UF2：

```sh
picotool load -u -v -x /tmp/pico-examples-build-full/blink/blink.uf2
```

## 常见错误和判断

### `Failed to initialise libUSB`

如果在 Codex 沙箱、容器或受限远程环境中运行 `picotool info`，可能看到：

```text
ERROR: Failed to initialise libUSB
```

这通常是运行环境没有 USB 访问权限。应在普通终端或沙箱外运行：

```sh
picotool info
```

若沙箱外输出：

```text
No accessible RP-series devices in BOOTSEL mode were found.
```

说明 libUSB 正常，只是当前没有 BOOTSEL 设备。

### `No accessible RP-series devices in BOOTSEL mode were found`

含义：

- Pico 没有按住 BOOTSEL 插入 USB。
- udev 规则尚未对当前设备生效。
- 修改用户组后还没有重新登录。

处理：

```sh
sudo udevadm control --reload-rules
```

然后重新插拔设备。必要时重新登录。

### Pico SDK 尝试下载 Picotool

错误表现：

```text
No installed picotool with version ... found - building from source
Downloading Picotool
fatal: unable to access 'https://github.com/raspberrypi/picotool.git/'
```

处理：

```sh
cd picotool
sudo cmake --install build
```

然后用新的构建目录重新配置 CMake，避免旧缓存影响：

```sh
cmake -S pico_learn/pico-examples -B /tmp/pico-examples-build-check -DPICO_BOARD=pico2
```

### TinyUSB / lwIP / cyw43 缺失

错误或警告表现：

```text
TinyUSB submodule has not been initialized
lib/lwip submodule needed ... not found
```

处理：

```sh
git -C pico-sdk submodule update --init
```

## 当前机器已完成状态

在 2026-06-15 的本机配置中，已完成：

- `thumbv8m.main-none-eabihf` Rust target 已安装。
- `riscv32imac-unknown-none-elf` Rust target 已安装。
- `arm-none-eabi-gcc` / `binutils` / `newlib` 已安装。
- `picotool v2.2.0-a4` 已安装到 `/usr/local/bin/picotool`。
- `/usr/local/lib/cmake/picotool/picotoolConfig.cmake` 已存在。
- `probe-rs 0.31.0`、`cargo flash`、`cargo embed` 已可用。
- `/etc/udev/rules.d/60-picotool.rules` 和 `/etc/udev/rules.d/69-probe-rs.rules` 已安装。
- `plugdev` 组已创建，用户 `ruoshui` 已加入。
- Pico SDK 子模块 `btstack`、`cyw43-driver`、`lwip`、`mbedtls`、`tinyusb` 已初始化。
- Rust 示例 `pico_learn/pico2-quick` 可 `cargo build`。
- C SDK 示例 `pico_learn/pico-examples` 的 `blink` 可构建并生成 UF2。
