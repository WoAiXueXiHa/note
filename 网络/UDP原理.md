# 一、通信基石：端口和五元组

**什么是五元组**

在TCP/IP协议中，**标识一个唯一的通信连接需要五个元素：**

* 源IP
* 源端口号
* 目的IP
* 目的端口号
* 协议号（如TCP或UDP）

在Linux中，可以使用`netstat -n`命令查看这些信息

**端口号划分**

* 知名端口号`0~1023`
* 私有端口号：`1024~65535`

**一个进程可以绑定多个端口号吗？**

> 可以，一个进程可以创建多个Socket文件描述符，每个Socket都可以绑定一个独立的端口

**一个端口号可以被多个进程绑定吗？**

> 默认不行，有特例
>
> * 若进程A绑定了8080，进程B再绑定8080，OS会抛出”地址被使用“错误
> * 特例：如果使用了`setsockopt`设置了`SO_REUSEADDR`或`SO_REUSEPORT`

# 二、 UDP报文结构

## UDP协议头格式

UDP头部固定**8个字节**，协议就是结构体：

在内核视角下：

```cpp
struct udphdr {
    uint16_t source;  // 16位源端口号
    uint16_t dest;    // 16位目的端口号
    uint16_t len;     // UDP长度（报头+数据）
    uint16_t check;   // 校验和
};
```



| **字段**       | **长度** | **作用**                            |
| -------------- | -------- | ----------------------------------- |
| **源端口号**   | 16位     | 标识发送进程                        |
| **目的端口号** | 16位     | 标识接收进程 (用于分用)             |
| **UDP长度**    | 16位     | 表示整个数据报(首部+数据)的最大长度 |
| **UDP校验和**  | 16位     | 如果校验和出错，数据会被直接丢弃    |

**怎么解包？**

> OS读取报文的**前8个字节**，剩下的内容就是有效载荷

**怎么分用？**

> 基于**16位目的端口号**，内核根据这个端口号，把数据推送绑定了该端口的应用层进程的接收缓冲区当中

## UDP的痛点：16位长度的限制

UDP长度只有16位，则为$2^{16} - 1= 65535$，这意味着一个UDP包（包含头部）最大只能是**64KB**

**如果UDP要上传大于64KB的数据怎么办？**

> 必须在**应用层**手动分包，程序员自己实现把大文件切成小块，发送过去，再接收拼装



## UDP特点

* 面向数据报

  * 定义：应用层交给UDP多长的报文，UDP原样转发，**不拆分，也不合并**

* 缓冲区机制：

  * **发送缓冲区：** UDP没有真正意义上的缓冲区，调用`sendto`会直接交给内核，内核拷贝给网卡驱动

  * **接收缓冲区：**UDP有接收缓冲区

    * 接收缓冲区不能保证收到数据包的顺序和发送顺序一致
    * 丢包：如果接收缓冲区满了，新到达的数据就会**直接被丢弃**

  * **怎么做到数据转发？**

    > 应用层调用`sendto`发数据时：
    >
    > 1. 数据从应用层拷贝到传输层
    > 2. 内核在数据前添加UDP报头
    > 3. 继续交给下层处理

总结：

* 面向数据报：上层给我什么样，我就加个报头，不做任何处理向下层交付
* 无连接：: 知道对端的IP和端⼝号就直接进⾏传输, 不需要建⽴连接
* 不可靠：没有确认机制，没有重传机制，如果因为⽹络故障该段⽆法发到对⽅, UDP协议也不会给应⽤层返回任何错误信息



# 三、 UDP Socket编程demo

## 服务端

```cpp
#include <iostream>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <cstring>
#include <unistd.h>

#define PORT 8080
#define BUFFER_SIZE 1024

int main(){
    // 1. 创建套接字 socket
    // AF_INET: IPv4
    // SOCK_DGRAM: UDP数据报类型
    // 0: 默认
    int sockfd = ::socket(AF_INET, SOCK_DGRAM, 0);
    if(sockfd < 0){
        std::cerr << "socket create failed: " << errno << std::endl;
        exit(-1);
    }
    std::cout << "socket create success! fd: " << sockfd << "\n";
    // 2. 填充服务器地址结构体
    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = ::htons(PORT);

    // 3. 绑定服务器 bind
    int n = ::bind(sockfd, (struct sockaddr*)&server_addr, sizeof(server_addr));
    if(n < 0){
        std::cerr << "bind failed: " << errno << std::endl;
        exit(-1);
    }
    std::cout << "UDP server is running on port " << PORT << std::endl;

    char buffer[BUFFER_SIZE];
    struct sockaddr_in client_addr;
    socklen_t len = sizeof(client_addr);
    // 4. 接收数据 recvfrom
    // 最后两个参数是输出型参数，内核会把"谁发的"填进去
    while(1){
        int n = ::recvfrom(sockfd, buffer, BUFFER_SIZE,
                           0, (struct sockaddr*)&client_addr, &len);
        if(n > 0) {
            buffer[n] = '\0';
            std::cout << "Clinet : " << buffer << std::endl;
        }
        else if(0 == n) {
            std::cout << "Received empty packet\n";
        } else {
            std::cerr << "recv err\n";
            break;
        }
        // 5. 发送回复 sendto
        const char* hello = "Hello from server";
        ::sendto(sockfd, hello, strlen(hello),
                 0, (struct sockaddr*)&client_addr, len);
    }  

    ::close(sockfd);

    return 0;
}
```

## 客户端

```cpp
#include <iostream>
#include <cstring>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>

#define PORT 8080
#define SERVER_IP "127.0.0.1" // 本地回环测试

int main() {
    // 1. 创建套接字 socket
    int sockfd = socket(AF_INET, SOCK_DGRAM, 0);
    if(sockfd < 0){
        std::cerr << "socket create failed: " << errno << std::endl;
        exit(-1);
    }
    std::cout << "socket create success! fd: " << sockfd << "\n";

    // 2. 填充目标服务器地址信息
    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = ::inet_addr(SERVER_IP);
    server_addr.sin_port = ::htons(PORT);

    // 3. 直接发送信息 sendto
    const char* hello = "Hello from client";
    ::sendto(sockfd, hello, strlen(hello),
             0, (struct sockaddr*)&server_addr, sizeof(server_addr));
    std::cout << "Msg sent\n";

    // 4. 接收回复
    char buffer[1024];
    socklen_t len = sizeof(server_addr);
    int n = ::recvfrom(sockfd, buffer, 1024,
                       0, (struct sockaddr*)&server_addr, &len);
    if(n > 0) {
        buffer[n] = '\0';
        std::cout << "server: " << buffer << std::endl;
    } 
    
    ::close(sockfd);
    return 0;
}
```

