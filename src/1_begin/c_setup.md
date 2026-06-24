{{#title 为 Raspberry Pi Pico 2 搭建 C SDK 开发环境 | impl Rust for RP2350}}

# 安装环境说明
树莓派pico在 Windows、macOS 和 Linux 上提供统一的开发体验，本书就基于linux平台完成开发工作。

> [!NOTE]
>如果你平时使用windows开发，你可以遵循这篇文章来配置环境：https://www.cnblogs.com/Koomee/p/1905224

# C SDK 环境搭建

本章覆盖 Raspberry Pi Pico C/C++ SDK 的本地构建环境。即使你主要写 Rust，也建议配置好这一部分，因为 `picotool`、官方 C 示例、部分工具和底层资料都会用到 Pico SDK。

Rust 开发所需的 `rustup target`、`probe-rs`、Cargo runner 等内容放在 [Rust 环境](./rust_setup.md) 中。

# AI帮助环境安装：
现在AI Agent开发时代，配置环境啥的也不用再折腾了，直接将我的文档链接发给AI agent即可：
> https://github.com/Zhruoshui/c-rp-rust/blob/main/src/agent_help/rp2350-environment-setup.md

注意：不要开启agent的全自动，最好每步都需要人工审核，第一个是安全，第二个是及时纠正

## 需要安装什么

C SDK 环境主要由四部分组成：

1. 系统构建工具：CMake、Make、C/C++ 编译器、Python、Git。
2. ARM bare-metal 交叉编译器：`arm-none-eabi-gcc`。
3. Pico SDK：官方 SDK 仓库及其子模块。
4. picotool：用于生成 UF2、查看二进制信息和 BOOTSEL 烧录。

## 安装系统依赖

Debian / Ubuntu:

```sh
sudo apt install build-essential cmake git python3 pkg-config libusb-1.0-0-dev
sudo apt install gcc-arm-none-eabi binutils-arm-none-eabi libnewlib-arm-none-eabi
```

Arch Linux:

```sh
sudo pacman -S --needed base-devel cmake git python pkgconf libusb
sudo pacman -S --needed arm-none-eabi-gcc arm-none-eabi-binutils arm-none-eabi-newlib
```

验证交叉编译器：

```sh
arm-none-eabi-gcc --version
arm-none-eabi-objcopy --version
```

如果 CMake 报错：

```text
Compiler 'arm-none-eabi-gcc' not found
```

说明交叉编译器没有安装，或者不在 `PATH` 中。工具链装在非标准路径时，可以设置 `PICO_TOOLCHAIN_PATH`。

## 获取 Pico SDK

这里假设把 SDK 放在 `~/embedded` 下：

```sh
mkdir -p ~/embedded
cd ~/embedded

git clone https://github.com/raspberrypi/pico-sdk
cd pico-sdk
git submodule update --init
```

`git submodule update --init` 会拉取 TinyUSB、BTstack、cyw43-driver、lwIP、mbed TLS 等 SDK 依赖。只初始化 `lib/mbedtls` 可以满足部分工具构建，但 USB、网络或 Pico W 相关示例会被跳过。

检查子模块状态：

```sh
git submodule status
```

每一行前面是空格，表示该子模块已经检出；如果是 `-`，表示还没有初始化。

## 设置 PICO_SDK_PATH

很多 CMake 项目会通过环境变量找到 Pico SDK：

```sh
export PICO_SDK_PATH="$HOME/embedded/pico-sdk"
```

如果你希望每个新终端都自动生效，可以把上面这行加入 `~/.bashrc`、`~/.zshrc` 或你正在使用的 shell 配置文件。

也可以不设置环境变量，改为每次 CMake 配置时传入：

```sh
cmake -S . -B build -DPICO_SDK_PATH="$HOME/embedded/pico-sdk"
```

> [!IMPORTANT]
> `PICO_SDK_PATH=/path/to/pico-sdk` 只会给当前 shell 变量赋值；如果没有 `export`，CMake 子进程不一定能读到它。

## 安装 picotool

C SDK 在生成 UF2、后处理二进制时会用到 `picotool`。推荐把 `picotool` 安装成完整的 CMake package，而不是只把二进制文件复制到 `PATH`。

```sh
cd ~/embedded
git clone https://github.com/raspberrypi/picotool

cd picotool
cmake -S . -B build -DPICO_SDK_PATH="$PICO_SDK_PATH"
cmake --build build -j8
sudo cmake --install build
```

验证：

```sh
picotool version
```

如果 CMake 配置 Pico 示例时出现类似提示：

```text
No installed picotool with version ... found - building from source
Downloading Picotool
```

说明 Pico SDK 没找到已安装的 `picotool` CMake package。执行 `sudo cmake --install build` 后，系统通常会拥有：

```text
/usr/local/bin/picotool
/usr/local/lib/cmake/picotool/picotoolConfig.cmake
```

这样 Pico SDK 的 CMake 就能直接使用本地 `picotool`，不会在配置时尝试联网下载。

## 配置 udev 规则

如果需要通过 BOOTSEL 模式烧录，安装 `picotool` 的 udev 规则：

```sh
cd ~/embedded/picotool
sudo install -m 0644 udev/60-picotool.rules /etc/udev/rules.d/60-picotool.rules
sudo udevadm control --reload-rules
```

如果规则引用 `plugdev`，但系统没有这个组：

```sh
getent group plugdev || sudo groupadd -r plugdev
sudo usermod -aG plugdev "$USER"
```

然后**重新登录,或者重启电脑**，并拔下重新插入 Pico。

## 构建官方示例验证环境

克隆官方示例：

```sh
cd ~/embedded
git clone https://github.com/raspberrypi/pico-examples
```

配置 Pico 2 构建目录：

```sh
cd ~/embedded/pico-examples
cmake -S . -B build-pico2 -DPICO_BOARD=pico2
```

构建 `blink`：

```sh
cmake --build build-pico2 --target blink -j8
```

成功后会生成：

```text
build-pico2/blink/blink.elf
build-pico2/blink/blink.uf2
build-pico2/blink/blink.bin
```

`blink.uf2` 可以拖拽到 BOOTSEL 模式下出现的 RPI-RP2 磁盘，也可以使用 `picotool` 烧录。

```sh
picotool load -u -v -x build-pico2/blink/blink.uf2
```

## 常见问题

### SDK location was not specified

错误示例：

```text
SDK location was not specified. Please set PICO_SDK_PATH
```

解决方式：

```sh
export PICO_SDK_PATH="$HOME/embedded/pico-sdk"
```

或者给 CMake 显式传参：

```sh
cmake -S . -B build -DPICO_SDK_PATH="$HOME/embedded/pico-sdk"
```

### TinyUSB submodule has not been initialized

错误或警告示例：

```text
TinyUSB submodule has not been initialized; USB support will be unavailable
```

解决方式：

```sh
cd "$PICO_SDK_PATH"
git submodule update --init
```

如果只想补 TinyUSB：

```sh
git submodule update --init lib/tinyusb
```

### No installed picotool found

如果你只把 `picotool` 可执行文件放进 `PATH`，命令行可以运行 `picotool`，但 Pico SDK 的 CMake 仍可能找不到 `picotoolConfig.cmake`。推荐在 `picotool` 构建目录执行：

```sh
sudo cmake --install build
```

### No accessible RP-series devices

错误示例：

```text
No accessible RP-series devices in BOOTSEL mode were found.
```

通常说明 Pico 没有进入 BOOTSEL 模式，或者 udev 规则没有对当前设备生效。按住 BOOTSEL 插入 USB，并重新插拔设备。如果刚刚修改过用户组，先重新登录。

### Failed to initialise libUSB

如果在容器、远程沙箱或受限环境中运行 `picotool`，可能会看到：

```text
ERROR: Failed to initialise libUSB
```

这通常不是 Pico SDK 安装问题，而是当前运行环境没有 USB 设备访问权限。请在普通终端中再次运行 `picotool info` 确认。
