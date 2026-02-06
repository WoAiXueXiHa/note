# 封装地址转换

计算机网络里最麻烦的是“语言不通”

* 用户想看`192.168.1.1`字符串和`8080`整数
* 网络**只认识大端字节序的二进制**
* Socket接口只认识`struct sockaddr`的通用结构体

可以封装一个类，把这些乱七八糟的格式统一转换，随时取用

```cpp
#pragma once

#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <netinet/in.h>
#include <string>
#include <cstring>
#include "Common.hpp"

// [本地 Host]           [转换函数]            [网络/Socket Network]
// string "1.2.3.4"  --> inet_pton/inet_addr --> sin_addr (大端整数)
// uint16_t 8080     --> htons               --> sin_port (大端端口)
class InetAddr{
private:
    // 网络字节序转主机字节序 ntohs 
    void PortNet2Host() { _port = ::ntohs(_net_addr.sin_port); }
    // 网络IP转点分十进制字符串
    void IpNet2Host(){
        char ipbuffer[64];
        const char* ip = ::inet_ntop(AF_INET, &_net_addr.sin_addr,
                                     ipbuffer, sizeof(ipbuffer));
        _ip = ipbuffer;   
    }
public:
    InetAddr() {}
    ~InetAddr() {}

    // 场景1：接收到网络中的一个连接，用对方的sockaddr_in初始化这个类
    InetAddr(const struct sockaddr_in& addr)
        :_net_addr(addr)
        {
            PortNet2Host();
            IpNet2Host();
        }
    
    // 场景2：监听本地端口，传输到网络
    InetAddr(uint16_t port)
        :_port(port)
        ,_ip("")
        {
            _net_addr.sin_family = AF_INET;
            _net_addr.sin_port = ::htons(_port);
            _net_addr.sin_addr.s_addr = INADDR_ANY;
        }
    
    // 场景3：客户端连接指定的IP
    InetAddr(std::string ip = gip, uint16_t port = gport)
        :_ip(ip)
        ,_port(port)
        {
            memset(&_net_addr, 0, sizeof(_net_addr));
            _net_addr.sin_family = AF_INET;
            _net_addr.sin_port = ::htons(_port);
            _net_addr.sin_addr.s_addr = ::inet_pton(AF_INET, _ip.c_str(), &_net_addr.sin_addr);
        }
    
    bool operator==(const InetAddr& addr){
        return _ip == addr._ip && _port == addr._port;
    }
    // 为了C的原生接口要求->bind,accept,connect
    struct sockaddr* NetAddr() { return Conv(&_net_addr); }
    socklen_t NetAddrLen() { return sizeof(_net_addr); }

    std::string Ip() { return _ip; }
    uint16_t Port() { return _port; }
    std::string PrintAddrIp() { return Ip() + ":" + std::to_string(Port()); }
private:
    std::string _ip;          
    uint16_t _port;
    struct sockaddr_in _net_addr;
};
```

梳理一下思路：

> * 三个视角：
>
>   * 本地视角给人看
>     * 形式：`string _ip`点分十进制，`uint16_t _port`16位整数
>     * 代码体现：成员变量，`Ip()` `Port()`
>   * 网络视角给网线传输
>     * 形式：二进制比特流
>     * 特点：大端字节序
>     * 代码体现：`_net_addr.sin_port` `_net_addr.sin_addr`
>   * Socket内核视角给系统看
>     * 形式`struct sockaddr_in`(IPv4)或`struct sockaddr`(通用接口)
>     * 特点：内核只认识这个结构体，是描述组织起来的网络视角数据
>     * 代码体现：`_net_addr`结构体本身、`NetAddr()`返回的指针
>
> * 实现思路：
>
>   ![image-20260205111814807](https://gitee.com/binary-whispers/pic/raw/master///20260205111816824.png)

# 公用资源头文件

```cpp
#pragma once

#define Conv(v) (struct sockaddr*)(v)

static const int gport = 8082;
static const int gfd = -1;
static const std::string gip = "127.0.0.1";

enum STATUS_INFO{
    SOCKET_ERR = 1,
    BIND_ERR,
    LISTEN_ERR,
    ACCEPT_ERR
};
```



# 服务器头文件

把服务器比作一家饭店，门口有迎客的招待员，店内有服务的服务员

* `_linstensockfd`：迎宾员，只负责拉客，招呼进店
* `sockfd`：专门负责服务客人点菜（读数据），上菜（写数据）
* 注意：**迎宾员只有一个，服务员可以有成千上万个**->后续解释TODO

还有`bool _isrunning;`

这三个变量是`TcpServer`类的成员变量

## 网络核心动作

```cpp
#pragma once
#include <istream>
#include <string>
#include <cerrno>
#include "InetAddr.hpp"
#include "Common.hpp"
#include "Log.hpp"

#define BACKLOG 8
using namespace LogModule;

class TcpServer{
public:
    // -----------------------------------------------------
    // 服务器心脏
    // socket->bind->listen->accept->recv/send
    // -----------------------------------------------------
    TcpServer(int port = gport, int listenfd = gfd)
        :_port(port)
        ,_listensockfd(listenfd)
        ,_isrunning(false)
        {}
    
    void InitServer(){
        // 1. Socket 创建套接字
            // AF_INET:IPv4
            // SOCK_STREAM:TCP协议
            // 返回值：_listensockfd迎宾员，唯一入口
            _listensockfd = ::socket(AF_INET, SOCK_STREAM, 0);
            if(_listensockfd < 0){
                LOG(FATAL) << "Socket create err! " << strerror(errno);
                exit(SOCKET_ERR);
            }
            LOG(INFO) << "Socket create success! fd: " << _listensockfd;
            
            // 1.5 端口复用
            // 允许服务器重启之后使用之前的端口，不用等TIME_WAIT结束
            // int opt = 1;
            setsockopt(_listensockfd, SOL_SOCKET, SO_REUSEADDR | SO_REUSEPORT, &opt, sizeof(opt));
        
            // 2. Bind 挂招牌
            // 生成绑定所需要的结构体
            InetAddr local(_port);
            // 非常重要！！！Bind之前必须要清空结构体
            memset(&local, 0, sizeof(local));
            int n = ::bind(_listensockfd, local.NetAddr(), local.NetAddrLen());
            if(n < 0){
                LOG(FATAL) << "Bind create err! " << strerror(errno);
                exit(BIND_ERR);
            }
            LOG(INFO) << "Bind success! Port: " << _port;

            // 3. Listen 拉客
            // TCP是面向连接的，要求随时随地等待连接
            // 门口排队超过8人，劝退后面来的
            n = ::listen(_listensockfd, BACKLOG);
            if(n < 0){
                LOG(FATAL) << "Listen err! " << strerror(errno);
                exit(LISTEN_ERR);
            }
            LOG(INFO) << "Listen success! Waiting for connections...";
    }
    
    void Start(){
        _isrunning = true;
        while(_isrunning){
            // 4. Accept 服务一个客人
            // 这是一个阻塞函数，如果没有人来，程序阻塞等待
            // 一旦有人来，内核会从队列里取出一个来，新建一个socket返回给你
            struct sockaddr_in peer;
            socklen_t peerlen = sizeof(peer);
            // 注意：这里返回的是服务员！客人已经进店！
            // 以后和客户沟通，全靠这个new_sockfd！
            int new_sockfd = ::accept(_listensockfd, Conv(&peer), &peerlen);
            if(new_sockfd < 0){
                LOG(WARNING) << "Accept err, continue...";
                // 注意，这里一个客人服务失败，无所谓，直接下一个客人就好！
                continue;
            }
            InetAddr clientAddr(peer);
            LOG(INFO) << "Accept success! Client: " << clientAddr.AddrIp()
                        << ", new sockfd: " << new_sockfd;
        
        // 5. 提供服务
        Service(new_sockfd);
        }
    }

    ~TcpServer(){
        if(_listensockfd >= 0)
            ::close(_listensockfd);
    }
private:
    int _listensockfd;      // 迎宾员
    int _port;
    bool _isrunning;
};
```

梳理一下步骤，这是所有网络编程的套路：

我们按照开店营业的流程类比：

> * **创建套接字->申请资源`socket()`**
>   * 本质：在内核里申请一块内存结构`struct socket`，并返回一个整数索引`fd`
>   * 状态：这时只是一个普通文件，没有名字，也不能联网
>   * 就好像是一个空房子，啥也没有
> * **绑定->挂招牌`bind()`**
>   * 本质：**把IP地址+端口号写入申请的内核结构中**
>   * 校验：OS会检查这个端口是否被占用，若被占用，`bind`会失败
>   * 在房子门口挂上自己的地址
> * **监听拉客->开启被动模式`listen`**
>   * 本质：把`Socket`的属性从“主动去连别人”改成“被动”（等着被别人连），同时在内核中开辟两个队列（半连接、全连接）--->后面讲两个队列TODO
>   * `BackLog`：指定全连接队列的长度
>   * 打开大门，搬出排队用的椅子
> * **连接、服务->叫号入座`accept()`**
>   * 阻塞：默认情况下，队列里没人，`accept`会让当前线程挂起，直到有人来才会被唤醒
>   * 分裂：它返回时，会**克隆**出一个全新的``Socket`(`new_sockfd`)
>     * 旧的：`_listensockfd`，继续回死循环的第一行，迎接下一位客人
>     * 新的：`new_sockfd`,记录了这次连接的信息（客人的IP，客人的端口，我的IP，我的端口，TCP），专门用来传输数据

## 单线程阻塞版本

```cpp
// 单线程服务版本 v0
void Service(int sockfd){
    char buffer[1024];
    while(1){
        // 1> 读数据
        // 和read类似
        ssize_t n = ::recv(sockfd, buffer, sizeof(buffer) - 1, 0);
        if(n > 0){  // 读到数据，处理字符串
            buffer[n] = '\0';
            LOG(INFO) << "Client[" << sockfd << "] says: " << buffer;
            // 2> 处理数据，简单回显
            std::string response = "Server Echo: " + std::string(buffer);
            // 3> 发数据
            ::send(sockfd, response.c_str(), response.size(), 0);
        } else if(0 == n) {     // 读到文件结尾，对方下线了
            LOG(INFO) << "Client[" << sockfd << "] disconnected.";
            break;
        } else{     // 出错了
            LOG(WARNING) << "recv err!";
            break;
        }
    }

    // 6. 结束服务
    // 文件描述符是有用的有限的资源
    // 不关闭可能导致错误使用或fd泄漏！！！
    ::close(sockfd);
    LOG(INFO) << "Connection closed. fd: " << sockfd;
}
```

运行：

![image-20260204155245777](https://gitee.com/binary-whispers/pic/raw/master///20260204155247997.png)

但是这个模式存在两个问题：

* 只能启动一个客户端

  ![image-20260204155525461](https://gitee.com/binary-whispers/pic/raw/master///20260204155528966.png)

  优化迭代版本，这个版本是只能服务一个客户端

* 关掉服务器重新打开发现不行

  ![image-20260204155742022](https://gitee.com/binary-whispers/pic/raw/master///20260204155743880.png)

  注意看，我注释了一部分代码：

  ![image-20260204155811229](https://gitee.com/binary-whispers/pic/raw/master///20260204155814189.png)

  现在将代码放开重新运行：

  ![image-20260204155957159](https://gitee.com/binary-whispers/pic/raw/master///20260204155959012.png)

## 多进程并发服务器版本

在运行函数中添加：

```cpp
struct sockaddr_in peer;
            socklen_t peerlen = sizeof(peer);
           
            // 主进程在这里卡住，等待连接
            // 注意：这里返回的是服务员！客人已经进店！
            // 以后和客户沟通，全靠这个new_sockfd！
            int new_sockfd = ::accept(_listensockfd, Conv(&peer), &peerlen);
            if(new_sockfd < 0){
                LOG(WARNING) << "Accept err, continue...";
                // 注意，这里一个客人服务失败，无所谓，直接下一个客人就好！
                continue;
            }
            InetAddr clientAddr(peer);
            LOG(INFO) << "Accept success! Client: " << clientAddr.AddrIp()
                        << ", new sockfd: " << new_sockfd;
        
        // 5. 提供服务
        // ==========================================================
        // Version 1: 多进程版核心逻辑 (Fork)
        pid_t id = fork();
        if(0 == id){
            // 子进程
            LOG(INFO) << "Child process working, pid: " << getpid();
            // 子进程不需要监听，但是基础了父进程的_listensockfd
            ::close(_listensockfd);
            if(fork() > 0) exit(0); //子进程退出
            // 孙子进程 -> 孤儿进程 -> 1
            Service(new_sockfd);
            exit(0);
        } else if(id > 0){
            // 父进程
            // 父进程只需要拉客
            ::close(new_sockfd);
            // 不会阻塞
            int id = ::waitpid(id, nullptr, 0);
            if(id < 0)
            {
                LOG(WARNING) << "Waitpid err！";
            }
        } else {
            LOG(WARNING) << "Fork err!";
            ::close(new_sockfd);
        }
        // ==========================================================

```



运行：

![image-20260204163335283](https://gitee.com/binary-whispers/pic/raw/master///20260204163337391.png)

发现可以启动多个客户端

![image-20260204163422170](https://gitee.com/binary-whispers/pic/raw/master///20260204163424455.png)

```txt
PPID     PID    ...   COMMAND
3979547  4065873 ...   ./tcp_server  <-- 【爷爷】(主进程)
      1  4065923 ...   ./tcp_server  <-- 【孙子1】(服务员) PPID是1！
      1  4066040 ...   ./tcp_server  <-- 【孙子2】(服务员) PPID是1！
```

变成了孤儿进程执行

**梳理思路**

> * 主进程（爷爷）-> 接客
>
>   * 动作：`accept`返回一个新的`new_sockfd`
>   * 思考：我只负责接客，服务客人我不能自己去（会被阻塞），我得找个人去
>   * 操作：`fork()`->第一次`fork`
>
> * 爷爷和爸爸的分工：
>
>   * 爷爷
>
>     * 不用干活，只用等爸爸安排好yiqie
>     * 调用`waitpid(id,...)`
>     * 关键：爸爸进去之后立刻`fork`并自杀，所以这个`waitpid`**几乎不耗时，瞬间返回**
>     * 爷爷立刻回到`accept`去接下一个客人
>
>   * 爸爸
>
>     * 职责：**制造一个真正干活的人，然后立刻消失**
>
>     * 操作：`if (fork() > 0) exit(0);`
>
>     * 为什么这么做？
>
>       * ->让`waitpid`能立刻返回，不阻塞接客动作
>       * ->产生孤儿进程，脱离爷爷的管理，交给OS管理，无需再次等待
>
>       

思考：进程创建代价还是太大，能不能多线程？

## 多线程并发服务器版本

**和多进程的区别：资源共享**

> * 上个版本，父子进程之间的内存是**隔离**的
> * 这个版本，所有线程**共享**同一个进程的地址空间
>   * 优势：创建快，不用复制内存，切换快
>   * 风险：两个线程同时抢一个变量，数据就乱了->加锁

* 定义传参结构体：

  ```cpp
  // 内部类：给线程传参
  struct ThreadData{  
      int _sockfd;            
      TcpServer* self;         
  
      ThreadData(TcpServer* t, int fd)
          :self(t)
              ,_sockfd(fd)
          {}
  };
  ```

  `self`指针：调用`Service`方法

  `_sockfd`：确定服务哪个客户端

* 编写线程入口函数

  这是一个**静态函数**，没有`this`指针，满足`pthread_create`的要求

  ```cpp
  static void* ThreadEntry(void* args){
      // 1. 线程分离
      // 临时工干完活自己走人，不需要回收
      pthread_detach(pthread_self());
      // 2. 拆包
      ThreadData* td = static_cast<ThreadData*>(args);
      // 3. 干活
      td->self->Service(td->_sockfd);
      // 4. 清理堆内存
      delete td;
  
      return nullptr;
  }
  ```

* 重写`Start`函数->主线程发任务

  ```cpp
  // Version 2: 多线程版 (pthread_create)
  // 必须是static 没有this指针
  // 1. 打包参数
  // 为什么不能用局部变量？
  // 主线程循环极快，进入下一次循环时，局部变量就被销毁或覆盖了！
  ThreadData* data = new ThreadData(this, new_sockfd);
  // 2. 创建线程
  pthread_t tid;
  int n = pthread_create(&tid, nullptr, ThreadEntry, data);
  if(0 != n){
      LOG(WARNING) << "Create thread err!";
      ::close(new_sockfd);
      delete data;
  }
  ```

运行：

![image-20260204173045638](https://gitee.com/binary-whispers/pic/raw/master///20260204173048560.png)

```txt
PID      LWP     CMD
4081151  4081151 tcp_server  <-- 主线程 (迎宾员)
4081151  4081314 tcp_server  <-- 线程 A (服务员 1)
4081151  4081390 tcp_server  <-- 线程 B (服务员 2)
```

**流程总结**

> #### 1. 核心逻辑
>
> * **主线程 (Acceptor)**：
>   * 只负责坐在死循环里 `accept`
>   * 每接到一个客人（`sockfd`），立刻**招募一个临时工**（`pthread_create`）
>   * 把客人交给临时工后，主线程立刻不管了（`detach`），回到门口接下一个
> * **工作线程 (Worker)**：
>   * 拿了任务包（`ThreadData`），开始陪客人聊天（`Service`）
>   * 聊完了，客人走了（`close`），工作线程也就**原地解散**了（`delete data` + 线程退出）
>
> #### 2. 相比多进程版的优势
>
> * **轻量**：创建线程比创建进程快得多（不用复制页表、文件描述符表等）
> * **通信方便**：线程间共享内存，如果以后要做“群聊功能”，线程之间交换数据非常容易

**缺陷分析**

> * 创建和销毁的开销依旧很大
>   * 现象：来一个连接，`new`一个线程，连接断开，`delete`一个线程
>   * 比喻：餐厅来一桌客人，都要招一个新服务员，客人离开立马辞退服务员
>   * 代价：CPU大量时间都在招人和裁员的**系统调用**上
> * 资源没有上限，容易崩溃
>   * 现象：如果瞬间来了5w个用户，系统会尝试创建5w个线程
>   * 代价：内存耗尽、CPU调度崩溃

# 客户端和服务器源文件

客户端：

```cpp
#include <iostream>
#include <string>
#include <cstring>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include "Log.hpp"
#include "Common.hpp"

using namespace LogModule;

#define SERVER_IP "127.0.0.1"
#define SERVER_PORT 8082

int main(int argc, char* argv[]){
    std::string ip = SERVER_IP;
    int port= SERVER_PORT;

    if(3 == argc){
        ip = argv[1];
        port = std::stoi(argv[2]);
    }

    // 客户端创建套接字
    int sockfd = ::socket(AF_INET, SOCK_STREAM, 0);
    if(sockfd < 0){
        LOG(FATAL) << "Create socket err!";
        exit(SOCKET_ERR);
    }

    // 填写服务器的地址信息
    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(port);
    server_addr.sin_addr.s_addr = inet_addr(ip.c_str());

    // 发起连接
    // 客户端需要bind吗？
    // 不需要显式bind，客户端发起连接的适合，OS回随机分配一个端口给客户端
    int n = ::connect(sockfd, Conv(&server_addr), sizeof(server_addr));
    if(n < 0){
        LOG(FATAL) << "Connect err! Is the server running?";
        exit(CONNECT_ERR);
    }

    // 业务循环，聊天
    while(1){
        std::cout << "Please Enter# ";
        std::string msg;
        std::getline(std::cin, msg);

        if(msg == "quit") break;
        
        // 1> 写数据
        ssize_t n = ::write(sockfd, msg.c_str(), msg.size());
        if(n > 0){
            // 2> 读回显
            char buffer[1024];
            memset(buffer, 0, sizeof(buffer));

            // 阻塞等待服务器回应
            ssize_t m = ::read(sockfd, buffer, sizeof(buffer) - 1);
            if(m > 0){
                buffer[m] = '\0';
                LOG(INFO) << "Server Echo: " << buffer;
            }
            else if(0 == m){
                LOG(INFO) << "Server closed connection.";
                break;
            } else {
                LOG(WARNING) << "Client read err!";
                break;
            }
        } else {
            LOG(WARNING) << "Client write err!";
            break;
        }
    }
    // 关闭连接
    ::close(sockfd);
    return 0;
}
```

服务器：

```cpp
#include "TcpServer.hpp"
#include <memory>

using namespace LogModule;

int main(){
    // 1. 实例化服务器对象
    std::unique_ptr<TcpServer> tsvr(new TcpServer(8082));
    // 2. 初始化服务器
    tsvr->InitServer();
    // 3. 启动服务
    tsvr->Start();

    return 0;
}
```

