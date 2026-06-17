# Pico 在 VS Code 中
{{#title Create an Embedded Rust Project for Raspberry Pi Pico 2 in VS Code}}

# 在 VS Code 中为 Raspberry Pi Pico 创建项目（使用相关扩展）

我们已经通过手动方式以及使用模板为 Pico 创建了 C 项目和 Rust 项目。现在我们打算尝试利用 VS Code 中的 Raspberry Pi Pico 扩展功能来辅助完成项目的创建。

## 使用 Pico 扩展

在 Visual Studio Code 中，搜索名为"Raspberry Pi Pico"的扩展，并确保安装的是官方版本；它应该有来自树莓派官方网站的验证发布者徽章。安装该扩展。

<div class="image-with-caption" style="text-align:center; display:inline-block;">
    <img src="./images/Raspberry Pi Pico Vscode extension.png" alt="VSCode Extension for Raspberry Pi Pico" style="max-width:100%; height:auto; display:block; margin:0 auto;"/>
    <div class="caption" style="font-size:0.9em; color:#555; margin-top:6px;">VSCode Extension for Raspberry Pi Pico</div>
</div>

不过，仅仅安装扩展可能还不够，这取决于你的计算机上已经安装了什么。在 Linux 上，你可能需要安装一些基本的依赖项：

```sh
sudo apt install build-essential libudev-dev
```

## 创建项目

让我们用 VS Code 中的 Pico 扩展创建 Rust 项目。打开左边的活动栏，点击 Pico 图标。然后选择"新建 Rust 项目"。

<div class="image-with-caption" style="text-align:center; display:inline-block;">
    <img src="./images/Create Project Raspberry Pi Pico Vscode extension.png" alt="Create Project Raspberry Pi Pico Vscode extension" style="max-width:100%; height:auto; display:block; margin:0 auto;"/>
    <div class="caption" style="font-size:0.9em; color:#555; margin-top:6px;">创建项目</div>
</div>

由于这是第一次设置，扩展将下载并安装必要的工具，包括 Pico SDK、picotool、OpenOCD 以及用于调试的 ARM 和 RISC-V 工具链。

## 项目结构

如果项目创建成功，你应该会看到像这样的文件夹和文件：

<div class="image-with-caption" style="text-align:center; display:inline-block;">
    <img src="./images/Raspberry Pi Pico Rust Project Created With VS Code Extension.png" alt="Raspberry Pi Pico Rust Project Created With VS Code Extension" style="max-width:100%; height:auto; display:block; margin:0 auto;"/>
    <div class="caption" style="font-size:0.9em; color:#555; margin-top:6px;">项目文件夹</div>
</div>

## 运行程序

现在你只需点击"运行项目（USB）"即可将程序刷入你的 Pico 并运行它。连接 Pico 到电脑时，别忘了按 BOOTSEL 按钮。否则，此选项将处于禁用状态。

<div class="image-with-caption" style="text-align:center; display:inline-block;">
    <img src="./images/Running Rust Project with Vscode for Raspberry Pi Pico 2 (RP2350).png" alt="Running Rust Project with Vscode for Raspberry Pi Pico 2 (RP2350)" style="max-width:100%; height:auto; display:block; margin:0 auto;"/>
    <div class="caption" style="font-size:0.9em; color:#555; margin-top:6px;">将 Rust 固件刷入树莓派 Pico</div>
</div>

刷入完成后，程序将立即在你的 Pico 上运行。你应该会看到板载 LED 闪烁。
