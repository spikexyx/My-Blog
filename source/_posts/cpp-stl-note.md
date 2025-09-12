---
title: C++ STL 容器总结
date: 2025-03-15 10:39:59
tags:
    - cpp
    - STL
categories:
    - C++
cover: "/imgs/wallhaven-exmo98.jpg"
excerpt: "本文是对 C++ STL 容器的总结笔记，包含十六大 STL 容器与 pair & tuple，内容包括它们的底层原理和特性分析，以及常用函数和横向对比等。"
comment: false
---

## 序言
本文一共总结了C++里的16大STL容器以及 pair & tuple，包含以下内容：
* array
* vector
* deque
* list
* forward_list
* queue
* priority_queue
* stack
* map
* multimap
* set
* multiset
* unordered_map
* unordered_multimap
* unordered_set
* unordered_multiset
* pair & tuple


## array

> https://en.cppreference.com/w/cpp/container/array.html

**1. 概述 (Overview)**

`std::array` 是 C++11 引入的容器，用于封装一个**固定大小**的数组。它结合了 C 风格数组 (`T arr[N]`) 的性能和内存布局优势，同时提供了现代 C++ 容器的便利性与安全性，如迭代器支持、大小查询和边界检查。

`std::array` 的核心设计哲学是“零成本抽象” (Zero-overhead Abstraction)。这意味着与 C 风格裸数组相比，使用 `std::array` 不会产生任何额外的性能或内存开销，同时又能获得 STL 容器的全部好处。

**2. 头文件 (Header)**

```cpp
#include <array>
```

**3. 特点与优势 (Features & Advantages)**

*   **固定大小 (Fixed Size):** 数组的大小在编译时通过模板参数确定，之后不可更改。例如，`std::array<int, 5>` 和 `std::array<int, 10>` 是两种完全不同的类型。
*   **连续内存 (Contiguous Memory):** 元素在内存中是紧密、连续存储的。这极大地提高了缓存命中率，对高性能计算至关重要。
*   **性能卓越 (High Performance):** 随机访问的时间复杂度为 O(1)。由于其数据通常直接在栈上分配（对于局部变量），避免了堆分配的开销。
*   **STL接口兼容 (STL Interface Compatibility):** 支持标准迭代器 (`begin()`, `end()`)，可与 `<algorithm>` 中的所有算法（如 `std::sort`, `std::for_each`）无缝集成。
*   **安全性 (Safety):** 提供 `.at()` 成员函数进行带边界检查的访问，能有效防止因越界访问导致的未定义行为。
*   **值语义 (Value Semantics):** `std::array` 对象可以像内置类型一样被拷贝、赋值和作为函数返回值。`array2 = array1;` 会执行逐元素的完整拷贝，而不是像 C 风格数组那样退化为指针。

**4. 缺点与限制 (Disadvantages & Limitations)**

*   **大小不可变 (Inflexibility):** 编译时固定大小的特性使其无法用于需要动态增删元素的场景。在这种情况下，应使用 `std::vector`。
*   **栈空间限制 (Stack Space Limitation):** 当 `std::array` 作为局部变量在函数内部声明时，其内存是在栈上分配的。如果数组过大（例如 `std::array<char, 2 * 1024 * 1024>`），极有可能耗尽有限的栈空间，导致程序崩溃（栈溢出）。

**5. 内部实现与内存布局 (Internal Implementation & Memory Layout)**

`std::array` 的美妙之处在于其极致的简洁。它本质上只是对 C 风格裸数组的一层薄薄的 `struct` 封装。

**内部结构**

一个 `std::array<T, N>` 的概念性实现如下：

```cpp
// 概念上的实现，实际标准库实现可能更复杂以支持某些特性
template<typename T, std::size_t N>
struct array {
    // 关键：没有指向堆内存的指针。
    // 关键：没有存储大小的运行时变量 (size)。
    // 关键：没有存储容量的运行时变量 (capacity)。

    // 唯一的数据成员就是一个C风格的裸数组。
    // 这个数组直接作为 struct 的一部分，决定了整个对象的大小和内存布局。
    // 在 C++ 标准中，这个成员的名字是未指定的，但其行为是确定的。
    T __elements[N];

    // ... 各种成员函数（如 begin, end, at, size）的实现...
    // 这些函数不增加对象的大小，它们是编译期的代码生成。
};
```

**内存布局**

由于其内部只有一个裸数组数据成员，`std::array` 对象的内存布局与等效的 C 风格数组**完全相同**。

*   **无额外开销:** `sizeof(std::array<T, N>)` 严格等于 `sizeof(T) * N`。没有任何隐藏的元数据（如大小、容量、指针等）存储在 `std::array` 对象中。数组的大小 `N` 是类型信息的一部分，在编译时便已确定，编译器可以直接使用这个信息，无需在运行时存储。

*   **连续性:** `std::array<int, 4>` 在内存中的布局如下（假设 `sizeof(int)` 为 4 字节）：

    ```
    内存地址: 0x1000      0x1004      0x1008      0x100C
              +-----------+-----------+-----------+-----------+
    arr:      | arr[0]    | arr[1]    | arr[2]    | arr[3]    |
              +-----------+-----------+-----------+-----------+
    ```
    整个 `std::array` 对象 `arr` 就是这一整块连续的内存。

**内存分配**

*   **栈分配:** 当 `std::array` 作为非 `static` 的局部变量时，它的整个数据区（`N` 个元素）都会在**栈 (Stack)** 上分配。这非常快，因为它只涉及移动栈指针。
    ```cpp
    void myFunction() {
        std::array<int, 10> my_stack_array; // 40字节在栈上分配
    }
    ```
*   **静态/全局分配:** 当作为全局变量、`static` 变量时，它会被分配在**静态存储区**。
*   **堆分配 (间接):** 如果你确实需要一个巨大的、固定大小的数组，并希望它在堆上，你可以使用智能指针来管理 `std::array` 的生命周期：
    ```cpp
    auto my_heap_array = std::make_unique<std::array<int, 1000000>>();
    // 此时，栈上只有一个 unique_ptr，而庞大的 array 数据在堆上。
    ```

这个设计与 `std::vector` 形成鲜明对比。`std::vector` 对象本身（通常在栈上）只包含指向**堆内存**的指针以及 `size` 和 `capacity` 变量。数据本身总是在堆上。

**6. 性能分析 (Performance Analysis)**

| 操作 (Operation) | 时间复杂度 (Time Complexity) | 备注 (Notes) |
| :--- | :--- | :--- |
| 随机访问 (`[]`, `at()`) | O(1) | 直接通过指针偏移计算地址，与裸数组一样快。 |
| 遍历 (Iteration) | O(N) | 遍历所有 N 个元素。 |
| 插入/删除 (Insert/Delete) | N/A | 不支持此操作。 |
| 头部/尾部访问 (`front/back`) | O(1) | 直接返回第一个或最后一个元素的引用。 |
| 获取大小 (`size()`) | O(1) | 这是一个 `constexpr` 函数，其值在编译时已知，无运行时开销。 |
| 交换 (`swap()`) | O(N) | 需要逐元素交换两个数组的内容。 |

**7. 常用成员函数 (Common Member Functions)**

```cpp
#include <iostream>
#include <array>
#include <algorithm> // for std::sort

void demonstrate_array() {
    // 1. 初始化 (Initialization)
    std::array<int, 5> arr1 = {1, 2, 3, 4, 5}; // C++11 列表初始化
    std::array<int, 5> arr2{6, 7, 8, 9, 10};    // 同上
    std::array<int, 5> arr3; // 未初始化，元素值不确定 (除非T有默认构造函数)

    // 2. 访问元素 (Element Access)
    arr1[1] = 20; // operator[]: 不进行边界检查，性能最高
    std::cout << "arr1.at(2): " << arr1.at(2) << std::endl;     // at(): 进行边界检查，越界会抛出 std::out_of_range
    std::cout << "arr1.front(): " << arr1.front() << std::endl; // 第一个元素
    std::cout << "arr1.back(): " << arr1.back() << std::endl;   // 最后一个元素

    // .data() 返回指向底层裸数组的指针，用于和C-API交互
    int* p_data = arr1.data();
    std::cout << "Data pointer points to element 0: " << *p_data << std::endl;

    // 3. 迭代器 (Iterators)
    std::cout << "Iterating through arr2: ";
    for(auto const& val : arr2) {
        std::cout << val << " ";
    }
    std::cout << std::endl;

    // 4. 容量 (Capacity)
    std::cout << "Is arr3 empty? " << (arr3.empty() ? "Yes" : "No") << std::endl; // 对于 N>0 的array, 永远是 false
    std::cout << "Size of arr1: " << arr1.size() << std::endl;      // 总是返回模板参数 N
    std::cout << "Max size of arr1: " << arr1.max_size() << std::endl; // 与 size() 相同

    // 5. 操作 (Operations)
    arr3.fill(100); // 将所有元素填充为 100
    std::cout << "arr3 after fill: " << arr3[0] << std::endl;
    
    arr1.swap(arr2); // 交换两个array的内容，O(N)操作
    std::cout << "arr1.front() after swap: " << arr1.front() << std::endl;

    // 配合 <algorithm>
    std::sort(arr1.begin(), arr1.end()); // 排序
    std::cout << "arr1.front() after sort: " << arr1.front() << std::endl;
}
```

**8. 使用场景与最佳实践 (Use Cases & Best Practices)**

*   **替代C风格数组：** 在任何需要 `T name[N];` 的场景，都应优先使用 `std::array<T, N>`。它提供了类型安全、STL兼容性和更丰富的接口，且无任何性能损失。
*   **小型、固定大小的数据聚合：** 非常适合表示几何向量 (`std::array<double, 3>`)、颜色值 (`std::array<uint8_t, 4>`)、固定大小的查找表或配置参数。
*   **高性能缓冲区：** 当需要一个固定大小的缓冲区用于I/O操作（如网络、文件读写），并希望避免堆分配开销时，`std::array` 是理想之选。
*   **函数参数传递：**
    *   `void func(std::array<int, 10> arr)`: 按值传递，会完整拷贝数组，开销大，应避免。
    *   `void func(const std::array<int, 10>& arr)`: **推荐！** 按常量引用传递，高效且防止意外修改。
    *   `void func(std::array<int, 10>& arr)`: 按引用传递，允许函数修改原数组内容。

**9. 与其他容器的对比 (Comparison)**

*   **`std::array` vs. C-style array (`T[]`)**
    *   **优势：** `std::array` 是一个真正的对象，它知道自己的大小 (`.size()`)，支持迭代器和STL算法，可以安全地按值传递和拷贝，并提供带边界检查的 `.at()` 方法。
    *   **劣势：** 无。`std::array` 是对 C 风格数组的全面现代化升级。

*   **`std::array` vs. `std::vector`**
    *   **核心区别：** 内存分配位置和大小的可变性。
    *   **`std::array`:** 大小编译时确定，不可变。数据通常在**栈**上分配（对局部变量而言）。无额外内存开销。
    *   **`std::vector`:** 大小运行时可变，可动态增长/收缩。数据**总是**在**堆**上分配。对象本身有额外的内存开销（存储 `size`、`capacity` 和指向数据的指针）。
    *   **选择时机：**
        *   当集合大小在编译时**已知且固定**，并且追求极致性能（如利用栈分配）时，选择 `std::array`。
        *   当集合大小需要**动态改变**，或大小在运行时才能确定时，必须使用 `std::vector`。


## vector

> https://en.cppreference.com/w/cpp/container/vector.html

**1. 概述 (Overview)**

`std::vector` 是 C++ 标准库中功能最强大、使用最广泛的序列容器。它本质上是一个**可以动态增长和收缩的数组**。`std::vector` 将元素存储在连续的内存块中，这使得它既能像普通数组一样进行快速的随机访问，又具备在运行时调整大小的灵活性。

由于其出色的综合性能和通用性，`std::vector` 通常被视为**默认的序列容器**。当你不确定应该使用哪种容器来存储一个元素序列时，`std::vector` 往往是最佳的起点。

**2. 头文件 (Header)**

```cpp
#include <vector>
```

**3. 特点与优势 (Features & Advantages)**

*   **动态大小 (Dynamic Size):** 可以在运行时添加或删除元素，容器大小随之改变。
*   **连续内存与快速随机访问 (Contiguous Memory & Fast Random Access):** 和 `std::array` 一样，元素在内存中连续存放。这带来了极佳的缓存局部性，并允许通过 `operator[]` 和 `.at()` 在 O(1) 时间内访问任意元素。
*   **均摊常数时间的尾部插入/删除 (Amortized O(1) Appending/Removing):** `push_back()` 的平均时间复杂度为 O(1)。虽然偶尔的重新分配是 O(N)，但其成本被分摊到多次快速的插入操作上。`pop_back()` 总是 O(1)。
*   **与 C 语言 API 兼容:** 可以通过 `.data()` 方法获取指向底层连续内存的裸指针，方便与需要 `T*` 类型参数的 C 风格函数交互。

**4. 缺点与限制 (Disadvantages & Limitations)**

*   **中间插入/删除昂贵 (Expensive Middle Insertion/Deletion):** 在 `vector` 的开头或中间插入/删除元素是一个 O(N) 操作，因为它需要移动该位置之后的所有元素来填补空位或腾出空间。
*   **重新分配开销 (Reallocation Overhead):** 重新分配过程（内存分配、元素移动/拷贝、内存释放）可能非常耗时，尤其是在元素数量巨大或元素拷贝成本高昂时。
*   **迭代器失效 (Iterator Invalidation):** 这是 `std::vector` 的一个关键且危险的特性。
    *   **导致重新分配的操作**（如 `push_back` 导致容量变化、`reserve`、`shrink_to_fit`）会使**所有**指向该 `vector` 元素的迭代器、指针和引用失效。
    *   **`insert()` 操作**会使插入点**及之后**的所有迭代器、指针和引用失效。
    *   **`erase()` 操作**会使被删除点**及之后**的所有迭代器、指针和引用失效。


**5. 内部实现与内存布局 (Internal Implementation & Memory Layout)**

理解 `std::vector` 的内部工作方式是精通它的关键，因为这直接关系到其性能特征和使用限制。

**内部结构**

`std::vector` 对象本身非常小，通常只包含三个成员（通常是指针或整型），无论它管理多少数据。一个常见的实现模型是：

1.  `T* _data`: 一个指向**堆内存 (Heap)** 中动态分配的数组的起始位置的指针。所有的元素都存储在这块内存中。
2.  `size_type _size`: 一个整数，记录当前 `vector` 中实际存储的元素数量。
3.  `size_type _capacity`: 一个整数，记录在**不重新分配内存**的情况下，`vector` **最多**可以容纳的元素数量。

**永远存在的关系：`0 <= size() <= capacity()`**

**内存布局**

`std::vector` 的内存模型是**分离式**的：

*   **Vector 对象本身:** `std::vector<T> vec;` 这个 `vec` 对象（包含上述三个成员 `_data`, `_size`, `_capacity`）通常位于**栈上**（如果作为局部变量）或静态存储区。它的大小是固定的，仅为 `sizeof(T*) + 2 * sizeof(size_t)`。
*   **元素数据区:** 所有元素实际存储的内存块位于**堆上**。这块内存的大小是 `capacity() * sizeof(T)`。

```
// 内存模型示意图

  Stack or Static Memory                   Heap Memory
+-----------------------------+
| vec object                  |           +---------------------------------+ - - - - -
|                             |           |                                 |
| T*       _data  ------------+---------> | elem[0] | elem[1] | ... | elem[S-1] | Unused Space... |
| size_t   _size = S          |           +---------------------------------+ - - - - -
| size_t   _capacity = C      |             <---------- size() ----------->
|                             |             <---------------- capacity() ------------------>
+-----------------------------+
```

**动态增长 (重新分配 / Reallocation)**

这是 `std::vector` 最核心的行为。当调用 `push_back()` 且 `size() == capacity()` 时，会发生以下昂贵的操作：

1.  **分配新内存:** 在堆上申请一块**更大**的内存。新容量通常是旧容量的某个倍数（常见的增长因子是 1.5 或 2，具体由标准库实现决定）。
2.  **移动或拷贝元素:** 将所有旧内存中的元素转移到新内存中。
    *   **移动 (Move):** 如果元素类型 `T` 提供了 `noexcept` 的移动构造函数 (C++11及以后)，编译器会优先使用移动，这非常高效，只涉及指针和资源的转移，不涉及深拷贝。
    *   **拷贝 (Copy):** 如果 `T` 没有移动构造函数，或其移动构造函数可能抛出异常，为了保证强异常安全（如果拷贝中途失败，原 vector 仍然有效），标准库会回退到使用拷贝构造函数。拷贝的开销可能非常大。
3.  **销毁旧元素:** 调用旧内存中所有元素的析构函数。
4.  **释放旧内存:** 将旧的内存块归还给系统。
5.  **更新成员:** 更新 `_data` 指针指向新内存，并更新 `_capacity` 为新容量。

由于重新分配的存在，`vector` 的尾部插入操作的性能被描述为**均摊 O(1)**。

**6. 性能分析 (Performance Analysis)**

| 操作 (Operation) | 时间复杂度 (Time Complexity) | 备注 (Notes) |
| :--- | :--- | :--- |
| 随机访问 (`[]`, `at()`) | O(1) | |
| 尾部插入 (`push_back`) | **均摊 O(1)**, 最坏 O(N) | O(N) 发生在需要重新分配内存时。 |
| 尾部删除 (`pop_back`) | O(1) | 仅减少 `size`，不释放内存。 |
| 中间/头部插入/删除 | O(N) | 需要移动后续元素。 |
| 获取大小 (`size()`) | O(1) | 只是返回一个成员变量的值。 |
| 预留空间 (`reserve()`) | O(N) or O(1) | 如果请求的容量大于当前容量，则为 O(N)；否则为 O(1)。|

**7. 常用成员函数 (Common Member Functions)**

```cpp
#include <iostream>
#include <vector>
#include <string>

void demonstrate_vector() {
    // 1. 初始化 (Initialization)
    std::vector<std::string> vec1; // 空 vector
    std::vector<int> vec2(5, 100); // 5个 int，每个都初始化为 100
    std::vector<int> vec3 = {1, 2, 3, 4, 5}; // C++11 列表初始化
    std::vector<int> vec4(vec3.begin(), vec3.end()); // 从其他容器迭代器范围构造

    // 2. 容量管理 (Capacity)
    vec1.push_back("hello");
    vec1.push_back("world");
    std::cout << "vec1 size: " << vec1.size() << std::endl;         // 实际元素数量
    std::cout << "vec1 capacity: " << vec1.capacity() << std::endl; // 可容纳元素数量

    // 关键优化：如果预知大小，提前 reserve
    std::vector<int> vec5;
    vec5.reserve(1000); // 提前分配1000个元素的空间，避免后续 push_back 的重新分配
    for (int i = 0; i < 1000; ++i) {
        vec5.push_back(i); // 这些 push_back 不会触发重新分配
    }
    std::cout << "vec5 size: " << vec5.size() << ", capacity: " << vec5.capacity() << std::endl;
    vec5.shrink_to_fit(); // 请求释放多余容量（非强制）
    std::cout << "vec5 after shrink_to_fit, capacity: " << vec5.capacity() << std::endl;

    // 3. 元素访问 (Element Access)
    std::cout << "vec3[1]: " << vec3[1] << std::endl;       // 不检查边界
    std::cout << "vec3.at(2): " << vec3.at(2) << std::endl; // 检查边界
    std::cout << "vec3.front(): " << vec3.front() << std::endl;
    std::cout << "vec3.back(): " << vec3.back() << std::endl;
    int* data_ptr = vec3.data(); // 获取裸指针

    // 4. 修改器 (Modifiers)
    vec3.pop_back(); // 删除最后一个元素 {1, 2, 3, 4}
    vec3.insert(vec3.begin() + 1, 99); // 在索引1处插入99 {1, 99, 2, 3, 4}
    vec3.erase(vec3.begin() + 2); // 删除索引2处的元素 {1, 99, 3, 4}

    // resize vs reserve
    vec3.resize(10); // 改变 size 为 10，新元素会被值初始化(为0) {1, 99, 3, 4, 0, 0, 0, 0, 0, 0}
    vec3.resize(2);  // 改变 size 为 2，多余元素被销毁 {1, 99}
    
    vec3.clear(); // 清空所有元素，size变为0，但 capacity 不变
    std::cout << "vec3 after clear, size: " << vec3.size() << ", capacity: " << vec3.capacity() << std::endl;
}
```

**8. 使用场景与最佳实践 (Use Cases & Best Practices)**

*   **默认选择:** 当你需要一个元素序列时，如果没有特殊需求（如频繁的头部插入/删除），`std::vector` 应该是你的首选。
*   **预分配内存 (`reserve`):** 在将要添加大量元素之前，如果能预估最终的大小，请务必使用 `reserve()`。这是 `std::vector` 最重要的性能优化手段，可以完全避免重新分配的开销。
*   **使用 `emplace_back`:** 对于复杂对象，优先使用 `emplace_back()` 而不是 `push_back()`。`emplace_back` 可以就地构造对象，避免不必要的临时对象的创建和拷贝/移动。
*   **`erase-remove` 惯用法:** 要高效地删除 `vector` 中满足特定条件的多个元素，应使用 `erase-remove` 惯用法：
    ```cpp
    // v.erase(std::remove_if(v.begin(), v.end(), condition), v.end());
    ```
*   **注意迭代器失效:** 在循环中修改 `vector` 时要格外小心。一个常见的错误是在 `for` 循环中 `erase` 元素，导致迭代器失效。正确的做法是使用 `erase` 返回的有效迭代器。
*   **传递给函数:** 优先按常量引用 `const std::vector<T>&` 传递，以避免昂贵的整体拷贝。

**9. 与其他容器的对比 (Comparison)**

*   **`vector` vs `std::array`:**
    *   `vector` 大小动态，数据在堆上；`array` 大小固定，数据通常在栈上。
    *   选择 `array` 当且仅当大小在编译时已知且固定。

*   **`vector` vs `std::deque`:**
    *   `vector` 只支持高效的**尾部**插入/删除。
    *   `deque` (双端队列) 支持高效的**头部和尾部**插入/删除。
    *   `vector` 保证内存**完全连续**，`deque` 不保证（通常由分块的内存数组实现）。因此 `vector` 与 C API 交互更好。

*   **`vector` vs `std::list`:**
    *   `vector` 随机访问 O(1)，缓存友好；`list` 随机访问 O(N)，缓存不友好。
    *   `vector` 中间插入/删除 O(N)；`list` (双向链表) 只要有迭代器，中间插入/删除是 O(1)。
    *   `list` 的任何插入/删除操作都不会使指向其他元素的迭代器失效（除了被删除的那个）。


## deque

> https://en.cppreference.com/w/cpp/container/deque.html

**1. 概述 (Overview)**
`std::deque`（发音通常是 "deck"）是一个序列容器，其名称代表“双端队列”(Double-Ended Queue)。它的核心特性是支持在**序列的头部和尾部**进行快速的（均摊 O(1) 时间）插入和删除操作。

与 `std::vector` 类似，`std::deque` 也提供 O(1) 的随机访问能力。但与 `std::vector` 不同的是，`std::deque` 的元素**不保证存储在单个连续的内存块中**。这种独特的内部结构赋予了它在两端操作的灵活性，但也带来了一些与 `std::vector` 不同的性能权衡。

**2. 头文件 (Header)**
```cpp
#include <deque>
```

**3. 特点与优势 (Features & Advantages)**
*   **高效的双端操作:** `push_front()`, `pop_front()`, `push_back()`, `pop_back()` 的时间复杂度均为均摊 O(1)。这是 `deque` 最显著的优势。
*   **快速的随机访问:** 支持 `operator[]` 和 `.at()`，时间复杂度为 O(1)。虽然是常数时间，但由于其非连续内存的结构，其常数因子通常比 `std::vector` 更大，即实际速度会稍慢。
*   **较好的迭代器和引用稳定性:** 在 `deque` 的两端进行插入/删除操作时，**不会**使指向其他元素的指针或引用失效。但是，**迭代器可能会失效**（因为内部的索引表可能需要重新分配）。这比 `vector` 在扩容时所有迭代器、指针和引用都失效要好。

**4. 缺点与限制 (Disadvantages & Limitations)**
*   **非连续内存:** 这是 `deque` 最大的特点，也是其主要“缺点”的根源。
    *   **缓存性能稍差:** 元素分布在不同的内存块中，可能导致遍历时的缓存命中率低于 `std::vector`。
    *   **无法直接与C API交互:** `deque` 没有提供像 `vector::data()` 这样的方法来获取一个指向连续数据块的指针。
*   **中间插入/删除昂贵:** 和 `vector` 一样，在 `deque` 的中间插入或删除元素是 O(N) 操作，需要移动后续元素。
*   **更高的内存开销:** 除了存储元素本身，`deque` 还需要一个额外的索引结构来管理其内存块，因此它的内存开销比 `vector` 或 `array` 要大。

**5. 内部实现与内存布局 (Internal Implementation & Memory Layout)**
`std::deque` 的实现是其所有特性的根源，通常采用一种称为**“分块数组”**或**“块状链表”**的数据结构。

**内部结构**
`std::deque` 的内部通常由两部分组成：

1.  **映射/索引 (Map):** 一个连续的内存块，本身就像一个 `T**` 或 `std::vector<T*>`。它存储了一系列的**指针**。
2.  **数据块/节点 (Chunks/Nodes):** 每个在“映射”中的指针都指向一个固定大小的、在**堆上**分配的连续内存块。实际的元素就存储在这些数据块中。

**内存布局**
```
// 内存模型示意图

      "Map" (Contiguous array of pointers)
      +---------+---------+---------+---------+
      | ptr_0   | ptr_1   | ptr_2   |  ...    |
      +---------+---------+---------+---------+
         |         |         |
         |         |         +------------------------+
         |         |                                  |
         |         +----------------+                 v
         |                          |             Chunk 2 (Heap)
         v                          v            +---+---+---+---+
      Chunk 0 (Heap)             Chunk 1 (Heap)  | E | E | E | E |
     +---+---+---+---+          +---+---+---+---+  +---+---+---+---+
     | E | E | E | E |          | E | E | E | E |
     +---+---+---+---+          +---+---+---+---+
     (E = Element)
```
*   `deque` 对象本身持有指向这个“映射”的指针，以及记录 `size`、起始位置等信息的元数据。
*   元素在**单个数据块内部是连续的**，但**数据块之间在内存中是分散的**。

**操作如何工作**
*   **随机访问 (`dq[i]`):**
    1.  通过 `i` 和每个数据块的大小，计算出元素 `i` 位于哪个数据块（`chunk_index`）以及在块内的偏移量（`offset`）。例如：`chunk_index = i / CHUNK_SIZE`, `offset = i % CHUNK_SIZE`。
    2.  在“映射”中查找 `map[chunk_index]` 得到数据块的指针。
    3.  通过 `chunk_ptr + offset` 访问元素。
    这个过程涉及两次指针跳转和一些整数运算，但仍然是 O(1) 的。

*   **`push_back()`:** 如果最后一个数据块有空间，直接在末尾构造元素。如果满了，就分配一个新的数据块，在“映射”的末尾添加指向新块的指针，然后在新块的起始位置构造元素。如果“映射”本身也满了，就需要重新分配一个更大的“映射”，并把旧指针拷贝过去。

*   **`push_front()`:** 与 `push_back` 对称。如果第一个数据块的前面有空间，直接构造。如果满了，分配新块，在“映射”的**起始位置之前**添加指向新块的指针。为了支持这一点，“映射”通常被实现为一个环形缓冲区或在中间留有空闲空间，以便向两个方向扩展。

**6. 性能分析 (Performance Analysis)**
| 操作 (Operation) | 时间复杂度 (Time Complexity) | 备注 (Notes) |
| :--- | :--- | :--- |
| 随机访问 (`[]`, `at()`) | O(1) | 常数因子比 `vector` 大，实际速度稍慢。 |
| 头部插入/删除 (`push/pop_front`) | **均摊 O(1)** | 极少数情况需分配新块或扩展映射，成本被分摊。|
| 尾部插入/删除 (`push/pop_back`) | **均摊 O(1)** | 同上。 |
| 中间插入/删除 | O(N) | 涉及大量元素移动。 |
| 获取大小 (`size()`) | O(1) | 返回一个成员变量的值。 |

**7. 常用成员函数 (Common Member Functions)**
```cpp
#include <iostream>
#include <deque>

void demonstrate_deque() {
    // 1. 初始化
    std::deque<int> dq;

    // 2. 双端操作
    dq.push_back(10);    // dq: {10}
    dq.push_front(20);   // dq: {20, 10}
    dq.push_back(30);    // dq: {20, 10, 30}
    dq.push_front(40);   // dq: {40, 20, 10, 30}

    std::cout << "Deque elements: ";
    for(int val : dq) { std::cout << val << " "; }
    std::cout << std::endl;

    dq.pop_back();     // dq: {40, 20, 10}
    dq.pop_front();    // dq: {20, 10}

    // 3. 随机访问
    std::cout << "dq[1]: " << dq[1] << std::endl; // 输出 10
    dq[1] = 99; // 修改元素
    std::cout << "dq.at(1): " << dq.at(1) << std::endl; // 输出 99

    // 4. 其他操作
    std::cout << "Size: " << dq.size() << std::endl;
    dq.clear();
    std::cout << "Size after clear: " << dq.size() << std::endl;
    // 注意: deque 也有 shrink_to_fit()，尝试释放未使用的内存块
}
```

**8. 使用场景与最佳实践 (Use Cases & Best Practices)**
*   **实现队列或栈:** `std::deque` 是实现 `std::queue` 和 `std::stack` 这两个容器适配器的默认底层容器，因为它能高效地在特定一端或两端进行操作。
*   **需要双端操作的场景:** 任何需要频繁在序列头部和尾部添加/删除元素的场景。一个典型的例子是**滑动窗口**算法，需要不断从一端添加新元素，并从另一端移除旧元素。
*   **不确定大小的缓冲区:** 当你需要一个缓冲区，它会从两端增长或缩减时，`deque` 是一个很好的选择。
*   **避免选择 `deque` 的场景:**
    *   如果**只**在尾部操作，`std::vector` 通常性能更好（更好的缓存局部性，更低的内存开销）。
    *   如果需要频繁在**中间**插入/删除，`std::list` 是更好的选择。
    *   如果需要将数据传递给一个期望连续内存的C API，必须使用 `std::vector`。

**9. 与其他容器的对比 (Comparison)**
*   **`deque` vs. `vector`**
    *   **共同点:** O(1) 随机访问，O(N) 中间插入/删除。
    *   **`deque` 优势:** 高效的头部插入/删除 (`push_front`/`pop_front`)。
    *   **`vector` 优势:** 保证内存连续（缓存和C API友好），更快的随机访问（常数因子更小），更低的内存开销。

*   **`deque` vs. `list`**
    *   **`deque` 优势:** O(1) 随机访问。缓存局部性远好于 `list`。
    *   **`list` 优势:** O(1) 的中间插入/删除（只要有迭代器）。任何插入/删除操作都**不会**使指向其他元素的迭代器、指针或引用失效。


## list

> https://en.cppreference.com/w/cpp/container/list.html

**1. 概述 (Overview)**
`std::list` 是 C++ 标准库中的一个序列容器，它以**双向链表** (Doubly-Linked List) 的形式组织其元素。与 `std::vector` 和 `std::deque` 不同，`std::list` 的元素在内存中是**非连续存储**的。每个元素都作为一个独立的节点存在，节点内部除了存储数据外，还包含了指向前一个节点和后一个节点的指针。

这种数据结构决定了 `std::list` 的核心特性：它在序列的**任何位置**进行插入和删除操作都非常高效（O(1) 时间），但代价是失去了快速的随机访问能力。

**2. 头文件 (Header)**
```cpp
#include <list>
```

**3. 特点与优势 (Features & Advantages)**
*   **高效的任意位置插入/删除:** 只要你拥有一个指向某个位置的迭代器，就可以在 O(1) 的时间内在该位置插入或删除元素。这包括在列表的头部、尾部和中间。这是 `std::list` 最突出的优点。
*   **极佳的迭代器/指针/引用稳定性:** 这是 `std::list` 的另一个王牌特性。在 `std::list` 中进行任何插入操作，都**不会**使任何已存在的迭代器、指针或引用失效。删除操作只会使指向被删除元素的迭代器、指针和引用失效。这是所有标准序列容器中稳定性最好的。
*   **灵活的拼接操作 (`splice`)**: `std::list` 提供了独特的 `splice` 成员函数，可以在 O(1) 的时间内将另一个 `list` 的全部或部分元素移动到当前 `list` 中，而无需进行任何元素的拷贝或移动构造。

**4. 缺点与限制 (Disadvantages & Limitations)**
*   **不支持快速随机访问:** `std::list` 没有提供 `operator[]` 或 `.at()` 方法。要访问第 `i` 个元素，必须从头部或尾部开始，沿着指针逐个遍历，时间复杂度为 O(N)。
*   **糟糕的缓存性能:** 由于节点在内存中是分散存储的，遍历 `std::list` 时会发生频繁的指针跳转，导致极差的缓存局部性 (Cache Locality)。这使得即使是线性的遍历操作，其真实世界的性能也远不如 `std::vector` 或 `std::deque`。
*   **高内存开销:** 每个节点除了存储元素 `T` 本身，还必须额外存储两个指针（`prev` 和 `next`）。因此，`std::list` 的总内存开销大约是 `N * (sizeof(T) + 2 * sizeof(void*))`，远高于 `std::vector`。

**5. 内部实现与内存布局 (Internal Implementation & Memory Layout)**
`std::list` 的内部实现非常直观，就是一个经典的双向链表。

**内部结构**
1.  **节点 (Node):** `std::list` 的基本构成单位。每个节点通常是一个 `struct`，包含三个部分：
    *   `T data`: 存储的元素数据。
    *   `Node* prev`: 指向前一个节点的指针。
    *   `Node* next`: 指向后一个节点的指针。
    每个节点都是在**堆上**独立分配内存的。

2.  **List 对象:** `std::list` 对象本身非常小，它通常只包含：
    *   一个指向**哨兵节点 (Sentinel Node)** 的指针（或直接包含哨兵节点）。
    *   一个 `size_t` 类型的成员变量，用于记录元素数量，以便 `size()` 可以在 O(1) 时间内返回。

    **哨兵节点**是一种常见的优化技巧。它是一个特殊的、不存储实际数据的节点，它的 `next` 指针指向链表的第一个真实节点，`prev` 指针指向最后一个真实节点。同时，第一个真实节点的 `prev` 和最后一个真实节点的 `next` 都指向这个哨兵节点，形成一个环。这可以极大地简化 `insert`, `erase`, `begin`, `end` 等操作的逻辑，避免对空链表或边界情况做特殊处理。

**内存布局**
```
// 内存模型示意图

   list object (Stack/Static)
 +-----------------------------+
 | Node* sentinel_node_ptr ----+----> Sentinel Node (Heap)
 | size_t size                 |   +--------------------------+
 +-----------------------------+   | data: (unused)           |
                                   | prev: (points to last) --+---------+
                                   | next: (points to first) -+---+     |
                                   +--------------------------+  |     |
                                          ^                      |     |
                                          |                      v     |
  +---------------------------------------+   +------------------+     |
  |                                           |                        |
  v   Node 1 (Heap, Address 0xAAAA)           v   Node 2 (Heap, 0xBBBB)  |
+--------------------------+                +--------------------------+
| data: T1                 | <--------------+ data: T2                 |
| prev: (points to sent.)  +----------------+ prev: (points to Node 1) |
| next: (points to Node 2) +--------------->| next: (points to Node 3) +---> ...
+--------------------------+                +--------------------------+
```
从图中可以看出，节点 `Node 1`, `Node 2` 等在堆上的内存地址是完全不相关的，它们通过指针链接在一起。

**6. 性能分析 (Performance Analysis)**
| 操作 (Operation) | 时间复杂度 (Time Complexity) | 备注 (Notes) |
| :--- | :--- | :--- |
| 随机访问 | O(N) | 不支持 `[]` 或 `at()`。必须遍历。 |
| 头部/尾部插入/删除 | O(1) | `push/pop_front`, `push/pop_back`。 |
| **任意位置插入/删除** | **O(1)** | **前提是已拥有指向该位置的迭代器**。 |
| 遍历 (Iteration) | O(N) | 缓存性能差。 |
| 搜索 (Search) | O(N) | 必须逐个元素比较。 |
| 合并 (`merge`, `splice`) | O(N) 或 O(1) | `splice` 移动元素是 O(1)，`merge` 排序合并是 O(N)。 |

**7. 常用成员函数 (Common Member Functions)**
`std::list` 提供了一些 `std::vector` 和 `std::deque` 没有的特殊成员函数。

```cpp
#include <iostream>
#include <list>
#include <numeric> // for std::iota

void demonstrate_list() {
    // 1. 初始化
    std::list<int> list1 = {10, 20, 30, 40};
    std::list<int> list2(5);
    std::iota(list2.begin(), list2.end(), 1); // list2: {1, 2, 3, 4, 5}

    // 2. 插入和删除
    auto it = list1.begin();
    std::advance(it, 2); // it 指向 30
    list1.insert(it, 99); // list1: {10, 20, 99, 30, 40} - O(1) 操作

    list1.erase(it); // 删除 it 指向的元素(30), list1: {10, 20, 99, 40} - O(1) 操作
                     // 注意：it现在失效了！

    // 3. 特有的成员函数
    
    // splice: 将 list2 的所有元素移动到 list1 的开头
    list1.splice(list1.begin(), list2);
    // list1: {1, 2, 3, 4, 5, 10, 20, 99, 40}
    // list2: is now empty!
    // 这个操作只是修改了几个指针，非常快。

    list1.sort(); // 内部排序
    
    list1.unique(); // 移除连续的重复元素
    
    std::list<int> list3 = {0, 15, 35};
    list1.merge(list3); // 将已排序的 list3 合并到 list1 中，并保持有序
                        // list3 becomes empty.

    // remove / remove_if: 按值或条件删除元素
    list1.remove(99); // 删除所有值为 99 的元素
    list1.remove_if([](int n){ return n > 30; }); // 删除所有大于30的元素

    std::cout << "Final list1: ";
    for(int val : list1) { std::cout << val << " "; }
    std::cout << std::endl;
}
```

**8. 使用场景与最佳实践 (Use Cases & Best Practices)**
在现代 C++ 中，`std::list` 的使用场景已经变得**非常有限**。

*   **主要适用场景:**
    *   当你需要对一个大型集合进行**频繁的、在任意位置的插入和删除**操作时。
    *   当**迭代器/指针的稳定性**是首要需求时。例如，你可能需要存储指向容器中元素的持久指针，并且不希望这些指针因为其他位置的增删而失效。

*   **警惕性能陷阱:** 由于其糟糕的缓存性能，即使 `std::list` 在理论时间复杂度上有优势，在实际应用中，`std::vector` 重新分配并移动元素可能比 `std::list` 修改指针更快，特别是对于小型对象。**在选择 `std::list` 之前，务必进行性能分析和基准测试**。

*   **现代替代方案:** 在很多情况下，如果关心插入/删除性能，但又不想完全牺牲缓存性能，可以考虑使用**基于节点的无序关联容器**（如 `std::unordered_map`），或者第三方库提供的**侵入式链表 (Intrusive List)**，或**B+树**等更复杂的数据结构。

**9. 与其他容器的对比 (Comparison)**
*   **`list` vs. `vector` / `deque`**
    *   **核心权衡:** 随机访问 vs. 任意位置插入/删除。
    *   **`list`:** O(1) 任意位置插入/删除，O(N) 随机访问，迭代器稳定性好，缓存性能差，内存开销大。
    *   **`vector`/`deque`:** O(1) 随机访问，O(N) 任意位置插入/删除，迭代器稳定性差，缓存性能好，内存开销小。

*   **`list` vs. `forward_list`**
    *   `std::forward_list` 是 C++11 引入的**单向链表**。
    *   它比 `std::list` 更轻量，每个节点只存储一个 `next` 指针，内存开销更小。
    *   它只支持前向遍历，不支持 `size()` (必须O(N)计算)、`push_back`、`pop_back` 等操作。


## forward_list

> https://en.cppreference.com/w/cpp/container/forward_list.html

**1. 概述 (Overview)**
`std::forward_list` 是 C++11 中引入的一个序列容器，它以**单向链表** (Singly-Linked List) 的形式组织其元素。顾名思义，它只支持从头到尾的**前向遍历**。

`std::forward_list` 被设计为 `std::list` 的一个更节省空间、在某些操作上可能更快的替代品。它牺牲了双向遍历的能力，以换取更小的内存占用和更简单的节点结构。它的设计哲学是**极致的简约和效率**，只提供单向链表所必需的最基本功能。

**2. 头文件 (Header)**
```cpp
#include <forward_list>
```

**3. 特点与优势 (Features & Advantages)**
*   **极致的内存效率:** 这是 `std::forward_list` 最核心的优势。每个节点只包含元素数据和一个指向下一个节点的指针 (`next`)。相比 `std::list`（包含 `prev` 和 `next` 两个指针），它的每个节点的内存开销更小。
*   **高效的头部插入/删除:** 在链表头部进行插入 (`push_front`) 和删除 (`pop_front`) 操作是 O(1) 时间。
*   **高效的任意位置"之后"插入/删除:** `std::forward_list` 提供了 `insert_after` 和 `erase_after` 成员函数。只要拥有一个指向某位置的迭代器，就可以在 O(1) 的时间内在其**后面**插入或删除元素。
*   **极佳的迭代器/指针/引用稳定性:** 与 `std::list` 一样，插入操作不会使任何迭代器失效。删除操作仅使指向被删除元素的迭代器失效。

**4. 缺点与限制 (Disadvantages & Limitations)**
*   **仅支持前向遍历:** 无法从一个迭代器反向移动到前一个元素。迭代器是前向迭代器 (`ForwardIterator`)，而不是双向迭代器 (`BidirectionalIterator`)。
*   **不支持 `size()` 方法:** 为了保持极致的轻量级，`std::forward_list` 对象不存储元素数量。调用 `size()` 方法是不允许的（编译错误）。如果需要获取大小，必须手动遍历整个链表，这是一个 O(N) 操作。可以使用 `std::distance(fl.begin(), fl.end())` 来计算。
*   **没有尾部操作:** 不支持 `push_back`, `pop_back`, `back()` 等直接操作尾部元素的函数。要在尾部添加元素，必须遍历到最后一个节点，这是一个 O(N) 操作。
*   **“之后”操作的特殊性:** 它的插入和删除都是 `*_after` 形式，这意味着你不能直接删除或在迭代器指向的位置**之前**插入。要删除一个由迭代器 `it` 指向的元素，你需要一个指向它**前一个元素**的迭代器 `prev_it`，然后调用 `erase_after(prev_it)`。

**5. 内部实现与内存布局 (Internal Implementation & Memory Layout)**
`std::forward_list` 的内部实现比 `std::list` 更为简单。

**内部结构**
1.  **节点 (Node):**
    *   `T data`: 存储的元素数据。
    *   `Node* next`: 指向下一个节点的指针。
    每个节点同样是在**堆上**独立分配内存。

2.  **Forward_list 对象:**
    `std::forward_list` 对象本身是所有容器中最小的，它通常只包含一个成员：
    *   `Node* head`: 一个指向链表第一个节点的指针。
    当链表为空时，`head` 为 `nullptr`。它没有哨兵节点，也没有 `size` 成员。

**内存布局**
```
// 内存模型示意图

 forward_list object (Stack/Static)
 +---------------------------+
 | Node* head   -------------+----> Node 1 (Heap, Address 0xAAAA)
 +---------------------------+     +--------------------------+    Node 2 (Heap, 0xBBBB)
                                   | data: T1                 |   +--------------------------+
                                   | next: (points to Node 2) +-->| data: T2                 |
                                   +--------------------------+   | next: (points to Node 3) +---> ...
                                                                  +--------------------------+
```
其内存布局与 `std::list` 类似，都是非连续的节点通过指针链接。主要区别在于每个节点更小，且链接是单向的。

**6. 性能分析 (Performance Analysis)**
| 操作 (Operation) | 时间复杂度 (Time Complexity) | 备注 (Notes) |
| :--- | :--- | :--- |
| 访问第N个元素 | O(N) | 必须从头遍历。 |
| 头部插入/删除 (`push/pop_front`) | O(1) | |
| **在迭代器之后插入/删除** | **O(1)** | `insert_after`, `erase_after`。 |
| 遍历 (Iteration) | O(N) | 缓存性能与 `list` 同样差。 |
| 搜索 (Search) | O(N) | 必须逐个元素比较。 |
| 获取大小 (`size()`) | **N/A** (O(N) via `std::distance`) | 不提供 O(1) 的 `size()` 方法。 |
| 合并 (`merge`, `splice_after`) | O(N) 或 O(1) | `splice_after` 移动元素是 O(1)，`merge` 是 O(N)。 |

**7. 常用成员函数 (Common Member Functions)**
`std::forward_list` 的接口经过精心设计，以反映其单向和“之后”操作的特性。

```cpp
#include <iostream>
#include <forward_list>
#include <numeric>

void demonstrate_forward_list() {
    // 1. 初始化
    std::forward_list<int> fl = {10, 20, 30};

    // 2. 头部操作
    fl.push_front(5);  // fl: {5, 10, 20, 30}
    fl.pop_front();    // fl: {10, 20, 30}
    std::cout << "Front element: " << fl.front() << std::endl;

    // 3. *_after 操作
    auto it_before = fl.before_begin(); // 获取一个特殊的 "头前" 迭代器
    fl.insert_after(it_before, 1);      // 在最前面插入1. fl: {1, 10, 20, 30}
    
    auto it = fl.begin(); // it 指向 1
    fl.insert_after(it, 99); // 在 1 之后插入 99. fl: {1, 99, 10, 20, 30}

    fl.erase_after(it); // 删除 1 之后的元素 (99). fl: {1, 10, 20, 30}

    // 4. 遍历
    std::cout << "forward_list elements: ";
    for(int val : fl) { std::cout << val << " "; }
    std::cout << std::endl;

    // 5. 获取大小 (O(N) 操作)
    size_t sz = std::distance(fl.begin(), fl.end());
    std::cout << "Size: " << sz << std::endl;

    // 6. 其他特有操作
    std::forward_list<int> fl2 = {100, 200};
    fl.splice_after(fl.begin(), fl2); // 将 fl2 的所有元素移动到 fl 中 it=begin() 的后面
                                      // fl: {1, 100, 200, 10, 20, 30}, fl2 为空
    
    fl.sort();
    fl.unique();
    fl.remove(10);
}
```

**8. 使用场景与最佳实践 (Use Cases & Best Practices)**
`std::forward_list` 是一个**高度特化**的容器，其使用场景比 `std::list` 更加狭窄。

*   **主要适用场景:**
    *   当你处理一个**非常巨大**的元素列表，并且**内存消耗**是你最关心的瓶颈时。`forward_list` 每个节点节省一个指针，在数百万个元素的情况下，节省的内存可能非常可观。
    *   当你实现的算法**天然地只需要前向遍历**时。
    *   在一些**禁用异常**的环境中，`forward_list` 的某些操作提供了更强的 `noexcept` 保证。

*   **通常不被推荐:** 在绝大多数情况下，`std::vector` 的缓存优势会胜过 `forward_list` 的 O(1) 插入/删除。即使需要链表结构，功能更全面的 `std::list` 也通常是更方便的选择，除非内存开销确实是首要问题。

*   **与 `erase-remove` 惯用法:** 在 `forward_list` 中删除满足条件的元素比较棘手，因为 `erase_after` 需要前一个元素的迭代器。标准库为此提供了专门的 `remove` 和 `remove_if` 成员函数，它们在内部高效地处理了这个问题。所以应该直接使用 `fl.remove(value)` 或 `fl.remove_if(predicate)`。

**9. 与其他容器的对比 (Comparison)**
*   **`forward_list` vs. `list`**
    *   **核心区别:** 单向 vs. 双向。
    *   **`forward_list` 优势:** 更低的内存开销（每个节点少一个指针）。
    *   **`list` 优势:** 支持双向遍历，提供 `size()`，支持 `push_back` 等尾部操作，接口更符合常规使用习惯。

*   **`forward_list` vs. `vector`**
    *   这是一个**极端**的对比。它们位于数据结构谱系的两端：`vector` 追求缓存和随机访问，`forward_list` 追求低内存和O(1)插入。选择哪个取决于你的应用对内存、随机访问和插入/删除性能的优先级排序。几乎总是 `vector` 胜出，除非内存占用是压倒一切的考量。


## queue

> https://en.cppreference.com/w/cpp/container/queue.html

**1. 概述 (Overview)**
`std::queue` 是 C++ 标准库中的一个容器适配器，它提供了一种**先进先出 (First-In, First-Out, FIFO)** 的数据结构。在队列中，元素从一端（称为**队尾 back**）被添加，从另一端（称为**队头 front**）被移除。这就像现实生活中的排队：先来的人先得到服务。

`std::queue` 本身并不实现一个完整的数据结构，它只是对一个现有的序列容器（如 `std::deque` 或 `std::list`）的接口进行**包装和限制**，使其表现出队列的行为。它屏蔽了底层容器的随机访问、迭代等功能，只暴露队列所需要的特定操作。

**2. 头文件 (Header)**
```cpp
#include <queue>
```

**3. 特点与优势 (Features & Advantages)**
*   **清晰的 FIFO 语义:** 提供了明确的 `push` (入队), `pop` (出队), `front` (访问队头), `back` (访问队尾) 接口，使得代码意图非常清晰，不易误用。
*   **灵活性:** 可以基于不同的底层容器实现。默认是 `std.deque`，但也可以选择 `std.list`。
*   **封装性:** 隐藏了底层容器的复杂接口（如迭代器、随机访问、中间插入等），只提供队列必需的功能，降低了代码复杂度和出错的可能性。

**4. 缺点与限制 (Disadvantages & Limitations)**
*   **功能受限:** 作为一个适配器，它故意限制了功能。你**不能**遍历队列中的元素，也**不能**访问除了队头和队尾之外的任何元素。如果你需要这些功能，那么 `std::queue` 就不适合你，你应该直接使用 `std::deque` 或 `std::list`。
*   **无迭代器支持:** `std::queue` 不提供 `begin()` 和 `end()` 成员函数。

**5. 内部实现与模板参数 (Internal Implementation & Template Parameters)**
`std::queue` 的模板定义如下：
```cpp
template<
    class T,
    class Container = std::deque<T>
> class queue;
```
它有两个模板参数：
1.  **`T`:** 存储的元素的类型。
2.  **`Container`:** 用于实现队列的底层容器类型。它必须是一个满足**序列容器 (SequenceContainer)** 要求的类型，并且必须提供以下操作：
    *   `front()`
    *   `back()`
    *   `push_back()`
    *   `pop_front()`
    *   `empty()`
    *   `size()`

**底层容器的选择**
*   **`std::deque<T>` (默认):** 这是默认也是最常用的选择。`std::deque` 提供了 O(1) 的 `push_back` 和 O(1) 的 `pop_front`，完美匹配队列的需求。它的随机访问能力虽然被 `std::queue` 隐藏了，但其分块的内存结构在处理大量元素时比 `std::list` 缓存更友好。
*   **`std::list<T>`:** `std::list` 也提供了 O(1) 的 `push_back` 和 `pop_front`。如果你需要绝对的指针/引用稳定性（即使底层容器发生变化），或者在某些特殊情况下（例如，你的队列元素非常大且不可移动，而你希望避免 `deque` 内部映射的重新分配），可能会选择 `list`。但在绝大多数情况下，`deque` 是更好的选择。
*   **`std::vector<T>` (不适用):** `std::vector` **不能**作为 `std::queue` 的底层容器，因为它没有提供 O(1) 的 `pop_front()` 操作。`vector` 的 `erase(begin())` 是 O(N) 的，完全不符合队列的性能要求。

`std::queue` 的内部实现非常简单，它只包含一个底层容器的成员变量，并将其公开的成员函数映射到底层容器的相应函数上。

```cpp
// 概念上的实现
template<class T, class Container = std::deque<T>>
class queue {
protected:
    Container c; // 唯一的成员变量就是底层容器

public:
    // ... 构造函数 ...

    bool empty() const { return c.empty(); }
    size_type size() const { return c.size(); }

    T& front() { return c.front(); }
    const T& front() const { return c.front(); }

    T& back() { return c.back(); }
    const T& back() const { return c.back(); }

    void push(const T& value) { c.push_back(value); }
    void push(T&& value) { c.push_back(std::move(value)); }
    
    // emplace 成员 (C++11)
    template<class... Args>
    void emplace(Args&&... args) { c.emplace_back(std::forward<Args>(args)...); }

    void pop() { c.pop_front(); } // pop() 不返回值
};
```

**6. 性能分析 (Performance Analysis)**
`std::queue` 的所有操作的性能都直接等同于其底层容器的对应操作的性能。

| 操作 (Operation) | 底层 `deque`/`list` 实现 | 时间复杂度 |
| :--- | :--- | :--- |
| `push()`/`emplace()` | `c.push_back()` | 均摊 O(1) |
| `pop()` | `c.pop_front()` | O(1) |
| `front()` | `c.front()` | O(1) |
| `back()` | `c.back()` | O(1) |
| `empty()` | `c.empty()` | O(1) |
| `size()` | `c.size()` | O(1) |

**7. 常用成员函数 (Common Member Functions)**
```cpp
#include <iostream>
#include <queue>
#include <string>
#include <list>

void demonstrate_queue() {
    // 1. 使用默认容器 std::deque
    std::queue<std::string> q_default;

    // 2. 入队 (push/emplace)
    q_default.push("Alice");
    q_default.push("Bob");
    q_default.emplace("Charlie"); // C++11, 更高效

    // 3. 访问队头和队尾
    std::cout << "Front of queue: " << q_default.front() << std::endl; // "Alice"
    std::cout << "Back of queue: " << q_default.back() << std::endl;   // "Charlie"

    // 4. 获取大小
    std::cout << "Queue size: " << q_default.size() << std::endl; // 3

    // 5. 出队 (pop)
    q_default.pop(); // "Alice" is removed. pop() returns void.
    std::cout << "Front after pop: " << q_default.front() << std::endl; // "Bob"

    // 6. 遍历和清空队列 (典型模式)
    std::cout << "Processing all elements in queue: ";
    while (!q_default.empty()) {
        std::cout << q_default.front() << " ";
        q_default.pop();
    }
    std::cout << std::endl;
    std::cout << "Queue size after processing: " << q_default.size() << std::endl; // 0

    // 7. 使用 std::list 作为底层容器
    std::queue<int, std::list<int>> q_list;
    q_list.push(100);
    q_list.push(200);
    std::cout << "Front of list-based queue: " << q_list.front() << std::endl;
}
```

**8. 使用场景与最佳实践 (Use Cases & Best Practices)**
*   **任务调度:** 当有多个任务需要按顺序处理时，可以将它们放入队列中，工作线程从队列头部取出任务并执行。
*   **广度优先搜索 (BFS):** 在图论和树的遍历中，BFS 算法天然地使用队列来存储待访问的节点。
*   **缓冲区:** 在生产者-消费者模型中，队列可以作为生产者和消费者之间的缓冲区，用于平滑生产和消费速率的差异。
*   **模拟排队系统:** 任何需要模拟真实世界排队行为的场景。
*   **何时不使用 `queue`:** 如果你需要检查或修改队列中间的元素，或者需要对元素进行迭代，那么 `std::queue` 就不适用。你应该直接使用 `std::deque` 或其他合适的容器。

**9. 与其他容器的对比 (Comparison)**
`std::queue` 是一个适配器，因此主要与其它适配器或其底层容器进行比较。

*   **`queue` vs. `stack`:**
    *   **`queue`:** FIFO (先进先出)。
    *   **`stack`:** LIFO (后进先出)。它们代表了两种最基本的数据访问模式。

*   **`queue` vs. `priority_queue`:**
    *   **`queue`:** 元素按**插入顺序**出队。
    *   **`priority_queue`:** 元素按**优先级**出队。每次 `pop()` 移除的都是当前队列中优先级最高的元素，与插入顺序无关。

*   **`queue` vs. `deque`:**
    *   `queue` 是对 `deque` 的**限制性封装**。
    *   使用 `queue` 当你只需要纯粹的 FIFO 行为，这能让代码更安全、意图更明确。
    *   使用 `deque` 当你需要 FIFO 行为，但**同时**还需要遍历、随机访问或在两端进行更复杂的操作。


## priority_queue

> https://en.cppreference.com/w/cpp/container/priority_queue.html

**1. 概述 (Overview)**
`std::priority_queue` 是一个容器适配器，它提供了一种特殊的队列——**优先队列**。与普通队列的“先进先出” (FIFO) 不同，优先队列中的元素是按照**优先级**出队的。无论元素何时入队，每次从队列中取出的（通过 `top()` 访问，通过 `pop()` 移除）总是当前队列中**优先级最高**的那个元素。

默认情况下，“优先级最高”指的是**值最大**的元素。`std::priority_queue` 通常被实现为一种称为**堆 (Heap)** 的数据结构，这使得它能高效地完成插入和删除最高优先级元素的操作。

和 `std::queue` 一样，它也是一个适配器，通过包装和限制一个底层容器的接口来实现其功能。

**2. 头文件 (Header)**
```cpp
#include <queue>
```
是的，它和 `std::queue` 在同一个头文件中。

**3. 特点与优势 (Features & Advantages)**
*   **自动排序:** 元素在逻辑上总是保持有序（按优先级）。你不需要手动排序，容器会自动维护其内部结构，以保证 `top()` 总是返回优先级最高的元素。
*   **高效操作:** 插入元素 (`push`) 和移除最高优先级元素 (`pop`) 的时间复杂度都是对数时间 O(log N)，这对于需要频繁更新和查询极值的场景非常高效。
*   **接口简洁:** 提供了清晰的 `push`, `pop`, `top` 接口，专注于优先队列的核心功能。
*   **可定制的比较逻辑:** 可以通过模板参数轻松地自定义“优先级”的定义（例如，实现小顶堆或对自定义对象进行排序）。

**4. 缺点与限制 (Disadvantages & Limitations)**
*   **功能受限:** 作为一个适配器，它只允许访问**最高优先级**的元素 (`top()`)。你不能访问、修改或遍历队列中的任何其他元素。
*   **无迭代器支持:** 不提供 `begin()` 和 `end()`，因此无法直接遍历。
*   **没有 `find` 或 `update` 操作:** 不支持查找特定元素，也不支持在不移除元素的情况下更新其优先级。如果需要更新优先级，通常的做法是插入一个带有新优先级的副本，并在处理时忽略旧版本，或者使用更复杂的数据结构（如 Boost 库中的 `d_ary_heap`）。

**5. 内部实现与模板参数 (Internal Implementation & Template Parameters)**
`std::priority_queue` 的模板定义比 `std::queue` 更复杂：
```cpp
template<
    class T,
    class Container = std::vector<T>,
    class Compare = std::less<typename Container::value_type>
> class priority_queue;
```
它有三个模板参数：
1.  **`T`:** 存储的元素的类型。
2.  **`Container`:** 用于实现优先队列的底层容器。它必须是一个支持随机访问迭代器 (RandomAccessIterator) 的序列容器，并提供 `front()`, `push_back()`, `pop_back()` 操作。
    *   **`std::vector<T>` (默认):** 这是默认且最常用的选择。`vector` 的连续内存布局对于堆结构的缓存局部性非常友好。
    *   **`std::deque<T>`:** 也可以使用 `deque`。在某些极端的插入模式下，它可能比 `vector` 的重新分配表现得更好，但通常 `vector` 的缓存优势会胜出。
3.  **`Compare`:** 一个二元比较函数对象（或函数指针），用于定义元素间的优先级顺序。`Compare(a, b)` 返回 `true` 意味着 `a` 的优先级**低于** `b`。
    *   **`std::less<T>` (默认):** 这是默认的比较器。`std::less<T>` 实现了 `<` 操作。因此，如果 `a < b`，则 `b` 的优先级更高。这导致了 `std::priority_queue` 默认是一个**大顶堆 (Max-Heap)**，`top()` 总是返回最大的元素。

**内部实现：堆 (Heap)**
`std::priority_queue` 内部逻辑上是一个**二叉堆**，但物理上是使用底层容器（如 `std::vector`）来存储的。这种映射关系非常巧妙：
对于存储在数组索引 `i` 的一个节点：
*   其父节点位于索引 `(i - 1) / 2`。
*   其左子节点位于索引 `2 * i + 1`。
*   其右子节点位于索引 `2 * i + 2`。

这种结构使得我们可以在一个连续数组上模拟出树形结构，而无需使用指针，从而获得了极佳的缓存性能。

*   **`push(value)` 操作:**
    1.  将新元素 `value` 添加到数组的末尾 (`c.push_back(value)`)。
    2.  执行一次**上浮 (sift-up)** 操作：将新元素与其父节点比较，如果新元素的优先级更高，则交换它们。重复此过程，直到新元素到达根节点，或其优先级不再高于其父节点。此过程复杂度为 O(log N)。

*   **`pop()` 操作:**
    1.  将数组的第一个元素（根节点，即 `top()`）与最后一个元素交换。
    2.  移除数组的最后一个元素 (`c.pop_back()`)。
    3.  现在根节点是错误的元素，执行一次**下沉 (sift-down)** 操作：将新的根节点与其子节点中优先级最高的一个比较，如果子节点的优先级更高，则交换它们。重复此过程，直到该元素到达叶子节点，或其优先级不低于其子节点。此过程复杂度也为 O(log N)。

**6. 性能分析 (Performance Analysis)**
| 操作 (Operation) | 底层实现 (Heap on Vector/Deque) | 时间复杂度 |
| :--- | :--- | :--- |
| `push()`/`emplace()` | 上浮 (Sift-up) | O(log N) |
| `pop()` | 下沉 (Sift-down) | O(log N) |
| `top()` | 访问底层容器的 `front()` | O(1) |
| `empty()` | `c.empty()` | O(1) |
| `size()` | `c.size()` | O(1) |

**7. 常用成员函数与用法 (Common Member Functions & Usage)**
```cpp
#include <iostream>
#include <queue>
#include <vector>
#include <functional> // for std::greater

void demonstrate_priority_queue() {
    // 1. 默认大顶堆 (Max-Heap)
    std::priority_queue<int> max_heap;
    max_heap.push(30);
    max_heap.push(100);
    max_heap.push(20);
    
    std::cout << "Max-heap top: " << max_heap.top() << std::endl; // 100

    // 2. 小顶堆 (Min-Heap)
    // 需要提供底层容器类型和比较器类型
    std::priority_queue<int, std::vector<int>, std::greater<int>> min_heap;
    min_heap.push(30);
    min_heap.push(100);
    min_heap.push(20);

    std::cout << "Min-heap top: " << min_heap.top() << std::endl; // 20

    // 3. 处理队列
    std::cout << "Processing max-heap: ";
    while (!max_heap.empty()) {
        std::cout << max_heap.top() << " ";
        max_heap.pop();
    }
    std::cout << std::endl;

    // 4. 自定义类型的优先队列
    struct Task {
        int priority;
        std::string name;

        // 为大顶堆重载 < 操作符
        bool operator<(const Task& other) const {
            return priority < other.priority;
        }
    };

    std::priority_queue<Task> task_queue;
    task_queue.push({5, "Low priority task"});
    task_queue.push({10, "High priority task"});
    task_queue.push({7, "Medium priority task"});

    std::cout << "Highest priority task: " << task_queue.top().name << std::endl; // "High priority task"

    // 使用 lambda 自定义比较逻辑 (更灵活)
    auto cmp = [](const Task& a, const Task& b) {
        return a.priority > b.priority; // 注意这里是 >，实现小顶堆
    };
    std::priority_queue<Task, std::vector<Task>, decltype(cmp)> min_task_queue(cmp);
    min_task_queue.push({5, "Low"});
    min_task_queue.push({10, "High"});
    std::cout << "Lowest priority task: " << min_task_queue.top().name << std::endl; // "Low"
}
```

**8. 使用场景与最佳实践 (Use Cases & Best Practices)**
*   **Top-K 问题:** 寻找一个大数据流中的 K 个最大或最小的元素。可以维护一个大小为 K 的小顶堆（找 Top-K 大）或大顶堆（找 Top-K 小）。
*   **Dijkstra 算法和 A* 算法:** 在图论中，这些寻找最短路径的算法需要一个优先队列来存储待访问的节点，并优先访问“距离”最近的节点。
*   **事件驱动模拟:** 在模拟系统中，可以用优先队列来管理未来的事件，队列顶部总是下一个即将发生的事件。
*   **任务调度器:** 操作系统或应用程序中，需要根据任务的优先级来决定下一个执行哪个任务。
*   **霍夫曼编码:** 构建霍夫曼树时，需要一个优先队列来反复合并频率最低的两个节点。

**9. 与其他容器的对比 (Comparison)**
*   **`priority_queue` vs. `std::set` / `std::map`:**
    *   `set`/`map`（红黑树实现）也是有序的数据结构。它们也支持 O(log N) 的插入和删除。
    *   **`priority_queue` 优势:** 通常更快（常数因子更小），因为它基于堆，缓存局部性更好。内存开销也更低。
    *   **`set`/`map` 优势:** `set`/`map` 允许**遍历所有元素**，并支持**查找和删除任意元素**，而 `priority_queue` 只能访问 `top()`。
    *   **选择:** 如果你只需要反复获取和移除最大/最小元素，用 `priority_queue`。如果你需要维护一个有序集合，并需要查找、遍历或删除任意元素，用 `set` 或 `map`。

*   **`priority_queue` vs. 手动排序的 `vector`:**
    *   如果你需要频繁地插入新元素并保持集合有序，`priority_queue` (O(log N) 插入) 远胜于每次都对 `vector` 进行完全排序 (O(N log N))。
    *   如果你的操作模式是：先一次性插入所有元素，然后反复查询最大值，那么先对 `vector` 排序一次 (O(N log N)) 然后访问 `back()` (O(1)) 可能是更好的选择。



## stack

> https://en.cppreference.com/w/cpp/container/stack.html

**1. 概述 (Overview)**
`std::stack` 是 C++ 标准库中的一个容器适配器，它实现了**后进先出 (Last-In, First-Out, LIFO)** 的数据结构。在栈中，元素的添加（称为**入栈 push**）和移除（称为**出栈 pop**）都发生在同一端，这一端被称为栈顶 (top)。

这就像一叠盘子：你总是把新盘子放在最上面，也总是从最上面取走盘子。最后放上去的盘子总是最先被取走。

与 `std::queue` 类似，`std::stack` 也是通过包装和限制一个现有的序列容器的接口，使其表现出栈的行为。它屏蔽了底层容器的其他所有功能，只提供栈所需的核心操作。

**2. 头文件 (Header)**
```cpp
#include <stack>
```

**3. 特点与优势 (Features & Advantages)**
*   **清晰的 LIFO 语义:** 提供了明确的 `push` (入栈), `pop` (出栈), `top` (访问栈顶) 接口，使得代码意图非常清晰，符合栈的抽象数据类型。
*   **封装性:** 隐藏了底层容器的复杂性，防止了对非栈顶元素的意外访问或修改，增强了代码的健壮性。
*   **灵活性:** 可以基于不同的底层容器实现，默认是 `std.deque`，但也可以是 `std::vector` 或 `std::list`。

**4. 缺点与限制 (Disadvantages & Limitations)**
*   **功能受限:** 作为一个适配器，它故意限制了功能。你**不能**遍历栈中的元素，也**不能**访问除了栈顶之外的任何元素。如果需要这些功能，`std::stack` 并不适合，你应该直接使用其底层容器。
*   **无迭代器支持:** `std::stack` 不提供 `begin()` 和 `end()` 成员函数。

**5. 内部实现与模板参数 (Internal Implementation & Template Parameters)**
`std::stack` 的模板定义如下：
```cpp
template<
    class T,
    class Container = std::deque<T>
> class stack;
```
它有两个模板参数：
1.  **`T`:** 存储的元素的类型。
2.  **`Container`:** 用于实现栈的底层容器类型。它必须是一个满足**后端序列 (BackInsertionSequence)** 要求的类型，即必须提供以下操作：
    *   `back()`
    *   `push_back()`
    *   `pop_back()`
    *   `empty()`
    *   `size()`

**底层容器的选择**
*   **`std::deque<T>` (默认):** 这是默认选择。`std::deque` 提供了 O(1) 的 `push_back` 和 `pop_back`，完美满足栈的需求。虽然它的双端操作能力在这里被浪费了一半，但它在处理大量元素时，其分块内存模型有时比 `vector` 的单次大块分配更有弹性。
*   **`std::vector<T>`:** `std::vector` 也是一个绝佳的选择，甚至在很多情况下比 `deque` 更好。它也提供了均摊 O(1) 的 `push_back` 和 O(1) 的 `pop_back`。由于其完美的连续内存布局，`vector` 的缓存性能通常是最好的。如果你不需要处理可能导致栈溢出的海量元素，`vector` 可能是理论上性能最高的选择。
*   **`std::list<T>`:** `std::list` 也提供了 O(1) 的 `push_back` 和 `pop_back`。选择它的理由非常罕见，通常只在元素的移动/拷贝成本极高，或者对元素的地址稳定性有极端要求时才考虑。

`std::stack` 的内部实现与 `std::queue` 类似，只是将其接口映射到底层容器的 `back` 相关操作。

```cpp
// 概念上的实现
template<class T, class Container = std::deque<T>>
class stack {
protected:
    Container c; // 唯一的成员变量

public:
    // ... 构造函数 ...

    bool empty() const { return c.empty(); }
    size_type size() const { return c.size(); }

    T& top() { return c.back(); }
    const T& top() const { return c.back(); }

    void push(const T& value) { c.push_back(value); }
    void push(T&& value) { c.push_back(std::move(value)); }
    
    template<class... Args>
    void emplace(Args&&... args) { c.emplace_back(std::forward<Args>(args)...); }

    void pop() { c.pop_back(); } // pop() 不返回值
};
```

**6. 性能分析 (Performance Analysis)**
`std::stack` 的所有操作的性能都直接等同于其底层容器的**尾部操作**的性能。

| 操作 (Operation) | 底层 `deque`/`vector`/`list` 实现 | 时间复杂度 |
| :--- | :--- | :--- |
| `push()`/`emplace()` | `c.push_back()` | 均摊 O(1) (`vector`/`deque`), O(1) (`list`) |
| `pop()` | `c.pop_back()` | O(1) |
| `top()` | `c.back()` | O(1) |
| `empty()` | `c.empty()` | O(1) |
| `size()` | `c.size()` | O(1) |

**7. 常用成员函数 (Common Member Functions)**
```cpp
#include <iostream>
#include <stack>
#include <string>
#include <vector>

// 检查括号是否匹配的经典例子
bool areBracketsBalanced(const std::string& expr) {
    std::stack<char> s;
    for (char c : expr) {
        if (c == '(' || c == '{' || c == '[') {
            s.push(c);
        } else if (c == ')' || c == '}' || c == ']') {
            if (s.empty()) return false; // 右括号多于左括号

            char top = s.top();
            s.pop();

            if ((c == ')' && top != '(') ||
                (c == '}' && top != '{') ||
                (c == ']' && top != '[')) {
                return false; // 括号不匹配
            }
        }
    }
    return s.empty(); // 如果栈为空，则所有左括号都被匹配了
}

void demonstrate_stack() {
    // 1. 使用默认容器 std::deque
    std::stack<int> s_default;

    // 2. 入栈 (push)
    s_default.push(10);
    s_default.push(20);
    s_default.emplace(30); // C++11

    // 3. 访问栈顶
    std::cout << "Top of stack: " << s_default.top() << std::endl; // 30

    // 4. 出栈 (pop)
    s_default.pop(); // 30 is removed
    std::cout << "Top after pop: " << s_default.top() << std::endl; // 20

    // 5. 遍历和清空栈 (典型模式)
    std::cout << "Processing all elements in stack: ";
    while (!s_default.empty()) {
        std::cout << s_default.top() << " ";
        s_default.pop();
    }
    std::cout << std::endl;
    std::cout << "Stack size after processing: " << s_default.size() << std::endl; // 0

    // 使用示例函数
    std::cout << "()[]{} is balanced: " << std::boolalpha << areBracketsBalanced("()[]{}") << std::endl;
    std::cout << "([)] is balanced: " << std::boolalpha << areBracketsBalanced("([)]") << std::endl;
}
```

**8. 使用场景与最佳实践 (Use Cases & Best Practices)**
栈是一种非常基础且强大的数据结构，应用广泛。
*   **函数调用栈:** 编程语言自身就使用栈来管理函数调用。每次调用函数，其参数、返回地址和局部变量都被压入栈中；函数返回时，这些信息被弹出。
*   **表达式求值:** 在中缀表达式转后缀（逆波兰表示法）以及后缀表达式的求值中，栈是核心数据结构。
*   **括号匹配:** 如示例代码所示，用于检查各种括号是否成对出现。
*   **深度优先搜索 (DFS):** 在图论和树的遍历中，DFS 算法的递归版本隐式地使用了函数调用栈，其迭代版本则显式地使用 `std::stack` 来存储待访问的节点。
*   **撤销/重做 (Undo/Redo) 功能:** 许多应用程序使用两个栈来实现撤销和重做。执行一个操作时，其逆操作被压入“撤销栈”。执行撤销时，从撤销栈弹出一个操作执行，并将其正向操作压入“重做栈”。
*   **浏览器历史:** 浏览器的“后退”功能可以通过一个栈来实现，每次访问新页面就将其压栈。

**9. 与其他容器的对比 (Comparison)**
*   **`stack` vs. `queue`:**
    *   **`stack`:** LIFO (后进先出)。
    *   **`queue`:** FIFO (先进先出)。它们代表了两种最基础、最常用的数据访问模式。

*   **`stack` vs. `vector`/`deque`/`list`:**
    *   `stack` 是对这些容器的**限制性封装**。
    *   使用 `stack` 当你只需要纯粹的 LIFO 行为。这能让代码意图更明确，并防止对栈内非顶部元素的误操作。
    *   直接使用底层容器，当你除了 LIFO 行为外，还需要迭代、随机访问或其他更复杂的操作。


## map

> https://en.cppreference.com/w/cpp/container/map.html

**1. 概述 (Overview)**
`std::map` 是 C++ 标准库中的一个关联容器，它存储的元素是**键值对 (Key-Value pairs)**。每个元素都由一个唯一的**键 (Key)** 和一个与之关联的**值 (Value)** 组成。

`std::map` 最核心的特性是它会根据**键**自动对元素进行**排序**。默认情况下，它按键的升序进行排序。这种有序性是通过一种自平衡二叉搜索树（通常是**红黑树 (Red-Black Tree)**）的内部数据结构来实现的。

这使得 `std::map` 在查找、插入和删除操作上都能提供稳定且高效的对数时间复杂度 O(log N)。

**2. 头文件 (Header)**
```cpp
#include <map>
```

**3. 特点与优势 (Features & Advantages)**
*   **键的唯一性:** 每个键在 `map` 中必须是唯一的。试图插入一个已存在的键不会改变原有的键值对（除非使用 `operator[]`）。
*   **自动排序:** 元素总是根据键保持有序。这使得 `std::map` 非常适合需要按顺序遍历或查找某个范围内的元素的场景。
*   **高效的操作:** 查找、插入和删除操作的平均和最坏时间复杂度都是 O(log N)。
*   **键值对结构:** `std::map` 的元素类型是 `std::pair<const Key, T>`。注意键 `Key` 是 `const` 的，这意味着你不能修改一个已插入元素的键，只能删除它然后插入一个新的。
*   **强大的查找功能:** 提供了 `find()`, `lower_bound()`, `upper_bound()` 等方法，可以高效地进行点查找和范围查找。

**4. 缺点与限制 (Disadvantages & Limitations)**
*   **非连续内存:** 作为基于树的结构，其节点在内存中是分散存储的，这导致了较差的缓存性能，遍历速度通常慢于 `std::vector`。
*   **更高的内存开销:** 每个节点除了存储键值对外，还需要存储额外的指针（指向父节点、左子节点、右子节点）和颜色信息（红/黑），因此内存占用比 `std::unordered_map` 或 `std::vector` 要高。
*   **对数时间复杂度:** 虽然 O(log N) 很快，但对于追求极致性能的场景（如量化交易中的低延迟哈希表查找），它不如 `std::unordered_map` 的均摊 O(1) 快。

**5. 内部实现与内存布局 (Internal Implementation & Memory Layout)**
`std::map` 的标准要求它是一个有序的关联容器，但并未强制规定必须使用红黑树。然而，在所有主流的标准库实现中（如 GCC, Clang, MSVC），`std::map` 都采用**红黑树 (Red-Black Tree)** 作为其底层数据结构。

**红黑树 (Red-Black Tree)**
红黑树是一种自平衡的二叉搜索树，它通过满足以下几条规则来确保树的高度近似为 O(log N)，从而保证了操作的效率：
1.  每个节点要么是红色，要么是黑色。
2.  根节点是黑色的。
3.  所有叶子节点（NIL 节点，即空指针）都是黑色的。
4.  如果一个节点是红色的，那么它的两个子节点都是黑色的（不能有连续的红色节点）。
5.  从任一节点到其所有后代叶子节点的路径上，均包含相同数目的黑色节点。

在插入和删除节点后，可能会破坏这些规则。此时，红黑树会通过一系列的**旋转 (Rotation)** 和**颜色变换 (Recoloring)** 操作来重新恢复平衡，这些调整操作的时间复杂度也是 O(log N)。

**内存布局**
*   **节点 (Node):** 每个节点在**堆上**独立分配，通常包含：
    *   `std::pair<const Key, T> data`: 存储的键值对。
    *   `Node* parent`: 指向父节点的指针。
    *   `Node* left`: 指向左子节点的指针。
    *   `Node* right`: 指向右子节点的指针。
    *   `Color color`: 节点的颜色（红色或黑色）。
*   **Map 对象:** `std::map` 对象本身很小，通常只包含一个指向根节点的指针、一个 `size` 计数器以及比较函数对象的副本。

内存布局是典型的树状结构，节点散布在堆内存中，通过指针相互连接。

**6. 性能分析 (Performance Analysis)**
| 操作 (Operation) | 底层实现 (Red-Black Tree) | 时间复杂度 |
| :--- | :--- | :--- |
| 插入 (`insert`, `emplace`) | 树的查找 + 节点插入 + 平衡调整 | O(log N) |
| 查找 (`find`, `count`) | 树的查找 | O(log N) |
| 删除 (`erase`) | 树的查找 + 节点删除 + 平衡调整 | O(log N) |
| 访问 (`operator[]`, `at`) | 树的查找 (可能伴随插入) | O(log N) |
| 遍历 (Iteration) | 中序遍历 (In-order traversal) | O(N) |
| 范围查找 (`lower_bound`, `upper_bound`) | 树的查找 | O(log N) |

**7. 常用成员函数与用法 (Common Member Functions & Usage)**
```cpp
#include <iostream>
#include <map>
#include <string>

void demonstrate_map() {
    // 1. 初始化
    std::map<std::string, int> student_scores;

    // 2. 插入元素
    // 使用 insert 和 std::pair
    student_scores.insert(std::make_pair("Alice", 95));
    // 使用 insert 和列表初始化 (C++11)
    student_scores.insert({"Bob", 88});
    // 使用 emplace (C++11, 更高效，避免临时对象)
    student_scores.emplace("Charlie", 92);

    // 3. 访问和修改元素 (operator[])
    // operator[] 是一个强大的操作:
    // 如果键 "David" 不存在，它会先插入一个 "David" -> int() (即0)，然后返回对这个0的引用。
    student_scores["David"] = 75;
    std::cout << "David's score (initially): " << student_scores["David"] << std::endl;
    // 如果键 "Alice" 存在，它直接返回对其值的引用。
    student_scores["Alice"] = 96; // 更新 Alice 的分数

    // 4. 安全访问 (at)
    // .at() 只在键存在时返回引用，否则抛出 std::out_of_range 异常。
    try {
        std::cout << "Charlie's score: " << student_scores.at("Charlie") << std::endl;
        // student_scores.at("Eve"); // 这会抛出异常
    } catch (const std::out_of_range& e) {
        std::cout << e.what() << std::endl;
    }

    // 5. 查找元素 (find)
    auto it = student_scores.find("Bob");
    if (it != student_scores.end()) {
        // it 是一个指向 std::pair<const std::string, int> 的迭代器
        std::cout << "Found Bob, score: " << it->second << std::endl;
    } else {
        std::cout << "Bob not found." << std::endl;
    }

    // 6. 删除元素
    student_scores.erase("David"); // 按键删除
    it = student_scores.find("Charlie");
    if (it != student_scores.end()) {
        student_scores.erase(it); // 按迭代器删除
    }
    
    // 7. 遍历 (自动按键排序)
    std::cout << "\nAll students (sorted by name):" << std::endl;
    for (const auto& pair : student_scores) {
        std::cout << pair.first << ": " << pair.second << std::endl;
    }
    // 输出会是 Alice, Bob (因为 Charlie 和 David 已被删除)

    // 8. 范围查找
    std::map<int, char> m = {{1, 'a'}, {3, 'c'}, {5, 'e'}, {7, 'g'}};
    // lower_bound: 第一个不小于给定键的元素
    auto lb = m.lower_bound(3); // 指向 {3, 'c'}
    // upper_bound: 第一个大于给定键的元素
    auto ub = m.upper_bound(3); // 指向 {5, 'e'}
    std::cout << "lower_bound(3): " << lb->first << std::endl;
    std::cout << "upper_bound(3): " << ub->first << std::endl;
}
```

**8. 使用场景与最佳实践 (Use Cases & Best Practices)**
*   **需要键值对存储且关心顺序:** 当你需要一个字典或查找表，并且需要能够按键的顺序进行遍历或执行范围查询时，`std::map` 是完美的选择。例如，按时间戳存储日志，按字母顺序存储单词和其定义。
*   **`operator[]` vs. `insert()`:**
    *   使用 `operator[]` 当你希望**如果键不存在就创建它**，并且不关心是否覆盖了旧值。这对于计数（如词频统计）非常方便。
    *   使用 `insert()` 当你**不希望覆盖已存在的值**。`insert()` 会返回一个 `std::pair<iterator, bool>`，其中 `bool` 值表明插入是否成功。
    *   使用 `emplace()` 替代 `insert()` 以获得更好的性能，因为它可以在节点内就地构造对象。
*   **查找优于访问:** 如果你不确定一个键是否存在，并且不希望在它不存在时创建它，应该先用 `find()` 或 C++20 的 `contains()` 检查，而不是直接用 `operator[]`。

**9. 与其他容器的对比 (Comparison)**
*   **`map` vs. `unordered_map`:**
    *   这是最重要的对比。
    *   **`map` (红黑树):** 元素有序，操作复杂度 O(log N)，内存开销大，对哈希函数不敏感。
    *   **`unordered_map` (哈希表):** 元素无序，操作均摊复杂度 O(1)（最坏 O(N)），内存开销相对较小，对哈希函数性能敏感。
    *   **选择:**
        *   需要有序性或范围查找 -> `map`。
        *   只关心快速的点查找、插入、删除，不关心顺序 -> `unordered_map`（通常是首选）。
        *   Key 类型没有好的哈希函数，但有 `<` 比较 -> `map`。

*   **`map` vs. `vector<pair<K, V>>` + `std::sort`:**
    *   **`map`:** 动态维护有序性，每次插入/删除都是 O(log N)。
    *   **`vector`:** 如果你先一次性插入所有数据，然后排序一次 (O(N log N))，之后只进行查找（`std::lower_bound` 也是 O(log N)），那么 `vector` 的性能可能更好，因为它缓存友好。但只要有新的插入或删除，就需要重新排序或移动元素，成本很高。



## multimap

> http://en.cppreference.com/w/cpp/container/multimap.html

**1. 概述 (Overview)**
`std::multimap` 是一个关联容器，它存储的元素是**键值对 (Key-Value pairs)**，并且会根据**键**自动进行排序。它与 `std::map` 的核心区别在于：`std::multimap` **允许存储具有相同键的多个元素**。

就像 `std::map` 一样，`std::multimap` 内部通常也由一个自平衡二叉搜索树（如**红黑树**）实现，以保证高效的查找、插入和删除操作。所有具有相同键的元素在遍历时会相邻出现。

`std::multimap` 提供了一种有效的方式来组织一对多的关系，例如，一个作者（键）可以对应多本书（值）。

**2. 头文件 (Header)**
```cpp
#include <map>
```
是的，它和 `std::map` 在同一个头文件中。

**3. 特点与优势 (Features & Advantages)**
*   **允许键重复:** 这是它与 `std::map` 最根本的区别。你可以为一个键关联多个不同的值。
*   **自动排序:** 元素总是根据键保持有序。具有相同键的元素会聚集在一起，但它们之间的相对顺序是不确定的（取决于插入顺序和具体实现）。
*   **高效操作:** 查找、插入和删除操作的平均和最坏时间复杂度都是 O(log N)。
*   **专为多值查找设计:** 提供了 `equal_range()`, `lower_bound()`, `upper_bound()` 和 `count()` 等方法，可以方便地查找和处理与单个键关联的所有值。

**4. 缺点与限制 (Disadvantages & Limitations)**
*   **没有 `operator[]` 和 `.at()`:** 由于一个键可以对应多个值，使用像 `mm[key]` 这样的语法会产生歧义（应该返回哪个值的引用？）。因此，`std::multimap` **不提供** `operator[]` 访问器和 `.at()` 成员函数。你必须使用 `find()` 或 `equal_range()` 等方法来访问元素。
*   **与 `map` 相同的性能/内存缺点:**
    *   **非连续内存:** 基于树的结构，缓存性能较差。
    *   **高内存开销:** 每个节点需要存储指针和颜色信息。
    *   **对数时间复杂度:** 性能不如 `std::unordered_multimap` 的均摊 O(1)。

**5. 内部实现与内存布局 (Internal Implementation & Memory Layout)**
`std::multimap` 的内部实现与 `std::map` 几乎完全相同，通常都是**红黑树**。唯一的区别在于红黑树处理键重复的策略。

在 `std::map` 中，当插入一个已存在的键时，插入操作会失败。而在 `std::multimap` 中，树的插入算法被修改为**总是允许插入**，即使键已存在。新节点会被放置在树中与现有相同键的节点“旁边”的位置，以维持中序遍历的有序性。

例如，一个简单的二叉搜索树插入算法可能会将等值的键始终放在右子树（或左子树），以确保所有相同键的节点在遍历时是连续的。红黑树的平衡调整算法会确保在这些操作之后树依然保持平衡。

其内存布局、节点结构（包含键值对、父/子指针、颜色）和 `std::map` 完全一致。

**6. 性能分析 (Performance Analysis)**
`std::multimap` 的性能特征与 `std::map` 相同。

| 操作 (Operation) | 底层实现 (Red-Black Tree) | 时间复杂度 |
| :--- | :--- | :--- |
| 插入 (`insert`, `emplace`) | 树的查找 + 节点插入 + 平衡调整 | O(log N) |
| 查找一个键 (`find`) | 树的查找 | O(log N) |
| 删除一个键的所有条目 (`erase(key)`) | 查找 + 遍历删除 | O(log N + K)，K 是被删除的元素数量 |
| 删除单个条目 (`erase(iterator)`) | 节点删除 + 平衡调整 | 均摊 O(1) (C++11后)，最坏 O(log N) |
| 统计键数量 (`count`) | 查找 + 遍历计数 | O(log N + K)，K 是具有该键的元素数量 |
| 范围查找 (`equal_range`, `lower_bound`, `upper_bound`) | 树的查找 | O(log N) |

**7. 常用成员函数与用法 (Common Member Functions & Usage)**
由于没有 `operator[]`，与 `std::multimap` 的交互方式与 `std::map` 有显著不同。

```cpp
#include <iostream>
#include <map>
#include <string>

void demonstrate_multimap() {
    // 1. 初始化和插入
    std::multimap<std::string, std::string> author_books;

    // 一个作者可以有多本书
    author_books.insert({"George R.R. Martin", "A Game of Thrones"});
    author_books.insert({"George R.R. Martin", "A Clash of Kings"});
    author_books.emplace("J.R.R. Tolkien", "The Hobbit");
    author_books.emplace("J.R.R. Tolkien", "The Lord of the Rings");
    author_books.emplace("Frank Herbert", "Dune");

    // 2. 查找和处理一个键对应的所有值
    std::string author_to_find = "George R.R. Martin";

    // 方法一: 使用 find 和迭代器递增
    std::cout << "Books by " << author_to_find << " (method 1):" << std::endl;
    for (auto it = author_books.find(author_to_find);
         it != author_books.end() && it->first == author_to_find;
         ++it) {
        std::cout << " - " << it->second << std::endl;
    }

    // 方法二: 使用 count 和 find (效率稍低，因为 count 会遍历)
    int book_count = author_books.count(author_to_find);
    std::cout << "\nFound " << book_count << " books by " << author_to_find << " (method 2)." << std::endl;

    // 方法三: 使用 equal_range (最高效、最地道的方法)
    std::cout << "\nBooks by J.R.R. Tolkien (method 3 - equal_range):" << std::endl;
    auto range = author_books.equal_range("J.R.R. Tolkien");
    // range 是一个 std::pair<iterator, iterator>
    // range.first 是第一个匹配元素的迭代器
    // range.second 是最后一个匹配元素之后一个位置的迭代器
    for (auto it = range.first; it != range.second; ++it) {
        std::cout << " - " << it->second << std::endl;
    }

    // 3. 删除
    // 删除所有键为 "Frank Herbert" 的条目
    size_t num_erased = author_books.erase("Frank Herbert");
    std::cout << "\nErased " << num_erased << " entries for Frank Herbert." << std::endl;

    // 4. 遍历整个 multimap (自动按作者名排序)
    std::cout << "\nAll entries in the multimap:" << std::endl;
    for (const auto& pair : author_books) {
        std::cout << pair.first << " -> " << pair.second << std::endl;
    }
}
```

**8. 使用场景与最佳实践 (Use Cases & Best Practices)**
*   **一对多关系的建模:** 这是 `std::multimap` 的核心用例。任何你需要将一个键映射到多个值，并希望这些键保持有序的情况。
    *   **索引:** 建立一个单词到其在文档中出现的所有页码的索引。`multimap<string, int>`。
    *   **电话簿:** 一个人名可以对应多个电话号码。`multimap<string, string>`。
    *   **分类:** 按类别（键）组织项目（值）。
*   **使用 `equal_range`:** 当需要获取与某个键关联的所有值时，`equal_range` 是最标准、最高效的方法。它只需要两次对数时间的查找就能确定范围的起点和终点。
*   **替代方案:** 有时，使用 `std::map<Key, std::vector<Value>>` 也可以实现一对多的关系。
    *   **`map<Key, vector<Value>>` 的优点:**
        *   概念上更清晰地将“一个键”与“一个值的集合”关联起来。
        *   可以轻松地对单个键的值集合进行操作（例如，排序、删除某个特定的值）。
    *   **`multimap` 的优点:**
        *   插入单个值更简单直接（`multimap.insert({key, value})` vs `map[key].push_back(value)`）。
        *   遍历所有值（无论键是什么）更自然 (`for(const auto& pair : multimap)`）。
        *   如果每个键平均只对应很少的值（例如1-2个），`multimap` 可能有更低的内存开销，因为它避免了为每个键都创建一个 `std::vector` 对象（即使该 vector 只包含一个元素）。

**9. 与其他容器的对比 (Comparison)**
*   **`multimap` vs. `map`:**
    *   **核心区别:** `multimap` 允许键重复，`map` 不允许。
    *   **接口差异:** `multimap` 没有 `operator[]` 和 `.at()`。

*   **`multimap` vs. `unordered_multimap`:**
    *   这是 `map` vs `unordered_map` 对比的直接延伸。
    *   **`multimap` (红黑树):** 元素按键有序，操作 O(log N)。
    *   **`unordered_multimap` (哈希表):** 元素无序，操作均摊 O(1)。
    *   **选择:**
        *   需要有序性或范围查找 -> `multimap`。
        *   只关心快速的点查找和插入，不关心顺序 -> `unordered_multimap`。



## set

> https://en.cppreference.com/w/cpp/container/set.html

**1. 概述 (Overview)**
`std::set` 是 C++ 标准库中的一个关联容器，用于存储**唯一的**元素。与 `std::map` 存储键值对不同，`std::set` 只存储**键**本身。这些键（也就是 set 中的元素）是唯一的，并且容器会自动根据元素的值进行**排序**。

`std::set` 的行为可以看作是一个只有键、没有值的 `std::map`。它同样是基于自平衡二叉搜索树（通常是**红黑树**）实现的，因此在查找、插入和删除操作上都具有高效的对数时间复杂度 O(log N)。

`std::set` 非常适合用于需要快速判断一个元素是否存在于一个集合中，或者需要一个时刻保持有序的、不含重复元素的集合。

**2. 头文件 (Header)**
```cpp
#include <set>
```

**3. 特点与优势 (Features & Advantages)**
*   **元素唯一性:** `std::set` 中不允许存在重复的元素。任何试图插入已存在元素的尝试都会被忽略。
*   **自动排序:** 元素总是根据其值保持有序（默认升序）。这使得遍历 `set` 时可以得到一个有序序列。
*   **高效的操作:** 查找、插入和删除操作的时间复杂度都是 O(log N)。
*   **元素不可变性:** `std::set` 中的元素是 `const` 的。一旦一个元素被插入到 `set` 中，你**不能修改它**。这是因为修改元素的值会破坏 `set` 的内部有序结构。要“修改”一个元素，唯一的方法是先 `erase` 旧元素，再 `insert` 新元素。
*   **集合运算:** `std::set` 的有序性使得它非常适合与 `<algorithm>` 中的集合运算函数（如 `std::set_intersection`, `std::set_union`, `std::set_difference`）配合使用。

**4. 缺点与限制 (Disadvantages & Limitations)**
*   **与 `map` 相同的性能/内存缺点:**
    *   **非连续内存:** 基于树的结构，缓存性能不如 `std::vector`。
    *   **高内存开销:** 每个节点都需要存储额外的指针和颜色信息。
    *   **对数时间复杂度:** 不如 `std::unordered_set` 的均摊 O(1) 快。
*   **元素不可修改:** 无法直接修改已存在元素的值，必须通过“删除+插入”的方式完成。

**5. 内部实现与内存布局 (Internal Implementation & Memory Layout)**
`std::set` 的内部实现与 `std::map` 几乎完全相同，都是基于**红黑树**。

关键区别在于节点存储的数据：
*   `std::map` 的节点存储 `std::pair<const Key, T>`。
*   `std::set` 的节点存储 `const Key`。

在 `std::set` 的标准库实现中，为了代码复用，通常会用一个通用的红黑树模板来实现 `std::map` 和 `std::set`。一种常见的技巧是，`std::set<Key>` 在内部被实现为 `std::map<Key, Key>`，其中值部分被忽略。更现代的实现可能会直接特化树的节点，使其只存储一个值。

无论如何，其内存布局和 `std::map` 一样，都是一个由指针连接的、散布在堆上的节点集合。

**6. 性能分析 (Performance Analysis)**
`std::set` 的性能特征与 `std::map` 相同。

| 操作 (Operation) | 底层实现 (Red-Black Tree) | 时间复杂度 |
| :--- | :--- | :--- |
| 插入 (`insert`, `emplace`) | 树的查找 + 节点插入 + 平衡调整 | O(log N) |
| 查找 (`find`, `count`, `contains` (C++20)) | 树的查找 | O(log N) |
| 删除 (`erase`) | 树的查找 + 节点删除 + 平衡调整 | O(log N) |
| 遍历 (Iteration) | 中序遍历 (In-order traversal) | O(N) |
| 范围查找 (`lower_bound`, `upper_bound`) | 树的查找 | O(log N) |

**7. 常用成员函数与用法 (Common Member Functions & Usage)**
```cpp
#include <iostream>
#include <set>
#include <string>
#include <algorithm> // for set operations

void demonstrate_set() {
    // 1. 初始化和插入
    std::set<std::string> unique_words;

    unique_words.insert("apple");
    unique_words.insert("banana");
    unique_words.emplace("cherry");

    // 尝试插入重复元素，操作会被忽略
    auto result = unique_words.insert("apple");
    if (!result.second) {
        std::cout << "'apple' was already in the set. Insertion failed." << std::endl;
        // result.first 是指向已存在元素的迭代器
    }

    // 2. 查找元素
    // 使用 find
    auto it = unique_words.find("banana");
    if (it != unique_words.end()) {
        std::cout << "Found 'banana' in the set." << std::endl;
    }

    // 使用 count (对于 set，返回值只会是 0 或 1)
    if (unique_words.count("durian")) {
        std::cout << "Found 'durian'." << std::endl;
    } else {
        std::cout << "'durian' not found." << std::endl;
    }

    // 使用 contains (C++20, 更简洁)
    if (unique_words.contains("apple")) {
        std::cout << "The set contains 'apple'." << std::endl;
    }

    // 3. 删除元素
    unique_words.erase("banana");

    // 4. 遍历 (自动按字母顺序排序)
    std::cout << "\nElements in the set (sorted):" << std::endl;
    for (const auto& word : unique_words) {
        std::cout << word << " ";
    }
    std::cout << std::endl;

    // 5. 集合运算
    std::set<int> set1 = {1, 2, 3, 4, 5};
    std::set<int> set2 = {4, 5, 6, 7, 8};
    std::set<int> intersection_result;
    
    std::set_intersection(set1.begin(), set1.end(),
                          set2.begin(), set2.end(),
                          std::inserter(intersection_result, intersection_result.begin()));
    
    std::cout << "\nIntersection of set1 and set2:" << std::endl;
    for (int val : intersection_result) {
        std::cout << val << " ";
    }
    std::cout << std::endl;
}
```

**8. 使用场景与最佳实践 (Use Cases & Best Practices)**
*   **去重和排序:** 当你需要从一个集合中移除重复项并得到一个有序列表时，`std::set` 是最直接的工具。只需将所有元素插入 `set` 即可。
*   **成员资格测试:** 当你需要频繁地检查一个元素是否存在于一个集合中时，`std::set` 的 O(log N) 查找效率非常高。
*   **作为数学集合:** `std::set` 的行为与数学上的集合概念非常吻合，特别适合执行交集、并集、差集等运算。
*   **`find()` vs. C++20 `contains()`:** 在 C++20 之前，检查元素是否存在通常用 `set.find(key) != set.end()` 或 `set.count(key) > 0`。C++20 引入的 `contains(key)` 方法在语义上更清晰，应该优先使用。

**9. 与其他容器的对比 (Comparison)**
*   **`set` vs. `unordered_set`:**
    *   这是 `map` vs. `unordered_map` 对比的翻版。
    *   **`set` (红黑树):** 元素有序，操作复杂度 O(log N)。
    *   **`unordered_set` (哈希表):** 元素无序，操作均摊复杂度 O(1)。
    *   **选择:**
        *   需要有序性或范围查找 -> `set`。
        *   只关心快速的成员资格测试、插入、删除，不关心顺序 -> `unordered_set`（通常是首选）。

*   **`set` vs. `vector` + `sort` + `unique`:**
    *   **`set`:** 动态维护一个有序且唯一的集合。每次插入/删除都是 O(log N)。
    *   **`vector`:** 如果你是一次性获得所有数据，然后需要去重和排序，那么将数据放入 `vector`，调用 `std::sort`，然后调用 `vec.erase(std::unique(vec.begin(), vec.end()), vec.end())` 的组合操作通常会比逐个插入 `set` 更快，因为 `vector` 的操作是缓存友好的。但之后若有任何修改，`vector` 的维护成本会很高。



## multiset

> https://en.cppreference.com/w/cpp/container/multiset.html

**1. 概述 (Overview)**
`std::multiset` 是一个关联容器，用于存储**可重复的**元素，并且容器会自动根据元素的值进行**排序**。它与 `std::set` 的核心区别在于：`std::multiset` **允许存储多个值相等的元素**。

`std::multiset` 可以看作是 `std::multimap` 的一个特例，其中键和值是相同的。它同样基于自平衡二叉搜索树（通常是**红黑树**）实现，保证了高效的查找、插入和删除操作。

`std::multiset` 非常适合用于需要存储一个有序的元素列表，并且需要记录每个元素出现次数的场景。

**2. 头文件 (Header)**
```cpp
#include <set>
```
它和 `std::set` 在同一个头文件中。

**3. 特点与优势 (Features & Advantages)**
*   **允许元素重复:** 这是它与 `std::set` 最根本的区别。你可以存储多个值完全相同的元素。
*   **自动排序:** 元素总是根据其值保持有序（默认升序）。所有相等的元素在遍历时会相邻出现。
*   **高效操作:** 查找、插入和删除操作的平均和最坏时间复杂度都是 O(log N)。
*   **元素不可变性:** 与 `std::set` 一样，`multiset` 中的元素也是 `const` 的。一旦插入，就不能修改。要“修改”，必须 `erase` 旧元素，`insert` 新元素。
*   **为多值查找设计:** 提供了 `equal_range()`, `lower_bound()`, `upper_bound()` 和 `count()` 等方法，可以方便地查找和处理某个值的所有实例。

**4. 缺点与限制 (Disadvantages & Limitations)**
*   **与 `map`/`set` 相同的性能/内存缺点:**
    *   **非连续内存:** 基于树的结构，缓存性能较差。
    *   **高内存开销:** 每个节点需要存储指针和颜色信息。
    *   **对数时间复杂度:** 不如 `std::unordered_multiset` 的均摊 O(1)。
*   **元素不可修改:** 无法直接修改已存在元素的值。

**5. 内部实现与内存布局 (Internal Implementation & Memory Layout)**
`std::multiset` 的内部实现与 `std::multimap` 几乎完全相同，都是基于**红黑树**，只是节点存储的数据类型不同。

*   `std::multimap` 的节点存储 `std::pair<const Key, T>`。
*   `std::multiset` 的节点存储 `const Key`。

为了处理重复值，`multiset` 使用的红黑树插入算法与 `multimap` 相同，即**总是允许插入**，即使一个等价的元素已经存在。新节点会被放置在树中与现有等价节点“旁边”的位置，以维持中序遍历的有序性。

其内存布局、节点结构和 `std::set` 一致，都是一个由指针连接的、散布在堆上的节点集合。

**6. 性能分析 (Performance Analysis)**
`std::multiset` 的性能特征与 `std::multimap` 相同。

| 操作 (Operation) | 底层实现 (Red-Black Tree) | 时间复杂度 |
| :--- | :--- | :--- |
| 插入 (`insert`, `emplace`) | 树的查找 + 节点插入 + 平衡调整 | O(log N) |
| 查找一个值 (`find`) | 树的查找 | O(log N) |
| 删除一个值的所有实例 (`erase(value)`) | 查找 + 遍历删除 | O(log N + K)，K 是被删除的元素数量 |
| 删除单个实例 (`erase(iterator)`) | 节点删除 + 平衡调整 | 均摊 O(1) (C++11后)，最坏 O(log N) |
| 统计值的数量 (`count`) | 查找 + 遍历计数 | O(log N + K)，K 是具有该值的元素数量 |
| 范围查找 (`equal_range`, `lower_bound`, `upper_bound`) | 树的查找 | O(log N) |

**7. 常用成员函数与用法 (Common Member Functions & Usage)**
```cpp
#include <iostream>
#include <set>
#include <vector>

void demonstrate_multiset() {
    // 1. 初始化和插入
    std::multiset<int> scores = {95, 88, 92, 88, 95, 100};

    // 插入更多元素
    scores.insert(88);

    // 2. 遍历 (自动排序)
    std::cout << "All scores (sorted):" << std::endl;
    for (int score : scores) {
        std::cout << score << " ";
    }
    std::cout << std::endl; // 输出: 88 88 88 92 95 95 100

    // 3. 统计特定值的数量
    int score_to_count = 88;
    size_t num = scores.count(score_to_count);
    std::cout << "\nThe score " << score_to_count << " appears " << num << " times." << std::endl;

    // 4. 查找和处理一个值的所有实例 (使用 equal_range)
    std::cout << "\nFinding all instances of 95 using equal_range:" << std::endl;
    auto range = scores.equal_range(95);
    for (auto it = range.first; it != range.second; ++it) {
        std::cout << "Found a 95 at address " << &(*it) << std::endl;
    }

    // 5. 删除
    // 删除所有值为 88 的实例
    scores.erase(88);
    std::cout << "\nAfter erasing all 88s:" << std::endl;
    for (int score : scores) {
        std::cout << score << " ";
    }
    std::cout << std::endl; // 输出: 92 95 95 100

    // 只删除一个 95 的实例
    auto it_to_erase = scores.find(95);
    if (it_to_erase != scores.end()) {
        scores.erase(it_to_erase);
    }
    std::cout << "\nAfter erasing one 95:" << std::endl;
    for (int score : scores) {
        std::cout << score << " ";
    }
    std::cout << std::endl; // 输出: 92 95 100
}
```

**8. 使用场景与最佳实践 (Use Cases & Best Practices)**
*   **需要排序的直方图/词袋:** 当你需要统计各项出现的次数，并且希望能够按项的值进行有序访问时，`std::multiset` 是一个很好的选择。例如，统计一篇文章中所有单词的出现情况，并能按字母顺序列出所有单词。
*   **需要排序且允许重复的列表:** 任何你需要一个时刻保持有序，并且允许重复项存在的列表。例如，一个高分榜，允许多个玩家有相同的分数。
*   **替代 `std::priority_queue`:** 如果你需要一个优先队列，但又需要能够**遍历所有元素**或**删除非顶部元素**（通过迭代器），`std::multiset` 可以作为一种替代方案。`multiset.rbegin()` 可以访问最大值（模拟大顶堆的 `top`），`multiset.begin()` 可以访问最小值（模拟小顶堆的 `top`）。但其插入和删除效率（O(log N)）与 `priority_queue` 相同，缓存性能则更差。
*   **使用 `equal_range`:** 和 `multimap` 一样，当需要获取某个值的所有实例时，`equal_range` 是最标准、最高效的方法。

**9. 与其他容器的对比 (Comparison)**
*   **`multiset` vs. `set`:**
    *   **核心区别:** `multiset` 允许元素重复，`set` 不允许。
    *   `set::insert` 返回 `pair<iterator, bool>`，而 `multiset::insert` 只返回一个 `iterator`，因为它总是成功。

*   **`multiset` vs. `unordered_multiset`:**
    *   `multiset` 有序 (O(log N))，`unordered_multiset` 无序 (均摊 O(1))。选择逻辑与 `set` vs `unordered_set` 完全相同。

*   **`multiset` vs. `map<T, size_t>`:**
    *   `map<T, size_t>` 也可以用来统计元素出现次数，键是元素，值是计数。
    *   **选择 `map<T, size_t>` 的情况:**
        *   当你的主要目的是**计数**时，这种结构更明确。
        *   获取一个元素的计数是 O(1)（对于 `unordered_map`）或 O(log N)（对于 `map`）的直接查找，而不是像 `multiset::count` 那样需要 O(log N + K) 的遍历。
    *   **选择 `multiset` 的情况:**
        *   当你需要**实际存储**每一个重复的实例，而不仅仅是它们的计数时。
        *   当插入和删除单个实例是主要操作时。
        *   当需要对所有实例（包括重复的）进行统一的有序遍历时。


## unordered_map

> https://en.cppreference.com/w/cpp/container/unordered_map.html

**1. 概述 (Overview)**
`std::unordered_map` 是 C++11 引入的一个关联容器，它存储**唯一的键**和与之关联的**值**组成的键值对。与 `std::map` 最根本的区别在于，`std::unordered_map` **不保证元素的任何特定顺序**，它通过**哈希表 (Hash Table)** 而不是平衡二叉树来实现。

由于使用了哈希表，`std::unordered_map` 能够提供**平均时间复杂度为 O(1)** 的查找、插入和删除操作，这使得它在对性能要求极高的场景下，成为 `std::map` 的首选替代品。它的性能表现依赖于一个好的哈希函数和较低的哈希冲突率。

**2. 头文件 (Header)**
```cpp
#include <unordered_map>
```

**3. 特点与优势 (Features & Advantages)**
*   **极高的性能:** 查找、插入和删除操作的**平均时间复杂度为 O(1)**。这是它最核心的优势。
*   **键的唯一性:** 与 `std::map` 一样，键必须是唯一的。
*   **键值对结构:** 元素类型为 `std::pair<const Key, T>`，键同样是 `const` 的。
*   **不依赖于比较操作:** 元素的组织不依赖于 `<` 或 `>` 比较，而是依赖于键类型的**哈希函数 (`std::hash`)** 和**相等性比较 (`operator==`)**。

**4. 缺点与限制 (Disadvantages & Limitations)**
*   **无序性:** 元素在容器中的存储顺序是无序的，且可能在容器修改后发生变化。你不能依赖于遍历 `unordered_map` 时元素的顺序。
*   **最坏情况性能:** 在极端的哈希冲突情况下（例如，所有元素都哈希到同一个桶），操作的时间复杂度会退化到 **O(N)**。
*   **对哈希函数的依赖:** 性能严重依赖于哈希函数的质量。一个坏的哈希函数会导致大量的冲突，从而降低性能。
*   **更高的内存开销:** 为了维持较低的冲突率，哈希表通常需要预留比实际元素数量更多的空间（桶数组），这可能导致比 `std::map` 更大的内存占用，尤其是在元素较少时。
*   **迭代器稳定性差:** 任何导致**重哈希 (rehash)** 的操作（通常是插入元素导致负载因子超过阈值）都会使**所有**迭代器、指针和引用失效。

**5. 内部实现与内存布局 (Internal Implementation & Memory Layout)**
`std::unordered_map` 的内部实现是一个**哈希表**，具体来说，通常是采用**“开链法” (Separate Chaining)** 的哈希表。

**内部结构**
1.  **桶数组 (Bucket Array):** 一个连续的数组，通常是 `std::vector`。数组的每个位置被称为一个**桶 (Bucket)**。
2.  **哈希函数 (Hash Function):** 一个函数，它接受一个键 `Key` 作为输入，返回一个 `std::size_t` 类型的哈希值。
3.  **桶索引计算:** `bucket_index = hash(key) % bucket_count`。这个计算决定了给定的键应该被放入哪个桶。
4.  **链表 (Linked List):** 每个桶中存储一个指向**链表头**的指针（或直接是链表本身）。这个链表包含了所有哈希到该桶的键值对。

**内存布局**
```
// 内存模型示意图

 Bucket Array (on Heap)
 +----------+
 | bucket 0 | ----> Node{K1,V1} -> Node{K4,V4} -> nullptr
 +----------+
 | bucket 1 | ----> nullptr
 +----------+
 | bucket 2 | ----> Node{K2,V2} -> nullptr
 +----------+
 |   ...    |
 +----------+
 | bucket M | ----> Node{K3,V3} -> Node{K5,V5} -> nullptr
 +----------+
```
*   `unordered_map` 对象本身包含指向桶数组的指针，以及 `size`, `bucket_count`, `max_load_factor` 等元数据。
*   桶数组在堆上分配。
*   每个键值对作为一个节点在堆上独立分配，并通过指针链接在相应的桶链表中。

**哈希与冲突**
*   **操作流程 (`find(key)`):**
    1.  计算 `hash(key)`。
    2.  计算 `bucket_index = hash(key) % bucket_count`。
    3.  遍历 `bucket[bucket_index]` 中的链表，使用 `operator==` 逐个比较键，直到找到匹配的键或到达链表末尾。

*   **重哈希 (Rehashing):**
    `unordered_map` 维护一个**负载因子 (load factor)**，即 `size() / bucket_count()`。当插入一个新元素后，如果负载因子超过了**最大负载因子** (`max_load_factor()`, 默认通常为 1.0)，就会触发重哈希：
    1.  创建一个更大尺寸的新桶数组。
    2.  遍历旧表中的所有元素。
    3.  为每个元素重新计算其在新表中的桶索引，并将其插入新桶的链表中。
    4.  释放旧的桶数组。
    重哈希是一个 O(N) 的昂贵操作，这也是导致插入操作的复杂度是“均摊”O(1) 的原因。

**6. 性能分析 (Performance Analysis)**
| 操作 (Operation) | 底层实现 (Hash Table) | 时间复杂度 |
| :--- | :--- | :--- |
| 插入 (`insert`, `emplace`) | 哈希 + 链表插入 (+ 可能的重哈希) | **均摊 O(1)**, 最坏 O(N) |
| 查找 (`find`, `count`) | 哈希 + 链表搜索 | **平均 O(1)**, 最坏 O(N) |
| 删除 (`erase`) | 哈希 + 链表搜索和删除 | **平均 O(1)**, 最坏 O(N) |
| 访问 (`operator[]`, `at`) | 同上 | **平均 O(1)**, 最坏 O(N) |

**7. 常用成员函数与用法 (Common Member Functions & Usage)**
`std::unordered_map` 的接口与 `std::map` 非常相似，包括 `operator[]`, `at`, `find`, `insert`, `emplace`, `erase` 等。

```cpp
#include <iostream>
#include <unordered_map>
#include <string>

void demonstrate_unordered_map() {
    // 1. 初始化
    std::unordered_map<std::string, int> student_ages;

    // 2. 插入 (接口同 map)
    student_ages.insert({"Alice", 20});
    student_ages.emplace("Bob", 22);
    student_ages["Charlie"] = 21;

    // 3. 访问 (接口同 map)
    std::cout << "Bob's age: " << student_ages["Bob"] << std::endl;
    student_ages["Bob"] = 23; // 更新
    
    // 4. 遍历 (顺序是无序的，且不固定！)
    std::cout << "\nAll students (in arbitrary order):" << std::endl;
    for (const auto& pair : student_ages) {
        std::cout << pair.first << ": " << pair.second << std::endl;
    }
    
    // 5. 哈希策略相关的函数
    std::cout << "\nHash table details:" << std::endl;
    std::cout << "Size: " << student_ages.size() << std::endl;
    std::cout << "Bucket count: " << student_ages.bucket_count() << std::endl;
    std::cout << "Load factor: " << student_ages.load_factor() << std::endl;
    std::cout << "Max load factor: " << student_ages.max_load_factor() << std::endl;

    // 预留空间以避免重哈希
    std::unordered_map<int, int> large_map;
    large_map.reserve(1000); // 建议的优化！
    for (int i = 0; i < 1000; ++i) {
        large_map[i] = i; // 这些插入很可能不会触发重哈希
    }
}

// 为自定义类型提供哈希函数
struct Point {
    int x, y;
    bool operator==(const Point& other) const {
        return x == other.x && y == other.y;
    }
};

// 必须在 std 命名空间内特化 std::hash
namespace std {
    template<>
    struct hash<Point> {
        size_t operator()(const Point& p) const {
            // 一个简单的组合哈希
            auto h1 = std::hash<int>{}(p.x);
            auto h2 = std::hash<int>{}(p.y);
            return h1 ^ (h2 << 1); // 或者其他组合方式
        }
    };
}

void custom_type_demo() {
    std::unordered_map<Point, std::string> point_names;
    point_names[{10, 20}] = "Origin";
    std::cout << "\nName of point (10, 20): " << point_names.at({10, 20}) << std::endl;
}
```

**8. 使用场景与最佳实践 (Use Cases & Best Practices)**
*   **默认的关联容器:** 当你不需要对元素进行排序时，`std::unordered_map` 应该是你**首选的**字典/查找表类型，因为它提供了最佳的平均性能。
*   **性能关键的查找:** 在量化金融、游戏引擎、网络服务器等任何对低延迟查找有严格要求的应用中，`unordered_map` 都是核心工具。
*   **预分配空间 (`reserve`):** 如果你预先知道将要存储大约 N 个元素，请务必调用 `reserve(N)`。这会一次性分配足够的桶，可以有效地**避免**后续插入过程中的多次**重哈希**，是 `unordered_map` 最重要的性能优化手段。
*   **自定义类型的哈希:** 要将自定义类型作为 `unordered_map` 的键，你必须：
    1.  为该类型重载 `operator==`。
    2.  提供一个哈希函数。最常见的方式是特化 `std::hash<YourType>`。一个好的哈希函数应该尽可能地将不同的输入映射到不同的输出，以减少冲突。

**9. 与其他容器的对比 (Comparison)**
*   **`unordered_map` vs. `map`:**
    *   **核心权衡:** 速度 vs. 顺序。
    *   **`unordered_map` (哈希表):** 均摊 O(1) 查找，无序。
    *   **`map` (红黑树):** O(log N) 查找，有序。
    *   **选择:** 除非你需要有序性或范围查找，否则优先选择 `unordered_map`。



## unordered_multimap

> https://en.cppreference.com/w/cpp/container/unordered_multimap.html

**1. 概述 (Overview)**
`std::unordered_multimap` 是 `std::unordered_map` 的一个变体，它同样基于**哈希表**实现，并提供**平均 O(1)** 的操作性能。与 `std::unordered_map` 的核心区别在于：`std::unordered_multimap` **允许存储具有相同键的多个元素**。

它结合了 `std::unordered_map` 的高速性能和 `std::multimap` 的多值映射能力。这使得它成为在不关心顺序的情况下，实现高性能“一对多”关系映射的理想选择。

**2. 头文件 (Header)**
```cpp
#include <unordered_map>
```
是的，它和 `std::unordered_map` 在同一个头文件中。

**3. 特点与优势 (Features & Advantages)**
*   **极高的性能:** 查找、插入和删除操作的**平均时间复杂度为 O(1)**。
*   **允许键重复:** 可以为一个键关联多个不同的值。
*   **不依赖于比较操作:** 元素的组织依赖于键类型的**哈希函数 (`std::hash`)** 和**相等性比较 (`operator==`)**。

**4. 缺点与限制 (Disadvantages & Limitations)**
*   **没有 `operator[]` 和 `.at()`:** 与 `std::multimap` 一样，由于一个键可以对应多个值，使用 `umm[key]` 这样的语法会产生歧义。因此，`std::unordered_multimap` **不提供** `operator[]` 和 `.at()` 访问器。
*   **无序性:** 元素在容器中以及具有相同键的元素之间的相对顺序都是无序的。
*   **与 `unordered_map` 相同的性能/内存缺点:**
    *   **最坏情况性能:** 在大量哈希冲突时，操作会退化到 O(N)。
    *   **对哈希函数的依赖:** 性能严重依赖哈希函数的质量。
    *   **迭代器稳定性差:** 任何导致重哈希的操作都会使所有迭代器失效。

**5. 内部实现与内存布局 (Internal Implementation & Memory Layout)**
`std::unordered_multimap` 的内部实现与 `std::unordered_map` 几乎完全相同，都是采用**开链法 (Separate Chaining)** 的**哈希表**。

唯一的区别在于处理键重复的策略：
*   在 `std::unordered_map` 中，当插入一个已存在的键时，`insert` 操作会失败。
*   在 `std::unordered_multimap` 中，`insert` 操作**总是会成功**。如果一个键已经存在，新的键值对节点会被简单地添加到对应桶的链表中。

因此，一个桶的链表中可能包含多个具有不同值的、但键相同的节点。

其内存布局（桶数组 + 链表节点）、哈希流程、重哈希机制都与 `std::unordered_map` 完全一致。

**6. 性能分析 (Performance Analysis)**
`std::unordered_multimap` 的性能特征与 `std::unordered_map` 相同。

| 操作 (Operation) | 底层实现 (Hash Table) | 时间复杂度 |
| :--- | :--- | :--- |
| 插入 (`insert`, `emplace`) | 哈希 + 链表插入 (+ 可能的重哈希) | **均摊 O(1)**, 最坏 O(N) |
| 查找一个键 (`find`) | 哈希 + 链表搜索 | **平均 O(1)**, 最坏 O(N) |
| 删除一个键的所有条目 (`erase(key)`) | 哈希 + 遍历链表删除 | 平均 O(1+K), 最坏 O(N)，K是删除数量 |
| 删除单个条目 (`erase(iterator)`) | 链表节点删除 | 平均 O(1), 最坏 O(N) (取决于链表长度) |
| 统计键数量 (`count`) | 哈希 + 遍历链表计数 | 平均 O(1+K), 最坏 O(N)，K是计数值 |
| 范围查找 (`equal_range`) | 哈希 + 遍历链表 | 平均 O(1+K), 最坏 O(N)，K是范围大小 |

**7. 常用成员函数与用法 (Common Member Functions & Usage)**
由于没有 `operator[]`，与 `std::unordered_multimap` 的交互方式与 `std::multimap` 类似。

```cpp
#include <iostream>
#include <unordered_map>
#include <string>

void demonstrate_unordered_multimap() {
    // 1. 初始化和插入
    std::unordered_multimap<std::string, std::string> city_attractions;

    city_attractions.insert({"Paris", "Eiffel Tower"});
    city_attractions.insert({"Paris", "Louvre Museum"});
    city_attractions.emplace("London", "Tower of London");
    city_attractions.emplace("London", "British Museum");
    city_attractions.emplace("Paris", "Notre-Dame Cathedral"});

    // 2. 查找和处理一个键对应的所有值
    std::string city_to_find = "Paris";

    // 方法一: 使用 find 和迭代器递增 (注意，这在无序容器中不如下面的方法好)
    // 因为相同键的元素不保证连续存储（虽然通常实现会这么做）
    
    // 方法二: 使用 equal_range (最高效、最地道的方法)
    std::cout << "Attractions in " << city_to_find << " (using equal_range):" << std::endl;
    auto range = city_attractions.equal_range(city_to_find);
    for (auto it = range.first; it != range.second; ++it) {
        std::cout << " - " << it->second << std::endl;
    }

    // 3. 统计
    size_t num_attractions = city_attractions.count("London");
    std::cout << "\nLondon has " << num_attractions << " attractions listed." << std::endl;

    // 4. 删除
    // 删除所有键为 "London" 的条目
    city_attractions.erase("London");

    // 5. 遍历 (顺序是无序的)
    std::cout << "\nAll remaining entries (in arbitrary order):" << std::endl;
    for (const auto& pair : city_attractions) {
        std::cout << pair.first << " -> " << pair.second << std::endl;
    }
}
```

**8. 使用场景与最佳实践 (Use Cases & Best Practices)**
*   **高性能的一对多关系映射:** 当你需要一个“字典”，其中一个键可以映射到多个值，并且你**不关心**这些键或值的顺序，只关心最快的查找和插入速度时，`std::unordered_multimap` 是不二之选。
    *   **网络服务器:** 将一个 IP 地址映射到它发起的所有连接。
    *   **倒排索引 (Inverted Index) 的简单实现:** 将一个单词（键）映射到所有包含该单词的文档ID（值）。
*   **使用 `equal_range`:** 这是获取与某个键关联的所有值的最标准、最高效的方法。
*   **`reserve` 优化:** 与 `unordered_map` 一样，如果你能预估元素总数，提前调用 `reserve(N)` 可以显著提升批量插入的性能。
*   **替代方案 `unordered_map<Key, vector<Value>>`:**
    *   这个选择的权衡与 `multimap` vs. `map<Key, vector<Value>>` 类似。
    *   **选择 `unordered_map<Key, vector<Value>>`:** 当你需要对单个键的值集合进行复杂操作（排序、随机访问等），或者代码可读性更重要时。
    *   **选择 `unordered_multimap`:** 当插入/删除单个值是主要操作，且希望避免为每个键都创建 `vector` 对象的开销时。

**9. 与其他容器的对比 (Comparison)**
*   **`unordered_multimap` vs. `unordered_map`:**
    *   **核心区别:** `unordered_multimap` 允许键重复，`unordered_map` 不允许。
    *   **接口差异:** `unordered_multimap` 没有 `operator[]` 和 `.at()`。

*   **`unordered_multimap` vs. `multimap`:**
    *   **核心权衡:** 速度 vs. 顺序。
    *   **`unordered_multimap` (哈希表):** 均摊 O(1)，无序。
    *   **`multimap` (红黑树):** O(log N)，有序。
    *   **选择:**
        *   需要有序性 -> `multimap`。
        *   追求极致性能且不关心顺序 -> `unordered_multimap`。


## unordered_set

> https://en.cppreference.com/w/cpp/container/unordered_set.html

**1. 概述 (Overview)**
`std::unordered_set` 是 C++11 引入的一个关联容器，用于存储**唯一的**元素。与 `std::set` 不同，`unordered_set` 中的元素是**无序的**。它通过**哈希表 (Hash Table)** 实现，这使得它在插入、删除和查找操作上能够提供**平均 O(1)** 的时间复杂度。

`unordered_set` 的行为可以看作是一个只有键、没有值的 `std::unordered_map`。它是在不关心元素顺序，只追求最快成员资格测试和去重速度的场景下的理想选择。

**2. 头文件 (Header)**
```cpp
#include <unordered_set>
```

**3. 特点与优势 (Features & Advantages)**
*   **极高的性能:** 查找、插入和删除操作的**平均时间复杂度为 O(1)**。这使得它在需要快速判断元素是否存在时非常高效。
*   **元素唯一性:** `unordered_set` 中不允许存在重复的元素。
*   **不依赖于比较操作:** 元素的组织不依赖于 `<` 比较，而是依赖于元素类型的**哈希函数 (`std::hash`)** 和**相等性比较 (`operator==`)**。

**4. 缺点与限制 (Disadvantages & Limitations)**
*   **无序性:** 元素在容器中的存储顺序是无序的，遍历 `unordered_set` 时得到的元素顺序是不确定的。
*   **最坏情况性能:** 在大量哈希冲突（即多个元素哈希到同一个桶）的情况下，操作性能会退化到 O(N)。
*   **对哈希函数的依赖:** 性能严重依赖于哈希函数的质量。一个糟糕的哈希函数会导致大量的冲突，从而降低性能。
*   **迭代器稳定性差:** 任何导致重哈希 (rehashing) 的操作（通常是插入操作使得负载因子超过阈值时）都会使所有指向容器元素的迭代器、指针和引用失效。
*   **更高的内存开销（相对 `vector`）:** 需要为哈希表本身（桶数组）分配内存，以及为每个元素分配节点内存。
*   **元素不可变性:** 与 `std::set` 一样，`unordered_set` 中的元素也是 `const` 的。一旦插入，就不能修改，因为修改会影响其哈希值和在哈希表中的位置。

**5. 内部实现与内存布局 (Internal Implementation & Memory Layout)**
`std::unordered_set` 的内部实现与 `std::unordered_map` 几乎完全相同，都是基于**开链法 (Separate Chaining)** 的**哈希表**。

关键区别在于节点存储的数据：
*   `std::unordered_map` 的节点存储 `std::pair<const Key, T>`。
*   `std::unordered_set` 的节点存储 `const Key`。

其内部结构、哈希流程、负载因子 (load factor) 和重哈希 (rehashing) 机制都与 `std::unordered_map` 完全一致。

1.  **桶数组 (Bucket Array):** 一个指针数组，每个指针指向一个桶的头节点。
2.  **哈希函数:** 将元素的键转换为桶数组的索引。
3.  **节点链表:** 每个桶包含一个单向链表，用于存储哈希到该桶的所有元素。
4.  **重哈希:** 当元素数量增长，使得 `size() / bucket_count()`（负载因子）超过 `max_load_factor()` 时，哈希表会分配一个更大的桶数组，并重新计算所有元素的位置。这是一个 O(N) 的昂贵操作。

**6. 性能分析 (Performance Analysis)**
| 操作 (Operation) | 底层实现 (Hash Table) | 时间复杂度 |
| :--- | :--- | :--- |
| 插入 (`insert`, `emplace`) | 哈希 + 链表搜索与插入 (+ 可能的重哈希) | **均摊 O(1)**, 最坏 O(N) |
| 查找 (`find`, `count`, `contains` (C++20)) | 哈希 + 链表搜索 | **平均 O(1)**, 最坏 O(N) |
| 删除 (`erase`) | 哈希 + 链表搜索与删除 | **平均 O(1)**, 最坏 O(N) |
| 遍历 (Iteration) | 遍历桶数组及其所有链表 | O(N) |

**7. 常用成员函数与用法 (Common Member Functions & Usage)**
```cpp
#include <iostream>
#include <unordered_set>
#include <string>
#include <vector>

void demonstrate_unordered_set() {
    // 1. 初始化和插入
    std::unordered_set<std::string> fruit_set;

    fruit_set.insert("apple");
    fruit_set.insert("banana");
    fruit_set.emplace("cherry");

    // 尝试插入重复元素，操作会被忽略
    auto result = fruit_set.insert("apple");
    if (!result.second) {
        std::cout << "'apple' was already in the set. Insertion failed." << std::endl;
    }

    // 2. 查找元素 (成员资格测试)
    if (fruit_set.count("banana")) { // count 返回 0 或 1
        std::cout << "Found 'banana' using count." << std::endl;
    }

    // C++20 的 contains() 是更现代、更清晰的方式
    if (fruit_set.contains("cherry")) {
        std::cout << "The set contains 'cherry'." << std::endl;
    }

    // 3. 删除
    fruit_set.erase("banana");

    // 4. 遍历 (顺序是无序的)
    std::cout << "\nElements in the unordered_set (arbitrary order):" << std::endl;
    for (const auto& fruit : fruit_set) {
        std::cout << fruit << " ";
    }
    std::cout << std::endl;

    // 5. 哈希表相关的 API
    std::cout << "\nBucket count: " << fruit_set.bucket_count() << std::endl;
    std::cout << "Load factor: " << fruit_set.load_factor() << std::endl;
    std::cout << "Max load factor: " << fruit_set.max_load_factor() << std::endl;
    
    // 预留空间以避免重哈希
    std::unordered_set<int> large_set;
    large_set.reserve(1000); // 准备存储1000个元素
    for(int i = 0; i < 1000; ++i) {
        large_set.insert(i); // 插入操作很可能不会触发重哈希
    }
}
```

**8. 使用场景与最佳实践 (Use Cases & Best Practices)**
*   **高性能去重:** 从一个大数据集中快速移除重复项是 `unordered_set` 的经典应用。
*   **高速成员资格测试:** 当你需要一个集合，并频繁地检查某个元素是否存在于其中时（例如，一个黑名单、一个访问过的节点集），`unordered_set` 提供了最快的平均性能。
*   **自定义类型的哈希:** 如果要在 `unordered_set` 中存储自定义类型 `MyType`，你必须提供一个哈希函数和一个相等性比较操作。通常通过特化 `std::hash<MyType>` 模板来提供哈希函数。
*   **`reserve` 和 `max_load_factor`:**
    *   如果你能预估将要存储的元素数量 `N`，在插入前调用 `us.reserve(N)` 可以提前分配足够的桶，从而避免在插入过程中发生多次昂贵的重哈希。
    *   可以通过 `us.max_load_factor(float)` 来调整触发重哈希的负载因子阈值，以在内存使用和哈希冲突概率之间进行权衡。

**9. 与其他容器的对比 (Comparison)**
*   **`unordered_set` vs. `set`:**
    *   **核心权衡:** 速度 vs. 顺序。
    *   **`unordered_set` (哈希表):** 均摊 O(1)，无序。
    *   **`set` (红黑树):** O(log N)，有序。
    *   **选择:**
        *   需要有序性或范围查找 -> `set`。
        *   只关心快速的成员资格测试、插入、删除，不关心顺序 -> `unordered_set`（绝大多数情况下的首选）。

*   **`unordered_set` vs. `vector` + `sort` + `unique`:**
    *   **`unordered_set`:** 适合动态地维护一个唯一的元素集合，可以随时高效地增删查。
    *   **`vector` 方案:** 适合静态的一次性去重操作。对于已排序的 `vector`，使用 `std::binary_search` 也可以实现 O(log N) 的查找，但 `vector` 本身的插入和删除是线性的。



## unordered_multiset

> https://en.cppreference.com/w/cpp/container/unordered_multiset.html

**1. 概述 (Overview)**
`std::unordered_multiset` 是 C++11 引入的一个关联容器，它存储**可重复的**元素，并且这些元素在容器中是**无序的**。它同样是基于**哈希表 (Hash Table)** 实现的，提供了**平均 O(1)** 的操作性能。

`unordered_multiset` 结合了 `std::unordered_set` 的高速性能和 `std::multiset` 的多实例存储能力。它非常适合在不关心顺序的情况下，需要快速统计各项出现次数或维护一个包含重复项的“物品袋”的场景。

**2. 头文件 (Header)**
```cpp
#include <unordered_set>
```
它和 `std::unordered_set` 在同一个头文件中。

**3. 特点与优势 (Features & Advantages)**
*   **极高的性能:** 查找、插入和删除操作的**平均时间复杂度为 O(1)**。
*   **允许元素重复:** 可以存储多个值完全相同的元素。
*   **不依赖于比较操作:** 元素的组织依赖于元素类型的**哈希函数 (`std::hash`)** 和**相等性比较 (`operator==`)**。

**4. 缺点与限制 (Disadvantages & Limitations)**
*   **无序性:** 元素在容器中的存储顺序是无序的。
*   **与 `unordered_map`/`set` 相同的性能/内存缺点:**
    *   **最坏情况性能:** 在大量哈希冲突时，操作会退化到 O(N)。
    *   **对哈希函数的依赖:** 性能严重依赖哈希函数的质量。
    *   **迭代器稳定性差:** 任何导致重哈希的操作都会使所有迭代器失效。
*   **元素不可变性:** 与所有 `set` 容器一样，`unordered_multiset` 中的元素也是 `const` 的，不可修改。

**5. 内部实现与内存布局 (Internal Implementation & Memory Layout)**
`std::unordered_multiset` 的内部实现与 `std::unordered_multimap` 几乎完全相同，都是采用**开链法 (Separate Chaining)** 的**哈希表**。

区别在于节点存储的数据：
*   `std::unordered_multimap` 的节点存储 `std::pair<const Key, T>`。
*   `std::unordered_multiset` 的节点存储 `const Key`。

为了处理重复值，`unordered_multiset` 使用的哈希表插入算法与 `unordered_multimap` 相同，即**总是允许插入**。如果一个元素已经存在，新的节点会被简单地添加到对应桶的链表中。

其内存布局、哈希流程、重哈希机制都与 `unordered` 家族的其他成员完全一致。

**6. 性能分析 (Performance Analysis)**
`std::unordered_multiset` 的性能特征与 `std::unordered_multimap` 相同。

| 操作 (Operation) | 底层实现 (Hash Table) | 时间复杂度 |
| :--- | :--- | :--- |
| 插入 (`insert`, `emplace`) | 哈希 + 链表插入 (+ 可能的重哈希) | **均摊 O(1)**, 最坏 O(N) |
| 查找一个值 (`find`) | 哈希 + 链表搜索 | **平均 O(1)**, 最坏 O(N) |
| 删除一个值的所有实例 (`erase(value)`) | 哈希 + 遍历链表删除 | 平均 O(1+K), 最坏 O(N)，K是删除数量 |
| 删除单个实例 (`erase(iterator)`) | 链表节点删除 | 平均 O(1), 最坏 O(N) |
| 统计值的数量 (`count`) | 哈希 + 遍历链表计数 | 平均 O(1+K), 最坏 O(N)，K是计数值 |
| 范围查找 (`equal_range`) | 哈希 + 遍历链表 | 平均 O(1+K), 最坏 O(N)，K是范围大小 |

**7. 常用成员函数与用法 (Common Member Functions & Usage)**
```cpp
#include <iostream>
#include <unordered_set>
#include <string>

void demonstrate_unordered_multiset() {
    // 1. 初始化和插入
    std::unordered_multiset<std::string> word_bag;
    word_bag.insert("apple");
    word_bag.insert("banana");
    word_bag.insert("apple"); // 允许重复
    word_bag.emplace("cherry");
    word_bag.emplace("banana");

    // 2. 遍历 (顺序是无序的)
    std::cout << "Words in the bag (arbitrary order):" << std::endl;
    for (const auto& word : word_bag) {
        std::cout << word << " ";
    }
    std::cout << std::endl;

    // 3. 统计特定值的数量
    std::cout << "\n'apple' appears " << word_bag.count("apple") << " times." << std::endl;
    std::cout << "'banana' appears " << word_bag.count("banana") << " times." << std::endl;

    // 4. 查找和处理一个值的所有实例 (使用 equal_range)
    std::cout << "\nFinding all instances of 'banana':" << std::endl;
    auto range = word_bag.equal_range("banana");
    for (auto it = range.first; it != range.second; ++it) {
        std::cout << "Found a 'banana' at address " << &(*it) << std::endl;
    }

    // 5. 删除
    // 删除所有 "apple"
    word_bag.erase("apple");
    std::cout << "\nAfter erasing all 'apple's, 'apple' count is: " << word_bag.count("apple") << std::endl;

    // 只删除一个 "banana"
    auto it_to_erase = word_bag.find("banana");
    if (it_to_erase != word_bag.end()) {
        word_bag.erase(it_to_erase);
    }
    std::cout << "After erasing one 'banana', 'banana' count is: " << word_bag.count("banana") << std::endl;
}
```

**8. 使用场景与最佳实践 (Use Cases & Best Practices)**
*   **高性能直方图/词袋模型:** 当你需要快速统计各项出现的次数，并且不关心项的顺序时，`unordered_multiset` 是最快的工具。
*   **需要快速增删的“物品袋”:** 在游戏开发或模拟中，一个角色可能有一个物品袋，里面有多个同种物品。使用 `unordered_multiset` 可以快速地添加、移除和查询物品。
*   **替代 `unordered_map<T, size_t>`:**
    *   **选择 `unordered_map<T, size_t>`:** 当你的主要目的是**快速查询计数**时，`map[key]` 是最直接的方式。
    *   **选择 `unordered_multiset`:** 当你需要**实际存储每一个实例**，而不仅仅是计数时。例如，如果每个物品实例可能有不同的元数据（虽然这超出了 `multiset` 的范围，但说明了“实例”和“计数”的区别）。当插入/删除单个实例是主要操作时，`unordered_multiset` 的接口更自然。

**9. 与其他容器的对比 (Comparison)**
*   **`unordered_multiset` vs. `unordered_set`:**
    *   **核心区别:** `unordered_multiset` 允许元素重复，`unordered_set` 不允许。
    *   `unordered_set::insert` 返回 `pair<iterator, bool>`，而 `unordered_multiset::insert` 只返回一个 `iterator`。

*   **`unordered_multiset` vs. `multiset`:**
    *   **核心权衡:** 速度 vs. 顺序。
    *   **`unordered_multiset` (哈希表):** 均摊 O(1)，无序。
    *   **`multiset` (红黑树):** O(log N)，有序。
    *   **选择:**
        *   需要有序性 -> `multiset`。
        *   追求极致性能且不关心顺序 -> `unordered_multiset`。


## pair & tuple

这两个都不是传统意义上的“容器”，因为它们不管理可变数量的元素序列。它们是简单的**数据结构模板**，用于将固定数量的、类型可能不同的值聚合在一起。由于它们关系密切，将把它们放在一起讲解。

**`std::pair` & `std::tup**le` — 异构数据聚合**

`std::pair` 和 `std::tuple` 是 C++ 中用于捆绑不同类型值的基本工具。它们是轻量级的、通用的聚合体，经常用于函数返回值、作为其他容器（如 `std::map`）的元素等场景。

---

### `pair`

> https://en.cppreference.com/w/cpp/utility/pair.html

**1. 概述 (Overview)**
`std::pair` 是一个简单的结构体模板，它精确地存储**两个**成员对象，这两个对象的类型可以不同。这两个成员分别被命名为 `first` 和 `second`。

它是 C++98 就存在的元老级工具，最广为人知的应用就是作为 `std::map` 的元素类型。

**2. 头文件 (Header)**
```cpp
#include <utility>
```

**3. 特点与优势 (Features & Advantages)**
*   **简单直观:** 只有两个成员，通过 `.first` 和 `.second` 访问，非常易于理解和使用。
*   **轻量级:** 几乎是零成本抽象，其大小和内存布局与一个包含同样两个成员的普通 `struct` 相同。
*   **自动生成比较操作:** `std::pair` 自动提供了 `operator==`, `operator!=`, `operator<` 等比较运算符。比较操作是**字典序 (lexicographical)** 的：先比较 `first` 成员，如果相等，再比较 `second` 成员。
*   **与C++17结构化绑定完美配合:** 可以非常优雅地解构 `pair`。

**4. 缺点与限制 (Disadvantages & Limitations)**
*   **固定大小:** 只能存储两个元素。
*   **命名不具描述性:** 成员名 `first` 和 `second` 缺乏语义。当 `pair` 的两个成员代表的意义不那么直观时（例如，不是“键”和“值”），代码可读性会下降。

**5. 内部实现与内存布局 (Internal Implementation & Memory Layout)**
`std::pair` 的概念性实现极其简单：

```cpp
template<class T1, class T2>
struct pair {
    T1 first;
    T2 second;
    // ... 构造函数, 比较运算符等 ...
};
```
它就是一个简单的 `struct`。其内存布局是连续的，`first` 成员在前，`second` 成员在后（可能会有内存对齐产生的填充）。`sizeof(std::pair<T1, T2>)` 约等于 `sizeof(T1) + sizeof(T2)`。

**6. 性能分析 (Performance Analysis)**
`std::pair` 的所有操作（构造、访问、比较）都是**编译时**或 **O(1)** 的。它不涉及任何动态内存分配（除非其成员类型自身进行动态分配）或循环。

**7. 常用函数与用法 (Common Member Functions & Usage)**
```cpp
#include <iostream>
#include <utility>
#include <string>

void demonstrate_pair() {
    // 1. 构造
    std::pair<int, std::string> p1(1, "hello");
    std::pair<int, std::string> p2{2, "world"}; // C++11
    
    // 使用辅助函数 std::make_pair (类型推导)
    auto p3 = std::make_pair(3, "!");

    // 2. 访问成员
    std::cout << "p1: " << p1.first << ", " << p1.second << std::endl;
    p1.first = 10; // 成员是可修改的

    // 3. 比较
    std::cout << "p1 < p2: " << std::boolalpha << (p1 < p2) << std::endl; // false, 因为 10 > 2

    // 4. 结构化绑定 (C++17) - 最佳实践
    std::pair<std::string, double> product = {"Apple", 1.25};
    auto [name, price] = product;
    std::cout << "Product: " << name << ", Price: $" << price << std::endl;
}
```

---

### `tuple`

> https://en.cppreference.com/w/cpp/utility/tuple.html

**1. 概述 (Overview)**
`std::tuple` 是 C++11 引入的，可以看作是 `std::pair` 的**泛化版本**。它是一个可以存储**任意数量**、任意类型元素的固定大小的聚合体。

由于其大小和类型是可变的（通过可变参数模板实现），`tuple` 的元素不能通过命名成员访问，而是通过一个非成员函数 `std::get<N>()`，并使用编译时整数常量 `N`作为索引来访问。

**2. 头文件 (Header)**
```cpp
#include <tuple>
```

**3. 特点与优势 (Features & Advantages)**
*   **任意大小和类型:** 可以容纳任意数量、任意类型的元素，`std::tuple<int, std::string, double, char>`。
*   **轻量级:** 同样是零成本抽象，内存布局紧凑。
*   **自动生成比较操作:** 与 `pair` 类似，提供字典序的比较。
*   **与C++17结构化绑定完美配合:** 同样可以优雅地解构。
*   **强大的辅助函数:** `std::tie` 和 `std::tuple_cat` 等函数提供了强大的功能。

**4. 缺点与限制 (Disadvantages & Limitations)**
*   **访问方式不直观:** 必须使用 `std::get<N>(my_tuple)` 语法，索引 `N` 是一个编译时常量。
*   **命名极不具描述性:** `get<0>`, `get<1>`, ... 是典型的“魔术数字”，严重影响代码可读性。如果一个 `tuple` 超过3-4个元素，通常就应该考虑使用 `struct`。

**5. 内部实现与内存布局 (Internal Implementation & Memory Layout)**
`std::tuple` 的实现通常依赖于**可变参数模板 (Variadic Templates)** 和**模板递归继承**。一个常见的概念性实现思路是：

```cpp
// 概念性实现，非实际代码
template<class... Ts> struct tuple;

template<class Head, class... Tail>
struct tuple<Head, Tail...> : public tuple<Tail...> {
    Head head;
};

template<>
struct tuple<> {}; // 递归基例：空 tuple
```
在这个模型中，`tuple<int, string, double>` 继承自 `tuple<string, double>`，后者又继承自 `tuple<double>`，最后继承自 `tuple<>`。每个层级持有一个元素。这保证了所有元素都紧凑地存储在最终的对象中。

内存布局同样是连续的，`sizeof(std::tuple<...>)` 约等于其所有成员大小的总和（加上可能的内存对齐填充）。

**6. 性能分析 (Performance Analysis)**
与 `std::pair` 相同，所有操作都是**编译时**或 **O(1)** 的。

**7. 常用函数与用法 (Common Member Functions & Usage)**
```cpp
#include <iostream>
#include <tuple>
#include <string>

// 函数返回多个值
std::tuple<int, std::string, bool> get_status() {
    return {200, "OK", true};
}

void demonstrate_tuple() {
    // 1. 构造
    std::tuple<char, int, double> t1('a', 10, 3.14);
    auto t2 = std::make_tuple("hello", 42); // 类型推导

    // 2. 访问元素 (std::get)
    std::cout << "Element 0 of t1: " << std::get<0>(t1) << std::endl;
    std::get<2>(t1) = 2.718; // 修改元素

    // 3. 使用函数返回值
    auto result = get_status();
    std::cout << "Status code: " << std::get<0>(result) << std::endl;

    // 4. 使用 std::tie 解构 (C++11/14)
    int code;
    std::string msg;
    bool success;
    std::tie(code, msg, success) = get_status(); // 将 tuple 的成员“绑”到现有变量上
    // std::tie(std::ignore, msg, success) = get_status(); // 忽略第一个值

    // 5. 结构化绑定 (C++17) - 最佳实践
    auto [code_v2, msg_v2, success_v2] = get_status();
    std::cout << "Message: " << msg_v2 << std::endl;
}
```

---

**8. 对比 (`pair` vs. `tuple` vs. `struct`)**
这是一个非常重要的设计决策点。

| 特性 | `std::pair` | `std::tuple` | `struct` / `class` |
| :--- | :--- | :--- | :--- |
| **元素数量** | 2 | 任意 | 任意 |
| **成员访问** | `.first`, `.second` | `std::get<N>()` | `.member_name` |
| **可读性** | 中等 | 差 | **高** |
| **通用性** | 中等 | 高 | 低（类型是固定的） |
| **使用场景** | 键值对，简单的2元组 | 泛型编程，返回多个值的临时聚合 | 具有明确语义的数据模型 |
| **最佳实践** | `std::map` 元素，简单的函数返回值。 | 需要超过2个元素的临时聚合，泛型元编程。**C++17 结构化绑定可提升其可读性。** | 任何有意义的数据聚合，如 `Point{x, y}`，`Person{name, age}`。**代码会自文档化。** |

**结论：如果你的数据有明确的业务含义，优先使用 `struct`。如果只是临时、泛型地捆绑数据（尤其是作为函数返回值），`pair` 和 `tuple` 是很好的工具。**
