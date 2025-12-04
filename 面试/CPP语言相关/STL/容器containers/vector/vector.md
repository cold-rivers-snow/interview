# vector

vector 应该是最常用的容器了。它的名字“向量”来源于数学术语，但在实际应用中，我们把它当成**动态数组**更为合适。它基本相当于 Java 的 ArrayList 和 Python 的 list。

和 string 相似，vector 的成员在内存里连续存放，同时 begin、end、front、back 成员函数指向的位置也和 string 一样，大致如下图所示：

![img](https://static001.geekbang.org/resource/image/24/10/247951f886561c30ced2eb7700f9d510.png)

除了容器类的共同点，vector 允许下面的操作（不完全列表）：

* 可以使用中括号的下标来访问其成员（同 string）
* 可以使用 data 来获得指向其内容的裸指针（同 string）
* 可以使用 capacity 来获得当前分配的存储空间的大小，以元素数量计（同 string）
* 可以使用 reserve 来改变所需的存储空间的大小，成功后 capacity() 会改变（同 string）
* 可以使用 resize 来改变其大小，成功后 size() 会改变（同 string）
* 可以使用 pop_back 来删除最后一个元素（同 string）
* 可以使用 push_back 在尾部插入一个元素（同 string）
* 可以使用 insert 在指定位置前插入一个元素（同 string）
* 可以使用 erase 在指定位置删除一个元素（同 string）
* 可以使用 emplace 在指定位置构造一个元素
* 可以使用 emplace_back 在尾部新构造一个元素

大家可以留意一下 push\_… 和 pop_… 成员函数。它们存在时，说明容器对指定位置的删除和插入性能较高。vector 适合在尾部操作，这是它的内存布局决定的。只有在尾部插入和删除时，其他元素才会不需要移动，除非内存空间不足导致需要重新分配内存空间。

当 push_back、insert、reserve、resize 等函数导致内存重分配时，或当 insert、erase 导致元素位置移动时，vector 会试图把元素“移动”到新的内存区域。vector 通常保证强异常安全性，如果元素类型没有提供一个**保证不抛异常的移动构造函数**，vector 通常会使用拷贝构造函数。因此，对于拷贝代价较高的自定义元素类型，我们应当定义移动构造函数，并标其为 noexcept，或只在容器中放置对象的智能指针。这就是为什么我之前需要在 smart_ptr 的实现中标上 noexcept 的原因。

下面的代码可以演示这一行为：

```c++
#include <iostream>
#include <vector>

using namespace std;

class Obj1 {
public:
  Obj1()
  {
    cout << "Obj1()\n";
  }
  Obj1(const Obj1&)
  {
    cout << "Obj1(const Obj1&)\n";
  }
  Obj1(Obj1&&)
  {
    cout << "Obj1(Obj1&&)\n";
  }
};

class Obj2 {
public:
  Obj2()
  {
    cout << "Obj2()\n";
  }
  Obj2(const Obj2&)
  {
    cout << "Obj2(const Obj2&)\n";
  }
  // 注意里的不同
  Obj2(Obj2&&) noexcept
  {
    cout << "Obj2(Obj2&&)\n";
  }
};

int main()
{
  vector<Obj1> v1;
  v1.reserve(2);
  v1.emplace_back();
  v1.emplace_back();
  v1.emplace_back();

  vector<Obj2> v2;
  v2.reserve(2);
  v2.emplace_back();
  v2.emplace_back();
  v2.emplace_back();
}
```

我们可以立即得到下面的输出：

```
Obj1()
Obj1()
Obj1()
Obj1(const Obj1&)
Obj1(const Obj1&)
Obj2()
Obj2()
Obj2()
Obj2(Obj2&&)
Obj2(Obj2&&)
```

Obj1 和 Obj2 的定义只差了一个 noexcept，但这个小小的差异就导致了 vector 是否会移动对象。这点非常重要。

C++11 开始提供的 emplace… 系列函数是为了提升容器的性能而设计的。你可以试试把 v1.emplace_back() 改成 v1.push_back(Obj1())。对于 vector 里的内容，结果是一样的；但使用 push_back 会额外生成临时对象，多一次（移动或拷贝）构造和析构。如果是移动的情况，那会有小幅性能损失；如果对象没有实现移动的话，那性能差异就可能比较大了。

现代处理器的体系架构使得对连续内存访问的速度比不连续的内存要快得多。因而，vector 的连续内存使用是它的一大优势所在。当你不知道该用什么容器时，缺省就使用 vector 吧。

vector 的一个主要缺陷是大小增长时导致的元素移动。如果可能，尽早使用 reserve 函数为 vector 保留所需的内存，这在 vector 预期会增长很大时能带来很大的性能提升。

## vector越界访问下标？

通过下标访问vector中的元素时不会做边界检查，即便下标越界。

也就是说，下标与first迭代器相加的结果超过了finish迭代器的位置，程序也不会报错，而是返回这个地址中存储的值。

如果想在访问vector中的元素时首先进行边界检查，可以使用vector中的**at函数**。通过使用at函数不但可以通过下标访问vector中的元素，而且在at函数内部会对下标进行边界检查。

## STL 中vector删除其中的元素，迭代器如何变化？为什么是两倍扩容？释放空间？

size()函数返回的是已用空间大小，capacity()返回的是总空间大小，capacity()-size()则是剩余的可用空间大小。当size()和capacity()相等，说明vector目前的空间已被用完，如果再添加新元素，则会引起vector空间的动态增长。

由于动态增长会引起重新分配内存空间、拷贝原空间、释放原空间，这些过程会降低程序效率。因此，可以使用reserve(n)预先分配一块较大的指定大小的内存空间，这样当指定大小的内存空间未使用完时，是不会重新分配内存空间的，这样便提升了效率。只有当n>capacity()时，调用reserve(n)才会改变vector容量。

 resize()成员函数只改变元素的数目，不改变vector的容量。

1、空的vector对象，size()和capacity()都为0

2、当空间大小不足时，新分配的空间大小为原空间大小的2倍。

3、使用reserve()预先分配一块内存后，在空间未满的情况下，不会引起重新分配，从而提升了效率。

4、当reserve()分配的空间比原空间小时，是不会引起重新分配的。

5、resize()函数只改变容器的元素数目，未改变容器大小。

6、用reserve(size_type)只是扩大capacity值，这些内存空间可能还是“野”的，如果此时使用“[ ]”来访问，则可能会越界。而resize(size_type new_size)会真正使容器具有new_size个对象。

 不同的编译器，vector有不同的扩容大小。在vs下是1.5倍，在GCC下是2倍；

空间和时间的权衡。简单来说， 空间分配的多，平摊时间复杂度低，但浪费空间也多。

使用k=2增长因子的问题在于，每次扩展的新尺寸必然刚好大于之前分配的总和，也就是说，之前分配的内存空间不可能被使用。这样对内存不友好。最好把增长因子设为(1,2)

 对比可以发现采用成倍方式扩容，可以保证常数的时间复杂度，而增加指定大小的容量只能达到O(n)的时间复杂度，因此，使用成倍的方式扩容。

如何释放空间：

由于vector的内存占用空间只增不减，比如你首先分配了10,000个字节，然后erase掉后面9,999个，留下一个有效元素，但是内存占用仍为10,000个。所有内存空间是在vector析构时候才能被系统回收。empty()用来检测容器是否为空的，clear()可以清空所有元素。但是**即使clear()，vector所占用的内存空间依然如故，无法保证内存的回收。**

**如果需要空间动态缩小，可以考虑使用deque**。如果vector，可以用swap()来帮助你释放内存。

```cpp
vector(Vec).swap(Vec);
 //将Vec的内存空洞清除；
 vector().swap(Vec);
 //清空Vec的内存；
```

# C++ vector 缩容机制

`std::vector` 在 GCC 中的扩容规则（默认2倍扩容）是开发者熟知的特性，但**缩容并非 vector 的“自动行为”** ——C++ 标准未规定 vector 有自动缩容逻辑，GCC（libstdc++）也未实现“元素减少时自动释放多余内存”，需通过手动手段触发缩容。

## 一、核心结论：vector 无自动缩容（GCC/libstdc++ 实现）

### 1. 为什么不自动缩容？

C++ 标准设计 vector 的核心原则是「避免不必要的内存分配/释放开销」：

- 扩容是“预判未来需求”（2倍扩容可减少频繁扩容的拷贝开销）；
- 若自动缩容（如 `erase`/`pop_back` 后立即释放多余内存），会导致：
  1. 频繁的内存释放+重新分配（若后续又添加元素，反而增加性能损耗）；
  2. 破坏“迭代器/指针/引用的稳定性”（缩容会触发内存拷贝，导致原有迭代器失效）。

因此 GCC/libstdc++ 中，vector 的 `capacity()`（已分配内存容量）仅会在**扩容**时增长，不会随 `size()`（实际元素数）减少而自动下降。

### 2. 关键概念区分（面试必背）

| 概念         | 含义                          | 变化规则（GCC）                            |
| ---------- | --------------------------- | ------------------------------------ |
| size()     | 实际存储的元素个数                   | `push_back` 增长，`erase`/`pop_back` 减少 |
| capacity() | 已分配的内存可容纳的元素个数              | 扩容时（如 `push_back` 触发）增长，无自动减少        |
| 内存占用       | `capacity() * sizeof(元素类型)` | 仅扩容时增加，缩容需手动触发                       |

**示例验证（GCC）**：

```cpp
#include <vector>
#include <iostream>
using namespace std;

int main() {
    vector<int> v;
    // 扩容：capacity 自动增长（GCC 2倍扩容）
    for (int i = 0; i < 10; ++i) {
        v.push_back(i);
        cout << "size: " << v.size() << ", capacity: " << v.capacity() << endl;
    }
    // 输出：capacity 从 1→2→4→8→16（2倍扩容）

    // 缩容？erase 仅减少 size，capacity 不变
    v.erase(v.begin() + 5, v.end());  // 删除后 5 个元素
    cout << "erase 后：size=" << v.size() << ", capacity=" << v.capacity() << endl;
    // 输出：size=5, capacity=16（capacity 仍为 16，未缩容）

    // pop_back 同理：size 减少，capacity 不变
    v.pop_back();
    cout << "pop_back 后：size=" << v.size() << ", capacity=" << v.capacity() << endl;
    // 输出：size=4, capacity=16
    return 0;
}
```

## 二、手动缩容的三种方式（GCC 下的实现）

若确需释放 vector 多余内存（如元素大幅减少且长期不再添加），可通过以下三种方式触发缩容，核心原理是「创建临时 vector 转移数据，利用 RAII 释放原内存」。

### 1. 方式1：swap 技巧（C++11 前主流）

核心逻辑：创建临时 vector（仅包含有效元素），通过 `swap` 交换内部数据，临时 vector 析构时释放原 vector 的多余内存。

```cpp
// 缩容函数封装（通用）
template <typename T>
void shrink_vector(vector<T>& v) {
    vector<T> temp(v.begin(), v.end());  // 临时 vector：capacity = size
    v.swap(temp);  // 交换内部指针，原内存由 temp 析构释放
}

// 调用示例
vector<int> v(1000);  // size=1000, capacity=1000
v.erase(v.begin() + 10, v.end());  // size=10, capacity=1000
shrink_vector(v);
cout << "swap 后：size=" << v.size() << ", capacity=" << v.capacity() << endl;
// 输出：size=10, capacity=10（成功缩容）
```

### 2. 方式2：clear + swap（清空并缩容）

若需清空 vector 且释放所有内存（capacity 归 0），可结合 `clear()` 和 `swap`：

```cpp
vector<int> v(1000);
v.clear();  // size=0, capacity=1000（仅清空元素，内存未释放）
vector<int>().swap(v);  // 空 vector 交换，v 的 capacity 变为 0
cout << "clear+swap 后：size=" << v.size() << ", capacity=" << v.capacity() << endl;
// 输出：size=0, capacity=0
```

### 3. 方式3：shrink_to_fit（C++11 引入，推荐）

C++11 为 vector 新增 `shrink_to_fit()` 成员函数，语义是「请求」vector 将 capacity 缩减至与 size 一致——GCC/libstdc++ 中该函数会触发缩容（底层实现类似 swap 技巧），但需注意：

- 标准将其定义为“非强制请求”（允许编译器忽略），但 GCC/Clang/MSVC 均实现了缩容逻辑；
- 缩容后迭代器/指针/引用会失效（内存重新分配）。

```cpp
vector<int> v(1000);
v.erase(v.begin() + 10, v.end());  // size=10, capacity=1000
v.shrink_to_fit();  // C++11 缩容
cout << "shrink_to_fit 后：size=" << v.size() << ", capacity=" << v.capacity() << endl;
// 输出：size=10, capacity=10（GCC 下成功缩容）
```

## 三、缩容的性能代价与适用场景

### 1. 缩容的性能损耗（面试必知）

缩容本质是「内存重新分配 + 元素拷贝/移动」，会带来两类开销：

- 时间开销：拷贝/移动元素的耗时（O(n)，n 为有效元素数）；
- 稳定性开销：缩容后原有迭代器、指针、引用全部失效。

### 2. 推荐缩容的场景

- 元素数量大幅减少（如 size 仅为 capacity 的 10% 以下）；
- 长期不再向 vector 添加元素（如内存敏感的嵌入式场景、长期运行的后台服务）；
- 需释放大量内存以降低程序整体内存占用（如大 vector 使用完毕后）。

### 3. 禁止缩容的场景

- 后续可能频繁添加元素（缩容后再扩容会重复分配/拷贝，得不偿失）；
- 依赖迭代器/指针/引用的稳定性（缩容会导致失效）；
- 元素数量少（如 capacity - size < 100），缩容收益远小于开销。

## 四、GCC 下 vector 扩容/缩容的核心对比

| 操作             | 触发条件                                       | capacity 变化 | 性能特征                     | 迭代器稳定性       |
| -------------- | ------------------------------------------ | ----------- | ------------------------ | ------------ |
| 扩容（2倍）         | push_back/emplace_back 导致 size == capacity | 增长为原容量的 2 倍 | O(n) 拷贝（旧→新内存）           | 失效           |
| 缩容（手动）         | shrink_to_fit/swap 技巧                      | 缩减至 size 大小 | O(n) 拷贝/移动               | 失效           |
| erase/pop_back | 元素删除                                       | 不变          | O(k) 元素移动（k 为删除后需移动的元素数） | 被删除位置后的迭代器失效 |

## 五、面试易错点与避坑指南

### 1. 误区1：vector 的 erase/pop_back 会自动缩容

❌ 错误：erase/pop_back 仅修改 size，capacity 保持不变，内存不会释放；
✅ 正确：需手动调用 shrink_to_fit() 或 swap 技巧触发缩容。

### 2. 误区2：shrink_to_fit 是强制缩容

❌ 错误：C++ 标准将其定义为“请求”，编译器可忽略（但主流编译器均实现）；
✅ 正确：GCC 下可放心使用，是 C++11+ 缩容的首选方式。

### 3. 误区3：缩容一定提升性能

❌ 错误：若后续需添加元素，缩容会导致重新扩容（重复拷贝），反而降低性能；
✅ 正确：仅在“长期不再添加元素”时缩容才有意义。

### 4. 误区4：clear() 会释放内存

❌ 错误：clear() 仅销毁元素（size=0），capacity 不变，内存仍占用；
✅ 正确：需结合 swap 或 shrink_to_fit() 释放内存。

## 六、面试回答模板（简洁版）

GCC 中 vector 采用 2 倍扩容策略，但**无自动缩容机制**——capacity 仅会随扩容增长，不会因 erase/pop_back 减少。

缩容需手动触发，核心方式：

1. C++11 前用 swap 技巧（创建临时 vector 交换，释放多余内存）；
2. C++11+ 推荐 shrink_to_fit()（语义清晰，底层实现类似 swap）；
3. 清空并释放所有内存可用 clear()+swap。

缩容的核心注意点：

- 缩容本质是内存重分配+元素拷贝，会导致迭代器失效；
- 仅在“元素大幅减少且长期不再添加”时缩容，否则会因重复扩容降低性能。
