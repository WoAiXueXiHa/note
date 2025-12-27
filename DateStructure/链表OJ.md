[TOC]



# 1. 203.移除链表元素元素

[203. 移除链表元素 - 力扣（LeetCode）](https://leetcode.cn/problems/remove-linked-list-elements/description/)

思路：创建一个哨兵位指向`head`，创建一个`cur`临时指针，遍历链表，遇到等于`val`的节点跳过即可，最后返回`sentinel->next`即链表的头节点

![image-20250901111024299](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20250901111024299.png)

创建一个哨兵位就避免了讨论`head`是否为空

代码实现：

```c++
class Solution {
public:
    ListNode* removeElements(ListNode* head, int val) {
        // 创建哨兵位
        ListNode* sentinel = new ListNode(-1);
        sentinel->next = head;
        // 创建临时指针遍历链表
        ListNode* cur = sentinel;
        // 遍历链表，跳过等于val的节点
        while(cur->next){
            if(cur->next->val == val){
                ListNode* tmp = cur->next;
                // 跳过tmp节点，并删除
                cur->next = cur->next->next;
                delete tmp;
            }
            else{
                cur = cur->next;
            }
        }

        // 创建返回值
        ListNode* ret = sentinel->next;
        delete sentinel;
        return ret;
    }
};
```

# 2. 206 反转链表

[206. 反转链表 - 力扣（LeetCode）](https://leetcode.cn/problems/reverse-linked-list/description/)

思路：创建三个指针`n1 n2 n3`，`n1`指向空，`n2`指向头，`n3`指向头的`next`，三个指针不断向后移动，改变原链表的指针指向

![image-20250901112454775](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20250901112454775.png)

```c++
class Solution {
public:
    ListNode* reverseList(ListNode* head) {
        if(head == nullptr) return nullptr;  // 考虑到链表为空的情况

        // 创建三个指针
        ListNode* n1,*n2,*n3;
        n1 = nullptr;
        n2 = head;
        if(head->next) n3 = head->next;

        while(n2){
            n2->next = n1;
            n1 = n2;
            n2 = n3;
            // n3走到空之后就没有必要继续走了
            if(n3) n3 = n3->next;
        }

        return n1;
    }
};

```

注意循环条件的控制：`n2走到空就结束循环`，而`n3`的移动由自己控制，`n3`在最后，如果走到空，就不能再往后走了，停止移动即可

# 3. 876 链表的中间节点

思路：快慢指针法，快指针走两步，慢指针走一步，形成了距离差，快指针走到空，慢指针所在位置即是中间节点

![image-20250901113247548](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20250901113247548.png)

```c++
class Solution {
public:
    ListNode* middleNode(ListNode* head) {
        ListNode* fast, *slow;
        fast = slow = head;

        while(fast && fast->next){
            fast = fast->next->next;
            slow = slow->next;
        }
        return slow;
    }
};
```

# 4. 21 合并两个有序链表

[21. 合并两个有序链表 - 力扣（LeetCode）](https://leetcode.cn/problems/merge-two-sorted-lists/description/)

思路：创建一个带哨兵位的链表，同时遍历`list1`和`list2`，小的尾插到新链表上，注意：总会有一个链表先走完，另一个链表剩余节点尾插到`cur`之后即可

![image-20250901114733726](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20250901114733726.png)

```c++
class Solution {
public:
    ListNode* mergeTwoLists(ListNode* list1, ListNode* list2) {
        ListNode* sentinel = new ListNode(-1);
        ListNode* cur = sentinel;

        while(list1 && list2){
            if(list1->val <= list2->val){
                cur->next = list1;
                list1 = list1->next;
            }else{
                cur->next = list2;
                list2 = list2->next;
            }
            cur = cur->next;
        }

        // 总会有一个链表先走完，另外一个链表剩余节点直接尾插到cur之后即可
        ListNode* tail = list1 == nullptr ? list2 : list1;
        cur->next = tail;

        // 返回最后拼装的节点
        ListNode* ret = sentinel->next;
        delete sentinel;
        return ret;
    }
};
```



# 5. 面试题02.04 分割链表

[面试题 02.04. 分割链表 - 力扣（LeetCode）](https://leetcode.cn/problems/partition-list-lcci/)

思路：创建一个大链表和一个小链表，遍历原链表，大于等于x的节点尾插到大链表，小于x的节点尾插到小链表，小链表的尾链接大链表的头

<video src="D:\CODE\NewDateStructure\list_OJ\9月1日.mp4" controls></video>

```c++
class Solution {
public:
    ListNode* partition(ListNode* head, int x) {
        ListNode* sentinelB = new ListNode(-1);
        ListNode* sentinelS = new ListNode(-1);
        if(head == nullptr) return nullptr;

        ListNode* cur = head;
        ListNode* curB = sentinelB;
        ListNode* curS = sentinelS;

        while(cur){
            if(cur->val < x){
                curS->next = cur;
                curS = curS->next;
            }else{
                curB->next = cur;
                curB = curB->next;
            }
            cur = cur->next;
        }
        // 处理带环问题 
        curB->next = nullptr;
        curS->next = sentinelB->next;

        ListNode* ret = sentinelS->next;
        delete sentinelB;
        delete sentinelS;

        return ret;
    }
};
```



# 6. OR36 回文链表

[链表的回文结构_牛客题霸_牛客网](https://www.nowcoder.com/practice/d281619e4b3e4a60a2cc66ea32855bfa?tpId=49&&tqId=29370&rp=1&ru=/activity/oj&qru=/ta/2016test/question-ranking)

思路：逆置后半段，比较两个链表是否相同，相同即是回文链表

1. 找中间节点：快慢指针
2. 逆置：三个指针改变链表指向
3. 判断回文：遍历链表

![image-20250901153302874](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20250901153302874.png)

```c++
class PalindromeList {
public:
    // 找中间节点
    ListNode* midNode(ListNode* A){
        // 空链表处理
        if(A == nullptr) return nullptr;

        // 快慢指针
        ListNode* fast, *slow;
        fast = slow = A;
        while(fast && fast->next){
            fast = fast->next->next;
            slow = slow->next;
        }
        return slow;
    }
    // 逆置链表
    ListNode* inverseNode(ListNode* A){
        // 空链表处理
        if(A == nullptr) return nullptr;

        ListNode* n1,*n2,*n3;
        n1 = nullptr;
        n2 = A;
        n3 = A->next;

        while(n2){
            n2->next = n1;
            n1 = n2;
            n2 = n3;
            if(n3) n3 = n3->next;
        }

        return n1;
    }
    bool chkPalindrome(ListNode* A) {
        // 找中间节点
        ListNode* mid = midNode(A);
        // 逆置后半段
        ListNode* inversion = inverseNode(mid);

        // 遍历两个链表
        ListNode* curTail = inversion; // 后半段链表
        ListNode* curHead = A;// 前半段链表

        while(curHead && curTail){
            if(curHead->val != curTail->val){
                return false;
            }
            curHead = curHead->next;
            curTail = curTail->next;
        }
        return true;
    }
};
```

# 7. 160 相交链表

[160. 相交链表 - 力扣（LeetCode）](https://leetcode.cn/problems/intersection-of-two-linked-lists/description/)

![image-20250901162751443](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20250901162751443.png)

```c++
class Solution {
public:
    bool isCrossing(ListNode* headA, ListNode* headB){
        if(headA == nullptr || headB == nullptr) return false;

        ListNode* curA = headA;
        ListNode* curB = headB;

        while(curA->next) curA = curA->next;
        while(curB->next) curB = curB->next;

        return curB == curA;
    }

    ListNode *getIntersectionNode(ListNode *headA, ListNode *headB) {
        bool ret = isCrossing(headA, headB);
        if(ret == false) return nullptr;

        // 找两个链表的长度
        size_t lenA = 0, lenB = 0;
        ListNode* curA = headA;
        ListNode* curB = headB;
        while(curA->next){
            ++lenA;
            curA = curA->next;
        }

        while(curB->next){
            ++lenB;
            curB = curB->next;
        }

        // 长的链表先走长度查
        size_t gap = std::abs(static_cast<long>(lenA - lenB));
        ListNode* longList = headA;
        ListNode* shortList = headB;
        if(lenB > lenA){
            longList = headB;
            shortList = headA;
        }

        while(gap--) longList = longList->next;
        
        // 现在两个链表同时走
        while(longList != shortList){
            longList = longList->next;
            shortList = shortList->next;
        }
        return longList;
    }
};
```

# 8. 141 142 环形链表 

[141. 环形链表 - 力扣（LeetCode）](https://leetcode.cn/problems/linked-list-cycle/description/)

思路：快慢指针，快指针会先进环，慢指针后进环
如果链表带环，快指针一定会和慢指针相遇，反之则不会

![image-20250901164425908](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20250901164425908.png)

```c++
class Solution {
public:
bool hasCycle(ListNode *head) {
    if(head == nullptr) return false;
    ListNode* fast, *slow;
    fast = slow = head;

    while(fast && fast->next){
        fast = fast->next->next;
        slow = slow->next;
        if(fast == slow) return true;
    }
    return false;
}
};
```

## 链表带环，`fast`走两步，`slow`走一步一定能追上吗？

`fast`先进环，`slow`后进环，假设`slow`进环时与`fast`距离为$N$

每次运动，二者距离减一

这是二者距离变化：

![image-20250901171328448](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20250901171328448.png)

所以`fast`走两步，`slow`一定能追上

## 链表带环，`fast`走三步，`slow`走一步一定能追上吗？

`fast`先进环，`slow`后进环，假设`slow`进环时与`fast`距离为$N$

每次运动，二者距离减二

这是二者距离变化：

![image-20250901171816365](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20250901171816365.png)

假设环长为$C$，若$C - 1$为偶数，下一轮就追上了，若$C - 1$为奇数，则仍需分析

考略一下：永远追不上的条件：$C - 1$为奇数并且$N$为奇数

这个条件能成立吗？

我们先来看一道题铺垫一下：

[142. 环形链表 II - 力扣（LeetCode）](https://leetcode.cn/problems/linked-list-cycle-ii/)

假设链表头到环的入口点距离为$L$，环长$C$，入口点到相遇点距离$X$ 

此时`fast`走两步，`slow`走一步

相遇时：

`slow` 走了$L+X$，

`fast`已经绕环$x$圈，`fast`走了$L+xC+X$

`fast`走两步，`slow`走一步，二者可以得到距离关系：

$L+xC+X= 2(L+X)$

化简可得：

$L = (x-1)C + (C-X)$

![image-20250901173940300](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20250901173940300.png)

得到结论：**一个指针从链表头开始走，一个指针从相遇点开始走，最终会在环的入口点相遇（两个指针走一样的步长）**

所以，对于这道题目，先找相遇点，然后创建两个新的指针，一个从链表头开始，一个从相遇点开始，同时走，相遇的节点就是入口点

```c++
class Solution {
public:
    ListNode *detectCycle(ListNode *head) {
        // 处理空链表
        if(head == nullptr) return nullptr;
        // 创建快慢指针找相遇点
        ListNode* fast = head, *slow = head;
        // 相遇点
        ListNode* meet = nullptr;
        // 找相遇点
        while(fast && fast->next){
            slow = slow->next;
            fast = fast->next->next;
            if(fast == slow){
                // 一个从链表头 一个从相遇点
                meet = slow;
                while(meet != head){
                    meet = meet->next;
                    head = head->next;
                }
                return meet;
            }
        }
        return nullptr;
    }
};
```

现在回到原来的问题，

## `fast`走三步，`slow`走一步，永远追不上的条件：$C - 1$为奇数并且$N$为奇数成立吗？

假设链表头到入口点距离$L$，环长$C$，`slow`进环时，距离`fast`为$N$，此时`fast`已经走了$x$圈，并多走了$C-N$

根据数量关系可得：

$3L = L + xC + C-N$	

化简得：

$2L = (x+1)C - N$

$C - 1$为奇数，那么$C$就是偶数，任意数乘以偶数还是偶数，$N$为奇数

根据等式： 偶数 = 偶数 - 奇数    这是个错误结论！所以我们之前的设想：永远追不上的猜测是错误的，`fast`走三步，`slow`走一步也可以追上！

# 9.  138 随机链表的复制

[138. 随机链表的复制 - 力扣（LeetCode）](https://leetcode.cn/problems/copy-list-with-random-pointer/description/)

思路：

1. 复制原节点，构建拼接链表，例如：原链表为$node1->node2->...$  ，

   构建拼接的链表就是$node1->node1_{copy}->node2->node2_{copy}->...$

 ![image-20250901202830013](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20250901202830013.png)

2. 控制拷贝节点的`random`：拷贝节点的`random`是原节点`random`的`next`

![image-20250901202923750](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20250901202923750.png)

3. 拆下拷贝链表，尾插形成拷贝链表，并恢复原链表

  ![image-20250901202946960](C:\Users\hp\AppData\Roaming\Typora\typora-user-images\image-20250901202946960.png)

```c++
class Solution {
public:
    Node* copyRandomList(Node* head) {
        // 1. 复制节点
        // 处理空链表
        if(head == nullptr) return nullptr;

        Node* cur = head;
        while(cur){
            Node* copy = new Node(cur->val);
            copy->next = cur->next;
            cur->next = copy;

            cur = cur->next->next;
        }

        // 2. 控制拷贝节点的random
        cur = head;
        while(cur){
            Node* copy = cur->next;  
            if(cur->random) 
                copy->random = cur->random->next;
            cur = cur->next->next;
        }

        // 3. 拆下拷贝链表 尾插形成新链表 并恢复原链表
        Node* sentinel = new Node(-1);
        Node* copyCur = sentinel;  // 用于构建新链表
        cur = head;

        while(cur){
            Node* copy = cur->next;
            Node* next = copy->next;

            // 尾插
            copyCur->next = copy;
            copyCur = copy;
            
            // 恢复原链表
            cur->next = next;
            cur = next;
        }
        
        Node* ret = sentinel->next;
        delete sentinel;
        return ret;
    }
};
```

