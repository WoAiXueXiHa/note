# 1. 封装RAII风格的锁

在多线程并发中最基础、最重要的问题就是线程互斥

## 为什么需要锁？

> * 临界资源：多个线程都能看到的公共资源（全局变量、屏幕、容器等）
> * 临界区：操作公共资源的代码段
> * 竞态条件：多个线程同时进入临界区，就会发生争抢资源，导致数据错乱
> * 互斥锁：为了解决这个问题，引入锁
>   * 进门前：加锁，外面的线程阻塞等待
>   * 出门后：解锁，唤醒门口排队的线程

## 封装原生锁并且实现RAII

C风格的使用起来有几个痛点：

* 要手动初始化和销毁
* 容易忘记解锁导致死锁

在`Lock.hpp`中：

```cpp
#pragma once
#include <iostream>
#include <pthread.h>

namespace LockModule{
    // 1. 封装原生锁对象
    class Mutex{
    public:
        // ==========================================================
        // 重要！！！禁止拷贝！！！
        // 锁是系统资源，不允许被复制，如果复制了，两个对象管理同一个锁
        // 析构时会destroy两次，导致程序崩溃
        Mutex(const Mutex&) = delete;
        const Mutex& operator=(const Mutex&) = delete; 
        // ==========================================================

        Mutex(){
            int n = ::pthread_mutex_init(&_lock, nullptr);
            if(0 != n){
                std::cerr << "Mutex init failed!\n";
            }
        }

        ~Mutex() { ::pthread_mutex_destroy(&_lock); }
        void Lock() { ::pthread_mutex_lock(&_lock); }
        void Unlock(){ ::pthread_mutex_unlock(&_lock); }

        // 获取原生锁指针，条件变量会用到
        pthread_mutex_t* GetLock(){ return &_lock; }
    private:
        pthread_mutex_t _lock;
    };

    // 2. 封装RAII风格的锁
    class LockGuard{
    public:
        // 构造时自动加锁
        // 关键：必须传引用！！！！
        // 传值会发生拷贝，上面禁止拷贝了，那么编译器会报错
        // 即使允许拷贝，锁住的是副本，原来的锁根本没动，没起到保护作用！
        LockGuard(Mutex& mutex) : _mutex(mutex) { _mutex.Lock(); }

        // 析构时自动解锁
        ~LockGuard() { _mutex.Unlock(); }
    private:
        Mutex& _mutex;          // 必须是引用类型，直接修改本身而不是副本
    };
}
```

## 细节剖析

### 为什么要`LockGuard`？

* 不好的写法：

  ```cpp
  mutex.Lock();
  if (error) return; // 忘记 Unlock 了，死锁！
  // ... 业务逻辑
  mutex.Unlock();
  ```

* RAII写法：

  ```cpp
  {
      LockGuard lock(mutex); // 构造时 Lock
      if (error) return;     // 自动调用 lock 的析构 -> Unlock
      // ... 业务逻辑
  } // 出作用域，自动 Unlock
  ```

**让编译器帮我们管理资源的生命周期**

### 为什么 `LockGuard` 里的成员变量必须是 `Mutex&`？

* `Mutex _mutex;`，这是一个**对象成员**，初始化时，会把外面的锁**拷贝**一份进来，锁住的是家里钥匙的**复印件**，而家门用的还是**原件**，任何人都能进
* `Mutex& _mutex;`，这是一个**引用成员**，给外面的锁起个别名，操作的是同一把锁

# 2. 封装线程池

## 封装线程

在`Thread.hpp`中：

```cpp
#pragma once
#include <iostream>
#include <string>
#include <functional>
#include <pthread.h>

namespace ThreadModule{
    // 定义线程要执行的任务类型
    // void(std::string name):线程执行时，可以把线程的名字传给它，方便打印日志
    using func_t = std::function<void(std::string)>;

    class Thread{
    public:
        // 构造：只初始化，不启动
        Thread(func_t func, const std::string& name = "Thread-Base")
            :_func(func)
            ,_name(name)
            ,_is_running(false)
            {}

        // 启动线程
        void Start(){
            _is_running = true;
            // 传入Routine作为入口函数
            // 传入this指针作为参数，让静态函数找到对象
            int n = ::pthread_create(&_tid, nullptr, Routine, this);
            if(0 != n){
                std::cerr << "Thread create failed!\n";
                _is_running = false;
            }
        }

        // 等待线程结束
        void Join(){
            if(_is_running){
                ::pthread_join(_tid, nullptr);
                _is_running = false;
            }
        }

        std::string Name() const { return _name; }

        ~Thread(){
            // 析构时不要Join，由外部管理生命周期
        }
    private:
        // 核心：静态成员函数做跳板
        static void* Routine(void* args){
            // 1. 恢复this指针
            Thread* self = static_cast<Thread*>(args);
            // 2. 调用真正的业务
            self->_func(self->_name);
            return nullptr;
        }
    private:
        pthread_t _tid;
        std::string _name;
        func_t _func;           // 线程要执行的任务
        bool _is_running;
    };
}
```

**为什么要设计 `static void* Routine(void* args)` ？**

> * **根本冲突：C和C++的代沟**
>
>   Linux的线程库`pthread`是用C写的，`pthread_create`的函数要求如下：
>
>   ```cpp
>   // 第三个参数要求是：一个返回 void*，接收 void* 的函数指针
>   int pthread_create(..., void *(*start_routine) (void *), void *arg);
>   ```
>
>   而在C++中，定义的普通成员函数：
>
>   ```cpp
>   class Thread {
>       void Run() { ... } // 业务逻辑
>   };
>   ```
>
>   **所有的普通成员有一个隐藏的`this`指针参数：**
>
>   ```cpp
>   void Run(Thread* this); // 只有拿到 this，才能访问对象里的 _name, _tid
>   ```
>
>   发生冲突：
>
>   * `pthread_create` 想要：`void* func(void*)` （1个参数）
>   *  `Thread::Run` 其实是：`void Run(Thread* this)` （类型不兼容，且返回值也不对）
>
> * **解决：第一步、`static`修饰的成员函数无`this`指针**
>
>   ```cpp
>   class Thread {
>       // 加上 static，它就不属于任何对象了，它就变成了一个普通的全局函数，
>       // 只是作用域还在 Thread 类里面。
>       static void* Routine(void* args); 
>   };
>   ```
>
> * **解决：第二步、偷天换日**
>
>   编译器开心了，但我们傻眼了。 因为 `Routine` 是静态的，它没有 `this` 指针，它**看不到**对象里的 `_name`、`_func`、`_tid`。 如果它什么都访问不了，那它怎么执行任务？
>
>   这时候，`pthread_create` 的**第四个参数 `void \*arg`** 可以传任何类型参数！这个参数的设计初衷，就是给回调函数传数据的
>
>   1. 在`Start()`里打包
>
>      ```cpp
>      void Start() {
>          // 这里的 this，就是当前这个 Thread 对象（比如 main 函数里定义的 t1）
>          // 我们把 this 指针，强行塞给 pthread_create 的第四个参数
>          // void* 是万能指针，它可以接收任何类型的指针
>          pthread_create(&_tid, nullptr, Routine, this); 
>      }
>      ```
>
>      
>
>      动作：我把我自己（对象地址）打包发出去了
>
>   2. 在OS内核中运输快递
>
>      操作系统创建线程，准备调用 `Routine`。它手里攥着刚才传给它的第四个参数（也就是 `this` 指针），但它并不知道这是啥，它只知道这是个 `void*`
>
>   3. 在`Routine()`里拆包
>
>      这里的`args`就是刚才传进来的`this`
>
>      ```cpp
>      static void* Routine(void* args) { // args 其实就是 this
>          // 1. 既然我知道你是 Thread* 变过来的，我就把你变回去！
>          Thread* self = static_cast<Thread*>(args);
>                    
>          // 2. 见证奇迹的时刻
>          // 现在有了 self 指针，我又可以访问对象里的成员了！
>          // 虽然 Routine 是静态的，但 self 指针指向的是一个活生生的对象
>          self->_func(self->_name); 
>                    
>          return nullptr;
>      }
>      ```
>
> ```txt
> [ C++ 对象世界 ]              [ C 语言 API 世界 ]              [ 新线程内部 ]
>       |                             |                             |
>       |   (1) Start() 调用          |                             |
>    Thread -------------------> pthread_create                     |
>   (this指针)        (传参)     (函数 ptr, void* arg)              |
>       |                             |                             |
>       |                             | (2) 创建线程                 |
>       |                             | 携带 arg (即this)             |
>       |                             +-------------------------> Routine(void* arg)
>       |                                                           |
>       |                                                           | (3) 还原
>       |                                                 Thread* self = (Thread*)arg
>       |                                                           |
>       |             (4) 回调回来                                   |
>       +------------------------------------------------------- self->Run()
> ```

## 组装线程池

### 添加条件变量

我们有了锁（Mutex）用来**保护**队列，但我们还没有机制让线程在队列为空时**睡觉（Wait）**，在有任务时**醒来（Notify）**

如果不解决这个问题，线程池里的线程就会不停地检查队列（忙等待）， 我们需要 **条件变量 (Condition Variable)**

在`Lock.hpp`中添加：

```cpp
 class Cond{
    public:
        Cond(){
            int n = ::pthread_cond_init(&_cond, nullptr);
            if(0 != n){
                std::cerr << "Cond init failed!\n";
            }
        }

        ~Cond() { ::pthread_cond_destroy(&_cond); }

        // 等待
        // 等待时，必须把锁传进去！
        // why？pthread_cond_wait会在沉睡前自动释放锁
        // 醒来后重新竞争锁，如果不传锁，无法原子操作
        void Wait(Mutex& mutex){
            ::pthread_cond_wait(&_cond, mutex.GetLock());
        }

        // 唤醒一个线程
        void Notify(){ :: pthread_cond_signal(&_cond); }
        // 唤醒所有线程
        void NotifyAll() { ::pthread_cond_broadcast(&_cond); }

    private:
        pthread_cond_t _cond;
    };
```

### 组装线程池

`ThreadPool`的本质是：**一个任务队列+一堆工人线程+一把锁/条件变量**

```cpp
#pragma once
#include <iostream>
#include <vector>
#include <queue>
#include <memory>
#include "Thread.hpp"
#include "Lock.hpp"

using namespace ThreadModule;
using namespace LockModule;

// T是任务类型，未来传给它 std::function<void()> 或者 Task对象
template <typename T>
class ThreadPool{
private:
    // 消费者接口：线程工人
    void HandlerTask(std::string name){
        std::cout << name << " start working...\n";
        while(1){
            T task;
            {
                // 定义临界区
                LockGuard lock(_mutex);
            
                // 为什么用while不用if？
                // 防止虚假唤醒->有时候线程可能莫名其妙醒来（OS原因）
                // 或者被唤醒了但是任务被别人抢走了
                // 必须循环检查队列是否真的有货
                while(_task_queue.empty() && _is_running){
                    _cond.Wait(_mutex);     // 没货就睡觉
                }

                if(!_is_running && _task_queue.empty()) break;

                // 取任务
                task = _task_queue.front();
                _task_queue.pop();
            }
            // 出了括号，锁自动释放
            // 任务处理一定在锁外面！！！
            // 否则串行执行，多线程无意义

            // 执行任务
            task();
        }
    }
public:
    ThreadPool(int num = 5)
        :_thread_num(num)
        ,_is_running(false)
        {
            for(int i = 0; i < _thread_num; i++){
                // Thread需要一个 void(std::string)类型的安徽念书
                // HandlerTask是成员函数，有一个this指针参数
                // 用bind把this绑定死，编程一个可以直接调用的函数对象
                func_t func = std::bind(&ThreadPool::HandlerTask, this, std::placeholders::_1);

                _threads.push_back(new Thread(func, "Thread-" + std::to_string(i)));
            }
        }
    
    // 启动线程
    void Start(){
        _is_running = true;
        for(auto t : _threads){
            t->Start();
        }
    }

    // 停止线程池
    void Stop(){
        if(_is_running){
            _is_running = false;
            _cond.NotifyAll();
            for(auto t : _threads){
                t->Join();
            }
        }
    }

    // 生产者接口：塞任务
    void Enqueue(const T& task){
        LockGuard lock(_mutex);
        if(_is_running){
            _task_queue.push(task);
            _cond.Notify();
        }
    }
    
    ~ThreadPool(){
        Stop();
        for(auto t : _threads){
            delete t;
        }
    }
private:
    int _thread_num;
    bool _is_running;
    std::vector<Thread*> _threads;          // 线程容器
    std::queue<T> _task_queue;              // 任务队列
    Mutex _mutex;                           // 保护队列的锁               // 
    Cond _cond;                             // 协调生产消费的条件变量
};
```

### 细节剖析

#### 生产者-消费者模型

**仓库**：`std::queue<T> _task_queue` 是临界资源

**生产者**：外部调用 `Enqueue()` 的线程。它们负责生产任务，加锁放入队列，并按门铃（`Notify`）

**消费者**：线程池内部的 `HandlerTask` 线程。它们负责消费任务，没货就睡觉（`Wait`），有货就干活

**协调者**：`Mutex` 保证仓库不被同时踩踏，`Cond` 保证没货时不空转

#### 虚假唤醒

```cpp
// 正确写法
while(_task_queue.empty() && _is_running){
    _cond.Wait(_mutex);
}
```

**为什么必须用 `while` 而不能用 `if`？**

* **原因 1 (OS层面)**：在多核系统中，`pthread_cond_wait` 可能会在没有任何人 `Notify` 的情况下返回（系统调度原因）。如果是 `if`，线程醒来直接往下走去取任务，结果队列是空的，直接崩溃
* **原因 2 (竞争)**：假设 `Notify` 唤醒了线程A，但在A拿到锁之前，线程B抢先拿到了锁并把队列里的唯一一个任务取走了。等A拿到锁进去一看，队列又空了
* **结论**：醒来后的第一件事，必须是**再次检查条件**

# 3. 应用层协议-序列化和反序列化

我们使用JSON库，把它当成一个黑盒工具：

* 输入：传入结构体对象，如`x=10, y=20`
* 输出：一个标准格式的字符串，如`{"x":10, "y":20, "op":43}`

在Linux下安装`jsoncpp`库：`sudo apt-get install libjsoncpp-dev`

在`Protocol.hpp`中：

```cpp
#pragma once
#include <iostream>
#include <string>
#include <cstring>
#include <sys/types.h>
#include <sys/socket.h>
#include <jsoncpp/json/json.h>

namespace ProtocolModule{
    // =============================================
    // 第一层：解决TCP粘包问题
    // ---------------------------------------------
    // TCP面向字节流
    // 像是自来水厂直接一次性给你提供很多水
    // 你作为用户，可以接一杯，可以接一桶
    // 所以我们需要人为规定：多长才算一个完整的报文？
    // 格式规定为："长度\r\nJSON字符串\r\n"
    // =============================================

    //  封包：给数据加报头
    // 输入: "{"x":10...}"
    // 输出: "15\r\n{"x":10...}\r\n"
    std::string Encode(const std::string& json_str){
        std::string len_str = std::to_string(json_str.size());
        std::string packet = len_str + "\r\n" + json_str + "\r\n";
        return packet;
    }

    // 解包：从缓冲区里提取一个完整的报文
    // 返回值: 提取出的 JSON 字符串 (如果没有完整的包，返回空字符串)
    std::string Decode(std::string &buffer) {
        // 1. 找报头分隔符 "\r\n"
        size_t pos = buffer.find("\r\n");
        if (pos == std::string::npos) return ""; // 说明连长度都没收全，继续等

        // 2. 提取长度
        std::string len_str = buffer.substr(0, pos);
        int len = std::stoi(len_str);

        // 3. 计算本包总长度 = 长度字符串 + 2字节分隔符 + 数据长度 + 2字节分隔符
        int total_len = len_str.size() + 2 + len + 2;

        // 4. 检查缓冲区剩下的数据够不够一个包
        if (buffer.size() < total_len) return ""; // 数据不够，继续等

        // 5. 提取数据 (切掉报头和报尾)
        std::string json_str = buffer.substr(pos + 2, len);

        // 6. 从缓冲区移除这个已经提取的包
        buffer.erase(0, total_len);

        return json_str;
    }

    // ================================================================
    // 第二层：序列化与反序列化 (Serialize / Deserialize)
    // ----------------------------------------------------------------
    // 作用：把 C++ 的 struct 对象 <---> 字符串
    // 这里我们用 JSON 库来做这个转换
    // ================================================================

    // 请求结构体：客户端发给服务器的
    class Request {
    public:
        int _x;
        int _y;
        char _op; // '+', '-', '*', '/'

        Request() : _x(0), _y(0), _op(0) {}
        Request(int x, int y, char op) : _x(x), _y(y), _op(op) {}

        // 序列化: struct -> string
        bool Serialize(std::string *out) {
            Json::Value root;
            root["x"] = _x;
            root["y"] = _y;
            root["op"] = _op; // char 会被转成 ASCII 整数存储，问题不大

            // FastWriter 负责把对象转成字符串
            Json::FastWriter writer;
            *out = writer.write(root);
            return true;
        }

        // 反序列化: string -> struct
        bool Deserialize(const std::string &in) {
            Json::Value root;
            Json::Reader reader;
            // parse 负责把字符串解析回对象
            bool res = reader.parse(in, root);
            if (!res) return false;

            _x = root["x"].asInt();
            _y = root["y"].asInt();
            _op = (char)root["op"].asInt();
            return true;
        }
    };

    // 响应结构体：服务器回给客户端的
    class Response {
    public:
        int _result;
        int _code; // 0:成功, 1:除0错误, 2:非法操作

        Response() : _result(0), _code(0) {}
        Response(int res, int code) : _result(res), _code(code) {}

        // 序列化
        bool Serialize(std::string *out) {
            Json::Value root;
            root["result"] = _result;
            root["code"] = _code;

            Json::FastWriter writer;
            *out = writer.write(root);
            return true;
        }

        // 反序列化
        bool Deserialize(const std::string &in) {
            Json::Value root;
            Json::Reader reader;
            bool res = reader.parse(in, root);
            if (!res) return false;

            _result = root["result"].asInt();
            _code = root["code"].asInt();
            return true;
        }
    };
}
```

![image-20260205202412803](https://gitee.com/binary-whispers/pic/raw/master///20260205202415754.png)

####  序列化层 

* **矛盾**：内存里的 `struct` 是二进制的，里面可能有指针、内存对齐填充，直接发出去对方看不懂（对方可能是 Java 写的，或者 CPU 架构不同）
* **动作**：`Serialize` / `Deserialize`
* **奥义**：**“翻译”**，把“方言”（C++ 对象）翻译成“普通话”（JSON/XML/Protobuf 字符串）
  * *Client*: `Request` -> `{"x":10}`
  * *Server*: `{"x":10}` -> `Request`

#### 协议层 

* **矛盾**：TCP 是**流 (Stream)**。就像水管放水，你发了两个包，接收方可能收到的是“半个包”或者“两个半包粘在一起”。TCP 不管边界，它只管字节顺序
* **动作**：`Encode` / `Decode`
* **奥义**：**“断句”**，在普通话（Payload）外面包上一层信封（Header），告诉对方哪里开始，哪里结束
  * *策略*：通常是 `Length + \r\n + Body`
  * *Decode 的核心*：`while(buffer.size() > len)` 循环检查

#### 缓冲层 

* **矛盾**：网络是不确定的。你发了 100 字节，我可能先收到 10 字节，过了 1 秒才收到剩下 90 字节
* **动作**：`in_buffer += new_data`
* **奥义**：**“蓄水”**，不能读多少处理多少，必须先把水蓄在池子（Buffer）里，等水位够了（够一个完整包长度了），再舀出来处理

# 4. 搭建服务器

## 地址的管家 —— `InetAddr.hpp`

这个类的核心任务只有一个：**翻译**。

* 它负责把**人类能看懂的**（IP字符串、端口号）和**操作系统能看懂的**（大端序二进制结构体）互相转换

```cpp
#pragma once

#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <netinet/in.h>
#include <string>
#include <cstring>
#include "Common.hpp"

class InetAddr{
private:
    // 字节序转换：网络转主机
    // 网络传过来大端序，主机不管是小段还是大端，做个转换最保险
    // ntohs
    void PortNet2Host() { _port = ::ntohs(_net_addr.sin_port); }

    // IP格式转换 二进制->字符串
    // 网络传过来32位整数，要转成人看得懂的点分十进制字符串
    // inet_ntop
    void IpNet2Host(){
        char ipbuffer[64];
        ::inet_ntop(AF_INET, &_net_addr.sin_addr, ipbuffer, sizeof(ipbuffer));
        _ip = ipbuffer;
    }
public:
    // 场景1：作为接收方，accept时使用
    // OS accept 了一个连接，扔给你一块生肉sockaddr_in
    // 现在需要把生肉塞进InetAddr，翻译成人能懂的IP和Port
    InetAddr(const struct sockaddr_in& addr)
        :_net_addr(addr)
        {
            PortNet2Host();
            IpNet2Host();
        }
    
    // 场景2：作为发送方/监听方 bind/connect时用
    // 把人看得懂的IP和Port 填进 struct sockaddr_in
    InetAddr(std::string ip, uint16_t port)
        :_ip(ip)
        ,_port(port)
        {
            // 这一步很重要，防止脏数据
            memset(&_net_addr, 0, sizeof(_net_addr));
            _net_addr.sin_family = AF_INET;
            _net_addr.sin_port = ::htons(_port);
            ::inet_pton(AF_INET, _ip.c_str(), &_net_addr.sin_addr);
        }

    // 为了配合C接口bind/accept
    // 需要 struct sockaddr* 原生指针
    struct sockaddr* NetAddr() { return (struct sockaddr*)&_net_addr; }
    socklen_t NetAddrLen() { return sizeof(_net_addr); }
    
private:
    std::string _ip;
    uint16_t _port;
    struct sockaddr_in _net_addr;       // 系统原生的结构体
};
```

## 连接的经理：`TcpServer.hpp`

```cpp
#pragma once

#include <iostream>
#include <string>
#include <cstring>
#include <functional>
#include <unistd.h>
#include <signal.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>

#include "InetAddr.hpp"
#include "ThreadPool.hpp"

using namespace ThreadPoolModule;

const static int gport = 8080;
const static int gbacklog = 5;

// 业务层需要实现的接口形式
using Handler = std::function<void(int,InetAddr)>;

class TcpServer{
public:
    TcpServer(int port = gport)
        :_port(port)
        ,_listensock(-1)
        ,_is_running(false)
        {
            // 1. 创建线程池
            _threadpool = new ThreadPool<std::function<void()>>(5);

            // 2. 忽略SIGPIPE信号
            // 防止客户端断开连接时，服务器写入触发信号导致进程崩溃
            signal(SIGPIPE, SIG_IGN);
        }

    void InitServer(){
        // 1. 创建套接字
        _listensock = ::socket(AF_INET, SOCK_STREAM, 0);
        if(_listensock < 0){
            std::cerr << "[FATAL] Socket create failed!\n";
            exit(1);
        }

        // 2. 端口复用
        // 允许服务器重启后立刻绑定原来的端口
        int opt = 1;
        ::setsockopt(_listensock, SOL_SOCKET, SO_REUSEADDR | SO_REUSEADDR, &opt, sizeof(opt));

        // 3. 绑定
        InetAddr local(_port);
        if(::bind(_listensock, local.NetAddr(), local.NetAddrLen()) < 0){
            std::cerr << "[FATAL] Bind failed!\n";
            exit(2);
        }

        // 4. 监听
        if(::listen(_listensock, gbacklog) < 0){
            std::cerr << "[FATAL] Listen failed!\n";
            exit(3);
        }

        std::cout << "[INFO] Server init succss, Listening on port: " << _port << "\n";
    }

    // 核心循环： 负责accept连接并分发给线程池
    void Loop(Handler handler){
        _is_running = true;

        // 启动线程池
        _threadpool->Start();
        std::cout << "[INFO] ThreadPool started, waiting for connection...\n";

        while(_is_running){
            // 1. accept 拉客
            struct sockaddr_in peer;
            socklen_t peerlen = sizeof(peer);

            // 阻塞等待新连接
            int sockfd = ::accept(_listensock, (struct sockaddr*)&peer, &peerlen);
            if(sockfd < 0){
                std::cerr << "[WARNING] Accept failed, continue...\n";
                continue;
            }

            // 2. 获取客户端信息
            InetAddr client(peer);
            std::cout << "[INFO] New connection: " << client.Ip() << ":" << client.Port() << "\n";

            // 3. 构建任务
            // 使用lambda捕获sockfd和client信息
            auto task = [handler, sockfd, client](){
                // 执行具体业务
                handler(sockfd, client);

                // 业务结束关闭
                ::close(sockfd);
                std::cout << "[INFO] Connection closed: " << client.Ip() << ":" << client.Port() << "\n";
            };

            // 4. 扔进线程池队列
            _threadpool->Enqueue(task);
        }
    }

    ~TcpServer() {
        if (_listensock >= 0) ::close(_listensock);
        if (_threadpool) {
            _threadpool->Stop(); // 优雅退出
            delete _threadpool;
        }
    }
private:
    int _port;
    int _listensock;
    bool _is_running;

    ThreadPool<std::function<void()>>* _threadpool;
};
```



# 5. 启动服务器和客户端

## 启动服务器

```cpp
#include <iostream>
#include <string>
#include <memory>
#include "TcpServer.hpp"
#include "Protocol.hpp"

using namespace ProtocolModule;

// -----------------------------------------------------------
//  业务逻辑
// -----------------------------------------------------------
Response Calculate(const Request& req) {
    Response resp(0, 0);
    switch (req._op) {
        case '+': resp._result = req._x + req._y; break;
        case '-': resp._result = req._x - req._y; break;
        case '*': resp._result = req._x * req._y; break;
        case '/': 
            if (req._y == 0) resp._code = 1;
            else resp._result = req._x / req._y;
            break;
        default: resp._code = 2; break;
    }
    return resp;
}

// ==============================================================
// 核心回调
// ==============================================================
void CalculatorService(int sockfd, InetAddr client){
    std::cout << "[INFO] Start service for " << client.Ip() << ":" << client.Port() << "\n";

    // 缓冲区必须在循环外
    // 缓冲区是蓄水池，用来存包，如果放在里面，每次循环数据就会丢失
    std::string in_buffer;

    while(1){
        // 读数据
        char buffer[1024];
        ssize_t n = ::read(sockfd, buffer, sizeof(buffer) - 1);
        
        if(n > 0){
            buffer[n] = '\0';
            // 把buffer拼接到蓄水池里
            in_buffer += buffer;

            // 双层循环，必须用while循环切割
            // 一次read可能读上来 包1 + 包2 + 半个包3
            // 要连续处理完 包1和包2，把半个包3留下
            while(1){
                std::string payload = Decode(in_buffer);
                if(payload.empty()) break;  // 数据不够一个包，跳出内存循环，去外层循环读

                // 流水线
                // 1> 发送请求
                Request req;
                if (!req.Deserialize(payload)) {
                    std::cerr << "[ERROR] Deserialize failed!" << std::endl;
                    continue; 
                }
                // 2> 调业务回应请求
                Response resp = Calculate(req);
                std::string resp_str;
                resp.Serialize(&resp_str);

                // 3> 封包
                std::string send_packet = Encode(resp_str);

                // 4> 发送
                ::write(sockfd, send_packet.c_str(), send_packet.size()); 
                
                std::cout << "[LOG] Processed one request." << std::endl;
            } 
        } else if (n == 0) {
            std::cout << "[INFO] Client disconnected." << std::endl;
            break;
        }
        else {
            break;
        }
    }
}

int main(int argc, char* argv[]) {
    if (argc != 2) {
        std::cout << "Usage: " << argv[0] << " <port>" << std::endl;
        return 1;
    }

    uint16_t port = std::stoi(argv[1]);
    std::unique_ptr<TcpServer> server(new TcpServer(port));
    server->InitServer();
    server->Loop(CalculatorService); // 注册回调

    return 0;
}

```

## 启动客户端

```cpp
#include <iostream>
#include <string>
#include <cstring>
#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include "Protocol.hpp"

using namespace ProtocolModule;

int main(int argc, char* argv[]) {
    if (argc != 3) {
        std::cout << "Usage: " << argv[0] << " <server_ip> <server_port>" << std::endl;
        return 1;
    }

    // 连接服务器 
    std::string ip = argv[1];
    uint16_t port = std::stoi(argv[2]);
    int sockfd = ::socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(port);
    ::inet_pton(AF_INET, ip.c_str(), &server_addr.sin_addr);

    if (::connect(sockfd, (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        perror("connect");
        return 3;
    }
    std::cout << "Connect Success!" << std::endl;

    // 重点1：客户端也要有蓄水池！
    std::string in_buffer; 

    while (true) {
        Request req;
        std::cout << "Please Enter (x op y): ";
        std::cin >> req._x >> req._op >> req._y;
        if (!std::cin.good()) break;

        // 发送流程：序列化 -> 封包 -> 发送
        std::string json_str;
        req.Serialize(&json_str);     
        std::string packet = Encode(json_str); 
        ::write(sockfd, packet.c_str(), packet.size());

        // 重点2：接收响应不能只 read 一次！
        // 必须循环读，直到 Decode 出一个完整包为止
        char buffer[1024];
        while (true) {
            ssize_t n = ::read(sockfd, buffer, sizeof(buffer) - 1);
            if (n > 0) {
                buffer[n] = 0;
                in_buffer += buffer; // 拼接

                std::string payload = Decode(in_buffer); // 尝试切割
                if (!payload.empty()) {
                    // 拿到完整包了，反序列化并打印
                    Response resp;
                    resp.Deserialize(payload);
                    std::cout << "Result: " << resp._result << " [Code: " << resp._code << "]" << std::endl;
                    
                    // 拿到结果后，跳出内层循环，准备下一次用户输入
                    break; 
                }
            } else {
                goto END;
            }
        }
    }

END:
    ::close(sockfd);
    return 0;
}
```

