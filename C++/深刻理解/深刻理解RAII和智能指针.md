# RAII思想

RAII：**资源获取即初始化**--->**把资源的生命周期绑定到栈上对象的生命周期，栈上对象离开作用域就自动销毁了**

* 对象构造，拿到资源
* 对象析构，释放资源

资源的释放不再靠手动完成，而是根据类的特性自动完成

==RAII管理的是资源，不仅仅是内存==

常见的资源：`uniqe_ptr`

* 堆内存：`new/delete`
* 文件句柄：`fopen/fclose`
* 互斥锁：`lock/unlock`
* socket：`创建/关闭`

> ==RAII可以保证异常安全==

例如：

```cpp
#include <iostream>
using namespace std;

class IntHolder {
private:
    int* ptr;
public:
    IntHolder(int value) : ptr(new int(value)) {
        cout << "构造申请资源\n";
    }
    ~IntHolder() {
        cout << "析构释放资源\n";
        delete ptr;
    }
};
void func() {
    IntHolder obj(100);
    throw runtime_error("出错");
}
int main() {
    try {
        func();
    } catch (const exception& e) {
        cout << "捕获到异常: " << e.what() << endl;
    }
    return 0;
}
```

`obj` 是局部对象，异常抛出后`func()` 在栈展开过程中退出作用域，局部对象会自动调用析构函数，因此 `IntHolder` 能在析构中释放资源，这就是 RAII 的异常安全性。



# unique_ptr 独占型

`unique_ptr`是一种**独占所有权**的智能指针，同一时刻只能有一个`unique_ptr`拥有这个资源，对象析构自动释放资源--->**我独有，不能拷贝，可移动，转移所有权**

例如：`unique_ptr<int> p1(new int(10));`

这个`int`归`p1`所有，不允许`p2`也说这是我的

因为：

**如果两个指针都认为自己拥有释放权：**

* 两边都去`delete`，Double Free
* 未定一行为

这也是禁止拷贝，允许移动（**转移所有权的本质还是只有一个指针独享资源**）的原因

> ==`unique_ptr`有哪些使用场景？==

**明确只有一个所有者的资源且需要自动释放**

* 树结构里父节点独占子节点
* 容器保存的独占对象

> ==手撕一个独占型的智能指针==

```cpp
#include <iostream>
template <typename T>
class UniquePtr {
private:
    T* _ptr;
public:
    // 构造获取资源
    UniquePtr(T* ptr) :_ptr(ptr) { }
    // 析构释放资源
    ~UniquePtr() { if(_ptr) delete _ptr; }

    // 禁止拷贝、赋值重载
    UniquePtr(const UniquePtr& other) = delete;
    UniquePtr& operator=(const UniquePtr& other) = delete;

    // 移动构造，_ptr <- other._ptr
    UniquePtr(UniquePtr&& other) :_ptr(other._ptr) { other._ptr = nullptr; }
    // 移动赋值，我接收别人的所有权，我先把我的资源扔了，再接收被人的资源
    UniquePtr& operator=(UniquePtr&& other) {
        if(this != &other) {
            if(_ptr) delete _ptr;
            _ptr = other._ptr;
            other._ptr = nullptr;
        }
        return *this;
    }
    // 指针行为
    // 解引用，取到具体的值
    T& operator*() const { return *_ptr; }
    // 找地址，取到地址
    T* operator->() const { return _ptr; }
    T* get() const { return _ptr; }
};
```

# shared_ptr 共享型

`shared_ptr`是一种**共享所有权**的智能指针，允许有多个`shared_ptr`共同拥有同一块资源，最后一个拥有者销毁时，释放资源

> ==核心特点==

* **共享所有权：允许有多个`shared_ptr`共同拥有同一块资源**

* **通过引用计数管理生命周期**

  内部维护一个计数器：记录**当前有多少个`shared_ptr`正在拥有这块资源**

  * 新拷贝->计数+1
  * 新析构->计数-1
  * 计数减到0，释放资源

* **开销大于`unique_ptr`**

  需要维护：

  * 引用计数
  * 控制块
  * 多线程环境原子操作

> ==`shared_ptr`控制块==

```cpp
shared_ptr<int> p1(new int(10));
shared_ptr<int> p2 = p1;
shared_ptr<int> p3 = p2;
```

* `p1`、`p2`、`p3` 各自是不同对象
* 但它们必须知道自己在管理的是同一块资源
* 所以它们要共同指向同一个控制块

控制块就是那个“共享管理中心”

控制块核心组件：

* 强引用计数：拥有资源的指针数量
* 弱引用计数：观察资源的指针数量

> ==`shared_ptr`工作流程==

* 创建时
  * 分配资源给指针
  * 创建控制块，强引用计数初始化为1
* 拷贝时
  * 新指针指向同一块资源和控制块
  * 强引用计数+1
* 析构时
  * 某个指针离开作用域：
    * 强引用计数-1
    * 如果==0，释放资源，否则不释放
* 最后一个析构时
  * 强引用计数变为0
  * 删除真正的对象
  * 控制块是否释放，取决于`weak_ptr`

> ==`shared_ptr`使用场景==

* **资源需要多方共享**
  * 一个对象被多个模块使用
  * 回调要延迟对象存活时间
* **谁最后结束不确定**--->说不清谁负责delete
* **需要把资源生命周期交给最后一个使用者**

> ==`shared_ptr`线程安全吗？==

线程安全部分：**不同副本，引用计数++--线程安全**

* 拷贝
* 析构
* 赋值

重点是：`shared_ptr`被按值捕获了，每个线程操作的是自己的副本，**不同时修改同一个`shared_ptr`变量本体**

线程不安全部分：

* **对象内容并发修改：产生数据竞争**
* **同一个`shared_ptr`变量并发修改，存在竞争关系**

> ==手撕`shared_ptr`==

```cpp
template <typename T>
class SharedPtr {
private:
    T* _ptr;
    int* _cnt;  // 放到堆上，多个对象共享堆资源

    // 手动实现释放逻辑
    void Release() {
        if(0 == --(*_cnt)) {
            delete _ptr;
            delete _cnt;
        }
    }
public:
    // 构造获取资源，计数++
    SharedPtr(T* ptr = nullptr) : _ptr(ptr), _cnt(new int(1)) { }
    // 析构直接调用Release
    ~SharedPtr() { Release(); }

    // 拷贝构造和赋值
    SharedPtr(const SharedPtr& other) :_ptr(other._ptr), _cnt(other._cnt) {
        ++(*_cnt);
    }
    SharedPtr& operator=(const SharedPtr& other) {
        if(this != &other) {
            // other覆盖_ptr
            Release();
            _ptr = other._ptr;
            _cnt = other._cnt;
            ++(*_cnt);
        }
        return *this;
    }

    // 指针行为
    T& operator*() const { return *_ptr; }
    T* operator->() const { return _ptr; }
    T* get() const { return _ptr; }
    int cnt() const { return *_cnt; }
};
```

> `shared_ptr`循环引用问题

当多个对象彼此持有`shared_ptr`，形成强引用环，引用计数永远到不了0，资源无法释放

```cpp
// 循环引用问题
class B;
class A {
public:
    std::shared_ptr<B> _pb;
};
class B {
public:
    std::shared_ptr<A> _pa;
};
int main() {
    std::shared_ptr<A> a(new A());
    std::shared_ptr<B> b(new B());
    a->_pb = b;
    b->_pa = a;
    return 0;
}
```

此时：

* `a`强持有`b`
* `b`强持有`a`

成环了，**引用计数只能处理树状链式拥有关系，不能处理环状拥有关系**

# weak_ptr 弱引用型

`weak_ptr` 是一种**弱引用智能指针**，它可以观察 `shared_ptr` 管理的对象，但**不拥有对象，不增加强引用计数**

它的意义是：**我想知道对象还在不在，想访问它，但我不想因为我指着它，它就一直存活**

```cpp
// weak_ptr解决循环引用
class B;
class A {
public:
    std::shared_ptr<B> _pb;
};

class B {
public:
    std::weak_ptr<A> _pa;
};

int main() {
    // A 强计数1 B 强计数1
    std::shared_ptr<A> a(new A());
    std::shared_ptr<B> b(new B());

    // B 强计数2
    a->_pb = b; // b赋给_pb了，_pb是shared，B的强计数+1
    // A 强计数1 弱计数1
    b->_pa = a; // a赋给_pa了，_pa是weak，A的强计数不变
    return 0;
}
```

**环里至少有一条边不能是强引用**

# 总结

* RAII 是把资源生命周期绑定到对象生命周期上，依赖析构自动释放资源。

* `unique_ptr` 表达独占所有权，不能拷贝，只能移动。
* `shared_ptr` 表达共享所有权，底层依赖控制块和强引用计数。
* 循环引用的本质是强引用环导致计数无法归零。
* `weak_ptr` 不拥有对象、不增加强计数，通过 `lock()` 安全访问对象。