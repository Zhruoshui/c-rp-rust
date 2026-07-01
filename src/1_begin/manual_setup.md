# 手动配置开发环境（可选）

本节保留完全手动安装路线。它适合这些情况：

- 不使用 VS Code 插件，只想在终端里构建。
- 想把 SDK 和工具放在自己指定的位置，比如 `~/embedded`。
- 需要修改或调试 Pico SDK、picotool、OpenOCD 源码。
- 要在 CI、远程服务器或容器中固定一套工具链。

如果你已经安装了官方 VS Code 插件，并且 `~/.pico-sdk` 已经配置好，优先使用 [C 环境](./c_setup.md) 里的插件复用路线即可。

## 目录约定

下面仍然沿用本书早期的手动目录：

```sh
mkdir -p ~/embedded
cd ~/embedded
```

如果你换成其他路径，后面的 `PICO_SDK_PATH`、`PICO_TOOLCHAIN_PATH`、OpenOCD 路径都要对应修改。

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

## 获取 Pico SDK

```sh
cd ~/embedded
git clone https://github.com/raspberrypi/pico-sdk

cd pico-sdk
git submodule update --init
```

设置环境变量：

```sh
export PICO_SDK_PATH="$HOME/embedded/pico-sdk"
```

如果希望每个新终端都自动生效，可以把上面这行加入 `~/.bashrc`、`~/.zshrc` 或你正在使用的 shell 配置文件。

也可以不写入 shell 配置，而是在每次配置 CMake 时显式传入：

```sh
cmake -S . -B build -DPICO_SDK_PATH="$HOME/embedded/pico-sdk"
```

> [!IMPORTANT]
> `PICO_SDK_PATH=/path/to/pico-sdk` 只会给当前 shell 变量赋值；如果没有 `export`，CMake 子进程不一定能读到它。

## 构建并安装 picotool

`picotool` 用来查看 UF2/ELF 元数据，也可以在 Pico 处于 BOOTSEL 模式时烧录程序。推荐安装成完整 CMake package，而不是只复制二进制。

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

安装后系统通常会有：

```text
/usr/local/bin/picotool
/usr/local/lib/cmake/picotool/picotoolConfig.cmake
```

这样 Pico SDK 的 CMake 能找到本地 `picotool`，不会在配置项目时尝试下载。

## 配置 picotool udev 规则

Linux 默认可能不允许普通用户访问 BOOTSEL 模式下的 Pico。安装 udev 规则后，`picotool` 通常不需要 `sudo`。

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

然后重新登录，或者重启电脑，并拔下重新插入 Pico。

## 构建 OpenOCD

对于 Pico 2 / RP2350，推荐使用 Raspberry Pi 官方维护的 [openocd](https://github.com/raspberrypi/openocd) 下游分支，它包含 RP2350 支持。

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

克隆、构建并安装到 `~/embedded/openocd-install`：

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

加入 `PATH`：

```sh
export PATH="$HOME/embedded/openocd-install/bin:$PATH"
```

验证：

```sh
openocd --version
test -f "$HOME/embedded/openocd-install/share/openocd/scripts/target/rp2350.cfg"
```

配置 OpenOCD udev 规则：

```sh
cd ~/embedded/openocd
sudo install -m 0644 contrib/60-openocd.rules /etc/udev/rules.d/60-openocd.rules
sudo udevadm control --reload-rules
sudo udevadm trigger
```

连接 Debug Probe 后验证：

```sh
openocd -f interface/cmsis-dap.cfg \
    -f target/rp2350.cfg \
    -c "adapter speed 5000" \
    -c "init; exit"
```

## 手动安装 RISC-V 工具链

Pico 2 的 RISC-V 模式需要单独的 RISC-V 裸机工具链。官方 SDK 文档推荐设置：

```sh
export PICO_TOOLCHAIN_PATH=/path/to/riscv/toolchain
export PICO_PLATFORM=rp2350-riscv
```

如果你使用发行版包，例如 Arch Linux：

```sh
sudo pacman -S riscv64-elf-gcc riscv64-elf-newlib riscv64-elf-binutils
```

构建时可以显式指定 triple：

```sh
cmake -S . -B build-riscv \
    -DPICO_BOARD=pico2 \
    -DPICO_PLATFORM=rp2350-riscv \
    -DPICO_GCC_TRIPLE=riscv64-elf
cmake --build build-riscv -j
```

如果使用官方插件下载到 `~/.pico-sdk/toolchain/` 的 RISC-V 工具链，则不需要这一步，直接看 [RISC-V 初探](./riscv_start.md)。
