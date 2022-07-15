## 【C++】magic static 的析构次序
日期：2022-07-15

都说 magic static 好，但是我们是不是忘了点什么东西？
### 从 non-local static 说起
non-local static 的缺点在于：
- 不能用时再初始化
- 不同编译单元间的初始化次序不同

针对第二点，广泛使用的解决方案是 magic static。但是关于对象的析构，好像没人强调。C++ 语言中确定性析构很重要，因此我们会问：  
magic static 对象的析构次序是怎样的？

### 翻标准草案吧
C++17 标准草案中，描述程序终止的 `[basic.start.term]` 部分内容如下：
> If the completion of the constructor or dynamic initialization of an object with static storage duration strongly happens before that of another, the completion of the destructor of the second is sequenced before the initiation of the destructor of the first.

如果某个具有静态存储期的对象的构造或动态初始化的完成严格地比另一个这样的对象早，则第二个对象完成析构比第一个对象开始析构要早。

此处的“具有静态存储期”的对象指的就是任何标有 `static` 的对象。

### 回到 magic static
magic static 对象在控制首次经过声明时初始化。如果对象的初始化是零初始化和常量初始化，那么可以在首次经过声明之前进行。本文不考虑零初始化和常量初始化的情况。

对于多个互不相干的 magic static 而言，事情非常简单明了：
```cpp
class A
{
private:
    A(int) { /*...*/ }
public:
    static A& instance() { static A a(1); return a; }
};


class B
{
private:
    B(int) { /*...*/ }
public:
    static B& instance() { static B b(1); return b; }
};


int main()
{
    auto& a = A::instance();
    auto& b = B::instance();
    return 0;
}
```
上例中 `a` 先构造完，`b` 后构造完，因此先销毁 `b`，后销毁 `a`。

如果 magic static 嵌套构造呢？

```cpp
class Inner
{
private:
    Inner(int) { /*...*/ }
public:
    static Inner& instance() { static Inner inner(1); return inner; }
}

class Outer
{
private:
    Outer(int) { static Inner inner(1); }
public:
    static Outer& instance() { static Outer outer(1); return outer; }
}

int main()
{
    auto& outer = Outer::instance();
    return 0;
}
```
上例中 `inner` 在 `outer` 构造中先构造完，`outer` 再构造完。因此先销毁 `outer`，后销毁 `inner`。

来点递归？
```cpp
#include <iostream>

class Outer
{
private:
    Outer(int);
public:
    static Outer& instance()
    {
        static Outer outer(1);
        return outer;
    }
};

class Inner
{
private:
    Inner(int);
public:
    static Inner& instance()
    {
        static Inner inner(1);
        return inner;
    }
};

Outer::Outer(int)
{
    std::cout << "Outer::Outer starts here\n";
    Inner::instance();
    std::cout << "Outer::Outer ends here\n";
}

Inner::Inner(int)
{
    std::cout << "Inner::Inner starts here\n";
    Outer::instance();
    std::cout << "Inner::Inner ends here\n";
}

int main()
{
    Outer::instance();
    return 0;
}
```


C++ 标准中提到：
> If control enters the declaration concurrently while the variable is being initialized, the concurrent execution shall wait for completion of the initialization.

如果并发进入了 magic static 的声明，则需要等待初始化完成。上例中 `outer` 初始化需要等待 `inner` 初始化完成，`inner` 的初始化需要等待 `outer` 初始化完成；换言之，`outer` 的初始化需要等待自己初始化完成。这一“我等我自己”的情况会导致程序死锁。有点像多个循环引用的 `std::shared_ptr`。

因此，程序输出结果为
```
Outer::Outer starts here
Inner::Inner starts here

```
随后程序在主线程陷入死锁。什么？你问析构次序？我们看析构次序是为了保证程序行为正确。现在程序瘫痪在床了，析构次序也不重要了。