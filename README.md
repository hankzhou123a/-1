STM32F103C8T6 USART2 DMA 不定长收发

本项目实现电脑与 STM32 之间的串口回环通信：电脑发送一段数据，STM32 通过 UART DMA 接收，在检测到 UART 空闲事件后按实际长度 DMA 原样回传。

## 功能说明

- 主控：STM32F103C8T6，使用 STM32 HAL 库。
- USART2，默认 115200 baud、8 数据位、无校验、1 停止位（8N1）。
- UART 接收 DMA + UART 空闲事件，支持不定长数据帧。
- DMA 接收与发送缓冲区各 128 字节；接收内容按字节回传，不依赖字符串结束符。
- 上电后串口发送一次 `UART ready\r\n`，随后等待电脑发送数据。
- DMA 发送完成后重新启动接收。

当前采用单缓冲区顺序收发：收完一帧后回传，回传完成再启动下一帧接收。缓冲区长度为 128 字节。连续高速发送的数据可能在回传期间丢失；超过缓冲区长度的数据需要在串口助手中分段发送并确认分段效果。

## 硬件清单

| 数量 | 硬件 | 型号或要求 |
|---|---|---|
| 1 | MCU | STM32F103C8T6（Cortex-M3，64 KB Flash，20 KB SRAM） |
| 1 | 开发板 | 江协科技配套 STM32F103C8T6 开发板/套件 |
| 1 | USB 转串口模块 | USB-TTL，3.3 V TTL 电平 |
| 若干 | 杜邦线 | 连接串口 TX、RX、GND |
| 1 | 电脑 | Keil MDK5、STM32CubeMX、串口调试助手 |
| 1 | 下载器 | ST-LINK（用于下载和调试；若开发板另有下载方式，可按板卡方式使用） |

### USB-TTL 接线

| USB-TTL | STM32F103C8T6 |
|---|---|
| TXD | PA3（USART2_RX） |
| RXD | PA2（USART2_TX） |
| GND | GND |

TX/RX 交叉连接，USB-TTL 和开发板必须共地。确认使用 3.3 V TTL 电平；不要将 RS-232 电平接到 MCU。USB-TTL 的 VCC 不必连接，开发板单独供电即可。

## CubeMX 与外设配置

工程文件：

- CubeMX：[`chuan kou tong xing.ioc`](chuan%20kou%20xing.ioc)
- Keil：[`MDK-ARM/chuan kou tong xing.uvprojx`](MDK-ARM/chuan%20kou%20xing.uvprojx)

主要配置如下：

1. MCU 选择 STM32F103C8T6（Keil 设备名 `STM32F103C8`）。
2. USART2 设为 Asynchronous：PA2 设为 TX，PA3 设为 RX。
3. USART2 参数：115200 baud、8 位数据、无校验、1 个停止位、无硬件流控。
4. 系统时钟使用 HSI 8 MHz，未启用 PLL；AHB/APB 分频为 1。
5. USART2_RX 使用 DMA1 Channel 6，方向为外设到内存；USART2_TX 使用 DMA1 Channel 7，方向为内存到外设。两路均为字节宽度、内存地址递增、Normal 模式。
6. 需要启用 USART2、DMA1 Channel 6、DMA1 Channel 7 中断。

**工程配置备注：**当前 `.ioc` 保存了 MCU、USART2 和引脚配置；DMA 初始化及相应 NVIC 配置实际由 `Core/Src/stm32f1xx_hal_msp.c` 中的 `HAL_UART_MspInit()` 完成，DMA 项未完整记录在 CubeMX 图形配置中。重新通过 CubeMX 生成代码时，请检查 DMA/NVIC 设置和 MSP 初始化，避免配置丢失或重复。

## 串口收发处理流程

1. `main()` 初始化 HAL、系统时钟、GPIO 和 USART2，发送启动提示 `UART ready\r\n`。
2. `UART2_StartRxDMA()` 调用 `HAL_UARTEx_ReceiveToIdle_DMA()`，启动最多 128 字节的 DMA 接收，并关闭 DMA 半传输中断。
3. 收到一段数据后，UART 空闲事件（或 DMA 缓冲区达到长度）触发 `HAL_UARTEx_RxEventCallback()`，参数 `Size` 表示本次收到的字节数。
4. 回调将 `rx_buf` 中的有效字节复制到独立的 `tx_buf`，调用 `HAL_UART_Transmit_DMA()` 原样回传。
5. DMA 发送完成后触发 `HAL_UART_TxCpltCallback()`，重新启动 DMA 接收，等待下一帧。

接收与发送缓冲区分开，避免 DMA 发送期间接收数据覆盖待发送内容。应用逻辑和回调位于 `Core/Src/main.c`，UART 与 DMA 引脚、中断初始化位于 MSP 和中断文件。新增应用函数、缓冲区已有 Doxygen 风格基础注释。

## 编译、下载与验证

1. 用 Keil MDK5 打开 `MDK-ARM/chuan kou tong xing.uvprojx`，选择工程目标并执行 Rebuild。
2. 编译成功后通过 ST-LINK 下载；按开发板供电方式给板子供电。
3. USB-TTL 连接电脑和开发板，打开串口调试助手，选择对应 COM 口并设置 115200、8N1。
4. 复位开发板，接收区应看到 `UART ready`。
5. 发送 `hello123` 等内容，接收区应看到 STM32 原样回传。可关闭串口助手的本地回显，避免将电脑自身回显误认为回传。
6. 改变发送内容和长度重复验证。当前 DMA 缓冲区大小为 128 字节。

## 目录结构

```text
Core/       应用源码、头文件及 HAL 配置
Drivers/    STM32F1 HAL 与 CMSIS 驱动
MDK-ARM/    Keil MDK5 工程
EWARM/      IAR 工程文件
*.ioc       STM32CubeMX 配置
```
