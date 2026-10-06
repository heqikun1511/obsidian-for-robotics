> 场景：ESP32-P4 读取 VCU 的 CAN 转速报文，用 RMT 驱动约 8 颗、12 V 供电的 WS2815B 灯珠。基于 ESP-IDF 5.x 新版 RMT 驱动接口；具体 GPIO 请对照所用开发板的原理图确认。资料核对日期：2026-09-28。

## 1. 一句话理解 RMT

RMT（Remote Control Transceiver）最初用于红外遥控收发。它把 GPIO 的**高低电平持续时间**交给硬件定时，因此可以输出或采集一串精确脉冲。适用红外、单线可寻址灯带、部分步进脉冲等。CAN 接收仍由 CAN/TWAI 控制器负责；RMT 只控制 LED 的数据线。

```
VCU → CAN 收发器 → ESP32-P4 的 CAN/TWAI 驱动 → 转速/状态变量
                                               ↓
                                  灯光任务 → RMT TX → GPIO → WS2815B DIN
12 V 外部电源 ───────────────────────────────────→ 灯带 +12 V
ESP32-P4 GND ────────────────────────────────────→ 灯带 GND
```

## 2. RMT 的数据表示

- **Tick**：时长刻度。若 `resolution_hz = 10 MHz`，一个 tick = 100 ns。
- **Symbol**：一个 `rmt_symbol_word_t` 有两段 `(level0, duration0)`、`(level1, duration1)`。每段时长字段 15 位；常用一个 symbol 表示“高一段，再低一段”。
- **Channel**：配置为 TX 或 RX，通过 GPIO 矩阵连接引脚。ESP32-P4 官方 SoC 能力页列出一组共 8 个 RMT 通道，每通道 48 个 32 位存储字；应用一般让驱动分配通道，不必手工指定物理通道号。
- **Encoder**：把字节或业务数据转换为 symbol。RMT 硬件并不会自己理解 RGB、红外地址或 CAN 转速。

例如以 10 MHz 时钟表达一个约 1.3 µs 的位：

|位|高电平|低电平|RMT 时长（tick）|
|---|---|---|---|
|`0`|300 ns|1000 ns|高 3、低 10|
|`1`|700 ns|600 ns|高 7、低 6|

这组数值针对下文所引 WS2815B-V1 数据手册的范围；换了灯珠型号或批次，要按对应资料和示波器实测修正。

## 3. ESP-IDF 5.x 的使用顺序

1. `rmt_new_tx_channel()`：设置 GPIO、时钟、分辨率、存储字数、队列深度。
2. `rmt_new_bytes_encoder()`：指定逻辑 0 / 1 各自的高低电平时间和 MSB 优先。
3. `rmt_enable()`：使能通道。
4. 整理要发送的 GRB 字节；`rmt_transmit()` 只提交发送任务。
5. `rmt_tx_wait_all_done()`：等待硬件确实发完，期间不要修改发送缓冲区。
6. 保证数据线保持低电平足够长，让灯带锁存这一帧。关停时执行 `rmt_disable()`、`rmt_del_encoder()`、`rmt_del_channel()`。

RX 则用 `rmt_new_rx_channel()`、`rmt_enable()`、`rmt_receive()`，接收完成由 `on_recv_done` 回调通知；回调在中断上下文，不做阻塞处理。RMT RX 适合测量脉冲时序，不能替代 CAN 接口。

### 常用 TX 配置的意义

|字段|含义|这里的选择|
|---|---|---|
|`gpio_num`|输出引脚|由板级原理图确认可用引脚|
|`clk_src`|计时源|`RMT_CLK_SRC_DEFAULT`|
|`resolution_hz`|一个 tick 的时间分辨率|10 MHz，即 100 ns|
|`mem_block_symbols`|不用 DMA 时的通道内存符号数|48，达到 P4 单通道基本块大小|
|`trans_queue_depth`|待发送任务队列长度|4；本例每次等待完成，不依赖大队列|
|`flags.with_dma`|使用 DMA 后端|8 灯先用 `false`；长灯带/高负载再评估|