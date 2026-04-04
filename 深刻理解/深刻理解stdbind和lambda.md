# std::bind 与 Lambda 用法详解

## 一、std::bind：延迟调用的包装器

核心思想：**有一个函数，现在不想调用，想把它打包成对象，以后再调用。**

### 基本用法

```cpp
void sayHello() { printf("hello\n"); }

auto f = std::bind(sayHello);
f();  // 等价于 sayHello()
```

### 预填参数（部分应用）

```cpp
void add(int a, int b) { std::cout << a + b << std::endl; }

// 固定 b=10，a 留着以后传
auto f = std::bind(add, std::placeholders::_1, 10);
f(30);  // add(30, 10) → 40
f(11);  // add(11, 10) → 21
```

`std::placeholders::_1` 是占位符，`_1` 对应调用时第 1 个参数，`_2` 对应第 2 个，以此类推。

### 绑定成员函数（最常用）

成员函数隐含 `this` 指针，必须显式传入：

```cpp
class Timer {
public:
    void onTimeOut() { ... }
    void onTimeWithVal(int val) { ... }
};
Timer t;

auto f1 = std::bind(&Timer::onTimeOut, &t);           // 无参
auto f2 = std::bind(&Timer::onTimeWithVal, &t, 666);  // 固定参数
auto f3 = std::bind(&Timer::onTimeWithVal, &t, std::placeholders::_1); // 留参数

f1();     // t.onTimeOut()
f2();     // t.onTimeWithVal(666)
f3(777);  // t.onTimeWithVal(777)
```

**一句话总结**：`std::bind(函数, 参数...)` 返回一个可延迟调用的函数对象，成员函数必须传 `this`，不确定的参数用占位符。

---

## 二、Lambda：就地定义的匿名函数

### 基本结构

```text
[捕获列表](参数列表) -> 返回类型 { 函数体 }
```

返回类型通常可以省略，由编译器推导。

**Lambda 相当于一个独立的作用域**，默认看不到外部变量，捕获列表负责把需要的外部变量"带进来"。

### 捕获列表详解

| 写法 | 含义 |
|------|------|
| `[]` | 不捕获任何变量 |
| `[x]` | 按值捕获 x（拷贝一份） |
| `[&x]` | 按引用捕获 x（操作原变量） |
| `[=]` | 按值捕获所有外部局部变量 |
| `[&]` | 按引用捕获所有外部局部变量 |
| `[this]` | 在类的成员函数里捕获 this，可访问成员变量和方法 |
| `[=, &x]` | 其他按值，x 按引用 |

### 全局变量与局部变量的区别

```cpp
int cnt = 0;  // 全局变量

// 全局变量不需要捕获，lambda 内直接访问
auto f1 = []() { cnt++; };   // ✅
auto f2 = [cnt]() { ... };   // ❌ 不能捕获全局变量，编译报错
```

**规则**：只有**自动存储期的局部变量**才需要捕获，全局变量和 `static` 变量直接可见，不能也不需要捕获。

### 按值捕获 vs 按引用捕获

```cpp
int num = 100;

// 按值捕获：拷贝一份，默认只读
// 加 mutable 可修改副本，但不影响原变量
auto f4 = [num]() mutable { num += 100; std::cout << num; };

// 按引用捕获：直接操作原变量
auto f5 = [&num]() { num += 100; std::cout << num; };

f4();  // 输出 200，num 仍为 100
f5();  // 输出 200，num 变为 200
```

**注意**：mutable lambda 的副本在多次调用之间**持久保留**，不会每次重置：

```cpp
f4();  // 副本 100 → 200，输出 200
f4();  // 副本 200 → 300，输出 300（不是 200！）
```

---

## 三、最重要的注意事项

### 引用捕获的生命周期陷阱

看到 `[&]` 或 `[&x]` 立刻要问：**这个引用指向的变量，在 lambda 被调用时还活着吗？**

```cpp
// 危险：返回引用了局部变量的 lambda
std::function<void()> getCallback() {
    int x = 10;
    return [&x]() { std::cout << x; };
    // 函数返回后 x 销毁，lambda 持有悬空引用 → 未定义行为
}

// 正确：按值捕获，把变量生命周期带进 lambda
std::function<void()> getCallback() {
    int x = 10;
    return [x]() { std::cout << x; };
}
```

**原则**：lambda 传出函数作用域时，一律按值捕获局部变量。

---

## 四、Lambda vs std::bind

两者完全等价，但现代 C++ 推荐 lambda：

```cpp
// std::bind（老风格）
auto f1 = std::bind(&Timer::onTimeOut, this);

// lambda（推荐）
auto f2 = [this]() { onTimeOut(); };
```

**Lambda 的优势**：

1. **语义更清晰**：函数体直接可见，不用推导 bind 的参数对应关系
2. **支持重载**：bind 遇到重载函数会编译报错，lambda 没有此问题
3. **可以在任何地方定义**：直接写在使用的地方，不需要提前声明
4. **编译器优化更好**：lambda 通常能被内联，bind 生成的中间对象有额外开销

---

## 五、捕获方式选择口诀

```txt
变量在哪里？
  全局 / static  → 直接用，不捕获
  类的成员变量   → 捕获 [this]
  局部变量
    只读          	 → [x]           按值
    要改原变量          → [&x]          按引用
    改副本不影响原值     → [x] mutable
    传出作用域          → 一定按值，禁止按引用
```