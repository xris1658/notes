## 【现代 C++】Lambda 表达式和 `std::function` 的一个不同
日期：2022-01-15

通常，每一个 Lambda 表达式的类型是独一无二的。可以使用 `decltype` 来推导 Lambda 表达式的返回类型。
```cpp
int a = 0;
auto l1 = [&a]() { a += 1; };
auto l2 = [&a]() { a += 2; };
std::cout << std::boolalpha
          << std::is_same_v<decltype(l1), decltype(l2)>
          << std::endl; // 输出 false
```
`std::function` 的类型由参数类型和返回类型决定。
```cpp
std::function<void()> f1(l1);
std::function<void()> f2(l2);
std::cout << std::boolalpha
          << std::is_same_v<decltype(f1), decltype(f2)>
          << std::endl; // 输出 true
```
在实际用途中，这一区别可能会导致序行为上的不同。

我们知道，现代 C++ 中，可以使用 magic static 来实现单例模式。根据这一方式，我们可以让一些用于初始化的函数只执行一次。下面是这一想法的一个简单的实现：
```cpp
// 文件：Initializer.hpp
template<typename FunctorType>
class Initializer
{
private:
    Initializer(const FunctorType& functor)
    {
        functor();
    }
public:
    static void initialize(const FunctoryType& functor)
    {
        static Initializer<FunctorType> ret(functor);
    }
};
```
- 我们将 `Initializer` 类的构造函数设为私有，并提供了一个用于生成 magic static 对象的函数 `initialize()`。用户调用这一函数，以实现单次初始化。
- `Initializer` 依据传入的仿函数（functor）的类型实例化不同的模板类。

接下来我们尝试使用这个类模板：
```cpp
// 文件：main.cpp
#include "Initializer.hpp"

#include <iostream>
#include <thread>
#include <vector>
int main()
{
    int a = 0;
    auto l1 = [&a]() { ++a; };
    auto l2 = [&a]() { ++a; };
    std::vector<std::thread> pool;
    pool.reserve(1024);
    for(int i = 0; i < 1024; ++i)
    {
        if(i % 2)
        {
            pool.emplace_back([&l1]() { Initializer<decltype(l1)>::initialize(l1); });
        }
        else
        {
            pool.emplace_back([&l2]() { Initializer<decltype(l2)>::initialize(l2); });
        }
    }
    for(auto& thread: pool)
    {
        thread.join();
    }
    std::cout << a << std::endl;
    return 0;
}
```
`l1` 和 `l2` 的类型不同。虽然二者的函数体一样，但是这并不影响我们开头提出的结论。由于类型不同，`Initializer` 会被实例化出两种类型：`Initializer<decltype(l1)>` 和 `Initializer<decltype(l2)>`。因此两个实例化中的构造函数都会被调用恰好一次，使得变量 `a` 自增了两次。

将上面的代码稍作修改，情况就大不一样了：
```cpp
// 文件：main.cpp
    //...
    std::function<void()> f1 = [&a]() { ++a; };
    std::function<void()> f2 = [&a]() { ++a; };
    // ...
        if(i % 2)
        {
            pool.emplace_back([&f1]() { Initializer<decltype(f1)>::initialize(f1); });
        }
        else
        {
            pool.emplace_back([&f2]() { Initializer<decltype(f2)>::initialize(f2); });
        }
    }
    // ...
}
```
此时 `l1` 和 `l2` 的类型相同。因此 `Initializer` 只会被实例化出一种类型，即 `Initializer<std::function<void()>>`。这个实例化中的构造函数会被调用恰好一次，因此变量 `a` 实际上只自增了一次。  
>思考：如果将 `f2` 的初始化语句改为
>```cpp
>std::function<void()> f2 = [&a]() { --a; };
>```
>那么输出的内容会变成什么？（提示：请结合操作系统课程学过的知识来得出这一问题的答案。）

细心的读者应该注意到了开头的“通常”二字。接下来就提一下特殊情况。

C++14 开始，Lambda 表达式有了复制构造和移动构造的能力。引入这一点后，我们可以写出这样的代码：
```cpp
int a = 0;
auto l1 = [&a]() { ++a; };
auto l2 = l1; // 调用了复制构造
std::cout << std::is_same_v<decltype(l1), decltype(l2)> << std::endl; // 输出 true
```
这一代码片段中，`l1` 和 `l2` 的类型相同。因此，我们需要对开头的陈述进行一些修改：
- 通常，每一个 Lambda 表达式的类型都是独一无二的。
- 使用复制构造或者移动构造初始化的 Lambda 表达式与构造时传入的实参具有相同的类型。