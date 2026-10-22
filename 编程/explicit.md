[[编程]]

## 一句话概括

**`explicit` 用来禁止编译器对构造函数（或 C++11 起的转换运算符）进行"隐式类型转换"，要求必须显式调用。**

## 一、没有 `explicit` 会怎样

### 隐式转换的例子

cpp

class Point {
public:
    Point(int x) { /* ... */ }   // 单参数构造函数
};
void foo(Point p);
int main() {
    Point p1 = 5;      // ✅ 隐式转换：5 → Point(5)
    foo(10);           // ✅ 隐式转换：10 → Point(10)
    Point p2 = 3.14;   // ✅ 隐式转换：double → int → Point
}

编译器会**自动**把 `int` 转成 `Point`，因为 `Point(int)` 是单参数构造函数。这种"方便"有时会带来**意外的 bug**。