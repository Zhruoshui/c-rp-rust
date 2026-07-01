# 快速入门 for Rust

我们同样用 Rust 运行一个闪灯程序。在 Rust 中，我们使用 Embassy 库。Embassy 是 Rust 嵌入式生态中的异步框架，它可以让我们用 `async/await` 写微控制器程序。后面会再详细介绍这个框架，现在先有个印象即可。

## Pico 2 和 Pico 2 W 的差异

还记得在[Pico 2 和 Pico 2 W 的差异](./different_between_pico2_pico2W.md)中提及的吗？

> [!NOTE]
> Pico 2 和 Pico 2 W 都基于 RP2350，但它们的板载 LED 接法不同：
> - Pico 2 的闪灯代码非常直接，只需要配置 `PIN_25` 为输出引脚；
> - Pico 2 W 则必须先初始化 CYW43 芯片，再通过 CYW43 的 GPIO 控制 LED。

示例项目已经用 `Cargo feature` 来做类似 C 语言条件编译的板型切换：

- 默认 feature 是 `pico2`。
- 使用 `--no-default-features --features pico2w` 切换到 Pico 2 W。

## 克隆示例项目

```sh
git clone https://github.com/Zhruoshui/pico2-quick
cd pico2-quick
```

项目里已经配置好了常用命令别名：

```text
cargo build-pico2
cargo run-pico2
cargo flash-pico2
cargo embed-pico2

cargo build-pico2w
cargo run-pico2w
cargo flash-pico2w
cargo embed-pico2w
```

## 代码核心

Pico 2 的 LED 是普通 GPIO，所以核心代码很短：

```rust
#[embassy_executor::main]
async fn main(_spawner: Spawner) {
    let p = embassy_rp::init(Default::default());
    let mut led = Output::new(p.PIN_25, Level::Low);

    loop {
        led.set_high();
        Timer::after_millis(100).await;

        led.set_low();
        Timer::after_millis(100).await;
    }
}
```
Pico 2 W 的完整代码会长很多，因为它需要初始化 CYW43 固件、NVRAM、PIO SPI 和后台 runner。


在示例项目中，最终点亮 Pico 2 W LED 的代码是：
```rust
control.gpio_set(0, true).await;
Timer::after_millis(100).await;

control.gpio_set(0, false).await;
Timer::after_millis(100).await;
```

这里的 `0` 指的是 CYW43 芯片上的 GPIO0。

## 只编译

先确认代码可以编译。

Pico 2：

```sh
cargo build-pico2
```

Pico 2 W：

```sh
cargo build-pico2w
```

等价的原始 Cargo 命令分别是：

```sh
cargo build --release
cargo build --release --no-default-features --features pico2w
```

## 使用 BOOTSEL 运行

`cargo run` 这条路径使用的是 `picotool`，也就是 BOOTSEL 模式。

步骤如下：

1. 断开 Pico 的 USB 连接。
2. 按住 `BOOTSEL` 按钮不放。
3. 保持按住按钮，用 USB 线把 Pico 接到电脑。
4. 电脑识别到 BOOTSEL 设备后，松开按钮。
5. 运行对应命令
<img style="display: block; margin: auto;" alt="bootsel" src="./images/bootsel.png"/>

Pico 2：

```sh
cargo run-pico2
```

Pico 2 W：

```sh
cargo run-pico2w
```

如果你看到类似下面的错误：

```text
No accessible RP-series devices in BOOTSEL mode were found.
```

通常说明板子当前没有进入 BOOTSEL 模式，或者系统的 udev 规则还没有对当前用户生效。先重新按住 `BOOTSEL` 插入 USB，再试一次。

另外你也可以直接把对应文件拖到 `RP2350` 盘里即可，不过能自动化是最好的：
![](./images/programming.png)


## 使用 Debug Probe 烧录

如果你连接了 Debug Probe，就不需要按 `BOOTSEL`。这时使用 `cargo flash` 或 `cargo embed`。

Pico 2：

```sh
cargo flash-pico2
cargo embed-pico2
```

Pico 2 W：

```sh
cargo flash-pico2w
cargo embed-pico2w
```

两者区别是：

- `cargo flash-*`：只烧录并运行程序。
- `cargo embed-*`：烧录后打开 RTT，可以看到 `defmt` 日志输出。

这个示例程序会输出类似下面的日志：

```text
[INFO] Pico 2 W CYW43 LED blink
[INFO] led on
[INFO] led off
```

Pico 2 W 在启动时会初始化 CYW43，所以底层初始化流程比 Pico 2 多；但示例项目已经关闭了不必要的 CYW43 调试日志，`cargo embed-pico2w` 里主要看到的就是应用自己的 `led on/off` 输出。

## 命令选择速查

| 场景 | Pico 2 | Pico 2 W |
| --- | --- | --- |
| 只编译 | `cargo build-pico2` | `cargo build-pico2w` |
| BOOTSEL / picotool | `cargo run-pico2` | `cargo run-pico2w` |
| Debug Probe 烧录 | `cargo flash-pico2` | `cargo flash-pico2w` |
| Debug Probe + RTT 日志 | `cargo embed-pico2` | `cargo embed-pico2w` |

运行成功后，板载 LED 会持续闪烁。 
