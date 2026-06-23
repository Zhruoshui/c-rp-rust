# 基于 pico-examples 的 C 项目模板

`pico-examples` 是 Raspberry Pi 官方维护的 Pico SDK 示例仓库。可以把它当成 Pico C 工程的参考模板，学习官方推荐的 CMake 组织方式，然后把其中一个示例裁剪成自己的工程。

先克隆官方示例：

```sh
git clone https://github.com/raspberrypi/pico-examples.git
cd pico-examples
```

## 先编译官方 blink闪灯程序

如果你使用的是 Pico 2：

```sh
cmake -S . -B build-pico2 \
    -DPICO_BOARD=pico2

cmake --build build-pico2 --target blink -j8
```

如果你使用的是 Pico 2 W：

```sh
cmake -S . -B build-pico2w \
    -DPICO_BOARD=pico2_w

cmake --build build-pico2w --target blink -j8
```

构建成功后，`blink` 的输出文件在对应构建目录下：

```text
build-pico2/blink/blink.elf
build-pico2/blink/blink.uf2
build-pico2/blink/blink.bin
```

`-DPICO_BOARD=pico2` 或 `-DPICO_BOARD=pico2_w` 会让 SDK 选择对应开发板配置。Pico SDK 默认面向 RP2040；对于 Pico 2 / Pico 2 W，推荐显式传入 `PICO_BOARD`，这样 SDK 会自动选择 RP2350 平台和板级引脚定义。

## pico-examples 的顶层结构

`pico-examples` 的根目录大致可以分成三类文件：

```text
pico-examples/
├── CMakeLists.txt
├── pico_sdk_import.cmake
├── pico_extras_import_optional.cmake
├── example_auto_set_url.cmake
├── blink/
├── hello_world/
├── gpio/
├── pwm/
├── spi/
└── ...
```

其中：

- `CMakeLists.txt` 是整个示例仓库的唯一顶层入口。
- `pico_sdk_import.cmake` 用来定位并导入 Pico SDK，必须在 `project()` 之前 `include`。
- `pico_extras_import_optional.cmake` 用来可选导入 `pico-extras`，普通 LED、GPIO、PWM、SPI 示例不一定需要它。
- `example_auto_set_url.cmake` 是官方示例仓库自己的辅助脚本，用来给生成的程序写入示例 URL。自己的项目通常可以不带它。
- `blink/`、`gpio/`、`pwm/`、`spi/` 这些目录只是示例分类，是否参与构建由顶层 `CMakeLists.txt` 决定。

## 顶层 CMakeLists.txt 的构建次序

阅读 `pico-examples/CMakeLists.txt` 时，最重要的是顺序。它不是随便写的，Pico SDK 的 CMake 工程需要按下面的顺序组织。

第一步，声明 CMake 版本，然后在 `project()` 之前导入 SDK：

```cmake
cmake_minimum_required(VERSION 3.12)

include(pico_sdk_import.cmake)
include(pico_extras_import_optional.cmake)

project(pico_examples C CXX ASM)
```

这里的关键点是：`include(pico_sdk_import.cmake)` 必须出现在 `project()` 之前。这个脚本会根据 `PICO_SDK_PATH` 找到 SDK，并导入 SDK 提供的初始化脚本。

第二步，设置语言标准、检查 SDK 版本，并初始化 SDK：

```cmake
set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)

if (PICO_SDK_VERSION_STRING VERSION_LESS "2.2.0")
    message(FATAL_ERROR "Raspberry Pi Pico SDK version 2.2.0 (or later) required.")
endif()

pico_sdk_init()
```

`pico_sdk_init()` 必须在创建应用 target 之前调用。调用后，`pico_stdlib`、`hardware_pwm`、`hardware_spi`、`pico_add_extra_outputs()` 这些 SDK target 和函数才准备好，子目录里的示例才能正常链接它们。

第三步，顶层开始把示例目录加入构建：

```cmake
add_subdirectory_exclude_platforms(blink)
add_subdirectory_exclude_platforms(blink_simple)
add_subdirectory_exclude_platforms(hello_world)
```

`add_subdirectory_exclude_platforms()` 是官方示例自己定义的小函数。它会先检查当前 `PICO_PLATFORM` 是否被排除，如果不排除，再调用真正的 `add_subdirectory()`。

简化后可以理解为：

```cmake
function(add_subdirectory_exclude_platforms NAME)
    # 如果当前平台不支持这个示例，就跳过。
    add_subdirectory(${NAME})
endfunction()
```

第四步，加入更多外设示例目录。当前官方示例的顶层顺序是：

```text
blink
blink_simple
hello_world
adc
binary_info
bootloaders
clocks
cmake
dcp
divider
dma
encrypted
flash
gpio
hstx
i2c
interp
multicore
otp
picoboard
pico_w
pio
pwm
reset
rtc
spi
status_led
system
timer
uart
universal
usb
watchdog
sha
freertos
```

这说明 CMake 的配置过程是从根目录进入每个示例目录，再由子目录继续进入更小的叶子示例。例如：

```text
pico-examples/CMakeLists.txt
└── add_subdirectory(pwm)
    └── pwm/CMakeLists.txt
        ├── add_subdirectory(hello_pwm)
        ├── add_subdirectory(led_fade)
        └── add_subdirectory(measure_duty_cycle)
```

有些分类目录还会先检查 SDK target 是否存在。比如 `pwm/CMakeLists.txt` 会检查 `hardware_pwm`，`spi/CMakeLists.txt` 会检查 `hardware_spi`。如果当前平台不支持，就在 CMake 配置阶段跳过对应示例。

> [!NOTE]
> 官方顶层文件里的 `add_compile_options(-Wall ...)` 放在 `blink`、`blink_simple`、`hello_world` 之后。CMake 的顺序会影响后续目录继承到的编译选项。你自己写项目时，如果希望所有 target 都使用某些编译选项，最好在创建 target 之前设置，或者更明确地使用 `target_compile_options()`。

## blink 示例的叶子 CMakeLists.txt

`blink/CMakeLists.txt` 是一个很适合作为模板的最小示例：

```cmake
add_executable(blink
    blink.c
)

target_link_libraries(blink pico_stdlib)

if (PICO_CYW43_SUPPORTED)
    target_link_libraries(blink pico_cyw43_arch_none)
endif()

pico_add_extra_outputs(blink)

example_auto_set_url(blink)
```

这个文件只负责定义 `blink` 这个 target。它假设 SDK 已经在顶层被初始化，所以它可以直接使用：

- `add_executable()`：声明要生成的可执行程序。
- `target_link_libraries(... pico_stdlib)`：链接 Pico SDK 的基础库，包含 GPIO、时间、stdio、断言等常用功能。
- `PICO_CYW43_SUPPORTED`：如果目标板支持 CYW43，比如 Pico 2 W，就额外链接 `pico_cyw43_arch_none`。
- `pico_add_extra_outputs()`：在 ELF 之外生成 `.uf2`、`.bin`、`.hex`、`.map` 等文件。
- `example_auto_set_url()`：官方示例专用，自己的项目可以删掉。

`blink` 还额外定义了两个自定义目标：

```cmake
add_custom_target(blink_run
    COMMAND picotool load -u -v -x -t elf $<TARGET_FILE:blink>
    DEPENDS blink
    USES_TERMINAL
)

add_custom_target(blink_probe
    COMMAND probe-rs run --chip RP235x --protocol swd $<TARGET_FILE:blink>
    DEPENDS blink
    USES_TERMINAL
)
```

这里的 `DEPENDS blink` 很重要。它告诉 CMake：运行 `blink_run` 或 `blink_probe` 之前，必须先把 `blink` 构建出来。

构建并烧录可以写成：

```sh
cmake --build build-pico2 --target blink_run
```

或者使用 Debug Probe：

```sh
cmake --build build-pico2 --target blink_probe
```

## 复用成自己的工程

如果你只是想建立一个自己的 Pico C 项目，不需要复制整个 `pico-examples`。建议只保留最小结构：

```text
my-pico-app/
├── CMakeLists.txt
├── pico_sdk_import.cmake
└── src/
    └── main.c
```

可以从 SDK 复制导入脚本：

```sh
mkdir -p my-pico-app/src
cd my-pico-app

cp "$PICO_SDK_PATH/external/pico_sdk_import.cmake" .
touch CMakeLists.txt src/main.c
```

然后把 `CMakeLists.txt` 写成下面这样：

```cmake
cmake_minimum_required(VERSION 3.13)

include(pico_sdk_import.cmake)

project(my_pico_app C CXX ASM)

set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)

pico_sdk_init()

add_executable(my_pico_app
    src/main.c
)

target_link_libraries(my_pico_app
    pico_stdlib
)

if (PICO_CYW43_SUPPORTED)
    target_link_libraries(my_pico_app
        pico_cyw43_arch_none
    )
endif()

pico_add_extra_outputs(my_pico_app)

add_custom_target(my_pico_app_run
    COMMAND picotool load -u -v -x -t elf $<TARGET_FILE:my_pico_app>
    DEPENDS my_pico_app
    USES_TERMINAL
)

add_custom_target(my_pico_app_probe
    COMMAND probe-rs run --chip RP235x --protocol swd $<TARGET_FILE:my_pico_app>
    DEPENDS my_pico_app
    USES_TERMINAL
)
```

这个文件保留了官方模板里最关键的构建顺序：

1. `include(pico_sdk_import.cmake)` 在 `project()` 之前。
2. `project(... C CXX ASM)` 启用 C、C++ 和汇编。
3. `pico_sdk_init()` 在所有 `add_executable()` 之前。
4. `add_executable()` 创建自己的程序。
5. `target_link_libraries()` 链接 SDK 库。
6. `pico_add_extra_outputs()` 生成 UF2 等烧录文件。
7. `add_custom_target()` 把烧录命令接到 CMake 构建目标上。

## 构建自己的工程

Pico 2：

```sh
cmake -S . -B build-pico2 \
    -DPICO_BOARD=pico2

cmake --build build-pico2 -j8
```

Pico 2 W：

```sh
cmake -S . -B build-pico2w \
    -DPICO_BOARD=pico2_w

cmake --build build-pico2w -j8
```

生成文件位于：

```text
build-pico2/my_pico_app.elf
build-pico2/my_pico_app.uf2
build-pico2/my_pico_app.bin
```

使用 BOOTSEL + `picotool` 烧录：

```sh
cmake --build build-pico2 --target my_pico_app_run
```

使用 Debug Probe 烧录：

```sh
cmake --build build-pico2 --target my_pico_app_probe
```

## 什么时候继续使用 pico-examples

当你需要某个外设的官方写法时，继续回到 `pico-examples` 查对应目录即可。例如：

- LED / 板载灯：`blink/`
- USB 串口输出：`hello_world/usb/`
- GPIO 中断：`gpio/hello_gpio_irq/`
- PWM：`pwm/hello_pwm/`
- SPI：`spi/`
- I2C：`i2c/`
- UART：`uart/`
- Watchdog：`watchdog/`

查示例时建议先看叶子目录的 `CMakeLists.txt`，再看 `.c` 文件。因为 CMake 文件会告诉你这个示例依赖哪些 SDK 库，例如 `hardware_pwm`、`hardware_spi`、`hardware_i2c`。把代码复制进自己的工程后，也要把对应库加入 `target_link_libraries()`。

例如你从 `pwm/hello_pwm` 复制代码，就需要链接：

```cmake
target_link_libraries(my_pico_app
    pico_stdlib
    hardware_pwm
)
```

从 `spi/max7219_32x8_spi` 复制代码，则需要链接：

```cmake
target_link_libraries(my_pico_app
    pico_stdlib
    hardware_spi
)
```

这样使用 `pico-examples`，它就不只是"能跑的例子"，而是你后续写 Pico C 工程时最可靠的一组官方模板。
