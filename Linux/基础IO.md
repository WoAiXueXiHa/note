# 0. 铺垫

文件=文件内容+文件属性

文件**存储在磁盘**，访问一个文件，先要从磁盘加载到内存中，这是打开文件的过程

加载到内存中的文件，需要管理起来---->先描述，再组织---->所以一定有一个数据结构在管理着内存中的文件

那么现在就可以知道，**内核中的文件=文件内容+文件内核数据结构**

加载到内存中的文件，是谁在访问？--->**进程**，CPU调度进程，进程访问文件

所以，打开的文件，本质是研究**进程和文件的关系**

而对于这段代码：

```cpp
// 上文
FILE* fp = fopen("./log.txt","rw");
// 下文
```

只有执行到第二行，才开始打开文件，文件类的函数都是运行时操作

现在就有一个宏观的认识：

![image-20260106164045589](https://gitee.com/binary-whispers/pic/raw/master///20260106164048492.png)

本文主要探讨内存中的文件

****

# 1.回顾C文件接口

## 打开文件

```bash
vect@VM-0-11-centos file]$ gcc -o hello hello.c 
[vect@VM-0-11-centos file]$ ./hello
[vect@VM-0-11-centos file]$ ls
hello  hello.c  myfile
[vect@VM-0-11-centos file]$ cat hello.c
#include <stdio.h>

int main(){
  FILE* fp = fopen("myfile","w");
  if(!fp){
    perror("fp err");
    return 1;
  }
  while(1);
  fclose(fp);
  return 0;
}
```

`myfile`和`hello.c`在同一目录下

`ls /proc/[pid] -l`查看当前正在运行的进程信息：

![image-20260106165147366](https://gitee.com/binary-whispers/pic/raw/master///20260106165149553.png)

复习：

> **cwd：** 指向当前工作路径->当前进程运行目录
>
> **exe：** 指向启动当前进程的可执行文件完整路径

**打开文件，本质是进程打开，进程知道自己在哪里，文件不带路径，OS也知道创建的文件放到哪里**

## 写文件

```cpp
#include <stdio.h>
#include <string.h>
int main()
{
    FILE *fp = fopen("myfile", "w");
    if(!fp){
    	printf("fopen error!\n");
	}
    const char *msg = "hello file!\n";
    
    int count = 5;
    while(count--){
    	fwrite(msg, strlen(msg), 1, fp);
    }
    fclose(fp);
    
    return 0;
}

```

## 读文件

```cpp
#include <stdio.h>
#include <string.h>

int main(){
  FILE* fp = fopen("myfile","r");
  if(!fp){
    perror("fp err");
    return 1;
  }

  char buffer[1024];
  const char* msg = "hello file\n";
  while(1){
    size_t s = fread(buffer,1,strlen(msg),fp);
    if(s > 0){
      buffer[s] = 0;
      printf("%s",buffer);
    }
  }
  fclose(fp);
  return 0;

}

```

## 总结

```cpp
#include <stdio.h>
// 打开文件 参数：文件路径 打开方式
FILE *fopen(const char *path, const char *mode);

// 读文件
// 参数1：指向缓冲区的指针，将读取的内容存入缓冲区
// 参数2：单个数据元素字节大小
// 参数3： 读取元素个数
// 参数4：读取的目标文件流
size_t fread(void *ptr, size_t size, size_t nmemb, FILE *stream);

// 写文件
// 参数1：指向要写入文件的缓冲区
// 参数2：单个数据元素字节大小
// 参数3：写入元素个数
// 参数4：写入的目标文件流
size_t fwrite(const void *ptr, size_t size, size_t nmemb,FILE *stream);
```

**文件打开的方式**

![image-20260106172414494](https://gitee.com/binary-whispers/pic/raw/master///20260106172416698.png)



****

# 2. 系统文件IO

## 标志位

对于`r/w/a`语言层面的打开方式，Linux底层设计了一些宏（只有一个比特位为1的值）作为位图传递

> * `O_RDONLY`：只读
> * `O_WRONLY`：只写
> * `O_RDWR`：读写
> * `O_CREAT`：创建文件
> * `O_APPEND`：追加
> * `O_TRUNC`：清空

用一段代码模拟标志位的传递

```cpp
#include <stdio.h>

#define ONE   0001    // 0000 0001
#define TWO   0002    // 0000 0010
#define THREE 0004    // 0000 0100

void func(int flags){
  if(flags & ONE) printf("ONE\n");
  if(flags & TWO) printf("TWO\n");
  if(flags & THREE) printf("THREE\n");
  printf("-----------\n");
}

int main(){
  func(ONE);
  func(ONE | TWO);
  func(ONE | TWO | THREE);

  return 0;
}
```

## 写文件

```cpp
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <string.h>

int main(){
  int fd = open("log.txt",O_CREAT | O_WRONLY, 0666);  // fd文件描述符
  if(fd < 0){
    perror("open err");
    return 1;
  }

  int cnt = 5;
  const char* msg = "hello file\n";
  while(cnt--){
    write(fd,msg,strlen(msg));
  }

  close(fd);

  return 0;
}

```

`fd`后续讲解

## 读文件

```cpp
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <string.h>

int main(){
  int fd = open("log.txt",O_CREAT | O_WRONLY, 0666);  // fd文件描述符
  if(fd < 0){
    perror("open err");
    return 1;
  }

  const char* msg = "hello file\n";
  char buffer[1024];
  while(1){
    ssize_t s = read(fd,buffer,strlen(msg));
    if(s > 0){
      printf("%s",buffer);
    }else break;
  }

  close(fd);

  return 0;
}

```



## 接口总结

> * `open`
>
>   ```cpp
>   #include <sys/types.h>
>   #include <sys/stat.h>
>   #include <fcntl.h>
>   
>   // 打开存在的文件
>   int open(const char* pathname, int flags);
>   // 打开不存在的文件
>   int open(const char* pathname, int flags, mode_t mode);
>   
>   // 参数：
>   // pathname: 打开或创建的文件
>   // flags： 打开文件方式 传标志位
>   // mode: 初始文件权限，最终权限=初始文件权限&(~umask)
>   
>   // 返回值： 成功返回新打开文件的描述符 失败返回-1
>   ```
>
> * `write`
>
>   ```cpp
>   #include <unistd.h>
>   ssize_t write(int fd, const void* buf, sizet sount);
>   
>   // 参数：
>   // fd: 文件描述符->指定唯一文件
>   // buf: 缓冲区指针，写入的内容来自这个缓冲区
>   // count: 写入元素总数
>   
>   // 返回值：返回成功写入的n个字节数  没写入返回0 失败返回-1
>   
>   ```
>
> * `read`
>
>   
>
>   ```cpp
>   #include <unistd.h>
>   ssize_t read(int fd, void* buf, size_t count);
>   
>   // 参数：
>   // fd：文件描述符->指定唯一文件
>   // buf: 缓冲区指针，读到的内容放到这个缓冲区
>   // count： 读到元素总数
>   
>   // 返回值：返回成功读取的n个字节数 到达文件末尾，没有数据可读返回0 失败返回-1
>   ```
>
> * `close`
>
>   ```cpp
>   #include <unistd.h>
>   int close(int fd);
>   ```

C语言各种文件操作的函数就是封装了系统文件IO的函数

****

# 3. 文件描述符

## 0 1 2

先看一段代码：

```cpp
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>

int main(){
  int fd1 = open("file1.txt",O_RDONLY | O_CREAT, 0666);
  int fd2 = open("file2.txt",O_RDONLY | O_CREAT, 0666);
  int fd3 = open("file3.txt",O_RDONLY | O_CREAT, 0666);
  int fd4 = open("file4.txt",O_RDONLY | O_CREAT, 0666);

  printf("fd:%d\n",fd1);
  printf("fd:%d\n",fd2);
  printf("fd:%d\n",fd3);
  printf("fd:%d\n",fd4);

  close(fd1);
  close(fd2);
  close(fd3);
  close(fd4);
  
  return 0;

}

```

输出结果：

```bash
[vect@VM-0-11-centos file]$ ./fd
fd:3
fd:4
fd:5
fd:6
```

解释：

> 进程运行，默认会打开三个文件流：`stdin stdout stderr`分别代表键盘文件、显示器文件、显示器文件
>
> 而对应的`fd`分别为0 1 2
>
> ![image-20260106193931151](https://gitee.com/binary-whispers/pic/raw/master///20260106193933971.png)

文件描述符本质是数组索引，从0开始

打开文件时，OS在内存中创建相应的数据结构描述文件->file结构体，表示一个已经打开的文件对象

进程执行`open`系统调用，必须让进程和文件关联起来->每个进程都一个指针`*files`，指向`files_struct`，这张表包含一个指针数组，每个元素都是指向对应打开文件的指针！

只要有文件描述符，就能找到对应文件

## 文件描述符分配规则

```cpp
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>

int main(){
  close(0);
  int fd1 = open("file1.txt",O_RDONLY | O_CREAT, 0666);
  int fd2 = open("file2.txt",O_RDONLY | O_CREAT, 0666);
  int fd3 = open("file3.txt",O_RDONLY | O_CREAT, 0666);
  int fd4 = open("file4.txt",O_RDONLY | O_CREAT, 0666);

  printf("fd:%d\n",fd1);
  printf("fd:%d\n",fd2);
  printf("fd:%d\n",fd3);
  printf("fd:%d\n",fd4);

  close(fd1);
  close(fd2);
  close(fd3);
  close(fd4);
  
  return 0;

}

```

关闭0，1和2打开，输出结果：

```bash
fd:0
fd:3
fd:4
fd:5
```

 很明显：**文件描述符从最小的未被使用的索引开始，作为新文件描述符，以后依次**

## 重定向

关掉1呢？

```cpp
#include <unistd.h>
#include <sys/types.h>
#include <fcntl.h>
#include <stdio.h>
#include <sys/stat.h>
#include <stdlib.h>

int main(){
  close(1);
  int fd = open("mychdir",O_WRONLY | O_CREAT,00666);
  if(fd < 0){
    perror("open err");
    return 1;
  }
  printf("fd:%d\n",fd);
  fflush(stdout);

  close(fd);

  return 0;
}

```

输出：

```bash
[vect@VM-0-11-centos file]$ gcc -o chdir chdir.c 
[vect@VM-0-11-centos file]$ ./chdir 
[vect@VM-0-11-centos file]$ cat mychdir 
fd:1

```

本来应该打印到显示器上的内容，打印到了`mychdir`文件中！重定向了！！！

![image-20260106195524592](https://gitee.com/binary-whispers/pic/raw/master///20260106195528510.png)

这是重定向的本质

但是通过`close`重定向不够优雅

## `dup2`系统调用重定向

```cpp
#include <unistd.h>
int dup2(int oldfd, int newfd);
// dup()2 makes newfd be the copy of oldfd,close newfd first if necessary
```

现在有`fd`和`1`，要将`1`输出重定向到`fd`，参数怎么填？

![image-20260106200733063](https://gitee.com/binary-whispers/pic/raw/master///20260106200734903.png)

```cpp
#include <unistd.h>
#include <sys/types.h>
#include <fcntl.h>
#include <stdio.h>
#include <sys/stat.h>
#include <stdlib.h>
#include <string.h>

int main(){
  int fd = open("mydup2",O_CREAT | O_RDWR | O_APPEND,0666);
  if(fd < 0){
    perror("open err");
    return 1;
  }
  const char* msg = "hello\n";
  dup2(fd,1);
  int cnt = 3;
  while(cnt--){
    write(1,msg,strlen(msg));
  }

  close(fd);
  return 0;
}

```

输出：

```bash
[vect@VM-0-11-centos file]$ gcc -o chdir chdir.c 
[vect@VM-0-11-centos file]$ ./chdir 
[vect@VM-0-11-centos file]$ cat mydup2 
hello
hello
hello

```

****

# 4. 理解“一切皆文件”

* **Linux将所有的系统资源（硬件、进程、网络、管道等）都抽象成文件的形态，为它们提供统一的操作接口（`open`/`read`/`write`/`close`等）**
* 在Linux视角，“文件”不是传统的“存储在磁盘的字节序列”，而是**可交互的资源对象**，具备两个特征：
  * 有唯一标识符：要么是文件路径（如`/dev/sda` 、`/proc/1234`），要么是文件描述符
  * 有统一的操作方法：都能通过系统调用`open`、`read`、`write`等操作交互
* 核心优势：**统一接口便于用户操作，不用关心底层实现细节**

****

# 6. 缓冲区

## 本质价值

**减少低俗IO操作次数**

* 用户态缓冲：减少「用户态 ↔ 内核态」的系统调用切换（切换开销大）
* 内核态缓冲：减少「内核 ↔ 磁盘 / 硬件」的物理 IO 操作（磁盘是毫秒级，内存是纳秒级）

## 用户态缓冲区 C语言视角

C标准库为`FILE*`类型文件他提供了**用户态内存缓冲区**毕比如`printf`、`fread`等都会用到，属于应用程序自己的内存空间，内核感知不到

### 缓冲区类型

| 缓存类型 | 场景                         | 刷新触发条件                                |
| -------- | ---------------------------- | ------------------------------------------- |
| 行缓存   | 终端（屏幕、键盘、`stdout`） | 遇到`\n`、缓冲区满、遇到`fflush`、程序退出  |
| 全缓冲   | 普通文件（磁盘文件）         | 缓冲区满（默认4KB/8KB）、`fflush`、`fclose` |
| 无缓冲   | `stderr`                     | 立即输出                                    |

#### 行缓存演示

```cpp
#include <stdio.h>
#include <unistd.h>

int main(){
  // 1. 无\n 数据保留在缓冲区 不输出
  printf("无换行的输出");
  sleep(3);

  // 2. 手动fflush刷新缓冲区 屏幕立即输出
  fflush(stdout);
  sleep(3);

  // 3. 加\n 自动刷新缓冲区
  printf("\n有换行的输出");
  sleep(3);

  return 0;
}

```

输出效果：

* 前 3 秒：屏幕空白（数据在用户态缓冲区）

* 中间 3 秒：看到 “无换行的输出”

* 最后 3 秒：看到 “有换行的输出“

```cpp
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
int main() {
    close(1); // 关闭标准输出（fd=1）
    int fd = open("log.txt", O_WRONLY | O_CREAT | O_TRUNC, 0666);
    printf("hello world: %d\n", fd); // stdout被重定向到文件，缓冲类型变为全缓冲
    close(fd);
    return 0;
}
```

运行后`log.txt`为空->重定向到普通文件了，全缓冲，`\n`刷新不了缓冲区，`close`是系统调用，直接关闭文件，缓冲区数据丢失

### FILE结构体核心：封装fd和缓冲区

先来一段代码：

```cpp
#include <stdio.h>
#include <string.h>
#include <unistd.h>

int main(){
  const char* msg1 = "hello printf\n";
  const char* msg2 = "hello fwrite\n";
  const char* msg3 = "hell write\n";

  printf("%s",msg1);// 用户态缓冲
  fwrite(msg2,strlen(msg2),1,stdout);// 用户态缓冲
  write(1,msg3,strlen(msg3)); // 内核缓冲

  fork();
  return 0;
}
```

输出：

```bash
[vect@VM-0-11-centos file]$ gcc -o mfile mFILE.c 
[vect@VM-0-11-centos file]$ ls
bit.c       chdir.c  hello.c  mFILE.c  mydup2  write.c
buf_line.c  fd.c     mfile    mychdir  myfile
[vect@VM-0-11-centos file]$ ./mfile 
hello printf
hello fwrite
hell write
[vect@VM-0-11-centos file]$ ./mfile  > file
[vect@VM-0-11-centos file]$ cat file
hell write
hello printf
hello fwrite
hello printf
hello fwrite

```

分析：

> * 在终端输出：**行缓存遇到`\n`提前刷新，用户态缓冲为空**，也就是说父进程的用户态缓冲区为空，`fork`之后子进程复制不到任何缓冲数据，父子进程都不会重复输出
> * 重定向到文件输出：**普通文件是全缓冲，`\n`不触发缓冲区刷新**，父进程的用户态缓冲区没有刷新，`fork`之后，父进程的用户态缓冲区被拷贝到子进程，父子进程有同一份相同的缓冲数据，`write`写入内核缓冲区，进程退出时：父子进程均触发用户态缓冲区刷新，所以`printf`和`fwrite`出现两次，`write`只出现一次

所以C封装的库函数自带缓冲区，`FILE`结构体封装了`fd`和用户态缓冲区

## 内核态缓冲区 Linux内核层面 todo



# 7. 封装简易stdio文件流

```cpp
#include <sys/stat.h>
#include <unistd.h>
#include <string.h>
#include <fcntl.h>
#include <stdlib.h>
#include <sys/types.h>
#include <stdio.h>

#define SIZE 1024

#define FLUSH_NO    0
#define FLUSH_LINE  1
#define FLUSH_FULL  2

typedef struct{
  int fileno;       // 对应fd
  int flags;        // 缓冲类型
  int cap;
  int size;
  char outbuf[SIZE];
}mFILE;

mFILE* mfopen(const char* path, const char* mode){
  int fd = -1;
  if(strcmp(mode,"r") == 0){
   fd = open(path,O_CREAT | O_RDONLY, 0666);
  }
  if(strcmp(mode,"w") == 0){
    fd = open(path,O_CREAT | O_WRONLY | O_TRUNC, 0666);
  }
  if(strcmp(mode,"a") == 0){
    fd = open(path,O_CREAT | O_APPEND | O_RDWR, 0666);
  }

  if(fd < 0) return NULL;

  mFILE* mf = (mFILE*)malloc(sizeof(mFILE));
  if(!mf){
    close(fd);
    return NULL;
  }

  mf->fileno = fd;
  mf->cap = SIZE;
  mf->size = 0;
  mf->flags = FLUSH_LINE;
}

void mffulsh(mFILE* stream){
  // 缓冲区有数据才刷新
  if(stream->size > 0){
    // 1. 写入内核缓冲区
    write(stream->fileno, stream->outbuf, stream->size);

    // 2. 强制刷新到终端
    fsync(stream->fileno);

    // 3. 重置缓冲区
    stream->size = 0;
  }
}

size_t mfwrite(const void* ptr, size_t size, mFILE* stream){
  // 1. 拷贝数据
  memcpy(stream->outbuf+stream->size, ptr, size);
  stream->size += size;

  // 2. 行缓冲检测 
  if(stream->flags == FLUSH_LINE && stream->size > 0 && stream->outbuf[stream->size - 1] == '\n')
    mffulsh(stream);

  return size;
}

void mfclose(mFILE* stream){
  if(stream->size > 0)
    mffulsh(stream);
  close(stream->fileno);
  free(stream);
}

int main(){
  mFILE* fp = mfopen("./test.txt","w");
  if(!fp){
    perror("err");
    return 1;
  }
  const char* msg = "hello stdio\n";
  mfwrite(msg,strlen(msg),fp);

  mfclose(fp);

  return 0;
}

```







