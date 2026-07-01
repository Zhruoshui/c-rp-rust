# 快速入门 for C

本章从一个空目录开始，手写一个最小 Pico 2 / Pico 2 W C 工程。环境仍然使用官方 VS Code 插件安装到 `~/.pico-sdk` 的 SDK、CMake、Ninja、ARM 工具链、picotool 和 OpenOCD；我们只是不用插件向导生成代码，而是自己把项目结构写出来。

如果你只是想快速生成工程，可以直接使用 [VS Code 官方插件](./pico_in_vscode.md) 里的 `Raspberry Pi Pico: New Pico Project`。本章的目的，是让你看清楚一个 Pico C 工程最小需要哪些文件，以及每一步构建命令到底在做什么。

## 准备终端环境

先确认已经完成 [C 环境](./c_setup.md) 中的插件工具链配置。普通终端里需要先导入环境变量：

```sh
source ~/.config/pico-env.sh
```

如果你把那段环境变量写进了 `~/.bashrc` 或 `~/.zshrc`，打开新终端后可以直接检查：

```sh
echo "$PICO_SDK_PATH"
echo "$PICO_TOOLCHAIN_PATH"
"$PICO_CMAKE" --version
"$PICO_NINJA" --version
arm-none-eabi-gcc --version
picotool version
```

如果这些命令里有空输出或 `command not found`，先回到 [C 环境](./c_setup.md) 修正 `~/.pico-sdk` 版本变量。

## Pico 2 和 Pico 2 W 的差异

还记得在 [Pico 2 和 Pico 2 W 的差异](./different_between_pico2_pico2W.md) 中提到的吗？

> [!NOTE]
> Pico 2 和 Pico 2 W 都基于 RP2350，但板载 LED 接法不同：
> - Pico 2 的 LED 在 RP2350 的普通 GPIO 上，SDK 会定义 `PICO_DEFAULT_LED_PIN`。
> - Pico 2 W 的 LED 在 CYW43 无线芯片上，SDK 会定义 `CYW43_WL_GPIO_LED_PIN`，并且需要链接 `pico_cyw43_arch_none`。

因此，本章会写一份同时兼容 Pico 2 和 Pico 2 W 的代码，通过 `-DPICO_BOARD=pico2` 或 `-DPICO_BOARD=pico2_w` 切换开发板。

## 建立目录

建立一个目录来放置工程文件：

```sh
mkdir pico2-plugin-blink
cd pico2-plugin-blink
touch CMakeLists.txt main.c
```

把 SDK 提供的导入脚本复制到当前项目：

```sh
cp "$PICO_SDK_PATH/external/pico_sdk_import.cmake" .
```

这个文件是 Pico SDK 的 CMake 入口脚本。它会根据 `PICO_SDK_PATH` 找到真正的 SDK。

最终项目结构是：

```text
pico2-plugin-blink/
├── CMakeLists.txt
├── main.c
└── pico_sdk_import.cmake
```

## 写 CMakeLists.txt

把 `CMakeLists.txt` 写成下面这样：

```cmake
cmake_minimum_required(VERSION 3.13)

include(pico_sdk_import.cmake)

project(pico2_plugin_blink C CXX ASM)

set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)

pico_sdk_init()

add_executable(pico2_plugin_blink
    main.c
)

target_link_libraries(pico2_plugin_blink
    pico_stdlib
)

if (PICO_CYW43_SUPPORTED)
    target_link_libraries(pico2_plugin_blink
        pico_cyw43_arch_none
    )
endif()

pico_add_extra_outputs(pico2_plugin_blink)

set(PICOTOOL picotool CACHE FILEPATH "Path to picotool executable")

add_custom_target(pico2_plugin_blink_run
    COMMAND ${PICOTOOL} load -u -v -x -t elf $<TARGET_FILE:pico2_plugin_blink>
    DEPENDS pico2_plugin_blink
    USES_TERMINAL
)

set(OPENOCD openocd CACHE FILEPATH "Path to OpenOCD executable")
set(OPENOCD_SCRIPTS "" CACHE PATH "Path to OpenOCD scripts")
set(OPENOCD_SCRIPT_ARGS "")
if (OPENOCD_SCRIPTS)
    list(APPEND OPENOCD_SCRIPT_ARGS -s ${OPENOCD_SCRIPTS})
endif()

add_custom_target(pico2_plugin_blink_probe
    COMMAND ${OPENOCD}
            ${OPENOCD_SCRIPT_ARGS}
            -f interface/cmsis-dap.cfg
            -f target/rp2350.cfg
            -c "adapter speed 5000"
            -c "program $<TARGET_FILE:pico2_plugin_blink> verify reset exit"
    DEPENDS pico2_plugin_blink
    USES_TERMINAL
)
```

这里有几个关键点：

- `include(pico_sdk_import.cmake)` 必须在 `project()` 之前。
- `project(... C CXX ASM)` 要启用 C、C++ 和汇编，因为 Pico SDK 的启动代码会用到它们。
- `pico_sdk_init()` 必须在创建可执行 target 之前调用。
- `pico_add_extra_outputs()` 会在 ELF 之外生成 `.uf2`、`.bin`、`.hex` 等文件。
- `pico2_plugin_blink_run` 使用 `picotool` 通过 BOOTSEL / USB 烧录。
- `pico2_plugin_blink_probe` 使用 OpenOCD 通过 Debug Probe / SWD 烧录。

## 写 main.c

把 `main.c` 写成下面这样：

```c
#include <stdbool.h>

#include "pico/stdlib.h"

#if defined(CYW43_WL_GPIO_LED_PIN)
#include "pico/cyw43_arch.h"
#endif

#define LED_DELAY_MS 250

static int board_led_init(void) {
#if defined(PICO_DEFAULT_LED_PIN)
    gpio_init(PICO_DEFAULT_LED_PIN);
    gpio_set_dir(PICO_DEFAULT_LED_PIN, GPIO_OUT);
    return PICO_OK;
#elif defined(CYW43_WL_GPIO_LED_PIN)
    return cyw43_arch_init();
#else
    return -1;
#endif
}

static void board_led_put(bool on) {
#if defined(PICO_DEFAULT_LED_PIN)
    gpio_put(PICO_DEFAULT_LED_PIN, on);
#elif defined(CYW43_WL_GPIO_LED_PIN)
    cyw43_arch_gpio_put(CYW43_WL_GPIO_LED_PIN, on);
#endif
}

int main(void) {
    if (board_led_init() != PICO_OK) {
        return 1;
    }

    while (true) {
        board_led_put(true);
        sleep_ms(LED_DELAY_MS);

        board_led_put(false);
        sleep_ms(LED_DELAY_MS);
    }
}
```

这份代码可以同时构建 Pico 2 和 Pico 2 W：

- Pico 2 会走 `PICO_DEFAULT_LED_PIN` 分支，直接控制普通 GPIO。
- Pico 2 W 会走 `CYW43_WL_GPIO_LED_PIN` 分支，先初始化 CYW43，再控制无线芯片上的 LED。

## 编译 Pico 2

配置构建目录：

```sh
"$PICO_CMAKE" -S . -B build-pico2 -G Ninja \
    -DCMAKE_MAKE_PROGRAM="$PICO_NINJA" \
    -DPICO_BOARD=pico2 \
    -DPICO_SDK_PATH="$PICO_SDK_PATH" \
    -DPICO_TOOLCHAIN_PATH="$PICO_TOOLCHAIN_PATH" \
    -Dpicotool_DIR="$picotool_DIR" \
    -Dpioasm_DIR="$pioasm_DIR" \
    -DPICOTOOL="$picotool_DIR/picotool" \
    -DOPENOCD="$PICO_OPENOCD" \
    -DOPENOCD_SCRIPTS="$PICO_OPENOCD_SCRIPTS"
```

构建：

```sh
"$PICO_CMAKE" --build build-pico2 -j8
```

生成的 UF2 文件在：

```text
build-pico2/pico2_plugin_blink.uf2
```

## 编译 Pico 2 W

配置构建目录：

```sh
"$PICO_CMAKE" -S . -B build-pico2w -G Ninja \
    -DCMAKE_MAKE_PROGRAM="$PICO_NINJA" \
    -DPICO_BOARD=pico2_w \
    -DPICO_SDK_PATH="$PICO_SDK_PATH" \
    -DPICO_TOOLCHAIN_PATH="$PICO_TOOLCHAIN_PATH" \
    -Dpicotool_DIR="$picotool_DIR" \
    -Dpioasm_DIR="$pioasm_DIR" \
    -DPICOTOOL="$picotool_DIR/picotool" \
    -DOPENOCD="$PICO_OPENOCD" \
    -DOPENOCD_SCRIPTS="$PICO_OPENOCD_SCRIPTS"
```

构建：

```sh
"$PICO_CMAKE" --build build-pico2w -j8
```

生成的 UF2 文件在：

```text
build-pico2w/pico2_plugin_blink.uf2
```

Pico 2 W 的构建会自动链接 `pico_cyw43_arch_none`，否则 `cyw43_arch_init()` 和 `cyw43_arch_gpio_put()` 会链接失败。

## 使用 BOOTSEL 运行

Pico 2 和 Pico 2 W 的 UF2 烧录方式相同：

1. 断开 Pico 的 USB 连接。
2. 按住 `BOOTSEL` 按钮不放。
3. 保持按住按钮，用 USB 线把 Pico 接到电脑。
4. 电脑识别到 BOOTSEL 设备后，松开按钮。
5. 复制对应的 UF2 文件，或者使用 `picotool` 烧录。

<img style="display: block; margin: auto;" alt="bootsel" src="./images/bootsel.png"/>

拖拽或复制 UF2。Pico 2：

```sh
cp build-pico2/pico2_plugin_blink.uf2 /run/media/$USER/RP2350/
```

Pico 2 W：

```sh
cp build-pico2w/pico2_plugin_blink.uf2 /run/media/$USER/RP2350/
```

如果你的系统挂载路径不是 `/run/media/$USER/RP2350/`，可以先用文件管理器或 `lsblk` 确认实际路径。复制完成后，U 盘会自动断开，板子重启，板载 LED 开始闪烁。

也可以直接用 `picotool` 烧录 ELF。Pico 2：

```sh
"$PICO_CMAKE" --build build-pico2 --target pico2_plugin_blink_run
```

Pico 2 W：

```sh
"$PICO_CMAKE" --build build-pico2w --target pico2_plugin_blink_run
```

这种方式仍然需要 Pico 处于 BOOTSEL 模式。如果看到：

```text
No accessible RP-series devices in BOOTSEL mode were found.
```

通常说明板子没有进入 BOOTSEL 模式，或者 Linux udev 规则还没有生效。

另外你也可以直接把对应文件拖到 `RP2350` 盘里：

![](./images/programming.png)

## 使用 Debug Probe 烧录

如果你连接了 Raspberry Pi Debug Probe，就不需要按 `BOOTSEL`。目标板仍然需要通过 USB 或外部电源供电，并且 Debug Probe 的 `SWDIO`、`SWCLK`、`GND` 要和 Pico 正确连接。

Pico 2：

```sh
"$PICO_CMAKE" --build build-pico2 --target pico2_plugin_blink_probe
```

Pico 2 W：

```sh
"$PICO_CMAKE" --build build-pico2w --target pico2_plugin_blink_probe
```

成功时会看到类似：

```text
** Programming Finished **
** Verify Started **
** Verified OK **
** Resetting Target **
```

如果命令提示找不到 CMSIS-DAP 设备，通常需要检查 Debug Probe 的 USB 连接、SWD 接线，以及当前用户是否有访问调试器的权限。

## 用 VS Code 插件接管这个项目

上面的工程是手写 CMake 项目，不依赖插件生成 `.vscode/` 配置。你仍然可以让插件接管它：

```text
Ctrl+Shift+P
Raspberry Pi Pico: Import Pico Project
```

导入后插件会补充 `.vscode/settings.json`、`tasks.json`、`launch.json` 等文件，并在 `CMakeLists.txt` 顶部加入插件管理块。之后可以用插件按钮：

- `Compile Pico Project`
- `Run Pico Project (USB)`
- `Flash Pico Project (SWD)`

如果你只是想理解 CMake 和命令行，本章前面的终端流程已经足够；如果你想日常在 VS Code 里点按钮构建和调试，再执行导入。

## 常见问题

### PICO_SDK_PATH 为空

如果复制 `pico_sdk_import.cmake` 或配置 CMake 时失败，先检查：

```sh
echo "$PICO_SDK_PATH"
test -f "$PICO_SDK_PATH/external/pico_sdk_import.cmake"
```

普通终端里通常是忘了执行：

```sh
source ~/.config/pico-env.sh
```

### arm-none-eabi-gcc not found

检查插件工具链是否在 `PATH` 中：

```sh
echo "$PICO_TOOLCHAIN_PATH"
which arm-none-eabi-gcc
```

如果 `which` 找不到，重新导入 [C 环境](./c_setup.md) 中的 `~/.pico-sdk` 环境变量。

### picotool 找不到或无权限

先确认 CMake 配置时传入了：

```sh
-Dpicotool_DIR="$picotool_DIR"
-DPICOTOOL="$picotool_DIR/picotool"
```

如果命令能找到 `picotool`，但访问 BOOTSEL 设备失败，通常是 Pico 没进 BOOTSEL 模式，或者 Linux udev 规则没有生效。

### Pico 2 W 链接失败

如果 Pico 2 W 构建时出现 `cyw43_arch_init` 或 `cyw43_arch_gpio_put` 未定义，通常说明没有链接 CYW43 库。确认 `CMakeLists.txt` 里有：

```cmake
if (PICO_CYW43_SUPPORTED)
    target_link_libraries(pico2_plugin_blink
        pico_cyw43_arch_none
    )
endif()
```

并且配置时使用的是：

```sh
-DPICO_BOARD=pico2_w
```
