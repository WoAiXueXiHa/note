## 一、UDP协议：直肠子的“暴力美学”

UDP（User Datagram Protocol）在传输层的作用非常直接：它负责将应用层的数据封装，从发送端“搬运”到接收端，不做过多的干涉。

### 1.1 内核视角：报头与封装

在内核视角下，协议就是**结构体**。所谓的“封装”，本质上是：**移动指针 + 内存拷贝**。

Linux 内核中 `struct udphdr` 定义如下（`include/linux/udp.h`）：

```cpp
struct udphdr {
    __be16 source;  // 16位源端口号
    __be16 dest;    // 16位目的端口号
    __be16 len;     // UDP总长度（报头+数据）
    __be16 check;   // 校验和
};
```

* **如何解包？** OS 直接读取报文的前 **8个字节**，剩下的即为有效载荷。
* **如何分用？** 依靠 **16位目的端口号**。内核根据这个端口号，将数据推送到绑定了该端口的应用层进程的接收缓冲区中。

### 1.2 生产环境痛点：64K 限制与 IP 分片

这是 UDP 开发中最大的坑。

* **结构限制**：`len` 字段是 16 位的，意味着 UDP 报文最大长度是 $2^{16} - 1 = 65535$ 字节（约 64KB）。
* **分片危机**：如果你发送的数据超过了链路层的 **MTU**（通常 1500 字节）：
  * IP 层会进行**分片 (Fragmentation)**。
  * **后果**：一旦某个分片丢失，IP 层无法重组，整个 UDP 报文会被丢弃。应用层没有重传机制，对此一无所知。
  * **最佳实践**：在应用层控制数据包大小。推荐 **1472 字节**（1500 - 20字节IP头 - 8字节UDP头），确保不发生 IP 分片。

### 1.3 缓冲区与“零拷贝”

* **发送缓冲区？不存在的。**

  UDP 没有真正意义上的发送缓冲区。调用 `sendto` 后，数据直接交给内核网络层。因为 UDP 不保证可靠性（无确认、无重传、无流控），它**只管发**，这正好保留了全双工的特性。

* **接收缓冲区：**

  有，但如果满了，后续到达的报文会被直接丢弃（**丢包**）。且不保证顺序。

------

## 二、TCP协议：OS 掌管的精密机器

TCP（Transmission Control Protocol）极其复杂，因为它把数据包的处理权（何时发、发多少、出错咋办）全权交给了操作系统。

### 2.1 报头深挖：那个神秘的“4位首部长度”

TCP 报头标准长度为 20 字节，但含有扩展选项。

**核心字段解析（源码 `include/linux/tcp.h`）：**

```cpp
struct tcphdr {
    __be16 source;
    __be16 dest;
    __be32 seq;
    __be32 ack_seq;
    // ...
    __u16 doff:4; // Data Offset (首部长度)
    // ...
};
```

* **doff (Data Offset)**：
  * 单位是 **4字节**。
  * 只有 4 bit，最大值 15。这意味着 TCP 头部最大长度 = $15 \times 4 = 60$ 字节。
  * **必考点**：标准头 20 字节，意味着**选项 (Options) 最多只能有 40 字节**。

### 2.2 可靠性深挖：累积确认 (Cumulative ACK)

你理解了“ACK是告诉对方收到了”，但内核实现得更聪明。

* **确认序号** = 序号 + 1。含义是：“我收到了该序号之前的所有数据”。
* **场景推演**：
  1. 服务端发送了包 1001-2000，ACK 2001 丢失。
  2. 服务端发送了包 2001-3000，客户端回了 ACK 3001。
  3. **结果**：发送端收到 ACK 3001 后，**不需要重传** 1001-2000。因为 ACK 3001 隐含表示“3001 之前的数据我都收到了”。
  4. 这大大减少了不必要的重传。

### 2.3 流量控制深挖：零窗口死锁与探测

TCP 利用 **16位窗口大小**（接收方缓冲区剩余空间）来控制发送速率。

* **死锁危机**：
  1. 接收方缓冲区满，发回 `Window = 0`，发送方暂停。
  2. 接收方应用读走数据，缓冲区空了，发一个 `Window = 4096` 的更新报文。
  3. **如果这个更新报文丢了？** 发送方以为是 0 在等，接收方以为通知了在等，形成死锁。
* **内核破局：坚持定时器 (Persist Timer)**
  * 发送方收到 `Window = 0` 时，启动定时器。
  * 定期发送一个 **1字节的探测包 (Window Probe)**。
  * 接收方必须对探测包回应当前的窗口大小，从而打破死锁。

### 2.4 内核黑科技：`sk_buff`

你提到的“缓冲区指针管理”，在 Linux 内核中就是 `struct sk_buff`。

* **原理**：数据从网卡上来，内核并不是一次次拷贝数据，而是传递 `sk_buff` 指针。
* **零拷贝**：封装协议头（如添加 TCP/UDP 头）时，内核只是移动 `data` 指针，在预留的 `headroom` 里写入头部，而不是搬运整个数据负载。这是 Linux 网络高性能的核心。

------

## 三、连接管理：握手与挥手的底层逻辑

### 3.1 标志位与状态机

* **SYN**：同步，请求建立连接。
* **ACK**：确认，大部分报文此位都为1。
* **FIN**：结束，请求断开。

### 3.2 三次握手 (The 3-Way Handshake)

* **本质**：
  1. **验证全双工信道**：Client 发 SYN 验证自己发和 Server 收；Server 发 SYN+ACK 验证自己发和 Client 收。
  2. **为什么是三次？** 其实是四次（SYN -> ACK -> SYN -> ACK），但 Server 将中间的 ACK 和 SYN **捎带应答**合并了。

### 3.3 四次挥手与状态深挖

由于 TCP 是全双工的，断开需要双方互发 FIN。

**两个让开发者头秃的状态：**

1. **CLOSE_WAIT (被动关闭方)**
   * **触发**：收到对方的 FIN，内核自动回了 ACK，但**应用层没有调用 `close()`**。
   * **危害**：如果服务器出现大量 `CLOSE_WAIT`，说明代码有 Bug（忘记关文件描述符），会耗尽服务器 fd 资源。
2. **TIME_WAIT (主动关闭方)**
   * **触发**：主动调用 `close()` 的一方，发完最后一个 ACK 后进入。
   * **时长**：**2MSL** (Maximum Segment Lifetime，通常 60s)。
   * **为什么是 2MSL？**
     * **理由1**：保证最后一个 ACK 被对方收到。如果 ACK 丢了，对方会重传 FIN。2MSL 允许“去程 ACK 丢失 + 回程 FIN 重传”的时间。
     * **理由2**：等待网络中游离的旧报文消散，防止新连接混淆。

------

## 四、代码实战：复现 CLOSE_WAIT

空谈误国，代码兴邦。我们编写一个简单的 TCP Server，**特意不调用 close**，模拟资源泄露。

### 4.1 服务端代码 (Server.cpp)

```cpp
#include <iostream>
#include <cstring>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>

int main() {
    int listen_sock = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in local;
    memset(&local, 0, sizeof(local));
    local.sin_family = AF_INET;
    local.sin_port = htons(8081);
    local.sin_addr.s_addr = INADDR_ANY;

    // 允许地址复用，解决 TIME_WAIT 导致的 Bind Error
    int opt = 1;
    setsockopt(listen_sock, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    bind(listen_sock, (struct sockaddr*)&local, sizeof(local));
    listen(listen_sock, 5);

    std::cout << "Server listening on 8081..." << std::endl;
    struct sockaddr_in client;
    socklen_t len = sizeof(client);
    // 接受连接
    int service_sock = accept(listen_sock, (struct sockaddr*)&client, &len);
    
    std::cout << "Client connected. I will NOT close the socket to simulate bug." << std::endl;
    
    // 模拟服务端繁忙或逻辑 BUG，死循环不 close
    while(true) {
        sleep(1); 
    }
    // 正常应该调用 close(service_sock);
    return 0;
}
```

### 4.2 客户端代码 (Client.cpp)

```cpp
#include <iostream>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

int main() {
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in server;
    server.sin_family = AF_INET;
    server.sin_port = htons(8081);
    inet_pton(AF_INET, "127.0.0.1", &server.sin_addr);

    if (connect(sock, (struct sockaddr*)&server, sizeof(server)) == 0) {
        std::cout << "Connected. Closing immediately..." << std::endl;
        close(sock); // 主动断开，发送 FIN
    }
    return 0;
}
```

### 4.3 现象观测

1. 运行 Server，再运行 Client。

2. 使用 `netstat -natp | grep 8081` 查看。

3. **结果**：

   Plaintext

   ```
   tcp ... 127.0.0.1:8081 ... CLOSE_WAIT 12345/./server
   ```

   **验证成功**：Server 卡在 `CLOSE_WAIT`，因为它没调 `close`，发不出第三次挥手的 FIN。

------

## 五、思维导图 (Markdown)

Markdown

```
# 传输层深度解析 (TCP/UDP)

## UDP (User Datagram Protocol)
### 核心特性
- **无连接**: 知道 IP:Port 即发
- **不可靠**: 无重传、无确认
- **缓冲区**: 无发送缓冲(直接给内核)，有接收缓冲(会丢包)
### 开发痛点
- **64K 限制**: 报头长度字段 (16bit) 限制
- **IP 分片**: 建议包长 < 1472 字节，避免分片导致整包丢失
- **应用场景**: 直播、QUIC、局域网广播

## TCP (Transmission Control Protocol)
### 报头解密
- **4位首部长度**: 决定 Header 最大 60 字节 (选项最多 40 字节)
- **sk_buff**: 内核零拷贝核心，移动指针添加报头
### 可靠性与流控
- **累积确认**: ACK N 代表 N 之前全收到 (防 ACK 丢包)
- **流量控制**: 滑动窗口 (Window Size)
- **死锁破解**: 坚持定时器 (Window Probe) 解决零窗口死锁
### 连接管理 (重点)
- **三次握手**: 验证全双工，捎带应答 (SYN+ACK)
- **四次挥手**:
    - **CLOSE_WAIT**: 被动方没调 close，资源泄露预警
    - **TIME_WAIT**: 主动方等待 2MSL
        - 1. 防最后一个 ACK 丢失
        - 2. 等待旧报文消散
```