---
layout: post
title: Algorithm-Sort(Basic)
subtitle: picture from https://www.pexels.com/search/wild%20animals/
author: maxshuang
categories: Algorithm
banner:
  image: /assets/images/post/algorithm-sort/pexels-bijoy.jpg
  opacity: 0.618
  background: "#000"
  height: "55vh"
  min_height: "38vh"
  heading_style: "font-size: 3.00em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: Algorithm
---

## Backgrond

排序是每个程序员学习算法的入门章节，除了教科书中提及的 selection sort, bubble sort, insertion sort, shell sort, merge sort, quick sort, heap sort, counting sort 和 bucket sort 之外，还有非常多的[排序算法](https://www.geeksforgeeks.org/sorting-algorithms/?ref=lbp)。

这篇文章不是为了涵盖所有的排序算法，而是为了回答一个问题： *可以无脑调用 C qsort 或者 std::sort 解决所有排序问题吗？*

答案显然是**不能**，除了一些应用场景的原因外，比如原本已经有序的多个数组 merge 使用 merge sort 是更合适的选择，还因为他们不是稳定排序算法。如果业务需要保持原有的次序不变，就需要调用 *std::stable_sort*。

## 算法性质

Stability 是排序算法的一个重要性质，它指的是对数据进行排序后，原来数值相同的数据能否保持次序不变。举个例子：  
5, 4, 3(1), 1, 3(2), 6  
其中我们通过 3(1) 和 3(2) 区分次序不同的 3。
* 稳定排序后的结果：1, 3(1), 3(2), 4, 5, 6  
* 不稳定排序可能的结果：1, 3(2), 3(1), 4, 5, 6  

在对数据的多个属性依次进行排序时，如果使用 unstable sort 算法，就会导致结果出错，[算法4](https://algs4.cs.princeton.edu/home/) 中的一个例子就是：
![unstable sort](/assets/images/post/algorithm-sort/unstable_sort.png)
在这个例子中，首先对 time 属性排序，然后对 location 属性排序。在对 time 属性排序之后，location 相同的数据已经在 time 属性上有序，只是位置不是临近的。如果对 location 排序的时候使用了 unstable sort，就可能在移动 location 相同数据时打乱原来的 time 次序，导致 same location 的数据是 time 无序的。

常见算法的稳定性如下：
![sort compare](/assets/images/post/algorithm-sort/sort_compare.png)

有意思的是，针对稳定算法，常见的对比算法并没有提供太多选择，在这个表中只有 insertion sort 和 merge sort 可以选择(还依赖于正确实现)。非比较排序中的 counting sort 的正确实现也是稳定的。

除了稳定性，我们还喜欢观察排序算法的其他几类性质：
1. in-place: 是否是就地排序，还是需要额外的空间
2. average time complexity: 平均时间复杂度
3. best time complexity: 最佳时间复杂度
4. worst time complexity: 最坏时间复杂度
5. space complexity

平时我们总会说 quick sort 是最快的排序算法，使用 quick sort 算法做任何数据的排序就可以了。这个说法的根据是 quick sort 是 in-place 且 average time complexity 为 $O(NlogN)$ 的算法，并且根据经验看，他有比 heap sort 更小的常数系数。

但是这是不准确的说法，起码你看系统库提供的 quick sort 实现很少是原生的 quick sort algorithm，更多的是一些 hybrid solution。它们会在不同场景下使用不同的排序算法，从而规避 quick sort $O(N^2)$ 的 worst time complexity。

这其实是一件挺有意思的事情，里面有一些你在学习排序算法时*无意遗漏的小细节*。

比如：
1. selection sort 的 average exchange 是 $O(N)$ 线性级别的，意味着对于 *写代价很高* 的介质，它可以是一个好的选择，即使它的 average time complexity 是 $O(N^2)$。
2. insertion sort 可以非常好得利用数据的原始分布，如果数据本身是 *sorted* 或者 *nearly sorted*，则它的每轮插入导致的数据移动可能是 $O(1)$，这样它就可以达到 best time complexity $O(N)$。这是 quick sort, merge sort 和 heap sort 等算法无法实现的 best time complexity。  
另外，它的实现简单，没有优化时间复杂度的一些代码 overhead，比如维护 heap 之类的代码，所以在很多生产级别排序算法实现中，在数据小时会选择 insertion sort。
3. 相对于 quick sort, heap sort 虽然在常数系数上会相对大，但是 heap sort 的  worst time complexity 是 $O(NlogN)$，并且 space complexity 是 $O(1)$。  
所以在一些小数据规模下，heap sort 能表现出比 quick sort 更好的 space efficiency 和 time efficiency。
4. merge sort 是 stable 的，同时 best time complexity 和 worst time complexity 都是 $O(NlogN)$，对于稳定算法来说真是好，唯一遗憾的是需要 $O(N)$ extra space。  
有一些 [inplace merge sort](https://www.geeksforgeeks.org/in-place-merge-sort/) 的实现，比如 SGI STL 的 stable sort 是使用的就是 inplace 算法：

```
template <class _RandomAccessIter>
void __inplace_stable_sort(_RandomAccessIter __first,
                           _RandomAccessIter __last) {
  ...
  __inplace_stable_sort(__first, __middle);
  __inplace_stable_sort(__middle, __last);
  __merge_without_buffer(__first, __middle, __last,
                         __middle - __first,
                         __last - __middle);
}
```

这些算法中有些只能达到 average time complexity $O(N*log^{2}N)$ 或者 $O(N^{2}*logN)$。
5. 在不同类型的比较算法设计中，可以观察到一些有意思的结构，$O(NlogN)$ 级别的算法总会在每个 round 共享一些结构信息，比如 merge sort 和 quick sort 这种 divide and conquer 算法，每个 round 都在使得整体数据表现得 roughly sorted。对于 heap sort，其核心思路更是不需要一开始就完全有序，只需要 roughly sort 就可以以 $O(1)$ 的时间复杂度获取到最大或者最小值，然后再以 $O(logN)$ 的时间复杂度新增或者删除元素。

## quick sort 算法实现
quick sort 算法作为基础算法，无论是理论还是实现都被研究得很透彻，但是 quick sort 算法并不是那么容易实现，里面有很多细节，需要一些技巧才能快速从零开始实现 quick sort 算法。

quick sort 算法的逻辑在任何一本算法书中都可以找到，我们主要是仔细看下里面的细节。

首先算法的大体流程是一个 divide and conquer，通过 partition 找到一个中间位置，然后再分别对左边和右边调用相同的流程。

```
void quick_sort(int* vec, int lf, int rt) {
    // partition
    int cut = partition(vec, lf, rt);
    // recursively sort left part and right part
    quick_sort(vec, lf, cut-1);
    quick_sort(vec, cut+1, rt);
}
```
### 如何划分 partition
我们的问题从算法结构描述就开始了：**如何划分 partition?**。
对于 quick sort 的分而治之算法结构，我们有以下几种方式做 partition:
1. vec[lf, cut-1] <= vec[cut] <= vec[cut+1, rt]
2. vec[lf, cut-1] <= vec[cut] < vec[cut+1, rt]
3. vec[lf, cut-1] < vec[cut] <= vec[cut+1, rt]
4. vec[lf, cut-1] < vec[cut] < vec[cut+1, rt]

这几种方式都是正确合理的划分，都可以保证在依次 partition 之后数据以 cut 为中心是 roughly sorted 的。

**所以这些不同的划分有区别吗?**  

**还真有**，在排序数据存在大量相同值的场景下，方式 2/3 划分会出现严重的 data skew。原因是，方式 2/3 可能将值为 vec[cut] 全部放在一边，导致最后的数据分割点 cut 偏离数据中位数点，此时 quick sort 从一个平衡树退化成线性链表，递归轮次变成 $O(N)$，整体的 time complexity 达到了 worst $O(N^2)$。举个例子：  
3, 4, 3, 3, 1, 3, 5, 3, 3, 3, 4, 3

选择 vec[cut]==3 时，一轮 partition 后结果为：  
3, 3, 3, 3, 1, 3, 3, 3, 3[cut], 5, 4, 4  

此时 cut 已经偏离数据中位数点。

为了避免 data skew 造成的性能退化问题，实际的 partition 会选择方式 1 和 4，其中方式 4 是我们后面会提及的 3-way partition，正常的 2-way partition 选择的是方式 1。

### 实现中的细节

对于核心的 partition 函数，大致的逻辑是：
1. 使用某种方式选择一个基准值 pivot。
2. 使用左右指针分别从数组的左右边界开始向中间收缩，将小于等于 pivot 的元素调换到左边，大于等于 pivot 的元素调换到右边。
3. 当左右指针相遇或者左指针大于右指针时流程结束，可以找到一个中间位置 cut，满足 partition 要求。

我们看一个可能的实现：

```
// check range: [l, r]
int partition_maybe_right(int *vec, int l, int r)
{
    // ??? Q1: Randomly select the first element as pivot
    int pivot = vec[l];
    // ??? Q2: Should I use [l, r] or [l, r+1) 
    int i = l, j = r;
    // ??? Q3: Should I use i<j or i<=j
    while (i < j)
    {
        // ??? Q4: Still have data skew here
        // Here, we follow the partition:
        // vec[lf, cut-1] <= vec[cut] <= vec[cut+1, rt]
        while (vec[i] <= pivot && i < j)
            i++;
        while (vec[j] >= pivot && j > i)
            j--;

        // ??? Q5: Should I use i<j or i<=j
        if (i < j)
        {
            // exchange the left and right element
            exchange(vec, i, j);
            i++;
            j--;
        }
    }

    // ??? Q6: Should I return i or j?
    return j;
}
```

在这个实现中，看起来它可以 work，但是我们起码遇到了 5 个细节问题：
1. Q1： 如何选择一个合理的 pivot，随机选可能导致 cut 点偏离数据中位数点，导致性能退化到 $O(N^2)$。
2. Q2： 我们应该使用开右边界 [l, r+1) 还是闭右边界 [l, r]？
3. Q3： 在使用闭右边界时，while 结束的标志是 i\<j 还是 i\<\=j？ 标准是什么？
4. Q4:  为什么遵循了划分规则还是有 data skew？
5. Q5： 应该是 i<j，因为 i==j or i>j 都不需要交换。
6. Q6： 这里应该返回 i 还是 j？

### pivot 选择
pivot 选择关系到 quick sort 的性能，*理想上应该选择数据的中位数*，这样 partition 划分后左右区间数据量是大致相同的。

由于在大数据量下获取数据中位数性能损耗比较高，所以一般的做法是：
1. 对数据做 shuffle，随机选 pivot； 
2. 采样的方式计算中位数，比如3分位点，5分位点，或者随机采样；

选择完 pivot 后可以交换到数组头或者尾的位置，方便获取 pivot 的值。

### 如何选择右开闭区间
*选择右开闭区间都可以，前提是不影响正确性*。问题 Q2 和 Q3 是一起的，需要保持一致保证正确性。比如：
1. 选择右开区间 [l, r+1)
此时 int i = l, j = r+1，对于循环而言，其运行条件为 while(i\<j)，因为当 i>=j 时，区间 [i, j) 内已经没有任何数据了。

2. 选择右闭区间 [l, r]
此时 int i = l, j = r，对于循环而言，其运行条件为 while(i\<=j)，因为当 i>j 时，区间 [i, j] 内已经没有任何数据了。但是当 i==j 时区间 [i, i] 内还有数据 vec[i]，此时是不能结束循环的。

### 实现中的 data skew
```
// Here, we follow the partition:
// vec[lf, cut-1] <= vec[cut] <= vec[cut+1, rt]
while (vec[i] <= pivot && i < j)
    i++;
while (vec[j] >= pivot && j > i)
    j--;
```
在这个实现中，左右指针在移动时为了满足 vec[lf, cut-1] <= vec[cut] <= vec[cut+1, rt]，会在 vec[i] == pivot 或者 vec[j] == pivot 继续移动，它存在的问题是：**无法解决原始数据分布就存在的 data skew**。  
比如：
3, 3, 1, 3, 3, 3, 2, 3, 3(cut), 4, 5  
上述实现会导致 3 都分布到 cut 的左区间。

正确的实现应该允许在遇到 pivot 值时进行交换，保证左右区间 pivot 值的数量大致相同。

**有意思的地方在于：** 这种正确实现正是 quick sort 算法不具备 stability 性质的原因，*它交换了前后 pivot 值的次序*。*而上面错误的实现却是稳定的*。

正确的实现：
```
while (vec[i] < pivot && ...) i++;
while (vec[j] > pivot && ...) j--;
```

### 改进的实现

按照上面的改进，我们使用右闭区间，修改后的代码：

```
// check range: [l, r]
int partition_better(int *vec, int l, int r)
{
    int pivot = vec[l];
    int i = l+1, j = r;
    while (i <= j)
    {
        while (vec[i] < pivot && i < r) i++;
        while (vec[j] > pivot && j > l) j--;
        if (i < j) exchange(vec, i++, j--);
        // three situations here:
        // 1. i>j
        // 2. i==j && vec[i]==pivot, need break the loop
        // 3. i==j && vec[i]!=pivot, next term will meet i>j
        else if(i==j && vec[i]==pivot) break;
    }
    
    // j is the cut, exchang l and j, so vec[j]==pivot
    // meet the requirement: vec[lf, cut-1] <= vec[cut] <= vec[cut+1, rt]
    exchange(vec, l, j);
    return j;
}
```

这个看起来正确的实现还是有 bug，然后考虑这样一个数据：  
5, 4, 3, 2, 1  
按照上面的逻辑，while (vec[i] < pivot && i < r) i++; 会触发 i==r 的条件导致 while 退出，但是其他的所有分支都无法推进逻辑，导致出现 dead loop。

我们可以看到，整个实现过程中即使我们注意到了一些细节问题，逻辑还是非常容易出错，corner cases 也很多。

那有没有比较好的算法实现原则，既可以有效验证算法正确性，又可以指导实现？

***循环不变式**。

### 循环不变式实现
一个比较好的做法是利用[算法导论](https://edutechlearners.com/download/Introduction_to_algorithms-3rd%20Edition.pdf)中的循环不变性质(loop invariant)保证算法正确性，简单说就是保证算法初始状态符合一个算法性质的约定，每次循环都保证约定成立，这样算法递推到循环结束自然也满足约定。

对于 quick sort algorithm，由于需要满足 vec[i] >= pivot 停止，vec[j]<= pivot 停止，并且我们想要使用右开区间，所以我们构造一个可行的 invariant A，比如：

```
check range: [l, r]
keep the following invariant for any situation: 
1. [l, i) <= pivot
2. [i, j) not sure
3. [j, r] >= pivot
```

根据这个不变式 invariant A，我们实现 partition:

```
// check range: [l, r]
int partition(int *vec, int l, int r)
{
    // select a proper pivot，then exchange pivot to pos `l`
    int pivot = select(vec, l, r);
       
    // initial state: 
    // meet: [l, l) <= pivot, no data here
    // meet: [l, r+1) not sure, original unsorted data
    // meet: [r+1, r] >= pivot, no data here
    int i = l, j = r + 1;

    // open end, use i<j
    while (i<j)
    {
        // after this while, [l, i) <= pivot
        while (vec[++i] < pivot) if(i==r) break;

        // after this while, (j, r] >= pivot
        while (vec[--j] > pivot) if(j==l) break;
                
        // after this exchange,
        // meet: [l, i) <= pivot
        // meet: [i, j) not sure
        // meet: [j, r] >= pivot
        if(i<j)
            exchange(vec, i, j);
    }

    // After while (i<j) exits:
    // 1. i==j, vec[i]==vec[j]==pivot
    // 2. i==j==r, vec[i]=vec[j]<pivot
    // !!!! violate: [j, r] >= pivot !!!
    // 3. i>j && (vec[j] <= pivot || vec[i] >= pivot)
    // meet: [l, i) <= pivot
    // !!!! violate: [j, r] >= pivot !!!
    // meet: (i, j) not sure, because no data

    // fix the above violation to meet: [j, r] >= pivot
    exchange(vec, l, j);
    return j;
}
```

在这个实现中，我们可以看到，当出现 i\<j 的场景时，可能出现 vec[j] \<= pivot 或者 vec[i] \>= pivot， 为了不违反 [j, r] >= pivot 的不变式，需要用 pivot 进行修正。

### 3-way quick sort

对于有大量重复元素的数组，使用 3-way quick sort 可以实现 Average Time Complexity O(N) 的性能，该算法排序后大致的样子是：
![three-way-partition](/assets/images/post/algorithm-sort/three_way_partition.png)

实现 3-way quick sort 也可以依赖 loop invariant:

```
// check range: [l, r]
// keep the following invariant for any situation: 
// 1. vec[l..lt] < pivot
// 2. vec[lt+1..i-1] == pivot
// 3. vec[i, gt) not sure
// 4. vec[gt..r] > pivot

void quick_3way_range(int *vec, int l, int r)
{
    if (l >= r)
        return;

    int pivot = vec[l];
    // assign lt, gt, i to keep the invariant
    int lt = l - 1; // less than boundary
    int gt = r + 1; // greater than boundary
    int i = l;
    while (i < gt)
    {
        if (vec[i] == pivot)
            ++i; // expand the range of pivot
        else if (vec[i] < pivot)
            exchange(vec, ++lt, i++); // expand the range of less than, since original vec[lt+1]==pivot, we need to skip vec[i]
        else
            exchange(vec, --gt, i); // expand the range of greater than, since original vec[gt-1]!=pivot, we need to recheck vec[i]
    
        // Also keep the invariant here.
    }

    quick_3way_range(vec, l, lt);
    quick_3way_range(vec, gt, r);
}
```

## 应用中的 sort
Python 标准库中的 sort 采用 [TimSort](https://bugs.python.org/file4451/timsort.txt) 算法，它的 best time complexity $O(N)$，average time complexity 和 worst time complexity 都可以满足 $O(NlogN)$。

C++ 标准库中的 std::sort 使用 [Introsort](https://www.geeksforgeeks.org/introsort-or-introspective-sort/)，这是一个结合了 quick sort, heap sort 和 insertion sort 的方案，保证算法的 worst time complexity 满足 $O(NlogN)$。其算法流程上大致为：
1. quick sort 不断划分 partition。
2. 当 quick sort 递归深度过深时，采用 heap sort，前面我们说过 heap sort 也是 inplace 的算法，并且 average time complexity 和 worst time complexity 都是 $O(NlogN)$ 级别的，只是常数系数会相对大。
3. 当整个数组 nearly sorted 之后，采用 insertion sort 排序。

SGI STL 中实现如下：

```
template <class _RandomAccessIter>
inline void sort(_RandomAccessIter __first, _RandomAccessIter __last) {
  ...
  if (__first != __last) {
    __introsort_loop(__first, __last,
                     __VALUE_TYPE(__first),
                     __lg(__last - __first) * 2);
    __final_insertion_sort(__first, __last);
  }
}

template <class _RandomAccessIter, class _Tp, class _Size, class _Compare>
void __introsort_loop(_RandomAccessIter __first,
                      _RandomAccessIter __last, _Tp*,
                      _Size __depth_limit, _Compare __comp)
{
  while (__last - __first > __stl_threshold) {
    if (__depth_limit == 0) {
      //!!! [NOTE]: This is the heap sort
      partial_sort(__first, __last, __last, __comp);
      return;
    }

    //!!! [NOTE]: tail recursion version of quick sort
    --__depth_limit;
    _RandomAccessIter __cut =
      __unguarded_partition(__first, __last,
                            _Tp(__median(*__first,
                                         *(__first + (__last - __first)/2),
                                         *(__last - 1), __comp)),
       __comp);
    __introsort_loop(__cut, __last, (_Tp*) 0, __depth_limit, __comp);
    __last = __cut;
  }
}
```