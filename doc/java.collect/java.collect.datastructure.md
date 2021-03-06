# 1.链表

# 2.栈

# 3.队列

# 4.散列表

# 5.树

- **度：** 节点的子树个数（树的度：任意节点的度的最大值）
- **层：** 根在第0层，其余为其前驱节点层数+1
- **高度：** 树中节点的最大层数
- **遍历：**
 - 前序：先双亲、再左孩子、最后右孩子
 - 中序：先左孩子、再双亲、最后右孩子
 - 后序：先左孩子、再右孩子、最后双亲
 - 层次：一层一层，从左到右、从上到下遍历
- **结构表示法：**
 - 双亲表示法：每个节点存储：数据、parent在数组中的下标
 - 孩子表示法：全部节点组成一个数组，每个数组指向一个单链表，存放其孩子
 - 双亲孩子表示法
 - 孩子兄弟表示法
- **存储结构：**
 - 顺序存储：只适用于完全二叉树
 - 链式存储：最通用的存储方法

```java
 树
 |— — 二叉树(2个子节点)
 |       |
 |       |— — 完全二叉树
 |       |        |
 |       |        |— — 堆（大/小堆）
 |       |
 |       |— — 满二叉树
 |       |
 |       |— — 二叉查找树
 |                |
 |                |— — 平衡二叉查找树
 |                         |
 |                         |— — 平衡查找树AVL树
 |                         |
 |                         |— — 平衡查找树红黑树(TreeMap,TreeSet)
 |
 |— —2-3树/2-4树(n个子节点)
          |
          |— — B树(B-树/平衡查找树)
                |
                |— — B+树
                      |
                      |— — B*树

```
#### 5.1二叉树
 1. 最大只能2个子节点
 2. 层数为i的节点最多有2的i次方个节点i >=0

![满二叉树/完全二叉树](png/full_bi_tree.png)
##### 5.2满二叉树
除最后一层无任何子节点外，每一层上的所有结点都有两个子结点。也可以这样理解
除叶子结点外的所有结点均有两个子结点。节点数达到最大值，所有叶子结点必须在同一层上

满二叉树的性质：

1. 一颗树深度为h，最大层数为k，深度与最大层数相同，k=h;
2. 叶子数为2h;
3. 第k层的结点数是：2k-1;
4. 总结点数是：2k-1，且总节点数一定是奇数

##### 5.3完全二叉树
若设二叉树的深度为h，除第 h 层外，其它各层 (1～(h-1)层) 的结点数都达到最大个数
第h层所有的结点都连续集中在最左边，这就是完全二叉树

##### 5.4二叉查找树（二叉排序树（Binary Sort Tree）或二叉搜索树）
1. 若左子树不空，则左子树上所有结点的值均小于它的根结点的值
2. 若右子树不空，则右子树上所有结点的值均大于或等于它的根结点的值
3. 没有键值相等的节点
4. 对二叉查找树进行中序遍历，即可得到有序的数列
5. 它和二分查找一样，插入和查找的时间复杂度均为O(logn)，但是在最坏的情况下仍然会有O(n)的时间复杂度
6. `二叉查找树的高度决定了二叉查找树的查找效率`

对于一般的二叉搜索树（Binary Search Tree），其期望高度（即为一棵平衡树时）为log2n，其各操作的时间复杂
度O(log2n)同时也由此而决定。但是，在某些极端的情况下（如在插入的序列是有序的时），二叉搜索树将退化成
近似链或链，此时，其操作的时间复杂度将退化成线性的，即O(n)。我们可以通过随机化建立二叉搜索树来尽量的
避免这种情况，但是在进行了多次的操作之后，由于在删除时，我们总是选择将待删除节点的后继代替它本身，这样
就会造成总是右边的节点数目减少，以至于树向左偏沉。这同时也会造成树的平衡性受到破坏，提高它的操作的时间
复杂度。于是就有了我们下边介绍的平衡二叉树
##### 5.5平衡二叉树(或称AVL树)
它是一 棵空树或它的左右两个子树的高度差的绝对值不超过1，并且左右两个子树都是一棵平衡二叉树，
平衡二叉树的常用算法有红黑树、AVL树等。在平衡二叉搜索树中，我们可以看到，其高度一般都良好地维持在O(log2n)，大大降低了操作的时间复杂度

1. 左旋转
2. 右旋转
3. 左右旋转
4. 右左旋转

![](png/avl_tree_right_rote.jpg)
![](png/avl_tree_left_right_rote.jpg)
###### 5.5.1平衡查找树AVL树
平衡因子：二叉树上节点的左子树的深度减去它右子树的深度的值
`AVL要求左右子树深度差不能超过1`
###### 5.5.2平衡查找树红黑树
![](png/red-black_tree.png)

https://zhuanlan.zhihu.com/p/24687801?refer=dreawer

http://www.tuicool.com/articles/yUf6neV

https://zhuanlan.zhihu.com/p/24810439?refer=dreawer

1. 节点是红色或黑色
2. 根是黑色
3. 所有叶子都是黑色（叶子是NIL节点）
4. 每个红色节点必须有两个黑色的子节点(从每个叶子到根的所有路径上不能有两个连续的红色节点)
5. 从任一节点到其每个叶子的所有简单路径都包含相同数目的黑色节点

#### 5.6 B树(B-树，平衡查找树)
B树作为一种多路搜索树：
1. 定义任意非叶子结点最多只有M个儿子；且M>2；
2. 根结点的儿子数为[2, M]；
3. 除根结点以外的非叶子结点的儿子数为[M/2, M]；
4. 每个结点存放至少M/2-1（取上整）和至多M-1个关键字；（至少2个关键字）
5. 非叶子结点的关键字个数=指向儿子的指针个数-1；
6. 非叶子结点的关键字：K[1], K[2], …, K[M-1]；且K[i] < K[i+1]；
7. 非叶子结点的指针：P[1], P[2], …, P[M]；其中P[1]指向关键字小于K[1]的子树，P[M]指向关键字大于K[M-1]的子树，其它P[i]指向关键字属于(K[i-1], K[i])的子树；
8. 所有叶子结点位于同一层；

![](png/btree.jpg)

###### 5.6.1 B+树
1. 其定义基本与B-树相同，除了：
2. 非叶子结点的子树指针与关键字个数相同；
3. 非叶子结点的子树指针P[i]，指向关键字值属于[K[i], K[i+1])的子树（B-树是开区间）；
4. 为所有叶子结点增加一个链指针；
5. 所有关键字都在叶子结点出现；

####### B+的性质：
1. 所有关键字都出现在叶子结点的链表中（稠密索引），且链表中的关键字恰好是有序的；
2. 不可能在非叶子结点命中；
3. 非叶子结点相当于是叶子结点的索引（稀疏索引），叶子结点相当于是存储（关键字）数据的数据层；
4. 更适合文件索引系统

![](png/b+tree.jpg)

###### 5.6.2 B*树
在B+树的非根和非叶子结点再增加指向兄弟的指针，将结点的最低利用率从1/2提高到2/3

![](png/bxingtree.jpg)

# 6.图