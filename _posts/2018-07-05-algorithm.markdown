---
layout:    post
title:      "数据结构和算法总结"
subtitle:   "Data Structure and algorithm"
date:       2018-07-05 12:00:00
author:     "scyhssm"
header-img: "img/data-structure-algorithm.jpeg"
header-mask: 0.3
catalog:    true
tags:
    - Java
---

> 对复习的算法和数据结构简单总结

1.二叉平衡树实现要点
* 不平衡的情况分为LL，LR，RL，RR四种情况，旋转的代码根据这四种情况写
* 树除了要保存左右节点以及节点值以外还要保存高度，用以快速查询
* 一般情况下将根结点都不存在的空二叉树高度定义为0，而存在一个根结点高度为1

LL的旋转代码为:
```
static Node* left_left_rotation(AVLTree k2)
{
    AVLTree k1;

    k1 = k2->left;
    k2->left = k1->right;    //要将左子树的右节点连到父节点上
    k1->right = k2;

    k2->height = MAX( HEIGHT(k2->left), HEIGHT(k2->right)) + 1;    //必须先求k2高度，再求k1高度
    k1->height = MAX( HEIGHT(k1->left), k2->height) + 1;

    return k1;
}
```
RR旋转代码同理：
```
static Node* right_right_rotation(AVLTree k1)
{
    AVLTree k2;

    k2 = k1->right;
    k1->right = k2->left;
    k2->left = k1;

    k1->height = MAX( HEIGHT(k1->left), HEIGHT(k1->right)) + 1;
    k2->height = MAX( HEIGHT(k2->right), k1->height) + 1;

    return k2;
}
```
LR的旋转，不只涉及到k1和k2两个节点了，还涉及到他们的父节点k3，旋转操作可以复用LL和RR：
```
static Node* left_right_rotation(AVLTree k3)
{
    k3->left = right_right_rotation(k3->left);

    return left_left_rotation(k3);
}
```
RL同LR：
```
static Node* right_left_rotation(AVLTree k1)
{
    k1->right = left_left_rotation(k1->right);

    return right_right_rotation(k1);
}
```
插入代码是一个递归的过程，从低到高调整：
```
Node* avltree_insert(AVLTree tree, Type key)
{
    if (tree == NULL)
    {
        // 新建节点
        tree = avltree_create_node(key, NULL, NULL);
        if (tree==NULL)
        {
            printf("ERROR: create avltree node failed!\n");
            return NULL;
        }
    }
    else if (key < tree->key) // 应该将key插入到"tree的左子树"的情况
    {
        tree->left = avltree_insert(tree->left, key);
        // 插入节点后，若AVL树失去平衡，则进行相应的调节。调整是一个递归的过程
        if (HEIGHT(tree->left) - HEIGHT(tree->right) == 2)
        {
            if (key < tree->left->key)
                tree = left_left_rotation(tree);
            else
                tree = left_right_rotation(tree);
        }
    }
    else if (key > tree->key) // 应该将key插入到"tree的右子树"的情况
    {
        tree->right = avltree_insert(tree->right, key);
        // 插入节点后，若AVL树失去平衡，则进行相应的调节。
        if (HEIGHT(tree->right) - HEIGHT(tree->left) == 2)
        {
            if (key > tree->right->key)
                tree = right_right_rotation(tree);
            else
                tree = right_left_rotation(tree);
        }
    }
    else //key == tree->key)
    {
        printf("添加失败：不允许添加相同的节点！\n");
    }

    // 更新树的高度
    tree->height = MAX( HEIGHT(tree->left), HEIGHT(tree->right)) + 1;

    return tree;
}
```
删除节点的操作，关键在于从左右子树中找到高度更高的树，左子树找最大点替换根，右子树找最小点替换，这样还是平衡的状态，由于原本是平衡的，高的删除一个节点仍旧保持平衡，令操作简化：
```
static Node* delete_node(AVLTree tree, Node *z)
{
    // 根为空 或者 没有要删除的节点，直接返回NULL。
    if (tree==NULL || z==NULL)
        return NULL;

    if (z->key < tree->key)        // 待删除的节点在"tree的左子树"中
    {
        tree->left = delete_node(tree->left, z);
        // 删除节点后，若AVL树失去平衡，则进行相应的调节。
        if (HEIGHT(tree->right) - HEIGHT(tree->left) == 2)
        {
            Node *r =  tree->right;
            if (HEIGHT(r->left) > HEIGHT(r->right))
                tree = right_left_rotation(tree);
            else
                tree = right_right_rotation(tree);
        }
    }
    else if (z->key > tree->key)// 待删除的节点在"tree的右子树"中
    {
        tree->right = delete_node(tree->right, z);
        // 删除节点后，若AVL树失去平衡，则进行相应的调节。
        if (HEIGHT(tree->left) - HEIGHT(tree->right) == 2)
        {
            Node *l =  tree->left;
            if (HEIGHT(l->right) > HEIGHT(l->left))
                tree = left_right_rotation(tree);
            else
                tree = left_left_rotation(tree);
        }
    }
    else    // tree是对应要删除的节点。
    {
        // tree的左右孩子都非空
        if ((tree->left) && (tree->right))
        {
            if (HEIGHT(tree->left) > HEIGHT(tree->right))
            {
                // 如果tree的左子树比右子树高；
                // 则(01)找出tree的左子树中的最大节点
                //   (02)将该最大节点的值赋值给tree。
                //   (03)删除该最大节点。
                // 这类似于用"tree的左子树中最大节点"做"tree"的替身；
                // 采用这种方式的好处是：删除"tree的左子树中最大节点"之后，AVL树仍然是平衡的。
                Node *max = avltree_maximum(tree->left);
                tree->key = max->key;
                tree->left = delete_node(tree->left, max);
            }
            else
            {
                // 如果tree的左子树不比右子树高(即它们相等，或右子树比左子树高1)
                // 则(01)找出tree的右子树中的最小节点
                //   (02)将该最小节点的值赋值给tree。
                //   (03)删除该最小节点。
                // 这类似于用"tree的右子树中最小节点"做"tree"的替身；
                // 采用这种方式的好处是：删除"tree的右子树中最小节点"之后，AVL树仍然是平衡的。
                Node *min = avltree_maximum(tree->right);
                tree->key = min->key;
                tree->right = delete_node(tree->right, min);
            }
        }
        else
        {
            Node *tmp = tree;
            tree = tree->left ? tree->left : tree->right;
            free(tmp);
        }
    }

    return tree;
}

/*
 * 删除结点(key是节点值)，返回根节点
 *
 * 参数说明：
 *     tree AVL树的根结点
 *     key 待删除的结点的键值
 * 返回值：
 *     根节点
 */
Node* avltree_delete(AVLTree tree, Type key)
{
    Node *z;

    if ((z = avltree_search(tree, key)) != NULL)
        tree = delete_node(tree, z);
    return tree;
}
```
2.红黑树
* 每个节点只有红黑两种颜色，根结点黑色，叶子结点黑色
* 一个节点为红色，其子节点必为黑色
* 从一个节点到该节点的叶子节点所有路径上黑节点数目相同（保证了红色节点不会连续出现两次）
* Java集合中的TreeSet和TreeMap都由红黑树实现
* 红黑树是2-3-4树多一种等同
红黑树解决不平衡的问题都可以在三步内解决，插入和删除效率很高，查询稍逊于AVL，会比AVL多一层。

3.二叉堆
* 二叉堆总是一棵完全树
* 任意节点值总是不大于或不小于子节点的值
构造堆从有子节点的位置开始从后往前依次调整，一直构造到最顶上的点。用向下调整的办法。

向上调整和插入代码：
```
static void maxheap_filterup(int start)
{
    int c = start;            // 当前节点(current)的位置
    int p = (c-1)/2;        // 父(parent)结点的位置
    int tmp = m_heap[c];        // 当前节点(current)的大小

    while(c > 0)
    {
        if(m_heap[p] >= tmp)
            break;
        else
        {
            m_heap[c] = m_heap[p];
            c = p;
            p = (p-1)/2;   
        }       
    }
    m_heap[c] = tmp;
}
int maxheap_insert(int data)
{
    // 如果"堆"已满，则返回
    if(m_size == m_capacity)
        return -1;

    m_heap[m_size] = data;        // 将"数组"插在表尾
    maxheap_filterup(m_size);    // 向上调整堆
    m_size++;                    // 堆的实际容量+1

    return 0;
}
```
向下调整和删除代码：
```
int get_index(int data)
{
    int i=0;

    for(i=0; i<m_size; i++)
        if (data==m_heap[i])
            return i;

    return -1;
}
static void maxheap_filterdown(int start, int end)
{
    int c = start;          // 当前(current)节点的位置
    int l = 2*c + 1;     // 左(left)孩子的位置
    int tmp = m_heap[c];    // 当前(current)节点的大小

    while(l <= end)
    {
        // "l"是左孩子，"l+1"是右孩子
        if(l < end && m_heap[l] < m_heap[l+1])
            l++;        // 左右两孩子中选择较大者，即m_heap[l+1]
        if(tmp >= m_heap[l])
            break;        //调整结束
        else
        {
            m_heap[c] = m_heap[l];
            c = l;
            l = 2*l + 1;   
        }       
    }   
    m_heap[c] = tmp;
}
int maxheap_remove(int data)
{
    int index;
    // 如果"堆"已空，则返回-1
    if(m_size == 0)
        return -1;

    // 获取data在数组中的索引
    index = get_index(data);
    if (index==-1)
        return -1;

    m_heap[index] = m_heap[--m_size];        // 用最后元素填补
    maxheap_filterdown(index, m_size-1);    // 从index位置开始自上向下调整为最大堆

    return 0;
}
```
4.图的概念

邻接点：一条边上的两个顶点叫做邻接点，无论有向图还是无向图都存在邻接点的概念。

连通图：无向图任意两顶点间都存在无向路径，称其为连通图。有向图图中任意两个顶点存在一条有向路径，称其为强连通图。

连通分量：非连通图中各个子连通图称为连通分量。

邻接矩阵：矩阵表示图中顶点之间关系（权或者是否存在弧或边）

邻接表：邻接表对稀疏矩阵更省空间，把边用数组+每个数组一个链表的形式保存起来，缺点是没有随机存取的速度。判断两个点是否有边要花费一点时间。基于这个缺点改进出了十字链表法等。

拓扑排序：将有向无环图进行排序进而得到的一个有序的线形序列。检测拓扑图中是否存在环的方法是先将无父节点的顶点入队列，维护一个出度数组，有顶点出队列则将该顶点出度相关的顶点父顶点个数-1，如果为0则无父顶点，入队列。一直到走完为止，看是否所有的点都走过进队列流程了，都走过则无环。

最小生成树：在n个顶点的连通图中选择最小的n-1条边使其构成一个极小连通子图，称为最小生成树。

克鲁斯卡尔算法：不断在连通子图里找最小的边加入边集，找n-1条边，条件边不会构成环。

普里姆算法：根据随机选择的连通图中的顶点开始延展，不断挑选不构成环的最小边加入。

Dijsktra算法：从一个顶点开始找每个顶点到该顶点最近的距离是多少，可以标记路径。初始化起点到各个点之间的距离，每次添加一个之前没添加过的顶点，条件是这条路径必须是可得到的最短路径，基于这条最短路径更新路径，进行下一轮迭代。

5.排序

冒泡：O(N^2)时间复杂度，稳定的排序算法。从前往后比较依次比较值，交换逆序的值。如果一轮冒泡中没有产生交换可以认为已经有序，不需要继续进行直接跳出。在循环中加标记。

快排：最坏O(N^2),平均O(N*lgN),不稳定排序算法。挑出一个基准值，把比基准值小的放前面，基准值大的放后面，按基准值前后继续进行递归。注意不是交换而是直接覆盖。

直接插入：O(N^2)，稳定。从未排序的序列中获得第一个值，插入已排序序列中的合适位置。

选择排序：O(N^2)，不稳定。从待排序序列中找到一个最值，和给定位置的值交换。

希尔排序：最差增量为1时间复杂度为O(N^2),Hibbard增量的希尔排序的时间复杂度为O(N3/2)，希尔排序的时间复杂度不好估计，和增量有关，不稳定。按照不同的增量分别插入排序，直到增量为1.

堆排序：O(NlogN),不稳定。需了解堆结构，先构造堆，每次令头和尾交换，获得头加入到排序数组中，去掉尾从头开始向下调整，执行下一轮迭代。

归并：O(NlgN),稳定。将两个有序数列合并成一个有序数列，有由下而上和由上而下两种归并方法。

桶排序：数值均匀分配O(N),桶排序不是比较排序，不收到O(NlgN)影响。设置一个定量数组当空桶，寻访序列把项目一个个放到桶中，每个不是空的桶进行排序，从不是空的桶里把项目放回序列。

基数排序：时间复杂度O(k*n),n是排序元素个数，k是数字位数。基数排序是桶排序的扩展。加入是数字个位十位百位从小到大分到数字桶里，在个位的时候分到桶中将数字串在一起，对十位分到桶后将数字串在一起，循环计算。

6.KMP算法

找到在一串字符串中是否含有子字符串。求出子字符串的next数组可以在不匹配的时候移动尽可能少的步数，使用下一个位置匹配。求next数组的算法：
```
public void getNext(String s){
    int[] next = new int[s.length()];
    next[0]=-1;
    int i=0;
    int j=-1;
    while (i<s.length()-1){
        if (j==-1||s.charAt(i)==s.charAt(j)){
            j++;
            i++;
            next[i]=j;
        }
        else
            j = next[j];
    }
}
```
匹配两个字符串的方法:
```
public int kmpSearch(String s,String p){
    int j=0;
    int i=0;
    while (i<s.length()&&j<p.length()){
        if (j==-1||s.charAt(i)==p.charAt(j)){
            j++;
            i++;
        }
        else
            j = next[j];
    }
    if(j==p.length())
        return i-j;
    else
        return -1;
}
```
