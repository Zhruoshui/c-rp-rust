# impl Rust & C for RP2350

基于 [ImplFerris/pico-pico](https://github.com/ImplFerris/pico-pico) 改编，在原作 Rust 嵌入式内容的基础上，新增 **C (Pico SDK)** 实现作为对照。采用「同一个概念，C 和 Rust 两种写法」的编排方式，在 Raspberry Pi Pico 2 (RP2350) 上同时学习嵌入式 C 与 Rust。

## 项目结构

```
每个外设章节 = 原理 & 电路 → C 实现 (Pico SDK) → Rust 实现 (Embassy / rp-hal)
```

涵盖 LED、PWM、舵机、超声波、ADC、I2C、SPI、OLED、LCD、RFID、SD 卡等外设实战。

## 阅读

在线版：https://pico.implrust.com/（原版）

本地运行：

```sh
mdbook serve --open
```

## 致谢

- 原作者：[ImplFerris](https://github.com/ImplFerris) 的 [pico-pico](https://github.com/ImplFerris/pico-pico)
- 原版在线阅读：https://pico.implrust.com/

## License

本书沿用原项目的许可证：

- 代码示例及 Cargo 项目：[MIT License](./LICENSE-MIT) 和 [Apache License v2.0](./LICENSE-APACHE)
- 文字内容：[CC-BY-SA v4.0](./LICENSE-CC-BY-SA)

