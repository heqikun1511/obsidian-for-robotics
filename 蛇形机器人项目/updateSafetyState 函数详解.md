
这个函数是安全节点的**状态机核心**：根据前方净空和转向偏置，决定当前的安全状态（畅通 / 转向 / 减速 / 停车）。它用**时序确认 + 停车锁存**来避免抖动和危险。

---

## 一、函数签名

cpp

CloudSafetyNode::SafetyState CloudSafetyNode::updateSafetyState(
    const float forward_clearance, const float turn_bias)

| 部分                  | 含义                                   |
| ------------------- | ------------------------------------ |
| `SafetyState`       | 返回类型：安全状态枚举                          |
| `forward_clearance` | 前方净空（来自 `calculateForwardClearance`） |
| `turn_bias`         | 转向偏置（来自 `calculateTurnBias`）         |
## 二、整体逻辑

text

┌─────────────────────────────────────────┐
│ 第一部分：更新内部计数器（时序防抖）        │
│   - danger_frame_count_                 │
│   - clear_frame_count_                  │
│   - stop_latched_（停车锁存）             │
└─────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│ 第二部分：根据锁存和距离，返回状态         │
│   - kStop → kSlow → kTurn → kClear      │
└─────────────────────────────────────────┘

**核心思想**：

- **危险要连续 N 帧才确认**（防止一帧误检就急停）
    
- **解除要连续 M 帧才确认**（防止刚停车又马上恢复）
    
- **停车是锁存的**（一旦停，必须确实安全才解除）



## 三、第一部分：时序防抖与锁存

### 情况 A：前方障碍物进入停车距离

cpp

if (forward_clearance <= stop_distance_m_) {
    ++danger_frame_count_;      // 危险帧 +1
    clear_frame_count_ = 0;     // 清空"畅通"计数
    if (danger_frame_count_ >= confirm_frames_) {
        stop_latched_ = true;   // 连续 N 帧危险 → 锁存停车
    }
}

**参数**：

- `stop_distance_m_`（默认 0.35m）：停车距离阈值
    
- `confirm_frames_`（默认 2）：连续 N 帧才确认
    

**逻辑**：

| 帧序  | forward_clearance | danger_count | stop_latched |
| --- | ----------------- | ------------ | ------------ |
| 1   | 0.30              | 1            | false        |
| 2   | 0.32              | 2            | **true** ✅   |
| 3   | 0.31              | 3            | true         |