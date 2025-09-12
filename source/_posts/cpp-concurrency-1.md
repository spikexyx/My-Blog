---
title: Modern Cpp 并发编程笔记（1）- 基础篇
date: 2025-04-10 15:24:04
tags:
    - cpp
categories:
    - C++
cover: "/imgs/wallhaven-exmo98.jpg"
excerpt: "本文是对 C++ 并发编程基础部分的总结笔记，包含基础的thread使用、同步原语，以及一些modern cpp的高级抽象等。"
comment: false
---


## 本文大纲

1.  **并发编程基石：`std::thread`**
    *   创建和管理线程
    *   线程的生命周期：`join` vs `detach`
    *   向线程传递参数
2.  **并发编程的痛点：共享数据与竞态条件 (Race Condition)**
    *   什么是竞态条件？
    *   一个经典的错误案例
3.  **解决之道：同步原语 (Synchronization Primitives)**
    *   **互斥锁 Mutex**：`std::mutex` 与 `std::lock_guard`
    *   **更灵活的锁**：`std::unique_lock`
    *   **线程间的通信**：`std::condition_variable`
4.  **更现代、更高级的抽象**
    *   **任务并行**：`std::async`, `std::future`, `std::promise`
    *   **无锁编程入门**：`std::atomic`
5.  **总结与进阶之路**

---

## Part 1: 并发编程基石 `std::thread`

C++11 引入了 `std::thread`，这是我们进行并发编程的起点。它允许你在一个进程中创建多个执行流。

### 1.1 创建和管理线程

一个 `std::thread` 对象在创建时，需要接收一个可调用对象（函数、lambda表达式、函数对象等）作为其入口点。

```cpp
#include <iostream>
#include <thread>
#include <chrono>

void worker_function() {
    for (int i = 0; i < 5; ++i) {
        std::cout << "Worker thread running..." << std::endl;
        std::this_thread::sleep_for(std::chrono::milliseconds(500));
    }
}

int main() {
    std::cout << "Main thread starting..." << std::endl;

    // 创建一个新线程 t，它将执行 worker_function
    std::thread t(worker_function);

    // 主线程继续做其他事情
    for (int i = 0; i < 3; ++i) {
        std::cout << "Main thread running..." << std::endl;
        std::this_thread::sleep_for(std::chrono::milliseconds(300));
    }

    // 等待子线程 t 执行完毕
    t.join(); 

    std::cout << "Main thread finished." << std::endl;
    return 0;
}
```

**讲解**:
*   `#include <thread>` 是必须的。
*   `std::thread t(worker_function);` 创建并立即启动了一个新线程。
*   主线程和 `t` 线程是并发执行的，你会看到它们的输出交织在一起。
*   `t.join()` 是一个关键操作。它会**阻塞**主线程，直到子线程 `t` 执行完毕。**不调用 `join` (或 `detach`) 会导致程序在 `t` 的析构函数中调用 `std::terminate` 而崩溃。**

### 1.2 线程的生命周期：`join` vs `detach`

一个 `std::thread` 对象必须在其生命周期结束前被明确地处理，你有两种选择：

1.  **`join()`**: 如上例所示，等待线程结束。这是最常用、最安全的方式。一个线程只能被 `join` 一次。在调用 `join` 后，`t.joinable()` 会返回 `false`。
2.  **`detach()`**: 将子线程从 `std::thread` 对象中分离，让它在后台独立运行。主线程不再能与它交互，也无法等待它结束。这通常用于“发射后不管”的后台任务，但非常危险，因为你失去了对线程的控制。如果主线程结束，所有分离的线程也会被操作系统强制终止。

**`detach` 的危险示例**:
```cpp
// ... (类似上面的代码)
int main() {
    std::thread t(worker_function);
    t.detach(); // 分离线程
    std::cout << "Main thread finished immediately." << std::endl; 
    // main 函数可能在 worker_function 完成前就结束了
    // 这会导致 worker_function 被突然中断
    return 0; 
}
```
**核心建议**：**优先使用 `join()`**。只有在你非常确定子线程的生命周期与主线程无关，且子线程能自行管理资源时，才考虑 `detach()`。

### 1.3 向线程传递参数

向线程函数传递参数非常直接，但有一个巨大的陷阱：**默认情况下，所有参数都是按值复制的。**

```cpp
#include <iostream>
#include <thread>
#include <string>

void print_message(const std::string& msg) {
    std::cout << "Thread received: " << msg << std::endl;
}

void update_value(int& val) { // 期望通过引用修改
    val = 100;
}

int main() {
    std::string message = "Hello from main";
    std::thread t1(print_message, message); // message 被复制到线程内部

    int my_value = 10;
    // 错误的方式！my_value 仍然是被复制的，线程修改的是一个副本
    // std::thread t2(update_value, my_value); 
    
    // 正确的方式：使用 std::ref 将参数包装成引用
    std::thread t2(update_value, std::ref(my_value));

    t1.join();
    t2.join();

    std::cout << "Final value: " << my_value << std::endl; // 输出 100
    return 0;
}
```
**讲解**:
*   `std::thread` 的构造函数不理解引用。它会复制你传递的所有东西。
*   如果你想传递引用，必须使用 `std::ref()` 包装器。对于 `const` 引用，使用 `std::cref()`。这是为了防止你无意中犯下难以调试的错误。

---

## Part 2: 共享数据与竞态条件 (Race Condition)

当多个线程访问同一个变量，并且至少有一个线程会修改它时，问题就来了。

**竞态条件** (Race Condition)：程序的最终结果取决于多个线程执行的相对顺序。这通常会导致不可预测的、随机的错误。

### 一个经典的错误案例：多线程累加

我们让两个线程同时对一个共享变量各加 100 万次，期望结果是 200 万。

```cpp
#include <iostream>
#include <thread>
#include <vector>

long long counter = 0;

void increment() {
    for (int i = 0; i < 1000000; ++i) {
        counter++;
    }
}

int main() {
    std::thread t1(increment);
    std::thread t2(increment);

    t1.join();
    t2.join();

    std::cout << "Expected: 2000000" << std::endl;
    std::cout << "Actual:   " << counter << std::endl; // 结果几乎肯定不是 2000000
    return 0;
}
```

**为什么错了？**
`counter++` 并非一个原子操作。它至少包含三个步骤：
1.  **读取** `counter` 的当前值到寄存器。
2.  在寄存器中**增加**该值。
3.  将寄存器中的新值**写回** `counter`。

**想象一下这个场景**：
1.  线程1读取 `counter` (值为 5)。
2.  操作系统切换到线程2。
3.  线程2读取 `counter` (值仍然是 5)。
4.  线程2在寄存器中加1，得到6，并写回 `counter`。现在 `counter` 是 6。
5.  操作系统切换回线程1。
6.  线程1（对线程2的操作一无所知）在它的寄存器中加1（基于它之前读取的5），得到6，并写回 `counter`。现在 `counter` 还是 6。

**两个 `++` 操作，结果只增加了1。数据丢失了！** 这就是竞态条件。

---

## Part 3: 解决之道：同步原语

为了解决竞态条件，我们需要确保对共享数据的访问是互斥的，即一次只有一个线程可以访问。

### 3.1 互斥锁 (Mutex)：`std::mutex` 与 `std::lock_guard`

`std::mutex`（互斥量）是最基本的同步原语。它像一个只能由一人持有的令牌。

*   `mtx.lock()`: 尝试获取锁。如果锁已被其他线程持有，则当前线程阻塞，直到锁被释放。
*   `mtx.unlock()`: 释放锁。

**手动管理锁（不推荐）**:
```cpp
std::mutex mtx;
void increment_safe_but_bad() {
    for (int i = 0; i < 1000000; ++i) {
        mtx.lock();
        counter++;
        mtx.unlock();
    }
}
```
这很危险！如果在 `lock()` 和 `unlock()` 之间发生异常，`unlock()` 就不会被调用，导致锁永远不被释放，所有其他等待该锁的线程都会永久阻塞（即**死锁**）。

**现代C++的优雅方案：RAII 与 `std::lock_guard`**

RAII (Resource Acquisition Is Initialization) 是一种C++编程范式，保证在对象创建时获取资源，在对象销毁时释放资源。`std::lock_guard` 就是为 `std::mutex` 设计的RAII包装器。

```cpp
#include <iostream>
#include <thread>
#include <vector>
#include <mutex> // 引入 mutex 头文件

long long counter = 0;
std::mutex mtx; // 创建一个互斥锁实例

void increment_safe() {
    for (int i = 0; i < 1000000; ++i) {
        // 当 lock_guard 被创建时，它会自动调用 mtx.lock()
        std::lock_guard<std::mutex> guard(mtx);
        counter++;
    } // guard 在这里超出作用域，其析构函数会自动调用 mtx.unlock()
}

int main() {
    std::thread t1(increment_safe);
    std::thread t2(increment_safe);
    t1.join();
    t2.join();
    std::cout << "Actual with mutex: " << counter << std::endl; // 正确输出 2000000
    return 0;
}
```
**讲解**:
*   `std::lock_guard<std::mutex> guard(mtx);` 在其构造函数中锁定了 `mtx`。
*   当函数返回或因异常退出时，`guard` 对象的析构函数**保证**会被调用，从而自动释放锁。
*   **黄金法则**：总是使用 `std::lock_guard` 或其他RAII锁来管理 `std::mutex`。

### 3.2 更灵活的锁：`std::unique_lock`

`std::lock_guard` 非常好用，但功能单一。`std::unique_lock` 提供了更多的灵活性，但开销也稍大。

**`std::unique_lock` 的特点**:
*   同样遵循RAII，但它不要求在构造时就锁定。
*   可以手动调用 `lock()` 和 `unlock()`。
*   可以将锁的所有权转移（`std::move`）。
*   可以与 `std::condition_variable` 配合使用（稍后讲解）。

```cpp
std::mutex mtx;
void some_function() {
    std::unique_lock<std::mutex> lock(mtx, std::defer_lock); // 创建但不锁定

    // ... 做一些不需要锁的操作 ...

    lock.lock(); // 手动加锁
    // ... 访问共享资源 ...
    lock.unlock(); // 手动提前解锁，不必等到作用域结束

    // ... 做一些其他不需要锁的操作 ...

    lock.lock(); // 再次加锁
    // ... 再次访问共享资源 ...
} // lock 在析构时会自动解锁（如果它还处于锁定状态）
```
**建议**: 优先使用 `std::lock_guard`，因为它更简单、开销更小。只有当你需要 `std::unique_lock` 提供的额外灵活性时才使用它。

### 3.3 线程间的通信：`std::condition_variable`

有时候，一个线程需要等待某个**条件**达成才能继续执行，而这个条件是由另一个线程改变的。例如，生产者-消费者模型。

*   **生产者**：向队列中添加数据。
*   **消费者**：从队列中取出数据。

如果队列是空的，消费者应该**等待**，而不是空转浪费CPU。这就是 `std::condition_variable` 的用武之地。

`std::condition_variable` 总是与 `std::mutex` 和 `std::unique_lock` 一起使用。
*   `cv.wait(lock, predicate)`: 原子地释放 `lock` 并让当前线程休眠。当被唤醒时，它会重新获取 `lock` 并检查 `predicate`（一个返回 `bool` 的 lambda）。如果 `predicate` 为 `true`，`wait` 返回；否则继续休眠。
*   `cv.notify_one()`: 唤醒一个正在等待的线程。
*   `cv.notify_all()`: 唤醒所有正在等待的线程。

**生产者-消费者示例**:
```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>

std::mutex mtx;
std::condition_variable cv;
std::queue<int> data_queue;
bool finished = false;

void producer() {
    for (int i = 0; i < 10; ++i) {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        {
            std::lock_guard<std::mutex> lock(mtx);
            std::cout << "Producer pushing " << i << std::endl;
            data_queue.push(i);
        } // 锁在这里释放
        cv.notify_one(); // 唤醒一个等待的消费者
    }
    {
        std::lock_guard<std::mutex> lock(mtx);
        finished = true;
    }
    cv.notify_all(); // 唤醒所有可能在等待的消费者，告知生产结束
}

void consumer() {
    while (true) {
        std::unique_lock<std::mutex> lock(mtx);
        // wait会检查lambda，如果为false，则解锁并休眠
        // 被唤醒后，重新加锁，再次检查lambda
        cv.wait(lock, []{ return !data_queue.empty() || finished; });

        if (!data_queue.empty()) {
            int data = data_queue.front();
            data_queue.pop();
            std::cout << "Consumer popped " << data << std::endl;
        } else if (finished) {
            std::cout << "Consumer finished." << std::endl;
            break;
        }
    }
}

int main() {
    std::thread p(producer);
    std::thread c1(consumer);
    std::thread c2(consumer);

    p.join();
    c1.join();
    c2.join();

    return 0;
}
```
**讲解**:
*   消费者使用 `std::unique_lock` 因为 `wait` 需要能够解锁和重新锁定它。
*   `wait` 的 lambda `[]{ return !data_queue.empty() || finished; }` 至关重要。它处理了“虚假唤醒”（spurious wakeups），并正确地处理了生产结束的逻辑。线程被唤醒后必须重新检查条件，确保不是被错误地唤醒。

---

## Part 4: 更现代、更高级的抽象

直接操作 `std::thread` 和 `std::mutex` 有时很繁琐，特别是当你只关心一个任务的结果时。

### 4.1 任务并行：`std::async`, `std::future`, `std::promise`

**`std::async`**: 启动一个异步任务，并返回一个 `std::future` 对象。它让你像调用一个普通函数一样启动一个后台任务，并方便地获取其返回值。

**`std::future`**: 一个“未来的凭证”。你可以用它来查询异步任务是否完成，并用 `get()` 方法阻塞并获取结果。`get()` 只能被调用一次。

```cpp
#include <iostream>
#include <future> // 引入 future 头文件
#include <chrono>

int long_computation(int x) {
    std::cout << "Starting long computation..." << std::endl;
    std::this_thread::sleep_for(std::chrono::seconds(2));
    return x * x;
}

int main() {
    std::cout << "Main thread starting a task." << std::endl;
    
    // 启动一个异步任务。std::launch::async 保证在新线程中运行
    std::future<int> fut = std::async(std::launch::async, long_computation, 10);

    std::cout << "Main thread doing other work." << std::endl;
    // ...

    std::cout << "Main thread waiting for result." << std::endl;
    int result = fut.get(); // 阻塞，直到 long_computation 完成并返回结果

    std::cout << "Result is: " << result << std::endl;
    return 0;
}
```
**讲解**:
*   `std::async` 比手动创建 `std::thread`、传递引用、设置 `std::promise` 要简单得多。它为你处理了所有细节。
*   `std::future` 优雅地解决了从线程获取返回值的问题。

**`std::promise`**: 当 `std::async` 不够灵活时，你可以使用 `std::promise` 和 `std::future` 对。`promise` 用于在一个线程中**设置**值，`future` 用于在另一个线程中**获取**该值。

```cpp
std::promise<int> p;
std::future<int> f = p.get_future();

std::thread t([&p]{
    try {
        // ... do work ...
        p.set_value(42); // 设置值
    } catch (...) {
        p.set_exception(std::current_exception()); // 或者设置异常
    }
});

// 在主线程
int result = f.get(); // 等待并获取值
t.join();
```

### 4.2 无锁编程入门：`std::atomic`

对于非常简单的操作（如递增计数器、设置布尔标志），使用互斥锁的开销可能过大。`std::atomic` 为此而生。

`std::atomic<T>` 模板对类型 `T` 的所有操作都变成**原子操作**，即不可分割的。

让我们用 `std::atomic` 重新实现那个错误的累加器：

```cpp
#include <iostream>
#include <thread>
#include <atomic> // 引入 atomic 头文件

std::atomic<long long> atomic_counter = 0; // 使用原子类型

void atomic_increment() {
    for (int i = 0; i < 1000000; ++i) {
        atomic_counter++; // 这个操作是原子的
    }
}

int main() {
    std::thread t1(atomic_increment);
    std::thread t2(atomic_increment);

    t1.join();
    t2.join();

    std::cout << "Actual with atomic: " << atomic_counter << std::endl; // 正确输出 2000000
    return 0;
}
```
**讲解**:
*   `std::atomic<long long>` 保证了 `atomic_counter++` 在硬件级别是原子的。它通常比使用互斥锁快得多，因为它避免了线程阻塞和上下文切换。
*   **适用场景**：`std::atomic` 非常适合用于简单的标志、计数器或指针。对于复杂数据结构（如 `std::vector`）的保护，你仍然需要使用互斥锁。

---

## 总结与进阶之路

本文总结了现代C++并发编程的核心工具集：

*   **`std::thread`**: 创建和管理执行流。
*   **`std::mutex` / `std::lock_guard`**: 保护共享数据，防止竞态条件。
*   **`std::condition_variable`**: 实现线程间的等待/通知机制。
*   **`std::async` / `std::future`**: 进行简单的任务并行和获取结果。
*   **`std::atomic`**: 对简单类型进行高效、无锁的原子操作。

**下一步学什么？**

1.  **死锁 (Deadlock)**: 了解死锁产生的原因（两个或更多线程互相等待对方释放锁）和避免方法（如 `std::scoped_lock` (C++17)，它可以同时锁定多个互斥锁而不会死锁）。
2.  **线程池 (Thread Pool)**: 在实际应用中，频繁创建和销毁线程开销很大。学习如何实现和使用线程池来复用线程。
3.  **C++17并行算法**: C++17 在 `<algorithm>` 和 `<numeric>` 中引入了执行策略（如 `std::execution::par`），可以轻松地将标准算法并行化。
4.  **C++20新特性**:
    *   **`std::jthread`**: 一个支持自动 `join` 的 `std::thread`，更安全。
    *   **协程 (Coroutines)**: 一种更轻量级的并发模型，适合大规模I/O密集型任务。
    *   **`std::latch` 和 `std::barrier`**: 新的、更简单的同步原语。
5.  **内存模型 (Memory Model)**: 这是一个非常深入的话题，涉及到底层硬件如何保证不同线程之间的内存可见性。`std::atomic` 的 `memory_order` 参数就与此相关。

**推荐书籍**：
*   **《C++ Concurrency in Action, 2nd Edition》 by Anthony Williams**
> https://beefnoodles.cc/assets/book/C++%20Concurrency%20in%20Action.pdf