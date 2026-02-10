# 认识URL

网址就是URL

请求的资源就是文件，URL就是Linux的路径结构

**背景理解**

1. 上网时，我的数据给别人or别人的数据给我->本质都在做IO
2. 图片，视频，音频，文本->资源
3. 先确认资源在哪一台服务器上（网络，IP），在什么路径下（系统，路径）IP和路径都有唯一性->URL，IP+Port负责帮我们进行通信
4. URL中的/不一定是根目录，叫做web根目录，两者不一定是同一个

URL没有体现端口号啊！--->成熟的应用层协议，和端口号是强关联的，协议的端口号定死了，二者绑死

URL中一些特殊符号会被编码成16进制格式



**http协议的宏观格式**

http是应用层协议，基于TCP协议

1. 协议的格式是什么？

   

2. 如何确保手法完整性？TCP是面向字节流

请求行：`请求方法[空格]url[空格]http协议的版本\r\n`

请求报头（多行内容）：

`key:[空格]value\r\n`

`key:[空格]value\r\n`

`key:[空格]value\r\n`

空行：`\r\n`

请求正文：用户数据、头像、图片...



## 封装socket

模板方法模式

基类：Socket，规定创建套接字的方法，提供一个/若干个固定模式的socket方法

```cpp
class Socket{
public:
    virtual ~Socket() = default;
    virtual int SocketOrDie() = 0;
    virtual void SetSocketOpt() = 0;
    virtual bool BindOrDie(int listensockfd) = 0;
    virtual bool ListenOrDie(int listensockfd) = 0;
    virtual int Accepter(int listensockfd) = 0;
    virtual void Close(int fd) = 0;
    // 其他方法，需要再加
    
   // // 提供一个创建lsitensockfd的固定方法
   // void BuildTcpSocket(){
   //    
   // }
    
   //// 提供一个UDPSockfd的固定方法
   // void BuildUdpsockfd(){
   //     
   // }  
};
```

派生类：TcpSocket、UdpSocket

1. 创建套接字->创建失败就die

2. 绑定，只要个端口号参数就行，绑定创建来的返回值sockfd，要保证sockfd有效

   创建一个本地变量，置零，填充对应信息->可以直接用InetAddr.hpp里的方法，直接穿个端口号构造就行

   ->绑定失败就die

3. 监听，监听创建套接字的返回值->监听失败就die

4. 连接，连接远端网络来的信息

   未来返回的全都是网络套接字类，不要返回文件描述符

   `using SockPtr = std::shared_ptr<Socket>;`这个放到开头就行

   连接失败就返回空，继续等待下一个远端连接，WARNING

   **我需要知道谁链接的我->文件描述符+客户端信息**

   所以参数要传一个合法的`InetAddr* clinet`

   在InetAddr.hpp里面添加SetAddr方法，来自客户端的结构体信息和结构体大小，把远端来的转成主机序

   返回智能指针类型

5. 读写方法

   recv

   读到的是string* out

   从文件描述符里读，和read一模一样，搞个缓冲区buffer，拼接缓冲区

   send

   发的是string& in

   往文件描述符里写，返回写的大小就行

6. 关闭，关闭创建的套接字文件

**使用套接字**

new一个Sokcet对象

`Socket* sk = new TcpSocket();	sk->初始化方法`

## Tcp服务器

**只负责IO，流式的IO，不进行协议处理**

这里直接调用封装的就行

先把整个类的骨架写出来，成员变量，构造析构

成员变量必须要有个listensock，是否运行

构造->传一个端口号，搞一个智能指针端口号，调用建立tcp方法

运行函数，死循环->

1. 获取连接

2. IO处理

   基类方法添加recv和send方法，调用

   创建新的进程处理

   子进程关掉自己不用的

   直接用孙子进程处理，->处理方法是啥，前面定义

   `using tcphandler_t = std::function<bool(SockPtr,InetAddr)>;`因为是智能指针，可以拷贝

   所以成员变量要加一个处理方法-->耦合度太高了解耦

   父进程关掉自己不用的

   等待进程处理结束

解耦处理初始化tcp，搞定任务处理

析构，调用关闭

## Http服务器和客户端

### 服务器

还是先写骨架，成员变量智能指针构造一个_tsvr对象

构造

开始

_tsvr->InitServer，参数搞个lambda捕获this

_tsvr->Loop();



处理http请求，客户端是谁？->Socket sockfd，InetAddr client

现在底层啥也不关心了！解耦很棒！

模板方法再加一个文件描述符返回的方法->我要打日志

1. 读取请求的完整性，暂时不做

2. 完整http反序列化，http response序列化

   demo1：

   返回一个固定的内容

   status

析构



## 协议定制化

### http请求类



### http回应类





## User-Agent















# http_done

![image-20260207152727146](C:/Users/hp/AppData/Roaming/Typora/typora-user-images/image-20260207152727146.png)

http服务器可以充当代理服务器

相当于进程池，我们自己的http服务器就是父进程

后台服务器就是进程池里的子进程



## refer 当前页面是从哪个页面跳转过来的

权限管理，不能乱跳

## http状态码

表明本次请求的处理状态

* 1xx：接受的请求正在处理
* 2xx：请求正常处理完毕
* 3xx：需要进行附加操作以完成请求
* 4xx：服务器无法处理请求
* 5xx：服务器处理请求错误



测试重定向：

在Build里

**临时重定向**

重定向的原理

重定向要带全路径







