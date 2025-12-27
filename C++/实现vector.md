[TOC]



# 0. 前言

在` C++` 标准模板库（`STL`）中，`vector` 是最常用的容器之一，它以动态数组的形式存在，兼顾了数组的随机访问效率和动态扩容的灵活性。很多开发者日常使用 `vector` 时，往往只关注其表层接口，却对其底层实现原理一知半解。本文将通过手写一个简化版 `vector`（`my_vector`），深入剖析 `vector` 的底层机制，重点讲解内存管理、深浅拷贝、迭代器失效、构造函数设计、交换机制等核心技术点，和大家交流我的一些理解。

***

# 1. `vector`核心结构：动态数组实现的基石

在正式分析之前，我们要明确`vector`的本质——**动态数组**，与静态数组（如`int arr[10]`）相比，`vector`的优势在于**可以根据数据数量变化动态调整内存大小，合理使用内存空间**。要实现这一功能，`vector`底层需要解决三个关键问题：

1. 如何跟踪当前已存储的数据范围？
2. 如何跟踪已分配的内存空间范围？
3. 如何在数据数量超过内存容量时高效扩容？

## 1.1. 核心成员：三个指针

`vector`的底层实现依赖三个核心指针，这三个指针共同定义了`vector`的内存布局和元素范围。我们在`my_vector`中，这三个指针定义如下：

```c++
template <class T>
class my_vector {
private:
    iterator _start;          // 指向内存块的起始位置（第一个元素）
    iterator _finish;         // 指向已存储元素的末尾位置（最后一个元素的下一个位置）
    iterator _end_of_storage; // 指向已分配内存块的末尾位置（内存的最后一个位置的下一个位置）
};
```

三个指针关系可以用下图表示：

![image-20250921105755150](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20250921105755150.png)

通过这三个指针，我们可以快速得到两个核心属性：

1. **大小(size)**：已存储元素的个数，` _finish - _start`

2. **容量(capacity)**：已分配内存可容纳最大元素个数，`_end_of_storage - _start`

这种设计的优势在于：

* 随机访问效率高：通过`_start + index`可以直接定位到第`index`个元素，时间复杂度$O(1)$
* 空间利用率可控：容量大于等于大小，预留的空间可以减少频繁扩容的开销
* 状态计算简单：无需额外存储`size`和`capacity`，只用指针差即可实现

## 1.2. 核心属性计算：`size`和`capacity`的实现

基于上述三个指针，`my_vector` 中 `size` 和 `capacity `的实现非常简洁：

```c++
// 容量相关函数
size_t capacity() const {
	if (!_start || !_end_of_storage) return 0;
	return _end_of_storage - _start; 
}
size_t size() const { 
	if (!_start || !_end_of_storage) return 0;
	return _finish - _start; 
}
bool empty() const { return _start == _finish; }
```

这里值得注意的是，当`my_vector`调用默认构造时，`_start _finish _end_of_storage`均为`nullptr`，直接计算差值会导致未定义行为，所以需要先判断指针是否为空

***

# 2. `vector`构造家族：从无到有的实现逻辑

`vector` 提供了多种构造方式，以满足不同场景下的初始化需求。手写 `my_vector` 时，需要覆盖核心的构造函数，包括**默认构造**、**n 个元素构造**、**迭代器区间构造**和**拷贝构造**。每种构造函数的实现都有其特定的设计考量，尤其是内存分配和元素初始化的逻辑。

## 2.1. 默认构造：空`vector`的初始化

创建一个空的`vector`，此时没有开辟内存，没有任何元素

```c++
my_vector() 
    :_start(nullptr)
    ,_finish(nullptr)
    ,_end_of_storage(nullptr)
{}
```

**设计要点**：

- 初始状态下不分配内存，避免不必要的空间浪费
- 三个指针统一初始化为`nullptr`，保证状态一致性
- 后续调用`push_back`或`reserve`时才会分配内存

## 2.2. n个元素构造：批量初始化的实现

当需要创建包含 n 个相同元素的 `vector` 时，就需要用到 n 个元素的构造函数。实现如下：

```c++
my_vector(size_t n,const T& val = T())
    :_start(nullptr)
    ,_finish(nullptr)
    ,_end_of_storage(nullptr)
{
      reserve(n); // 预留n个空间的内存
      // 逐个尾插
      for(size_t i = 0; i < n; i++){
          push_back(val); // 调用尾插函数
      }
}
```

**设计要点和注意事项**：

* 提前预留空间，避免频繁扩容
* 默认参数注意事项：

	1. **`T()`的本质是 “调用默认构造生成默认值”**：第二个参数`val`的缺省值是`T()`，在 `C++` 中，`T()`是一种**值初始化（value initialization）** 的语法，它的核心作用是：为一个`T`类型的变量创建一个 “默认的、合法的初始值”，而这个过程**必须依赖`T`的默认构造函数**，如`int`为0，`string`为空字符串
	1. **省略`val`时必须支持默认构造**：若调用`my_vector(n)`（没有传递第二个参数），`T`必须有默认构造函数，否则`T()`无法生成默认值，编译报错；
	1. **显式传`val`可规避限制**：若`T`不支持默认构造，只要调用时显式传入一个已有的`T`对象（如`my_vector(n, val)`），就能正常构造，无需依赖默认构造。

画个图理解：

![image-20250921113605291](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20250921113605291.png)

* 类型匹配问题：

第一个参数类型是无符号整数，调用时传参需要注意类型匹配，例如：`my_vector<int> v(3, 10)`中，3会被视为`int`类型，需要转化为`size_t`类型：`my_vector<int> v((size_t)3, 10)`

注意看调用调试代码：

![image-20250921114457554](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20250921114457554.png)

对于迭代器区间构造，我们马上实现，这样两份代码进行对比就更好理解了

# 2.3. 迭代器区间构造：通用容器的初始化方式

迭代器区间构造是`vector`的灵活构造方式之一，它支持了从任意提供迭代器的容器（如`list` `array` `map`等）中拷贝元素来初始化，实现如下：

```c++
template <class InputIterator>
my_vector(InputIterator first, InputIterator last)
    :_start(nullptr)
    ,_finish(nullptr)
    ,_end_of_storage(nullptr)
{
    // 计算迭代器区间内元素个数
    size_t n = 0;
    InputIterator tmp = first;
    while(tmp != last){
        ++tmp;
        ++n;
    }
        
    // 逐个拷贝新元素
    reserve(n);
    while(first != last){
        push_back(*first);
        ++first;
    }
}
```

**设计要点：**

1. 函数模板通用性

构造函数本身是一个函数模板，支持任意类型的输入迭代器，这意味着可以实现跨容器的初始化

2. 元素类型的隐式转换

支持从其他容器的初始化，例如从`vector<double>`的迭代器区间初始化`my_vector<int>`,此时`*first`（`double`类型）会隐式转换成`int`类型

使用示例：

```c++
//  测试迭代器范围构造
Vect::my_vector<int> v3(v1.begin() + 2, v1.begin() + 6);
print_my_vector(v3, "v3(迭代器范围v1[2]-v1[5])");

// 从数组的迭代器区间初始化（数组名可视为指针）
int arr[] = { 10,20,30 };
my_vector<int> v0(arr, arr + 3); // v0包含元素10,20,30
print_my_vector(v0, "v0(迭代器范围arr[0] - arr[2])");
```

## 2.4. 拷贝构造函数：深拷贝的实现与意义

拷贝构造函数用于创建一个新 `vector`，其内容与另一个 `vector` 完全相同。这里的关键是**深拷贝**—— 新` vector` 拥有独立的内存空间，修改新 `vector` 不会影响原 `vector`，反之亦然。

实现如下：

```c++
my_vector(const my_vector<T>& v)
     :_start(nullptr)
    ,_finish(nullptr)
    ,_end_of_storage(nullptr)
{
       // 预留和源对象一样大的空间
       reserve(v.capacity());
       
       // 逐个拷贝
       for(const auto& tmp : v){
           push_back(tmp);
       }
}
```

设计要点：

1. 为什么深拷贝？

* 采用浅拷贝（直接拷贝三个指针），新的`vector`会和旧的`vector`指向同一块内存空间
* 当其中一个被析构时，会释放内存空间，导致另一个`vector`的指针成为野指针，访问时会触发内存错误

![image-20250921120902449](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20250921120902449.png)

2. 范围`for`的使用：

* 通过`for (const auto& tmp : v)`遍历原 `vector` 的元素，范围 for 的本质是**遍历容器的每个元素，用循环变量接收元素**，`const auto&`可以避免不必要的拷贝（如果接收的是值而不是引用，就会调用拷贝构造），同时保证不修改原元素
* 对于复杂类型（如`my_vector<string>`），`push_back(tmp)`会调用 `string` 的拷贝构造函数（深拷贝），实现元素级别的深拷贝

浅拷贝的危害：

```c++
// 错误写法：浅拷贝，导致内存共享
my_vector(const my_vector<T>& v)
    :_start(v._start)
    , _finish(v._finish)
    , _end_of_storage(v._end_of_storage)
{ }
```

当发生如下操作时，会触发内存错误：

```c++
my_vector<int> v1;
v1.push_back(10);
my_vector<int> v2 = v1; // 浅拷贝，v2与v1共享内存
v1.~my_vector(); // 析构v1，释放内存
cout << v2[0]; // 错误：v2._start指向已释放的内存（野指针）
```

![image-20250921122327367](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20250921122327367.png)

# 3. `vector`内存管理：扩容、预留空间与释放

内存管理是 `vector` 底层实现的核心，直接影响 `vector` 的性能和正确性。`vector` 的内存管理主要包括预留空间（`reserve`）、扩容机制、内存释放三个部分。理解这部分内容，能够帮助开发者写出更高效的代码（如避免不必要的扩容）。

## 3.1. 预留空间（`reserve`）：提前分配内存的艺术

`reserve`函数的作用是**提前分配足够的内存空间**，但不创建任何元素。它的核心目的是避免后续插入元素时频繁扩容，提升性能。

实现如下：

```c++
void reserve(size_t input_num){
   // 保存旧空间的元素个数 后续需要重新定位_finish
    size_t old_size = size();
    
   // 扩容
    if(input_num > capacity()){
        T* tmp = new T[input_num];
        if(_start){
            // 此处不能用memcpy 需要循环赋值实现深拷贝
            for(size_t i = 0; i < old_size;++i)
            {
                tmp[i] = _start[i];
            }
            delete[] _start;
        }
        // 指向新的内存
        _start = tmp;
        _finish = _start + old_size;
        _end_of_storage = _start + input_num;
    }
}
```

设计要点：

1. 扩容机制的触发：只有传递的`input_num > capacity()`时，才进行扩容，
2. 旧元素的拷贝方式：**不能用`memcpy`拷贝！！！**，`memcpy`是逐字节的浅拷贝，会导致新老元素共享同一块内存，析构时会析构两次；采用循环赋值，会调用`T`类型的赋值运算符，实现深拷贝
3. 指针的更新：为什么保存旧元素的大小？我们开辟出一个新的空间，要释放`_start`

```c++
_start = tmp;
        // 致命错误：此时size() = _finish - _start，但_start已指向新内存，_finish还是旧内存的地址
        _finish = _start + size(); 
        _end_of_storage = _start + input_num;
```

经过调试发现，`size()`这个值不确定，陷入了死循环

![image-20250921132850714](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20250921132850714.png)

进一步思考：`size() = _finish - _start`，此时`_start`是新的，而`_finish`是旧的，两个指针指向的不是同一块空间，会引发错误，所以，一定要先记录旧元素大小

## 3.2. 扩容机制：`vector`的动态增长策略

当 `vector` 的元素个数（`size`）超过容量（`capacity`）时，就需要进行扩容。`vector `的扩容不是简单地在原有内存空间后追加，而是遵循以下步骤：

1. 分配一块更大的新内存空间（通常是原容量的 2 倍）
2. 将旧内存中的元素拷贝到新内存
3. 释放旧内存空间
4. 更新三个核心指针，指向新内存空间

在 `my_vector `中，扩容逻辑被封装在`reserve`函数中，由`push_back`或`insert`等函数间接触发。例如，`push_back`的实现如下：

```c++
void push_back(const T& input_val) {
    // 复用insert函数，在_end位置插入元素
    insert(_finish, input_val);
}

// 在pos之后插入元素
iterator insert(iterator pos, const T& input_val) {
	// 检查pos的合理性
	assert(pos <= _finish && pos >= _start);
	// 判断是否需要扩容
	if (_finish == _end_of_storage) {
		//  在reserve的逻辑里 会将_start原来的空间删除 而pos指向的是_start的旧空间
		// 所以释放旧空间之后 pos是野指针 会引发迭代器失效
		// 需要记录pos和_start的相对位置
		size_t len = pos - _start;
		size_t new_cap = capacity() == 0 ? 4 : 2 * capacity();
		reserve(new_cap);
		// 更新pos位置
		pos = _start + len;
	}

	// 从后往前挪动数据
	iterator end = _finish;
	while (end != pos) {
		*end = *(end - 1);
		end--;
	}
	*pos = input_val;
	_finish++;

	return pos;
}
```

### 3.2.1. 扩容后的关键问题： 迭代器失效

在`insert`函数中有个细节很容易被忽略

```c++
// 需要记录pos和_start的相对位置
		size_t len = pos - _start;
// ...
// 更新pos位置
		pos = _start + len;
// ...
```

这两行代码是为了解决**扩容导致迭代器失效的问题**，我们需要从迭代器本质和扩容对内存的影响两方面理解：

1. **迭代器的本质：指向内存的像指针一样的东西**

`vector` 的迭代器（如 `iterator`）本质是对 "指向元素内存地址的指针" 的封装（在我们的 `my_vector` 中，`iterator` 直接` typedef` 为 `T*`）。迭代器的有效性依赖于 "它指向的内存地址始终有效且属于 `vector`"。

2. **扩容为什么会导致迭代器失效？**

扩容的核心是 **分配新内存 → 拷贝旧元素 → 释放旧内存**，这个过程会让**所有指向旧内存的迭代器变成 野指针"**：

- 假设扩容前，`pos` 指向旧内存中的某个位置（如 `_start_old + 2`）；
- 扩容后，旧内存被 `delete[]` 释放（变为无效内存），`_start` 指向新内存；
- 此时若不更新 `pos`，`pos` 仍然指向已释放的旧内存地址，后续对 `pos` 的操作（如 `*pos = input_val`）会触发 "访问非法内存" 的未定义行为（程序崩溃、数据错乱等）。

3. **如何解决迭代器失效？——重新计算迭代器位置**

`size_t len = pos - _start;` `pos = _start + len;`这两行代码是先计算`pos`与旧的`_start`的相对偏移量，这个偏移量是索引，不受内存地址变化的影响，然后更新`pos`在新空间内相对于`_start`的位置。

利用这种方式`pos`从**指向旧内存的野指针**变为了**指向新内存对应位置的有效迭代器**

***

# 4. `vector`元素访问与修改：随机访问的实现

`vector` 作为动态数组，核心优势之一是**随机访问**（即通过索引快速定位元素），这一特性通过`operator[]`、`front()`、`back()`等接口实现。但元素访问的背后，不仅有 “高效定位” 的逻辑，还有 “边界检查” 的安全性考量 —— 我们需要同时理解 “如何实现随机访问” 和 “如何避免越界错误”。

## 4.1. 随机访问的底层逻辑：指针偏移

`vector` 的随机访问能力，本质源于其**连续的内存布局**—— 所有元素在内存中连续存储，通过 “起始指针 + 索引偏移” 即可直接定位到目标元素，时间复杂度为$ O (1)$。

以`operator[]`为例，其实现逻辑如下：

```c++
// 非const版本：支持修改元素
T& operator[](size_t index){
    // 边界检查
    assert(index < size());
    return _start[index];
}

// const版本：仅支持读取元素，不允许修改
const T& operator[](size_t index) const {
    // 边界检查
    assert(index < size()); 
    return _start[index];
}
```

**关键细节：`_start[index]`的本质**

`_start`是指向内存起始位置的指针（`T*`类型），`_start[index]`在编译器中会被解析为：`*( _start + index )`

即 “从起始指针向后偏移`index`个`T`类型大小的位置，再解引用获取元素”。例如，`T=int`（占 4 字节）时，`_start[2]`等价于`*(0x100 + 2*4) = *(0x108)`，直接定位到第 3 个元素（索引从 0 开始）。

这种 “指针偏移 + 解引用” 的方式，是随机访问效率的核心 —— 无需遍历元素，直接通过数学计算定位。

## 4.2. 首位元素快速访问

```c++
// 非const版本
T& front() {
    assert(!empty()); // 确保vector非空
    return *_start; // 首元素即_start指向的位置
}

T& back() {
    assert(!empty()); // 确保vector非空
    return *(_finish - 1); // 尾元素是_finish前一个位置（_finish指向最后一个元素的下一个）
}

// const版本（逻辑一致，仅返回const引用）
const T& front() const { ... }
const T& back() const { ... }
```

关键细节：`_finish - 1`的合理性

由于`_finish`指向 “已存储元素的末尾（最后一个元素的下一个位置）”，因此`_finish - 1`恰好指向最后一个元素。例如：

- 若 vector 有 3 个元素（size=3），`_start`指向 0x100，`_finish`指向 0x10C（int 类型，3*4=12 字节偏移）
- `_finish - 1` = 0x10C - 4 = 0x108，指向第 3 个元素（索引 2），即尾元素。

***

# 5. `vector`元素插入与删除：内存挪动与迭代器失效解析

`vector` 的`insert()`（插入元素）和`erase()`（删除元素）是高频操作，但这两个接口的底层逻辑复杂 —— 涉及 “元素挪动”“内存扩容” 和 “迭代器失效”，也是开发者最容易踩坑的地方。

## 5.1. `insert()`：插入元素

`insert()`的功能是 “在指定迭代器`pos`位置插入一个元素”，其实现需要处理 “扩容”“元素挪动”“迭代器更新” 三个核心步骤，具体代码如下：

```c++
iterator insert(iterator pos, const T& input_val) {
    // 步骤1：检查pos的合法性（必须在[_start, _finish]范围内）
    assert(pos <= _finish && pos >= _start);

    // 步骤2：判断是否需要扩容（元素个数达到容量上限）
    if (_finish == _end_of_storage) {
        // 计算pos与旧_start的相对偏移量（用于扩容后更新pos）
        size_t len = pos - _start;
        // 计算新容量（空vector初始4，否则2倍）
        size_t new_cap = capacity() == 0 ? 4 : 2 * capacity();
        reserve(new_cap); // 调用reserve扩容（释放旧内存，分配新内存）
        // 步骤3：更新pos（扩容后旧pos失效，用偏移量重新计算）
        pos = _start + len;
    }

    // 步骤4：从后往前挪动元素，腾出pos位置
    iterator end = _finish;
    while (end != pos) {
        *end = *(end - 1); // 后一个元素 = 前一个元素（拷贝）
        end--;
    }

    // 步骤5：在pos位置插入新元素
    *pos = input_val;
    // 步骤6：更新_finish（元素个数+1）
    _finish++;

    // 返回插入位置的迭代器（供用户后续使用）
    return pos;
}
```

**插入流程：**

假设 `vector` 当前状态：`size=3`，`capacity=4`，元素为`[1,2,3]`，在索引1位置插入`4`：

![image-20250921140218428](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20250921140218428.png)

## 5.2. `earse()`：删除元素

`erase()`的功能是 “删除指定迭代器`pos`位置的元素”，其实现需要处理 “元素挪动”“迭代器失效” 两个核心问题，代码如下：

```c++
	// 删除pos位置元素
	iterator erase(iterator pos) {
		// 检查pos的合理性
		assert(pos < _finish && pos >= _start);

		// 将pos之后的元素依次往前挪动一位
		iterator end = pos + 1;

		while (end != _finish) {
			*(end - 1) = *end; // 前一个元素 = 后一个元素（拷贝覆盖）
			end++;
		}

		_finish--;
	// 返回删除位置的下一个迭代器（原pos已失效，返回有效迭代器）
    return pos; // 注意：此时pos指向的是原pos+1的元素（因元素已挪动）
}
```

###  `earse()`的关键陷阱：迭代器失效

删除元素后，**被删除位置的`pos`**及之后的所有迭代器都会失效，这是因为元素发生了向前挪动，原迭代器指向的内存地址对应的元素已经改变：

![image-20250921141316057](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20250921141316057.png)

- 假设删除前，`it`指向`pos=0x104`（元素 4）；
- 删除后，`0x104`位置的元素变为 2，`it`仍然指向`0x104`，但此时`it`对应的 “原元素 4” 已被删除，若继续使用`it`（如`*it`），会访问到错误的元素（2）；
- 更严重的是，若删除后调用`erase(it)`，会再次删除`0x104`位置的元素（2），导致逻辑错误。

所以：正确的删除方式是，利用`erase()`的返回值——`erase()`返回**删除位置的下一个有效迭代器**（即原来`pos+1`位置的元素在挪动后的地址），通过 “用返回值更新迭代器” 避免使用失效的迭代器。

```c++
// 错误示例：使用失效的迭代器
my_vector<int> v = {1,4,2,3};
auto it = v.begin() + 1; // it指向4（pos=0x104）
v.erase(it); // 删除4后，it失效（仍指向0x104，但元素已变为2）
v.erase(it); // 错误：使用失效的it，实际删除的是2（原pos+1的元素）

// 正确示例：用erase的返回值更新迭代器
my_vector<int> v = {1,4,2,3};
auto it = v.begin() + 1; // it指向4
it = v.erase(it); // 删除4后，it被更新为指向2（原pos+1的元素）
v.erase(it); // 正确：删除2，此时v变为{1,3}
```

## 5.3. 按区间删除

除了删除单个元素，`vector` 还支持 “删除` [first, last) `区间内的所有元素”（范围 `erase`），其实现逻辑是对单个删除的批量优化：

```c++
// 按区间删除
iterator erase(iterator first, iterator last) {
	// [first, last)
	assert(first <= last && first >= _start && last <= _finish);

	// 计算需要删除的元素个数
	size_t len = last - first;

	// 从last开始，向前挪动元素，覆盖被删除区间
	iterator end = last;
	while (end != _finish) {
		*(first++) = *end++; // 用后面的元素覆盖被删除区间
	}

	// 更新_finish（减少删除的元素个数）
	_finish -= len;

	// 返回删除区间的起始位置（此时已指向原last位置的元素）
	return first - len;
}

```

## 5.4. 清空元素

`clear()` 函数的功能是 “删除所有元素”，但它的实现非常简单：

```c++
void clear() {
    _finish = _start; // 直接将_finish重置为_start，逻辑上清空所有元素
}
```

***

# 6. 完整代码（带测试案例）

```c++
// ==== my_vector.h ====
#include <iostream>
#include <cassert>
#include <algorithm>
using namespace std;

namespace Vect {
	template <class T>
	class my_vector {
	public:
		typedef T* iterator;
		typedef const T* const_iterator;

		// 迭代器
		iterator begin() { return _start; }
		iterator end() { return _finish; }
		iterator begin() const { return _start; }
		iterator end() const { return _finish; }
		const_iterator cbegin()const { return _start; }
		const_iterator cend() const { return _finish; }


		// 容量相关函数
		size_t capacity() const {
			if (!_start || !_end_of_storage) return 0;
			return _end_of_storage - _start; 
		}
		size_t size() const { 
			if (!_start || !_end_of_storage) return 0;
			return _finish - _start; 
		}
		bool empty() const { return _start == _finish; }

		void clear() { _finish = _start; }

		// 构造函数
		// 默认构造
		my_vector()
			:_start(nullptr)
			, _finish(nullptr)
			,_end_of_storage(nullptr)
		{ }

		// 构造n个元素
		my_vector(size_t n, const T& val = T()) 
			:_start(nullptr)
			, _finish(nullptr)
			, _end_of_storage(nullptr) 
		{
			reserve(n);
			// 将所有数据尾插
			for (size_t i = 0; i < n; i++)
			{
				push_back(val);
			}
		}

		// 迭代器区间初始化
		// 类模板的成员函数也能写成函数模板 支持了任意容器的迭代器初始化
		template <class InputIterator>
		my_vector(InputIterator first, InputIterator last) 
			:_start(nullptr)
			, _finish(nullptr)
			, _end_of_storage(nullptr)
		{
			// 计算元素个数
			size_t n = 0;
			InputIterator tmp = first;
			while (tmp != last) {
				++tmp;
				++n;
			}

			// 尾插n个元素
			reserve(n);
			while (first != last) {
				push_back(*first);
				++first;
			}
		}

		// 拷贝构造
		my_vector(const my_vector<T>& v)
			:_start(nullptr)
			, _finish(nullptr)
			, _end_of_storage(nullptr)
		{
			// 逐个拷贝源vector的元素
			reserve(v.capacity());
			for (const auto& tmp : v) push_back(tmp);
		}

		//// 浅拷贝
		//my_vector(const my_vector<T>& v)
		//	:_start(v._start)
		//	, _finish(v._finish)
		//	, _end_of_storage(v._end_of_storage)
		//{
		//}

		void swap(my_vector<T>& v) {
			// 交换三个核心指针
			std::swap(_start, v._start);
			std::swap(_finish, v._finish);
			std::swap(_end_of_storage, v._end_of_storage);
		}

		// 赋值运算符重载
		my_vector<T>& operator=(my_vector<T> v) {
			swap(v);
			return *this;
		}

		// 析构
		~my_vector() {
			delete[] _start;
			_start = _finish = _end_of_storage = nullptr;
	
		}

		// 元素访问
		T& operator[](size_t index){
			if(index < size()) return _start[index];
		}

		const T& operator[](size_t index) const {
			assert(index < size());
			return _start[index];
		}

		T& front() {
			assert(!empty());
			return *_start;
		}

		const T& front() const {
			assert(!empty());
			return *_start;
		}

		T& back() {
			assert(!empty());
			return *(_finish - 1);
		}

		const T& back() const {
			assert(!empty());
			return *(_finish - 1);
		}

		// 预留空间
		void reserve(size_t input_num) {
			// 提前保存旧空间大小 释放之后 _start是新的 但_finish是旧的
			size_t old_size = size();
			// 输入的值大于容量才触发扩容机制
			if (input_num > capacity()) {
				// 开辟一块新的空间 将旧空间数据拷贝到新空间
				T* tmp = new T[input_num];
				// 保证_start不为空 才可以将旧空间数据拷贝到新空间
				if (_start) {
					// memcpy有局限性 浅拷贝 遇到string list类会出问题
					//memcpy(tmp, _start, sizeof(T) * size());

					// 深拷贝
					for (size_t i = 0; i < old_size; i++)
					{
						tmp[i] = _start[i]; // 将值一个一个拷贝过去
					}
					delete[] _start;
				}
				_start = tmp;
				_finish = _start + size();
				_end_of_storage = _start + input_num;
			}
		}

		void push_back(const T& input_val) {
			//// 判断是否需要扩容
			//if (_finish == _end_of_storage) {
			//	size_t new_cap = capacity() == 0 ? 4 : 2 * capacity();
			//	reserve(new_cap);
			//}

			//*_finish = input_val;
			//_finish++;

			// 复用insert
			insert(_finish, input_val);
		}

		// 在pos位置插入元素
		iterator insert(iterator pos, const T& input_val) {
			// 检查pos的合理性
			assert(pos <= _finish && pos >= _start);
			// 判断是否需要扩容
			if (_finish == _end_of_storage) {
				//  在reserve的逻辑里 会将_start原来的空间删除 而pos指向的是_start的旧空间
				// 所以释放旧空间之后 pos是野指针 会引发迭代器失效
				// 需要记录pos和_start的相对位置
				size_t len = pos - _start;
				size_t new_cap = capacity() == 0 ? 4 : 2 * capacity();
				reserve(new_cap);
				// 更新pos位置
				pos = _start + len;
			}

			// 从后往前挪动数据
			iterator end = _finish;
			while (end != pos) {
				*end = *(end - 1);
				end--;
			}
			*pos = input_val;
			_finish++;

			return pos;
		}

		// 删除pos位置元素
		iterator erase(iterator pos) {
			// 检查pos的合理性
			assert(pos < _finish && pos >= _start);

			// 将pos之后的元素依次往前挪动一位
			iterator end = pos + 1;

			while (end != _finish) {
				*(end - 1) = *end;
				end++;
			}

			_finish--;

			// 返回删除位置的下一个迭代器（原pos已失效，返回有效迭代器）
			return pos; // 注意：此时pos指向的是原pos+1的元素（因元素已挪动）
		}

		// 按区间删除
		iterator erase(iterator first, iterator last) {
			// [first, last)
			assert(first <= last && first >= _start && last <= _finish);

			// 计算需要删除的元素个数
			size_t len = last - first;

			// 从last开始，向前挪动元素，覆盖被删除区间
			iterator end = last;
			while (end != _finish) {
				*(first++) = *end++; // 用后面的元素覆盖被删除区间
			}

			// 更新_finish（减少删除的元素个数）
			_finish -= len;

			// 返回删除区间的起始位置（此时已指向原last位置的元素）
			return first - len;
		}

	private:
		iterator _start;
		iterator _finish;
		iterator _end_of_storage;
	};


	// 打印vector信息的辅助函数
	template<class T>
	void print_my_vector(const Vect::my_vector<T>& v, const std::string& name) {
		std::cout << name << " - 大小: " << v.size()
			<< ", 容量: " << v.capacity()
			<< ", 元素: ";
		for (size_t i = 0; i < v.size(); ++i) {
			std::cout << v[i] << " ";
		}
		std::cout << std::endl;
	}

	// 封装所有测试逻辑的函数
	void test() {
		std::cout << "===== 开始测试my_vector所有接口 =====" << std::endl;

		//  测试默认构造函数
		Vect::my_vector<int> v1;
		print_my_vector(v1, "v1(默认构造)");

		//  测试push_back、size、capacity
		for (int i = 1; i <= 5; ++i) {
			v1.push_back(i);
		}
		print_my_vector(v1, "v1(push_back 1-5后)");

		// 测试reserve
		v1.reserve(10);
		print_my_vector(v1, "v1(reserve(10)后)");

		//  测试带参数构造函数
		Vect::my_vector<int> v2((size_t)3, 10);
		print_my_vector(v2, "v2(构造3个10)");

		//  测试迭代器范围构造
		Vect::my_vector<int> v3(v1.begin() + 2, v1.begin() + 6);
		print_my_vector(v3, "v3(迭代器范围v1[2]-v1[5])");

		// 从数组的迭代器区间初始化（数组名可视为指针）
		int arr[] = { 10,20,30 };
		my_vector<int> v0(arr, arr + 3); // v0包含元素10,20,30
		print_my_vector(v0, "v0(迭代器范围arr[0] - arr[2])");


		//  测试拷贝构造
		Vect::my_vector<int> v4(v1);
		print_my_vector(v4, "v4(拷贝v1)");


		//  测试赋值运算符
		Vect::my_vector<int> v5;
		v5 = v2;
		print_my_vector(v5, "v5(赋值v2)");

		// 测试operator[]和修改元素
		v5[0] = 99;
		v5[1] = 88;
		print_my_vector(v5, "v5(修改第0位为99、第1位为88后)");

		//  测试front和back
		std::cout << "v5 front: " << v5.front() << ", back: " << v5.back() << std::endl;

		//  测试insert
		auto it = v5.insert(v5.begin() + 1, 55);
		print_my_vector(v5, "v5(在第1位插入55后)");

		//  测试erase
		it = v5.erase(v5.begin() + 3);
		print_my_vector(v5, "v5(删除第3位元素后)");

		// 测试empty
		Vect::my_vector<int> v6;
		std::cout << "v6是否为空: " << (v6.empty() ? "是" : "否") << std::endl;
		v6.push_back(1);
		std::cout << "v6添加1后是否为空: " << (v6.empty() ? "是" : "否") << std::endl;
		print_my_vector(v6, "v6(添加1后)");

		std::cout << "===== 所有接口测试完成 =====" << std::endl;
	}

	void test_shallow_copy() {
		// 浅拷贝测试
		my_vector<int> v10;
		v10.push_back(10);
		my_vector<int> v20 = v10; // 浅拷贝，v2与v1共享内存
		v10.~my_vector(); // 析构v1，释放内存
		cout << v20[0]; // 错误：v2._start指向已释放的内存（野指针）
	}

	void test_reserve() {
		Vect::my_vector<int> v;
		v.push_back(1);
		v.push_back(2); 
		v.push_back(2); 
		v.push_back(2); 
		v.push_back(2); 
		v.push_back(2); 
		cout << "reserve前：size=" << v.size() << ", capacity=" << v.capacity() << endl;

		v.reserve(10); // 调用reserve，触发错误逻辑
		cout << "reserve后：size=" << v.size() << ", capacity=" << v.capacity() << endl;

	}
}

// ==== test.cpp ====
#include "my_vector.h"

int main() {
	Vect::test();
	//Vect::test_shallow_copy();

	// Vect::test_reserve();
	return 0;
}
```

***

# 7. 总结

1. **核心结构**：以 `_start`（内存起始）、`_finish`（元素末尾）、`_end_of_storage`（容量末尾）三个指针为基础，`size = _finish - _start`，`capacity = _end_of_storage - _start`，所有功能围绕内存布局展开。
2. **构造与初始化**：支持默认构造、n 个元素构造、任意容器迭代器区间构造（先计数再 `reserve` 优化性能），以及深拷贝构造（避免内存共享问题）。
3. **扩容机制**：容量不足时触发扩容，空容器初始容量默认 4，非空容器按 2 倍扩容；扩容流程为 “分配新内存→拷贝旧元素→释放旧内存→更新指针”，需注意扩容后迭代器失效，需通过偏移量重新计算。
4. **元素操作**：
   - 插入（`insert`/`push_back`）：需先检查扩容，再从后往前挪动元素，避免覆盖；
   - 删除（`erase`/`clear`）：`erase` 从后往前覆盖元素，返回有效迭代器，`clear` 仅重置 `_finish` 不释放内存；
   - 访问（`operator[]`/`front`/`back`）：需加边界检查，避免越界访问。
5. **效率与安全**：
   - 用 `reserve` 提前分配内存，减少扩容次数；
   - 区分深 / 浅拷贝，坚决避免内存共享导致的双重释放、野指针问题。

完结撒花，欢迎各位在评论区交流~
