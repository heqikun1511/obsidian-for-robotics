[[编程]]`std::sig_atomic_t` 是 C++ 中的一种整数类型，主要用于**信号处理函数和普通程序代码之间传递简单状态**。它保证对该类型对象的一次读取或写入不会被信号中断到一半。

常见写法：

```
#include <csignal>

volatile std::sig_atomic_t stop_requested = 0;

void handle_sigint(int) {
    stop_requested = 1;  // 收到 Ctrl+C 时，只设置标志
}

int main() {
    std::signal(SIGINT, handle_sigint);

    while (!stop_requested) {
        // 执行工作
    }

    // 在正常执行流程中做清理
}
```

这里两部分各有作用：

- `std::sig_atomic_t`：保证标志的一次读写对信号处理而言是原子的。
- `volatile`：让编译器在循环中重新读取标志，而不是一直沿用先前的值。

**边界很关键：**它适合单次赋值、读取这样的简单操作；`stop_requested++` 涉及读取、计算、写回，整体不受这个保证保护。它也不是通用的线程同步工具。