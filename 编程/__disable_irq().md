---
tags:
  - 嵌入式编程
---
## 一句话概括

**`__disable_irq()` 是 ARM Cortex-M 内核的底层函数，用于"关闭全局中断"——它设置 PRIMASK 寄存器，让 CPU 暂时不响应任何可屏蔽中断。**

## 一、这是 ARM Cortex-M 的专用函数

### 全名与来源

|函数|作用|来源|
|---|---|---|
|**`__disable_irq()`**|**关闭全局中断**|ARM CMSIS / 编译器内置|
|`__enable_irq()`|开启全局中断|同上|

- **CMSIS**（Cortex Microcontroller Software Interface Standard）：ARM 定义的标准接口
    
- 由编译器（Keil、IAR、GCC）**内置实现**，通常是一条汇编指令