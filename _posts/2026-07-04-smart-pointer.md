---
title: C++智能指针
date: 2026-07-04 10:00:00 +0800
categories: [技术]
tags: [C++, 智能指针]
pin: true
---
## 智能指针

在C++中，**智能指针**本质上是一个“像指针一样使用的对象”，通过RAII（Resource Acquistion is Initialization，资源获取即初始化）技术自动管理分配资源的生命周期。避免手工new/delete可能带来的资源泄漏问题。RAII是C++98中最重要的技术之一，想法就是每个资源都应该有一个所有者，它由作用域对象表示：构造函数获取资源、析构函数隐式地释放它。
传统的裸指针，在下面的例子中。如果在delete之前发生异常，就会导致p所指向的内存得不到释放，造成内存泄露。

```c++
void func()
{
    int* p = new int(100);
    if (someExpection) {
        throw std::runtime_error("error");
    }
    delete p;
}
```

智能指针，把资源管理封装到对象中。

```c++
void func()
{
    std::unique_ptr<int> p(new int(100));

    throw std::runtime_error("error");
}
```

当函数退出时，自动释放资源。

```c++
~unique_ptr();
```

## auto_ptr

`auto_ptr`是C++98 提供的第一代智能指针，拷贝会转移所有权。在下面的代码中，`p2 = p1`执行后，p1会变成`nullptr`，p2并不是复制p1,相当于`p2 = std::move(p1);`但在 C++98 时代没有移动语义。

```c++
// g++ -std=c++03 demo_auto_ptr.cpp -o demo_auto_ptr
#include <iostream>
#include <memory>

struct Widget {
    int id;
    Widget(int i) : id(i) {}
    void hello() const { std::cout << "Widget " << id << "\n"; }
};

int main() {
    std::cout << "=== 1) Copy transfers ownership unexpectedly ===\n";
    std::auto_ptr<Widget> p1(new Widget(1));
    std::auto_ptr<Widget> p2 = p1; // ownership moved from p1 -> p2

    if (p1.get() == NULL) {
        std::cout << "p1 is now NULL after copy (surprising copy semantics)\n";
    }

    // Dangerous: many developers "expect" p1 still valid after copy.
    // Uncommenting next line can crash (null dereference):
    // p1->hello();

    p2->hello();

    return 0;
}
```

输出:

```
=== 1) Copy transfers ownership unexpectedly ===
p1 is now NULL after copy (surprising copy semantics)
Widget 1
```

如果将`p1->hello();`放开执行，则输出如下:

```
=== 1) Copy transfers ownership unexpectedly ===
p1 is now NULL after copy (surprising copy semantics)
Segmentation fault (core dumped)
```

**auto_ptr 违背了程序员对“拷贝”的直觉。**


p2=p1, 大家普遍认为p1还在，p2只是副本，但实际上p1被清空，p2接管资源。

因引存在以下几个的典型问题。

- 1 容器无法使用
  STL容器要求拷贝不能改变源对象。

```c++
std::vector<std::auto_ptr<int>> vec;
std::sort(vec.begin(), vec.end()); // 排序过程中不断拷贝，元素源对象被置空
```

结果不可预测。因此，auto_ptr与stl容器天然不兼容。

- 2 函数传参容易踩坑
  传参发生拷贝，p对象所有权被转移没了，调用后p为空指针。

```c++
void func(std::auto_ptr<int> p)
{
    ;
}

std::auto_ptr<int> p (new int(10));
func(p); // 所有权从 p 转移到 func 的形参 p
```

很多人会误以为，只是按值传递。实际上是对象没了。

- 3 异常和泛型编程中行为反直觉
  对于普通对象，`process(x)`不会修改x。但若 `T = auto_ptr`, 调用后，x会失效。
  破坏了泛型编程最基本的语义。

```c++
template<typename T>
void process(T obj)
{
}
```

## unique_ptr

为了解决auto_ptr存在的问题，C++11通过引入移动语义`T(T&&)`以及`std::move()`，设计了unique_ptr。
其特点是**禁止拷贝**

```c++
unique_ptr<int> p1 (new int(10));
unique_ptr<int> p2 = p1; // 编译错误

// Call to deleted constructor of 'std::unique_ptr<int>'clang(ovl_deleted_init)
// unique_ptr.h(468, 7): 'unique_ptr' has been explicitly marked deleted here
```

必须**显式移动**,明确我要转移所有权。

```c++
std::unique_ptr<int> p2 = std::move(p1);

p2.reset(); //释放内存。
p1.reset(); //实际上什么都没做。
```

## shared_ptr

既然 unique_ptr 已经能自动释放内存了，为什么还需要 shared_ptr ？
uniqeu_ptr 解决的是“生命周期唯一所有者”的问题，而 shared_ptr 解决的是“多个对象共同拥有一个资源”的问题。
uniqeu_ptr 的局限性是一个资源只能有一个主人。现实场景中可能存在多个对象共同拥有一个资源的场景。
比如Company,Department都需要访问 Employee 这个对象，谁来负责释放Employee？如果使用裸指针，很容易重复释放，提前释放。而`unique_ptr`又只能有一个拥有者。这时候就需要`shared_ptr<Employee>`,共享指针其核心是共享所有权, 通过**引用计数**进行管理。谁最后离开，谁触发释放。

```c++
auto p1 = std::make_shared<int>(100);
auto p2 = p1;
auto p3 = p1; // count = 3

p1.reset(); count = 2
p2.reset(); count = 1
p3.reset(); count = 0 // delete object
```

shared_ptr 的代价是引用计数，每次拷贝需要refCount++，每次释放refCount-- 。
在多线程环境下，还要保证线程安全。因此，shared_ptr 比 unique_ptr 慢。

```c++
atomic++
atomic--
```

此外，shared_ptr还引入了新问题，循环引用（Cycle Reference）。

```c++
#include <iostream>
#include <memory>

struct B; // forward declaration

struct A {
    std::shared_ptr<B> bptr;
    ~A() { std::cout << "A destroyed\n"; }
};

struct B {
    std::shared_ptr<A> aptr;
    ~B() { std::cout << "B destroyed\n"; }
};

int main() {
    {
        std::shared_ptr<A> a = std::make_shared<A>();
        std::shared_ptr<B> b = std::make_shared<B>();

        a->bptr = b; // A owns B
        b->aptr = a; // B owns A  -> cycle formed

        std::cout << "a use_count = " << a.use_count() << "\n"; // usually 2
        std::cout << "b use_count = " << b.use_count() << "\n"; // usually 2
    }

    // 预期如果无循环应该打印 A destroyed / B destroyed
    // 但这里通常不会打印，因为 A 和 B 互相持有，引用计数无法归零
    std::cout << "end\n";
    return 0;
}
```

输出如下, 析构日志不能触发，将造成内存泄漏。

```
a use_count = 2
b use_count = 2
end
```


## weak_ptr

为了打破shared_ptr存在的这种循环，引入了weak_ptr。
weak_ptr的特点是，观察对象，但不拥有对象。那**不增加引用计数**。
将上面shared_ptr的代码稍做修改

```c++
struct B {
    std::weak_ptr<A> aptr;
    ~B() { std::cout << "B destroyed\n"; }
};
```

输出如下，可见析构日志得以顺利触发。

```
a use_count = 1
b use_count = 2
A destroyed
B destroyed
end
```

weak_ptr 的最大缺点是“它不拥有对象，因此无法保证对象仍然存在”，每次使用都要 lock() 检查；同时带来额外的控制块和原子操作开销。
但这些代价换来了对对象生命周期的安全观察能力，避免悬空指针（Dangling Pointer）, 以及解决 shared_ptr 循环引用问题的能力。

```c++
#include <iostream>
#include <memory>

struct Node {
    int value;
    explicit Node(int v) : value(v) {
        std::cout << "Node ctor\n";
    }
    ~Node() {
        std::cout << "Node dtor\n";
    }
};

int main() {
    std::weak_ptr<Node> wp; // 弱引用，不增加引用计数

    {
        std::shared_ptr<Node> sp = std::make_shared<Node>(42);
        wp = sp; // 观察对象

        if (auto locked = wp.lock()) { // 对象还活着
            std::cout << "inside scope, value = " << locked->value << "\n";
        }
    } // sp离开作用域，对象被销毁

    // 这里如果是裸指针，可能悬空；weak_ptr不会
    if (auto locked = wp.lock()) {
        std::cout << "still alive, value = " << locked->value << "\n";
    } else {
        std::cout << "object expired, safe to know it's gone\n";
    }

    return 0;
}
```

输出：

```
Node ctor
inside scope, value = 42
Node dtor
object expired, safe to know it's gone
```

再看一个例子[1]，加深印象，详见注释。

```c++
std::shared_ptr<int> p1(new int(5));
std::weak_ptr<int> wp1 = p1; // 还是只有p1有所有权。

{
  std::shared_ptr<int> p2 = wp1.lock(); // p1和p2都有所有权
  if (p2) // 使用前需要检查
  { 
    // 使用p2
  }
} // p2析构了，现在只有p1有所有权。

p1.reset(); // 内存被释放。

std::shared_ptr<int> p3 = wp1.lock(); // 因为内存已经被释放了，所以得到的是空指针。
if (p3)
{
  // 不会执行到这。
}
```

## 总结

现代 C++ 最佳实践基本就是：
优先 unique_ptr，确实存在多个所有者时使用 shared_ptr，出现观察关系或循环引用风险时使用 weak_ptr。

| unique_ptr       | shared_ptr   | weak_ptr         |
| ---------------- | ------------ | ---------------- |
| 资源唯一拥有者   | GUI组件树    | 父子节点互相引用 |
| 工厂模式返回对象 | 插件系统     | 发布/订阅模式    |
| RAII资源管理     | 任务调度系统 | Observer模式     |
|                  | 对象缓存     | 打破循环引用     |

## 扩展阅读

[1] https://zh.wikipedia.org/wiki/%E6%99%BA%E8%83%BD%E6%8C%87%E9%92%88