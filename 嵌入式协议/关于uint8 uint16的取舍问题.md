

# 为什么 `FaultFlag` 用 `uint16_t` 而 `TelemetryPage` 用 `uint8_t`

## 一句话概括

**因为它们是两种不同用途的枚举：`TelemetryPage` 是"页号"（单值，选一个），`FaultFlag` 是"位标志"（多值，可组合）。位标志需要足够的位数来容纳多个可同时置位的状态，所以 `FaultFlag` 用 `uint16_t`。**

## 一、核心区别：单值枚举 vs 位标志枚举

### `TelemetryPage` —— 单值枚举（选一个）

cpp

enum class TelemetryPage : uint8_t {
    Joints = 0,
    Imu = 1,
    ServoCurrent = 2
};

- **同一时刻只能是其中一个值**
    
- 页号 0、1、2，**互斥**
    
- 取值范围小（0~2），**1 字节足够**
    
- 用 `uint8_t` 合理
### `FaultFlag` —— 位标志枚举（可组合）

cpp

enum FaultFlag : uint16_t {
    FaultNone = 0,              // 0000 0000
    FaultBadCrc = 1U << 0,      // 0000 0001
    FaultBadSequence = 1U << 1, // 0000 0010
    FaultTimeout = 1U << 2,     // 0000 0100
    FaultJointLimit = 1U << 3,  // 0000 1000
    FaultEstop = 1U << 4,       // 0001 0000
    FaultImu = 1U << 5,         // 0010 0000
    FaultServoTelemetry = 1U << 6 // 0100 0000
};

- **多个故障可以同时发生**
    
- 每个故障占**一个二进制位**
    
- 用 `|` 组合，用 `&` 检查
    
- 目前用了 7 位（bit 0 ~ bit 6）
## 二、位标志的用法

### 组合多个故障

cpp

uint16_t faults = FaultBadCrc | FaultTimeout | FaultEstop;
//              = 0b0001_0101
//                bit0+bit2+bit4 置位

### 检查某个故障

cpp

if (faults & FaultBadCrc) {
    // CRC 错误发生了
}

### 清除某个故障

cpp

faults &= ~FaultTimeout;   // 清掉 timeout 位

**每个故障是独立的位，互不干扰。**