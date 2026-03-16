# 一 、 底层地址转换和Socket封装

## 地址转换

计算机网络里最麻烦的是“语言不通”

* 用户想看`192.168.1.1`字符串和`8080`整数
* 网络**只认识大端字节序的二进制**
* Socket接口只认识`struct sockaddr`的通用结构体

可以封装一个类，把这些乱七八糟的格式统一转换，随时取用

```cpp
#pragma once

#include <iostream>
#include <string>
#include <sys/socket.h>
#include <sys/types.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <cstring>
#include "Common.hpp"

class InetAddr{
private:
    // 人能看懂的
    std::string _ip;
    uint16_t _port;

    // OS视角包装成结构体
    struct sockaddr_in _net_addr;

    // 网络转主机
    void IpNet2Host() { 
        char buffer[64];
        //  const char *inet_ntop(int af, const void *restrict src,
        //                     char dst[restrict .size], socklen_t size);
        // 协议族、源地址、目标地址缓冲区、目标地址缓冲区大小
        const char *ip = ::inet_ntop(AF_INET, &_net_addr.sin_addr ,buffer, sizeof(buffer));
        _ip = buffer;
        (void)ip;
    }

};
```

再梳理整个思路，三个视角：

> * 用户视角（主机）：用户只认识点分十进制`'127.0.0.1:8080'`
>
> * 网络视角（INET）：规定网络地址以大端字节序存储，而市面上的主机有大端也有小端，所以向网络通信之前，一律先转成大端字节序
>
> * 内核视角（OS）：为了沟通用户和网络之间，内核定义了一个`struct sockaddr`的结构体建立了二者的联系
>
>   ![image-20260302110833247](https://gitee.com/binary-whispers/pic/raw/master///20260302110835422.png)
>
>   * **为什么一定要清零？**
>
>     `sockaddr_in`为了强制派生类和基类大小相等，塞入了一个8字节的占位符`sin_zero[8]`，如果不清零，可能会产生垃圾数据拒绝连接



## Socket封装



# 一、线程安全基建：封装RAII机制的锁

## 为什么要造这个轮子

在高并发的Reator线程池中，如果有多个线程取抢夺资源，不加锁一定会导致数据竞争。C时代可能会有这种写法：

```cpp
pthread_mutex_lock(&mutex);

if (some_error_happens) {
    // 糟糕！忘记写 pthread_mutex_unlock(&mutex) 就直接 return 了！
    return false; 
}
// 或者中间抛出了一个 C++ 异常，直接跳出了当前作用域...

pthread_mutex_unlock(&mutex);
```

**致命后果：**在庞大的业务逻辑中，哪怕只有一个分支漏写了`unlock`，或者因为抛异常没有执行到`unlock`，整个服务器就会因为**死锁**瞬间挂掉

## RAII：资源获取即初始化

C++有个伟大的特性：**局部对象离开作用域时，会自动调用析构函数**。不管是`return`出去，或者抛异常，析构函数一定会执行

我们可以利用这个特性：**把加锁放在对象的构造函数里，把解锁房子啊对象的析构函数里**

以下是手撕的具体思路：

* `class Mutex`：
  * 内部包含一个原生的`pthread_mutex_t _mutex;`
  * 在构造函数里初始化锁，在析构函数里销毁锁
  * 提供加锁和解锁的方法
* `class LockGuard`：
  * 内部持有**`Mutex`的引用**
  * 构造函数中调用加锁逻辑
  * 析构函数中调用解锁逻辑

以下是具体代码：

```cpp
#pragma once
#include <iostream>
#include <pthread.h>

namespace LockModule{
    class Mutex{
    private:
        pthread_mutex_t _mutex;
    public:
        // 禁用拷贝构造和拷贝赋值
        Mutex(const Mutex&) = delete;
        const Mutex operator=(const Mutex&) = delete;
        
        Mutex() { ::pthread_mutex_init(&_mutex, nullptr); }

        void Lock() { ::pthread_mutex_lock(&_mutex); }

        void UnLock() { ::pthread_mutex_unlock(&_mutex); }

        ~Mutex() { ::pthread_mutex_destroy(&_mutex); }

        // 获得原生锁
        pthread_mutex_t* GetMutex() { return &_mutex; }
    };

    class LockGuard{
    private:
        Mutex& _lock;
    public: 
        LockGuard(Mutex& lock) :_lock(lock)  { _lock.Lock(); }
        ~LockGuard() { _lock.UnLock(); }
    };
}
```

## 面试考点：浅拷贝引发的双重释放(Double Free)灾难

假设没有禁用拷贝，并且写了这样的代码

```cpp
Muetx m1;
Mutex m2 = m1;	// 赋值是浅拷贝
```

C++中，如果没有自定义拷贝构造函数，编译器会进行**按字节复制（浅拷贝）**

内存中会发生：

![image-20260301132905576](https://gitee.com/binary-whispers/pic/raw/master///20260301132907682.png)

## 继续延申：深浅拷贝机制讲解

假设有一个类，内部管理着一块堆内存

**浅拷贝：编译器默认行为**

![image-20260301133400672](https://gitee.com/binary-whispers/pic/raw/master///20260301133402682.png)

当1和2离开作用域时，它们的析构函数都会执行 `delete[] buffer`。同一块堆内存被释放了两次，直接触发操作系统底层的 **Double Free** 保护机制，进程当场 **Coredump（崩溃）**！

**深拷贝：真正的值隔离**

![image-20260301133629174](https://gitee.com/binary-whispers/pic/raw/master///20260301133630925.png)

1和2各自管理自己的堆内存，互不干扰，析构时完美释放

**如何理清楚深浅拷贝机制？**

> * **定义与现象：**
>   * 浅拷贝是编译器默认行为，只是复制指针变量的地址，导致两个对象指向同一块空间
>   * 深拷贝不仅复制指针，还在堆上开辟一块相同大小的内存，把真实的数据完整复制过去，两个对象在物理和逻辑上都隔离开来
> * **危害：**
>   * 管理了系统资源（套接字、锁、动态内存）的类，浅拷贝场景两个对象会`delete`同一块内存两次，总有先后顺序，引发野指针继而内存泄漏。如果其中一个对象先修改了这块内存，另一个对象也会收到影响
> * **解决：**
>   * 只要类自定义了析构函数，就必须同时手写**拷贝构造和赋值重载**，但是深拷贝代价大
>   * 现在可以：
>     * 资源有独占性（锁，套接字），直接禁用拷贝构造和赋值重载
>     * 需要转移资源的所有权，使用**移动语义**，通过右值引用，把底层指针“偷过来”，把原指针置空

****

# 二 、`Epoller`侦察兵

## `epoll`底层原理剖析

### `epoll`内核微观世界：

![image-20260301151521719](https://gitee.com/binary-whispers/pic/raw/master///20260301151523482.png)

### 三大核心API剖析

#### `epoll_create`--建模型

```cpp
#include <sys/epoll.h>
int epoll_create(int size);
```

* **参数`size`：** 现已废弃，填一个**大于0**的数接客
* **返回值**
  * 成功返回一个**epoll句柄**，占用一个文件描述符
  * 失败返回-1，并设置`errno`

**在底层干了什么？**

> * **表层：**创建一个epoll
>
> * **底层：**调用这个函数，Linux内核会在内核空间中申请一块内存，并创建一个名为`struct eventepoll`的结构体
>
>   ```cpp
>   // 内核视角：为你创建的管家
>   struct eventpoll {
>       spinlock_t lock;          // 保护就绪链表的自旋锁
>       struct mutex mtx;         // 保护红黑树的互斥锁
>       wait_queue_head_t wq;     // 阻塞等待队列（epoll_wait 没数据时睡在这里）
>       struct list_head rdllist; // 【核心】双向链表：专门放来消息的 fd
>       struct rb_root rbr;       // 【核心】红黑树：专门放所有被监控的 fd
>   };
>   ```
>
> * **做两件事：**
>
>   * 初始化一棵**空的红黑树（存放要监听的socket）**
>   * 初始化一个**空的双向链表（作为就绪队列，存放来消息的socket）**

#### `epoll_ctl`--增删改监控fd

```cpp
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

* **参数`epfd`：** `epoll_create`返回的fd

* **参数`op`：**

  * `EPOLL_CTL_ADD`：新增监控
  * `EPOLL_CTL_MOD`：修改监控
  * `EPOLL_CTL_DEL`：取消监控

* **参数`fd`：** 要监控的文件描述符

* **参数`event`：**要监控什么动作？是一个结构体：

  ```cpp
  struct epoll_event {
      uint32_t events;  // 关心的事件：EPOLLIN(可读), EPOLLOUT(可写), EPOLLET(边缘触发)
      epoll_data_t data; // 用户数据载体，通常是一个联合体，我们最常用的是 data.fd = sockfd
  };
  ```

  * `uint32_t events`是一个位图

    * `epoll_ctrl`中是站在**用户->内核**的用户视角下，告诉内核：”帮我盯着这个`fd`，并且我只关心它能不能读(EPOLLIN)，并且用**边缘触发(EPOLLET)**方式通知我。“--->这里具体化了
    * `epoll_wait`中是站在**内核->用户**的内核视角下。`epoll_wait`返回时，内核把`events`填好还给用户，告诉用户：”你交代的那个`fd`，现在确实可读了“

    所以，`uint32_t events`是一个事件类型（我关心什么动作）+ 通知策略（你该怎么通知我）的集合

**如何做到网卡回调？**

> * **包装节点：** 内核把传来的`fd`和关心的`event`包装成一个`struct epitem`对象
> * **挂载红黑树：** 把这个`epitem`作为树节点，按$O(logN)$的速率插入到管家的红黑树中
> * **注册回调：** 内核走到协议栈底层的`vfs_poll`，指着这个`fd`对底层的网卡驱动说：”兄弟，以后如果有发给这个`fd`的报文到了，别让我去轮询找你，你直接触发我留给你的回调函数--`ep_poll_callback`“
>
> 数据包顺着网线到达网卡，**触发硬件中断**，底层驱动执行回调方法，找到对应的`epitem`，插入到就绪队列中

#### `epoll_wait`

```cpp
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

* **参数`epfd`：** 监控中心的钥匙
* **参数`events`：** **传出参数**，在用户态提前开辟好的一段数组，内核会把真正发生事件的`fd`信息填入到这个数组给你
* **参数`maxevents`：** 这个数组最大能装多少个事件，不能越界
* **参数`timeout`：**等多久？
  * `-1`：阻塞等待，直到有时间发生或异常返回
  * `0`：非阻塞轮询，看一眼立刻走，不管有没有数据
  * `>0`：最多等这么多毫秒，如果有事件或超时返回
* **返回值：**
  * `>0`：有几个fd发生了事件
  * `0`：超时了，目前没发现有事件发生
  * `-1`：系统调用出错

**底层做了什么？**

> * **查就绪队列：** 直接看管家里的双向链表是不是空的
> * **睡眠等待：** 如果是空的且设置了阻塞等待，内核就把你的线程丢进等待队列里睡觉，让出CPU
> * **搬运：** 只要就绪队列里有节点（无论是本来就有，还是刚进来的），内核立刻把链表里的节点**拷贝到传入的`events`用户态数组中**
> * **清理现场：** 拷贝完后，把这些节点删除（如果是LT模式且数据没读完，会塞回来，后话）

### `epoll`一些细节梳理

**`epoll`比`select`快，到底快在哪里？**

> * **拷贝开销剥离：** `select`每次调用，需要把fd集合从用户态拷贝到内核态，调用完再拷贝回来，代价极大。`epoll`把配置`epoll_ctl`和等待`epoll_wait`分开了，只要`ctl`加进红黑树，红黑树就相当于`select`的辅助数组，后续就不用管了
> * **没有无脑$O(N)$的遍历：** `select`不知道到底是哪几个fd来了数据，每次必须要暴力遍历所有的`fd`，并发越高越慢。`epoll`**基于事件驱动的回调机制**，只有真正活跃的`fd`会被回调放入就绪队列，`epoll_wait`是$O(1)$的，只和被激活的连接数有关，和总连接数无关

**回调机制，一个网络包从网卡到达，到`epoll_wait`返回，内核发生了什么？**

> * **硬件中断：** 数据包到达网卡，网卡把数据包拷贝到内存，然后向CPU发硬件中断
> * **协议栈解析：** CPU暂停当前任务，执行中断。内核协议栈剥离Mac帧、IP投、TCP头，把有效子啊和放到对应socket的**接收缓冲区**
> * **触发回调：** 内核发现这个socket之前通过`epoll_ctl`注册过回调函数，立即执行
> * **进入就绪队列：** 回调函数找到对应socket在红黑树上对应的节点，把它拎下来，**插入到就绪队列中**，同时唤醒`epoll_wait`阻塞的线程
> * **用户态收网：** 被唤醒后，直接将就绪队列中的数据拷贝回用户空间，业务代码拿到`fd`开始执行后续逻辑

**为什么epoll用红黑树存关心的fd，用双向链表存就绪的fd，哈希表不行吗？**

> * **双向链表：** **插入和删除都是$O(1)$**，只用检测是否为空，空休眠，非空就拷贝
> * **红黑树：** 
>   * 红黑树增删查改复杂度稳定为$O(logN)$，高并发场景也是
>   * 为啥不用哈希表？哈希表理论是$O(1)$,但遇上高并发场景，哈希冲突极其严重，并且哈希表需要巨大的连续内存，扩容代价大

## 封装`Epoller`

```cpp
#pragma once

#include <iostream>
#include <cstring>
#include <unistd.h>
#include <sys/epoll.h>
#include "Common.hpp"

class Epoller{
private:
    int _epfd;

   // fd: 要关心的fd
   // events: 以什么方式关心（读写？）就绪之后以什么方式通知（ET/LT）
   // po: 操作方式
    void Ctrl(int sockfd, uint32_t events, int op){
        // 挂载节点、修改节点
        struct epoll_event ev;
        ev.events = events;

        // 必须把sockfd绑到event的user_data里
        // 当网卡触发回调，会把这个节点塞进就绪队列
        // epoll_wait返回给你的时候，如果不绑定fd，就不知道谁来的消息
        ev.data.fd = sockfd;

        int res = ::epoll_ctl(_epfd, op, sockfd, &ev);
        if(res < 0){
            std::cerr << "epoll_ctl error for fd: " << sockfd << std::endl;
        }
    }

public:
    // 这里先硬编码
    Epoller() :_epfd(-1) { }
    ~Epoller() { if(_epfd >= 0) ::close(_epfd);}
    // 禁用拷贝构造和赋值，_epfd是独占资源
    Epoller(const Epoller&) = delete;
    const Epoller& operator=(const Epoller&) = delete;


    void Init(){
        _epfd = ::epoll_create(1);
        if(_epfd < 0) {
            std::cout << "epoll_create err, errno: " << errno << std::endl;
            exit(EPOLL_CREATE_ERR);
        }
        std::cout << "epoll_create success, epfd: " << _epfd << std::endl;
    }

    // 给Reactor用的API
    void Add(int sockfd, uint32_t events) { Ctrl(sockfd, events, EPOLL_CTL_ADD); }
    void Update(int sockfd, uint32_t events) { Ctrl(sockfd, events, EPOLL_CTL_MOD); }
    void Delete(int sockfd){
        // 从红黑树上摘除节点，内核不关心你的events是什么
        // 最后一个参数传nullptr，省去构造epoll_wait的开销
        int res = ::epoll_ctl(_epfd, EPOLL_CTL_DEL, sockfd, nullptr);
        if (res < 0) {
                std::cerr << "epoll_ctl DEL error for fd: " << sockfd << std::endl;
        }
    }

    // revs[]: 输出型参数，在用户态准备好的空数组，内核会把就绪队列里的数据拷贝到这个数组
    // num: 数组容量
    int Wait(struct epoll_event revs[], int num, int timeout){
        // 有东西 -> 拷贝到revs数组 -> 返回抓取的数量
        // 没东西 -> 线程挂到等待队列，让出CPU
        return ::epoll_wait(_epfd, revs, num, timeout);
    }
};
```

**漏写`ev.data.fd = sockfd`会发生什么？**

> * **唤醒：** 网卡收到数据，触发中断，执行回调，把节点挂载到就绪队列，唤醒`epoll_wait`
> * **bug：** `epoll_wait`被唤醒，拷贝到`revs`数组，但是！**注册时没有给`data.fd`赋值，未初始化的局部结构体是随机值！**，Reactor拿到一个包含垃圾值的`fd`的事件找连接，可能导致数组越界，**服务器段错误崩溃**
>
> `ev.data`是用户态和内核态之间传递上下文的**唯一凭证**，没有它，用户态拿到的数据是垃圾数据

**内核事件怎么跑到用户态的`revs`数组？**

> **直接拷贝**



# 三、`Connection`流水线管理

## 为什么要设计这个类？

传统的`epoll_wait`循环中，代码很臃肿：

```cpp
// 纯 C 时代的灾难现场
for (int i = 0; i < n; i++) {
    int fd = revs[i].data.fd;
    if (fd == listen_sockfd) {
        // 这是接客的 Socket，去调 accept
        Accept(fd);
    } else {
        // 这是普通的 Socket，去调 recv
        Recv(fd);
    }
}
```

* **扩展性极差：**如果后续加了定时器`fd`，信号`fd`，这套`if-else`逻辑会不断拉长
* **非阻塞IO前提：** 我们用的**ET+非阻塞IO**，如果调用`send`时内核缓冲区满了，剩下的数据存到哪？如果调用`recv`只收到了半个报文，这个半个报文存哪？

**一切皆文件思想指导一切皆连接：**

Linux下一切皆文件，在Reactor严重，**一切皆连接**，无论是接客的Listener，还是通信的DataSocket，在框架眼里没有区别：

1. **多态回调**：我们把 `fd` 包装成一个 `Connection` 对象，提供 `Recver()` 和 `Sender()` 虚函数。管家拿到事件后，闭着眼睛无脑调 `conn->Recver()`，具体干嘛由子类决定

   * 如果指向`Listener`对象，底层就自动执行`accept`接客
   * 如果指向`IOService`对象，底层就自动执行`recv`读数据

   **让管家的分发逻辑和具体的业务逻辑解耦！**

2. **应用层缓冲区**：每个 `Connection` 对象自己兜里揣两个口袋（`_inbuffer` 和 `_outbuffer`）。读到残缺数据，先放进兜里；发不完的数据，也先放进兜里

3. **回指大管家**：如果兜里的 `_outbuffer` 突然有数据要发了，员工必须能找到老板（`Reactor`），让老板去 `epoll` 里把自己的 `EPOLLOUT` 事件打开。所以必须有个指针指向管家

## 代码实现

```cpp
#pragma once

#include <iostream>
#include <string>
#include <unistd.h>
#include <functional>
#include <cstdint>

class Reactor; // 前置声明，告诉编译器epoll模型存在
class Connection{
protected:
    int _sockfd;
    uint32_t _events;       // 在epoll中关心的事件
    std::string _inbuffer;
    std::string _outbuffer;
    Reactor *_owner;        // 回指指针：我是哪个管家的员工
public:
    Connection(int fd = -1) 
        :_sockfd(fd)
        ,_events(0)
        ,_owner(nullptr)
    {}

    // 多态基类必须要有虚析构
    virtual ~Connection() {}

    // 暴露给Reactor和业务层的接口
    void SetSockfd(int fd) { _sockfd = fd; }
    int GetSockfd() { return _sockfd;}

    void SetEvents(uint32_t events) { _events = events; }
    uint32_t GetEvents() { return _events; }

    void SetOwner(Reactor *owner) { _owner = owner; }
    Reactor *GetOwner() { return _owner; }

    // 缓冲区操作接口
    // TCP不保证每次发来的都是一个完整的包裹
    // 网卡收到一段碎数据，不能直接丢给业务层，先存到自己的接受仓库攒着
    void AppendInBuffer(const std::string &in) { _inbuffer += in; }
    std::string &GetInBuffer() { return _inbuffer; }
    // 协议层成功切分走一个完整包裹后，必须把这部分数据删除
    void DiscardInBuffer(int len) { _inbuffer.erase(0,len); }
    
    // 非阻塞IO下，如果内核发送窗口满了，发不完的数据先存进发送仓库
    void AppendOutBuffer(const std::string &out) { _outbuffer += out; }
    std::string &GetOutBuffer() { return _outbuffer; }
    // 清理库存：底层发走了100字节，就必须把这100字节从发送仓库删除，剩下的下次再发
    void DiscardOutBuffer(int len) { _outbuffer.erase(0,len); }
    bool IsOutBufferEmpty(){ return _outbuffer.empty(); }

    // 关闭socket
    void Close(){
        if(_sockfd >= 0){
            ::close(_sockfd);
            _sockfd = -1;
        }
    }
    // 留给派生类去实现
    virtual void Recver() = 0;
    virtual void Sender() = 0;
    virtual void Excepter() = 0;

};
```

## 代码相关问题

### 基类析构函数为什么必须是虚函数？

用基类指针管理不同的派生类对象

C++底层，如果基类`Connection`不是虚函数，在`delete conn`时，会**静态绑定**，一看指针类型是`Connection*`，就直接去调用基类的析构函数，然后释放内存，这样会导致派生类自己申请的资源没有清理，造成内存泄漏

如果设置为虚函数，编译器就会**动态绑定**，在对象的内存模型最前端，生成一个**虚函数表指针（vptr）**，执行`delete conn`时，会先顺着vptr找到真正的**虚函数表**，在表里查出这个实际对象，进行**精确地析构，最后析构基类**，保证资源完整释放

## 1GB大文件传输，会撑爆`_outbuffer`吗？怎么解决？

会，不让文件数据进入用户态的 `_outbuffer`，直接砍掉常规的 `read/send`，改用 Linux 操作系统提供的 **Zero-Copy（零拷贝）** 技术，核心系统调用是 **`sendfile()`**。 调用 `sendfile(out_fd, in_fd, offset, count)` 时，数据会直接在内核空间中，从磁盘的页缓存（Page Cache）**直接通过 DMA 引擎传输到网卡的 Socket 发送缓冲区**，全程不需要 CPU 参与拷贝，也不需要穿越到用户态