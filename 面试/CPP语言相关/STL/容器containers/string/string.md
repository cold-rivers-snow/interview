### string

string 一般并不被认为是一个 C++ 的容器。但鉴于其和容器有很多共同点，我们先拿 string 类来开说。

string 是模板 basic_string 对于 char 类型的特化，可以认为是一个只存放字符 char 类型数据的容器。“真正”的容器类与 string 的最大不同点是里面可以存放任意类型的对象。

跟其他大部分容器一样， string 具有下列成员函数：

* begin 可以得到对象起始点
* end 可以得到对象的结束点
* empty 可以得到容器是否为空
* size 可以得到容器的大小
* swap 可以和另外一个容器交换其内容

(需要知道 C++ 的 begin 和 end 是**半开半闭区间**[)：在容器非空时，begin 指向第一个元素，而 end 指向最后一个元素后面的位置；在容器为空时，begin 等于 end。在 string 的情况下，由于考虑到和 C 字符串的兼容，end 指向代表字符串结尾的 \0 字符。）

上面就几乎是所有容器的共同点了。也就是说：

* 容器都有开始和结束点
* 容器会记录其状态是否非空
* 容器有大小
* 容器支持交换

当然，这只是容器的“共同点”而已。每个容器都有其特殊的用途。

string 的内存布局大致如下图所示：

![img](https://static001.geekbang.org/resource/image/ee/62/eec393f933220a9998b7235c8acc1862.png)

下面你会看到，不管是内存布局，还是成员函数，string 和 vector 是非常相似的。

string 当然是为了存放字符串。和简单的 C 字符串不同：

* string 负责自动维护字符串的生命周期
* string 支持字符串的拼接操作（如之前说过的 + 和 +=）
* string 支持字符串的查找操作（如 find 和 rfind）
* string 支持从 istream 安全地读入字符串（使用 getline）
* string 支持给期待 const char* 的接口传递字符串内容（使用 c_str）
* string 支持到数字的互转（stoi 系列函数和 to_string）
* 等等

推荐你在代码中尽量使用 string 来管理字符串。不过，对于对外暴露的接口，情况有一点复杂。我一般不建议在接口中使用 const string&，除非确知调用者已经持有 string：如果函数里不对字符串做复杂处理的话，使用 const char* 可以避免在调用者只有 C 字符串时编译器自动构造 string，这种额外的构造和析构代价并不低。反过来，如果实现较为复杂、希望使用 string 的成员函数的话，那就应该考虑下面的策略：

* 如果不修改字符串的内容，使用 const string& 或 C++17 的 **string_view** 作为参数类型。后者是最理想的情况，因为即使在只有 C 字符串的情况，也不会引发不必要的内存复制。
* 如果需要在函数内修改字符串内容、但不影响调用者的该字符串，使用 string 作为参数类型（自动拷贝）。
* 如果需要改变调用者的字符串内容，使用 string& 作为参数类型（通常不推荐）。

估计大部分同学对 string 已经很熟悉了。我们在此只给出一个非常简单的小例子：

```c++
string name;
cout << "What's your name? ";
getline(cin, name);
cout << "Nice to meet you, " << name
     << "!\n";
```

# C++ 面试：string_view 与 string 与 const string& 核心区别

`string_view`（C++17 引入）、`std::string`、`const std::string&` 是 C++ 中处理字符串的核心类型，三者的核心差异集中在**内存管理、拷贝开销、修改权限、适用场景**四个维度，是面试中考察字符串优化的高频考点。

## 一、核心区别总览（面试必背对比表）

| 特性维度         | std::string                  | const std::string&           | std::string_view（C++17+）                          |
| ------------ | ---------------------------- | ---------------------------- | ------------------------------------------------- |
| **本质**       | 可修改的字符串容器（管理堆内存）             | 字符串的只读引用（无内存管理）              | 字符串的只读“视图”（无内存管理，仅持有指针+长度）                        |
| **内存管理**     | 拥有内存（堆上存储字符数据），析构时释放         | 不拥有内存，仅引用 `string` 的内存       | 不拥有内存，仅指向任意字符串的内存（C风格/`string`/字面量）               |
| **拷贝开销**     | 深拷贝（复制字符数据，O(n) 时间）          | 浅拷贝（仅复制引用，O(1) 时间）           | 浅拷贝（仅复制指针+长度，O(1) 时间）                             |
| **修改权限**     | 可读写（如 `push_back`、`replace`） | 只读（无法修改引用的 `string`）         | 只读（仅读取，无法修改原字符串）                                  |
| **空指针/空值**   | 无空状态（空 `string` 指向空字符数组）     | 不能引用空 `string`（否则未定义行为）      | 支持空视图（指针为 `nullptr` 或长度为 0）                       |
| **原内存失效风险**  | 无（自身管理内存）                    | 有（引用的 `string` 析构/修改会导致悬垂引用） | 有（原字符串内存释放/修改会导致视图失效）                             |
| **支持的字符串类型** | 仅自身（可从 C 风格字符串构造）            | 仅 `std::string`              | C 风格字符串、`std::string`、字符串字面量、字符数组                 |
| **额外开销**     | 堆分配开销（小字符串优化 SSO 除外）、内存拷贝    | 无额外开销（仅引用）                   | 无额外开销（仅指针+长度）                                     |
| **适用场景**     | 需要修改字符串、长期持有字符串              | 函数参数（避免拷贝 `string`）、短期只读访问   | 高频只读字符串访问、跨类型字符串读取（C风格/`string`）、避免临时 `string` 构造 |

## 二、关键差异拆解（结合原理+示例）

### 1. 内存模型：“拥有” vs “引用” vs “视图”

这是三者最核心的区别，直接决定性能和使用风险：

#### （1）std::string：内存的“所有者”

`string` 自身管理字符数据的内存（默认在**堆上**，**小字符串优化 SSO 会用栈内存**），析构时自动释放内存。

```cpp
std::string s = "hello";
// s 拥有内存：堆上存储 'h' 'e' 'l' 'l' 'o' '\0'（SSO 场景则在栈上）
s += " world";  // 可修改，内存不足时自动扩容（重新分配+拷贝）
```

- 核心特点：**深拷贝**——赋值/传参时会复制整个字符数组，开销与字符串长度成正比（O(n)）。

#### （2）const std::string&：string 的“只读引用”

仅指向已存在的 `string` 对象，不管理内存，也无法修改原 `string`：

```cpp
void func(const std::string& s) {
    // 仅能读取，无法修改 s（如 s.push_back('!') 编译报错）
    std::cout << s << std::endl;
}

std::string s = "hello";
func(s);  // 仅传递引用，无拷贝开销（O(1)）
```

- 核心特点：**浅拷贝（引用）**——避免拷贝，但仅能引用 `std::string` 类型，无法直接接收 C 风格字符串（如 `func("hello")` 会隐式构造临时 `string`，产生拷贝开销）。

#### （3）std::string_view：任意字符串的“只读视图”

仅存储「**指向字符数据的指针 + 字符串长度**」，不管理内存，也不要求原字符串是 `std::string`（支持 C 风格字符串、字面量、字符数组）：

```cpp
void func(std::string_view sv) {
    // 仅能读取，无法修改原字符串
    std::cout << sv << std::endl;
}

// 支持多种字符串类型，均无拷贝开销（O(1)）
std::string s = "hello";
const char* c_str = "world";
func(s);        // string → string_view（隐式转换）
func(c_str);    // C 风格字符串 → string_view
func("test");   // 字符串字面量 → string_view
```

- 核心特点：**极致轻量**——仅占 16 字节（64 位系统：8 字节指针 + 8 字节长度），无堆分配，支持所有字符串类型的只读访问。

### 2. 性能对比：拷贝/构造开销（面试核心）

| 操作场景           | std::string  | const string&                     | string_view    |
| -------------- | ------------ | --------------------------------- | -------------- |
| 函数参数传递（长字符串）   | O(n)（深拷贝）    | O(1)（仅引用）                         | O(1)（指针+长度）    |
| 接收字符串字面量       | O(n)（构造临时对象） | O(n)（隐式构造临时对象）                    | O(1)（无构造）      |
| 子串截取（`substr`） | O(k)（深拷贝子串）  | O(k)（需先调用 `substr` 生成临时 `string`） | O(1)（仅调整指针+长度） |

**示例：子串截取性能对比**

```cpp
#include <string>
#include <string_view>
#include <iostream>

int main() {
    std::string s = "1234567890abcdefghijklmnopqrstuvwxyz";

    // string::substr：深拷贝，O(k) 开销
    std::string sub1 = s.substr(5, 10);  // 拷贝 10 个字符

    // string_view::substr：仅调整指针+长度，O(1) 开销
    std::string_view sv(s);
    std::string_view sub2 = sv.substr(5, 10);  // 无拷贝

    std::cout << sub1 << " " << sub2 << std::endl;  // 结果一致，性能差异巨大
    return 0;
}
```

### 3. 风险点：悬垂引用/视图失效（面试避坑）

#### （1）const string& 的风险：悬垂引用

若引用的 `string` 析构或被修改，引用会失效（未定义行为）：

```cpp
const std::string& bad_ref() {
    std::string temp = "hello";
    return temp;  // 错误：返回局部 string 的引用，函数结束后 temp 析构，引用悬垂
}
```

#### （2）string_view 的风险：视图失效

原字符串内存释放/修改，`string_view` 会指向无效内存（未定义行为）：

```cpp
std::string_view bad_view() {
    std::string temp = "hello";
    std::string_view sv = temp;
    return sv;  // 错误：temp 析构后，sv 指向的内存失效
}

// 另一个常见风险：修改原 string 导致视图失效
std::string s = "hello";
std::string_view sv = s;
s += " world";  // s 扩容（重新分配内存），sv 指向原失效内存
std::cout << sv;  // 未定义行为（输出乱码/崩溃）
```

#### （3）std::string 的优势：无失效风险

`string` 自身管理内存，只要对象存活，内存就有效，无悬垂风险。

## 三、适用场景（面试必答）

### 1. 优先用 std::string 的场景

- 需要修改字符串（如拼接、替换、插入字符）；
- 长期持有字符串（如类成员变量），避免依赖外部内存；
- 需兼容 C++17 之前的编译器。

### 2. 优先用 const string& 的场景

- C++17 之前的项目，函数参数需只读访问 `std::string`；
- 函数参数仅接收 `std::string` 类型，且无需兼容 C 风格字符串/字面量；
- 需避免 `string` 传参的深拷贝（短期只读访问）。

### 3. 优先用 string_view 的场景（C++17+）

- 函数参数：只读访问任意字符串类型（`string`/C 风格/字面量），避免拷贝；
- 高频只读操作：如字符串比较、子串截取、解析（如 JSON/XML 解析）；
- 性能敏感场景：如高并发服务器的字符串处理、大量字符串遍历/匹配。

### 4. 绝对禁止用 string_view 的场景

- 需要修改字符串（`string_view` 无写接口）；
- 长期持有字符串（易因原内存失效导致问题）；
- 原字符串可能被修改/释放（如临时字符串、动态扩容的 `string`）。

## 四、面试易错点与避坑指南

### 1. 误区1：string_view 可以替代 const string&

- `string_view` 无内存管理，仅适合**短期只读访问**；
- `const string&` 虽仅支持 `string`，但语义更明确（引用一个合法的 `string` 对象），风险更低（只要原 `string` 存活，引用就有效）。

### 2. 误区2：string_view 是“更好的 const string&”

二者定位不同：

- `const string&` 是“对 `string` 的引用”，依赖 `string` 存在；
- `string_view` 是“对任意字符序列的视图”，不依赖具体类型，更轻量但风险更高。

### 3. 误区3：string_view 可以直接赋值给 string

`string_view` 可隐式转换为 `string`，但转换时会触发深拷贝（O(n) 开销），需避免频繁转换：

```cpp
std::string_view sv = "hello";
std::string s = sv;  // 深拷贝（O(5) 开销），与直接构造 string 无异
```

### 4. 误区4：忽略小字符串优化（SSO）

`std::string` 的小字符串优化（通常 <16 字节）会将字符存储在栈上，无堆分配开销，此时 `string` 的拷贝开销与 `string_view` 接近，无需强行替换为 `string_view`。

## 五、面试回答模板（简洁版）

`string`、`const string&`、`string_view` 的核心区别在于**内存管理和拷贝开销**：

1. `std::string` 是字符串的“所有者”，管理堆内存，支持修改，无失效风险但拷贝开销高（O(n)）；
2. `const string&` 是 `string` 的只读引用，拷贝开销低（O(1)），但仅支持 `string` 类型，且有悬垂引用风险；
3. `string_view`（C++17+）是任意字符串的只读视图，仅存指针+长度，拷贝/截取开销极致低（O(1)），支持所有字符串类型，但无内存管理，有视图失效风险。

适用场景：

- 修改/长期持有字符串用 `string`；
- C++17 前只读访问 `string` 用 `const string&`；
- C++17+ 高频只读、跨类型字符串访问用 `string_view`（仅短期使用）。

核心原则：**只读场景优先选轻量类型（const string&/string_view），修改/长期持有选 string**。
