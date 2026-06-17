# 快速开始
在复杂的概念及原理之前，我们先实践一下，了解在pico 2 / pico 2W 上开发是怎么样的过程，首先我会介绍以传统的C/C++开发，然后再介绍Rust部分。

## 项目模板
前面的章节已经安装好了pico-sdk，你可以`echo $PICO_SDK_PATH`来看环境变量是否生效了。

要建立一个工程文件，最简单的方法是clone一下官方的例子，我们这里从最简单的闪灯程序开始：

### 板界hello-world
```
git clone https://github.com/raspberrypi/pico-examples.git
cd pico-examples/blink
```

可看到源码，源码还是很容易看懂的，主要是条件编译部分：
```c
#include "pico/stdlib.h"

// Pico W devices use a GPIO on the WIFI chip for the LED,
// so when building for Pico W, CYW43_WL_GPIO_LED_PIN will be defined
#ifdef CYW43_WL_GPIO_LED_PIN
#include "pico/cyw43_arch.h"
#endif

#ifndef LED_DELAY_MS
#define LED_DELAY_MS 250
#endif

// Perform initialisation
int pico_led_init(void) {
#if defined(PICO_DEFAULT_LED_PIN)
    // A device like Pico that uses a GPIO for the LED will define PICO_DEFAULT_LED_PIN
    // so we can use normal GPIO functionality to turn the led on and off
    gpio_init(PICO_DEFAULT_LED_PIN);
    gpio_set_dir(PICO_DEFAULT_LED_PIN, GPIO_OUT);
    return PICO_OK;
#elif defined(CYW43_WL_GPIO_LED_PIN)
    // For Pico W devices we need to initialise the driver etc
    return cyw43_arch_init();
#endif
}

// Turn the led on or off
void pico_set_led(bool led_on) {
#if defined(PICO_DEFAULT_LED_PIN)
    // Just set the GPIO on or off
    gpio_put(PICO_DEFAULT_LED_PIN, led_on);
#elif defined(CYW43_WL_GPIO_LED_PIN)
    // Ask the wifi "driver" to set the GPIO on or off
    cyw43_arch_gpio_put(CYW43_WL_GPIO_LED_PIN, led_on);
#endif
}

int main() {
    int rc = pico_led_init();
    hard_assert(rc == PICO_OK);
    while (true) {
        pico_set_led(true);
        sleep_ms(LED_DELAY_MS);
        pico_set_led(false);
        sleep_ms(LED_DELAY_MS);
    }
}
```
> [!NOTE]
> 官方 blink.c 已经兼容 Pico 2 和 Pico 2 W：普通 Pico 2 用 `PICO_DEFAULT_LED_PIN` 控制 GPIO LED，Pico 2 W 用 `cyw43_arch_gpio_put()`
控制 Wi-Fi 芯片上的板载 LED。这个差异在[引脚说明](./pico2-pinout.md)中有提及。

### 运行代码

#### 编译 Pico 2：
推荐详细阅读一下构建文件cmakelists
```shell
cmake -S . -B build-pico2 \
    -DPICO_BOARD=pico2

cmake --build build-pico2 -j --target blink
```
生成文件：`build-pico2/blink/blink.uf2`

**编译过程说明**：

- 第一条 `cmake` 命令是配置工程。
  - `-S .` 表示源码目录就是当前目录，`-B build-pico2` 表示把构建文件放到当前目录下的 `build-pico2`。

  - 配置时，CMake 会从当前目录的 `CMakeLists.txt` 开始读取：先通过 `include(pico_sdk_import.cmake)` 找到 Pico SDK，再通过 `pico_sdk_init()` 按 `PICO_BOARD=pico2` 准备 Pico 2 / RP2350 的板级配置、启动代码和链接脚本。

  - 根 `CMakeLists.txt` 里的 `add_subdirectory_exclude_platforms(blink)` 也是配置阶段执行的。它会进入 `blink/CMakeLists.txt`；在那里，`add_executable(blink blink.c)` 创建 `blink` 目标，`target_link_libraries(blink pico_stdlib)` 记录它需要链接 GPIO、sleep 等 SDK 功能，`pico_add_extra_outputs(blink)` 记录在 `blink.elf` 之后还要生成 `blink.uf2`。

  - `PICO_SDK_PATH` 环境变量生效时，这里不需要再传 `-DPICO_SDK_PATH`。如果没有设置环境变量，可以在配置命令中额外加上 `-DPICO_SDK_PATH=../pico-sdk`，具体路径按你的 SDK 位置调整。
  
- 第二条 `cmake --build ... --target blink` 开始编译。
  `--target blink` 表示只沿着 `blink` 这条依赖链构建：`blink.c` -> `blink.elf` -> `blink.uf2`。

#### 编译 Pico 2 W：
```shell
cmake -S . -B build-pico2w \
    -DPICO_BOARD=pico2_w

cmake --build build-pico2w -j --target blink
```
生成文件：`build-pico2w/blink/blink.uf2`

Pico 2 W 的流程和 Pico 2 相同，关键区别是 `-DPICO_BOARD=pico2_w`。这个板级配置会让 SDK 知道板载 LED 接在 CYW43 Wi-Fi 芯片上，而不是普通 GPIO 上。

因此 `blink/CMakeLists.txt` 中的判断会生效：

```cmake
if (PICO_CYW43_SUPPORTED)
    target_link_libraries(blink pico_cyw43_arch_none)
endif()
```

这样 `blink.c` 里针对 Pico W / Pico 2 W 的 `cyw43_arch_gpio_put()` 才能正常链接。最终依然只需要构建 `blink` 目标，生成的烧录文件是 `build-pico2w/blink/blink.uf2`。

#### 烧录到板子

Pico 2 和 Pico 2 W 的 UF2 烧录方式相同：

1. 先断开 Pico 的 USB 连接。
2. 按住板子上的 `BOOTSEL` 按钮不放。
   
![BOOTSEL按钮，图上的例子是2的，2W也在这个位置](./images/bootsel.png)

1. 保持按住 `BOOTSEL`，用 USB 线把 Pico 接到电脑。
2. 电脑识别出名为 `RP2350` 的 U 盘后，松开 `BOOTSEL`。
3. 把前面生成的 `.uf2` 文件复制到这个 U 盘中。


如果使用图形界面，直接把对应文件拖到 `RP2350` 盘里即可：
![](./images/programming.png)

```text
build-pico2/blink/blink.uf2
build-pico2w/blink/blink.uf2
```

如果使用命令行，Linux 下挂载路径通常类似 `/run/media/$USER/RPI-RP2` 或 `/media/$USER/RPI-RP2`，可以先用文件管理器或 `lsblk` 确认。确认后复制 UF2：

```shell
# Pico 2
cp build-pico2/blink/blink.uf2 /run/media/$USER/RP2350/

# Pico 2 W
cp build-pico2w/blink/blink.uf2 /run/media/$USER/RP2350/
```

复制完成后，`RP2350` 盘会自动断开，Pico 会重启并运行刚烧录进去的程序。如果烧录的是 blink，板载 LED 就会开始闪烁。

#### 自动化烧录
1. 不使用Debug Probe
   与前面的烧录一样，需要进入`BOOTSEL`模式，添加`blink/CMakeLists.txt`：
   ```cmakelists
   add_custom_target(blink_run
      COMMAND picotool load -u -v -x -t elf $<TARGET_FILE:blink>
      DEPENDS blink
      USES_TERMINAL
    )
   ```
   第二步替换执行：`cmake --build build-pico2 --target blink_run`即可

2. 使用Debug Probe
   连接好Debug Probe，无需再进入`BOOTSEL`模式了，可以使用openocd或者probe-rs来进行烧录，依然是添加`blink/CMakeLists.txt`
   ```cmakelists
   add_custom_target(blink_probe
      COMMAND probe-rs run --chip RP235x --protocol swd $<TARGET_FILE:blink>
      DEPENDS blink
      USES_TERMINAL
   )
   ```

   第二步替换执行：`cmake --build build-pico2 --target blink_probe`即可