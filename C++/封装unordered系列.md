本文按照我的思路实现简单的`unordered_map / unordered_set`封装

一上来我先会这样写哈希表：

- 要么「开放定址 + KV 模型」
- 要么「拉链法 + KV 模型（`pair<K,V>` 写死在结构里）」

能跑，但一旦想封装成 `unordered_map` / `unordered_set`，问题就来了：

- `unordered_map` 的底层元素是 `pair<const K, V>`（key 是 const）
- `unordered_set` 只存一个 `K`
- 想让二者共用同一套哈希桶，还要有迭代器支持 `begin/end` 

这篇文章就从**一个普通的 KV 模型哈希表**出发，一步一步改造为：

1. 通用哈希桶 `HashTable<K, T, KeyOfT, Hash>`
2. 支持迭代器的哈希表（`HTIterator`）
3. 在上面封装 `unordered_map`
4. 在上面封装 `unordered_set`
5. 最后列一大堆**我自己踩过的坑**，让你避坑

[TOC]

# 1.起点：KV 模型哈希表的局限

刚开始我写了一个**KV 模型**的哈希表，大概长这样（拉链法版本）：

```cpp
template<class K, class V>
struct KVNode {
    std::pair<K, V> _kv;
    KVNode<K, V>* _next;

    KVNode(const std::pair<K, V>& kv)
        : _kv(kv)
        , _next(nullptr)
    {}
};

template<class K, class V, class Hash = HashFunc<K>>
class HashTableKV {
    typedef KVNode<K, V> Node;
public:
    HashTableKV()
        : _tables(10, nullptr)
        , _nums(0)
    {}

    bool Insert(const std::pair<K, V>& kv);
    Node* Find(const K& key);
    bool Erase(const K& key);

private:
    std::vector<Node*> _tables; // 桶数组（每个桶是一个链表头）
    size_t _nums;               // 当前元素个数
};
```

**问题在于：**

- 节点类型**写死**为 `pair<K,V>`；
- 一旦我想实现 `unordered_set<K>`（只有 K 没 V），就没法复用；
- STL 的 `unordered_map` 里元素类型是 `pair<const K, V>`（key 是 const）；
- 希望 map 和 set 能共用同一套哈希桶设计。

所以我们要往「**泛型哈希桶**」的方向重构。

# 2. 第一步：从 `pair<K,V>` 节点抽象到泛型 `T` 节点

**目标：**
 让哈希表不再关心「底层存的是不是 pair<K,V>」，而只是存“某种类型 T”。

## 2.1. 泛型节点 HashNode

```cpp
template<class T>
struct HashNode {
    HashNode<T>* _next; // 单链表指针
    T _data;            // 真实数据

    HashNode(const T& data)
        : _next(nullptr)
        , _data(data)
    {}
};
```

这一改动把节点从 「KV 模型」解耦成了「泛型 T 模型」，以后：

- 对 `unordered_map`：T 可以是 `std::pair<const K, V>`
- 对 `unordered_set`：T 可以是 `const K`

> ✅ **核心变化：**
>  节点不再和 `pair` 绑死了，只是负责存一个 `T`。

### 🔴 易错点 1：Node 类型用成 `HashNode<K>` 而不是 `HashNode<T>`

我自己就犯过这个错：

```cpp
// ❌ 错误版本
template<class K, class T, class KeyOfT, class Hash = HashFunc<K>>
class HashTable {
    typedef HashNode<K> Node; // 错了！应该是 T
};
```

后果：

- `Node` 里的 `_data` 类型变成 `K`
- 在 `Insert` 时传的是 `T`（比如 `pair<const K,V>`），构造 `Node(data)` 就会类型不匹配、编译不过。

**正确写法：**

```cpp
typedef HashNode<T> Node; // ✅ 节点里存 T
```

------

# 3. 第二步：引入 KeyOfT —— 从 T 中抽 key

哈希表的所有关键操作，从本质上看只干一件事：

> “给我一个 `key`，我通过 hash 函数算出它应该在哪个桶。”

但现在节点存的是 T，例如：

- map 的 T：`std::pair<const K, V>`
- set 的 T：`const K`

哈希表不应该理解 T 的内部结构，它只需要知道：

> **如何从 T 里抽出 key K。**

于是我引入了一个策略类：**KeyOfT**

## 3.1. map 里的 KeyOfT

```cpp
struct MapKeyOfT {
    const K& operator()(const std::pair<const K, V>& kv) const {
        return kv.first;  // key 在 pair<const K,V> 的 first 里
    }
};
```

## 3.2. set 里的 KeyOfT

```cpp
struct SetKeyOfT {
    const K& operator()(const K& key) const {
        return key;       // T 本身就是 K（或 const K）
    }
};
```

## 3.3. HashTable 内部统一调用 KeyOfT

在哈希表内部，不再直接写 `node->_data.first`，而是：

```cpp
KeyOfT kot;
K key = kot(node->_data);
```

**不管 T 是 pair 还是 K，本质都是：
 给我一个 T，我用 KeyOfT 把它变成 key。**

# 4. 第三步：通用哈希桶 `HashTable<K, T, KeyOfT, Hash>`

## 4.1. 整体框架

```cpp
template<class K, class T, class KeyOfT, class Hash = HashFunc<K>>
class HashTable {
    typedef HashNode<T> Node; // 节点存 T

    template<class K, class T, class Ptr, class Ref, class KeyOfT, class Hash>
    friend struct HTIterator; // 迭代器需要访问 _tables

public:
    typedef HTIterator<K, T, T*, T&, KeyOfT, Hash> Iterator;
    typedef HTIterator<K, T, const T*, const T&, KeyOfT, Hash> ConstIterator;

    HashTable()
        : _tables(10, nullptr)
        , _nums(0)
    {}

    ~HashTable() {
        for (size_t i = 0; i < _tables.size(); ++i) {
            Node* cur = _tables[i];
            while (cur) {
                Node* next = cur->_next;
                delete cur;
                cur = next;
            }
            _tables[i] = nullptr;
        }
    }

    Iterator Begin();
    Iterator End();
    ConstIterator Begin() const;
    ConstIterator End() const;

    std::pair<Iterator, bool> Insert(const T& data);
    Iterator Find(const K& key);
    bool Erase(const K& key);

private:
    std::vector<Node*> _tables; // 桶数组
    size_t _nums;               // 元素个数
};
```

------

## 4.2. Begin / End：给迭代器一个起点和终点

```cpp
Iterator Begin() {
    if (_nums == 0) return End(); // 空表

    for (size_t i = 0; i < _tables.size(); ++i) {
        if (_tables[i]) {
            return Iterator(_tables[i], this); // 桶中第一个非空
        }
    }
    return End();
}

Iterator End() {
    return Iterator(nullptr, this); // End 用 node==nullptr 表示
}

// const 版本
ConstIterator Begin() const {
    if (_nums == 0) return End();

    for (size_t i = 0; i < _tables.size(); ++i) {
        if (_tables[i]) {
            return ConstIterator(_tables[i], this);
        }
    }
    return End();
}

ConstIterator End() const {
    return ConstIterator(nullptr, this);
}
```

------

## 4.3. Insert：查重 + 扩容 + 头插

**接口：**

```cpp
std::pair<Iterator, bool> Insert(const T& data);
```

**实现思路：**

1. 用 `KeyOfT` 从 `data` 中抽出 `key`
2. 用 `Find(key)` 查重
3. 如果满了扩容：新建桶数组，重新 hash 所有节点
4. 最后把新节点头插到对应桶

**实现示例：**

```cpp
std::pair<Iterator, bool> Insert(const T& data) {
    KeyOfT kot;
    Hash hs;

    // 1. 查重
    K key = kot(data);
    Iterator it = Find(key);
    if (it != End()) {
        return std::make_pair(it, false); // 已存在，不插入
    }

    // 2. 扩容：这里简单用“元素个数 == 桶数”作为扩容条件
    if (_nums == _tables.size()) {
        std::vector<Node*> newTables(_tables.size() * 2, nullptr);

        for (size_t i = 0; i < _tables.size(); ++i) {
            Node* cur = _tables[i];
            while (cur) {
                Node* next = cur->_next;

                const K& key = kot(cur->_data);
                size_t hashIdx = hs(key) % newTables.size();

                cur->_next = newTables[hashIdx];
                newTables[hashIdx] = cur;

                cur = next;
            }
            _tables[i] = nullptr;
        }

        _tables.swap(newTables);
    }

    // 3. 插入：头插到对应桶
    size_t hashIdx = hs(key) % _tables.size();
    Node* newNode = new Node(data);
    newNode->_next = _tables[hashIdx];
    _tables[hashIdx] = newNode;
    ++_nums;

    Iterator ret(newNode, this);
    return std::make_pair(ret, true);
}
```

### 🔴 易错点 2：`return make_pair((newNode,this),true);`

我曾经这样写过：

```cpp
return make_pair((newNode,this), true);
```

**这是一个非常隐蔽的大坑：**

- `(newNode, this)` 在 C++ 里是**逗号表达式**，值是右边的 `this`；
- 类型是 `HashTable*`，而不是 `Iterator`；
- 函数返回类型是 `std::pair<Iterator,bool>`，里面却塞了个 `HashTable*`；
- 编译器直接报类型错误。

**正确做法：**

```cpp
Iterator it(newNode, this);
return std::make_pair(it, true);
```

> 记住：**迭代器就是一个对象，必须显式构造**

## 4.4. Find：在对应桶的链表中查找

```cpp
Iterator Find(const K& key) {
    KeyOfT kot;
    Hash hs;
    size_t hashIdx = hs(key) % _tables.size();

    Node* cur = _tables[hashIdx];
    while (cur) {
        if (kot(cur->_data) == key) {
            return Iterator(cur, this);
        }
        cur = cur->_next;
    }
    return End();
}
```

## 4.5. Erase：标准单链表删除

```cpp
bool Erase(const K& key) {
    KeyOfT kot;
    Hash hs;
    size_t hashIdx = hs(key) % _tables.size();

    Node* cur = _tables[hashIdx];
    Node* prev = nullptr;

    while (cur) {
        if (kot(cur->_data) == key) {
            if (prev == nullptr) {
                _tables[hashIdx] = cur->_next; // 删桶头
            } else {
                prev->_next = cur->_next;      // 删中间
            }
            delete cur;
            --_nums;
            return true;
        }
        prev = cur;
        cur = cur->_next;
    }
    return false;
}
```

### 🔴 易错点 3：删除时忘了 `--_nums`

- 这样会导致 `_nums` 一直不减；
- 扩容条件（比如 `_nums == _tables.size()` 或根据负载因子）会被误判；
- 表面看不一定 crash，但哈希表的行为越来越奇怪（疯狂扩容）。

# 5. 第四步：实现哈希表迭代器 HTIterator

## 5.1 迭代器要提供什么能力？

希望下面写法可以工作：

```cpp
HashTable<K,T,KeyOfT,Hash> ht;
for (auto it = ht.Begin(); it != ht.End(); ++it) {
    // *it 是 T&，it-> 可以访问 T 的成员
}

for (const auto& e : ht) {
    // range-for 也依赖 begin()/end() + 迭代器
}
```

所以迭代器要实现：

- `operator*`（解引用）
- `operator->`（成员访问）
- `operator++`（前置 ++）
- `operator== / !=`（比较）

## 5.2. 迭代器内部状态

```cpp
template<class K, class T, class Ptr, class Ref, class KeyOfT, class Hash>
struct HTIterator {
    typedef HashNode<T> Node;
    typedef HashTable<K, T, KeyOfT, Hash> HT;
    typedef HTIterator<K, T, Ptr, Ref, KeyOfT, Hash> Self;

    Node* _node;      // 当前节点
    const HT* _pht;   // 整个哈希表（用于 ++ 时跨桶）

    HTIterator(Node* node, const HT* pht)
        : _node(node)
        , _pht(pht)
    {}
```

为什么要 `_pht`？

- 当前桶内可以靠 `_node->_next` 走链表；
- 但是桶之间怎么跳？需要知道：
  - 当前节点在哪个桶（需要 hash(key) % bucket_count）
  - 后面还有哪些桶（需要访问 `_pht->_tables`）

## 5.3. 解引用 / -> / 比较

```cpp
    // 解引用：得到当前 T
    Ref operator*() const { return _node->_data; }

    // 成员访问：让 it->first / it->second 能用
    Ptr operator->() const { return &_node->_data; }

    bool operator!=(const Self& s) const { return _node != s._node; }
    bool operator==(const Self& s) const { return _node == s._node; }
```

这里 `Ptr` / `Ref` 是为了复用：

- 普通迭代器：`Ptr = T*`, `Ref = T&`
- const 迭代器：`Ptr = const T*`, `Ref = const T&`

## 5.4. 核心：前置 `++` —— 同桶走链表，跨桶走数组

```cpp
    Self& operator++() {
        if (_node == nullptr) return *this; // 已经是 end()

        if (_node->_next) {
            // 1. 同一个桶内链表还有下一个节点
            _node = _node->_next;
        } else {
            // 2. 当前桶链表走完了，跨桶
            KeyOfT kot;
            Hash hs;

            size_t hashIdx = hs(kot(_node->_data)) % _pht->_tables.size();

            ++hashIdx; // 从下一个桶开始找
            while (hashIdx < _pht->_tables.size()) {
                if (_pht->_tables[hashIdx]) {
                    _node = _pht->_tables[hashIdx]; // 下一个非空桶的第一个节点
                    return *this;
                }
                ++hashIdx;
            }
            // 3. 后面都没有非空桶了，设为 end()
            _node = nullptr;
        }
        return *this;
    }
};
```

配合 HashTable 中：

```cpp
typedef HTIterator<K, T, T*, T&, KeyOfT, Hash> Iterator;
typedef HTIterator<K, T, const T*, const T&, KeyOfT, Hash> ConstIterator;
```

我们就同时拥有了：

- 可写迭代器：`Iterator`
- 只读迭代器：`ConstIterator`

# 6. 第五步：在通用哈希桶上封装 `unordered_map`

来看最终暴露给用户的 `unordered_map`：

```cpp
namespace Vect {

template<class K, class V, class Hash = hash_bucket::HashFunc<K>>
class unordered_map {
    // 从 pair<const K,V> 中抽 key
    struct MapKeyOfT {
        const K& operator()(const std::pair<const K, V>& kv) const {
            return kv.first;
        }
    };

    // 底层哈希表类型：T = pair<const K, V>
    typedef hash_bucket::HashTable<K, std::pair<const K, V>, MapKeyOfT, Hash> HT;

public:
    typedef typename HT::Iterator iterator;
    typedef typename HT::ConstIterator const_iterator;

    unordered_map() = default;

    iterator begin() { return _ht.Begin(); }
    iterator end()   { return _ht.End();   }
    const_iterator begin() const { return _ht.Begin(); }
    const_iterator end()   const { return _ht.End();   }

    std::pair<iterator, bool> insert(const std::pair<K, V>& kv) {
        // pair<K,V> 可以隐式转换为 pair<const K,V>
        return _ht.Insert(kv);
    }

    iterator find(const K& key) { return _ht.Find(key); }

    bool erase(const K& key) { return _ht.Erase(key); }

    // 重点：operator[]
    V& operator[](const K& key) {
        auto ret = _ht.Insert(std::make_pair(key, V()));
        return ret.first->second;
    }

private:
    HT _ht;
};

} // namespace Vect
```

##  6.1. 为什么要 `pair<const K, V>` 而不是 `pair<K, V>`

STL 标准这么设计是有原因的：

- `unordered_map` 的元素类型是 `std::pair<const K,V>`
- 通过迭代器访问时：`it->first` 类型是 `const K&`
- 不能写：`it->first = newKey;`
   从语法层面禁止你修改 key

如果 key 能被随便改，那哈希表就会直接炸裂：

改了 key，节点所在桶没变，相当于表结构被破坏。

所以我也跟 STL 一样，让底层存 `pair<const K,V>`，**配合迭代器自然“禁止修改 key”**。

## 6.2. `operator[]` 的经典写法

标准 `std::unordered_map` 的语义：

```cpp
V& operator[](const K& key);
```

- 若 key 不存在：插入 { key, V() }，返回这个新元素的 value 引用；
- 若 key 存在：直接返回已有元素的 value 引用。

现在这两行刚好做到：

```cpp
auto ret = _ht.Insert(std::make_pair(key, V()));
return ret.first->second;
```

- 当 key 不存在：Insert 插入新节点，返回 `{新位置,true}`；
- 当 key 存在：Insert 直接返回 `{旧位置,false}`；
- 无论如何，`ret.first` 都指向该 key 对应的节点，所以 `->second` 就是正确的 value 引用。

------

# 7. 第六步：在通用哈希桶上封装 `unordered_set`

`unordered_set` 比 `unordered_map` 简单：只有 key 没有 value。

```cpp
namespace Vect {

template<class K, class Hash = hash_bucket::HashFunc<K>>
class unordered_set {
    struct SetKeyOfT {
        const K& operator()(const K& key) const {
            return key;  // T 本身就是 K
        }
    };

    typedef hash_bucket::HashTable<K, const K, SetKeyOfT, Hash> HT;

public:
    typedef typename HT::Iterator iterator;
    typedef typename HT::ConstIterator const_iterator;

    unordered_set() = default;

    iterator begin() { return _ht.Begin(); }
    iterator end()   { return _ht.End();   }
    const_iterator begin() const { return _ht.Begin(); }
    const_iterator end()   const { return _ht.End();   }

    iterator find(const K& key) { return _ht.Find(key); }
    bool erase(const K& key)    { return _ht.Erase(key); }

    std::pair<iterator, bool> insert(const K& key) {
        // T = const K，可以从 K 隐式构造
        return _ht.Insert(key);
    }

private:
    HT _ht;
};

} // namespace Vect
```

要点：

- `T = const K`，所以 `*it` 的类型是 `const K&`，不能通过迭代器修改 key；
- 其余接口全部交给底层哈希桶完成。

------

# 8. 踩坑总结：这些地方我真踩过（建议收藏）

最后，把我在这套封装里踩过/容易踩的坑按点列一下，方便你对照自查。

## 8.1. Node 类型写成 `HashNode<K>` 而不是 `HashNode<T>`

- 结果：节点里的 `_data` 变成 `K`；
- 但 Insert 传入的是 `T`（`pair<const K,V>` 等），构造器对不上，编译错误。

## 8.2. Insert 返回值写成 `make_pair((newNode,this),true)`

- `(newNode,this)` 是逗号表达式 → 值其实是 `this`（`HashTable*`）；
- Insert 返回类型是 `std::pair<Iterator,bool>`，类型完全不对。

**正确写法：**

```cpp
Iterator it(newNode, this);
return std::make_pair(it, true);
```

## 8.3. 把 `Find` 当成 `Node*` 用

比如这样写：

```cpp
Node* ret = Find(kot(data));
if (ret != End()) ...
```

但其实 `Find` 返回的是 `Iterator`，`End()` 也是 `Iterator`，类型全错。

应该是：

```cpp
Iterator it = Find(kot(data));
if (it != End()) ...
```

##  8.4. 删除时忘掉 `--_nums`

导致：

- `_nums` 一直不减；
- 负载因子判断永远认为“很满”，经常扩容。

## 8.5. 扩容时 rehash 用错模数

常见错误：

```cpp
size_t hashIdx = hs(key) % _tables.size(); // 错！应该对 newTables.size()
```

正确是：

```cpp
hashIdx = hs(key) % newTables.size();
```

否则所有节点会跑到错误的桶。

## 8.6. HTIterator 里没有前置声明 HashTable

迭代器里写了：

```cpp
typedef HashTable<K, T, KeyOfT, Hash> HT;
```

但编译器还没见过 `HashTable` 的声明，会报错。
 需要在前面先做前置声明：

```cpp
template<class K, class T, class KeyOfT, class Hash>
class HashTable;
```

## 8.7. const 成员函数里调用非 const 的 `End()`

比如：

```cpp
ConstIterator Begin() const {
    if (_nums == 0) return End(); // End() 返回 Iterator，不是 ConstIterator
}
```

- const 成员函数不能调用非 const 成员；
- 返回类型也不对。

可以直接提供 `End() const`，返回 `ConstIterator`，这样 const 版本 Begin() 调的就是 const End()。



## 8.8. 开放定址版本里 `Find` 忘记 `++hashIdx`，直接死循环

典型 bug：

```cpp
while (_tables[hashIdx]._state != EMPTY) {
    if (_tables[hashIdx]._state == EXIST
        && _tables[hashIdx]._kv.first == key)
        return &_tables[hashIdx];
    // 忘了 ++hashIdx
}
```

插第三个元素的时候直接卡死在查重。

# 9. 小结 & 建议

从最初的 KV 模型哈希表到最后的 `unordered_map / unordered_set`，实际上就是一条不断抽象和解耦的路线：

1. **KV 哈希表（存 pair<K,V>）**
2. 改造节点 → `HashNode<T>`
3. 引入 `KeyOfT` → 从 T 中抽 key
4. 通用哈希桶 → `HashTable<K,T,KeyOfT,Hash>`
5. 给哈希桶加迭代器 → `HTIterator`
6. 用 T + KeyOfT 封装 `unordered_map` / `unordered_set`

走完这个流程，不仅能自己手写 `unordered_map` / `unordered_set`，

 还真正理解了 STL 设计里那种「用策略（KeyOfT）解耦结构」的思想。

**最后的建议：**

- 真正关掉现成代码，只拿这篇文章当“提纲”，从空文件再实现一遍；
- 每一步都先写最简单可跑版本，再慢慢泛型化；
- 每次重构前，把“我这一层想解耦的到底是什么”写在注释里，你会清楚很多。

最后，完整代码，超详细注释：

