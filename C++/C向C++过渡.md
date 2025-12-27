# 1.命名空间的引入

首先看一段代码
```
#include <stdio.h>

int rand = 10;
int main()
{
	printf("%d\n", rand);
	return 0;
}
```
编译并不会报错，但如果加上头文件`stdlib`呢？
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/802ac96b8d9346dfbb428ca58c96ade5.png)

编译器告诉我们，`rand`重定义了，以前的定义是函数，因为在`stdlib`这个库中，`rand`是一个函数，将头文件展开变量`rand`会和库里面的`rand`冲突
我们在编码时，会创建很多的变量、函数，我们并不能保证每个变量和函数都不能同名，或者是不和库中的函数重名，所以我们引入关键字`namespace`，基本格式如下：
```
namespace name
{ }
```
`namespace`的本质是定义一个**命名空间域**，避免头文件展开后会和库函数冲突，这个域跟全局域各自独立，不同的域可以定义同名变量
```
#include <stdio.h>
#include <stdlib.h>
// 引入namespace进行命名隔离
namespace Vect
{
	int rand = 0;
}

int main()
{
	printf("%d", Vect::rand);
	return 0;
}
```
编译通过：![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/e018ab9bd05143b982937766bb9fce39.png)

下面介绍`namespace`的一些性质
## 1.1. `namespace`的性质
- 不影响变量/函数的生命周期，仅作名字的隔离作用
- `namespace`只能定义在全局，可以嵌套定义，多文件下同名的命名空间会进行合并
```
#include <stdio.h>
#include "head1.h"

namespace Vect
{
	// 1. 命名空间可以定义变量/函数/类型
	int num = 10;
	int* ptr = &num;

	void Print()
	{
		printf("HELLO,C++!\n");
	}

	struct node
	{
		int val;
		struct node* next;
	};

	// 2. 命名空间可以嵌套定义
	namespace coke
	{
		// ...
	}
}

int main()
{
	return 0;
}
```
## 1.2. `namespace`的使用
编译器在编译时，会按照一定的顺序进行查找变量/函数/类型
	1. 局部域，通俗理解是自己家
	2. 全局域，通俗理解是公共区域；展开的命名空间（别人家声明你可访问），这两个域相同优先级

而查找命名空间也有三种方式：
1. `using`将命名空间中某个成员展开，项目中推荐这种方式
2. 使用**域作用限定符 `::`** 指定查找
3. `using`展开全部的命名空间，项目中非常不推荐，日常练习无所谓
```c++
#include <stdio.h>
// 使用方法

// 1.域作用限定符指定访问
int num = 10;
namespace Vect
{
	int num = 0;
	char ch = '?';
}

int main()
{
	printf("%d\n", Vect::num);
	printf("%c\n", Vect::ch);
	// 若域作用限定符左边无命名空间，默认查找全局域
	printf("%d\n", ::num);
	printf("%d\n", num);
}
```

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/3b073115884f4883b844d21c3e97e80d.png)

```c++
// 2. using展开局部对象
namespace Vect
{
	int num = 0;
	char ch = '?';
}
using Vect::num;
int main()
{
	printf("%d\n", num);
	printf("%c\n", Vect::ch);
	// 若域作用限定符左边无命名空间，默认查找全局域
}
```
![!\[\[Pasted image 20250626191832.png\]\]](https://i-blog.csdnimg.cn/direct/c2e3addde5874584ad7af1f689362f76.png)

```c++
// 3. using展开全体对象
namespace Vect
{
	int num = 0;
	char ch = '?';
}
using namespace Vect;
int main()
{
	printf("%d\n", num);
	printf("%c\n", ch);
	// 若域作用限定符左边无命名空间，默认查找全局域
}
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/1ee070f0c82f4dec8b50b779ad2cf73b.png)


*思考：局部域、全局域、命名空间域有什么联系和区别*
局部域、全局域和命名空间域是 C++ 中三种作用域：
- **局部域**：在函数或代码块内部，只在当前块内有效。
  
- **全局域**：定义在所有函数外部，整个程序都能访问，可能需要注意`extern`。
  
- **命名空间域**：用来组织代码，防止命名冲突，通过 `namespace::变量` `展开局部变量` 或者`全部展开` 三种方式访问。
  

三者的**区别在于作用范围和访问方式**，它们之间可以嵌套使用，但同名时，遵循局部优先原则，命名空间通过域作用限定符来区分。实际开发中，我们倾向使用局部变量和命名空间来增强代码的可读性和安全性，尽量少使用全局变量。

***
# 2. C++的输入与输出
- `iostream`表示 `input output stream`是标准输入输出流的库
- `std::cin`是`stream`类的对象，用于输入窄字符
- `std::cout``stream`类的对象，用于输出窄字符
- `<<` `>>` 分别是流插入和流提取符
- `C++`的输⼊输出可以**自动识别变量类型**(本质是通过函数重载实现)
- `cout/cin/endl`等都属于`C++`标准库，`C++`标准库都放在一个叫`std(standard)`的命名空间中，所以要通过命名空间的使用方式去用他们
```
// 输入和输出
using namespace std;
int main()
{
	int x, y;
	char ch;
	cin >> x >> y >> ch;
	cout << x << " " << y << " " << ch;
}
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/2cc9b2c634034b578383d3aa06a7f84d.png)

***
# 3.缺省参数（默认参数）
- **缺省参数**是函数声明或定义时，给函数形参指定的一个默认值，如果调用该函数未传参，则使用这个缺省参数，若调用时传参了，则优先使用传递的实参
- 缺省参数分为全缺省和半缺省，全缺省是将所有形参都给一个默认值，而半缺省是给部分形参一个默认值，`C++`规定半缺省参数必须**从右往左依次连续缺省**，不能间隔跳跃给缺省值
- 带缺省值函数调用时，`C++`规定必须**从左往右依次传递实参**，不能间隔跳跃给实参
- 函数声明和定义分离时，缺省参数不能在函数声明和定义中同时出现，规定**必须函数声明给缺省值**
```
// 全缺省
int Add(int a = 10, int b = 20)
{
	return a + b;
}

// 半缺省 必须从从右往左依次连续缺省，不能间隔跳跃给缺省值
void Print(int a, int b = 10, int c = 20)
{
	cout << a << "   " << b << "   "<< c <<endl;
}

int main()
{
	// 如果调用该函数未传参，则使用这个缺省参数，若调用时传参了，则优先使用传递的实参
	int ret1 = Add();
	int ret2 = Add(1,2);
	cout << ret1<<"   " << ret2 << endl;
	// 半缺省调用 必须从左往右依次传递实参，不能间隔跳跃给实参
	Print(11);
	Print(11, 12);
	Print(11, 12, 13);

	return 0;
}
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/9218df9bb4ab4391a85577fe3db7f798.png)



```
// 函数声明定义分离情况,给声明缺省值，定义不要给
int Mul(int a = 1, int b = 6);

int main()
{
	return 0;
}
// 给缺省值了 C2572重定义默认参数
int Mul(int a = 1, int b = 6)
{
	return a * b;
}
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/5072c68e12124424b0fe9c921a3caf96.png)

将定义的缺省值去掉即可成功编译
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/4618bec34fa6459b9f20c90088dc598b.png)

***
# 4.缺省参数
 `C++`中允许有多个同名函数存在
 在同一作用域中,函数名相同，**形参**不同（类型、数量、顺序）与函数名无关
 ```
 // 函数重载：一词多义 C++中允许有多个同名函数存在
// 在同一作用域中 函数名相同 参数不同（类型、数量、顺序）与函数名无关

// 参数类型不同
void Swap(int* a, int* b)
{
	int tmp = *a;
	*a = *b;
	*b = tmp;
}
void Swap(double* x, double* y)
{
	double tmp = *x;
	*x = *y;
	*y = tmp;
}

// 参数数量不同
void Print(int a)
{
	cout << a << endl;
}
void Print(int x, int y)
{
	cout << x << "  " << y << endl;
}
// 参数顺序不同
void f(int a, double b)
{
	cout<<"f(int a, double b)" << endl;
}

void f(double b, int a)
{
	cout << "f(double b, int a)" << endl;
}

//// 返回值不能作为判断条件
//// C2556 只是在返回类型上不同
//void fun(){}
//int fun(){}

//// 这两个函数会产生调用歧义 编译器也不知道调用哪个函数
//// 将缺省值去掉后，给出具体实参即可顺利编译
//void f1()
//{
//	cout << "f()" << endl;
//}
//void f1(int a = 10)
//{
//	cout << "f(int a)" << endl;
//}

int main()
{
	int a = 10, b = 20;
	double x = 1.2, y = 2.4;
	cout << a << "  " << b << endl;
	cout << x << "  " << y << endl;
	Swap(&a, &b);
	Swap(&x, &y);
	cout << a << "  " << b << endl;
	cout << x << "  " << y << endl;

	Print(2);
	Print(1,2);
	f(1,1.2);
	f(2.1, 1);
	
	//// C2668 对重载函数的调用不明确
	//f1();
	//f1(20);
	return 0;
}
 ```

**重点**：为什么`C语言`不支持函数重载，而`C++`支持
C语言不支持函数重载，主要是因为它的编译器在链接阶段使用**函数名本身**来标识一个函数，函数名在C语言中必须是唯一的；而C++支持函数重载，是因为它采用了**名称修饰（Name Mangling）的机制（把参数类型带到函数名字中去），会在编译时将函数名、参数类型等信息编码到最终的符号名中，生成一个的函数标识，这样就能区分参数不同但函数名相同的多个函数**
```
// 演示C++函数名修饰规则
void print(int);
void print(double);
int main()
{
	// error LNK2019: 无法解析的外部符号 "void __cdecl print(int)" (?print@@YAXH@Z)，函数 main 中引用了该符号
	print(42);
	//  error LNK2019: 无法解析的外部符号 "void __cdecl print(double)" (?print@@YAXN@Z)，函数 main 中引用了该符号
	print(2.1);
	return 0;
}

```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/6fc29e8fcf2a4e0483b5b2fb9d0bd7aa.png)

注意观察：
`int`类型 ：`?print@@YAXH@Z`
`double`：   `?print@@YAXN@Z`
二者只有`H`和`N`的区别
***
# 5. 引用&
引用不是新定义一个变量，而是给已存在变量取了一个别名，编译器不会为引用变量开辟内存空间，它和它引用的变量共用同一块内存空间
比如说一个人，有自己的大名和小名，用哪个称呼都指代这个人，所以不会开辟新的内存空间
`类型& 引用名 = 引用对象`
使用引用时有如下规则
- 引用在定义时必须初始化
- 一旦确认了一个引用实体，就不能再引用其他实体了
- 一个变量可以有多个引用
```

// 引用&
using namespace std;
int main()
{
	int a = 1;
	// 引用在定义时必须初始化  error C2530: “nq”: 必须初始化引用
	//int& nq;
	// 一个变量可以有多个引用
	int& na = a;
	int& nb = a;
	int& nc = na;

	int num = 20;
	// 赋值
	na = num;

	// 一旦确认了一个引用实体后，就不能再引用其他实体
	int b = 2;
	int& np = b;
	np = na; // 这里是赋值，而不是改变引用指向
	
	return 0;
}

```
![!\[\[Pasted image 20250628165555.png\]\]](https://i-blog.csdnimg.cn/direct/624f11c3b2814f0abd57a4c0eca6801f.png)

这里`a` `na` `nb` `nc` 是同一个值
而改变`na`的值会发生什么？
![!\[\[Pasted image 20250628165613.png\]\]](https://i-blog.csdnimg.cn/direct/448992691d34438ea3ab0135b44527a8.png)

全都变成20
我们再看看地址，所有值的地址都是一样的，也证明了在语法上引用不开辟新的空间，而是变量的别名
## 5.1. 引用的使用
- 引用在实践中主要用于**引用传参**和**引用做返回值**从而**减少拷贝提高效率**和改变引用对象时同时改变被引用对象。
- 引用传参和指针传参功能类似，而引用传参相对更方便一些
```
// const修饰引用
using namespace std;
int main()
{
	const int a = 10;
	// 不可以 a只读 ra可读可写 权限放大
	// int& ra = a;

	// 可以 a和pa都是只读 权限平移
	const int& pa = a;

	// 可以 b可读可写 rb只读 权限缩小
	int b = 10;
	const int& rb = b;

}
```
而**表达式运算**（必须要有一个值来存结果）和**类型转换**会产生临时变量，临时变量具有常性
```
// 类型转换和表达式运算会产生临时变量 临时变量具有常性
using namespace std;
int main()
{
	int a = 20;
	int c = 10;
	double b = a;
	// E0434 非常量限定
	// int& rb = b;
	const int& rb = b;

	// E0461 非常量引用的初始值必须为左值
	// int& plus = a + c;
	const int& plus = a + c;// a c固定，而a+c必须有一个临时变量来存储结果

	return 0;
}
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/84e91fb16eef4fe0b3e883cdd592bc0e.png)
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/dcbc3a5a2bfa4a1f821c09d102e8ec57.png)
## 5.2. 指针和引用的区别
- 在语法上，引用不开辟新的内存空间，而指针开辟新的内存空间，但是在底层，二者没有区别
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/30f38d2ad90845d98b9d05652a268fef.png)

转到汇编，我们发现二者都是开辟了新的空间，将值存在`rax`中
以下是引用和指针的对比：
- 引用在定义时必须初始化，而指针没有强制要求
- 引用在引用了一个确定对象之后，就不能改变指向了，而指针可以
- 引用可以直接访问对象，指针必须解引用之后才能访问
- `sizeof(引用类型)`大小就是引用类型大小，`sizeof(指针类型)`，大小取决于环境，`64位`不论类型，指针大小均为`8`，而`32位`不论类型，指针大小均为`4`；`引用对象++`，是该`对象值+1`，`指针对象++`，是该`对象值+（类型大小）`
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/bd2bf487589c437db289c89d7edda93a.png)
- 一般来说没有空引用的概念，而有空指针
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/440231b6e8ec4a3881486748b69f7efd.png)
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/c7266b135edb4bdc9c6e2206499fdfad.png)
# 6. 内联函数
- 用`inline`修饰的函数叫内联函数，编译时会在调用的地方直接展开内联函数，并不会像传统函数一样建立栈帧，这样可以提高效率
- `inline`对于编译器只是一个**建议**，取决于不同编译器。`inline`适合频繁调用的代码量小的函数，对于递归函数和代码量大的函数，加上`inline`也会被编译器忽略
- `inline`不建议声明和定义分离到两个文件中，分离会导致链接错误。`inline`被展开，就没有函数地址了，链接时找函数会出现报错
想要实现内联函数，需要进行调整
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/94c124aeb4174c2db6223b0ed8634840.png)

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/faae9defa71c4d218e145cb28c631669.png)
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/df5ae63074ba472c91fad35179fa957c.png)


```
// 内联函数
using namespace std;
inline int Add(int a = 10, int b = 30)
{
	return a + b;
}
int main()
{
	// 可以通过汇编观察程序是否展开
	// 有call Add语句就是没有展开，没有就是展开了
	int a = 0, b = 3;
	Add(a, b);

	return 0;
}
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/3d57c95ee14647c49c523289dd5f73fc.png)
# 7. `nullptr`
`NULL`实际是⼀个宏，在传统的`C`头⽂件`(stddef.h)`中，可以看到如下代码：
```
#ifndef NULL
	#ifdef __cplusplus
		#define NULL 0
	#else
		#define NULL ((void *)0)
	#endif
#endif
```
在`C++`中，`nullptr`是空指针，可以理解为`(void*)0`而`NULL`是个宏，代表`0`