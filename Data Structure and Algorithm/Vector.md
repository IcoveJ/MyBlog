# 一 自建Vector模板类的接口

代码来源：《数据结构（c++）》--邓俊辉，并对其做出了一些改正

```c++
#define Default_Capacity 3//宏定义初始容量，实际情况可以改变

template <typename T>
class Vector{
    private:
    int size;//规模
    int capacity;//容量
    int elem;//数据
    
    protected:
    void copyFrom(T* const A, int lo, int hi);//复制数组区间A[li,hi)
    void expand();//空间不足时扩容
    void shrink();//装填因子过小时压缩
    bool buuble(int lo, int hi);//扫描交换算法
    void bubbleSort(int lo, int hi);//起泡排序算法
    void merge(int lo, int mi, int hi);//归并排序
    void mergeSort(int lo, int hi);//归并排序算法
    int partition(int lo, int hi);//轴点构造算法
    void quickSort(int lo, int hi);//快速排序算法
    void heapSort(int lo, int hi);//堆排序
    
    public:
    Vector(int c = Default_Capacity){elem = new T[capacity = c]; size = 0;}//构造函数
    Vector(T* A, int lo, int hi){copyFrom(A, lo, hi);}//数组区间复制
    Vector(T* A, int n){copyFrom(A, 0, n);}//数组整体复制
    Vector(Vector<T> const& V, int lo, int hi){copyFrom(V.elem, lo, hi);}//向量区间复制
    Vector(Vector<T> const& V){cpyFrom(V.elem, 0, V.size);}//区间整体复制
    ~Vector(){delete []elem};//析构函数，释放空间
    //只读访问接口
    int size() const {return size;}//得到向量的规模
    bool empty() const {return size <= 0;}//判空
    int disordered() const;//判断向量是否已经排序
    int find(T const& e) const {return find(e, 0, (int)size);}//无序向量整体查找
    int find(T const& e, int lo, int hi) const;//无序向量区间查找
    int search(T const& e) const {return (size <= 0)?-1:search(e, (int)0, (int)size);}//有序向量整体查找
    int search(T const& e, int lo, int hi) const;//有序向量区间查找
    //可写访问接口
    T& operator[](int r) const;//重载下标操作符，可以类似于数组形式引用各元素
    Vector<T> & operator=(Vector<T> const&);//重载赋值操作符，以便直接克隆向量
    T remove(int r);//删除秩为r的元素
    int remove(int lo, int hi);//删除秩在区间[lo, hi)之间的元素
    int insert(int r, T const& e);//插入元素
    int insert(T const& e){return insert(size, e);}//默认作为末元素插入
    void sort(int lo, int hi);//对[lo,hi)排序
    void sort(){sort(0, size);}//整体排序
    void unsort(int lo, int hi);//对[lo,hi)置乱
    void unsort(){unsort(0, size);}//整体置乱
    int deduplicate();//无序去重
    int uniquify();//有序去重
    void traverse(void (*)(T&));//遍历//只读或局部性修改
    template <typename VST> void traverse(VST&);//遍历//使用函数对象，可全局性修改
};
```

通过模板参数T指定向量元素的类型，不仅可以是int，float，还可以是Vector<Vector<char>>，定义存放字符的二维向量。

## 1.1 构造与析构

在这个模板类中，有两种构造方法：默认构造方法和基于复制的构造方法

### 1.1.1 默认构造方法

```c++
Vector(int c = Default_Capacity){elem = new T[capacity = c]; size = 0;}
```

根据创建者指定的初始容量向系统申请空间，来创建内部私有数组elem[]；若容量未明确指定，则使用默认值Default_Capacity。由于初生的向量不含任何元素，size的值就初始化为0。

整个构造过程就是一个顺序过程，没有任何迭代等过程，如果不考虑分配内存的时间，整个构造过程只需常数时间。

### 1.1.2 基于复制的构造方法

这种构造方法就相当于是clone，以已有向量或数组（的局部或整体）为蓝本克隆出一个新向量。

```c++
template <typename T>
void Vector<T>::copyFrom(T* const A, int lo, int hi){//基于数组复制的构造
    elem = new T[capacity = 2*(hi - lo)];//分配空间
    size = 0;//规模清零
    while(lo < hi) elem[size++] = A[lo++];//复制至elem[0,hi-lo)
}
```

先由原向量的前后边界lo和hi换算出新向量的初始规模，我这里是以二倍的容量申请的空间。

需注意，在进行向量间的赋值时，由于向量内部含有动态分配的空间，编译器所默认的运算符“=”不足以支持向量间的相互赋值。所以需要重载“=”。

```c++
template <typename T>
Vector<T>& Vector<T>::operator=(Vector<T> const& V){
    int n = V.size();//待克隆的向量长度
    if(capacity < n){
        delete []elem; elem = new T[capacity = 2*n];//如果有必要，数据区空间扩容
    }
    copFrom(V.elem, 0, n);//整体复制
    return *this;//返回当前对象的引用，以便直接链式赋值
}
```

### 1.1.3 析构

与其它类一样，Vector的对象不再被使用时就需要及时清理，释放内存空间，内部的变量也会随着对象一起被回收。整个析构过程也仅仅需要o(1)的时间。

## 1.2 动态空间的管理

无论是在构造函数中还是在所有的接口中，我们都使用了动态空间的使用。以避免出现capacity被固定后，若再加入新元素会出现的上溢(overflow)的情况。但是若预留的空间又过多，就会导致空间利用率极低。用向量的实际规模与内部数组容量的比值(即szie/capacity，称为装填因子)来衡量空间利用率的高低。使用动态空间管理测略，就是为了实现向量的装填因子不超过1，也不太接近0.

### 1.2.1 可扩充向量

因为数组存放元素的地址一定是连续的，操作系统是不能保证原数据区的尾部总是预留有足够的空间，那么就不能直接在原数组的基础上追加空间。需要一种新的算法来实现。

简单得来说，就是另行申请一个容量更大的空间，并将原数组搬迁至新的空间。当然还是要释放原数组的空间。

### 1.2.2 扩容expand()

根据上面的思路，实现向量内部的数组动态扩容算法expand()：

```c++
template <typename T>
void Vector<T>::expand(){
    if(szie < capacity) return;//尚未满时，就不扩容
    if(capacity < Default_Capacity) capacity = Default_Capacity;//不低于最小容量
    T* oldElem = elem; elem = new T[capacity <<= 1];//容量每次扩大一倍
    for(int i = 0; i < size; i++)
        elem[i] = oldElem[i];
    delete [] oldElem;
}
```

在往向量中插入新元素之前都要先调用expand()检查内部数组的容量是否已满。

要注意的是，这种方法一定要封装。因为新数据区与原数据区没有一点联系，若直接引用新数组，会导致共同指向原数据区的其他指针失效，变成野指针。

### 1.2.3 分摊分析

#### 1.2.3.1 时间代价

每一次的扩容，即使假定每一元素的复制仅仅需要常数时间，从n -> 2n，也需要花费o(n)的时间。感觉这种扩充策略效率很低，但注意，每花费o(n)的时间，数组容量加倍，就意味着至少要再经过n次插入操作，才会重新再次扩容。换句话说，随着向量规模的扩大，插入操作时需要进行扩容的概率会迅速降低。因此，从平均角度而言，用于扩容的时间成本不高。

#### 1.2.3.2 分摊复杂度

由于插入、删除操作的执行不确定性，很难采用通常的方式来度量和分析其复杂度，因此采用分摊复杂度：**计算在前后连续执行的多次操作中，每次操作所需的平均计算成本**。

这么看起来，用分摊复杂度去度量算法的执行时间与平均时间一样，但事实上而这是有本质区别的：平均时间是指在对各种情况出现概率的分布做过假设后，对各种情况所需执行时间的加权平均；而分摊时间是指在使用某一算法对同一数据结构连续的实施了足够多次的操作后，总体所需的计算时间分摊至期间所有操作的平均值。

由此可看出，分摊复杂度更为准确。

### 1.2.4 缩容shrink()

当发生了删除操作的多于插入操作时，装填因子极可能接近于0，当装填因子低于某一阈值时，称数组发生了下溢。发生下溢时，就有必要适当缩减内部数组容量。

```c++
template <typename T>
void Vector<T>::shrink(){
    if(capacity < Default_Capacity << 1) return;//不致收缩到Default_Capacity以下
    if(size << 2 > capacity) return;//以25%为界
    T* oldElem = elem; elem = new T[capacity >>= 1];//容量减半
    for(int i = 0; i < size; i++) elem[i] = oldElem[i];//复制原向量内容
    delete [] oldElem;//释放原空间
}
```

这里是以20%为界，也可以选用更低的阈值以避免出现频繁交替扩容和缩容的情况。当取值为0时，相当于禁止缩容。

## 1.3 向量

### 1.3.1 直接引用元素

通过重载操作符”[]“，实现沿用数组形式的通过下标对向量元素的访问。

```c++
template <typename T>
T& Vector<T>::operator[](int r) const{
    return elem[r];
}
```

### 1.3.2 置乱器

#### 1.3.2.1 置乱算法

通过使用重载的“[]”操作可以简明清晰的实现向量的置乱算法：

```c++
template <typename T>
void permute(Vector<T>& V){//随机置乱向量，使各元素在每一位置出现的概率均等
    for(int i = V.size(); i > 0; i--)
        swap(V[i-1], V[rand() % i]);//V[i-1]与V[0,i-1]中某一随机元素交换//应用了重载操作符“[]”地赋值
}
```

通过从待置乱区间的末元素向前逆序扫描，再通过调用rand()函数在[0, i)之间等概率地随机选取一个元素，并将两者交换。

#### 1.3.2.2 基于permute()的unsort()

```c++
template <typename T>
void Vector<T>::unsort(int lo, int hi){//等概率随机置乱向量区间V[lo,hi)
    T* V = elem + lo;//将子向量elem[lo,hi)视作另一向量V[0,hi-lo)
    for(int i = hi - lo; i > 0; i--)
        swap(V[i-1], V[rand() % i]);
}
```

unsort()与permute()有一点不同，unsort()是通过下标访问的是内部的数组元素。

### 1.3.3 无序向量的顺序查找

在无序查找中，最坏的情况是我们所查找的元素在向量的最后一位，或者我们直到查找到最后一位依然没有找到该元素，所以，不如直接从最后一位向前查找，逐一取出各个元素与目标元素做对比，直至查找成功或查找失败。

```c++
template <typename T>
int Vector<T>::find(T const& e, int lo, int hi) const{
    while((A[hi] != e) && (lo < hi--));//从后向前顺序查找，直到找到最前的一个目标值或者未找到
        return hi;//返回最后即下标最大的值或者未找到返回-1.
}
```

注意while循环的控制逻辑：原文(lo < hi--) && (A[hi] != e)是先判断是否已抵达通配符处，再去判断当前元素是否与目标元素相等。但这么做忽略了hi处的元素；所以将其改成先判断元素是否相等，再判断通配符。

其时间复杂度依然是o(n)。

### 1.3.4 插入

```c++
template <typename T>
int Vector<T>::insert(int r, T const& e){
    expand();//先判断是否有必要扩容
    for(int i = size; i > r; i--)//自后向前
        elem[i] = elem[i - 1];//后继元素顺次后移一个单位
    elem[i] = e;//置入新元素
    size++;//更新容量
    return r;
}
```

总体时间复杂度为o(size - r + 1)，由此可发现插入元素越靠后，所需时间就越短。

### 1.3.5 删除

#### 1.3.5.1 区间删除

```c++
template <typename T>
int Vector<T>::remove(int lo, int hi){
    if(lo == hi) return 0;
    while(hi < size) elem[lo++] = elem[hi++];
    size = lo;//更新规模，直接丢弃尾部[lo,size = hi)的区间
    shirnk();//若有必要就缩容
    return hi - lo;//返回的是删除元素的数目
}
```

[lo,hi)为原内部数组的合法区间，则将原秩不小于hi的所有后继元素自前向后的逐个前移hi - lo个单元。后继元素的移动次序不能颠倒，

前移完毕后，向量规模更新为size - hi + lo。

#### 1.3.5.2 单元素删除

```c++
template <typename T>
T Vector<T>::remove(int r){
    T e = elem[r];//备份被删除的元素
    remove(r, r+1);//调用区间删除算法，删除[r,r+1)的元素
    return e;//返回的是被删除的元素
}
```

时间复杂度，最好的情况是o(1)，最差的情况是o(n^2)（每次循环的耗时正比于删除区间的后缀长度，再乘以区间宽度。两者都是o(n)）。



无论是区间删除，还是单元素删除，我们所采用的区间都是[0，size)，因此输入的参数也必须满足0 ≤ lo ≤ hi ≤ size。

为避免人为故意的输入参数的超出范围，可以采用“try...catch...throw”来捕获并处理意外。

### 1.3.6 唯一化

就是从向量中剔除重复的元素。有什么实际意义呢？

比如：网页搜索引擎，对输入的词组识别其中的关键字，进行不同组合的查找，待多个计算节点返回各自搜索到的局部结果后，再剔除其中重复的项目，以合并一份完整的报表再提交给搜索用户。

```c++
//这里是无序向量的唯一化
template <typename T>
int Vector<T>::deduplicate(){
    int OldSize = size;
    inr i = 1;
    while(i < size)//自前向后逐一考察各元素
        (0 >find(elem[i], 0, i))?//在其前缀中寻找与之雷同
        i++:remove(i);//继续考察其后继或者删除雷同元素
    return OldSize - szie;//返回被删除元素的总数
}
```

### 1.3.7 遍历

通过traverse()实现将向量作为整体的操作：

```c++
//利用函数指针，只读或局部修改
template <typename T>
void Vector<T>::traverse(void (*visit)(T&)){
    for(int i = 0; i < size; i++) visit(elem[i]);
}
//利用函数对象，可全局性修改
template <typename T>
template <typename VST>//操作器
void Vector<T>::traverse(VST& visit){
    for(int i = 0; i < size; i++) visit(elem[i]);
}
```

前一种借助函数指针*visit()指定一函数，这个函数只有一个参数，其类型为对向量元素的引用

后一种借助的是函数对象，可以实现对元素的关联修改，即各元素的修改可以单独进行，也可以根据某个（些）元素的数值相应的修改另一元素。

这里是一个接口的具体实现，我用数组的形式，对其第二种进行具体的实例分析：

```c++
template <typename T, typename VST>
void traverse(VST& visit, T& A) {
	int size = sizeof(A) / sizeof(A[0]);
	for (int i = 0; i < size; i++)
		visit(A[i]);
}

//输出操作器的实现
void visit(int n) {
	cout << n << " ";
}

int main() {
	int A[] = { 0,1,2,3,4,5,6,7,8,9,1,2,1,1,1 };
	traverse(visit, A);//在traverse()中实现了遍历A[]中的元素，再用visit操作器来进行输出
	cout << endl;
	system("pause");
	return 0;
}
```

## 1.4 有序向量

有序向量是无序向量的特例，因为元素间的有序性，有序向量的查找、唯一化等问题都可以更快地完成。

### 1.4.1 判断向量是否已经有序

```c++
template <typename T>
int Vector<T>::disordered() const{
    int n = 0;
    for(int i = 0; i < size; i++)
        if(elem[i-1] > elem[i]) n++;
    return n;
}
```

### 1.4.2 唯一化

#### 1.4.2.1 低效版本

```c++
template <typename T>
int Vector<T>::uniquify(){
    int OldSize = size; int i = 0;
    while(i < size-1){
        if(elem[i] == elem[i + 1]) remove(i + 1);
        else i++;
    }
    return OldSize - size;//返回被删元素总数
}
```

这种算法是逐个操作，再剔除重复元素：因为在有序向量中，重复的元素必定前后紧邻。

该算法的时间主要消耗在while循环上。设想最坏的情况，一个向量里的n个元素全部相等，那么，每次while循环都要执行一次remove()操作。其复杂度线性正比于被删除元素的后继元素。

因此时间复杂度为：(n-2) + (n-3) + (n-4) + ... + 2 + 1 = o(n^2)。

看见时间复杂度很大。

#### 1.4.2.2 高效版本

在上面的分析中我们知道时间的消耗主要是因为它是逐个操作，就导致效率很低。因此就可以选择成批的删除前后紧邻的雷同元素，再将它们的后继元素(若有)一次性前移一次。就如，若V[lo, hi)为一组紧邻的重复元素，则将后继元素V[hi, size)整体向前移hi - lo - 1个单元。

```c++
template <typename T>
int Vector<T>::uniquify(){
    int i = 0, j = 0;
    while(++j < size)//逐一扫描，直至末元素
        if(elem[i] != elem[j])//跳过雷同者
            elem[++i] = elem[j];//发现不同元素时，向前移至紧邻于前者右侧
    size = ++i; shrink();//直接截去尾部多余的元素
    return j - i;
}

//以数组做的一个实例，此处没有使用shrink()
template <typename T>
int uniquify(T& A) {
	int size = sizeof(A) / sizeof(A[0]);
	int i = 0, j = 0;
	while (++j < size)
		if (A[i] != A[j])
			A[++i] = A[j];
	size = ++i;
	for (int i = 0; i < size; i++)
		cout << A[i] << " ";
	cout << endl;
	return j - i;
}
int main() {
	int A[] = { 0,0,1,1,2,2,3,3,4,4,5,5,6,6 };
	cout << uniquify(A) << endl;
	system("pause");
	return 0;
}
```

最后的输出结果是新数组的元素是[0，1，2，3，4，5，6 ]删除元素总数为7。

一进入循环，i = 0指向数组第一位，j = 1指向数组第二位，两者相等，j后移，j = 2指向第三位为1，两者不等，i后移一位，并将j = 2位赋值给i = 1位，此时的数组内容为{ 0,1,1,1,2,2,3,3,4,4,5,5,6,6 }......直至最后一位，再更新size的值。

由此可见，在while循环的每一迭代中，仅仅需要比较元素一次。由于j ≤ n，所以其时间复杂度仅仅为o(n)。相比于低效版本，计算速度提高了一个线性因子。

### 1.4.3 查找

查找的方法有很多：Fibonacci查找、三种二分查找

#### 1.4.3.1 二分查找（A版）

依然是“减而知之”的策略。

以任意元素为界，将[lo,hi)分为三个部分，根据有序向量的元素有序性，V[lo,hi) < V[mi] < V[mi,hi)。先将目标元素e与V[mi]作比较，小于则在V[lo,hi)内深入查找；大于则在V[mi,hi)内深入查找。

```c++
template <typename T>
static int binsearch(T* A, T const& e, int lo, int hi){
    while(lo < hi){
        int mi = (lo + hi) >> 1;
        if(e < A[mi]) hi = mi;
        else if(A[mi] < e) lo = hi + 1;
        else return mi;
    }
    return -1;
}
```

在此二分查找的迭代过程中，包括了元素的大小比较、秩的运算及赋值。其中占主要消耗时间的是元素大小的比较。

对所有的查找算法而言，都应该关注其中的关键码的比较次数，即查找长度。一般要从最好、最坏、平均(ASL)三个角度评估。

比如，在这个算法中，取数组{2，4，5，7，8，9，12}中找到8，取mi = (0+7)/2 = 3，这里就会比较2次；进入[4, 7)的分支，取mi = (4+7)/2 = 5，比较1次进入前半段[4,5)，再经两次比较在mi = 4处成功命中。一共就比较了5次。

对8这个元素，一共要比较5次；同样的道理，可以得到其他元素的比较次数，分别为：4，3，5，2，4，6；元素7只用比较2次，元素12要比较6次；就能计算出成功情况下的平均长度(4+3+5+2+4+6+5)/7 = 4.14。

失败查找长度也是如此，根据上述算法，只有在有效区间宽度缩减至零时才会终止，那么每个元素的失败查找都会比成功查找多一次，即(29+7)/7 = 4.50

那这个算法有什么不足呢？

主要就是在为确定左右分支，需要做的元素比较次数分别是1次和2次，这就造成了不同情况所对应查找的不均衡性。

#### 1.4.3.2 二分查找（B版）

既然我们已经知道了A版的比较次数分别是1和2，那么能不能就指定只做1次元素比较呢？

如果这样做，那么每次迭代（或递归）的分支就只有两个而不是三个。

或许你会问，那相等的情况怎么确定呢？在最后返回的时候再进行最后的判断

```c++
template <typename T>
static int binsearch(T* A, T const& e, int lo, int hi){
    while (1 < hi - lo){
		int mi = (lo + hi) >> 1;
		(e < A[mi]) ? hi = mi : lo = mi;
	}//出口时查找区间[lo,hi)只有一个元素A[lo].
	return (e == A[lo]) ? lo : -1;
}
```

在这个版本中，只有向量有效区间宽度缩短至1时才会终止，就不能如A版中命中就能及时退出。所以，最好的情况的效率有所倒退，但最差的情况的效率就有所提高。

#### 1.4.3.3 二分查找（C版）

在上述两种算法都回避了一个问题，就是当这个有序向量中有多个雷同的我们所查找的元素，那怎么确定最后返回哪个秩呢？

所以就有改进的C版本。我们选择返回秩最大者，这样做的好处是方便我们实现顺序的插入。

```C++
template <typename T>
static int binsearch(T* A, T const& e, int lo, int hi){
    while(lo < hi){
        int mi = (lo + hi) >> 1;
        (e < A[mi]) ? hi = mi : lo = mi + 1;
    }
    return --lo;
}
```

在B版上的三点改进：

①只有当向量有效区间的宽度缩短至0时迭代及算法才会终止

②每次转入后端分支时，子向量的左边界取mi + 1

③最后返回的是lo-1

#### 1.4.3.4 Fibonacci查找

这里就直介绍这种查找的思想，就不具体写出代码了。

因为在二分查找的A版本中，它的左右分支的比较是1和2，在B版本在我们是以强制性定为左右分支只比较1次为思路进行算法的设计，而在Fibonacci算法中，我们采用的思想是通过调整前后区域的宽度，适当的加长（缩短）前（后）子向量，使得两个分支的转入成本尽可能接近。

那怎么调整呢？黄金分割，使用Fibonacci数来确定切分点mi。

采用这种算法，成功查找长度与失败查找长度均有所提高，但并不大。

## 1.5  排序器

### 1.5.1 统一所有排序方法的入口

```c++
template <typename T>
void Vector<T>::sort(int lo, int hi){
    switch(rad() % 4){
        case 1: bubbleSort(lo,hi); break;//起泡排序
        case 2: mergeSort(lo,hi); break;//归并排序
        case 3: heapSort(lo,hi); break;//堆排序
        default: quickSort(lo,hi); break;//快速排序
    }
}
```

### 1.5.2 起泡排序

```c++
//起泡排序
template <typename T>
void Vector<T>::bubble(int lo, int hi){
    while(!bubble(lo,hi--));//逐趟做扫描交换，直至全序
}
//扫描交换
template <typename T>
bool Vector<T>::bubble(int lo, int hi){
    bool sorted = true;
    while(++lo < hi)
        if(elem[lo - 1] > elem[lo]){
            sorted = false;
            swap(elem[lo - 1],elem[lo]);
        }
    return sorted;
}
//元素的交换
template <typename T>
void Swap(int lo, int hi){//采用的位运算符来进行交换，降低内存消耗并提高运行速率
    lo ^= hi;
    hi ^= lo;
    lo ^= hi;
}
```

### 1.5.3 归并排序

```c++
//分治策略
template <typename T>
void vector<T>::mergeSort(int lo, int hi){
    if(hi - lo < 2) return;
    int mi = (lo + mi) >> 1;
    mergeSort(lo, mi);//对前半区间排序
    mergeSort(mi, hi);//对后半区间排序
    merge(lo, mi, hi);//两个区间的归并
}
//归并的实现
template <typename T>
void Vector<T>::merge(int lo, int mi, int hi){
    T* A = elem + lo;//合并后的向量A[0,hi-lo) = elem[lo,hi)
    int lb = mi - lo;
    T* B = new T[lb];//前子向量B[0,lb) = A[lo,mi)
    for(int i = 0; i < lb; B[i] = A[i++]);//复制前子向量
    int lc = hi - mi;
    T* C = elem + mi;//后子向量C[0,lc) = A[mi,hi)
    int i, j, k; i = j = k = 0;
    while((j < lb) && (k < lc)){//反复比较B和C的首元素
        while((j < lb) && B[j] <= C[k]) A[i++] = B[j++];//将小者续至A末尾
        while((k < lc) && C[k] <= B[j]) B[i++] = C[k++];//将小者续至A末尾
    }//出口时，B或C为空，即j = lb或k = lc
    while(j < lb) A[i++] = B[j++];
    delete []B;
}
```

通过递推关系式，可以得到其时间复杂度为o(nlogn)。

# 二 STL的vector类

官方的源码就不贴出来了，这里就只讲讲它的一些成员函数。

## 2.1 构造函数

vector类定义了很多种构造函数，用于定义和初始化vector对象。

```c++
vector<T> v;//使用元素的默认构造函数创建一个空序列，即size = 0；
vector<T> v(n, value);//创建一个有n个value值的序列，即size = n；
vector<T> v(n);//创建一个含有n个元素的序列，这n个元素是通过类型T的默认构造函数所返回的结果来初始化的，即size = n；
vector<T> v(list.begin(),list.end());//将数据从链表、双端队列、字符串或数组中拷贝至向量；
```

## 2.2 无返回值的成员函数

```c++
//清空v中的函数
v.clear();

//用于对v的判空，空则返回ture，否则返回false
v.empty();

//删除向量v的最后一个元素
v.pop_back();

//在v最后插入一个值为n的元素
v.push_banck(n);

//删除v中的第n个到第m个元素，这个区间包括了第n个元素，但不包括dim个元素
v.erase(v.begin()+n,v.begin()+m);

//删除向量中迭代器指定元素
v.erase(iterator it);

//向向量中迭代器指向元素it前增加一个元素x
v.insert(iterator it, const T& x);

//向向量中迭代器指向元素it前增加n个相同元素x
v.insert（iterator it, int n, const T& x）;

//向向量中迭代器指向元素(it)前插入另一个相同类型向量的[first,last)间的数据
v.insert(iterator it, const_iterator first, const_iterator last);
eg: a.insert(a.begin()+1,b+3,b+6);//设a、b都为{1,2,3,4,5,6,7,8,9},插入后为{1，4，5，6，2，3，4，5，6，7，8，9}//从第0个元素算起，在a的第1个元素后插入b的第3到第5个元素（不包括第6个元素）
```

## 2.3 有返回值的成员函数

### 2.3.1 返回值类型为指针类型

```c++
//返回v向量的头指针
v.begin();

//返回v向量的尾指针
v.end();
```

### 2.3.2 返回值类型为元素的数据类型

```c++
//返回v的最后一个元素
v.back();

//返回v的第一个元素
v.front();
```

### 2.3.3 返回值为其它类型

```c++
//返回向量中元素的个数(int)
v.size();

//返回最大可允许的vector元素数量值
v.max_size();

//返回向量对象在内存中总共可以容纳的元素个数(int)
v.capacity();
```





