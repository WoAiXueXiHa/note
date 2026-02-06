# 序列化反序列化

**太精华了这一部分！！！！**

内核底层原理

## 之前说的读取不完善，为什么？

**结论一：**创建一个TCP socket时，在TCP层创建两个缓冲区：发送+接收

写入->拷贝到发送缓冲区

读取->从接收缓冲区拷贝到内存->接收缓冲区为空->read/recv阻塞！！！

网络收发数据->只是拷贝到缓冲区或从缓冲区拷贝出去！！！！

**whyTCP支持全双工？**->收发时，有两套不同的缓冲区！！！

看源码：

`struct file{`

![image-20260204215947504](C:/Users/hp/AppData/Roaming/Typora/typora-user-images/image-20260204215947504.png)

如果是网络文件，`private_data`直接指向socket，socket

`struct socket{`

![image-20260204220132810](C:/Users/hp/AppData/Roaming/Typora/typora-user-images/image-20260204220132810.png)

sock里有：

![image-20260204220311519](C:/Users/hp/AppData/Roaming/Typora/typora-user-images/image-20260204220311519.png)

OS内部可能会存在大量的报文！！！

需要管理报文！！！！结构体管理--->sk_buff

![image-20260204220503864](C:/Users/hp/AppData/Roaming/Typora/typora-user-images/image-20260204220503864.png)

缓冲区里有各种包含报头的请求数据

![image-20260204220628531](C:/Users/hp/AppData/Roaming/Typora/typora-user-images/image-20260204220628531.png)

形成链表管理



TCP协议：传输控制协议



**面向字节流：**

UDP规定发出去的报文必须是完整的，把报文整体打包在发送->快递，要么完整拿到，要么丢包

TCP->自来水，水厂以流式方式供水，用户（应用层）怎么使用，拿桶接？拿杯子？

所以TCP传输的数据完整性由应用层决定！！！





json代码不用会写->LLM给，会用就行，掌握好网络创建，

会话层 表示层 应用层这三层需要程序员自己个性化实现



## 守护进程

**进程间关系**

我们现在有一个服务了，想让服务变成一个真正的服务->守护进程

键盘只有一个，标准输入只有一个，任何登录，只允许任何时刻只有一个前台进程，0个或多个后台进程

命令行启动任何进程，bash会自动变成后台进程，直到前台进程结束！

**进程组**

进程组是为了完成一个任务或者作业

如果是一个进程，单独成为一个进程组，单独处理任务或作业

session ID 和PPID 一样



**登录一个客户端系统做了什么？**

系统创建一个bash进程和一个会话，bash打开一个终端文件，创建一个进程组，往后所有的进程都是bash的子进程，都指向同一个终端，但是这些进程都同属于一个会话

无论前台还是后台进程，都同属于一个会话



守护进程要脱离终端

其实就是一个孤儿进程

**如何成为守护进程？**

手动完成守护进程化！

`stesid`创建一个新会话

调用的进程不能是进程组组长->保证自己不是第一个进程

fork之后，父进程直接退出，子进程代替父进程向后运行--->孤儿进程



**代码**

1. 屏蔽特定异常信号

2. 让自己不要成为组长

3. 建立新的会话

4. 每个进程都有自己的cwd，是否将当前进程的cwd更改为根目录

5. 已经成为一个守护进程了，不需要用户的输入输出了，关掉不要的资源

   `/dev/null`->系统提供的黑洞文件，写进去直接被丢弃

   

------

# 【源码级解密】Linux 网络栈：从 fd 到网卡的“证据链”闭环

> **核心疑问**：
>
> 我在代码里写 `int fd = socket(...)`，这个 `fd` 仅仅是个数字（比如 3）。
>
> 为什么我往 `3` 号文件里写数据，网卡就能发包？
>
> 内核到底是怎么把这个 `数字 3` 和复杂的 `TCP 协议栈` 关联起来的？
>
> **证据目标**：我们要找到内核源码中的**指针链条**，证明 `fd` -> `file` -> `socket` -> `sock` -> `sk_buff` 的物理存在。

------

## 第一环：`fd` 到底是什么？(用户态 -> 内核态入口)

在用户态，`fd` 是一个整数索引。在内核态，每个进程都有一个“档案袋”叫做 `task_struct`。

### 1. 源码证据：`task_struct`

在 `include/linux/sched.h` 中：

C

```
// 进程控制块 (PCB)
struct task_struct {
    // ...
    // 【证据 1】每个进程都有一个打开文件表
    struct files_struct *files; 
    // ...
};
```

### 2. 源码证据：`files_struct`

在 `include/linux/fdtable.h` 中：

C

```
struct files_struct {
    // ...
    // 【证据 2】这是一个数组！fd 就是这个数组的下标
    // fd_array[3] 就存放着 fd=3 对应的文件指针
    struct file __rcu * fd_array[NR_OPEN_DEFAULT];
    // ...
};
```

> **详细解释**：
>
> 当你调用 `socket()` 成功返回 `3` 时，内核其实是干了这件事：
>
> ```
> current->files->fd_array[3] = 创建好的 struct file 指针;
> ```
>
> 所以，`fd` 本质上就是一把**钥匙**，用来去数组里拿 `struct file*`。

------

## 第二环：万能容器 `struct file` (VFS 虚拟文件系统)

Linux 奉行“一切皆文件”。Socket 是文件，磁盘也是文件。内核用 `struct file` 来统一表示它们。

### 1. 源码证据：`struct file`

在 `include/linux/fs.h` 中，这是 VFS 层的核心：

C

```
struct file {
    // ...
    // 【证据 3】文件操作函数表
    // 如果是 socket，这里指向 socket_file_ops (包含 sock_recvmsg, sock_sendmsg)
    const struct file_operations    *f_op;

    // ...
    // 【关键证据 4】private_data (私有数据指针)
    // 这是一个 void* 指针。
    // 如果是磁盘文件，它指向 inode；
    // 如果是 Socket 文件，它指向 struct socket！
    void    *private_data;
    // ...
}
```

> **详细解释**：
>
> 这里的 `private_data` 是**多态**实现的关键。
>
> 内核在创建 Socket 时，会强制把 `struct socket` 的地址赋给 `private_data`。
>
> 当你调用 `write(fd, ...)` 时，内核先拿出 `file`，然后取出 `private_data`，把它强转成 `struct socket*`，从而进入网络世界。

------

## 第三环：BSD Socket 层 `struct socket` (从文件到网络)

这一层是“适配层”。它向下屏蔽了 TCP/UDP/UNIX 域套接字的差异。

### 1. 源码证据：`struct socket`

在 `include/linux/net.h` 中：

C

```
struct socket {
    socket_state        state;  // 连接状态 (SS_CONNECTED, SS_LISTENING...)
    short               type;   // 套接字类型 (SOCK_STREAM, SOCK_DGRAM...)
    unsigned long       flags;

    // 【关键证据 5】file 反向指针
    // 指回上面的 struct file，确保两者不分家
    struct file     *file;

    // 【核心证据 6】sk 指针 (重点！)
    // 指向具体的网络协议控制块 (struct sock)
    // 这就是通往 TCP 核心的“虫洞”
    struct sock     *sk;

    // 操作接口 (如 inet_stream_ops)
    const struct proto_ops  *ops; 
    // ...
};
```

> **详细解释**：
>
> `struct socket` 是一个外壳。它负责对接 VFS 文件系统。
>
> 真正的网络逻辑（比如 TCP 滑动窗口、拥塞控制），它自己不干，它全部委托给 `sk` 指向的那个对象去干。

------

## 第四环：协议栈核心 `struct sock` (全双工的家)

这里是 TCP/IP 协议栈的“真身”。所有的网络状态都存在这里。

### 1. 源码证据：`struct sock` (即 INET Sock)

在 `include/net/sock.h` 中。这个结构体非常巨大，我们只看核心：

C

```
struct sock {
    // ...
    // 【核心证据 7】接收队列 (全双工之一)
    // 这是一个双向链表，里面串着一个个 sk_buff (收到的数据包)
    struct sk_buff_head sk_receive_queue;

    // 【核心证据 8】发送队列 (全双工之二)
    // 用户 write 下来的数据，封装成 sk_buff 后挂在这里
    struct sk_buff_head sk_write_queue;

    // 【核心证据 9】等待队列
    // 当 buffer 空了(read) 或 满了(write) 时，
    // 用户线程会睡在这个队列上 (socket_wait_queue)
    struct socket_wq __rcu  *sk_wq;

    // 协议回调 (如 tcp_sendmsg, tcp_recvmsg)
    struct proto        *sk_prot;
    // ...
};
```

> **详细解释**：
>
> 这就是**全双工**的物理证据！
>
> `sk_receive_queue` 和 `sk_write_queue` 是两个独立的链表。
>
> * **读操作**：从 `sk_receive_queue` 摘节点。
>
> * **写操作**：往 `sk_write_queue` 挂节点。
>
>   互不干扰，所以可以同时读写。

------

## 第五环：数据载体 `struct sk_buff` (数据包本体)

数据在内存里到底长什么样？并不是一串连续的 char 数组，而是一个复杂的对象。

### 1. 源码证据：`sk_buff`

在 `include/linux/skbuff.h` 中：

C

```
struct sk_buff {
    struct sk_buff      *next;      // 链表指针
    struct sk_buff      *prev;

    struct sock         *sk;        // 属于哪个 socket
    ktime_t             tstamp;     // 时间戳

    // 【核心证据 10】数据指针 (滑窗设计)
    // 这种设计实现了“零拷贝”封包/解包
    unsigned char       *head;      // 缓冲区头
    unsigned char       *data;      // 有效载荷开始 (随着剥离报头向后移)
    unsigned char       *tail;      // 有效载荷结束
    unsigned char       *end;       // 缓冲区尾
    // ...
};
```

> **详细解释**：
>
> 当数据从 IP 层传给 TCP 层时，内核**不需要拷贝数据**！
>
> 它只需要修改 `data` 指针，让它跳过 IP 头，指向 TCP 头即可。
>
> 这就是 Linux 网络栈高效的原因。

------

## 六、 终极证据链图解 (串联源码)

现在，我们有了充足的证据。让我们把这些源码里的结构体串起来，看看 `write` 到底干了什么。

代码段

```
flowchart TB
    subgraph UserSpace [用户态]
        FD[int fd = 3]
        WriteOp[write(3, "hello", 5)]
    end

    subgraph KernelSpace [内核态]
        
        %% 进程层
        Task[task_struct]
        Files[files_struct]
        FDArray[fd_array[]]
        
        %% VFS层
        File[struct file]
        PrivateData[void *private_data]
        
        %% Socket层
        Socket[struct socket]
        SKPtr[struct sock *sk]
        
        %% 协议层
        Sock[struct sock]
        WriteQ[sk_write_queue]
        
        %% 数据层
        SKB[struct sk_buff]
        Data["data: 'hello'"]

        %% 链接关系
        FD -->|索引| FDArray
        Task --> Files --> FDArray
        FDArray -->|指针| File
        File -->|包含| PrivateData
        PrivateData -.->|强转| Socket
        Socket -->|包含| SKPtr
        SKPtr -->|指向| Sock
        Sock -->|包含| WriteQ
        WriteQ -->|挂载| SKB
        SKB -->|持有| Data
    end

    %% 动作流
    WriteOp --> FD
    style PrivateData fill:#f9f,stroke:#333,stroke-width:2px
    style SKPtr fill:#f9f,stroke:#333,stroke-width:2px
```

------

## 七、 总结：内核到底做了什么？

当你写下 `write(fd, buffer)` 时，内核其实在疯狂地进行**指针跳跃**：

1. **查表**：拿着 `fd` 去 `current->files->fd_array` 查到 `struct file*`。
2. **开门**：通过 `file->private_data` 找到 `struct socket*`。
3. **找人**：通过 `socket->sk` 找到 `struct sock*` (TCP 协议实体)。
4. **造车**：分配一个 `struct sk_buff`，把你的 `buffer` 拷贝进去。
5. **排队**：把 `sk_buff` 挂到 `sock->sk_write_queue` 链表上。
6. **返回**：`write` 函数返回。

**这就是“证据”！** 每一个环节在源码里都有迹可循，没有任何魔法，全是严谨的指针操作。

