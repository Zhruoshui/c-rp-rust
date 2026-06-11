# 写在前面

<div style="background: linear-gradient(135deg, rgba(230, 126, 34, 0.12) 0%, rgba(230, 126, 34, 0.05) 100%); border: 1px solid rgba(230, 126, 34, 0.3); border-left: 6px solid #e67e22; padding: 25px; margin: 25px 0; border-radius: 15px; box-shadow: 0 5px 20px rgba(0,0,0,0.08); backdrop-filter: blur(8px);">
    <div style="display: flex; align-items: center; margin-bottom: 18px;">
        <span style="font-size: 2.2em; margin-right: 15px;">📖</span>
        <h3 style="margin: 0; color: #e67e22; font-size: 1.6em; letter-spacing: 1px; font-weight: 700;">关于本书 / About This Book</h3>
    </div>
    <p style="font-size: 1.15em; line-height: 1.7; margin-bottom: 15px;">
        <strong>本书基于 <a href="https://github.com/ImplFerris/pico-pico" style="color: #e67e22;">ImplFerris/pico-pico</a> 原项目改编，在原作 Rust 嵌入式内容的基础上，新增了 C (Pico SDK) 实现作为对照。采用「同一个概念，C 和 Rust 两种写法」的编排方式，帮助读者在嵌入式学习中同时掌握两门语言。</strong>
    </p>
    <p style="opacity: 0.9; margin-bottom: 20px; font-size: 1em;">如果你在阅读过程中发现任何问题，欢迎通过以下方式反馈：</p>
    <div style="display: flex; gap: 15px; flex-wrap: wrap;">
        <a href="https://github.com/Zhruoshui/pico-pico-zh" target="_blank" style="text-decoration: none;">
            <img src="https://img.shields.io/badge/GitHub-Report_Issue-181717?style=for-the-badge&logo=github&logoColor=white" alt="GitHub Issues">
        </a>
        <a href="mailto:Aruoshui_Zh@outlook.com" style="text-decoration: none;">
            <img src="https://img.shields.io/badge/Email-Contact_Me-D14836?style=for-the-badge&logo=microsoftoutlook&logoColor=white" alt="Email">
        </a>
    </div>
</div>

## 为什么 C + Rust 双轨？

嵌入式开发的两个世界：

- **C + Pico SDK** — 离硬件最近，寄存器级别的控制，让你理解 MCU 真正在做什么
- **Rust + Embassy** — 现代异步框架，类型安全，编译期就能消灭一大类 bug

同一块 Pico 2，同一个外设，两种写法并排对照，比单独学任何一门都更高效。

## 阅读建议

- 如果你有 C 基础但没接触过 Rust → 先看 C 实现理解硬件，再对比 Rust 的抽象写法
- 如果你熟悉 Rust 但不熟嵌入式 → 先看 Rust 实现快速上手，再翻 C 实现理解底层
- 如果你两门都刚入门 → 按章节顺序走，每个外设的原理只看一遍，代码两边都读

<div style="text-align: center; margin: 40px 0; padding: 30px; border-top: 1px solid rgba(0,0,0,0.1); border-bottom: 1px solid rgba(0,0,0,0.1);">
    <p style="font-size: 1.2em; margin-bottom: 20px;">🌟 <strong>原项目仓库 / Original Repository</strong></p>
    <div style="display: flex; justify-content: center; align-items: center; gap: 20px; flex-wrap: wrap;">
        <a href="https://github.com/ImplFerris/pico-pico" target="_blank" style="text-decoration: none;">
            <img src="https://img.shields.io/github/stars/ImplFerris/pico-pico?style=social" alt="GitHub Repo stars">
        </a>
        <a href="https://github.com/ImplFerris/pico-pico" target="_blank" style="text-decoration: none;">
            <img src="https://img.shields.io/badge/Source_Code-ImplFerris/pico--pico-181717?style=for-the-badge&logo=github&logoColor=white" alt="GitHub">
        </a>
    </div>
    <p style="margin-top: 20px; opacity: 0.8; font-size: 0.95em;">
        点击上方按钮前往原项目仓库，为原作者 <strong>ImplFerris</strong> 点个 Star！<br/>
        <small>Please visit the original repository and give it a star!</small>
    </p>
</div>
