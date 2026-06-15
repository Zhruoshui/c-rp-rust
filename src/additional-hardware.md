{{#title Additional Hardware for Raspberry Pi Pico 2 Projects | impl Rust for RP2350}}

# Additional Hardware

## 可选硬件：调试探针

树莓派调试探针让为 Pico 2 刷写固件变得容易得多。没有它的话，每次想要上传新固件时都必须按下 BOOTSEL 按钮。该探针还提供了完善的调试支持，非常有用。

<div class="image-with-caption" style="text-align:center;">
    <img src="./images/Raspberry Pi Pico Debug Probe Hardware.jpg" alt="Raspberry Pi Debug Probe connected with Pico" style="width:400px; height:auto; display:block; margin:auto;"/>
    <div class="caption" style="font-size:0.9em; color:#555; margin-top:6px;">Raspberry Pi Pico Debug Probe</div>
</div>

本质上官方这个调试探针其实就是另一块 Pico 板子，如果你有另一块pico板子，也可以自己刷入**调试固件**：
- [数据手册18~19页](https://pip-assets.raspberrypi.com/categories/610-raspberry-pi-pico/documents/RP-008276-DS-1-getting-started-with-pico.pdf)
- [Debug using a Pico-series device](https://www.raspberrypi.com/documentation/microcontrollers/pico-series.html#debug-using-a-pico-series-device)
