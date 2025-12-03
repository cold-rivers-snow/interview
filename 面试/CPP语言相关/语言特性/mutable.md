### mutable

mutable的中文意思是“可变的，易变的”，跟constant（既C++中的const）是反义词。在C++中，mutable也是为了突破const的限制而设置的。被mutable修饰的变量，将永远处于可变的状态，即使在一个const函数中。我们知道，如果类的成员函数不会改变对象的状态，那么这个成员函数一般会声明成const的。但是，有些时候，我们需要**在const函数里面修改一些跟类状态无关的数据成员，那么这个函数就应该被mutable来修饰，并且放在函数后后面关键字位置**。

# mutable 关键字（核心用法+场景+与 const/volatile 关联）

`mutable` 是 C++ 中用于修饰**类成员变量**的关键字，核心作用是：**突破 `const` 成员函数的“只读限制”**——即使类对象被 `const` 修饰，或成员函数被声明为 `const`，`mutable` 成员变量仍可被修改。其本质是“允许类的部分状态（`mutable` 成员）在逻辑只读场景下被修改”。

## 一、核心语义与本质（面试必答）

### 1. 核心问题：const 成员函数的限制

C++ 中 `const` 成员函数的核心约定是：**不能修改类的“逻辑状态”**（即成员变量），编译器会强制检查——若 `const` 成员函数中尝试修改非 `mutable` 成员变量，直接编译报错。

但实际开发中，部分成员变量的修改**不会影响类的逻辑状态**（如缓存、计数、调试信息等），此时需要 `mutable` 打破限制：

### 2. mutable 的核心作用

`mutable` 修饰的成员变量，将被排除在 `const` 成员函数的“只读检查”之外，允许：

- `const` 成员函数修改该变量；
- `const` 修饰的类对象，其 `mutable` 成员仍可被修改（通过 `const` 成员函数或直接访问，若权限允许）。

### 3. 本质总结

`mutable` 是对 `const` 语义的“补充放松”——它区分了类的“逻辑只读”和“物理只读”：

- 逻辑只读：类对外提供的功能/状态不变（符合 `const` 约定）；
- 物理只读：所有成员变量均不可修改（默认 `const` 行为）；
  `mutable` 允许“逻辑只读但部分物理成员可修改”。

## 二、典型使用场景（面试高频）

### 1. 场景1：缓存数据（最常用）

类的 `const` 成员函数（如查询接口）可能需要计算复杂结果，为避免重复计算，可将结果缓存到 `mutable` 成员变量，后续直接复用。

**示例代码**：

```cpp
#include <iostream>
#include <string>
using namespace std;

class StringProcessor {
private:
    string raw_str;
    // 缓存：存储处理后的结果（修改不影响类的逻辑状态）
    mutable string processed_str;
    // 标记：缓存是否有效（修改不影响类的逻辑状态）
    mutable bool cache_valid = false;
public:
    StringProcessor(const string& s) : raw_str(s) {}

    // const 成员函数：逻辑上“只读”（返回处理结果，不改变原始字符串）
    string getProcessedStr() const {
        if (!cache_valid) {
            // 允许修改 mutable 成员（缓存计算结果）
            processed_str = "[" + raw_str + "]";  // 模拟复杂处理
            cache_valid = true;
        }
        return processed_str;
    }
};

int main() {
    const StringProcessor sp("hello");  // const 对象
    cout << sp.getProcessedStr() << endl;  // 首次计算，缓存生效
    cout << sp.getProcessedStr() << endl;  // 复用缓存，无需重复计算
    return 0;
}
```

- 关键：`getProcessedStr()` 是 `const` 成员函数（逻辑只读），但通过 `mutable` 成员实现了缓存优化，不违背 `const` 的语义约定。

### 2. 场景2：计数/统计（如访问次数）

类的 `const` 成员函数被调用时，需要统计访问次数，次数变量的修改不影响类的核心逻辑。

**示例代码**：

```cpp
class CounterDemo {
private:
    mutable int access_count = 0;  // 访问次数：mutable 修饰
    int data;
public:
    CounterDemo(int d) : data(d) {}

    int getData() const {
        access_count++;  // 允许修改 mutable 成员（统计访问次数）
        return data;
    }

    int getAccessCount() const {
        return access_count;
    }
};

int main() {
    const CounterDemo cd(100);
    cd.getData();
    cd.getData();
    cout << "访问次数：" << cd.getAccessCount() << endl;  // 输出 2
    return 0;
}
```

### 3. 场景3：线程同步锁（`mutex`）

`const` 成员函数可能需要线程安全，需通过 `mutable` 修饰 `std::mutex`（锁的加解锁属于“物理修改”，但不影响类的逻辑状态）。

**示例代码**：

```cpp
#include <mutex>
using namespace std;

class ThreadSafeData {
private:
    int data;
    mutable mutex mtx;  // 锁：mutable 修饰（加解锁需修改锁状态）
public:
    ThreadSafeData(int d) : data(d) {}

    // const 成员函数：逻辑只读（获取数据），但需加锁保证线程安全
    int getSafeData() const {
        lock_guard<mutex> lock(mtx);  // 允许修改 mutable 的 mutex
        return data;
    }
};
```

### 4. 场景4：调试信息/临时状态

存储调试用的日志、临时标记等，修改这些变量不影响类的核心功能。

## 三、关键规则与限制（面试避坑）

### 1. 只能修饰“类成员变量”

`mutable` 不能修饰：全局变量、局部变量、函数参数、`static` 成员变量（`static` 属于类级别的变量，与 `const` 对象无关）。

**错误示例**：

```cpp
// 错误1：修饰局部变量
void func() {
    mutable int x = 10;  // 编译报错：mutable 不能修饰局部变量
}

// 错误2：修饰 static 成员变量
class Test {
    static mutable int g_val;  // 编译报错：mutable 不能修饰 static 成员
};
```

### 2. 不影响 `volatile` 语义

`mutable` 仅与 `const` 交互，与 `volatile` 无直接冲突，可组合使用（但场景极少）：

```cpp
class Test {
    mutable volatile int x;  // 允许：x 可被 const 成员函数修改，且禁止编译器优化
};
```

### 3. 不能突破 `const` 对象的“顶层 const”限制

若类对象是 `const`，且 `mutable` 成员变量是指针/引用，`mutable` 仅允许修改指针/引用本身，不允许修改其指向的内容（除非指向的内容也有对应的放松限制）。

**示例代码**：

```cpp
class Test {
public:
    mutable int* p;  // mutable 指针
};

int main() {
    int a = 10, b = 20;
    const Test t;
    t.p = &a;  // 允许：修改 mutable 指针本身（指向 a）
    // *t.p = 30;  // 错误：t 是 const 对象，p 指向的内容是“逻辑只读”（需 p 是 mutable 且指向的内容无 const 限制）
    return 0;
}
```

### 4. 与 `const_cast` 的区别（面试高频对比）

`const_cast` 是“强制移除 const 限定”（语法层面的强转），`mutable` 是“语义层面的放松限制”，二者核心差异：
| 特性                | mutable                  | const_cast                |
|---------------------|--------------------------|---------------------------|
| 本质                | 语义约定，编译期检查      | 语法强转，运行期无检查    |
| 适用场景            | 类成员变量，长期复用      | 临时移除 const 限定（如调用非 const 接口） |
| 安全性              | 安全（符合语义约定）      | 风险高（可能导致未定义行为） |
| 作用范围            | 仅修饰的成员变量          | 任意 const 变量/对象      |

**示例对比**：

```cpp
class Demo {
private:
    int x;
    mutable int y;
public:
    void setX(int val) { x = val; }
    void setY(int val) const { y = val; }  // mutable 允许 const 函数修改 y
};

int main() {
    const Demo d;
    // d.setX(10);  // 错误：const 对象不能调用非 const 函数
    d.setY(20);  // 正确：mutable 成员允许修改

    // const_cast 强转（风险）
    const_cast<Demo&>(d).setX(30);  // 语法允许，但修改 const 对象的非 mutable 成员属于未定义行为
    return 0;
}
```

## 四、与 const/volatile 的关联总结

| 关键字      | 核心作用             | 交互关系                      |
| -------- | ---------------- | ------------------------- |
| const    | 限制当前代码显式修改       | mutable 可突破其对成员变量的修改限制    |
| volatile | 禁止编译器优化，强制内存访问   | 与 mutable 无冲突，可组合使用（场景极少） |
| mutable  | 放松 const 成员函数的限制 | 仅作用于类成员变量，不影响其他关键字语义      |

## 五、面试易错点提醒

1. **坑1：mutable 修饰 static 成员变量？（×）**  
   `static` 成员属于类，而非对象，`mutable` 仅用于对象的成员变量，二者冲突，编译报错。

2. **坑2：mutable 可以突破所有 const 限制？（×）**  
   仅突破 `const` 成员函数对成员变量的修改限制，不能突破 `const` 对象对“指向内容”的修改限制（如指针指向的内容）。

3. **坑3：mutable 与 volatile 是反义词？（×）**  
   二者语义无关：`mutable` 管“能不能修改”，`volatile` 管“要不要优化”，无直接关联。

4. **坑4：用 mutable 修饰核心业务成员？（×）**  
   `mutable` 仅适用于“非核心状态”（缓存、计数、锁等），若修饰核心业务成员（如用户姓名、订单金额），会破坏 `const` 语义的一致性，导致代码逻辑混乱。

## 六、面试回答模板（简洁版）

`mutable` 是 C++ 中修饰**类成员变量**的关键字，核心作用是突破 `const` 成员函数的只读限制——即使类对象是 `const` 或成员函数是 `const`，`mutable` 成员仍可被修改。

关键要点：

1. 本质：区分类的“逻辑只读”和“物理只读”，允许非核心状态（缓存、计数、锁）在逻辑只读场景下修改；
2. 适用场景：缓存数据、访问计数、线程锁、调试信息等；
3. 限制：只能修饰类成员变量，不能修饰局部变量/静态变量，不影响 `volatile` 语义；
4. 与 `const_cast` 区别：`mutable` 是安全的语义约定，`const_cast` 是风险高的语法强转。