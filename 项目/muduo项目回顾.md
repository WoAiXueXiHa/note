# 一 、 项目全貌：分层架构

![image-20260327092532060](https://gitee.com/binary-whispers/pic/raw/master///20260327092535580.png)

整个项目是基于**Reactor模式**的C++高并发网络库，实现了one loop one thread 的多线程模型。

# 二、各模块详解

## 基础工具层（只讲Buffer模块）

### 为什么要用户态缓冲区？

**TCP是字节流协议，没有消息边界**，会产生两个问题：

* **粘包：** 多条消息一次性合成一个包收到
* **半包：** 一条消息没收完整，多个包分批次到达

OS内核有自己的socket收发缓冲区，但是不懂用户的应用层协议。必须提供一个满足特定需求的**用户态的中间层**，把内核读到的字节先存起来，等攒够了一个完整的应用层消息再处理

### Buffer的内存结构是什么样的？

![image-20260327093440937](https://gitee.com/binary-whispers/pic/raw/master///20260327093443296.png)

三个关键量：

* `getHeadSize()` = `_readIndex`（头部废弃区）
* `getReadableSize()` = `_writeIndex - _readIndex`（真正有用的数据）
* `getTailSize()` = `_buffer.size() - _writeIndex`（可写空间）

### 如何解决内存碎片问题？

```cpp
void ensureWriteable(uint64_t len) {
    if(len <= getTailSize()) return;           // 尾部够，直接返回

    // 尾部不够，但头部 + 尾部加起来够
    uint64_t readableSize = getReadableSize();
    std::copy(getReadIndex(), getReadIndex() + readableSize, Begin()); // 数据左移
    _readIndex = 0;
    _writeIndex = readableSize;

    if(getTailSize() < len) {                  // 整理后还不够，才扩容
        _buffer.resize(_writeIndex + len);
    }
}
```

* 每次写入都`resize()`都要$O(n)$拷贝，代价巨大，而如果头部废弃区足够大，那么前面的数据也是白拷贝
* 头部废弃区是**历史读走的数据留下的空间**，数据左移就可以**无损回收**这块空间
* 只有回收不够时，再扩容，避免频繁内存分配

### 怎么读取字符串数据的？

* 常规思路：**开新内存->拷贝原来的字符串到新内存->返回新内存**，拷贝+返回临时拷贝->两次拷贝
* 换个思路：利用`std::string`的构造函数，直接返回这个构造函数，创建字符串时一次性分配好并直接指向Buffer的连续内存段，这样就省去一次无意义拷贝

## 核心事件驱动层

### Reactor模式：整个框架的灵魂

![image-20260327095028086](https://gitee.com/binary-whispers/pic/raw/master///20260327095030785.png)

#### 什么是Reactor模式？

**先区分问题是什么？**

传统服务器处理并发的方式：**每一个连接开一个线程**

一万个连接 = 一万个线程，线程的内存开销 + 上下文切换开销--->系统扛不住

**Reactor核心思想：用一个线程监控所有连接，谁有事件谁说话，没事件就统一阻塞等待**

**Reactor的三个核心角色：**

> * Reactor（反应堆/事件分发器）
>   * 职责：等待事件，时间来了通知事件处理器
>   * 实现：epoll / select / poll
> * Handler（事件处理器）
>   * 职责：绑定在某个 fd 上， 负责处理该 fd 上的具体事件
>   * 实现：回调函数
> * Event（事件）
>   * 类型：可读 / 可写 / 连接 / 断开 / 错误 / 任意

**Reactor工作流程：**

![image-20260327101720996](https://gitee.com/binary-whispers/pic/raw/master///20260327101723173.png)

**Reactor的类型**

* 单Reactor单线程

![image-20260327102013799](https://gitee.com/binary-whispers/pic/raw/master///20260327102019460.png)

​		特点：无锁，简单，业务阻塞会拖垮整个IO

* 单Reactor多线程

  ![image-20260327102655362](https://gitee.com/binary-whispers/pic/raw/master///20260327102657060.png)

  特点：业务不阻塞IO，但read/write还在单线程，高并发IO有瓶颈

* 多Reactor多线程

  ![image-20260327103029429](https://gitee.com/binary-whispers/pic/raw/master///20260327103030999.png)

  特点：主线程只负责`accept`，极轻量；每个子线程独立管理自己的连接，几乎无锁，IO和业务都有水平扩展

一句话总结本质：**把“我去找事件”变成“事件来找我”，再把“事件来了做什么”提前注册好**

#### epoll底层原理

三个关键系统调用：  

```cpp
// 创建 epoll 实例，内核维护一棵红黑树 + 一个就绪链表*
int epoll_create(int size);

// 对红黑树进行增删改*
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
// op: EPOLL_CTL_ADD / EPOLL_CTL_MOD / EPOLL_CTL_DEL*

// 阻塞等待就绪事件，从就绪链表中拷贝事件到 events 数组*
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

**内核数据结构：**

* 红黑树：存储所有被监控的 fd（$O(logN)$ 增删改）
* 就绪链表：当 fd 有事件就绪时，内核把对应节点链入就绪链表
* `epoll_wait` 只需要遍历就绪链表，不需要扫描全量 fd，$O(就绪数)$ 而非 $O(总fd数)$

ET vs LT：

* LT（水平触发，默认）：只要 fd 上有数据，每次 `epoll_wait` 都返回。本项目用 LT，实现简单，不会丢事件
* ET（边缘触发）：只在状态变化时通知一次。ET 必须配合非阻塞 fd + 循环读到 EAGAIN，否则会丢数据。ET 减少了系统调用次数，高性能场景下更优

本项目中 `Poller` 没有设置 `EPOLLET`，是 LT 模式

更多细节参考本文：[五大IO模型和多路转接详解-CSDN博客](https://blog.csdn.net/Vect__/article/details/158578559?spm=1001.2014.3001.5502)



### Poller：epoll的封装

**数据结构：**

```cpp
class Poller {
    int _epfd;                                       // epoll 实例（红黑树根）
    struct epoll_event _events[MAX_EPOLLEVENTS];     // 内核往这里写就绪事件
    std::unordered_map<int, Channel*> _channels;    // fd → Channel 的快速索引
};
```

**为什么要维护`_channels`这张哈希表？**

`epoll_wait` 返回的是 `epoll_event` 数组，每个元素只有 `fd` 和就绪的事件标志。要找到这个 fd 对应的 `Channel`（上面挂着回调），必须有一个 `fd → Channel*` 的映射。哈希表查找是 O(1)

**核心方法：Poll**



