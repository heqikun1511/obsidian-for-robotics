[[cpp语法]]
### 核心含义

cpp

reinterpret_cast<T>(expr)

把 `expr` 的**二进制位**原封不动地当成类型 `T` 来解释。**不做任何数值转换、不做任何调整**。

### 和 `static_cast` 的区别

这是理解 `reinterpret_cast` 的关键：

cpp

// static_cast：做「有意义的类型转换」
double d = 3.14;
int i = static_cast<int>(d);   // i = 3，数值被转换了
// reinterpret_cast：只重新解释二进制位
int x = 0x3F800000;
float f = *reinterpret_cast<float*>(&x);   // f = 1.0f

第二个例子里：

- `x` 的二进制是 `0x3F800000`
    
- 这段二进制如果按 `int` 解释，是 `1065353216`
    
- 如果按 `float` 解释，是 `1.0f`（IEEE 754 编码）
    
- `reinterpret_cast` 就是「把同一段内存换个类型看」
    

**`static_cast` 会改变值，`reinterpret_cast` 不改变值，只改变「解释方式」。**

### 四种转换对比

|转换|用途|是否改变值|
|---|---|---|
|`static_cast`|相关类型间的转换（数值、继承体系）|是（数值转换）|
|`dynamic_cast`|多态类型的安全向下转换|否，但有运行时检查|
|`const_cast`|去掉/添加 `const`|否|
|`reinterpret_cast`|不相关类型间重新解释二进制|否|