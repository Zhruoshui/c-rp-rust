# 使用 `cargo-generate` 创建项目模板

"cargo-generate 是一个开发者工具，它可以把已有的 git 仓库作为模板，帮助你快速创建并启动一个新的 Rust 项目。"

更多信息可以阅读 [cargo-generate GitHub 项目](https://github.com/cargo-generate/cargo-generate)。

## 前置条件

开始之前，请确保你已经安装了以下工具：

- [Rust](https://www.rust-lang.org/tools/install)
- [cargo-generate](https://github.com/cargo-generate/cargo-generate)，用于生成项目模板。

由于 cargo-generate 需要 OpenSSL 开发包，请先安装它：

```sh
sudo apt install  libssl-dev
```

可以使用下面的命令安装 `cargo-generate`：

```sh
cargo install cargo-generate
```

## 第 1 步：生成项目

运行下面的命令，通过模板生成项目：

```sh
cargo generate --git https://github.com/ImplFerris/pico2-template.git --tag v0.3.2
```

命令执行后，会提示你回答几个问题：

- Project name：为你的项目命名。
- HAL choice：可以在 `embassy` 和 `rp-hal` 之间选择。

## 第 2 步：默认 LED 闪烁示例

默认情况下，生成出来的项目会包含一个简单的 LED 闪烁示例。代码结构大致如下：

`src/main.rs`：包含默认的闪灯逻辑。

`Cargo.toml`：包含所选 HAL 需要的依赖。

## 第 3 步：选择 HAL 并修改代码

项目生成后，你可以选择保留默认的 LED 闪烁代码，也可以删除它，并根据所选的 HAL 替换成自己的代码。

## 删除不需要的代码

你可以从 `src/main.rs` 中删除闪灯逻辑，并换成自己的代码。根据项目需要，调整 `Cargo.toml` 中的依赖和项目结构即可。
