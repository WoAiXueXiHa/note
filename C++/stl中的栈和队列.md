[TOC]

# 1. 容器适配器

## 1.1. 什么是适配器

* 生活中，适配器**是两端接口不匹配的转换器**

例如：旅行插头：国标->美标；`Type-C`转`HDMI`：手机/电脑->显式器；翻译：中文<->英文

> 都有如下共同点：**不改变两端事物本身**，只在中间“转一下接口/协议/形状”，让他们可以合作

* 回到`C++`设计，容器适配器的本质是一种设计模式：**在已有容器之上，封装出特定用法的外壳，只暴露少量接口，不让用户随意访问修改**

> 为什么要适配器？专注“行为”而非“存放方式”。对于一个需求的实现，我们只需要关心实现的行为“后进先出/先进先出/拿最大的先出”，而对于底层，我们无需关心

## 1.2. 容器适配器家族

* 家族成员：`std::stack` 、`std::queue` 、 `std::priority_queue`
* 默认底层结构：

		1. `stack<T,Container=deque<T>>`
		1. `queue<T,Container=deque<T>>`
		1. `priority_queue<T,Container=vector<T>,Compare=less<T>>`默认是大根堆

## 1.3. deque容器

`deque`名为**双端队列**，是一种顺序容器

### 基本思想

* **分块存储(bloks)+指针表(map)：** `deque`不像`vector`那样一整条连续的内存，而是把元素分配在若干个**定长的数据块中**，在用一张**指针表(map)(又称中控数组)记录各个区块的地址。可以在两端扩张**

![image-20251012151500423](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20251012151500423.png)

* **复杂度：**

		1. 随机访问`operator[]`:$O(1)$，经过指针表->块->元素的两次寻址
		1. 头/尾插入删除：$O(1)$
		1. 中间插入删除：$O(n)$

> 可以把`deque`想象成一条大街切割成一段段街区，手里拿着一张“街区索引表”。在两端增加删除街区很容易；若要在中间增加删除，则需要挪一串

### 内存分布

![image-20251012152312222](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20251012152312222.png)

双端队列是一段假象的连续空间，实际上是分段连续，为了维护“整体连续”和随机访问的假象，设计了复杂的`deque`迭代器

![image-20251012154341849](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20251012154341849.png)

`deque`如何借助迭代器维护其假想的连续结构呢？

![image-20251012154946343](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20251012154946343.png)

### deque的优缺点

* 优势：和`vector`比较，头部插入删除时，**不需要挪动数据，效率高**，**扩容时也不需要挪动大量数据**，和`list`相比，底层是连续空间，**空间利用率较高**，不用存储额外字段
* 劣势：**不适合遍历，在遍历时，`deque`的迭代器要频繁地检测是否移动到某段小空间的边界，导致效率低下**。在序列场景下，需要经常遍历，所以选择线性结构时，优先考虑`vector`和`list`，`deque`主要作为`satck`和`queue`的底层结构

### 为什么选择`deque`作为`stack`和`queue`的底层默认容器

`satck`**后进先出**，因此只需要`push_back()`和`pop_back()`操作的线性结构，那么`vector`和`list`也适配，`queue`**先进先出**，因此只需要`push_back()`和`pop_front()`操作的线性结构，那么`list`适配。

但是`STL`中却采用了`deque`，理由如下：

* `satck`和`queue`无需遍历操作（所以`stack`和`queue`没有迭代器），只需要在固定的一段或者两端操作
* `satck`中元素增长时，`deque`比`vector`的效率高（扩容不用挪动大量数据），`queue`中元素增长时，`deque`的效率和内存使用率都很高

# 2. stack的模拟实现和应用

## 2.1. 模拟实现

对于`satck`更多详细介绍请看这篇文章：

[从直线到环形：解锁栈、队列背后的空间与效率平衡术-CSDN博客](https://blog.csdn.net/Vect__/article/details/152415695?spm=1001.2014.3001.5502)

```cpp
#pragma once
#include <iostream>
#include <deque>

namespace Vect {
	// 第二个参数：适配器 传一个容器
	template <class T, class Container = std::deque<T>>
	class stack {
	public:
		stack(){}
		void push(const T& val) { _con.push_back(val); }
		void pop() { _con.pop_back(); }
		bool empty() const { return _con.empty(); }
		const size_t size() const { return _con.size(); }
		const T& top() const { return _con.back(); }
		T& top() { return _con.back(); }
	private:
		// 底层是容器 
		Container _con;
	};
}
```

## 2.2. 应用

[155. 最小栈 - 力扣（LeetCode）](https://leetcode.cn/problems/min-stack/description/)

思路：用一个栈存序列所有元素，一个栈存序列的最小值

过程演示：

![image-20251012164308697](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20251012164308697.png)

代码：

```cpp
class MinStack {
public:
    MinStack() { }
    
    void push(int val) {
        // _min为空or_min的栈顶元素>=val 入栈
        if(_min.empty() || _min.top() >= val) _min.push(val);

        // _element正常入栈
        _element.push(val);
    }
    
    void pop() {
        // _min的栈顶元素==_element的栈顶元素，出栈，保证_min存的永远是当前阶段最小值
        if(_min.top() == _element.top()) _min.pop();

        _element.pop();
    }
    
    int top() {
        return _element.top();
    }
    
    int getMin() {
        return _min.top();
    }
private:
    stack<int> _element; // 存序列所有元素
    stack<int> _min; // 存当前序列的最小值
};
```



[栈的压入、弹出序列_牛客题霸_牛客网](https://www.nowcoder.com/practice/d77d11405cc7470d82554cb392585106?tpId=13&&tqId=11174&rp=1&ru=/activity/oj&qru=/ta/coding-interviews/question-ranking)

思路：

1. 入栈序列入栈一个元素
2. 用栈顶元素和出栈序列进行比较，会有两种情况：

* 栈顶元素==出栈序列当前元素，出栈序列往后遍历，弹出当前元素，回到步骤2继续
* 栈顶元素!=出栈序列当前元素或栈为空，回到步骤1继续

具体步骤：

![image-20251012181742297](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20251012181742297.png)

代码：

```cpp
 bool IsPopOrder(vector<int>& pushV, vector<int>& popV) {
        // 入栈序列和出栈序列size不同 一定不匹配
        if(pushV.size() != popV.size()) return false;

        // 定义出栈序列和入栈序列索引
        size_t pushIdx = 0, popIdx = 0;
        stack<int> st;
       while(popIdx < popV.size()){
            // 栈为空或者栈顶元素和出栈序列元素不相等 入栈
            while(st.empty() || st.top() != popV[popIdx]){
               if(pushIdx < pushV.size()) st.push(pushV[pushIdx++]);
               else return false;
            }
            // 栈顶元素和出栈序列元素相等 出栈
            st.pop();
            ++popIdx;
        }
        return true;
    }
};
```

# 3. queue的模拟实现和应用

## 3.1. 模拟实现

对于`queue`更多详细介绍请看这篇文章：

[从直线到环形：解锁栈、队列背后的空间与效率平衡术-CSDN博客](https://blog.csdn.net/Vect__/article/details/152415695?spm=1001.2014.3001.5502)

```cpp
#pragma once
#include <iostream>
#include <deque>

namespace Vect {
	template <class T, class Container = std::deque<int>>
	class queue {
	public:
		void push(const T& val) { _con.push_back(val); }
		void pop() { _con.pop_front(); }
		bool empty() const { return _con.empty(); }
		const T& front() cosnt { _con.front(); }
		T& front() { _con.front(); }
		const T& back() const { _con.back(); }
		T& back() { _con.back(); }
		const size_t size() const { return _con.size(); }
	private:
		Container _con;
	};
}
```

## 3.2. 应用

[102. 二叉树的层序遍历 - 力扣（LeetCode）](https://leetcode.cn/problems/binary-tree-level-order-traversal/submissions/669925847/)

思路：

利用`levelSize`变量获取每一层的节点数，利用队列先进先出的特性，第一层入队列，出队列前将第二层子节点带入队列，然后出队列，循环往复

代码：

```cpp
class Solution {
public:
    vector<vector<int>> levelOrder(TreeNode* root) {
        vector<vector<int>> ret;
        if(root == nullptr) return ret;
        // 控制一层一层进队列
        size_t levelSize = 1;
        queue<TreeNode*> q;
        q.push(root);
        while(!q.empty()){
            vector<int> v;
            
            // 控制一层一层出队列
            while(levelSize--){
                TreeNode* front = q.front();
                q.pop();
                v.push_back(front->val);

                if(front->left) q.push(front->left);
                if(front->right) q.push(front->right);
            }
            ret.push_back(v);
            // 当前层已经出完 下一层也带到了队列中
            levelSize = q.size();
        }
        return ret;
    }
};
```

# 4. priority_queue

## 4.1. 功能介绍

1. 优先级队列是一种容器适配器，它的第一个元素总是所有元素中最大或最小的
2. 本质就是堆，在堆中可以随时插入元素，并且只能检索堆顶元素（优先队列中位于顶部的元素）
3. 底层容器可以是任何标准容器类模板，也可以是其他特定设计的容器类。支持以下操作：

* `empty()`：检测容器是否为空
* `size()`：返回容器中有效元素个数‘
* `front`：返回容器中第一个元素的引用
* `push_back()`：在容器尾部插入元素
* `pop_back()`:删除容器尾部元素

## 4.2. 模拟实现

### 设计模式

* **底层容器：** 用`vector<int>`存堆（连续内存+随机访问效率高）
* **堆的公式：** 索引0是堆顶`top()`，对任意节点`i`满足：
  1. 父节点：`parent = (i - 1) / 2`
  2. 左孩子：`left = 2 * i + 1`
  3. 右孩子：`right = 2 * i + 2`
* **`Compare`仿函数：** 类型参数`Compare`(默认是`less<T>`)

约定：若`Compare(a,b)`为真，表示**a的优先级低于b**，于是：

* 默认`less<T>` -> 大顶堆
* 改成`great<T>` ->小顶堆

### 完整实现

```cpp
#pragma once
#include <iostream>
#include <vector>
#include <functional>
#include <utility>

namespace Vect {
	// 维护一个二叉堆
	// Compare(a,b) == true 表示a的优先级低于b

    template <class T>
    struct myLess {
        // 返回 true 表示 a 小于 b
        // 用途：
        //   1) std::sort(vec.begin(), vec.end(), myLess<int>{})  → 升序
        //   2) std::priority_queue<int, std::vector<int>, myLess<int>>
        //        使用a<b为低优先级 → 形成大顶堆
        bool operator()(const T& a, const T& b) const { return a < b;}
    };

    template <class T>
    struct myGreater {
        // 返回 true 表示 a 大于 b
        // 用途：
        //   1) std::sort(vec.begin(), vec.end(), myGreater<int>{}) → 降序
        //   2) std::priority_queue<int, std::vector<int>, myGreater<int>>
        //        使用a>b为低优先级 → 形成小顶堆
        bool operator()(const T& a, const T& b) const { return a > b;}
    };


	template <class T, class Container = std::vector<T>, class Compare = myLess<T>>
	class priority_queue {
    public:
        // 默认构造 
        priority_queue() = default;
        // 迭代器区间构造
        template <class InputIterator>
        priority_queue(InputIterator first, InputIterator last) {
            while (first != last) {
                _con.push_back(*first);
                ++first;
            }
            // 建堆
            int n = (int)_con.size();
            for (int i = (n - 2) / 2; i >= 0; --i) {
                // 向下调整
                adjustDown(i);
            }
        }

        // 向上调整 
        void adjustUp(int child) {
            Compare comFunc;
            // 找父节点
            int parent = (child - 1) / 2;
            while (child > 0) {
                // if(_con[parent] < _con[child])
                if (comFunc(_con[parent], _con[child])) {
                    std::swap(_con[parent], _con[child]);
                    child = parent;
                    parent = (child - 1) / 2;
                }
                else {
                    break;
                }
            }
        }

        // 向下调整
        void adjustDown(int parent) {
            Compare comFunc;
            // 假设左孩子是两个孩子中更大的
            int child = 2 * parent + 1;
            while (child < (int)_con.size()) {
                // 假设错误
                //  if (child + 1 < _con.size() && _con[child] < _con[child + 1])
                if (child + 1 < (int)_con.size() && comFunc(_con[child], _con[child + 1])) {
                    ++child;
                }
                // 比较孩子和父亲
                // if (_con[parent] > _con[child])
                if (comFunc(_con[parent], _con[child])) {
                    std::swap(_con[child], _con[parent]);
                    parent = child;
                    child = 2 * parent + 1;
                }
                else {
                    break;
                }
            }
        }

        // 交换堆顶堆底元素 删除堆底元素 向下调整
        void pop() {
            std::swap(_con[0], _con[_con.size() - 1]);
            _con.pop_back();
            if (!_con.empty()) adjustDown(0);
        }
        // 尾插 向上调整
        void push(const T& val) {
            _con.push_back(val);
            adjustUp((int)_con.size() - 1);
        }

        const T& top() const { return _con[0]; }
        T& top() { return _con[0]; }
        size_t size() const { return _con.size(); }
        bool empty() const { return _con.empty(); }

    private:
        Container _con;
	};

}

```

### 核心：仿函数的使用

**仿函数(functor)是像函数一样可以调用的对象，本质是重载了`operator()`的类**。好处有：

* 能**作为模板参数的类型**出现
* 比函数指针灵活（可以内联，实例化成对象）

#### 1. 纯逻辑比较/无状态仿函数

```cpp
// ============== priority_queue.h ============
template <class T>
struct myLess {               // a<b ⇒ a 的优先级低 ⇒ 形成【大顶堆】
    bool operator()(const T& a, const T& b) const { return a < b; }
};

template <class T>
struct myGreater {            // a>b ⇒ a 的优先级低 ⇒ 形成【小顶堆】
    bool operator()(const T& a, const T& b) const { return a > b; }
};


// ============== test.cpp ============
#include "queue.h"
#include "stack.h"
#include "priority_queue.h"


int main() {
	// 大顶堆 myLess return a < b
	Vect::priority_queue<int, std::vector<int>, Vect::myLess<int>> maxHeap;
	for (int arr : {5,3,1,6,20,12,60,999})
		maxHeap.push(arr);
	std::cout << "堆顶：" << maxHeap.top() << std::endl;

	// 小顶堆 myGreater return a > b
	Vect::priority_queue<int, std::vector<int>, Vect::myGreater<int>> minHeap;
	for (int arr : {5, 3, 1, 6, 20, 12, 60, 999})
		minHeap.push(arr);
	std::cout << "堆顶：" << minHeap.top() << std::endl;
	return 0;
}

```

#### 2. 自定义排序逻辑：按绝对值、按长度、按字段

按绝对值大的优先：

```cpp
// ============== priority_queue.h ============
  // 按照绝对值小的排 |a| < |b|
  template <class T>
  struct absLess {
      bool operator()(const T& a, const T& b) const { return std::abs(a) < std::abs(b); }
};

// ============== test.cpp ============
#include "queue.h"
#include "stack.h"
#include "priority_queue.h"
int main() {
    // -1 -10 -2 -6 -7 
    // 按照绝对值小的走 反而是小根堆了 如果全负数
    Vect::priority_queue<int, std::vector<int>, Vect::absLess<int>> absHeap;
    for (int arr : {-1, -10, -2, -6, -7})
        absHeap.push(arr);
    std::cout << "堆顶：" << absHeap.top() << std::endl;
    for (size_t i = 0; i < absHeap.size(); i++)
    {
        std::cout << absHeap[i] << " ";
    }
    return 0;
}
```

按字符串长度大的优先：

```cpp
// ============== priority_queue.h ============
 // 按字符串长度大的优先 a.size() < b.size()
 struct strLess {
     bool operator()(const std::string& a, const std::string& b) {
         return a.size() < b.size();
     }
 };

// ============== test.cpp ============
#include "queue.h"
#include "stack.h"
#include "priority_queue.h"
int main() {
    	// 按字符串长度大的优先 a.size() < b.size()
	Vect::priority_queue<std::string, std::vector<std::string>, Vect::strLess> strHeap;
	for (std::string strArr : {"nihao", "hhh", "1234", "1433223"})
		strHeap.push(strArr);
	std::cout << "堆顶：" << strHeap.top() << std::endl;

	return 0;
}
```

按照结构体规则排序：

```cpp
// ============== priority_queue.h ============
 // 按照结构体比较 分数高的优先 分数相同的按照名字字典序优先
 struct StudentInfo {
     int score;
     std::string name;
 };
 struct structLess {
     // 返回 true 表示 左操作数 优先级 低 于 右操作数
     bool operator()(const StudentInfo& stuA, const StudentInfo& stuB) const {
         if (stuA.score != stuB.score) return stuA.score < stuB.score; // 分数高的优先
         return stuA.name > stuB.name; // 分数相同，name 小的优先（ASCII 小在前）
     }
 };
// ============== test.cpp ============
#include "queue.h"
#include "stack.h"
#include "priority_queue.h"
int main() {
    	// 按照结构体比较 分数高的优先 分数相同的按照名字字典序优先
	Vect::priority_queue<Vect::StudentInfo,
						std::vector<Vect::StudentInfo>, 
						Vect::structLess> structHeap;

	for (const Vect::StudentInfo& stu :
		std::initializer_list<Vect::StudentInfo>{
			{91,  "coke"},
			{50,  "hhh"},
			{100, "1234"},
			{100, "22"}
		}) {
		structHeap.push(stu);
	}

	std::cout << "堆顶：" << structHeap.top().score
		<< " " << structHeap.top().name << std::endl;

	return 0;
}
```
# 5. 总结
## 两种设计模式
截至目前，我们已经掌握了两种**设计模式：** 迭代器和适配器
* **迭代器：** 
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/15e6e59f14fa4618a37e4d95d94e399c.png)
* **容器适配器：**核心作用是转换，将一个类的接口转换成用户所期望的另一个接口，而不修改底层细节，也无需关心底层细节，这也是封装的思想

## 容器适配器总结

| 适配器           | 底层默认容器 | 数据结构         | 功能                                                         |
| ---------------- | ------------ | ---------------- | ------------------------------------------------------------ |
| `stack`          | `deque`      | 栈（先进后出）   | 只允许在一端（栈顶）插入(`push`) 删除(`pop`) 访问(`top`)数据 |
| `queue`          | `deque`      | 队列（先进先出） | 允许在两端，队头删除(`pop`）、队尾插入(`push`)数据，两端都可访问数据(`front和back`) |
| `priority_queue` | `vector`     | 堆（优先级队列） | 元素出队顺序按照优先级（默认大根堆），从堆顶出数据）         |

## 写在最后

**适配器本质**：在已有容器之上封装“行为接口”，屏蔽实现细节；关注“怎么用”（LIFO/FIFO/优先级），而非“怎么存”。

**家族成员与默认底层**：

- `stack<T, deque<T>>`（只用尾部 `push/pop`）
- `queue<T, deque<T>>`（尾进头出 `push/pop`）
- `priority_queue<T, vector<T>, less<T>>`（大顶堆，`top` 最大）

**为何用 `deque`**：分块+中控表，**两端扩张 $O(1)$**、随机访问近似$O(1)$，扩容挪动数据次数少；适合仅在端点操作的 `stack/queue`。

**`deque` 要点**：

- 两端插删 ~ $O(1)$，中间插删 ~$O(n)$
- 迭代器需跨块判断，**不适合重遍历场景**（遍历密集优先 `vector`/`list`）。

**`priority_queue` 核心**：二叉堆；`push/pop` ~ $O(log n)$，`top` ~$O(1)$；比较器定义“谁优先”。

- `less<T>` ⇒ 大顶堆；`greater<T>` ⇒ 小顶堆；可自定义仿函数（绝对值、长度、结构体多字段）。

**接口**：

- `stack`：`push/pop/top/empty/size`（无迭代器）
- `queue`：`push/pop/front/back/empty/size`
- `priority_queue`：`push/pop/top/empty/size`（无遍历）

**选型指南**：行为先行——LIFO 用 `stack`，FIFO 用 `queue`，按重要度出队用 `priority_queue`；若需算法/遍历，直接用序列容器再按需封装。

