# RP2350 / Raspberry Pi Pico 2 环境搭建方案

本文面向 AI agent 或维护者，用于快速复现 Raspberry Pi Pico 2 / RP2350 的 Rust 与 C SDK 开发环境。它不是面向读者的叙事章节，而是一份可执行的检查清单、安装步骤和排错记录。

## 先和用户确认的路径约定

在执行任何安装命令前，Agent 应先和用户确认一个统一的 RP2350 工作区目录。后续所有 SDK、工具源码、示例项目和环境脚本都放在这个目录下，避免把依赖散落到不同位置。

用户确认路径后，本文统一用 `$RP2350_WORKSPACE` 指代这个目录。后续命令都应基于这个变量展开，不要把某台机器上的个人路径写死到文档或脚本中。

```sh
export RP2350_WORKSPACE="<用户提供的 RP2350 工作区绝对路径>"
cd "$RP2350_WORKSPACE"
```

## 先处理系统级安装权限

Agent 通常没有，也不应该假设自己拥有 `sudo` 权限。因此，在执行克隆、构建、配置 SDK 等步骤之前，默认应先提醒用户手动安装系统依赖包。若用户明确表示已经授权 Agent 执行 `sudo` 或审批了对应的提权命令，Agent 可以代为执行，但必须让命令保持可审计、非破坏性，并在执行前说明用途。

Agent 的处理顺序：

1. 读取 `/etc/os-release` 判断用户系统。
2. 把对应发行版的安装命令展示给用户，让用户在自己的终端中手动执行；或者在用户明确授权后由 Agent 执行。
3. 如果由用户手动执行，则暂停后续安装流程，等待用户确认系统包已经安装完成。
4. 用户确认后，Agent 先执行“安装完善性检查”。
5. 只有检查通过后，Agent 才继续执行 Rust target、Pico SDK、picotool、probe-rs 和示例验证等后续步骤。

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

安装完善性检查：

```sh
for c in cmake make gcc g++ git pkg-config arm-none-eabi-gcc arm-none-eabi-objcopy; do
  printf '%-24s' "$c"
  command -v "$c" || exit 1
done

arm-none-eabi-gcc --version
arm-none-eabi-objcopy --version
```

如果检查失败，Agent 应指出缺失的命令，并让用户补装对应系统包，而不是继续执行后续环境配置。

## 安装过程概览

正式安装前，先告诉用户这套流程主要会做这些事：

1. 让用户手动安装系统级构建工具和交叉编译器，例如 CMake、GCC、libusb、`arm-none-eabi-gcc`。
2. 给 Rust 添加 RP2350 所需 target，至少包括 ARM 模式的 `thumbv8m.main-none-eabihf`，RISC-V target 可按需安装。
3. 克隆并初始化 `pico-sdk`，包括 TinyUSB、lwIP、cyw43、BTstack 等子模块。
4. 配置 `PICO_SDK_PATH`，并建议写入 `pico-env.sh`，让后续 shell、CMake 和 Agent 都能稳定找到 Pico SDK。
5. 从源码构建并安装 `picotool`，确保命令行和 Pico SDK CMake 都能找到它。
6. 安装 udev 规则，让普通用户可以访问 BOOTSEL 模式下的 Pico 和 Debug Probe。
7. 安装 `probe-rs`、`cargo flash`、`cargo embed`，用于 Rust 固件烧录和调试。
8. 分别用 Rust 示例和 C SDK 示例验证环境是否可用。

这套流程会写入用户的工作区目录，并且后续仍可能涉及需要 `sudo` 的操作，例如安装 `/usr/local/bin/picotool`、写入 `/etc/udev/rules.d/`，以及修改当前用户所属用户组。执行这些步骤前应明确告知用户；若用户不授权 Agent 提权执行，则让用户手动运行对应命令。

## 目标

把一台 Linux 机器配置到以下状态：

- 可以编译 RP2350 Rust 固件。
- 可以编译 Pico C SDK 示例。
- `picotool` 可以被命令行和 Pico SDK CMake 同时找到。
- `probe-rs`、`cargo flash`、`cargo embed` 可用。
- 普通用户可以通过 udev 规则访问 BOOTSEL 模式下的 Pico 和 Debug Probe。

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

这一节的命令需要 `sudo`。默认由用户手动执行；如果用户明确授权 Agent 提权执行，Agent 可以代为运行。用户完成或 Agent 执行完成后，必须先运行本节的验证命令。

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
for c in cmake make gcc g++ git pkg-config arm-none-eabi-gcc arm-none-eabi-objcopy; do
  printf '%-24s' "$c"
  command -v "$c" || exit 1
done

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
export PICO_SDK_PATH="$RP2350_WORKSPACE/pico-sdk"
```

如果工作区中已经创建了 `pico-env.sh`，也可以直接加载：

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
cd "$RP2350_WORKSPACE/picotool"
cmake -S . -B build -DPICO_SDK_PATH="$PICO_SDK_PATH"
cmake --build build -j8
```

安装到系统目录需要 `sudo`，由用户手动执行：

```sh
sudo cmake --install build
```

验证：

```sh
picotool version
test -f /usr/local/lib/cmake/picotool/picotoolConfig.cmake
```

实际部署中可能同时存在两个 `picotool` 入口：

- `~/.local/bin/picotool`：指向工作区内 `picotool/build/picotool` 的用户级符号链接，可能会被当前 shell 的 `PATH` 优先命中。
- `/usr/local/bin/picotool`：`sudo cmake --install build` 安装的系统级二进制，同时配套安装 `/usr/local/lib/cmake/picotool/picotoolConfig.cmake`。

只要两者版本一致即可。命令行 `command -v picotool` 命中 `~/.local/bin/picotool` 不代表 CMake 配置有问题；CMake 关键是能找到 `/usr/local/lib/cmake/picotool/picotoolConfig.cmake`。

成功后 Pico SDK CMake 输出应包含：

```text
Using picotool from /usr/local/bin/picotool
```

## udev 规则

以下命令需要 `sudo`。默认由用户手动执行；如果用户明确授权 Agent 提权执行，Agent 可以代为运行。用户完成或 Agent 执行完成后，Agent 再运行后面的权限检查命令。

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

注意：在 Codex 沙箱或受管文件系统里，普通沙箱命令可能把系统文件 owner 显示成 `nobody:nobody`。这不一定表示真实系统权限错误。需要以普通终端或沙箱外命令的 `stat` 结果为准；udev 规则本身至少应保持 `0644`，确保系统能读取。

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
cd "$RP2350_WORKSPACE/pico_learn/pico2-quick"
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
cd "$RP2350_WORKSPACE"
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
