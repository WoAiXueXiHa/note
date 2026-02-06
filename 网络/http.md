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



