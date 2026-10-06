**`std::clamp(v, lo, hi)` 把值 `v` 限制在区间 `[lo, hi]` 内：小于 `lo` 返回 `lo`，大于 `hi` 返回 `hi`，否则原样返回 `v`。**

## 一、基本用法

cpp

#include <algorithm>   // C++17 起
std::clamp(value, lo, hi)

**语义**：

clamp(v,lo,hi)={lo若 v<lohi若 v>hiv否则（lo≤v≤hi）clamp(v,lo,hi)=⎩⎨⎧​lohiv​若 v<lo若 v>hi否则（lo≤v≤hi）​