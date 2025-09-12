---
title: Effective Modern C++ 笔记（番外）- 智能指针手撕（面试简化版）
date: 2025-04-16 16:49:08
tags:
    - cpp
    - effective-modern-cpp
categories:
    - C++
cover: "/imgs/wallhaven-exmo98.jpg"
excerpt: "Effective Modern C++ 笔记番外篇 - 智能指针手撕"
comment: false
---

> 注：本文由大模型辅助生成

本文将深入探讨 `std::unique_ptr`、`std::shared_ptr` 和 `std::weak_ptr` 的核心机制，并提供一套可以在面试中手写的、简化的 C++ 实现。

这套实现将专注于核心逻辑，以清晰地展示你对这些智能指针工作原理的理解。

---

在 C++ 的世界里，内存管理曾经是一场混乱的战争，充满了内存泄漏（忘记 `delete`）和悬空指针（`delete` 后继续使用）的陷阱。智能指针的出现，如同三位大将，为这场战争带来了秩序和纪律，它们的核心是 **RAII (Resource Acquisition Is Initialization)** 原则——资源的生命周期与对象的生命周期绑定。

### `unique_ptr` —— 独占天下的霸主

**核心思想：独占所有权 (Exclusive Ownership)**

`unique_ptr` 如同一位霸主，它宣称：“这个资源，普天之下，唯我一人所有”。它不允许任何形式的分享或复制，确保在任何时刻，只有一个指针能管理该资源。

**深入讲解：**

1.  **轻量级**：它是一个零成本抽象。在没有自定义删除器的情况下，`sizeof(std::unique_ptr<T>)` 与 `sizeof(T*)` 完全相同。它本质上只是一个带有析构函数的裸指针。
2.  **移动语义**：它的“霸道”体现在禁止拷贝上。你不能复制一个 `unique_ptr`，因为那会产生两个所有者。但你可以**转让（移动）**所有权，就像禅让王位一样。一旦所有权被移走，原来的 `unique_ptr` 就变为空指针 (`nullptr`)。
3.  **RAII**：当 `unique_ptr` 本身被销毁时（例如离开作用域），它的析构函数会自动被调用，从而释放它所拥有的资源（默认调用 `delete`）。

**面试手写实现 `MyUniquePtr`**

```cpp
#include <iostream>
#include <utility> // for std::move and std::forward

template<typename T>
class MyUniquePtr {
private:
    T* ptr_;

public:
    // 构造函数：获取裸指针的所有权。explicit 防止隐式转换。
    explicit MyUniquePtr(T* ptr = nullptr) noexcept : ptr_(ptr) {}

    // 析构函数：RAII核心，自动释放资源
    ~MyUniquePtr() noexcept {
        if (ptr_) {
            delete ptr_;
        }
    }

    // --- 禁止拷贝 ---
    MyUniquePtr(const MyUniquePtr& other) = delete;
    MyUniquePtr& operator=(const MyUniquePtr& other) = delete;

    // --- 实现移动 ---
    // 移动构造函数：从 other 窃取所有权
    MyUniquePtr(MyUniquePtr&& other) noexcept : ptr_(other.ptr_) {
        other.ptr_ = nullptr; // 重要：将源指针置空，防止它析构时重复释放
    }

    // 移动赋值运算符
    MyUniquePtr& operator=(MyUniquePtr&& other) noexcept {
        if (this != &other) { // 防止自赋值
            if (ptr_) {
                delete ptr_; // 释放当前持有的资源
            }
            ptr_ = other.ptr_; // 窃取所有权
            other.ptr_ = nullptr; // 将源指针置空
        }
        return *this;
    }

    // --- 操作符重载 ---
    T& operator*() const { return *ptr_; }
    T* operator->() const { return ptr_; }
    T* get() const { return ptr_; }
    explicit operator bool() const { return ptr_ != nullptr; }

    // --- 修改器 ---
    // 释放所有权，返回裸指针
    T* release() noexcept {
        T* temp = ptr_;
        ptr_ = nullptr;
        return temp;
    }

    // 销毁当前资源，并接管新资源
    void reset(T* ptr = nullptr) noexcept {
        if (ptr_) {
            delete ptr_;
        }
        ptr_ = ptr;
    }
};

// make_unique 的辅助函数
template<typename T, typename... Args>
MyUniquePtr<T> make_my_unique(Args&&... args) {
    return MyUniquePtr<T>(new T(std::forward<Args>(args)...));
}
```

---

### `shared_ptr` —— 联合执政的元老

**核心思想：共享所有权 (Shared Ownership)**

`shared_ptr` 是一位善于合作的元老，它允许：“这个资源，大家都可以用，我们共同管理”。它通过一个精巧的“引用计数”机制来跟踪资源的所有者数量。

**深入讲解：**

1.  **引用计数**：这是 `shared_ptr` 的灵魂。与资源一同存在的还有一个**控制块 (Control Block)**，它至少包含一个**强引用计数**。
    *   每当一个 `shared_ptr` 被拷贝（或新创建一个指向同一资源的 `shared_ptr`），强引用计数 `+1`。
    *   每当一个 `shared_ptr` 被销毁或指向别处，强引用计数 `-1`。
    *   当强引用计数减到 `0` 时，资源被销毁。
2.  **开销**：`shared_ptr` 不是免费的。
    *   **内存开销**：一个 `shared_ptr` 对象通常是裸指针的两倍大，因为它需要一个指针指向资源，另一个指针指向控制块。
    *   **性能开销**：引用计数的增减必须是**原子操作**，以保证多线程安全。原子操作比普通整数操作要慢。
3.  **控制块**：除了强引用计数，控制块还存储了弱引用计数（为 `weak_ptr` 服务）和自定义删除器等信息。

**面试手写实现 `MySharedPtr`**

这个实现相对复杂，关键在于**分离控制块**。

```cpp
#include <atomic> // for std::atomic

// 控制块，用于管理引用计数
struct ControlBlock {
    std::atomic<int> strong_count;
    std::atomic<int> weak_count;

    ControlBlock() : strong_count(1), weak_count(0) {}
    virtual ~ControlBlock() = default;
    virtual void destroy_resource() = 0;
};

// 模板化的控制块，知道如何销毁具体类型的资源
template<typename T>
struct ControlBlockImpl : public ControlBlock {
    T* resource_ptr;

    ControlBlockImpl(T* ptr) : resource_ptr(ptr) {}
    
    void destroy_resource() override {
        delete resource_ptr;
    }
};


template<typename T>
class MyWeakPtr; // 前置声明

template<typename T>
class MySharedPtr {
private:
    T* ptr_;
    ControlBlock* control_block_;
    
    friend class MyWeakPtr<T>; // 允许 weak_ptr 访问内部

    void cleanup() {
        if (control_block_) {
            if (--control_block_->strong_count == 0) {
                control_block_->destroy_resource();
                if (control_block_->weak_count == 0) {
                    delete control_block_;
                }
            }
        }
    }

public:
    explicit MySharedPtr(T* ptr = nullptr) : ptr_(ptr) {
        if (ptr) {
            control_block_ = new ControlBlockImpl<T>(ptr);
        } else {
            control_block_ = nullptr;
        }
    }

    ~MySharedPtr() {
        cleanup();
    }

    // 拷贝构造
    MySharedPtr(const MySharedPtr& other) noexcept 
        : ptr_(other.ptr_), control_block_(other.control_block_) {
        if (control_block_) {
            ++control_block_->strong_count;
        }
    }

    // 拷贝赋值
    MySharedPtr& operator=(const MySharedPtr& other) noexcept {
        if (this != &other) {
            cleanup(); // 先清理自己
            ptr_ = other.ptr_;
            control_block_ = other.control_block_;
            if (control_block_) {
                ++control_block_->strong_count;
            }
        }
        return *this;
    }
    
    // 移动构造
    MySharedPtr(MySharedPtr&& other) noexcept 
        : ptr_(other.ptr_), control_block_(other.control_block_) {
        other.ptr_ = nullptr;
        other.control_block_ = nullptr;
    }

    // 移动赋值
    MySharedPtr& operator=(MySharedPtr&& other) noexcept {
        if (this != &other) {
            cleanup();
            ptr_ = other.ptr_;
            control_block_ = other.control_block_;
            other.ptr_ = nullptr;
            other.control_block_ = nullptr;
        }
        return *this;
    }

    T& operator*() const { return *ptr_; }
    T* operator->() const { return ptr_; }
    T* get() const { return ptr_; }
    int use_count() const { return control_block_ ? control_block_->strong_count.load() : 0; }
};

template<typename T, typename... Args>
MySharedPtr<T> make_my_shared(Args&&... args) {
    return MySharedPtr<T>(new T(std::forward<Args>(args)...));
}
```

---

### `weak_ptr` —— 洞察全局的谋士

**核心思想：非拥有型观察者 (Non-owning Observer)**

`weak_ptr` 是一位谋士，它不带兵（不拥有资源），也不参与执政（不增加引用计数）。它只是静静地观察着 `shared_ptr` 管理的资源，并能告诉你这个资源“是否还健在”。

**深入讲解：**

1.  **解决循环引用**：这是 `weak_ptr` 最重要的使命。当两个对象通过 `shared_ptr` 相互引用时，会形成一个引用环，它们的引用计数永远无法归零，导致内存泄漏。将其中一方的引用改为 `weak_ptr`，即可打破这个环。
2.  **不增加强引用**：`weak_ptr` 从一个 `shared_ptr` 创建，它会共享控制块，并增加**弱引用计数**，但强引用计数不变。
3.  **安全访问**：你不能直接访问 `weak_ptr` 指向的资源。必须先调用 `lock()` 方法。
    *   `lock()` 会检查强引用计数。如果 > 0，它会返回一个指向该资源的有效的 `shared_ptr`（同时强引用计数+1）。
    *   如果 == 0，说明资源已被销毁，`lock()` 返回一个空的 `shared_ptr`。
    这种“先检查后锁定”的机制保证了访问的绝对安全。

**面试手写实现 `MyWeakPtr`**

```cpp
template<typename T>
class MyWeakPtr {
private:
    T* ptr_;
    ControlBlock* control_block_;

public:
    MyWeakPtr() noexcept : ptr_(nullptr), control_block_(nullptr) {}

    MyWeakPtr(const MySharedPtr<T>& sp) noexcept 
        : ptr_(sp.ptr_), control_block_(sp.control_block_) {
        if (control_block_) {
            ++control_block_->weak_count;
        }
    }

    MyWeakPtr(const MyWeakPtr& other) noexcept 
        : ptr_(other.ptr_), control_block_(other.control_block_) {
        if (control_block_) {
            ++control_block_->weak_count;
        }
    }

    MyWeakPtr& operator=(const MyWeakPtr& other) noexcept {
        if (this != &other) {
            cleanup();
            ptr_ = other.ptr_;
            control_block_ = other.control_block_;
            if (control_block_) {
                ++control_block_->weak_count;
            }
        }
        return *this;
    }

    ~MyWeakPtr() {
        cleanup();
    }
    
    // 核心方法：尝试获取资源的 shared_ptr
    MySharedPtr<T> lock() const {
        if (expired()) {
            return MySharedPtr<T>(nullptr);
        }
        // 这里需要一种方法从裸指针和控制块构造 shared_ptr
        // 为简化，我们假设 MySharedPtr 有一个私有构造函数
        // 实际上，这需要更精巧的设计，但为了面试演示，这个逻辑是清晰的
        MySharedPtr<T> sp;
        sp.ptr_ = ptr_;
        sp.control_block_ = control_block_;
        ++control_block_->strong_count;
        return sp;
    }

    bool expired() const {
        return !control_block_ || control_block_->strong_count == 0;
    }
    
private:
    void cleanup() {
        if (control_block_) {
            if (--control_block_->weak_count == 0 && control_block_->strong_count == 0) {
                delete control_block_;
            }
        }
    }
};
```

### 总结与演示

```cpp
// 演示
struct MyData {
    MyData() { std::cout << "MyData created\n"; }
    ~MyData() { std::cout << "MyData destroyed\n"; }
    void greet() { std::cout << "Hello from MyData\n"; }
};

struct Node {
    ~Node() { std::cout << "Node destroyed\n"; }
    // MySharedPtr<Node> next; // 这会造成循环引用
    MyWeakPtr<Node> next;   // 使用 weak_ptr 打破循环
};


void test_unique() {
    std::cout << "\n--- Testing MyUniquePtr ---\n";
    auto u_ptr = make_my_unique<MyData>();
    u_ptr->greet();
} // u_ptr 在此销毁，MyData 被自动 delete

void test_shared() {
    std::cout << "\n--- Testing MySharedPtr ---\n";
    MySharedPtr<MyData> s_ptr1;
    {
        auto s_ptr2 = make_my_shared<MyData>();
        s_ptr1 = s_ptr2;
        std::cout << "Use count: " << s_ptr1.use_count() << std::endl; // 2
    } // s_ptr2 销毁
    std::cout << "Use count: " << s_ptr1.use_count() << std::endl; // 1
} // s_ptr1 销毁, MyData 被 delete

void test_weak_and_cycle() {
    std::cout << "\n--- Testing MyWeakPtr for cycles ---\n";
    auto n1 = make_my_shared<Node>();
    auto n2 = make_my_shared<Node>();
    n1->next = n2;
    n2->next = n1; // 如果 next 是 shared_ptr，这里就泄漏了
    std::cout << "n1 use count: " << n1.use_count() << std::endl; // 1
    std::cout << "n2 use count: " << n2.use_count() << std::endl; // 1
} // n1, n2 销毁，它们指向的 Node 也被正确销毁

int main() {
    test_unique();
    test_shared();
    test_weak_and_cycle();
    return 0;
}
```

这套代码虽然简化了许多细节（如对数组的支持 `T[]`，自定义删除器，`make_shared` 的单次分配优化），但它**完整地、正确地**实现了三种智能指针的核心机制，足以在面试中展示你对现代 C++ 资源管理的深刻理解。