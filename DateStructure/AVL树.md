[TOC]



# 0. 二叉搜索树的痛点

对于一个二叉搜索树，**多次删除或插入节点的操作可能会使得二叉搜索树退化为链表**，所有的操作复杂度将由$O(logn)$变为$O(n)$

如下图所示，对于一个中序遍历的序列`[10，20，30，40，50]`，经过两次删除节点操作，这棵二叉搜索树便会退化为链表：

![二叉搜索树的退化](D:/CODE/note/%E5%9B%BE%E7%89%87/%E4%BA%8C%E5%8F%89%E6%90%9C%E7%B4%A2%E6%A0%91%E7%9A%84%E9%80%80%E5%8C%96.png)

再如下图所示，一个满二叉树经过插入节点操作，树将严重向左倾斜，查找操作的时间复杂度也随之变慢：

![满二叉树向普通二叉树演变过程](D:/CODE/note/%E5%9B%BE%E7%89%87/%E6%BB%A1%E4%BA%8C%E5%8F%89%E6%A0%91%E5%90%91%E6%99%AE%E9%80%9A%E4%BA%8C%E5%8F%89%E6%A0%91%E6%BC%94%E5%8F%98%E8%BF%87%E7%A8%8B.png)

而就有大佬提出了AVL 树，在需要频繁进行增删查改操作的场景中，AVL 树能始终保持高效的数据操作性能。

AVL 树既是二叉搜索树，也是平衡二叉树，同时满足这两类二叉树的所有性质，因此是一种平衡二叉搜索树（balanced binary search tree）

# 1. AVL树的结构

AVL树相比BS树，引入了**平衡因子**作为变量，观测这棵树是否为平衡二叉搜索树。

我们定义**平衡因子 = 右子树高度 - 左子树高度**

**AVL树的每个节点的平衡因子绝对值<=1**

而AVL树可以从BST改造而来，可以将KV模型变量用`pair`来封装，`pair`是个键值对，存储唯一的“键”和唯一对应的“值”（每个键都是唯一的，它指向一个特定的值）

那么结构如下：

```cpp
template<class K,class V>
struct AVLTreeNode {
    pair<K, V> _kv;					// 键值对
    AVLTreeNode<K, V>* _left;		// 指向左子树
    AVLTreeNode<K, V>* _right;		// 指向右子树
    AVLTreeNode<K, V>* _parent;		// 指向父节点
    int _bf;						// balanced factor 右子树高度-左子树高度

    // 构造
    AVLTreeNode(const pair<K,V>& kv)
        :_kv(kv)
            ,_left(nullptr)
            ,_right(nullptr)
            ,_parent(nullptr)
            ,_bf(0){ }
};
template<class K, class V>
class AVLTree{
public:
    typedef AVLTreeNode<K, V> Node;
private:
    Node* _root; 					// 根节点
};
```

# 2. AVL树的插入

AVL树就是在二叉搜索树的基础上引入了平衡因子，因此AVL树可以看成搜索树，插入的过程分为两步：

> 1. **按照二叉搜索树的方式插入新节点**
> 2. **调整节点的平衡因子**

```cpp
// 插入节点
bool insert(const pair<K,V>& kv){
    // 1. 按照搜索树插入逻辑插入
    Node* cur = _root;
    Node* parent = nullptr; 		// 记录cur的父节点 便于插入后链接

    // 1> 查找合适的插入位置
    // 标准二分逻辑
    while(cur){
        if(kv.first < cur->_kv.first) {				// 小于往左走
            parent = cur;
            cur = cur->_left;
        }else if(kv.first > cur->_kv.first){		// 大于往右走
            parent = cur;
            cur = cur->_right;
        }else return false;							// 相同值不插入 保持搜索树的性质
    }

    // 2> 执行插入操作
    // cur的位置就是待插入位置
    Node* newNode = new Node(kv)
        if(parent->_left == cur) parent->_left = kv.first;
    else parent->_right = kv.first;

    // 2. 更新平衡因子
    /*
			cur插入之后，parent的平衡因子一定要调整
			插入之前， parent的平衡因子分为 0 1 -1 三种情况

			parent		    &
				           /
			newNode(左) 	  &
			cur插入到parent的左侧 parent._bf-1

			parent		    &
							 \
			newNode(右) 	      &
			cur插入到parent的左侧 parent._bf+1

			插入之后，parent的平衡因子可能有 0 +-1 +-2 这三种情况
			1> parent._bf==0  	说明插入之前为+-1 插入之后调整为0 满足AVL树 插入成功
			2> parent._bf==+-1 	说明插入前一定为0 插入之后调整为+-1 
								此时以parent为根的树高度增加 需要向上更新
			3> parent._bf==2	违反了AVL树性质 旋转处理
		*/

    while(parent){
        // 更新父节点的平衡因子
        if(cur == parent->_left) parent->_bf--;
        else parent->_right++;

        // 更新后检测父节点的平衡因子
        if(0 == parent->_bf) break; 	
        else if(1 == parent->_bf || -1 == parent->_bf){
            // 插入前父节点的平衡因子是0 说明是以parent为根的二叉树
            // 高度增加了一层 需要继续向上调整
            cur = parent;
            parent = cur->_parent;
        }
        else{
            // 父节点的平衡因子为+-2 旋转操作
            if(2 == parent->_bf){

            }else{

            }
        }

        return true;
    }
```

# 3. 旋转操作

## 3.1. 新节点插入到较高左子树的左侧LL——右旋

![右旋](D:/CODE/note/%E5%9B%BE%E7%89%87/%E5%8F%B3%E6%97%8B.png)

从底至顶看，二叉树中首个失衡节点是节点30，我们关注以该失衡节点为根节点的子树，将该节点记为 `node`，其左子节点记为`child` ，执行右旋操作。完成右旋后，子树恢复平衡，并且仍然保持二叉搜索树的性质

当节点`child`有右子节点（记为`childKid` ）时，需要在右旋中添加一步：将`childKid`作为`node`的左子节点

![右旋child有右孩子](D:/CODE/note/%E5%9B%BE%E7%89%87/%E5%8F%B3%E6%97%8Bchild%E6%9C%89%E5%8F%B3%E5%AD%A9%E5%AD%90.png)

再来看普适性的情况：

![右旋普遍情况](D:/CODE/note/%E5%9B%BE%E7%89%87/%E5%8F%B3%E6%97%8B%E6%99%AE%E9%81%8D%E6%83%85%E5%86%B5.png)

在上图插入之前，AVL树是平衡的，新节点插入到B的左子树（注意：不是左孩子），B的左子树增加一层，导致以A为根的二叉树不平衡，要让A平衡，只能==将A的左子树减少一层，右子树增加一层==

即左子树上提，A以B为支撑转下来，因为A>B，只能放到B的右子树，而B如果有右子树，那么一定大于B小于A，所以将其放到A的左边，旋转完成，更新平衡因子

以上是具体的旋转过程，而写代码时，需要考虑到：

> 1. **B节点的右孩子可能存在也可能不存在**
> 2. **A可能是根节点，也可能是子树**
>    1. 根节点： 旋转完成后更新根节点
>    2. 子树，可能是某个节点的左子树or右子树

```cpp
// 右旋
void rotateRight(Node* node){
    Node* child = node->_left;			// 失衡节点的左孩子
    Node* childKid = child->_right; 	// 失衡节点左孩子的右孩子
    Node* nodeParent = node->_parent;	// 失衡节点的父节点

    // 右旋操作 
    // 1). 以child为支撑点旋转 node成为child的右孩子
    // 2). childNext成为node的左孩子
    /*
		node -->	50                           30
				   /  \           右旋          /  \
		child --> 30  60        =====>        10   50
				 /  \                        /    /  \
				10  40 <-- childKid         0   40   60
			   /
			  0
		*/	

    // 1. 下沉node 提高child
    child->_right = node;
    node->_parent = child;

    node->_left = childKid;
    if(childKid) childKid->_parent = node;

    // 2.接回父节点和根
    child->_parent = nodeParent;
    if(nodeParent == nullptr) _root = child;
    else if(nodeParent->_left == node) nodeParent->_left = child;
    else nodeParent->_right = child;

    // 3. 更新平衡因子
    node->_bf = child->_bf = 0;
}
```

## 3.2. 新节点插入较高右子树的右侧RR——左旋

左旋解决的问题关于右旋对称，如下图所示：

![左旋](D:/CODE/note/%E5%9B%BE%E7%89%87/%E5%B7%A6%E6%97%8B.png)

![左旋child有左孩子](D:/CODE/note/%E5%9B%BE%E7%89%87/%E5%B7%A6%E6%97%8Bchild%E6%9C%89%E5%B7%A6%E5%AD%A9%E5%AD%90.png)

![左旋 普遍情况](D:/CODE/note/%E5%9B%BE%E7%89%87/%E5%B7%A6%E6%97%8B%20%E6%99%AE%E9%81%8D%E6%83%85%E5%86%B5.png)

而对于代码书写，也同样对称，只需要把所有的`_left`换成`_right`，把所有的`_right`换成`_left`即可：

```cpp
// 左旋
void rotateLeft(Node* node){
    Node* child = node->_right;			// 失衡节点的右孩子
    Node* childKid = child->_left; 		// 失衡节点左孩子的左孩子
    Node* nodeParent = node->_parent;	// 失衡节点的父节点

    // 左旋操作 
    // 1). 以child为支撑点旋转 node成为child的左孩子
    // 2). childNext成为node的右孩子
    /*
			node -->	 10									 40
						/  \								/  \
					   0   40    <--- child	 左旋	      	  10  50
						  /  \			    =====>		  /  \   \
		  childNext ---> 30  50				             0   30  60
							   \
							   60     		 
		*/

    // 1. 下沉node 提高child
    child->_left = node;
    node->_parent = child;

    node->_right = childKid;
    if(childKid) childKid->_parent = node;

    // 2.接回父节点和根
    child->_parent = nodeParent;
    if(nodeParent == nullptr) _root = child;
    else if(nodeParent->_left == node) nodeParent->_left = child;
    else nodeParent->_right = child;

    // 3. 更新平衡因子
    node->_bf = child->_bf = 0;
}
```

## 3.3. 新节点插入较高左子树的右侧LR——先左旋后右旋

![先左旋后右旋](D:/CODE/note/%E5%9B%BE%E7%89%87/%E5%85%88%E5%B7%A6%E6%97%8B%E5%90%8E%E5%8F%B3%E6%97%8B.png)

这里我们就可以复用之前实现的左旋和右旋代码了，但是，需要更新平衡因子，更新平衡因子需要考虑到以下几点，请看图：

* `childKid`左边插入

![LR左边插入](D:/CODE/note/%E5%9B%BE%E7%89%87/LR%E5%B7%A6%E8%BE%B9%E6%8F%92%E5%85%A5.png)

* `childKid`右边插入

![LR右边插入](D:/CODE/note/%E5%9B%BE%E7%89%87/LR%E5%8F%B3%E8%BE%B9%E6%8F%92%E5%85%A5.png)

代码实现：

```cpp
// 先左旋后右旋
void rotateLR(Node* node){
    Node* child = node->_left;
    Node* childKid = child->_right;
    int bf = childKid->_bf;

    rotateLeft(child);
    rotateRight(node);

    // 更新平衡因子
    if(0 == bf){
        child->_bf = 0;
        node->_bf = 0;
        childKid->_bf = 0;
    }
    else if(1 == bf){			// 右边插入
        child->_bf = -1;
        node->_bf = 0;
        childKid->_bf = 0;
    }
    else if(-1 == bf){			// 左边插入
        child->_bf = 0;
        node->_bf = 1;
        childKid->_bf = 0;
    }
    else assert(false);
}
```

## 3.4. 新节点插入较高右子树的左侧RL——先右旋后左旋

同理，具体步骤请看图：

![先右旋再左旋](D:/CODE/note/%E5%9B%BE%E7%89%87/%E5%85%88%E5%8F%B3%E6%97%8B%E5%86%8D%E5%B7%A6%E6%97%8B.png)

![RL左边插入](D:/CODE/note/%E5%9B%BE%E7%89%87/RL%E5%B7%A6%E8%BE%B9%E6%8F%92%E5%85%A5.png)

![RL右边插入](D:/CODE/note/%E5%9B%BE%E7%89%87/RL%E5%8F%B3%E8%BE%B9%E6%8F%92%E5%85%A5.png)

代码实现：

```cpp
// 先右旋后左旋
void rotateRL(Node* node){
    Node* child = ndoe->_right;
    Node* childKid = child->_left;
    int bf = childKid->_bf;			//  小孙子一路往上 平衡因子先记录下来

    rotateRight(child);
    rotateLeft(node);

    // 更新平衡因子
    if(0 == bf){		// 自己是新增
        child->_bf = 0;
        childKid->_bf = 0;
        node->_bf = 0;
    }
    else if(1 == bf){ // 右边插入
        child->_bf = 0;
        childKid->_bf = 0;
        node->_bf = -1;
    }
    else if(bf == -1){ // 左边插入
        child->_bf = 1;
        childKid->_bf = 0;
        node->_bf = 0;
    }
    else assert(false);
}

```

## 3.5. 旋转的选择

![旋转方式的选择](D:/CODE/note/%E5%9B%BE%E7%89%87/%E6%97%8B%E8%BD%AC%E6%96%B9%E5%BC%8F%E7%9A%84%E9%80%89%E6%8B%A9.png)

假如以`node`为根的子树不平衡，即`node`的平衡因子为2或-2，分以下情况考虑：

1. `node`的平衡因子为2，说明`node`的右子树高，设其右子树为`child`
   1. 当`child`的平衡因子为1时，左旋
   2. 当`child`的平衡因子为-1时，先右旋后左旋
2. `node`的平衡因子为-2，说明`node`的左子树高，设其左子树为`child`
   1. 当`child`的平衡因子为-1时，右旋
   2. 当`child`的平衡因子为1时，先左旋后右旋

旋转完成后，原`node`为根的子树高度降低，已经平衡，不需要再向上更新
