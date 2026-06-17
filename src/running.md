# 构建和烧录程序

在深入了解更多示例之前，让我们先了解在树莓派 Pico 2 上构建和运行任何程序的常规步骤。

Pico 2 包含 ARM Cortex-M33 和 Hazard3 RISC-V 处理器，我们将为这两种架构提供说明。

## ARM 构建和烧录
### for c

C 工程的从零搭建（目录结构、`CMakeLists.txt`、`main.c`）已在[《快速入门 for C》](./quick-start-for-c.md)详细讲过，这里只对构建和烧录的常规步骤做最小回顾。沿用那一章的 `pico2-manual-blink` 工程：

```sh
# 构建（Pico 2，ARM Cortex-M33），产物在 build-pico2/
cmake -S . -B build-pico2 -DPICO_BOARD=pico2
cmake --build build-pico2 -j
```

Pico 2 W 只需把 `-DPICO_BOARD=pico2` 换成 `-DPICO_BOARD=pico2_w`。

烧录与《快速入门 for C》的「烧录」一节相同：按住 `BOOTSEL` 接 USB，待 `RP2350` U 盘出现后复制 UF2：

```sh
cp build-pico2/pico2_manual_blink.uf2 /run/media/$USER/RP2350/
```

### for rust
使用此命令在 ARM 模式下在树莓派 Pico 2 / Pico 2 W 上构建和运行程序，利用 Cortex-M33 处理器。

```sh
# 构建程序
cargo build --target=thumbv8m.main-none-eabihf
```

要将应用程序刷新到 Pico 2，请按住 BOOTSEL 按钮。按住时，使用 USB 连接线将 Pico 2 连接到计算机。插入 USB 后可以释放该按钮。

<img style="display: block; margin: auto;" alt="bootsel" src="./images/bootsel.png"/>

```sh
# 烧录程序
cargo run --target=thumbv8m.main-none-eabihf
```

> [!NOTE]
> 示例代码在 `.cargo/config.toml` 文件中包含运行器配置，定义为：
`runner = "picotool load -u -v -x -t elf"`。这意味着当您执行 `cargo run` 时，它实际上调用 `picotool` 的 `load` 子命令来刷新程序。

## RISC-V 构建和运行

在 RISC-V 模式下，程序运行在 Pico 2 的 Hazard3 处理器上。C 与 Rust 的构建方式分别见下。

> [!IMPORTANT]
> 本书重点介绍 ARM。某些示例在 RISC-V 模式下工作前可能需要进行修改。为简便起见，建议在阅读本书时遵循 ARM 工作流程。

### for c

RISC-V 模式**不需要改动源码，也不需要改 `CMakeLists.txt`**——同一个 `pico2-manual-blink` 工程，只在配置时把平台切到 `rp2350-riscv`，并指定 RISC-V 工具链：

```sh
# 构建（Hazard3 RISC-V），产物在 build-riscv/
cmake -S . -B build-riscv \
    -DPICO_BOARD=pico2 \
    -DPICO_PLATFORM=rp2350-riscv \
    -DPICO_GCC_TRIPLE=riscv64-elf
cmake --build build-riscv -j
```

生成的 UF2 文件在 `build-riscv/pico2_manual_blink.uf2`。

> [!NOTE]
> RISC-V 需要单独的编译器，ARM 的 `arm-none-eabi-gcc` 不能用。这里用 `riscv64-elf` 多库工具链，它能生成 RP2350 所需的 `rv32imac/ilp32` 代码。Arch Linux 安装：`sudo pacman -S riscv64-elf-gcc riscv64-elf-newlib riscv64-elf-binutils`。Hazard3 没有硬件 FPU，浮点走软件库（soft-float）。

烧录方式与 ARM 完全相同（BOOTSEL + 复制 UF2）：

```sh
cp build-riscv/pico2_manual_blink.uf2 /run/media/$USER/RP2350/
```

> [!NOTE]
> 板子架构跟着固件走：烧 RISC-V 固件，上电即以 Hazard3 RISC-V 启动；烧回 ARM 固件即切回。UF2 中带有架构标记，bootrom 据此自动切换，并非不可逆的 OTP 熔丝操作。

> [!TIP]
> 每次手敲这串 `-D...` 参数比较繁琐。可以把两种架构的配置固化成项目根目录的 `CMakePresets.json`（之后只需 `cmake --preset arm` / `cmake --preset riscv`），再配一个按架构烧录的 `flash.sh`，构建和烧录就各剩一行命令。完整示例见 `pico2-c-blink/blink` 工程的 `README.md` 和 `CMakePresets.json`。

### for rust

```sh
# 构建程序
cargo build --target=riscv32imac-unknown-none-elf
```

按照上面描述的相同 BOOTSEL 步骤进行操作。

```sh
# 运行程序
cargo run --target=riscv32imac-unknown-none-elf
```

## 使用调试探针

使用调试探针时，您可以直接将程序刷新到 Pico 2：

```sh
# cargo flash --chip RP2350
# cargo flash --chip RP2350 --release
cargo flash --release
```

如果您想刷新程序并实时查看其输出，请使用：

```sh
# cargo embed --chip RP2350
# cargo embed --chip RP2350 --release
cargo embed --release
```

[cargo-embed](https://probe.rs/docs/tools/cargo-embed/) 是 cargo-flash 的更高级版本。它可以刷新您的程序，并可以打开 RTT 终端和 GDB 服务器。
