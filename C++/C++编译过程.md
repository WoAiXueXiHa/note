# 1. C++如何从代码到可执行文件

## 1.0. 一小段代码进行演示

```cpp
// add.h
#pragma once 

template <class T>
T add(T a, T b){
  return a + b;
}

```

```cpp
// foo.cpp
#include "add.h"

int foo(){
  return add(10,30);        // add<int>实例化
}
```

```cpp
// main.cpp
#include "add.h"
#include <iostream>
using namespace std;

int foo();

int main(){
  cout<<add(1,2)<<endl;
  cout<<foo()<<endl;

  return 0;
}
```

## 1.1. 预处理阶段: `g++ -E`

预处理阶段编译器只做**文本级**工作：

* 展开`#include`：把头文件的内容拷贝进来
* 宏替换`#define`
* 条件编译`#if/#ifdef`
* 去掉注释

> 注意：**模板实例化不在预处理阶段**，预处理器不懂C++语义，只做文本级处理

**命令演示**

```bash
[vect@VM-0-11-centos ~]$ g++ -E main.cpp -o main.i
[vect@VM-0-11-centos ~]$ g++ -E foo.cpp -o foo.i
```

![image-20251214154255848](https://gitee.com/binary-whispers/pic/raw/master///20251214154541679.png)

> `-E`:只执行预处理操作，预处理结束就停止
>
> `-o`：指定输出文件名，后面紧跟文件名
>
> `filename.i`：预处理后的源文件后缀为`.i`
>
> 如果不加` -o main.i` 预处理过后的文件会输出到终端

## 1.2.编译阶段：`g++ -S`

**把预处理后的`.i`文件变成汇编文件`.s`**：

* **词法/语法分析：**把字符流变成token流（我们写的代码对于机器来说就是一串字符，编译器会把这串字符组合成有意义的”单词“即token流）；判断token流的排列是否符合C++语法并构建AST树(抽象语法树)

  > * 写的一行代码：`int a = b + 3;` 在计算机眼里就是一串字符：`i n t a = b + 3 ;`,现在把这些字符组成有意义的`token`，具体包括了关键字、标识符、运算符、字面量、语句结束符，这个阶段只关心”单词的构建“，不关系语法是否正确
  >
  > * 现在已经形成token流`[int] [a] [=] [b] [+] [3] [;]`，语法分析的结果不是对与错，而是构建AST树（这句代码真正的结构含义）
  >
  > ```css
  >         =
  >        / \
  >       a   +
  >          / \
  >         b   3
  > 
  > ```
  >
  > 然后补充上类型信息：
  >
  > ```css
  > 声明语句
  >  ├── 类型: int
  >  └── 赋值
  >      ├── 变量: a
  >      └── 加法
  >          ├── 变量: b
  >          └── 常量: 3
  > 
  > ```

  这里可以类比：词法分析-认识单词  语法分析-分析句子主谓宾 AST-句子的语法树状图

* **语义分析：**类型检查、重载检查、名字查找、访问控制、模板相关规则
* **模板实例化：**当编译器看到”需要用到的模板“时，会生成具体版本的函数体，例如：`add(1,2)--->add<int>(int,int)`
* **优化：**基于AST树，修改AST树，常量折叠(直接进行运算，不留到运行期`int x = 5 + 3 -> int x = 8`)、内联（直接替换函数调用，在这里展开函数，内联只是建议）、死代码删除（删除永远不会执行的代码）、寄存器分配...
* **生成汇编：**输出`.s`

**命令演示：**

```bash
[vect@VM-0-11-centos ~]$ g++ -S main.i -o main.s
[vect@VM-0-11-centos ~]$ g++ -S foo.i -o foo.s
```

![image-20251214162632977](https://gitee.com/binary-whispers/pic/raw/master///20251214162636085.png)

可以观察到，此时已经形成了汇编代码

## 1.3. 汇编阶段：`g++ -c`

汇编把`.s`汇编代码变成机器码目标文件（二进制文件）`.o`，二进制文件包含：

* `.text`段：机器指令

* `.rodata`：只读常量

* `.data/.bss`：全局/静态数据

* **符号表：**目标文件中定义了哪些符号、还需要外部提供哪些符号

  > 符号：函数名、全局变量名、静态变量名
  >
  > 符号表：**目标文件里的一张”名字->状态/地址“**的表，回答定义了哪些符号，使用了哪些符号但是还不知道地址，举个例子理解一下：
  >
  > ```cpp
  > // show_signal.cpp
  > int foo(){return 23;}
  > ```
  >
  > 编译：
  >
  > ```bash
  > [vect@VM-0-11-centos ~]$ g++ -c show_signal.cpp -o foo.o
  > [vect@VM-0-11-centos ~]$ nm -C foo.o	# 查看符号表
  >                  U __cxa_atexit
  >                  U __dso_handle
  > 0000000000000048 t _GLOBAL__sub_I__Z3foov
  > 0000000000000000 T foo()
  > 000000000000000b t __static_initialization_and_destruction_0(int, int)
  >                  U std::ios_base::Init::Init()
  >                  U std::ios_base::Init::~Init()
  > 0000000000000000 b std::__ioinit
  > 
  > ```
  >
  > `0000000000000000 T foo()` `偏移地址 在.text段中定义 符号名`
  >
  > 说明了在`foo.o`里面，**自己定义了foo**
  >
  > 再看需要外部提供符号的情况：
  >
  > ```cpp
  > int foo(); // 声明未定义
  > 
  > int main(){
  >   return foo();
  > }
  > ```
  >
  > 编译：
  >
  > ```bash
  > [vect@VM-0-11-centos ~]$ g++ -c main_signal.cpp -o main.o
  > [vect@VM-0-11-centos ~]$ nm -C main.o
  >                  U __cxa_atexit
  >                  U __dso_handle
  > 0000000000000048 t _GLOBAL__sub_I_main
  > 0000000000000000 T main
  >                  U foo()
  > 000000000000000b t __static_initialization_and_destruction_0(int, int)
  >                  U std::ios_base::Init::Init()
  >                  U std::ios_base::Init::~Init()
  > 0000000000000000 b std::__ioinit
  > 
  > ```
  >
  > 看这段代码：
  >
  > ```bash
  > 0000000000000000 T main
  >                  U foo()
  > ```
  >
  > **U = undefined**未定义，这里说明在`mian.o`中用到了`foo()`，但是在`mian.o`中没有实现，需要外部提供的符号！
  >
  > 所以**符号表的作用：**当链接器拿到`main.o`发现需要`foo`，而`foo.o`定义了`foo`，则指向`foo.o`里的`foo`地址

* **重定位信息：**哪些地址等链接时再决定

  >1. **一个残酷的事实：在`.o`文件中，所有地址都是临时的！！！**因为`.o`不知道将来和谁链接，不知道程序从内存哪里开始，所以编译器只能做到：”**将来这里要用一个地址，我先占个坑**“
  >
  >2. 举个例子：
  >
  >   ```cpp
  >   int foo(); // 声明未定义
  >   
  >   int main(){
  >     return foo();
  >   }
  >   ```
  >
  >   在汇编层面：` call  _Z3foov` 这里`foo`被改名为`_Z3foov`，这里可以补充一个知识点：为什么C++支持函数重载？**在汇编和链接层面，名字必须唯一，C++函数会进行函数名改编，把函数名编进符号表里**
  >
  >   ```css
  >   _Z  3  foo  v
  >   │   │   │    │
  >   │   │   │    └── 参数列表：v = void（无参数）
  >   │   │   └─────── 函数名 foo
  >   │   └─────────── 3 表示 foo 这个名字长度是 3
  >   └─────────────── _Z = C++ 符号前缀
  >   ```
  >
  >   ```cpp
  >   void foo();        // _Z3foov
  >   void foo(int);     // _Z3fooi
  >   void foo(double);  // _Z3food
  >   ```
  >
  >   常见的类型编码：
  >
  >   | 类型   | 编码 |
  >   | ------ | ---- |
  >   | void   | v    |
  >   | int    | i    |
  >   | double | d    |
  >   | char   | c    |
  >   | long   | l    |
  >   | 指针   | P    |
  >
  >   这里我们也可以用`nm main_signal.o`来查看
  >
  >3. 编译器如何解决这个问题？
  >
  >   **生成机器码+留一个备注（重定位信息）**
  >
  >4. **重定位表是啥：一张需要补地址的清单**

总结一下，在汇编阶段编译器的行为：**收集所有符号表->给所有符号分配最终地址->处理重定位的信息** 而符号表表达了**”谁是谁“**，重定位信息表达了**”地址填哪“**

## 1.4. 链接阶段：`g++ main.o foo.o -o app`

编译器把多个`.o`文件（包括库文件`.a/.so`）拼成一个可执行文件或共享库：

* **符号解析：**把`main.o`里**未定义的符号**去别的`.o/.a/.so`里找定义
* **地址分配和段合并：**把各个`.text/.data`合并，给每个符号分配最终地址
* **重定位：**把机器码/数据中”占位“的地址改成最终地址
* **处理库依赖：**
  * 静态库`.a`：把需要的目标文件成员抽取进行最终程序
  * 动态库`.so`：记录依赖关系，运行时由动态装载器加载

**命令演示**

```bash
[vect@VM-0-11-centos link]$ g++ main.o foo.o -o app
[vect@VM-0-11-centos link]$ ./app
[vect@VM-0-11-centos link]$ ldd ./app	# 查看依赖的动态库
	linux-vdso.so.1 =>  (0x00007ffd751fe000)
	libstdc++.so.6 => /home/vect/.VimForCpp/vim/bundle/YCM.so/el7.x86_64/libstdc++.so.6 (0x00007ff731b8b000)
	libm.so.6 => /lib64/libm.so.6 (0x00007ff731889000)
	libgcc_s.so.1 => /lib64/libgcc_s.so.1 (0x00007ff731673000)
	libc.so.6 => /lib64/libc.so.6 (0x00007ff7312a5000)
	/lib64/ld-linux-x86-64.so.2 (0x00007ff731f0c000)
```

## 1.5. 把模板定义放到`.cpp`会发生什么？

`add.h`只写声明

```cpp
// add.h
#pragma once

template <typename T>
T add(T a, T b);   // 只有声明，没有定义
```

`add.cpp`定义模板

```cpp
// add.cpp
#include "add.h"

template <typename T>
T add(T a, T b) {
    return a + b;
}
```

```cpp
// main.cpp
#include <iostream>
#include "add.h"

int main() {
    std::cout << add(1, 2) << std::endl;
    return 0;
}
```

**编译每个cpp都成功了：**

```bash
[vect@VM-0-11-centos template]$ g++ -c add.cpp -o add.o
[vect@VM-0-11-centos template]$ g++ -c main.cpp -o main.o
```

这里已经埋雷了

**链接出错：**

```bash
[vect@VM-0-11-centos template]$ g++ add.o main.o -o app
main.o: In function `main':
main.cpp:(.text+0xf): undefined reference to `int add<int>(int, int)'
collect2: error: ld returned 1 exit status

```

**为什么会出错？**

> 1. 编译`main.cpp`发生了什么？
>
>    ​    `add(1,2);`
>
>    * 编译器知道这是个模板
>
>    * 需要生成`add<int>`函数体
>
>    * 但在`add.h`里只看到声明，没有定义
>
>      于是编译器只能假设将来有人实现`add<int>`
>
>      在`mian.o`里：符号表是`U int add<int>(int, int)`
>
> 2. 编译`add.cpp`发生了什么？
>
>    ```bash
>    [vect@VM-0-11-centos template]$ nm -C add.o
>    [vect@VM-0-11-centos template]$ 
>    ```
>
>    符号表是空的！！！
>
>    在`add.cpp`里**没有任何地方用到`add<int>`**，编译器遵循**模板哪里使用哪里实例化**的原则
>
> 3. 链接发生了什么？
>
>    | 文件   | 情况                |
>    | ------ | ------------------- |
>    | main.o | 我需要 `add<int>`   |
>    | add.o  | 我没定义 `add<int>` |
>
>    现在没人提供这个符号！！！
>
>    报错：`undefined reference to int add<int>(int, int)`

**本质说明了：模板的实例化发生在编译期，而不是链接期**

所以，怎么解决？

* 模板定义放在头文件
* 在`add.cpp`文件中显式实例化

# 2. 动态库和静态库

> 动态库：用的时候，**程序只记住去哪里找**，真正运行时再加载
>
> 静态库：用的时候，**把代码直接拷贝到程序里**

我们还是用`add`这份代码，不要模板

## 2.1. 动态库

Linux下：`.so`为后缀的文件，本质是**独立存在的二进制文件**

**怎么生成？**

> * 生成位置无关代码：`g++ -fPIC add.cpp -o add.o` `-fPIC`告诉编译器这段代码将来被共享
>
> * 生成动态库：`g++ -shared add.o -o libadd.so` 生成了动态库：`libadd.so`
>
> * 动态库链接：`g++ main.cpp -L. -ladd -o app_dynamic`
>
>   `-L.`:在当前目录找库
>
>   `-ladd`：优先找`libadd.so`
>
> * 验证依赖： `ldd app_dynamic`

完整代码：

```bash
[vect@VM-0-11-centos rep]$ g++ -fPIC -c add.cpp -o add.o
[vect@VM-0-11-centos rep]$ g++ -shared add.o -o libadd.so
[vect@VM-0-11-centos rep]$ g++ main.cpp -L. -ladd -o app_dynamic
[vect@VM-0-11-centos rep]$ g++ main.cpp -L. -ladd -o app_dynamic
[vect@VM-0-11-centos rep]$ ./app_dynamic
3
[vect@VM-0-11-centos rep]$ ldd app_dynamic
	linux-vdso.so.1 =>  (0x00007ffe6c5bd000)
	libadd.so (0x00007fdfcef90000)
	libstdc++.so.6 => /home/vect/.VimForCpp/vim/bundle/YCM.so/el7.x86_64/libstdc++.so.6 (0x00007fdfcec0f000)
	libm.so.6 => /lib64/libm.so.6 (0x00007fdfce90d000)
	libgcc_s.so.1 => /lib64/libgcc_s.so.1 (0x00007fdfce6f7000)
	libc.so.6 => /lib64/libc.so.6 (0x00007fdfce329000)
	/lib64/ld-linux-x86-64.so.2 (0x00007fdfcf192000)
```

**动态库：编译时只记住地址，运行时再加载代码**

## 2.2. 静态库

Linux下：`.a`为后缀的文件，本质是**一堆`.o`文件的打包**

**怎么生成？**

> * 编译成目标文件：`g++ -c add.cpp -o add.o` 
>
>   此时`add.o`里面有**add的机器码**，还没有生成程序
>
> * 打包成静态库：`ar rcs libadd.a add.o`
>
>   现在有静态库：`libadd.a`
>
> * 链接生成可执行程序：`g++ main.cpp libadd.a -o app_static`
>
>   `main.cpp`用了`add`，`libadd.a`里刚好有`add.o`，**直接把`add.o`复制进最终程序**
>
> * 查看依赖：`ldd app_static`
>
>   会发现没有`libadd.a`,其实代码已经拷贝到程序里了

完整代码：

```bash
[vect@VM-0-11-centos rep]$ g++ -c add.cpp -o add.o
[vect@VM-0-11-centos rep]$ ar rcs libadd.a add.o
[vect@VM-0-11-centos rep]$ g++ main.cpp libadd.a -o app_static
[vect@VM-0-11-centos rep]$ ./app_static
3
[vect@VM-0-11-centos rep]$ ldd app_static
	linux-vdso.so.1 =>  (0x00007ffd18faa000)
	libstdc++.so.6 => /home/vect/.VimForCpp/vim/bundle/YCM.so/el7.x86_64/libstdc++.so.6 (0x00007f51a6245000)
	libm.so.6 => /lib64/libm.so.6 (0x00007f51a5f43000)
	libgcc_s.so.1 => /lib64/libgcc_s.so.1 (0x00007f51a5d2d000)
	libc.so.6 => /lib64/libc.so.6 (0x00007f51a595f000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f51a65c6000)

```

**静态库：编译时把代码拷贝到而可执行文件**

## 2.3. 二者对比

| 对比点     | 静态库     | 动态库           |
| ---------- | ---------- | ---------------- |
| add 的代码 | 拷进 app   | 在 libadd.so     |
| 可执行文件 | 大         | 小               |
| ldd        | 看不到 add | 能看到 libadd.so |
| 运行依赖   | 无         | 必须有 so        |



# 3. 总结

![image-20251214184053188](https://gitee.com/binary-whispers/pic/raw/master///20251214184056702.png)
