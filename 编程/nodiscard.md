[[cpp语法]]


### 基本用法

cpp

[[nodiscard]] int computeSomething();
// 调用
computeSomething();              // 警告：忽略了返回值
int x = computeSomething();      // OK

如果调用方忽略了返回值，编译器会发出警告（不是错误），类似：

text

warning: ignoring return value of 'int computeSomething()',
         declared with attribute 'nodiscard' [-Wunused-result]

### 为什么需要它

有些函数的返回值**承载了关键信息**，忽略它往往意味着 bug。经典例子：

cpp

[[nodiscard]] bool isValid(const CanFrame& frame);
// 错误写法：检查了但没用结果
isValid(frame);   // 警告！这行代码等于白写
// 正确写法
if (!isValid(frame)) {
    return;
}

再比如资源分配、错误码返回：

cpp

[[nodiscard]] void* allocate(std::size_t size);
[[nodiscard]] std::error_code connect();

`[[nodiscard]]` 让编译器帮你抓住「调了但没接返回值」这类低级错误。

### 能加在哪

**1. 函数声明**

cpp

[[nodiscard]] int foo();

**2. 类或结构体**

C++20 起，可以加在类上，表示**这个类所有返回该类型对象的函数**都受此约束：

cpp

class [[nodiscard]] ErrorCode {
    // ...
};
ErrorCode doSomething();
doSomething();   // 警告：返回了 ErrorCode 但被忽略

**3. 构造函数**

C++20 起可以加在构造函数上：

cpp

class [[nodiscard]] Handle {
public:
    [[nodiscard]] Handle(int fd);
};
Handle(3);   // 警告：创建了临时对象就丢弃

**4. 枚举**

cpp

enum class [[nodiscard]] Status { Ok, Error };

### 带说明信息的写法（C++20）

cpp

[[nodiscard("check the error code")]]
std::error_code connect();

忽略返回值时，警告信息里会带上这句说明，方便定位问题。

### 和 `[[maybe_unused]]` 的区别

这两个容易混，但用途相反：

|属性|作用|
|---|---|
|`[[nodiscard]]`|加在**函数**上，警告调用方别忽略返回值|
|`[[maybe_unused]]`|加在**变量/参数/函数**上，告诉编译器“我知道它没被用，别警告”|

cpp

[[nodiscard]] int foo();
// 我就是不想用返回值，明确表示
[[maybe_unused]] int x = foo();   // 不警告

### 在 CAN 代码里的典型用法

结合你之前看的 `CanFrame` 结构体，常见场景：

cpp

// 解析函数：返回值表示是否成功，必须检查
[[nodiscard]] bool decodeCanFrame(const CanFrame& frame, Signal& out);
// 校验函数：返回值是判断依据，不能忽略
[[nodiscard]] bool isValid(const CanFrame& frame);
// 发送函数：返回错误码，必须处理
[[nodiscard]] std::error_code send(const CanFrame& frame);

这样如果有人写了 `send(frame);` 而不检查返回码，编译器会直接警告，避免「发了但不知道成没成功」的隐患。

### 编译器支持

- **C++17**：`[[nodiscard]]` 正式进入标准
    
- **C++20**：支持加在类、构造函数、枚举上，支持带说明信息
    

主流编译器都支持：

|编译器|支持情况|
|---|---|
|GCC|7+ 支持 C++17 版本，10+ 支持 C++20 扩展|
|Clang|3.9+ 支持 C++17 版本，10+ 支持 C++20 扩展|
|MSVC|2017 15.3+ 支持|

### 注意事项

**1. 它只是警告，不是错误**

忽略返回值不会导致编译失败，除非你开了 `-Werror` 把警告升级成错误。在 CI 里通常建议开 `-Werror`，强制处理。

**2. 不要滥用**

只给**确实需要检查返回值**的函数加。如果每个函数都加，警告会泛滥，反而让人麻木。

**3. 标准库里的例子**

很多标准库函数已经标了 `[[nodiscard]]`，比如：

cpp

[[nodiscard]] bool empty() const;
[[nodiscard]] std::size_t size() const;
[[nodiscard]] T* data() noexcept;

所以 `vec.empty();` 这种写法会触发警告——因为调用 `empty()` 却不看结果，逻辑上没意义。

### 一句话总结

**`[[nodiscard]]` 是 C++17 的属性，加在函数（或类、构造函数、枚举）上，表示返回值不应被忽略。调用方如果丢弃返回值，编译器会发出警告，帮助抓住「调了但没接结果」这类常见 bug。**