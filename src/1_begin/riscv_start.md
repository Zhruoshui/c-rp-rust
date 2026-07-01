Pico 2 / Pico 2W 也可以使用RISC-V核心编程，RISC-V架构支持通过动态切换Cortex-M33（Armv8-M）处理器与Hazard3（RV32IMAC+）处理器来实现。RISC-V核心支持通过SWD进行调试，并且可以使用与ARM核心相同的SDK进行编程。


# RISC-V 构建和运行

在 RISC-V 模式下，程序运行在 Pico 2 的 Hazard3 处理器上。C 与 Rust 的构建方式分别见下。

> [!IMPORTANT]
> 本书重点介绍 ARM。某些示例在 RISC-V 模式下工作前可能需要进行修改。为简便起见，建议在阅读本书时遵循 ARM 工作流程。
> 
> 建议严格遵循sdk中的说明来使用：https://github.com/raspberrypi/pico-sdk#risc-v-support-on-rp2350
> 
> 同时RISC-V的学习演示可以看：http://github.com/mytechnotalent/RP2350_Blink_Driver_RISCV

## for c

如果你已经通过官方 VS Code 插件安装了 RISC-V 工具链，可以直接复用 `~/.pico-sdk/toolchain/` 下的工具。先确认本机目录名：

```sh
ls "$HOME/.pico-sdk/toolchain"
```

下面以插件安装的 `RISCV_ZCB_RPI_2_2_0_3` 为例。RISC-V 模式**不需要改动源码，也不需要改 `CMakeLists.txt`**，同一个 `pico2-manual-blink` 工程，只在配置时把平台切到 `rp2350-riscv`：

```sh
export PICO_SDK_PATH="$HOME/.pico-sdk/sdk/2.2.0"
export PICO_TOOLCHAIN_PATH="$HOME/.pico-sdk/toolchain/RISCV_ZCB_RPI_2_2_0_3"
export PATH="$PICO_TOOLCHAIN_PATH/bin:$PATH"

# 构建（Hazard3 RISC-V），产物在 build-riscv/
cmake -S . -B build-riscv \
    -DPICO_BOARD=pico2 \
    -DPICO_PLATFORM=rp2350-riscv \
    -DPICO_TOOLCHAIN_PATH="$PICO_TOOLCHAIN_PATH" \
    -DPICO_COMPILER=pico_riscv_gcc_zcb_zcmp
cmake --build build-riscv -j
```

生成的 UF2 文件在 `build-riscv/pico2_manual_blink.uf2`。

> [!NOTE]
> RISC-V 需要单独的编译器，ARM 的 `arm-none-eabi-gcc` 不能用。官方插件安装的是 `riscv32-unknown-elf-*` 工具链，并且 `pico_riscv_gcc_zcb_zcmp` 会启用适合 RP2350 Hazard3 的 RISC-V 编译器配置。Hazard3 没有硬件 FPU，浮点走软件库（soft-float）。

如果你没有使用官方插件，也可以按 [手动配置开发环境（可选）](./manual_setup.md) 安装发行版或官方预编译 RISC-V 工具链，再把 `PICO_TOOLCHAIN_PATH` 和 `PICO_PLATFORM=rp2350-riscv` 指向你的安装路径。

烧录方式与 ARM 完全相同（BOOTSEL + 复制 UF2）：

```sh
cp build-riscv/pico2_manual_blink.uf2 /run/media/$USER/RP2350/
```

## for rust
切换到rust项目中

```sh
# 构建程序
cargo build --target=riscv32imac-unknown-none-elf
```

按照上面描述的相同 BOOTSEL 步骤进行操作。

```sh
# 烧录程序
cargo run --target=riscv32imac-unknown-none-elf
```

