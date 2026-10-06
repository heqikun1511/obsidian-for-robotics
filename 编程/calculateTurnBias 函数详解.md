## 一、函数签名

cpp

float CloudSafetyNode::calculateTurnBias(const Cloud::Ptr & obstacles) const

|部分|含义|
|---|---|
|`float`|返回值：转向偏置（弧度），正=左转，负=右转|
|`const Cloud::Ptr & obstacles`|输入：障碍物点云|
|`const`（末尾）|不修改成员变量|

---

## 二、整体逻辑

text

1. 把前方视场分成 N 个扇区，每扇区记录最近障碍距离
2. 遍历障碍物，按角度分配到对应扇区
3. 把扇区分成左半 / 右半，各取最近距离
4. 比较左右：哪边更空，就往哪边转
5. 归一化 + 限制最大偏置，返回

**核心思想**：左边空 → 往左转（正偏置）；右边空 → 往右转（负偏置）。


## 三、逐段解析

### 段 1：初始化扇区

cpp

const float max_range = static_cast<float>(max_range_m_);
std::vector<float> sector_clearance(static_cast<std::size_t>(sector_count_), max_range);
const double half_fov_rad = sector_half_fov_deg_ * kPi / 180.0;
const double sector_width = 2.0 * half_fov_rad / static_cast<double>(sector_count_);

|变量|含义|默认值|
|---|---|---|
|`max_range`|最大量程，扇区初值（乐观）|2.5m|
|`sector_clearance`|每个扇区的最近障碍距离，初值 max_range|7 个|
|`half_fov_rad`|半视场角转弧度|55° → 0.96 rad|
|`sector_width`|每个扇区的角宽度|2×0.96/7 ≈ 0.274 rad ≈ 15.7°|



**扇区划分示意**（sector_count = 7，半视场 55°）：

text

       左 ←──────── 视场 110° ────────→ 右
       
  扇区:  0    1    2    3    4    5    6
       -55° -39° -24° -8°  8°  24°  39° 55°
         │    │    │    │    │    │    │
       -55°                  0°                 +55°
       
  索引:  0    1    2    3    4    5    6
  位置: 最左  左  中左  中  中右  右  最右

**注意**：`angle = atan2(y, x)`，正角度在**左**（ROS 约定：x 前，y 左，z 上）。

