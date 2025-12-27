# 1. 内存分布

[TOC]



注：本文代码在`VS2022 debug x64`环境下运行

![image-20250815213532213](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20250815213532213.png)

* **栈**：非静态局部变量、函数参数、返回值存储区域，向下增长
* **内存映射段**：I/O映射方式，用于装载一个共享的动态内存库，用户可在系统接口创建共享内存，做进程间通信
* **堆**：动态分配的内存存贮在此，向上增长
* **数据段**：存储全局数据和静态数据
* **代码段**：存储可执行代码、只读常量

# 2. C++内存管理方式

`C++`利用`new`和`delete`操作符来进行动态内存管理

## 2.1 操作内置类型

```c++
void Test()
{
	// 动态申请一个int类型空间
	int* ptr1 = new int;

	// 动态申请一个int类型空间，并初始化为1
	int* ptr2 = new int(1);

	// 动态申请10个float类型的空间
	float* ptr3 = new float[10];

	// 动态申请3个float类型的空间,并分别初始化为1.1，1.2，1.3
	float* ptr4 = new float[10] {1.1, 1.2, 1.3};

	delete ptr1;
	delete ptr2;
	delete[] ptr3;
	delete[] ptr4;
}
```

![image-20250815215310706](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20250815215310706.png)

## 2.2 操作自定义类型

```c++
class A
{
public:
	A(int a1 = 1, int a2 = 2)
		:_a1(a1)
		, _a2(a2)
	{
		cout << "A(int a1, int a2)" << endl;
	}

	~A()
	{
		cout << "~A()" << endl;
	}

private:
	int _a1;
	int _a2;
};

int main()
{
	// 与C的malloc、realloc、calloc、free不同 new和delete对于自定义类型除了开辟空间还会调用构造和析构
	A* ptr1 = (A*)malloc(sizeof(A));
	A* ptr2 = new A{1,2};
	free(ptr1);
	delete ptr2;

	// 内置类型几乎一样
	int* ptr3 = (int*)malloc(sizeof(int));
	int* ptr4 = new int;
	free(ptr3);
	delete ptr4;

	A* ptr5 = (A*)malloc(sizeof(A) * 10);
	A* ptr6 = new A[10];

	free(ptr5);
	delete[] ptr6;
}
```

在申请动态内存空间时，`new`会调用构造，`delete`会调用析构

# 3. `operator new`与`operator delete`函数

`operator new`和`operator delete`是`C++`提供的的**全局函数**，`new`在底层调用`operator new`全局函数来申请空间，`delete`在底层通过调用`opreator delete`来释放空间

# 4. `new` `delete`的实现原理

## 4.1 内置类型

`new`和`malloc`基本类似，`delete`和`free`基本类似，不同的是：`new delete`申请和释放的是单个元素的空间，`new[] delete[]`申请和释放的是连续空间，`new`在申请失败时会抛异常，`malloc`会返回空

```c++
// 演示malloc和new的区别
int main()
{
    // 演示new分配失败时抛异常
    try {
        // 尝试分配一个极大的内存块，大概率会失败
        size_t largeSize = 1024ULL * 1024 * 1024 * 1024; // 1TB
        int* ptr1 = new int[largeSize];

        // 如果成功分配（几乎不可能），释放内存
        delete[] ptr1;
        std::cout << "new 分配成功" << std::endl;
    }
    catch (const std::bad_alloc& e) {
        // 捕获new分配失败的异常
        std::cout << "new 分配失败: " << e.what() << std::endl;
    }

    size_t largeSize = 1024ULL * 1024 * 1024 * 1024; 
    int* ptr2 = (int*)malloc(sizeof(largeSize));
    if (ptr2 == nullptr)
    {
        cout << "malloc 失败" << endl;
    }
    else
    {
        free(ptr2);
        ptr2 = nullptr;
        cout << "malloc 成功" << endl;
    }

    return 0;
};
```

## 4.2 自定义类型

* `new`：调用`operator new`函数申请空间，在申请的空间上执行构造函数，完成对象的构造
* `delete`：在空间上执行析构函数，完成对象的清理工作，调用`operator delete`函数释放对象的空间

```C++
// new delete原理
class obj
{
public:
	obj(int o)
		:_o(o)
	{
		cout << "调用obj(int o)" << endl;
	}
	~obj()
	{
		cout << "调用~obj()" << endl;
	}
private:
	int _o;
};

int main()
{
	// new的原理：调用operator new函数申请空间，执行构造函数完成对象的构造
	// 00007FF75AE32651  call        operator new (07FF75AE3104Bh)  
	obj* ptr = new obj(1);
    // delete的原理：执行析构函数，完成对象的析构，调用operator delete完成空间清理
	// 00007FF75AE326C6  call        obj::`scalar deleting destructor' (07FF75AE31280h)  
	delete ptr;
}
```

![image-20250816215540686](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20250816215540686.png)

* `new T[N]`：调用`operator new[]` ，在`operator new[]`实际调用`operator new`函数完成N个对象空间的申请，在申请的空间上执行N次构造函数
* `delete[]`：在释放的对象空间上执行N次析构函数，完成N个对象的资源清理，调用`operator delete[]`释放空间，时间在`operator delete[]`中调用`operator delete`来释放空间

```c++
// new[] delete[]原理
class obj
{
public:
	obj(int o = 1)
		:_o(o)
	{
		cout << "obj(int o)" << endl;
	}
	~obj()
	{
		cout << "~obj()"<<endl;
	}
private:
	int _o;
};

int main()
{
	// new[]的原理：调用operator new[]申请3个空间，在申请的空间上执行构造函数
	// 00007FF630742771  call        operator new[] (07FF630741212h)
	obj* ptr = new obj[3];// 16 要多开一个4字节空间存对象个数，方便知道调用几次析构函数，如果没有显式写析构，则为12
	// delete[]的原理： 执行三次析构函数，完成函数的析构，调用operator delete[]释放三个空间
	// 00007FF630742832  call        obj::`vector deleting destructor' (07FF63074105Ah)  
	delete[] ptr;

	return 0;
}
```

![image-20250816223635130](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20250816223635130.png)

![image-20250816223654559](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20250816223654559.png)



