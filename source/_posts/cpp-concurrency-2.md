---
title: Modern Cpp 并发编程笔记（2）- 进阶篇1
date: 2025-04-11 17:37:41
tags:
    - cpp
categories:
    - C++
cover: "/imgs/wallhaven-exmo98.jpg"
excerpt: "本文是对 C++ 并发编程一些进阶知识的学习笔记，包含死锁处理、线程池、C++17&20 新特性等。"
comment: false
---

## 1. 死锁 (Deadlock)

### 1.1 概念与原因

**死锁**是指两个或更多的线程在执行过程中，因争夺资源而造成的一种互相等待的现象，若无外力作用，它们都将无法推进下去。

产生死锁的四个必要条件（必须同时满足）：
1.  **互斥** (Mutual Exclusion): 资源不能被共享，一次只能被一个线程使用（`std::mutex` 就满足这个条件）。
2.  **持有并等待** (Hold and Wait): 一个线程至少持有一个资源，并且正在等待获取另一个被其他线程持有的资源。
3.  **非抢占** (No Preemption): 资源不能被强制性地从持有它的线程中抢占。
4.  **循环等待** (Circular Wait): 存在一个线程资源的循环等待链，例如 T1 等待 T2 的资源，T2 等待 T1 的资源。

### 1.2 一个经典的死锁示例

想象一个银行转账的场景，从账户A转账到账户B。为了保证数据一致性，你需要同时锁定两个账户。

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <string>

struct Account {
    double balance;
    std::mutex mtx;
};

void transfer(Account& from, Account& to, double amount) {
    std::cout << "Transfering " << amount << " from " << &from << " to " << &to 
              << " on thread " << std::this_thread::get_id() << std::endl;

    // 锁住转出账户
    std::lock_guard<std::mutex> lock_from(from.mtx);
    std::cout << "Thread " << std::this_thread::get_id() << " locked 'from' account (" << &from << ")" << std::endl;

    // 制造一点延迟，让死锁更容易复现
    std::this_thread::sleep_for(std::chrono::milliseconds(10)); 

    // 锁住转入账户
    std::cout << "Thread " << std::this_thread::get_id() << " trying to lock 'to' account (" << &to << ")" << std::endl;
    std::lock_guard<std::mutex> lock_to(to.mtx);
    std::cout << "Thread " << std::this_thread::get_id() << " locked 'to' account (" << &to << ")" << std::endl;


    from.balance -= amount;
    to.balance += amount;
    
    std::cout << "Transfer complete." << std::endl;
}

int main() {
    Account a{1000.0};
    Account b{1000.0};

    // 线程1: 从 A 转到 B
    std::thread t1(transfer, std::ref(a), std::ref(b), 100.0);
    // 线程2: 从 B 转到 A (这是关键)
    std::thread t2(transfer, std::ref(b), std::ref(a), 50.0);

    t1.join();
    t2.join();

    std::cout << "Final balances: A=" << a.balance << ", B=" << b.balance << std::endl;
    return 0;
}
```

**死锁过程分析**:
1.  线程 `t1` 启动，执行 `transfer(a, b, ...)`，成功锁住 `a.mtx`。
2.  线程 `t2` 启动，执行 `transfer(b, a, ...)`，成功锁住 `b.mtx`。
3.  现在，`t1` 尝试锁住 `b.mtx`，但它被 `t2` 持有，于是 `t1` 阻塞。
4.  同时，`t2` 尝试锁住 `a.mtx`，但它被 `t1` 持有，于是 `t2` 也阻塞。
5.  `t1` 等待 `t2`，`t2` 等待 `t1`。**死锁发生**。程序会永远卡住。

### 1.3 避免死锁的方法

**方法一：保证锁的顺序**
破坏“循环等待”条件。规定所有线程都必须按照相同的顺序来获取锁。例如，我们可以规定，总是先锁地址较小的那个 `Account`。

```cpp
void transfer_safe(Account& from, Account& to, double amount) {
    // 按地址大小决定锁的顺序
    if (&from < &to) {
        std::lock_guard<std::mutex> lock_from(from.mtx);
        std::lock_guard<std::mutex> lock_to(to.mtx);
        // ... transfer logic ...
    } else {
        std::lock_guard<std::mutex> lock_to(to.mtx);
        std::lock_guard<std::mutex> lock_from(from.mtx);
        // ... transfer logic ...
    }
}
```
这种方法可行，但很繁琐且容易出错。

**方法二：使用 `std::scoped_lock` (C++17，推荐)**

> https://en.cppreference.com/w/cpp/thread/scoped_lock.html

C++17 提供了完美的解决方案：`std::scoped_lock`。它是一个可变参数模板，可以一次性锁定多个互斥锁，并且内部使用了**死锁避免算法**来保证安全。

```cpp
#include <scoped_lock> // C++17

void transfer_modern(Account& from, Account& to, double amount) {
    // 一次性、安全地锁定两个互斥锁
    std::scoped_lock lock(from.mtx, to.mtx); 
    
    // 锁已全部获取，可以安全地进行操作
    from.balance -= amount;
    to.balance += amount;
    
    std::cout << "Transfer complete on thread " << std::this_thread::get_id() << std::endl;
} // lock 在析构时会自动释放两个互斥锁
```
`std::scoped_lock` 保证了在所有给定的互斥锁都被成功锁定之前，不会发生死锁。它要么成功锁定所有互斥锁，要么一个也不锁。它同样遵循RAII，是现代C++中处理多锁问题的首选。

---

## 2. 线程池 (Thread Pool)

### 2.1 概念与原因

在需要处理大量短时任务的应用中（如Web服务器处理请求），如果为每个任务都创建一个新线程，开销会非常大。线程的创建和销毁涉及操作系统调用，会消耗大量时间和资源。

**线程池**通过维护一个固定数量的工作线程来解决这个问题。任务被提交到一个任务队列中，工作线程则不断地从队列中取出任务并执行。这**复用**了线程，避免了频繁创建和销毁的开销。

### 2.2 一个简化的线程池实现

一个基础的线程池需要以下组件：
*   一个工作线程的容器 (`std::vector<std::thread>`)。
*   一个线程安全的任务队列 (`std::queue` + `std::mutex`)。
*   一个条件变量 (`std::condition_variable`)，用于在队列为空时让工作线程休眠，在有新任务时唤醒它们。
*   一个停止标志，用于安全地关闭线程池。

```cpp
#include <vector>
#include <queue>
#include <functional>
#include <future>

class ThreadPool {
public:
    ThreadPool(size_t num_threads) : stop(false) {
        for (size_t i = 0; i < num_threads; ++i) {
            workers.emplace_back([this] {
                while (true) {
                    std::function<void()> task;
                    {
                        std::unique_lock<std::mutex> lock(this->queue_mutex);
                        this->condition.wait(lock, [this] { 
                            return this->stop || !this->tasks.empty(); 
                        });

                        if (this->stop && this->tasks.empty()) {
                            return;
                        }

                        task = std::move(this->tasks.front());
                        this->tasks.pop();
                    }
                    task();
                }
            });
        }
    }

    template<class F, class... Args>
    auto enqueue(F&& f, Args&&... args) -> std::future<typename std::result_of<F(Args...)>::type> {
        using return_type = typename std::result_of<F(Args...)>::type;

        auto task = std::make_shared<std::packaged_task<return_type()>>(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...)
        );
            
        std::future<return_type> res = task->get_future();
        {
            std::lock_guard<std::mutex> lock(queue_mutex);
            if(stop) throw std::runtime_error("enqueue on stopped ThreadPool");
            tasks.emplace([task](){ (*task)(); });
        }
        condition.notify_one();
        return res;
    }

    ~ThreadPool() {
        {
            std::lock_guard<std::mutex> lock(queue_mutex);
            stop = true;
        }
        condition.notify_all();
        for (std::thread &worker : workers) {
            worker.join();
        }
    }

private:
    std::vector<std::thread> workers;
    std::queue<std::function<void()>> tasks;
    
    std::mutex queue_mutex;
    std::condition_variable condition;
    bool stop;
};

// 使用示例
int main() {
    ThreadPool pool(4); // 创建一个有4个工作线程的线程池

    // 提交一些任务
    auto future1 = pool.enqueue([](int a, int b) { 
        std::this_thread::sleep_for(std::chrono::seconds(1));
        return a + b; 
    }, 5, 3);
    
    auto future2 = pool.enqueue([]{ 
        std::cout << "Task 2 running on thread " << std::this_thread::get_id() << std::endl;
        return "Hello from Task 2";
    });

    std::cout << "Main thread continues..." << std::endl;
    
    std::cout << "Result of Task 1: " << future1.get() << std::endl;
    std::cout << "Result of Task 2: " << future2.get() << std::endl;
    
    return 0; // 线程池在析构时会自动清理和join所有线程
}
```

**讲解**:
*   **构造函数**: 创建指定数量的线程，每个线程都进入一个无限循环的`worker`函数。
*   **`worker`循环**:
    *   使用 `std::condition_variable::wait` 等待任务。`wait`会原子性地解锁互斥锁并使线程休眠，避免了空转。
    *   被唤醒后，重新加锁，从队列中取出一个任务，解锁，然后执行任务。
*   **`enqueue`**: 这是向线程池提交任务的接口。
    *   它使用 `std::packaged_task` 将任意函数和参数包装成一个可调用对象，并关联一个 `std::future`。
    *   将包装好的任务放入队列，然后调用 `condition.notify_one()` 唤醒一个正在等待的工作线程。
*   **析构函数**: 设置 `stop` 标志，唤醒所有线程，然后 `join` 它们，确保所有线程都已安全退出。

---

## 3. C++17 并行算法

### 3.1 概念

C++17 对标准库中的许多算法（如 `for_each`, `sort`, `transform`, `accumulate`）进行了扩展，允许它们以并行或向量化的方式执行。你只需要向算法传递一个**执行策略 (Execution Policy)** 参数即可。

**执行策略**:
*   `std::execution::seq`: 顺序执行（默认行为）。
*   `std::execution::par`: 并行执行。允许库在多个线程上并发执行算法。
*   `std::execution::par_unseq`: 并行和向量化执行。不仅可以在多线程上执行，还允许单个线程内的指令重排（SIMD）。这是最强的优化，但要求也最苛刻（循环体内的操作不能有依赖，不能有锁）。

### 3.2 示例

假设我们需要对一个巨大的 vector 中的每个元素进行一个耗时的计算。

```cpp
#include <iostream>
#include <vector>
#include <numeric> // for std::accumulate
#include <algorithm> // for std::for_each
#include <execution> // for execution policies
#include <chrono>

long long expensive_computation(long long val) {
    // 模拟一个耗时操作
    return val * val;
}

int main() {
    std::vector<long long> v(10'000'000);
    std::iota(v.begin(), v.end(), 1); // 用 1, 2, 3, ... 填充 vector

    auto start = std::chrono::high_resolution_clock::now();

    // C++17 之前：串行执行
    // std::transform(v.begin(), v.end(), v.begin(), expensive_computation);
    
    // C++17 之后：并行执行
    std::transform(std::execution::par, v.begin(), v.end(), v.begin(), expensive_computation);

    auto end = std::chrono::high_resolution_clock::now();
    std::chrono::duration<double> diff = end - start;
    std::cout << "Parallel transform took: " << diff.count() << " s\n";

    // 另一个例子: 并行求和
    start = std::chrono::high_resolution_clock::now();
    long long sum = std::reduce(std::execution::par, v.begin(), v.end()); // C++17 新增的 reduce
    // 注意: std::accumulate 不保证可以并行, std::reduce 可以
    end = std::chrono::high_resolution_clock::now();
    diff = end - start;
    std::cout << "Parallel reduce took: " << diff.count() << " s\n";
    std::cout << "Sum: " << sum << std::endl;
    
    return 0;
}
```
**讲解**:
*   你所需要做的仅仅是添加 `std::execution::par` 这个参数。
*   标准库的实现（通常依赖底层的线程池）会负责将数据分块，分配给不同线程，然后合并结果。
*   **重要警告**: 只有在循环体内的操作是相互独立、没有副作用、不访问共享可变状态时，并行算法才是安全的。如果你在 `for_each` 的 lambda 中操作一个没有锁保护的全局变量，那么你又会遇到竞态条件！

---

## 4. C++20 新特性

C++20 在并发方面带来了革命性的改进。

### 4.1 `std::jthread`

**问题**: `std::thread` 的析构函数如果发现线程仍是 `joinable()`（即未被 `join` 或 `detach`），会调用 `std::terminate` 中止程序。这是个常见的陷阱。

**解决方案**: `std::jthread` (joining thread) 是一个遵循 RAII 的 `std::thread` 封装。它的析构函数会自动 `join()` 线程。

```cpp
#include <iostream>
#include <thread> // for jthread

void worker() {
    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << "Worker finished." << std::endl;
}

void run() {
    std::cout << "Entering run()..." << std::endl;
    std::jthread jt(worker);
    // 无需手动调用 jt.join()
    std::cout << "Exiting run()..." << std::endl;
} // jt 在这里超出作用域，其析构函数会自动调用 join()

int main() {
    run();
    std::cout << "run() has finished." << std::endl;
    return 0;
}
```
**更强大的功能：协作式取消**
`std::jthread` 内置了协作式取消机制。每个 `jthread` 都有一个 `std::stop_token`。你可以在线程函数中检查这个 token，以响应外部的停止请求。

```cpp
void cancellable_worker(std::stop_token st) {
    int i = 0;
    while (!st.stop_requested()) { // 检查是否被请求停止
        std::cout << "Worker running... " << i++ << std::endl;
        std::this_thread::sleep_for(std::chrono::milliseconds(200));
    }
    std::cout << "Worker was requested to stop." << std::endl;
}

int main() {
    std::jthread jt(cancellable_worker);
    
    std::this_thread::sleep_for(std::chrono::seconds(1));
    
    std::cout << "Main thread requesting stop." << std::endl;
    jt.request_stop(); // 请求线程停止
    
    // jt 的析构函数会自动 join，等待线程真正结束
    return 0;
}
```

### 4.2 协程 (Coroutines)

**概念**: 协程是一种比线程更轻量级的并发模型。你可以把它看作是一个**可以暂停和恢复**的函数。当协程等待一个耗时操作（如网络I/O）时，它会**暂停**自己（`co_await`），将执行权交还给调用者，而**不阻塞**整个线程。当操作完成后，协程可以在暂停点恢复执行。

这对于编写高并发、I/O密集型（如服务器）的应用极具价值，可以用少量线程处理大量并发连接。

**关键词**:
*   `co_await`: 暂停协程，等待一个异步操作完成。
*   `co_yield`: (用于生成器) 产生一个值并暂停。
*   `co_return`: 从协程返回值。

**示例（概念性）**:
协程的完整实现依赖于库（如`cppcoro`或`asio`），因为它需要复杂的 `promise_type` 定义。下面是一个展示其**使用方式**的伪代码风格的例子：

```cpp
// 假设我们有一个返回 future 的异步网络库
std::future<std::string> async_fetch(const std::string& url);

// 这是一个协程
task<void> fetch_and_process() {
    std::cout << "Fetching data from url1..." << std::endl;
    std::string data1 = co_await async_fetch("http://example.com/data1"); // 暂停，不阻塞线程
    std::cout << "Got data1: " << data1.substr(0, 30) << "..." << std::endl;

    std::cout << "Fetching data from url2..." << std::endl;
    std::string data2 = co_await async_fetch("http://example.com/data2"); // 再次暂停
    std::cout << "Got data2: " << data2.substr(0, 30) << "..." << std::endl;
    
    process(data1, data2);
    co_return;
}

int main() {
    // 启动协程，它会立即返回
    fetch_and_process(); 
    // 主线程可以继续做其他事情，而网络请求在后台进行
    // 事件循环会负责在数据到达时恢复协程的执行
    event_loop.run(); 
}
```
**核心优势**: 代码看起来是同步的、线性的，但实际上是完全异步、非阻塞的。这极大地简化了异步编程的复杂性（告别回调地狱）。

### 4.3 `std::latch` 和 `std::barrier`

这两个是更简单的同步原语。

*   **`std::latch`**: 一次性同步点。可以把它想象成一个倒计时门闩。初始化时指定一个计数值。线程可以通过 `count_down()` 使计数减一。所有调用 `wait()` 的线程都会阻塞，直到计数变为0。一旦变为0，门闩就永久打开了。
    *   **用途**: 等待多个工作线程完成它们的初始化工作。

```cpp
std::latch work_ready(3); // 等待3个线程

void worker_latch(int id) {
    std::cout << "Worker " << id << " doing init work..." << std::endl;
    std::this_thread::sleep_for(std::chrono::milliseconds(id * 100));
    std::cout << "Worker " << id << " ready." << std::endl;
    work_ready.count_down(); // 完成，计数减一
    // ... 可以继续做其他事
}

int main() {
    std::jthread t1(worker_latch, 1);
    std::jthread t2(worker_latch, 2);
    std::jthread t3(worker_latch, 3);
    
    std::cout << "Main thread waiting for all workers to be ready..." << std::endl;
    work_ready.wait(); // 阻塞，直到计数为0
    
    std::cout << "All workers are ready. Main thread continues." << std::endl;
    return 0;
}
```

*   **`std::barrier`**: 可重用同步点。与latch类似，但它可以在所有线程通过后**重置**，用于下一轮同步。这非常适合迭代式、分阶段的计算。
    *   **用途**: 在并行算法的每个迭代步骤中同步所有线程。

```cpp
int main() {
    const int num_threads = 3;
    // 创建一个 barrier，计数值为 num_threads，
    // 并在每轮同步完成时执行一个 lambda
    std::barrier sync_point(num_threads, []() noexcept {
        std::cout << "--- Phase Complete --- \n\n";
    });

    auto work = [&](int id){
        // Phase 1
        std::cout << "Thread " << id << " starting phase 1...\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(id * 50));
        sync_point.arrive_and_wait(); // 等待所有线程完成 phase 1

        // Phase 2
        std::cout << "Thread " << id << " starting phase 2...\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(id * 60));
        sync_point.arrive_and_wait(); // 等待所有线程完成 phase 2
    };

    std::vector<std::jthread> threads;
    for(int i=0; i<num_threads; ++i) threads.emplace_back(work, i);
    
    return 0;
}
```

---

## 5. 内存模型 (Memory Model)

这是C++并发中最深入、最底层的部分。

### 5.1 概念

**问题**: 为了性能，编译器和CPU都会对指令进行重排。在单线程中，这种重排只要不改变最终结果就是安全的。但在多线程中，一个线程看到的另一个线程的内存操作顺序，可能与代码中写的顺序不一致。

**C++内存模型**是一套规则，它精确定义了在一个线程中对内存的修改，何时能被其他线程看到。它在程序员和硬件/编译器之间建立了一个契约。`std::atomic` 的 `memory_order` 参数就是你用来与这个模型交互的工具。

### 5.2 `std::memory_order`

*   **`memory_order_relaxed`**: 最弱的顺序。只保证操作本身的原子性，不提供任何跨线程的顺序保证。
    *   **用途**: 简单的计数器，你只关心最终结果，不关心中间过程。

*   **`memory_order_release`** (用于写操作) / **`memory_order_acquire`** (用于读操作):
    *   这是最常见的配对。它们用于同步两个线程。
    *   **`release`**: 它像一道“释放屏障”。在此屏障**之前**的所有内存写入，对于之后执行了相应`acquire`操作的线程来说，都是可见的。
    *   **`acquire`**: 它像一道“获取屏障”。在此屏障**之后**的所有内存读取，都能看到之前执行了相应`release`操作的线程写入的数据。

    **示例：使用 acquire-release 实现安全的生产者-消费者**
    ```cpp
    std::atomic<bool> data_ready{false};
    std::string shared_data;

    void producer() {
        shared_data = "Hello from producer!";
        // 写入数据 *必须* 发生在 store 之前
        data_ready.store(true, std::memory_order_release);
    }

    void consumer() {
        // 等待 data_ready 变为 true
        while (!data_ready.load(std::memory_order_acquire)) {
            // spin wait
        }
        // 一旦 load 返回 true, 对 shared_data 的读取是安全的
        std::cout << "Consumer received: " << shared_data << std::endl;
    }
    ```
    `release`操作确保了`shared_data`的写入对`consumer`是可见的。如果用`relaxed`，`consumer`可能看到`data_ready`是`true`，但`shared_data`仍然是空的！

*   **`memory_order_seq_cst` (Sequentially Consistent)**: 最强的顺序，也是所有原子操作的默认值。
    *   它不仅提供 acquire-release 的保证，还保证所有线程看到的**所有** `seq_cst` 操作的顺序是一致的，仿佛它们都在一个全局的总线上按序执行。
    *   **优点**: 最容易推理，最不容易出错。
    *   **缺点**: 在某些架构上可能比 acquire-release 慢，因为它限制了更多的重排。
    *   **黄金法则**: **如果你不确定用哪种内存顺序，就用默认的 `seq_cst`。** 只有在性能分析表明原子操作是瓶颈，并且你完全理解弱内存模型的后果时，才去考虑使用更弱的顺序。
