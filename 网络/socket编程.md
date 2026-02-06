## 预备知识

### 理解源IP地址和目的IP地址

IP在网络中，标识主机的唯一性

思考：数据传输到主机是目的吗？不是，数据是给人用的，人是怎么看到数据的呢？需要启动特定的app，而启动的app就是进程->进程是人在系统中的代表，只要把数据给进程，人就相当于拿到了数据

![image-20260201105912554](C:/Users/hp/AppData/Roaming/Typora/typora-user-images/image-20260201105912554.png)

用户（进程） + 网络（OS）/主机 -> 网络（OS）/主机 + 用户（进程）

IP地址4字节

### 端口号

port是传输层的内容

* 2字节的整数
* 标识一个进程，告诉OS，当前这个数据交给哪个进程来处理
* 一个端口号只能标识一个进程

IP地址->主机地址				端口号->进程

IP地址+端口号 （socket插头插座）->网络中特定主机的特定进程，**全网唯一的一个进程！**

本质：**进程间通信！**socket之间的通信

所以OS要维护一张端口号的哈希表

**端口号范围划分**

* 0~1023：知名端口号 `HTTP` `SSH`等 这些广为使用的应用层协议，端口号固定
* 1024-65535：OS动态分配的，客户端的端口号

### 传输层

TCP：传输控制协议

* 可靠
  * 面向字节流--->按照需求获取，随意量纲获取，用户层自己解释，怎么发，怎么读都无要求约束
* 有连接

UDP：用户数据报协议

* 不可靠
* 面向数据报--->寄快递，怎么读严格按照发送时候的规格
* 无连接

### 网络字节序

都按照大端发送，接收，不管机器是大端还是小端，都要转换一遍：

```cpp
// 为什么需要字节序转换？
// 答：不同CPU架构的字节序不同

// 大端序(Big-endian)：高位字节在低地址
//   网络字节序、PowerPC、SPARC
// 小端序(Little-endian)：低位字节在低地址
//   x86、x86-64、ARM（通常）

// 转换函数家族：
uint16_t htons(uint16_t hostshort);   // 主机序转网络序（16位）
uint32_t htonl(uint32_t hostlong);    // 主机序转网络序（32位）
uint16_t ntohs(uint16_t netshort);    // 网络序转主机序（16位）
uint32_t ntohl(uint32_t netlong);     // 网络序转主机序（32位）
```

主机host->hs

传输to->to

网络network->n

![image-20260201114009321](C:/Users/hp/AppData/Roaming/Typora/typora-user-images/image-20260201114009321.png)

socket接口统一：

基类：`struct sockaddr`

派生类`struct sockaddr_in`->inet 网络通信

派生类：`struct sockaddr_un`-> unix 本地通信

![image-20260201115030445](C:/Users/hp/AppData/Roaming/Typora/typora-user-images/image-20260201115030445.png)





## 直接上代码

先写好代码整体框架，创建客户端和服务端的类

### 服务器

初始化服务器

启动服务器

![image-20260201123706067](C:/Users/hp/AppData/Roaming/Typora/typora-user-images/image-20260201123706067.png)

#### 初始化服务器

第一、创建套接字

`int socket(int domain, int type, int protocol)`

参数一：协议族，进行什么通信`AF_INET`网络通信`AF_UNIX`本地通信

参数二： 类型`SOCK_DGRAM`今天创建UDP协议

参数三：传输层的类型，默认为0就够了

UDP创建套路固定：

`_soketid = socket(AF_INET, SOCK_DGRAM, 0);`

如果创建失败，直接结束就行，搞个接口`Die()`--->让进程直接退出，同时输出日志信息

![image-20260201124758413](C:/Users/hp/AppData/Roaming/Typora/typora-user-images/image-20260201124758413.png)

成功返回文件描述符，失败-1---->本质是创建了一个文件



第二、绑定套接字

`int bind(int sockfd, const struct sockaddr* addr, socklen_t len)`

参数一：套接字文件描述符socket的返回值

参数二：前两位的地址，代表通信类型

先创建好然后传参，`struct sockaddr_in local`

第二个参数要强转成人家函数要的类型

`int n = ::bind(_sockfd, (CONV)&local, sizeof(local)); `

对于`sockaddr_in`字段：要填充自己结构体里的字段,**先清空默认的，之后再填充**

```cpp
// 填充网络信息，还没有设置进内核
struct sockaddr_in local;
bzero(&local, sizeof(local));
local.sin_family = AF_INET;
// 服务器给别人发消息，也要收到别人的消息
// 我自己的端口号要发到网络中去的！
// 所以主机序列先要转成网络序列！！！
local.sin_port = ::htons(_port);
// string ip -> 4bytes -> network order
// in_addr_t inet_addr(const cha* p) 直接完成两步
// 但是sin_addr是结构体，结构体里只有一个对象，C结构体不能整体赋值，只能整体初始化，只能指定结构体内部对象赋值
local.sin_addr.s_addr = ::inet_addr(_ip.c_str());

// 设置进内核
int n = ::bind(_sockfd, (CONV)&local, sizeof(local));

```

这时候需要新变量了：`uint16_t _port;		std::string _ip;`



#### 运行服务器

加类变量，是否允许

 服务器启动之后不结束

**收消息：**

`ssize_t recvfrom(int sockfd, void* buf, size_t len, int flags, struct sockaddr* src_addr, socklen_t* addrlen)`

前几个参数：从指定套接字，收`len`字节大小的消息，放到缓冲区里，flags默认为0，阻塞等待

后两个参数：需要知道谁给我发的消息->**如何标识另一端？**

IP+PORT->socket

所以要定义远端

```cpp
struct sockadd_in peer;
cocklen_t len = &sizeof(peer);
char inbuf[1024];
ssize_t n = ::recvfrom(_sockfd, inbuf, sizeof(inbuf) - 1, 0, CONV(&peer), &len);
```

**发消息：**

`socket fd`既可以发，也可以收---全双工

读写用同一个socket

`ssize_t recvfrom(int sockfd, void* buf, size_t len, int flags, struct sockaddr* dest_addr, socklen_t* addrlen)`

后两个参数：目标主机进程

```cpp

::sendto(_sockfd, echo_string.c_str(), echo_string.size(), 0, CONV(&peer), sizeof(peer))
```



`netstat -uap`查看所有运行的UDP

### 客户端

客户端不需要绑定吗？

客户端必须要有自己的IP和端口号，但是客户端不需要自己显式调用bind

客户端首次sendto消息的时候，OS自动bind

怎么理解自动随机绑定端口号？一个端口号只能一个进程绑定，客户端不需要被别人找到，只要保证唯一性就好

服务器的端口号必须稳定，且必须众所周知且不能被改变





注意：**云服务器禁止绑定公网IP->这个是虚拟出来的**

虚拟机，可以绑定任何IP

所以，现在改变，服务器不需要IP地址，只需要设置为`INADDR_ANY`，服务器可以接收任何IP地址的信息，只要端口号符合

