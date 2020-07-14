# 一 自建List模板类的接口

代码来源：《数据结构（c++）》--邓俊辉，并对其做出了一些改正

```c++
//ListNode模板类
#define ListNodePosi(T) ListNode<T>*//列表节点位置

template <typename T>
class ListNode{//以双向链表形式实现//所以后pred和succ两个指针
    public:
    //成员
    T data;//数值
    ListNodePosi(T) pred;//前驱指针
    ListNodePosi(T) succ;//后继指针
    
    //构造函数
    ListNode(){}//针对header和trailer的构造
    ListNode(T e, ListNodePosi(T) p = NULL, ListNodePosi(T) s = NULL):data(e), pred(p), succ(s){}//默认构造器
    
    //操作接口
    ListNodePosi(T) insertAsPred(T const& e);//紧靠当前节点之前插入新节点
    ListNodePosi(T) insertAsSucc(T const& e);//紧靠当前节点之后插入新节点
}

//List模板类
//#include "ListNode.h"//引入列表节点类
template <typename T>
class List{
    
    private:
    int size;//规模
    ListNodePosi(T) header;//头哨兵
    ListNodePosi(T) trailer;//尾哨兵
    
    protected:
    void init();//列表创建的初始化
    int clear();//清除所有节点
    void copyNodes(ListNodePosi(T) p, int n);//复制列表中自位置p起的n项
    void merge(ListNodePosi(T)&, int, List<T>&, ListNodePosi(T), int);//有序列表区间归并
    void mergeSort(ListNodePosi(T)& p, int n);//对从p开始连续的n个节点归并排序
    void selectionSort(ListNodePosi(T) p, int n);//对从p开始连续的n个节点选择排序
    void insertionSort(ListNodePosi(T) p, int n);//对从p开始连续的n个节点插入排序
    
    public:
    //四种构造函数
    List(){init();}//默认构造函数
    List(List<T> const& L);//整体复制列表
    List(List<T const& L, int r, int n);//复制列表L中自第r项起的n项
    List(ListNodePosi(T) p, int n);//复制列表中自位置p起的n项
    //析构函数
    ~List();//释放所有节点（包含头尾哨兵）
    //只读访问接口
    int size() const {return size;}//规模
    bool empty() const {return size <= 0;}//判空
    LIstNodePosi(T) operator[](int r) const;//重载，支持寻秩访问（效率低）
    ListNoedPosi(T) first() const {return header->succ;}//首节点位置
    ListNoedPosi(T) last() const {return trailer->pred;}//末节点位置
    bool valid(ListNodePosi(T) p){//判断位置p是否对外合法
        return p && (trailer != p) && (header != p);//将头尾节点等同于NULL
    }
    void valid(ListNodePosi(T) p, int r, ListNodePosi(T) q){
        while(0 < r) if(p == (q = q->pred)) return true; return false;//判断位置p是否为q的前驱且距离不超过r
    }
    void valid(ListNodePosi(T) p, ListNodePosi(T) q， int r){
        while(0 < r) if(p == (q = q->succ)) return true; return false;//判断位置p是否为q的后继且距离不超过r
    }
    int disordered() const;//判断列表是否已排序
    ListNodePosi(T) find(T const& e) const{
        return find(e, size, trailer);//无序列表查找
    }
    ListNodePosi(T) find(T const& e, int n, ListNodePosi(T) q) const;//无序区间查找
    ListNodePosi(T) search(T const& e) const{
        return search(e, size, trailer);//有序列表查找
    }
    ListNodePosi(T) find(T const& e, int n, ListNodePosi(T) q) const;//有序区间查找
    LisyNodePosi(T) selectMax(ListNodePosi(T) p, int n);//在p及其前n-1个后继中选出最大者
    ListNodePosi(T) selectMax(){
        return selectMax(header->succ, size);//整体最大者
    }
    ListNodePosi(T) insertAsFirst(T const& e);//将e当作首节点插入
    ListNodePosi(T) insertAsLast(T const& e);//将e当作末节点插入
    ListNodePosi(T) insertBefore(ListNodePosi(T) p, T const& e);//将e当作p的前驱插入
    ListNodePosi(T) insertAfter(ListNodePosi(T) p, T const& e);//将e当作p的后继插入
    T remove(ListNodePosi(T) p);//删除合法位置p处的节点，返回被删除节点
    void merge(Lsit<T>& L){
        merge(first(), size, L, L.first(), L.size);//全列表归并
    }
    void sort(ListNodePosi(T) p, int n);//列表区间排序
    void sort(){
        sort(first(), size);//列表整体排序
    }
    int deduplicate();//无序去重
    int uniquify();//有序去重
    //遍历
    void traverse(void (*)(T&));//遍历，依次实施visit操作（函数指针，只读或局部性修改）
    template <typename VST>
    void traverse(VST&);//遍历，依次实施visit操作（函数对象，可全局性修改）
};
```

## 1.1 普通列表

### 1.1.1 头尾节点

$$
header\leftrightarrows first\leftrightarrows ......\leftrightarrows last  \leftrightarrows trailer
$$

头节点（header）和尾节点（trailer）对外并不可见，对外可见的是第一个节点（first node）和最后一个节点（last node）。经封装后从外部不可见的节点称位**哨兵节点（sentinel node）**，从外部等效地将其视之为**NULL**。

### 1.1.2 默认构造方法

```c++
template <typename T>
void List<T>::init(){//列表初始化，在创建列表对象是统一调用
    header = new ListNode<T>;//创建头哨兵节点
    trailer = new ListNode<T>;//创建尾哨兵节点
    header->succ = trailer; header->pred = NULL;
    trailer->pred = header; trailer->succ = NULL;
    size = 0;
}
```

初始化时，通过设置其前驱后继指针构成一个双向链表。该链表对外的有效部分为空，此后引入的新节点都将插入在头尾哨兵之间，

### 1.1.3 由秩到位置的转换

列表和向量一样都是线性结构，也是可以按照逻辑次序为每个节点确定具体的秩，但效率很低。

```c++
template <typename T>
ListNodePosi(T) List<T>::operator[](int r) const{
    ListNodePosi(T) p = first();//从首节点触发
    while(0 < r--) p = p->succ;//顺数的第r个节点
    return p;//目标节点
}
```

列表使用秩的效率低下的原因是列表元素的存储与访问方式不同于向量，数组等。对于向量和数组，通过下标访问的时间仅仅是o(1)，而列表，在各元素被访问概率相等的情况，平均也需要o(n)的时间，效率大大降低。

### 1.1.4 查找

```c++
template <typename T>
ListNodePosi(T) List<T>::find(T const& e, int n, ListNodePosi(T) p) const{
    while(0 < n--)//对于p的n个前驱，从右至左
        if(e == (p = p->pred)->data) return p;//逐个比对，直至命中或范围越界
    return NULL;//p越出左边界就意味着查找的区间内没有元素e，查找失败
}
```

与Vector::find()相仿，时间复杂度也是o(n)。

### 1.1.5 插入

一共四种插入：

```c++
//将e当作首节点插入
template <typaname T>
ListNodePosi(T) List<T>::insertAsFirst(T const& e){
    size++;
    return header->insertAsSucc(e);
}

//将e当作末节点插入
template <typename T>
ListNodePosi(T) List<T>::insertAsLast(T const& e){
    size++;
    return trailer->insertAsPred(e);
}

//将e当作p的前驱插入
template <typename T>
ListNodePosi(T) List<T>:;insertBefore(ListNodePosi(T) p, T const& e){
    size++;
    return p->insertAsPred(e);
}

//将e当作p的后继插入
template <typename T>
ListNodePosi(T) List<T>::insertAfter(ListNodePosi(T) p, T const& e){
    size++;
    return p->insertAsSucc(e);
}
```

#### 1.1.5.1 前插入

```c++
template <typename T>
ListNodePosi(T) ListNode<T>::insertAsPred(T const& e){
    ListNodePosi(T) x = new ListNode<T>(e,pred,this);//创建新节点
    pred->succ = x; pred = x;//设置正向链接
    return x;//返回新节点的位置
}
```

先创建一个新节点x，前插入就是要实现x成为当前节点的前驱以及当前节点旧前驱的后继。

#### 1.1.5.2 后插入

```c++
template <typename T>
ListNodePosi(T) ListNode<T>::insertAsSucc(T const& e){
    ListNodePosi(T) x = new ListNode<T>(e,this,succ);//创建新节点
    succ->pred = x; succ = x;//设置逆向链接
    return x；//返回新节点位置
}
```

与前插入的理解相同

### 1.1.6 基于复制的构造

无论是整体复制还是局部复制，根据向量的复制方法，复制过程都大同小异，所以都统一为一种底层copyNodes()。

```c++
template <typename T>
void List<T>:;copyNodes(ListNodePosi(T) p, int n){
    init();
    while(n--){ insertAsLast(p->data); p = p->succ;}//将自p的n项一次作为末节点插入
}
```

再使用copyNodes()实现三种复制：

```c++
//复制列表中自位置p起的n项
template <typename T>
T List<T>::List(ListNodePosi(T) p, int n)
{copyNodes(p,n);}

//整体复制列表L
template <typename T>
T List<T>::List(List<T> const& L)
{copyNodes(L.first(),L.szie);}

//复制列表L中自第r项起的n项
template <typename T>
T List<T>::List(List<T> const& L, int r, int n)
{copyNodes(L[r],n);}
```

### 1.1.7 删除

删除就是要实现最终在新列表中，原p节点的后继成为原p节点前驱的后继，原p节点的后继是原p节点前驱的后继。基于此：

```c++
template <typename T>
T List<T>::remove(ListNodePosi(T) p){
    T e = p->data;//备份待删除元素
    p->pred->succ = p->succ;
    p->succ->pred = p->pred;
    delete p; size++;
    return e;//返回被删除节点的值
}
```

被孤立出的节点p必须释放掉

### 1.1.8 析构

析构过程分为两步，先清除释放所有对外可见的节点，再释放头尾节点

```c++
template <typename T>
T List<T>:;~List()
{clear(); delete header; delete trailer;}

//clear()的实现
template <typename T>
int List<T>::clear(){
    int oldSize = size;
    while(0 < size) remove(header->succ);//反复删除首节点，直至列表变空
    return oldSize;
}
```

### 1.1.9  唯一化

List::deduplicate()与Vector::deduplicate()具有相同的不变性和单调性，其思想也是一样的：

```c++
template <typename T>
int List<T>:;deduplicate(){
    if(szie < 2) return 0;//平凡列表绝不会有重复的节点
    int oldSize = size;
    ListNodePosi(T) p = header; int r = 0;//从首节点开始
    while (trailer != (p = (p->succ))){//依次直到末节点
        ListNodePosi(T) q = find(p->data,r,p);//在p的r个前驱中查找雷同者
        q ? remove(q) : r++;//若p存在，则删除
    }
    return oldSize - size;//返回被删除元素总数
}
```

每次迭代，find()操作所需时间线性正比于查找区间的宽度，所以总体执行时间就是o(1+2+3+...+n) = o(n^2)。

### 1.1.10 遍历

List::traverse()的设计思路、实现方式以及时间复杂度都与Vector::traverse()完全一样

```c++
//利用函数指针实现遍历，只读或者局部性修改
template <typename T>
void List<T>:;traverse(void (*visit)(T&)){
    ListNodePosi(T) p = header;
    while((p = p->succ) != trailer)
        visit(p->data);
}

//利用函数对象实现遍历，可全局性修改
template <typename T>
template <typename VST>
void List<T>::traverse(VST& visit){
    ListNodePosi(T) p = header;
    while((p = p->succ) != trailer)
        visit(p->data);
}
```

## 1.2 有序列表

### 1.2.1 唯一化

采用的是成批删除重复元素，这样的效率更高。

```c++
template <typename T>
int List<T>::uniquify(){
    if(size < 2) return 0;
    int oldSize = size;
    ListNodePosi(T) q = first();
    while(trailer != q->succ){
        ListNodePosi(T) p = q; q = p->succ;//p和q依次指向紧邻的每对节点
        if(p->data == q->data){remove(q); q = p;}
    }
    return olsSize = size;//返回被删除元素总数
}
```

### 1.2.2 查找

```c++
//在p的n个真前驱中找到不大于e的最后者
template <typename T>
ListNodePosi(T) List<T>::search(T const& e, int n, ListNodePosi(T) p) const{
    while(0 <= n--)
        if(((p = p->pred)->data) <= e) break;
    return p;
}
```

会发现在有序列表的查找算法中我们没有采用有序向量那种的二分查找，这是因为：尽管有序列表中的节点已经按照次序单调排列，但因为在链式动态存储策略中节点的物理存储地址与其逻辑次序无关，所以就没法**轻松地**采用”减而知之“的策略。

### 1.2.3 排序

排序算法一共有三种：插入排序、选择排序、归并排序，所以先统一入口：

```c++
template <typename T>
void List<T>::sort(ListNodePosi(T) p, int n){
    switch (rand() % 3){
        case 1:insertionSort(p,n);break;//插入排序
        case 2:selectionSort(p,n);break;//选择排序
        case 3:mergeSort(p,n);break;//归并排序
    }
}
```

#### 1.2.3.1 插入排序

插入排序是一种非常广泛的排序方法，适用于任何序列结构。其思路简单得来说：将序列分为有序无序两类，有序的保持不动，通过迭代将无序中的元素插入至有序中的适当位置。

这个思路也体现出了插入排序算法的不变性：在任何时刻，以当前节点e为界的前子序列总是有序。

```c++
template <typename T>
void List<T>::insertionSort(ListNodePosi(T) p, int n){
    for(int i = 0; i < n; i++){
        inserAfter(search(p->data, i, p), p->data);//查找适当的位置并插入//在p的i个前驱中找到不大于p->data的最后者，并在其后插入p->data
        p = p->succ; remove(p->pred);//转向下一节点
    }
}
```

利用此种思路，设计出适合任意序列结构的插入排序：

```c++
template <typename T>
void insertionSort(T& A, int n) {
	for (int i = 1; i < n; i++) {//从第一个元素开始，第一个元素默认已被排序
		int temp = A[i];//从无序组取出第一个元素
		int j = i - 1;//有序数组的最后一个元素
		while (j >= 0 && A[j] > temp) {//j>=0是对其进行边界限制，A[j] > temp未插入判断条件
			A[j + 1] = A[j];
			j--;//若不是合适的位置，有序组元素向后移动
		}
		A[j + 1] = temp;//找到合适位置，将元素插入
	}
}
```

#### 1.2.3.2 选择排序

选择排序也同插入排序一样，分为有序和无序两组，区别在于，每次从无序组中找到最大值插入在有序组的合适位置上。

所以选择排序分为两步：先找最大值，再排序。

```c++
//寻找最大值
template <typename T>
ListNodePosi(T) List<T>::selectMax(ListNodePosi(T) p, int n){
    ListNodePosi(T) max = p;//最大者暂定为首节点p
    for(ListNodePosi(T) cur = p; i < n; n--)//从首节点p出发，将后续节点逐一与max作比较
        if(!lt((cur = cur->succ)->data, max->data))
            max = cur;//更新最大元素位置记录
    return max;
}
//实现选择排序
template <typename T> //列表的选择排序算法：对起始于位置p的n个元素排序
void List<T>::selectionSort(ListNodePosi(T) p, int n) {
	ListNodePosi(T) head = p->pred;
	ListNodePosi(T) tail = p;
	for (int i = 0; i < n; i++) tail = tail->succ; //待排序区间为(head, tail)
	while (1 < n) { //在至少还剩两个节点之前，在待排序区间内
		ListNodePosi(T) max = selectMax(head->succ, n); //找出最大者（歧义时后者优先）
		insertBefore(tail, remove(max)); //将其移至无序区间末尾（作为有序区间新的首元素）
		tail = tail->pred; n--;
	}
}
```

由于寻找最大值时，需要遍历整个无序列表，整个过程耗时o(n^2)。也可以发现选择排序算法也是基于比较式的算法。

#### 1.2.3.3 归并算法

有序列表的二路归并与有序向量的二路归并的效率同样高效。

```c++
//有序列表的归并：当前列表中自p起的n个元素，与列表L中自q起的m个元素归并
template <typename T> 
void List<T>::merge(ListNodePosi(T)& p, int n, List<T>& L, ListNodePosi(T) q, int m) {
	ListNodePosi(T) pp = p->pred; //借助前驱（可能是header），以便返回前 ...
	while (0 < m) //在q尚未移出区间之前
		if ((0 < n) && (p->data <= q->data)) //若p仍在区间内且v(p) <= v(q)，则
		{
			if (q == (p = p->succ)) break; n--;
		} //将p替换为其直接后继（等效于将p归入合并的列表）
		else //若p已超出右界或v(q) < v(p)，则
		{
			insertBefore(p, L.remove((q = q->succ)->pred)); m--;
		} //将q转移至p之前
	p = pp->succ; //确定归并后区间的（新）起点
}
//列表的归并排序算法：对起始于位置p的n个元素排序
template <typename T>
void List<T>::mergeSort(ListNodePosi(T)& p, int n) { //valid(p) && rank(p) + n <= size
	if (n < 2) return; //若待排序范围已足够小，则直接返回；否则...
	int m = n >> 1; //以中点为界
	ListNodePosi(T) q = p; for (int i = 0; i < m; i++) q = q->succ; //均分列表
	mergeSort(p, m); mergeSort(q, n - m); //对前、后子列表分别排序
	merge(p, m, *this, q, n - m); //归并
} //注意：排序后，p依然指向归并后区间的（新）起点
```

# 二、STL库的List模板类

## 2.1 list的构造函数

```c++
一共有四种
list<T>l;//创建一个空的list对象l
list<T>l1(l2);//复制另一个同类型元素的list
list<T>l(n);//创建有n个T类型元素的list
list<T>l(n,elem);///创建有n个T类型元素的list,每个元素都是elem
list<T>l(begin,end);//由迭代器创建list,迭代区间为[begin,end)
```

## 2.2 元素相关的函数

**Int size() cons**t:返回容器元素个数

**bool empty() const**:判断容器是否为空，若为空则返回true

## 2.3 增加删除函数

**void push_back(const T& x)**:list元素尾部增加一个元素x

**void push_front(const T& x)**:list元素首元素钱添加一个元素X

**void pop_back()**:删除容器尾元素，当且仅当容器不为空

**void pop_front()**:删除容器首元素，当且仅当容器不为空

**void remove(const T& x)**:删除容器中所有元素值等于x的元素

**void clear()**:删除容器中的所有元素

**iterator insert(iterator it, const T& x )**:在迭代器指针it前插入元素x,返回x迭代器指针

**void insert(iterator it,size_type n,const T& x)**:迭代器指针it前插入n个相同元素x

**void insert(iterator it,const_iterator first,const_iteratorlast)**:把[first,last)间的元素插入迭代器指针it前

**iterator erase(iterator it)**:删除迭代器指针it对应的元素

**iterator erase(iterator first,iterator last)**:删除迭代器指针[first,last)间的元素

## 2.4 遍历函数

**iterator begin()**:返回首元素的迭代器指针

**iterator end()**:返回尾元素之后位置的迭代器指针

**reverse_iterator rbegin(**):返回尾元素的逆向迭代器指针，用于逆向遍历容器

**reverse_iterator rend()**:返回首元素前一个位置的迭代器指针

**reference front()**:返回首元素的引用

**reference back()**:返回尾元素的引用 

## 2.5 操作函数

**void sort()**:容器内所有元素排序，默认是升序

**void swap(list& str)**:两list容器交换功能

**void unique()**:容器内相邻元素若有重复的，则仅保留一个

**void splice(iterator it,list& li)**:队列合并函数，队列li所有函数插入迭代指针it前，x变成空队列

**void splice(iterator it,list& li,iterator first)**:队列li中移走[first,end)间元素插入迭代指针it前

**void splice(iterator it,list& li，iterator first,iterator last)**:x中移走[first,last)间元素插入迭代器指针it前

**void reverse()**:反转容器中元素顺序
