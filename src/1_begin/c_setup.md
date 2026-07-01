{{#title 为 Raspberry Pi Pico 2 搭建 C SDK 开发环境 | impl Rust for RP2350}}

# 安装环境说明
树莓派pico在 Windows、macOS 和 Linux 上提供统一的开发体验，本书就基于linux平台完成开发工作。

> [!NOTE]
>如果你平时使用windows开发，你可以遵循这篇文章来配置环境：https://www.cnblogs.com/Koomee/p/19052241

# C SDK 环境搭建

本章覆盖 Raspberry Pi Pico C/C++ SDK 的本地构建环境。即使你主要写 Rust，也建议配置好这一部分，因为 `picotool`、官方 C 示例、部分工具和底层资料都会用到 Pico SDK。

Rust 开发所需的 `rustup target`、`probe-rs`、Cargo runner 等内容放在 [Rust 环境](./rust_setup.md) 中。

对于本书的 C 示例，最常用的流程有两种：

1. 使用 `picotool` 通过 BOOTSEL 模式烧录。
2. 使用 `openocd` 通过 Debug Probe 烧录和调试。

如果你暂时没有 Debug Probe，先配置 `picotool` 就足够完成大多数入门示例。


# AI帮助环境安装：
现在AI Agent开发时代，配置环境啥的也不用再折腾了，直接将我的文档链接发给AI agent即可（这也算是在linux上完成开发的一种优势吧，安装等都不依赖图形界面，非常适合ai自主完成）：
> https://github.com/Zhruoshui/c-rp-rust/blob/main/src/agent_help/rp2350-environment-setup.md

注意：不要开启agent的全自动，最好每步都需要人工审核，第一个是安全，第二个是及时纠正

## 需要安装什么

C SDK 环境主要由五部分组成：

1. 系统构建工具：CMake、Make、C/C++ 编译器、Python、Git。
2. ARM bare-metal 交叉编译器：`arm-none-eabi-gcc`。
3. Pico SDK：官方 SDK 仓库及其子模块。
4. picotool：用于生成 UF2、查看二进制信息和 BOOTSEL 烧录。
5. OpenOCD：通过 Debug Probe 使用 SWD 烧录和调试。

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

> [!IMPORTANT]
> *--注意，后续本书的文件路径等基本都放在这个路径下，如果你需要放在其他位置，请对应的做出修改*

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

## OpenOCD - 烧录与调试工具

`OpenOCD` 是嵌入式开发中常用的开源调试和烧录工具。对于 Pico 2 / RP2350，推荐使用 Raspberry Pi 官方维护的[下游分支](https://github.com/raspberrypi/openocd)，它包含了 RP2350 的完整支持。使用 Raspberry Pi Debug Probe 或其他 CMSIS-DAP 调试探针时，可以用它来下载程序和调试。

Rust 环境对应的工具是 `probe-rs`，相关内容在 [Rust 环境](./rust_setup.md) 中。

### 安装构建依赖

OpenOCD 从源码构建需要一些依赖，安装后可以用 `openocd --version` 验证。

Debian / Ubuntu:

```sh
sudo apt install build-essential pkg-config libusb-1.0-0-dev \
    libtool autoconf automake texinfo libftdi1-dev libhidapi-dev
```

Arch Linux:

```sh
sudo pacman -S --needed base-devel pkgconf libusb libtool autoconf \
    automake texinfo libftdi hidapi
```

### 克隆、构建并安装 OpenOCD

这里假设把工具放在 `~/embedded` 下，并把 OpenOCD 安装到同一个工作区中的 `openocd-install`，这样不需要 `sudo make install`，也不会和系统仓库里的 OpenOCD 混在一起：

```sh
cd ~/embedded
git clone https://github.com/raspberrypi/openocd --branch rpi-common --depth=1

cd openocd
git submodule update --init --recursive
./bootstrap
./configure \
    --prefix="$HOME/embedded/openocd-install" \
    --enable-ftdi \
    --enable-cmsis-dap \
    --enable-internal-jimtcl \
    --enable-internal-libjaylink \
    --disable-werror
make -j8
make install
```

`--enable-cmsis-dap` 启用 CMSIS-DAP 调试探针支持（Raspberry Pi Debug Probe 就是 CMSIS-DAP）。`--enable-ftdi` 启用 FTDI 芯片支持，这是很多第三方调试器的底层芯片。

`git submodule update --init --recursive` 很重要。OpenOCD 的 `rpi-common` 分支依赖 `jimtcl` 和 `libjaylink` 子模块；如果只克隆主仓库，配置阶段可能会报 `jimtcl is required but not found`。这里使用 `--enable-internal-jimtcl` 和 `--enable-internal-libjaylink`，优先使用随 OpenOCD 源码固定版本的依赖，减少不同发行版包版本带来的差异。

`--disable-werror` 是为了兼容较新的 GCC。某些发行版上 OpenOCD 下游分支会因为 warning 被当作 error 而构建失败；关闭 Werror 不会关闭正常 warning，只是不让 warning 中断构建。

把 OpenOCD 加入 `PATH`：

```sh
export PATH="$HOME/embedded/openocd-install/bin:$PATH"
```

如果你希望每个新终端都能直接运行 `openocd`，把上面这行写入 `~/.bashrc`、`~/.zshrc` 或你正在使用的 shell 配置文件。

验证安装：

```sh
openocd --version
test -f "$HOME/embedded/openocd-install/share/openocd/scripts/target/rp2350.cfg"
```

### 配置 OpenOCD 的 udev 规则

默认情况下，Linux 上的调试探针通常只能由 root 访问。安装 udev 规则后，普通用户也可以使用 `openocd`：

```sh
sudo install -m 0644 contrib/60-openocd.rules /etc/udev/rules.d/60-openocd.rules
sudo udevadm control --reload-rules
sudo udevadm trigger
```

如果系统没有 `plugdev` 组，同样需要创建并加入：

```sh
getent group plugdev || sudo groupadd -r plugdev
sudo usermod -aG plugdev "$USER"
```

修改用户组后需要重新登录，或者重启系统，当前终端会话才一定能拿到新组权限。

验证是否能连接 Debug Probe：

```sh
openocd -f interface/cmsis-dap.cfg -f target/rp2350.cfg -c "adapter speed 5000" -c "init; exit"
```

没有连接 Debug Probe 时，会看到类似 `Error: unable to find a matching CMSIS-DAP device` 的错误，这是正常的。连接成功时应能看到 CMSIS-DAP 设备信息、`Cortex-M33` 识别信息，以及 `rp2350.cm0` / `rp2350.cm1` examination succeeded。如果出现权限错误，优先检查 udev 规则是否已重载、当前用户是否已经重新登录。

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

`blink.uf2` 可以拖拽到 BOOTSEL 模式下出现的 `RP2350` 磁盘，也可以使用 `picotool` 烧录。

```sh
picotool load -u -v -x build-pico2/blink/blink.uf2
```

如果已经连接 Raspberry Pi Debug Probe，推荐直接使用 OpenOCD 烧录 ELF：

```sh
openocd -f interface/cmsis-dap.cfg -f target/rp2350.cfg \
    -c "adapter speed 5000" \
    -c "program build-pico2/blink/blink.elf verify reset exit"
```

成功时会看到：

```text
** Programming Finished **
** Verify Started **
** Verified OK **
** Resetting Target **
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

### jimtcl is required but not found

如果构建 OpenOCD 时看到：

```text
configure: error: jimtcl is required but not found via pkg-config and system includes
```

通常是没有初始化 OpenOCD 子模块，或者系统没有安装 `jimtcl` 开发文件。推荐使用随源码固定的内部依赖：

```sh
cd ~/embedded/openocd
git submodule update --init --recursive
./bootstrap
./configure \
    --prefix="$HOME/embedded/openocd-install" \
    --enable-ftdi \
    --enable-cmsis-dap \
    --enable-internal-jimtcl \
    --enable-internal-libjaylink \
    --disable-werror
make -j8
make install
```

### OpenOCD 构建时 warning 被当作 error

如果较新的 GCC 让 OpenOCD 因 warning 构建失败，可以在配置阶段加入：

```sh
--disable-werror
```

这只是不把 warning 升级为 error，不会关闭编译器 warning 本身。
