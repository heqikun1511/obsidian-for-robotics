[[CAN]]
###  内核只认 `can_filter`，不认你的自定义结构

你的代码里有一套自定义的过滤器表示，比如：

cpp

struct CanFilter {
    std::uint32_t identifier;          // 要匹配的 ID
    std::uint32_t mask;                // 掩码
    CanIdentifierFormat format;        // 标准/扩展/任意
};

这是**用户态**为了方便业务逻辑定义的。但 Linux 内核的 SocketCAN 子系统只认它自己定义的结构：

c

struct can_filter {
    canid_t can_id;
    canid_t can_mask;
};

当你调用 `setsockopt` 把过滤器设到 socket 上时：

c

setsockopt(fd, SOL_CAN_RAW, CAN_RAW_FILTER,
           filters, count * sizeof(struct can_filter));
  内核期望的是一块**按 `can_filter` 布局的二进制内存**。你传自定义结构进去，内核按 `can_filter` 的布局去解析，字段对不上，行为就完全错了。

所以**必须转换**：把自定义表示翻译成内核认识的 `can_filter`。


###  两套表示的语义不完全一样

自定义过滤器里的 `format` 字段（`kAny` / `kStandard` / `kExtended`），在 `can_filter` 里**没有对应字段**。内核的 `can_filter` 只有两个字段：`can_id` 和 `can_mask`。

那「标准帧还是扩展帧」这个信息怎么表达？**只能编码进 `can_id` 和 `can_mask` 的标志位里**。

这就是为什么需要那段 `switch`：

|自定义格式|翻译方式|
|---|---|
|`kAny`|不设置 `CAN_EFF_FLAG`，掩码里也不要求这一位|
|`kStandard`|掩码里置 `CAN_EFF_FLAG`，要求这一位为 0|
|`kExtended`|`can_id` 和掩码里都置 `CAN_EFF_FLAG`，要求这一位为 1|

**内核没有「格式」这个概念，只有「ID + 掩码」**。所以「标准/扩展」必须被「编码」成标志位，转换这一步不可避免。