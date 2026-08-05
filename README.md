# STM32F103 ADC 采集 + VOFA 串口传输

> 基于 STM32F103C8T6 的 ADC 多通道采集系统，通过 USART 将数据实时发送至 [VOFA+](https://www.vofa-plus.com/) 上位机进行波形显示。

---

## 📋 项目简介

本项目使用 STM32F103C8T6 的 **ADC1** 对模拟信号进行 **10 kHz** 定时采样，采用 **DMA 双缓冲**机制和**4 点滑动平均滤波**，将转换后的浮点电压值通过 **USART1** 以 VOFA+ JustFloat 协议格式实时发送至上位机。

### 核心技术点

| 技术 | 说明 |
|------|------|
| **定时触发 ADC** | TIM3 @ 10 kHz 触发 ADC 转换 |
| **DMA 双缓冲** | 200 采样点循环缓冲，半满/全满中断 |
| **滑动平均滤波** | 4 点移动平均，平滑噪声 |
| **VOFA+ JustFloat** | 浮点数 + 尾帧 `00 00 80 7F` 协议 |
| **CMake 构建** | 支持 GCC ARM 和 VS Code 开发 |

---

## 🛠️ 硬件需求

| 硬件 | 型号 |
|------|------|
| 主控 | STM32F103C8T6（Blue Pill / 最小系统板） |
| 调试器 | ST-Link V2 / J-Link / DAP-Link |
| USB-TTL | CH340 / CP2102（用于 VOFA+ 通信） |
| 信号源 | 电位器 / 传感器（0 ~ 3.3V 模拟输出） |

### 引脚连接

| 外设 | 引脚 | 功能 |
|------|------|------|
| USART1 TX | PA9 | VOFA+ 数据发送 |
| USART1 RX | PA10 | （可选） |
| ADC1 IN0 | PA0 | 模拟信号输入 |
| LED | PB8 | 数据发送指示（低电平亮） |

---

## 🚀 快速开始

### 1. 克隆仓库

```bash
git clone https://github.com/YMLGJ/STM32F103_ADC_VOFA.git
cd STM32F103_ADC_VOFA
```

### 2. 编译

```bash
# 使用 CMake + GCC ARM
cmake -B build -G Ninja -DCMAKE_TOOLCHAIN_FILE=cmake/gcc-arm-none-eabi.cmake -DCMAKE_BUILD_TYPE=Release
cmake --build build
```

### 3. 烧录

```bash
# ST-Link (OpenOCD)
openocd -f interface/stlink.cfg -f target/stm32f1x.cfg -c "program build/STM32F103_ADC_VOFA.elf verify reset exit"

# 或使用 STM32CubeProgrammer
STM32_Programmer_CLI -c port=SWD -w build/STM32F103_ADC_VOFA.elf -s
```

### 4. VOFA+ 配置

1. 打开 [VOFA+](https://www.vofa-plus.com/)
2. 添加 **串口** 控件，选择对应 COM 口
3. 波特率 **115200**，数据位 8，停止位 1，无校验
4. 添加 **波形图** 控件，协议选择 **JustFloat**
5. 点击连接，即可看到实时电压波形

---

## 📁 项目结构

```
├── Core/
│   ├── Inc/              # 头文件
│   │   ├── adc.h         # ADC 配置
│   │   ├── dma.h         # DMA 配置
│   │   ├── tim.h         # 定时器配置
│   │   ├── usart.h       # 串口配置
│   │   └── main.h        # 主程序头文件
│   └── Src/              # 源文件
│       ├── main.c        # 主程序（核心逻辑）
│       ├── adc.c         # ADC + DMA 初始化
│       ├── tim.c         # TIM3 定时触发配置
│       └── usart.c       # USART1 初始化
├── Drivers/              # HAL & CMSIS 驱动库
├── cmake/                # CMake 工具链配置
├── STM32F103xx_FLASH.ld  # 链接脚本
├── CMakeLists.txt        # CMake 构建文件
└── .gitignore
```

---

## 🔧 关键代码说明

### DMA 双缓冲 + 滑动滤波

```c
#define ADC_BUF_SIZE  200
#define HALF_BUF      100
uint16_t adc_dma_buf[ADC_BUF_SIZE];  // 循环 DMA 缓冲
float volt_filtered[HALF_BUF];       // 滤波后的电压值

/* 4 点滑动平均滤波 */
static float smooth_filter(uint16_t *buf, int idx) {
    int i0 = (idx - 3 + ADC_BUF_SIZE) % ADC_BUF_SIZE;
    int i1 = (idx - 2 + ADC_BUF_SIZE) % ADC_BUF_SIZE;
    int i2 = (idx - 1 + ADC_BUF_SIZE) % ADC_BUF_SIZE;
    float sum = buf[i0] + buf[i1] + buf[i2] + buf[idx];
    return sum * (3.3f / 4095.0f / 4.0f);  // 平均 + 电压转换
}
```

### VOFA+ JustFloat 协议

```c
/* 发送一个 float 值（4 字节）+ 尾帧 */
HAL_UART_Transmit(&huart1, (uint8_t*)&volt_filtered[i], 4, 10);
uint8_t tail_magic[4] = {0x00, 0x00, 0x80, 0x7F};
HAL_UART_Transmit(&huart1, tail_magic, 4, 10);
```

### 主循环处理逻辑

```c
while (1) {
    if (half_done) {   // DMA 半满 → 发送前半段
        half_done = 0;
        for (int i = 0; i < HALF_BUF; i++) { /* 发送... */ }
        HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_8);  // LED 闪烁
    }
    if (full_done) {   // DMA 全满 → 发送后半段
        full_done = 0;
        for (int i = 0; i < HALF_BUF; i++) { /* 发送... */ }
        HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_8);
    }
}
```

---

## 📊 VOFA+ 显示效果

连接 VOFA+ 后，波形图会实时显示 ADC 采样的电压曲线，更新率约为 **50 Hz**（200 点 × 50 帧/秒 = 10000 采样/秒）。

---

## 📄 License

本项目使用 MIT License。

---

# STM32F103 ADC Acquisition + VOFA Serial Transmission

> ADC multi-channel acquisition system based on STM32F103C8T6, transmitting real-time data to [VOFA+](https://www.vofa-plus.com/) for waveform visualization via USART.

---

## 📋 Overview

This project uses the **ADC1** of STM32F103C8T6 to sample analog signals at **10 kHz** via TIM-triggered conversion, with **DMA double-buffering** and a **4-point moving average filter**. Filtered floating-point voltage values are transmitted over **USART1** in VOFA+ JustFloat protocol format to the host computer.

### Key Features

| Feature | Description |
|---------|-------------|
| **Timer-Triggered ADC** | TIM3 triggers ADC conversion @ 10 kHz |
| **DMA Double-Buffer** | 200-sample circular buffer with half/full interrupts |
| **Moving Average Filter** | 4-point sliding window for noise reduction |
| **VOFA+ JustFloat** | Float values + tail frame `00 00 80 7F` protocol |
| **CMake Build** | Supports GCC ARM & VS Code development |

---

## 🛠️ Hardware Requirements

| Component | Model |
|-----------|-------|
| MCU | STM32F103C8T6 (Blue Pill / Minimum System Board) |
| Debugger | ST-Link V2 / J-Link / DAP-Link |
| USB-TTL | CH340 / CP2102 (for VOFA+ communication) |
| Signal Source | Potentiometer / Sensor (0 ~ 3.3V analog output) |

### Pin Connections

| Peripheral | Pin | Function |
|------------|-----|----------|
| USART1 TX | PA9 | VOFA+ data transmit |
| USART1 RX | PA10 | (Optional) |
| ADC1 IN0 | PA0 | Analog signal input |
| LED | PB8 | Data transmission indicator (active low) |

---

## 🚀 Quick Start

### 1. Clone

```bash
git clone https://github.com/YMLGJ/STM32F103_ADC_VOFA.git
cd STM32F103_ADC_VOFA
```

### 2. Build

```bash
cmake -B build -G Ninja -DCMAKE_TOOLCHAIN_FILE=cmake/gcc-arm-none-eabi.cmake -DCMAKE_BUILD_TYPE=Release
cmake --build build
```

### 3. Flash

```bash
# ST-Link (OpenOCD)
openocd -f interface/stlink.cfg -f target/stm32f1x.cfg -c "program build/STM32F103_ADC_VOFA.elf verify reset exit"

# Or using STM32CubeProgrammer
STM32_Programmer_CLI -c port=SWD -w build/STM32F103_ADC_VOFA.elf -s
```

### 4. VOFA+ Configuration

1. Launch [VOFA+](https://www.vofa-plus.com/)
2. Add **Serial Port** widget → select the correct COM port
3. Baud rate **115200**, 8 data bits, 1 stop bit, no parity
4. Add **Waveform** widget → set protocol to **JustFloat**
5. Click connect to view real-time voltage waveforms

---

## 📁 Project Structure

```
├── Core/
│   ├── Inc/              # Header files
│   │   ├── adc.h         # ADC config
│   │   ├── dma.h         # DMA config
│   │   ├── tim.h         # Timer config
│   │   ├── usart.h       # UART config
│   │   └── main.h        # Main header
│   └── Src/              # Source files
│       ├── main.c        # Main logic
│       ├── adc.c         # ADC + DMA init
│       ├── tim.c         # TIM3 trigger config
│       └── usart.c       # USART1 init
├── Drivers/              # HAL & CMSIS drivers
├── cmake/                # CMake toolchain config
├── STM32F103xx_FLASH.ld  # Linker script
├── CMakeLists.txt        # CMake build file
└── .gitignore
```

---

## 📄 License

This project is licensed under the MIT License.
```

---

### 使用前替换

把文中 `https://github.com/YMLGJ/STM32F103_ADC_VOFA.git` 改为你的实际仓库地址即可。

保存到 `e:\STM32CubeTest\2.ADC_Collect_Transmit_VOFA_F103C8T6\README.md`，然后提交推送：

```bash
git add README.md
git commit -m "Add bilingual README"
git push
```
