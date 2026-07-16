---
title: 内存模型
date: 2026-07-15 10:00:00 +0800
categories: [技术]
tags: [C++， Memory Model， Memory Order]
pin: true
---
## 什么是内存模型

Memory Model，是C++ 11引入的一个重要概念，它定义了

> 多线程环境下，不同线程如何观察内存，以及编译器和CPU可以做哪些优化。

### 1 为什么需要内存模型

* 硬件层面的乱序执行：为了提高CPU利用率，CPU会进行指令重排，且存在多级缓存（L1,L2,L3）和写缓冲区（Write Buffer）。导致不两只核心看到的内存修改顺序可能不同。
* 编译器优化：为优化性能，编译器会重排没有依赖关系的指令。
* C++内存模型的职责：它是一份契约。规定了编写多线程代码时，哪些是编译器和硬件必须保证的，哪引起是程序员必须保证的。

先来看个单线程的例子。

```C++
int a = 1;
int b = 2;
```

程序员看到 `a=1 -> b =2` 顺序执行。

实际上由于编译器优化、CPU乱序执行、Cache延迟同步等，机器执行可能变成 `b = 2 -> a = 1` 的顺序。

这对于单线程并无影响，因为结果一样。

多线程问题。

```c++
int data = 0;
bool ready = false;

# Thread A
data = 100;
ready = true;

# Thread B
while(!ready);
std::cout << data;
```

我们以为一定输出100，实际可能输出0。
因为 `data = 100; ready = true;` 可能被CPU重排序为 `ready = true; data = 100`。
线程B看到 ready 为 true, 而 data 为 0。

这就是经典的可见性问题。

### 2 数据竞争与未定义行为

* 定义：当多个线程同时访问一个内存位置，且至少有一个是写操作，同时他们之间没有使用同步机制（互斥锁或原子操作）来建立顺序关系时，就发生了数据竞争。
* C++标准规定

> Data Race = UB （Undefined Behavior）

 比如这里的读写x不是原子操作，可能结果是`1 1`,也可能`1 2`,甚至出现奇怪的问题。

```c++
 int x = 0;
 # Thread 1
 x++;

 # Thread 2
 x++;
```

 可以使用Atomic来实现安全

```c++
 std::atomic<int> cnt{0};
 cnt++;
```

 实际相当于`lock, read, modify, write, unlock`。
 这样一来，多个线程的`cnt++`就不会发生覆盖。


### 3 修改顺序

* Modification Order: 每个原子变量都有自己的修改顺序。所有线程看到该变量所有写操作的顺序必须是一致的。哪怕线程A看到的比线程B慢，但他们看到的数值演变方向必须是一致的。


## 内存顺序（Memory Order）

C++11引入了std::memory_order，提供了6种不同的内存顺序选项，用于平衡性能和可见性。

### 1 memory_order_relaxed  （松散顺序）

特性： 仅保证当前操作的原子性，不提供任何跨线程的同步或推导顺序。编译器和CPU可以任意重排非原子代码。

应用场景： 纯计数器（如 std::shared_ptr 的引用计数增加，但不包括减少到0时的析构）。

```c++
// ====================================================================
// 1. memory_order_relaxed: 只保证"原子性", 不保证任何顺序/可见性关系
// ====================================================================
void demo_relaxed_counter() {
    section("1. memory_order_relaxed —— 只有原子性, 没有同步语义");

    std::atomic<long> counter{0};
    constexpr int kThreads = 4;
    constexpr int kIter = 500000;

    std::vector<std::thread> ts;
    for (int i = 0; i < kThreads; ++i) {
        ts.emplace_back([&] {
            for (int j = 0; j < kIter; ++j) {
                counter.fetch_add(1, std::memory_order_relaxed); // 只要求不丢数据
            }
        });
    }
    for (auto& t : ts) t.join();

    std::cout << kThreads << " 个线程各自 relaxed fetch_add " << kIter << " 次\n";
    std::cout << "counter = " << counter.load()
              << " (期望 " << (long)kThreads * kIter << ")\n";
    std::cout << "结论: relaxed 保证 fetch_add 不会因为并发而“丢加”(原子性),\n"
                 "      但它不建立 happens-before 关系, 不能用来同步其它变量的可见性。\n";
}
```

输出：

```
========== 1. memory_order_relaxed —— 只有原子性, 没有同步语义 ==========
4 个线程各自 relaxed fetch_add 500000 次
counter = 2000000 (期望 2000000)
结论: relaxed 保证 fetch_add 不会因为并发而“丢加”(原子性),
      但它不建立 happens-before 关系, 不能用来同步其它变量的可见性。
```

### 2 memory_order_seq_cst （顺序一致性）

特性： 默认选项。最严格的模型。所有线程看到的全局操作顺序完全一致，如同所有操作都在一个单核CPU上串行执行一样。它还在所有线程间建立了一个全局唯一的总线顺序。

代价： 性能损耗最大（会插入大量的内存屏障/Memory Barrier指令）。

```c++
void demo_pure_seq_cst() {
    section("2. memory_order_seq_cst 示例 —— Peterson 互斥锁(全程只用 seq_cst)");

    PetersonLock lock;
    long counter = 0;                     // 故意用非原子的 long, 完全依赖锁来保护
    constexpr long kIter = 2000000;

    std::thread t0([&] {
        for (long i = 0; i < kIter; ++i) {
            lock.lock(0);
            ++counter;                    // 临界区: 如果互斥失效, 这里就会丢计数
            lock.unlock(0);
        }
    });
    std::thread t1([&] {
        for (long i = 0; i < kIter; ++i) {
            lock.lock(1);
            ++counter;
            lock.unlock(1);
        }
    });
    t0.join();
    t1.join();

    std::cout << "两个线程各自加锁/解锁 " << kIter << " 次, 临界区里对同一个非原子变量 ++counter\n";
    std::cout << "counter = " << counter << " (期望 " << 2 * kIter << ")\n";
    std::cout << "结论: 全程只用 memory_order_seq_cst, 不需要像 acquire/release 那样操心\n"
                 "      “谁 release、谁 acquire、跟哪个变量配对”, 只要每个原子操作都是\n"
                 "      seq_cst, 直接按“好像所有操作真的按某个全局顺序依次执行”去推理即可——\n"
                 "      这正是 seq_cst 被定为默认值(不写 memory_order 参数时就是它)的原因:\n"
                 "      心智负担最小、最不容易用错, 代价是每个操作都带一次全内存屏障, 更慢。\n";
}
```

输出

```
========== 2. 纯 memory_order_seq_cst 示例 —— Peterson 互斥锁(全程只用 seq_cst) ==========
两个线程各自加锁/解锁 2000000 次, 临界区里对同一个非原子变量 ++counter
counter = 4000000 (期望 4000000)
结论: 全程只用 memory_order_seq_cst, 不需要像 acquire/release 那样操心
      “谁 release、谁 acquire、跟哪个变量配对”, 只要每个原子操作都是
      seq_cst, 直接按“好像所有操作真的按某个全局顺序依次执行”去推理即可——
      这正是 seq_cst 被定为默认值(不写 memory_order 参数时就是它)的原因:
      心智负担最小、最不容易用错, 代价是每个操作都带一次全内存屏障, 更慢。
```

### 3 Acquire-Release（获取-释放语义）

memory_order_release（释放）： 当前线程中，所有在该操作之前的写操作，都不能被重排到该操作之后。

当该原子写操作完成后，所有之前的修改对其他进行了 acquire 读的线程可见。

memory_order_acquire（获取）： 当前线程中，所有在该操作之后的读/写操作，都不能被重排到该操作之前。

确保能看到同步的那个 release 线程之前的所有写入。

release 限制了它前面的代码不能往下掉。

acquire 限制了它后面的代码不能往上翻。

```c++
// ====================================================================
// 2. memory_order_release / memory_order_acquire: 经典的"消息传递"模式
// ====================================================================
void demo_release_acquire() {
    section("2. memory_order_release + memory_order_acquire —— 建立跨线程 happens-before");

    constexpr int kRounds = 3000;
    int fail = 0;

    for (int i = 0; i < kRounds; ++i) {
        std::atomic<int> data{0};
        std::atomic<bool> ready{false};

        std::thread producer([&] {
            data.store(42, std::memory_order_relaxed);      // ① 先准备好数据
            ready.store(true, std::memory_order_release);   // ② release: 之前的写对 acquire 端一定可见
        });
        std::thread consumer([&] {
            while (!ready.load(std::memory_order_acquire)) {} // ③ acquire: 与②的 release 同步
            if (data.load(std::memory_order_relaxed) != 42) { // ④ 因为②③配对, 这里必然能看到 42
                ++fail;
            }
        });
        producer.join();
        consumer.join();
    }

    std::cout << kRounds << " 轮 release/acquire 消息传递, 观察到的失败次数 = " << fail
              << " (标准保证恒为 0)\n";
    std::cout << "结论: release-store 与 acquire-load 配对后,\n"
                 "      producer 在 release 之前的所有写入, 对 consumer 在 acquire 之后一定可见。\n"
                 "      这是无锁编程里最常用、最重要的同步模式。\n";
}
```

输出

```
========== 2. memory_order_release + memory_order_acquire —— 建立跨线程 happens-before ==========
3000 轮 release/acquire 消息传递, 观察到的失败次数 = 0 (标准保证恒为 0)
结论: release-store 与 acquire-load 配对后,
      producer 在 release 之前的所有写入, 对 consumer 在 acquire 之后一定可见。
      这是无锁编程里最常用、最重要的同步模式。
```

### 4 memory_order_consume（消费语义）

特性： 弱化版的 acquire。它只限制对当前原子变量有数据依赖（Data Dependency）的后续操作不能重排。

现状： 由于过于复杂且编译器难以完美优化，目前标准委员会不建议显式使用它（很多编译器直接将其提升为 acquire）。

```c++
// ====================================================================
// 4. memory_order_consume: 基于"数据依赖链"的同步, 比 acquire 更弱、更省
//    (但实践中几乎所有主流编译器都把它当 acquire 实现, C++ 委员会也在讨论修订/弃用)
// ====================================================================
void demo_consume() {
    section("4. memory_order_consume —— 依赖链同步(弱于 acquire, 现代编译器基本等同 acquire 实现)");

    struct Data { int value; };
    Data payload{123};
    std::atomic<Data*> ptr{nullptr};

    std::thread producer([&] {
        payload.value = 123;
        ptr.store(&payload, std::memory_order_release);  // 发布指针
    });
    std::thread consumer([&] {
        Data* p;
        while ((p = ptr.load(std::memory_order_consume)) == nullptr) {}
        // consume 只承诺: 通过 p 这条“数据依赖链”能访问到的内存(如 p->value)是可见的;
        // 它不像 acquire 那样保证“之后所有的读写都不会被重排到前面”,
        // 只保证依赖于 p 的那些访问。理论上比 acquire 更省一次内存屏障(在弱顺序架构上)。
        std::cout << "consumer 通过 consume 读取到 p->value = " << p->value << "\n";
    });
    producer.join();
    consumer.join();

    std::cout << "注意: GCC / Clang 目前都直接把 memory_order_consume 当作\n"
                 "      memory_order_acquire 来实现, 因为精确追踪“依赖链”极其复杂、容易出错。\n"
                 "      日常代码里几乎不需要手写 consume。\n";
}
```

输出

```
========== 4. memory_order_consume —— 依赖链同步(弱于 acquire, 现代编译器基本等同 acquire 实现) ==========
consumer 通过 consume 读取到 p->value = 123
注意: GCC / Clang 目前都直接把 memory_order_consume 当作
      memory_order_acquire 来实现, 因为精确追踪“依赖链”极其复杂、容易出错。
      日常代码里几乎不需要手写 consume。
```

### 5 memory_order_acq_rel

特性： 同时具备 acquire 和 release 语义。通常用于 Read-Modify-Write（RMW）操作，如 fetch_add。

```c++
// ====================================================================
// 5. memory_order_acq_rel: 既要 acquire 又要 release 的原子 RMW
//    典型场景: 引用计数递减(最后一个线程需要看到所有其它线程之前的写入)
// ====================================================================
void demo_acq_rel() {
    section("5. memory_order_acq_rel —— RMW 操作同时需要 acquire + release 语义(引用计数)");

    constexpr int kThreads = 8;
    std::atomic<int> refcount{kThreads};
    std::vector<int> payload(kThreads, 0);   // 每个线程只写自己的下标, 无数据竞争
    std::atomic<int> last_one_count{0};

    std::vector<std::thread> ts;
    for (int i = 0; i < kThreads; ++i) {
        ts.emplace_back([&, i] {
            payload[i] = i * 100 + 1;        // ① 各自写入自己的数据(普通写, 不是原子操作)
            // fetch_sub 需要同时具备:
            //   release: 让①对"接下来看到归零的那个线程"可见
            //   acquire: 让本线程能看到"之前所有已经减过引用的线程"写的数据
            if (refcount.fetch_sub(1, std::memory_order_acq_rel) == 1) {
                int sum = 0;
                for (int v : payload) sum += v;   // ② 必须能看到全部 8 个线程写的 payload
                std::cout << "最后归零的线程看到 payload 总和 = " << sum
                          << " (期望 " << [&]{ int s=0; for(int k=0;k<kThreads;++k) s+=k*100+1; return s; }()
                          << ")\n";
                last_one_count.fetch_add(1, std::memory_order_relaxed);
            }
        });
    }
    for (auto& t : ts) t.join();

    std::cout << "判定“自己是最后一个”的线程数 = " << last_one_count.load()
              << " (必须恰好是 1, 否则会重复释放/重复销毁)\n";
    std::cout << "结论: 如果这里只用 memory_order_release(没有 acquire),\n"
                 "      最后归零的线程就不保证能看到其它线程写的 payload —— 这正是\n"
                 "      shared_ptr / boost::intrusive_ptr 等引用计数实现选择 acq_rel 的原因。\n";
}
```

输出

```
========== 5. memory_order_acq_rel —— RMW 操作同时需要 acquire + release 语义(引用计数) ==========
最后归零的线程看到 payload 总和 = 2808 (期望 2808)
判定“自己是最后一个”的线程数 = 1 (必须恰好是 1, 否则会重复释放/重复销毁)
结论: 如果这里只用 memory_order_release(没有 acquire),
      最后归零的线程就不保证能看到其它线程写的 payload —— 这正是
      shared_ptr / boost::intrusive_ptr 等引用计数实现选择 acq_rel 的原因。
```

## 安全的单例模式

先来看一个DCLP（Double-Checked Locking Pattern），即双重检查锁定标准写法。

```C++
class Singleton {
public:
    static Singleton* getInstance() {
        if (instance == nullptr) {      // 第一次检查
            std::lock_guard<std::mutex> lock(mtx);

            if (instance == nullptr) {  // 第二次检查
                instance = new Singleton();
            }
        }
        return instance;
    }

private:
    static Singleton* instance;
    static std::mutex mtx;
};
```
第一次检查，避免每次都加锁。
第二次检查，防止多个线程同时创建对象。
上面这个代码的问题在于，`instance = new Singleton();` 这句话不是一个原子操作。
理想顺序是：allocate -> construct -> publish pointer , 但实际可能发生重排序， allocate -> publish pointer -> construct 。
```c++
memory = operator new(...)
Singleton(memory)
instance = memory
// 变成
memory = operator new(...)
instance = memory      // 提前发布
Singleton(memory)      // 还在构造
```
假设两个线程
```c++
# Thread A
instance = new Singleton(); // allocate , publich pointer, 
// 构造还没完成

# Thread B
if(instance != nullptr)
{
    return instance; // 此时对象可能构造函数还没执行完，成员变量还没初始化
}
```
本质的问题，**对象地址已可见 ≠ 对象状态已可见**。
这就是一个典型的内存模型问题，因为这里缺少**happens before**关系。
线程A的写到线程B的读之间如果没有同步机制，则不存在happens-before。行为是未定义。

以下的修复代码，利用原子变量，通过
```
release
    ↓
synchronizes-with
    ↓
acquire
```
来保证
```
构造完成
    happens-before
看到指针
```

```c++
#include <atomic>
#include <mutex>

class Singleton {
private:
    static std::atomic<Singleton*> instance;
    static std::mutex mtx;
  
    Singleton() {} // 构造函数

public:
    static Singleton* getInstance() {
        // 1. 第一次读：使用 acquire 确保看到初始化完成后的内存
        Singleton* tmp = instance.load(std::memory_order_acquire);
        if (tmp == nullptr) {
            std::lock_guard<std::mutex> lock(mtx);
            tmp = instance.load(std::memory_order_relaxed);
            if (tmp == nullptr) {
                tmp = new Singleton();
                // 2. 写入：使用 release 确保 new 内部的构造函数执行完毕后，指针才被发布
                instance.store(tmp, std::memory_order_release);
            }
        }
        return tmp;
    }
};

// 初始化静态成员
std::atomic<Singleton*> Singleton::instance{nullptr};
std::mutex Singleton::mtx;
```

c++ 11之后有更简介的写法

```c++
class Singleton {
public:
    static Singleton& getInstance() {
        static Singleton instance;   // C++11 保证: 初始化线程安全, 且只执行一次
        return instance;
    }
private:
    Singleton() {}
};
```

C++11 标准明确写了这句话（大意）：

> 如果控制流并发地进入一个正在初始化的局部 static 变量声明，并发的执行必须等待该初始化完成。

也就是说，**函数局部 static 变量的"线程安全初始化 + 只初始化一次"是语言标准直接保证的**，不需要你手写任何原子操作或锁——编译器会替你生成。


## 总结

**分清原子性与可见性**：
原子性保证操作“不被打断”，内存模型保证修改“能被别人看见”。

**牢记硬件常识**：
x86 架构是强内存模型（Strong Memory Model），硬件层面上天然保证大部分 Acquire-Release 语义（只存在 Store-Load 乱序）。但 ARM 架构是弱内存模型（Weak Memory Model），乱序非常激进。因此，在 x86 上测试没问题的无锁代码，到了 ARM 上极易因为内存顺序选错而崩掉。

**什么是 Happens-Before 和 Synchronizes-With**
Happens-Before（先行发生）： 描述操作之间的可见性关系。如果 A happens-before B，那么 A 的结果对 B 必然可见。注意，它不代表时间上的先后，而是逻辑上的可见性。
Synchronizes-With（同步）： 描述跨线程的 Happens-Before 关系。一个线程的 release 操作，与另一个线程对同一原子变量的 acquire 操作，构成 Synchronizes-With 关系。

**无锁队列（Lock-free Queue）中，内存顺序怎么选**

控制结构（如链表的 head 或 tail 指针）在发布新节点时，必须用 memory_order_release。
消费者线程在读取 head 或 tail 以获取新节点时，必须用 memory_order_acquire，以确保新节点内部的数据已被完全写入。
如果是单纯的并发计数（如队列长度计），使用 memory_order_relaxed 即可。