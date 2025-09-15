---
title: Effective Modern C++ 笔记（4）- 智能指针
date: 2025-04-16 10:23:19
tags:
    - cpp
    - effective-modern-cpp
categories:
    - C++
cover: "/imgs/eff-modern-cpp-img.png"
excerpt: "Effective Modern C++ 第四章笔记 - 智能指针"
comment: false
---

## Item 18: Use std::unique_ptr for exclusive-ownership resource management
{% note success %} 

https://cntransgroup.github.io/EffectiveModernCppChinese/4.SmartPointers/item18.html

{% endnote %} 
我们来深入解读一下《Effective Modern C++》中这条关于 `std::unique_ptr` 的核心建议。这是现代 C++ 资源管理（RAII）的基石之一。

> **核心思想**：`std::unique_ptr` 是一种智能指针，它体现了**独占所有权（Exclusive Ownership）** 的语义。这意味着在任何时刻，只有一个 `std::unique_ptr` 可以拥有并负责管理一个特定的资源。当这个 `unique_ptr` 被销毁时（例如离开作用域），它所拥有的资源也会被自动释放。这种设计使得资源泄漏在实践中几乎不可能发生，同时由于其轻量级和只移特性，它的性能开销几乎为零。

让我们逐一分解这条建议的要点。

### 1. 独占所有权（Exclusive Ownership）与只移类型（Move-only）

这是 `std::unique_ptr` 最核心的特性。

*   **独占所有权**：想象一下房子的房产证。在任何时候，只有一个人或一个实体拥有这张证。`std::unique_ptr` 就是它所管理资源的“房产证”。
*   **如何实现独占？**：通过**禁止拷贝，只允许移动**。
    *   **拷贝被禁止**：你不能“复印”房产证，制造出两个所有者。同样，`std::unique_ptr` 的拷贝构造函数和拷贝赋值运算符都被删除了 (`= delete`)。

    ```cpp
    std::unique_ptr<Widget> p1 = std::make_unique<Widget>();
    // std::unique_ptr<Widget> p2 = p1; // 编译错误！不能拷贝。
    ```
    *   **移动被允许**：你可以将房产证**转让**给另一个人。`std::unique_ptr` 支持移动构造和移动赋值。当移动发生时，源 `unique_ptr` 会放弃所有权（变为空指针），目标 `unique_ptr` 接管所有权。

    ```cpp
    std::unique_ptr<Widget> p1 = std::make_unique<Widget>();
    std::unique_ptr<Widget> p2 = std::move(p1); // OK！所有权从 p1 转移到 p2。

    // 此时，p1 变成了 nullptr，不再拥有 Widget 对象。
    // p2 现在是 Widget 对象的唯一所有者。
    if (p1 == nullptr) {
        std::cout << "p1 is now empty." << std::endl;
    }
    ```
这种“只移”特性使得所有权的转移非常明确和安全。它常被用作工厂函数的返回类型，因为函数返回一个局部创建的对象的所有权给调用者是一种非常清晰的转移。

```cpp
std::unique_ptr<Widget> create_widget() {
    return std::make_unique<Widget>(); // 高效地返回所有权
}

auto my_widget = create_widget(); // my_widget 接管了新创建的 Widget
```

### 2. 轻量级与快速

“智能指针”这个名字可能会让人觉得它很“重”，有很多开销。但 `std::unique_ptr` 恰恰相反。

*   **零开销抽象**：在大多数情况下（特别是使用默认删除器时），一个 `std::unique_ptr<T>` 的大小与一个裸指针 `T*` 完全相同。它本质上只是对裸指针的一层封装，添加了在析构时自动调用 `delete` 的行为。
*   **性能**：访问 `unique_ptr` 所指向的对象（通过 `*` 或 `->`）与通过裸指针访问一样快。编译器可以轻松地优化掉这层封装。

因此，你可以毫无性能顾虑地用 `std::unique_ptr` 来替代几乎所有拥有独占所有权的裸指针。这也是 `std::auto_ptr`（C++98 的老旧智能指针）被废弃的主要原因之一，因为 `auto_ptr` 有一些不直观的“拷贝即移动”行为和性能问题。

### 3. 自定义删除器（Custom Deleters）

`std::unique_ptr` 的设计非常灵活。默认情况下，它用 `delete` 来释放资源，这适用于通过 `new` 创建的单个对象。但有时我们需要用不同的方式来释放资源。

*   **场景1：`new[]` 创建的数组**
    对于用 `new[]` 创建的数组，必须用 `delete[]` 来释放。`std::unique_ptr` 对此有内置支持。

    ```cpp
    // 使用 std::unique_ptr<T[]> 版本
    std::unique_ptr<int[]> p(new int[10]); 
    // 当 p 离开作用域时，它会自动调用 delete[]
    ```
*   **场景2：C 风格的 API**
    很多 C 语言库的资源释放函数不是 `delete`，而是一些自定义函数，如 `fclose()` 用于文件指针，`free()` 用于 `malloc` 分配的内存。

    ```cpp
    // 一个自定义删除器，用于关闭文件
    auto file_closer = [](FILE* fp) {
        if (fp) {
            fclose(fp);
            std::cout << "File closed.\n";
        }
    };
    
    FILE* f = fopen("test.txt", "w");
    // 在 unique_ptr 的模板参数中指定删除器的类型
    std::unique_ptr<FILE, decltype(file_closer)> pFile(f, file_closer);
    
    // 当 pFile 离开作用域时，file_closer 会被自动调用
    ```
**自定义删除器的开销问题**：

*   **无状态删除器（Stateless Deleters）**：如果你的删除器是一个不捕获任何状态的 lambda（如上例），或者是一个函数对象（functor）的空类，编译器可以进行**空基类优化（Empty Base Optimization, EBO）**。这种情况下，删除器不会增加 `std::unique_ptr` 的大小。`sizeof(unique_ptr)` 仍然等于 `sizeof(T*)`。

*   **有状态删除器（Stateful Deleters）**：如果删除器需要存储数据（比如捕获了局部变量的 lambda），或者是一个函数指针，那么这些状态或指针本身需要占用空间。此时，`std::unique_ptr` 的大小会增加，通常是 `sizeof(T*) + sizeof(deleter)`。

    ```cpp
    // 使用函数指针作为删除器
    using FileCloser = void (*)(FILE*);
    std::unique_ptr<FILE, FileCloser> pFile(fopen("test.txt", "w"), fclose);

    // sizeof(pFile) == sizeof(FILE*) + sizeof(void(*)(FILE*))
    // 这通常比使用无状态 lambda 的版本要大。
    ```
因此，**优先使用无状态的 lambda 或函数对象作为自定义删除器**，以保持 `unique_ptr` 的轻量级特性。

### 4. 轻松转换为 `std::shared_ptr`

`std::unique_ptr` 体现的是独占所有权，而 `std::shared_ptr` 体现的是共享所有权。有时，一个资源开始时可能是独占的，但后来需要被共享。`std::unique_ptr` 的设计使得这种所有权模型的转换非常容易。

`std::shared_ptr` 有一个构造函数，可以从一个 `std::unique_ptr` 的右值“窃取”所有权。

```cpp
auto p_unique = std::make_unique<Widget>();

// ... 一段时间内，p_unique 是唯一所有者 ...

// 现在需要共享这个 Widget，将其所有权转移给一个 shared_ptr
std::shared_ptr<Widget> p_shared = std::move(p_unique); 

// 转换完成！
// p_unique 现在是 nullptr。
// p_shared 成为了 Widget 的第一个所有者，其引用计数为 1。
```
这种单向转换（从独占到共享）是安全且符合逻辑的。反过来则不行，你不能把一个可能已经被多人共享的 `shared_ptr` 转换为一个 `unique_ptr`。

**为什么这个特性很重要？**

它提供了一个非常灵活的设计模式：**工厂函数默认返回 `std::unique_ptr`**。

*   返回 `unique_ptr` 表达了最清晰、最低开销的所有权转移。
*   调用者接收到这个 `unique_ptr` 后，如果他只需要独占使用，那就直接用，零开销。
*   如果他后续需要将这个资源分享给其他人，他可以随时、轻松地将其转换为 `shared_ptr`。

这让调用者拥有了选择权，避免了工厂函数过早地决定资源应该是共享的（`shared_ptr` 总是有引用计数的开销）。

### 总结

1.  **默认选择**：当你需要在堆上分配资源并管理其生命周期时，`std::unique_ptr` 应该是你的**默认选择**。
2.  **语义清晰**：它完美地表达了“独占所有权”这一最常见的资源管理模式。
3.  **高性能**：它是一个零开销的抽象，不会带来额外的性能负担。
4.  **灵活性**：通过自定义删除器，它可以管理任何类型的资源，不仅仅是通过 `new` 创建的内存。
5.  **可升级**：当需要从独占所有权转向共享所有权时，它可以无缝、高效地转换为 `std::shared_ptr`。

遵循这条规则，可以让你编写出几乎没有资源泄漏风险、同时又保持高性能的 C++ 代码。


---
## Item 19: Use std::shared_ptr for shared-ownership resource management
{% note success %} 

https://cntransgroup.github.io/EffectiveModernCppChinese/4.SmartPointers/item19.html

{% endnote %} 
我们来深入解读一下《Effective Modern C++》中关于 `std::shared_ptr` 的这条核心建议。这是现代 C++ 资源管理中与 `std::unique_ptr` 并驾齐驱的另一个重要工具。

> **核心思想**：`std::shared_ptr` 是一种智能指针，它实现了**共享所有权（Shared Ownership）** 的语义。这意味着多个 `std::shared_ptr` 实例可以共同拥有并管理同一个资源。它通过一个**引用计数**机制来跟踪有多少个“所有者”存在。只有当最后一个 `shared_ptr` 被销毁时，它所管理的资源才会被自动释放。这种机制类似于许多高级语言中的垃圾回收，但行为更确定、更可控。

让我们逐一分解这条建议的要点。

### 1. 共享所有权与自动垃圾回收

这是 `std::shared_ptr` 最核心的功能。

*   **共享所有权**：想象一下图书馆的一本热门书。多个读者可以同时“借阅”它（当然这里比喻为同时拥有它的引用）。图书馆需要知道这本书被多少人借出去了。`std::shared_ptr` 就是这本书的一个“借阅凭证”，而它所管理的资源就是那本书。
*   **引用计数（Reference Count）**：每个被 `shared_ptr` 管理的资源，都会附带一个“控制块”（Control Block），其中最重要的部分就是引用计数。
    *   当一个新的 `shared_ptr` 通过拷贝构造或拷贝赋值指向这个资源时，引用计数 `+1`。
    *   当一个 `shared_ptr` 被销毁（离开作用域）或指向其他资源时，引用计数 `-1`。
    *   当引用计数减到 `0` 时，意味着没有任何 `shared_ptr` 再拥有这个资源了，此时资源会被自动销毁。

**示例：**

```cpp
void process_widget(std::shared_ptr<Widget> spw) {
    std::cout << "In func, use_count: " << spw.use_count() << std::endl;
    // ... 使用 spw ...
} // spw 离开作用域，引用计数 -1

int main() {
    auto sp1 = std::make_shared<Widget>(); // 创建 Widget，引用计数为 1
    std::cout << "After sp1, use_count: " << sp1.use_count() << std::endl;

    {
        std::shared_ptr<Widget> sp2 = sp1; // 拷贝构造，引用计数变为 2
        std::cout << "After sp2, use_count: " << sp1.use_count() << std::endl;
        
        process_widget(sp1); // 传参是拷贝，函数内部引用计数会短暂变为 3
        std::cout << "After func, use_count: " << sp1.use_count() << std::endl;
    } // sp2 离开作用域，引用计数变回 1

    std::cout << "After block, use_count: " << sp1.use_count() << std::endl;
} // main 结束，sp1 离开作用域，引用计数变为 0，Widget 对象被销毁
```
这种自动管理机制，极大地简化了在复杂对象关系（如图、树等）中共享资源的生命周期管理，有效防止了内存泄漏和悬空指针问题。

### 2. `std::shared_ptr` 的开销

共享所有权并非没有代价。`std::shared_ptr` 比 `std::unique_ptr` “重”得多。

*   **大小**：一个 `std::shared_ptr` 对象的大小通常是裸指针的两倍。它内部包含两个指针：
    1.  一个指向被管理的资源（如 `Widget*`）。
    2.  一个指向**控制块（Control Block）**。

*   **控制块的开销**：
    *   **动态分配**：控制块本身通常是在堆上动态分配的。（`std::make_shared` 可以优化这一点，将资源和控制块一次性分配）。
    *   **内容**：控制块中至少包含：
        *   **强引用计数（Strong Reference Count）**：即我们通常说的 `use_count()`。
        *   **弱引用计数（Weak Reference Count）**：用于 `std::weak_ptr`，解决循环引用问题。
        *   **自定义删除器（Custom Deleter）**（如果提供了）。
        *   **自定义分配器（Custom Allocator）**（如果提供了）。

*   **原子性引用计数修改**：
    `std::shared_ptr` 必须是线程安全的，这意味着多个线程可以同时拷贝、销毁指向同一个资源的 `shared_ptr`。为了保证在多线程环境下引用计数的增减是正确的，这些操作必须是**原子操作**。原子操作通常比普通的整数增减要慢，尤其是在多核高竞争环境下。

**结论**：`shared_ptr` 带来了便利，但也带来了实实在在的性能开销。因此，**不应该滥用 `shared_ptr`**。如果一个资源只需要独占所有权，**请务必使用 `std::unique_ptr`**。只在确实需要共享所有权时才使用 `std::shared_ptr`。

### 3. 自定义删除器

与 `std::unique_ptr` 类似，`std::shared_ptr` 也支持自定义删除器，可以管理任意类型的资源。

```cpp
auto file_closer = [](FILE* fp) { if(fp) fclose(fp); };

FILE* f = fopen("test.txt", "w");
std::shared_ptr<FILE> pFile(f, file_closer); 
// 当最后一个 pFile 的拷贝被销毁时，file_closer 会被调用
```

但与 `unique_ptr` 有一个**关键区别**：

*   **删除器的类型不影响 `shared_ptr` 的类型**。
    *   对于 `std::unique_ptr<T, Deleter>`，删除器的类型是 `unique_ptr` 类型的一部分。这意味着 `std::unique_ptr<T, D1>` 和 `std::unique_ptr<T, D2>` 是两种不同的类型。
    *   对于 `std::shared_ptr<T>`，删除器（以及其类型）被存储在**类型擦除（type-erased）**的控制块中。因此，`std::shared_ptr<FILE>` 的类型永远就是 `std::shared_ptr<FILE>`，无论你用什么删除器。

这使得 `std::shared_ptr` 在使用上更加灵活。例如，你可以将一个用 lambda 做删除器的 `shared_ptr` 和一个用函数指针做删除器的 `shared_ptr` 放在同一个容器 `std::vector<std::shared_ptr<FILE>>` 中。而对于 `unique_ptr`，这是不可能的。

### 4. 避免从原始指针变量上创建 `std::shared_ptr`

这是使用 `shared_ptr` 时最重要、最危险的一个陷阱。

**规则：不要用同一个裸指针去初始化多个独立的 `std::shared_ptr`。**

**错误的示例：**

```cpp
Widget* raw_ptr = new Widget();

std::shared_ptr<Widget> sp1(raw_ptr); // OK, sp1 创建了一个控制块，引用计数为 1。

// 灾难性的错误！
std::shared_ptr<Widget> sp2(raw_ptr); // sp2 也用 raw_ptr 初始化，
                                      // 它不知道 sp1 的存在，
                                      // 于是又创建了第二个独立的控制块！
                                      // 此时，两个控制块的引用计数都是 1。
```

**后果：**
1.  当 `sp1` 离开作用域时，它看到自己的引用计数变为 0，于是它调用 `delete raw_ptr`，释放了 `Widget` 对象。
2.  当 `sp2` 离开作用域时，它也看到自己的引用计数变为 0，于是它**再次**调用 `delete raw_ptr`。
3.  对同一个指针 `delete` 两次，这是**未定义行为**，通常会导致程序崩溃。

**正确的做法是什么？**

1.  **首选 `std::make_shared`**：
    这是创建 `shared_ptr` 的最佳方式。它将资源对象的分配和控制块的分配合并为一次内存分配，效率更高，并且从根本上避免了上述问题。

    ```cpp
    auto sp1 = std::make_shared<Widget>(); // 安全，高效
    auto sp2 = sp1; // 从已有的 shared_ptr 拷贝，正确共享同一个控制块
    ```

2.  **如果必须从裸指针创建，请立即进行**：
    创建完裸指针后，应立即将其封装在 `shared_ptr` 中，并且之后绝不再使用这个裸指针。

    ```cpp
    Widget* raw_ptr = new Widget();
    std::shared_ptr<Widget> sp1(raw_ptr);
    raw_ptr = nullptr; // 最好将裸指针置空，防止误用
    
    std::shared_ptr<Widget> sp2 = sp1; // 总是从已有的 shared_ptr 创建新的
    ```
一个更隐蔽的陷阱是在 `this` 指针上犯错，这需要通过 `std::enable_shared_from_this` 来解决，这是另一个相关但更高级的话题。

### 总结

1.  **适用场景**：当你需要多个指针或对象共同管理一个资源的生命周期时，使用 `std::shared_ptr`。
2.  **工作原理**：通过线程安全的引用计数实现自动资源管理。
3.  **注意开销**：`shared_ptr` 在内存和性能上都有开销，不要滥用。如果独占所有权就足够，请坚持使用 `std::unique_ptr`。
4.  **创建方式**：**始终优先使用 `std::make_shared`**。这不仅更高效，而且能从根本上避免从裸指针重复创建 `shared_ptr` 导致的严重 bug。
5.  **灵活性**：自定义删除器的类型不会影响 `shared_ptr` 本身的类型，使其在某些泛型场景下比 `unique_ptr` 更灵活。


---
## Item 20: Use std::weak_ptr for std::shared_ptr-like pointers that can dangle
{% note success %} 

https://cntransgroup.github.io/EffectiveModernCppChinese/4.SmartPointers/item20.html

{% endnote %} 
我们来深入解读一下《Effective Modern C++》中关于 `std::weak_ptr` 的这条建议。`std::weak_ptr` 是与 `std::shared_ptr` 紧密配合的一种智能指针，专门用来解决 `shared_ptr` 自身无法处理的一些棘手问题。

> **核心思想**：`std::weak_ptr` 是一种**非拥有（non-owning）**的智能指针，它像一个“观察者”或“弱引用”，指向由 `std::shared_ptr` 管理的资源，但**不会增加资源的引用计数**。它的核心作用是让你可以在不影响资源生命周期的前提下，安全地观察一个资源。你可以在需要时，尝试将 `weak_ptr` “提升”为一个有效的 `shared_ptr`，如果资源仍然存在，提升成功；如果资源已被销毁，提升失败。这使得 `weak_ptr` 成为解决 `shared_ptr` 循环引用和实现缓存等模式的完美工具。

我们逐一分解这条建议的要点。

### 1. `std::weak_ptr` 是什么？它如何工作？

*   **不拥有资源**：一个 `weak_ptr` 只是一个“旁观者”。它的创建和销毁**不会**改变它所指向资源的强引用计数（strong reference count）。
*   **知道资源是否存在**：`weak_ptr` 共享 `shared_ptr` 的**控制块**。通过检查控制块中的强引用计数，`weak_ptr` 能够知道它指向的资源是否还存活。
*   **不能直接访问资源**：你不能像 `shared_ptr` 或裸指针那样，直接用 `*` 或 `->` 来解引用一个 `weak_ptr`。因为在你检查完资源存在性和实际访问它之间，资源可能就被另一个线程销毁了，这会导致悬空指针。
*   **提升（Locking）**：访问资源的唯一安全方式是调用 `weak_ptr::lock()` 方法。这个方法会：
    1.  原子性地检查控制块中的强引用计数是否大于 0。
    2.  如果大于 0，意味着资源还活着，`lock()` 会创建一个新的 `shared_ptr` 指向该资源（并使引用计数+1），然后返回这个 `shared_ptr`。
    3.  如果等于 0，意味着资源已被销毁，`lock()` 会返回一个空的 `shared_ptr`（`nullptr`）。

**示例：**
```cpp
auto sp = std::make_shared<Widget>(); // 强引用计数 = 1
std::weak_ptr<Widget> wp(sp);         // 强引用计数仍为 1，弱引用计数 = 1

// 尝试从 weak_ptr 获取一个 shared_ptr
if (std::shared_ptr<Widget> sp_locked = wp.lock()) { 
    // 提升成功！资源还活着。
    // 在这个 if 块内，sp_locked 保证了 Widget 对象不会被销毁。
    // 此时强引用计数至少为 2 (sp 和 sp_locked)
    sp_locked->doSomething();
} else {
    // 提升失败！资源已经被销毁了。
    std::cout << "Widget is gone." << std::endl;
}

sp.reset(); // 销毁 sp，强引用计数变为 0，Widget 对象被销毁

if (std::shared_ptr<Widget> sp_locked = wp.lock()) {
    // 这次提升会失败
} else {
    std::cout << "Widget is now gone." << std::endl;
}
```
`wp.lock()` 的这种“检查并获取”的原子操作，是 `weak_ptr` 安全性的核心。

### 2. 使用场景 1：打破 `std::shared_ptr` 的循环引用

这是 `weak_ptr` 最经典、最重要的用途。当两个或多个对象通过 `shared_ptr` 相互引用，形成一个闭环时，它们的引用计数永远不会降到 0，从而导致资源泄漏。

**一个典型的循环引用示例：**

```cpp
class Child;

class Parent {
public:
    std::shared_ptr<Child> child;
    ~Parent() { std::cout << "Parent destroyed\n"; }
};

class Child {
public:
    std::shared_ptr<Parent> parent; // 糟糕的设计！
    ~Child() { std::cout << "Child destroyed\n"; }
};

void create_cycle() {
    auto p = std::make_shared<Parent>(); // p 的强引用计数 = 1
    auto c = std::make_shared<Child>();  // c 的强引用计数 = 1

    p->child = c; // c 的强引用计数变为 2 (c 和 p->child)
    c->parent = p; // p 的强引用计数变为 2 (p 和 c->parent)

    std::cout << "p use_count: " << p.use_count() << std::endl; // 输出 2
    std::cout << "c use_count: " << c.use_count() << std::endl; // 输出 2
} // create_cycle 结束

// 局部变量 p 和 c 被销毁，它们各自指向的对象的强引用计数分别从 2 减到 1。
// p 指向的 Parent 对象，其引用计数为 1 (被 c->parent 指着)。
// c 指向的 Child 对象，其引用计数为 1 (被 p->child 指着)。
// 由于引用计数都不是 0，析构函数永远不会被调用，造成内存泄漏！
```
**解决方案：使用 `std::weak_ptr` 打破循环**

通常，我们需要分析对象之间的所有权关系。在这个例子中，可以说 `Parent` “拥有” `Child`，而 `Child` 只是“知道”或“回指”其 `Parent`。因此，从 `Child` 到 `Parent` 的链接应该是弱引用。

```cpp
class Child {
public:
    std::weak_ptr<Parent> parent; // 改为 weak_ptr！
    ~Child() { std::cout << "Child destroyed\n"; }
};
```
现在再看 `create_cycle` 结束时：
1.  局部变量 `p` 销毁，`Parent` 对象的强引用计数从 2 减到 1 (因为 `c->parent` 是 `weak_ptr`，不计入强引用)。等等，`c->parent` 是 `weak_ptr`，所以 `p` 的强引用计数一直是1，现在销毁，`Parent` 对象的强引用计数从 1 减到 0。`Parent` 对象被销毁。
2.  `Parent` 的析构函数执行，其成员 `p->child`（一个 `shared_ptr`）被销毁。
3.  这导致 `Child` 对象的强引用计数从 1 减到 0。`Child` 对象被销毁。
4.  内存泄漏问题解决！

当 `Child` 需要访问它的 `Parent` 时，它可以使用 `parent.lock()` 来安全地获取一个临时的 `shared_ptr`。

### 3. 使用场景 2：缓存（Caching）

假设你有一个昂贵的资源（比如从数据库加载的对象），你希望把它缓存起来。

*   你不能只用 `shared_ptr` 来做缓存，因为只要缓存在，资源就永远不会被释放，即使程序其他地方已经不再需要它了。这会导致缓存无限增长。
*   你需要的是一种“当有人在用时，我才缓存；当没人用时，缓存应该自动失效”的机制。

`std::weak_ptr` 正是为此而生。缓存系统可以持有一个资源的 `weak_ptr`。

```cpp
// 伪代码
std::map<ResourceID, std::weak_ptr<ExpensiveResource>> cache;

std::shared_ptr<ExpensiveResource> load_resource(ResourceID id) {
    // 1. 尝试从缓存中获取
    if (cache.count(id)) {
        if (auto sp = cache[id].lock()) { // 尝试提升 weak_ptr
            return sp; // 提升成功，资源还在，直接返回
        }
    }
    
    // 2. 提升失败或缓存中没有，从数据库加载
    auto sp = load_from_database(id);
    
    // 3. 将新加载的资源放入缓存（存入 weak_ptr）
    cache[id] = sp;
    
    return sp;
}
```
这个设计非常优雅：
*   当一个 `ExpensiveResource` 正在被程序的某个部分使用时（至少有一个 `shared_ptr` 指向它），缓存中的 `weak_ptr` 就能成功 `lock()`，实现快速获取。
*   当所有使用者都释放了对 `ExpensiveResource` 的 `shared_ptr` 后，它的强引用计数变为 0，对象被销毁。此时，缓存中的 `weak_ptr` 会变成悬空状态，下一次 `lock()` 就会失败，系统会重新从数据库加载。缓存自动地清除了不再被使用的条目。

### 4. 使用场景 3：观察者列表（Observer List）

在观察者模式中，一个“主题”（Subject）维护一个“观察者”（Observers）列表。当主题状态改变时，它会通知所有观察者。

如果主题通过 `std::vector<std::shared_ptr<Observer>>` 来持有观察者，就会有一个问题：只要主题存在，所有注册的观察者都不能被销-毁。如果一个观察者想在生命周期结束时自动从主题中注销，实现起来会很复杂。

使用 `std::vector<std::weak_ptr<Observer>>` 就简单多了：
*   主题持有一个观察者列表，但不拥有它们。
*   当一个观察者对象被销毁时，它在主题列表中的 `weak_ptr` 会自动失效。
*   当主题需要通知时，它遍历列表，对每个 `weak_ptr` 调用 `lock()`。如果 `lock()` 成功，就调用观察者的通知方法；如果失败，就知道这个观察者已经不存在了，可以顺便从列表中移除这个无效的 `weak_ptr`。

### 总结

1.  **`weak_ptr` 是 `shared_ptr` 的辅助工具**，它不拥有资源，也不影响资源的生命周期。
2.  **核心功能是 `lock()`**，它提供了一种安全地、原子性地检查资源是否存在并临时获取其所有权的方法。
3.  **主要用途**：
    *   **打破循环引用**：在对象间存在双向链接时，将其中一个方向（通常是“子”到“父”或“被拥有者”到“拥有者”）的链接设为 `weak_ptr`。
    *   **实现缓存**：允许缓存对象，但当对象不再被外部使用时，缓存能自动失效。
    *   **管理观察者列表**：允许观察者在生命周期结束时自动“注销”，而无需显式调用。

把 `weak_ptr` 看作是一个可以检查是否“悬空”的指针，你就能很好地理解它的设计意图和使用场景。它是现代 C++ 中构建健壮、无泄漏的复杂对象模型的关键一环。

---
## Item 21: Prefer std::make_unique and std::make_shared to direct use of new
{% note success %} 

https://cntransgroup.github.io/EffectiveModernCppChinese/4.SmartPointers/item21.html

{% endnote %} 
我们来深入解读一下《Effective Modern C++》中这条极其重要的建议。`std::make_unique` 和 `std::make_shared` 是现代 C++ 中创建智能指针的首选方式，理解它们的优势和局限性对于编写高质量的 C++ 代码至关重要。

> **核心思想**：`std::make_*` 函数（如 `std::make_unique` 和 `std::make_shared`）是创建智能指针的工厂函数。它们通过将**资源分配**（`new`）和**智能指针构造**封装在一个函数调用中，极大地提高了代码的简洁性、异常安全性和（对于 `make_shared`）性能。因此，除非有特殊情况，否则应始终优先使用它们，而不是手动 `new` 然后再构造智能指针。

我们来逐一分解这条建议的要点。

### 1. `make` 函数的共同优势

#### a. 消除代码重复（简洁性）

不使用 `make` 函数时，你需要写两次类型名：
```cpp
// 手动 new
std::unique_ptr<Widget> p1(new Widget());
std::shared_ptr<Widget> p2(new Widget("hello", 123));
```
使用 `make` 函数，代码更简洁，类型只出现一次：
```cpp
// 使用 make 函数
auto p1 = std::make_unique<Widget>();
auto p2 = std::make_shared<Widget>("hello", 123); // 参数完美转发给 Widget 的构造函数
```
这不仅减少了打字量，也降低了因类型不匹配而出错的风险，尤其是在使用 `auto` 时，代码非常干净。

#### b. 提高异常安全性（关键！）

这是使用 `make` 函数的一个非常重要的技术原因。考虑下面这个函数调用：
```cpp
// 一个函数，优先级高
void process_widget(std::shared_ptr<Widget> spw, int priority); 
```
现在，我们这样调用它：
```cpp
process_widget(std::shared_ptr<Widget>(new Widget), compute_priority());
```
这段代码存在一个**微妙的资源泄漏风险**。C++ 编译器在处理函数参数时，有权对表达式的求值顺序进行调整。一个可能的执行顺序是：
1.  执行 `new Widget()`，分配 `Widget` 对象的内存。
2.  执行 `compute_priority()`。
3.  执行 `std::shared_ptr` 的构造函数，接管 `new Widget()` 返回的指针。

**问题出在哪里？**
如果在第 2 步 `compute_priority()` 执行时**抛出了一个异常**，那么第 3 步 `shared_ptr` 的构造函数就永远不会被调用。这意味着在第 1 步中分配的 `Widget` 对象的内存将**永远不会被释放**，从而造成了资源泄漏。

**`make` 函数如何解决这个问题？**
```cpp
process_widget(std::make_shared<Widget>(), compute_priority());
```
`std::make_shared<Widget>()` 是一个**单一的函数调用**。编译器要么完整地执行它（在内部完成 `new` 和 `shared_ptr` 的构造），要么不执行。它不能像上面那样将 `make_shared` 的内部操作拆开，并在中间插入另一个函数的调用。因此，`new` 操作和智能指针的构造变成了一个不可分割的单元，`compute_priority()` 抛出异常的风险被完全规避。即使 `compute_priority()` 先于 `make_shared` 执行并抛出异常，`make_shared` 也根本不会被调用，自然也就没有资源泄漏。

这个异常安全性的提升是使用 `make` 函数的一个决定性优势。

### 2. `std::make_shared` 的额外优势：性能

`std::make_shared` 比手动 `new` 再构造 `shared_ptr` 的方式在性能上更有优势。

*   **手动 `new`**：需要**两次**内存分配。
    1.  `new Widget()`：为 `Widget` 对象本身分配内存。
    2.  `std::shared_ptr` 的构造函数：为**控制块**（包含引用计数等）分配内存。

*   **`std::make_shared`**：只需要**一次**内存分配。
    `std::make_shared` 会一次性地从堆上分配一块足够大的内存，同时容纳 `Widget` 对象和控制块。

这种单次分配带来了几个好处：
*   **速度更快**：减少了内存分配的次数。内存分配是相对昂贵的操作。
*   **代码更小**：减少了内存分配相关的代码量。
*   **更好的缓存局部性**：由于对象和它的控制块在内存中是相邻的，这有助于提高 CPU 缓存的命中率。

### 3. 不适合使用 `make` 函数的情况

尽管 `make` 函数非常优秀，但它们并非万能。在某些特定场景下，我们仍然需要直接使用 `new`。

#### a. 需要指定自定义删除器

`make` 函数不提供指定自定义删除器的接口。它们总是使用默认的删除器（`delete` 用于 `unique_ptr`，`delete` 和 `delete[]` 用于 `shared_ptr`）。如果你需要管理一个需要特殊方式释放的资源（比如用 `fclose` 的文件指针），你必须手动构造智能指针并传入自定义删除器。

```cpp
auto file_closer = [](FILE* fp) { if (fp) fclose(fp); };
std::unique_ptr<FILE, decltype(file_closer)> pFile(fopen("test.txt", "w"), file_closer);
// 这里不能用 make_unique
```

#### b. 希望使用花括号 `{}` 初始化

`make` 函数会将其参数**完美转发**给被构造对象的构造函数，这意味着它总是使用**圆括号 `()`** 的初始化语法。如果你想用 C++11 的**花括号 `{}`** 初始化（列表初始化），`make` 函数做不到。

```cpp
auto p = std::make_unique<std::vector<int>>(10, 20); // 构造一个含 10 个 20 的 vector
// auto p = std::make_unique<std::vector<int>>{10, 20}; // 语法错误！

// 如果你想构造一个含 {10, 20} 两个元素的 vector，必须：
std::unique_ptr<std::vector<int>> p(new std::vector<int>{10, 20}); 
// 或者 C++14 的一个变通方法
auto init_list = {10, 20};
auto p = std::make_unique<std::vector<int>>(init_list);
```
对于需要 `std::initializer_list` 构造函数的场景，直接使用 `new` 配合 `{}` 可能更直接。

### 4. `std::make_shared` 的额外不适用场景

以下几点是专门针对 `std::make_shared` 的，与 `make_unique` 无关。

#### a. 有自定义内存管理的类

如果一个类重载了 `operator new` 和 `operator delete`，它通常是希望对自己的内存分配和释放有精确的控制。`std::make_shared` 会使用全局的内存分配函数，而不是类的自定义 `operator new`，这可能会违背类的设计意图。

#### b. 内存敏感系统中的 `weak_ptr` 长寿问题

这是 `make_shared` 单次分配所带来的一个微妙的副作用。

*   `make_shared` 将对象和控制块放在**同一块内存**中。
*   这块内存只有在**对象被销毁**（强引用计数为 0）**并且控制块也不再被需要**（弱引用计数也为 0）时，才能被完全释放。

**问题场景**：
1.  你有一个非常大的对象 `BigObject`。
2.  你用 `std::make_shared<BigObject>()` 创建了它。
3.  程序中所有的 `shared_ptr` 都被销毁了，`BigObject` 的析构函数被调用，强引用计数变为 0。
4.  但是，还有一些 `weak_ptr` 指向这个（已经被析构的）`BigObject`。这些 `weak_ptr` 仍然需要访问控制块来判断对象是否存活（通过 `lock()`），所以弱引用计数不为 0。

**后果**：
由于 `BigObject` 和控制块在同一块内存中，即使 `BigObject` 对象本身已经被析构，它所占用的那部分**巨大内存也无法被释放**，直到最后一个 `weak_ptr` 被销毁。

相比之下，如果使用手动 `new`：
*   `BigObject` 和控制块是分两次分配的。
*   当强引用计数变为 0 时，`BigObject` 的内存被**立即释放**。
*   控制块的内存会等到弱引用计数也变为 0 时才被释放。

**结论**：在内存极其宝贵，且存在“大对象、`weak_ptr` 存活时间远长于 `shared_ptr`”的场景下，手动 `new` 再构造 `shared_ptr` 可能是更好的选择，因为它可以更快地回收大对象的内存。

### 总结

| | 优先使用 `make` 函数 | 何时直接用 `new` |
| :--- | :--- | :--- |
| **通用原因** | 1. **代码更简洁**，避免重复。<br>2. **异常安全**，防止资源泄漏。 | 1. 需要**自定义删除器**。<br>2. 需要使用**花括号 `{}` 初始化**。 |
| **`make_shared` 专属** | **性能更好**，单次内存分配。 | 1. 类的内存管理被**自定义 `new/delete`** 接管。<br>2. 内存极度敏感，且存在**大对象和长寿 `weak_ptr`** 的场景。 |

总的来说，"Prefer `std::make_unique` and `std::make_shared` to direct use of `new`" 是一条非常可靠的黄金法则。你应该将其作为默认习惯，只有在遇到上述明确的例外情况时，才考虑回退到手动 `new`。


---
## Item 22: When using the Pimpl Idiom, define special member functions in the implementation file
{% note success %} 

https://cntransgroup.github.io/EffectiveModernCppChinese/4.SmartPointers/item22.html

{% endnote %} 
我们来深入解读一下这条关于 Pimpl 惯用法（Pointer to Implementation）的重要建议。这可以说是现代 C++ 中一个相当高级但非常实用的技巧，它完美地结合了 C++ 的语言特性（特别是 `std::unique_ptr`）和软件工程原则。

> **核心思想**：Pimpl 惯用法旨在将一个类的**接口（interface）**与其**实现（implementation）**彻底分离。通过在类的内部持有一个指向其“真正实现”的指针，我们可以隐藏所有的实现细节。这不仅能保护知识产权，更重要的是它能**减少编译依赖**，从而大幅缩短大型项目的编译时间。然而，当使用 `std::unique_ptr` 作为这个实现指针时，我们必须特别处理类的特殊成员函数，以避免因“不完整类型（incomplete type）”而导致的编译错误。

我们一步步来分解。

### 1. Pimpl 惯用法是什么？它解决了什么问题？

想象一下，你有一个类 `Widget`，定义在 `Widget.h` 中：
```cpp
// Widget.h
#include <string>
#include <vector>
#include "some_library.h" // 一个很庞大的库

class Widget {
public:
    void doSomething();
    // ... 更多接口
private:
    std::string name_;
    std::vector<double> data_;
    SomeLibrary::Gadget gadget_; // 依赖于一个庞大的库
};
```
**问题**：
1.  **编译依赖**：任何 `#include "Widget.h"` 的文件，都必须间接地 `#include` `<string>`, `<vector>`, 和 `"some_library.h"`。如果这些头文件（特别是 `"some_library.h"`）非常大，编译时间会很长。
2.  **耦合**：如果 `Widget` 的任何一个私有成员发生变化（比如把 `std::vector` 换成 `std::list`），所有使用 `Widget` 的客户端代码都**必须重新编译**，即使它们根本不关心这些私有细节。

**Pimpl 的解决方案**：将所有实现细节移到一个单独的 `Impl` 结构体中，并在 `Widget` 类中只持有一个指向它的指针。

**`Widget.h` (使用 Pimpl 之后)**
```cpp
// Widget.h
#include <memory> // For std::unique_ptr

class Widget {
public:
    Widget();
    ~Widget(); // 关键：声明析构函数

    void doSomething();
    // ...

private:
    struct Impl; // 关键：前置声明（Forward Declaration）实现类
    std::unique_ptr<Impl> pImpl; // 指向实现的指针
};
```
**`Widget.cpp`**
```cpp
// Widget.cpp
#include "Widget.h"
#include <string>
#include <vector>
#include "some_library.h"

// 关键：在这里完整定义 Impl 结构体
struct Widget::Impl {
    std::string name_;
    std::vector<double> data_;
    SomeLibrary::Gadget gadget_;
    
    void doSomethingImpl() { /* ... */ }
};

// 在这里实现 Widget 的所有函数
Widget::Widget() : pImpl(std::make_unique<Impl>()) {}

// 关键：在这里定义析构函数
Widget::~Widget() = default; // 或者 {}

void Widget::doSomething() {
    pImpl->doSomethingImpl();
}
```
**好处**：
*   现在 `Widget.h` 极其轻量，不依赖任何具体的实现头文件。
*   如果 `Impl` 结构体内部发生任何变化，只有 `Widget.cpp` 需要重新编译，所有客户端代码都安然无恙。编译时间大大缩短。

### 2. 核心问题：`std::unique_ptr` 与不完整类型（Incomplete Type）

现在我们来解释为什么“必须在实现文件中定义特殊成员函数”。

当你 `#include "Widget.h"` 时，编译器只看到了 `struct Impl;` 这个前置声明。此时，`Impl` 是一个**不完整类型（incomplete type）**。编译器知道有这么个类型，但不知道它的大小、成员、析构函数等任何细节。

`std::unique_ptr` 的析构函数（以及它的默认删除器）会调用 `delete pImpl;`。为了正确地调用 `delete`，编译器**必须知道 `Impl` 的完整定义**，因为它需要调用 `Impl` 的析构函数并释放正确大小的内存。

**如果让编译器自动生成析构函数会发生什么？**
假设你在 `Widget.h` 中没有声明 `~Widget();`。
```cpp
// 客户端代码：Client.cpp
#include "Widget.h"

void some_func() {
    Widget w;
} // w 在这里离开作用域，需要调用 ~Widget()
```
编译器在处理 `Client.cpp` 时，发现需要为 `w` 生成析构函数的调用。由于你没有声明自己的析构函数，编译器会尝试在 `Client.cpp` 中**内联地生成一个默认的析构函数**。这个默认析构函数会依次销毁 `Widget` 的成员，包括 `pImpl`。

当它试图生成销毁 `pImpl` 的代码时，问题就来了：它需要调用 `delete pImpl`，但在 `Client.cpp` 这个上下文中，`Impl` 仍然是一个不完整类型。编译器不知道 `Impl` 的大小和析构函数，于是它会报错，通常是类似 `static_assert failed due to requirement '!is_void<Impl>::value && !is_array<Impl>::value && is_destructible<Impl>::value'` 或者 `cannot delete pointer to incomplete type 'Impl'` 的错误。

**移动操作也一样**：默认的移动构造函数/赋值运算符需要销毁目标对象中原有的 `pImpl`，这同样会触发对不完整类型的 `delete`。

### 3. 解决方案：在实现文件中定义特殊成员函数

这就是这条规则的核心。

1.  **在头文件（`.h`）中声明**：
    ```cpp
    class Widget {
    public:
        Widget();
        ~Widget(); // 声明
        Widget(Widget&&); // 声明移动构造
        Widget& operator=(Widget&&); // 声明移动赋值
        // ...
    };
    ```
    通过**声明**这些函数，你告诉编译器：“别急着自己生成它们，相信我，我会在别的地方提供定义。”

2.  **在实现文件（`.cpp`）中定义**：
    ```cpp
    // Widget.cpp
    #include "Widget.h"
    // ...
    struct Widget::Impl { /* ... */ }; // Impl 现在是完整类型了

    Widget::Widget() : pImpl(std::make_unique<Impl>()) {}
    
    // 在这里，Impl 是完整类型，编译器知道如何销毁它
    Widget::~Widget() = default; 
    Widget::Widget(Widget&&) = default;
    Widget& Widget::operator=(Widget&&) = default;
    ```
    关键在于，当你写下 `= default` 时，编译器会在这里（在 `.cpp` 文件中）生成默认的函数体。在这个位置，`Widget::Impl` 的完整定义是可见的，所以编译器能够成功生成销毁 `pImpl` 的代码。

**注意**：对于拷贝操作，由于 `std::unique_ptr` 本身是不可拷贝的，所以 `Widget` 的拷贝构造/赋值默认就是 `delete` 的。如果你需要让 `Widget` 可拷贝，你就必须自己实现深拷贝逻辑。

### 4. 为什么这条建议不适用于 `std::shared_ptr`？

这是一个非常重要的区别，源于 `shared_ptr` 和 `unique_ptr` 的内部实现差异。

*   `std::unique_ptr`：它的删除逻辑是**编译时**决定的。删除器的类型是 `unique_ptr` 类型的一部分。`sizeof(std::unique_ptr)` 取决于删除器是否有状态。它的析构函数直接调用 `delete`（或自定义删除器）。

*   `std::shared_ptr`：它的删除逻辑是**运行时**决定的。`shared_ptr` 内部除了指向资源的指针，还有一个指向**控制块（Control Block）**的指针。删除器是被**类型擦除（type-erased）**后存储在控制块中的。

当一个 `std::shared_ptr` 被销毁时，它仅仅是原子性地递减控制块中的引用计数。它**不需要知道**被管理对象的完整类型。只有当引用计数减到 0 时，控制块才会调用它内部存储的那个删除器来销毁资源。

因此，当编译器在一个只有 `Impl` 前置声明的上下文中，需要生成销毁 `std::shared_ptr<Impl>` 的代码时，它完全可以做到。它只需要生成递减引用计数的指令，而这与 `Impl` 是否是完整类型**无关**。

所以，如果 Pimpl 使用 `std::shared_ptr`，你**不需要**在头文件中手动声明析构函数和移动操作，让编译器自动生成它们是完全安全的。

### 总结

| 特性 | 使用 `std::unique_ptr` 的 Pimpl | 使用 `std::shared_ptr` 的 Pimpl |
| :--- | :--- | :--- |
| **Pimpl 指针** | `std::unique_ptr<Impl> pImpl;` | `std::shared_ptr<Impl> pImpl;` |
| **`Impl` 类型** | 只需要前置声明 (`class Impl;`) | 只需要前置声明 (`class Impl;`) |
| **特殊成员函数** | **必须**在头文件中声明，在 `.cpp` 文件中定义（即使是 `=default`） | **可以**让编译器自动生成，无需手动声明和定义 |
| **原因** | `unique_ptr` 的析构需要 `Impl` 的**完整类型**信息 | `shared_ptr` 的析构**不需要** `Impl` 的完整类型信息 |

这条规则是理解 `std::unique_ptr` 和不完整类型之间交互的一个绝佳例子，也是正确、高效地使用 Pimpl 惯用法的关键。