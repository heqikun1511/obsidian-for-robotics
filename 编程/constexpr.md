[[cpp语法]]


### 它解决什么问题

传统 `const` 只表示“只读”，但不保证编译期就知道值：

cpp

const int size = 10;        // 编译期常量，但 const 不保证
const int n = rand();       // 运行时才确定，const 也允许

`constexpr` 更进一步：**明确要求值在编译期就能算出来**。

cpp

constexpr int size = 10;         // 编译期常量
constexpr int n = rand();        // 编译错误：rand() 不是 constexpr

### 三种主要用法

**1. `constexpr` 变量**

cpp

constexpr int max_size = 100;
constexpr double pi = 3.14159;

编译器会在编译期确定值，可以用于数组大小、模板参数等需要编译期常量的地方：





### `constexpr` vs `const`

|对比项|`const`|`constexpr`|
|---|---|---|
|含义|只读，不可修改|编译期常量|
|是否保证编译期求值|不保证|保证（在可能的情况下）|
|能否用于数组大小|仅当编译期已知|可以|
|能否用于模板参数|仅当编译期已知|可以|
|修饰函数|不能（成员函数表示不修改对象）|可以，表示可编译期求值|

`constexpr` 变量**隐含** `const`，但反过来不成立。