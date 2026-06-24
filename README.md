# impl Rust & C for RP2350

基于 [ImplFerris/pico-pico](https://github.com/ImplFerris/pico-pico) 改编，在原作 Rust 嵌入式内容的基础上
1. 新增 **C (Pico SDK)** 实现作为对照。采用「同一个概念，C 和 Rust 两种写法」的编排方式，在 Raspberry Pi Pico 2 (RP2350) 上同时学习嵌入式 C 与 Rust。
2. 修改了章节排布
3. 提前列出了每章所需的硬件，方便学习过程中添补

## 构建

本书使用 [mdBook](https://rust-lang.github.io/mdBook/) 构建。

- 配置文件：[`book.toml`](./book.toml)
- 章节源码：`src/` 目录（Markdown 文件）
- 主题/样式：`theme/` 目录

### 从零开始

```sh
# 1. 拉取代码
git clone https://github.com/Zhruoshui/c-rp-rust.git
cd c-rp-rust

# 2. 安装 mdbook 及其预处理器，没有安装rust可以查阅：https://rustup.rs/
cargo install mdbook
cargo install mdbook-tabs

# 3. 启动本地服务器
mdbook serve --open

# 也可以只构建静态页面
# mdbook build
# 生成的 HTML 位于 book/ 目录
```

## 项目结构

```
每个外设章节 = 原理 & 电路 → C 实现 (Pico SDK) → Rust 实现 (Embassy / rp-hal)
```

涵盖 LED、PWM、舵机、超声波、ADC、I2C、SPI、OLED、LCD、RFID、SD 卡等外设实战。

## 阅读

在线版：https://rrpbook.aruoshui.fun/

## 致谢

- 原作者：[ImplFerris](https://github.com/ImplFerris) 的 [pico-pico](https://github.com/ImplFerris/pico-pico)
- 原版在线阅读：https://pico.implrust.com/

## License

本书沿用原项目的许可证：

- 代码示例及 Cargo 项目：[MIT License](./LICENSE-MIT) 和 [Apache License v2.0](./LICENSE-APACHE)
- 文字内容：[CC-BY-SA v4.0](./LICENSE-CC-BY-SA)

