[[编程]]
## 一、基本含义

cpp

std::mutex mutex_;

- `std::mutex`：C++11 起 `<mutex>` 头文件提供的**互斥量**类
    
- `mutex_`：成员变量名（末尾下划线表示私有成员）
    
- 作用：**保证同一时刻只有一个线程能访问被保护的资源**
    

**核心思想**：加锁 → 访问 → 解锁，其他线程必须等锁释放才能进入。

## 二、为什么需要 mutex？

### 没有锁的问题

假设两个线程同时修改同一个变量：

cpp

// 共享变量
int counter = 0;
// 线程 A 和线程 B 同时执行：
counter++;   // 实际是三步：读 → 加 1 → 写回

时间线：

text

时刻 1: A 读到 counter = 5
时刻 2: B 读到 counter = 5      ← 两个都读到 5
时刻 3: A 写回 counter = 6
时刻 4: B 写回 counter = 6      ← 期望是 7，结果丢了 1

这叫 **数据竞争（data race）**，结果不确定，是**未定义行为（UB）**。

### 加锁后

cpp

std::mutex mutex_;
int counter = 0;
// 两个线程都这样写：
{
    std::lock_guard<std::mutex> lock(mutex_);
    counter++;
}

时间线变成**串行**：

text

时刻 1: A 拿到锁，读 5，加 1，写 6，释放锁
时刻 2: B 拿到锁，读 6，加 1，写 7，释放锁
→ 正确结果 7