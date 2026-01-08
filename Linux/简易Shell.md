C++/C混编

```bash
myshell:myshell.cc
    g++ -o $@ $^
.PHONY:clean
clean:
	rm myshell
```

`shell/bash`就是一个进程

> 命令行提示符--->

> 获取命令行--->

> 解析命令行--->

> 执行命令行--->



## 打印命令行提示符

思考，获得用户名，主机名，路径---->`PATH`中获得

`HOSTNAME` `USER` `PWD`

`getenv`拿

拿到`NULL`要检查一下->`None`

构建命令行，怎么做？

![image-20260104215405459](C:/Users/hp/AppData/Roaming/Typora/typora-user-images/image-20260104215405459.png)

搞个缓冲区，固定1024大小

`snprintf`->缓冲区、大小、输出方式（和`printf`没区别了）

## 获取命令行

要获取包含空格的字符串--->`fgets`--->缓冲区，大小，获取源

 ![image-20260104220853939](C:/Users/hp/AppData/Roaming/Typora/typora-user-images/image-20260104220853939.png)

## 解析命令行

`"ls -a -l -n"` ----->`"ls" "-a" "-l" "-n" NULL`

拆成`argc/argv`的样子

用什么接口？

`char* strtok`---->目标字符串，分隔符

首次使用：`strtok(str," ");`str是要切的字符串，第一次必须传，后续要继续切割，就不用穿这个字符串了

后续使用：`strtok(nullptr," ");`

返回有效字符地址or`nullptr`

切了之后放到`argv`里

## 执行命令

程序替换！`execvp`

只能让子进程执行！！！

父进程阻塞等待







## 解决bug问题

`cd`无效

每一个进程都有当前路径的概念

改`pwd`，改谁的？改shell的，这样每个子进程都能继承shell的环境表

而改子进程的路径，子进程退出了就没了



所以，有些命令必须子进程执行，而有些必须shell自己执行（本质：**shell调用自己的函数**）--->**内建命令**

先检测内建命令，可以枚举内建命令（`strcmp(gargv[0],"命令") == 0`）

发现执行了内建命令之后，命令行提示符没变--->环境变量没变！环境变量要维护！所以现在不建议从环境变量获取了！！！直接从系统里获取同时更新环境变量，`getcwd`

修改环境变量`putenv`

我们自己的shell天然继承了系统的环境变量表--->所以我们的shell可以自己维护自己的全局的表

所以：

`echo`--->能打印全局资源 `echo $?`上一个退出的进程的退出码

`export`--->调整环境变量，调整的一定是父进程的，子进程的结束就没了

我们自己shell维护的自己的环境表，直接从全局来--->`char** environ`

`env` 打印所有环境变量 所以也是内建命令



# shell重定向的实现

"ls -a -b -c">log.txt

"ls -a -b -c">>log.txt

"ls -a -b -c" > log.txt

"ls -a -b -c">    log.txt

有多种格式

## 定义全局变量

```cpp
#define NoneRedir 0
#define InputRefir 1
#define 

int gredir = NoneRedir;
char* filename = nullptr;
```

## 过滤空格操作

#
