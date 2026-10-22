[[编程]][
## `volatile`：要求每次都访问该变量

考虑这个循环：

```
while (!stop_requested) {
    do_work();
}
```

信号处理函数可能突然把 `stop_requested` 改成 `1`。`volatile` 告诉编译器：这个变量的值可能在当前代码看不到的地方发生变化，因此每次检查循环条件时都要重新读取它。

```
volatile std::sig_atomic_t stop_requested = 0;
```

但 `volatile` **不保证线程安全，也不提供锁或内存同步**。它在这里的用途是配合信号处理场景，让主程序能观察到处理函数写入的标志。