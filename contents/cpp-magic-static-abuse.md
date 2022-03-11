## 【C++】magic static 虽好，可不要贪杯哦
日期：2022-02-11

Effective C++ 中提到，运用 magic static 可以轻松实现单例模式。但是滥用单例模式也可能会导致问题的出现。
### 问题复现 - 例一
我们假设需要在程序中使用一个外部的类库，它提供了手动加载和卸载的功能，因此写一个 RAII wrapper 是一个非常合适的选择。  
假设手动加载时需要传入一些信息，返回一个句柄（handle），卸载时传入句柄：
```cpp
// 文件：Library.hpp
int init(const InfoType& info); // 传入信息，返回句柄
void exit(int handle); // 传入句柄
// 文件：LibraryRAII.hpp
# include <Library.hpp>
class LibraryRAII
{
public:
    LibraryRAII(): handle_(0) {}
    LibraryRAII(const InfoType& info)
    {
        handle_ = init(info);
    }
    ~LibraryRAII()
    {
        if(hanle_)
        {
            exit(handle_);
        }
    }
private:
    int handle_;
}
```
我们在程序中只需要一个实例，因此使用了 magic static：
```cpp
// 文件：LibraryRAII.cpp
LibraryRAII& instance()
{
    static LibraryRAII ret;
    return ret;
}
```
在主函数里面加载类：
```cpp
// 文件：main.cpp
#include "LibraryRAII.hpp"

int main()
{
    InfoType info;
    // 完善加载类库需要的信息
    instance() = LibraryRAII(info);
    // 使用类库
    return 0;
}
```
看起来好像没什么问题。确实，很多情况下，这么做是没问题的。

我们来深入一些，看看各个对象的生存期。
可以发现，`info` 是主函数中的局部对象，会在主函数结束时销毁；而 `instance()` 中的 `ret` 是一个静态对象，在主函数结束后销毁。
如果我们使用的这个类库在 `init` 时，会保有 `info` 的引用，或者指向 `info` 的指针，而且卸载时需要使用这个地址，那么问题来了：`ret` 销毁时，`info` 已经不存在了，保有的引用变成了悬垂引用，或者保有的指针变成了悬垂指针。因而尝试使用保有的信息会导致未定义行为。
### 问题复现 - 例二
我们来看一个更具体的例子。我们要编写一款音频处理软件，需要使用 ASIO 驱动与音频硬件通信。在 Windows 平台下，它使用了 COM。我们可以找到驱动的 CLSID，进行加载。相关的部分接口如下：
```cpp
// 文件：unknwn.h
class IUnknown // COM 的接口，使用引用计数机制，引用为零时销毁资源，和 std::shared_ptr 相仿
{
    virtual ULONG AddRef() = 0; // 引用数加一，返回加一后的引用数
    virtual ULONG Release() = 0; // 引用数减一，返回减一后的引用数
};
// 文件：<ASIO SDK 目录>/common/iasiodrv.h
class IASIO: public IUnknown
{
public:
    virtual ASIOBool init(void* sysHandle) = 0; // Windows 平台下 sysHandle 是窗口的句柄
}
```
特别注意，`IASIO::init` 中，传入的参数在 Windows 平台是窗口的句柄。
我们需要获取驱动的 ID，然后加载驱动，使用完成后卸载：
```cpp
// 文件：ASIODriverRAII.hpp
#include <common/iasiodrv.h>
class ASIODriverRAII
{
public:
    ASIODriverRAII(): driver_(nullptr)
    ASIODriverRAII(const wchar_t* clsid, HWND hWnd)
    {
        // 通过 Windows COM API 加载驱动，引用数加一，更新了 driver_
        driver_->init(hWnd);
    }
    ~ASIODriverRAII()
    {
        if(driver_)
        {
            driver_->Release();
            driver_ = nullptr;
            // 一些 COM API 的清理工作
        }
    }
    ASIODriverRAII(ASIODriverRAII&& rhs): driver_(nullptr)
    {
        std::swap(driver_, rhs.driver);
    }
private:
    IASIO* driver_;
};
```
同样使用 magic static：
```cpp
// 文件：ASIODriverRAII.hpp
ASIODriverRAII& instance()
{
    static ASIODriverRAII ret;
    return ret;
}
```
接下来是主函数。由于初始化 ASIO 驱动时需要一个主窗口，我们需要在主函数中打开新建一个窗口，获取其句柄，用于加载 ASIO 驱动：
```cpp
// 文件：main.cpp
// 使用了 Qt
#include "ASIODriverRAII.hpp"

#include <QWindow>

int main()
{
    // Qt 的一些初始化工作
    QWindow window;
    auto hWnd = reinterpret_cast<HWND>(window.winId());
    const wchar_t clsid[] = L"{00000000-0000-0000-0000-000000000000}";
    AppASIODriver() = ASIODriverRAII(clsid, hWnd);
    // 进行一些操作
    return 0;
}
```
同样，`window` 的销毁比 `AppASIODriver()` 的早，而 `IASIO` 的析构函数需要使用初始化时传入的窗口句柄，而此时这个句柄已经没用了。因此驱动销毁时便会出现错误。

### 解决方法
如果 magic static 非用不可，那么解决方案也比较简单，那就是为 RAII 类提供移动赋值运算符：
```cpp
// 文件：ASIODriverRAII.hpp
class ASIODriverRAII
{
public:
    ASIODriverRAII& operator=(ASIODriverRAII&& rhs)
    {
        std::swap(driver_, rhs.driver_);
        return *this;
    }
}
```
然后在主函数中使用移动赋值运算符：
```cpp
#include "ASIODriverRAII.hpp"

#include <QWindow>

int main()
{
    // Qt 的一些初始化工作
    QWindow window;
    auto hWnd = reinterpret_cast<HWND>(window.winId());
    const wchar_t clsid[] = L"{00000000-0000-0000-0000-000000000000}";
    AppASIODriver() = ASIODriverRAII(clsid, hWnd);
    // 进行一些操作

    // 移动赋值运算符
    AppASIODriver() = ASIODriverRAII();
    return 0;
}
```
使用移动赋值运算符时发生了什么？很简单。我们将 magic static 对象和一个空的 RAII 交换。交换后 magic static 对象成了空的 RAII，而 `ASIODriverRAII()` 变成了原来 magic static 的 RAII，紧接着进行析构。这样，我们就成功地在主函数结束前卸载了驱动。

有人可能注意到移动赋值运算符实际上执行的是一个交换的动作，于是问：“我不加 `operator=`，加一个专门用来交换的函数，然后继续用 magic static，可以吗？”我的答案是不太合适。

我们观察 C++ 标准库，可以发现交换操作通常以两种形式出现：
```cpp
namespace std
{
template<class T> class shared_ptr
{
public:
    void swap(shared_ptr<T>& rhs);
};

template<class T>
void swap(T& lhs, T& rhs);
}
```
可以发现，交换两个对象的前提是两个对象可写（因此 `swap` 传入的是 non-`const` 左值引用，且作为成员函数的 `swap` 不是 `const`）。
通过这一点，我们很容易得知 `swap` 用在 magic static + RAII 的场景下不合适的原因。
假设我们为 `ASIODriverRAII` 添加了上面的两种交换操作：
```cpp
class ASIODriverRAII()
{
public:
    void swap(ASIODriverRAII& rhs)
    {
        std::swap(driver_, rhs.driver_);
    }
private:
    IASIO* driver_;
};
namespace std
{
template<> void swap(ASIODriverRAII& lhs, ASIODriverRAII& rhs)
{
    lhs.swap(rhs);
}
}
```
之前提到，交换两个对象的前提是两个对象可写，因此我们不能直接写出诸如
```cpp
AppASIODriver().swap(ASIODriverRAII());
```
这样的代码。因为 `ASIODriverRAII()` 是个右值，要么绑到右值引用上，要么绑到 `const` 左值引用上，不能绑到 non-`const` 左值引用上。
如果我们想要像原来的主函数中用交换操作，那么要改成这样：
```cpp
int main()
{
    // ...
    auto blank = ASIODriverRAII();
    AppASIODriver().swap(blank);
    return 0;
}
```
看起来没有问题，但是需要注意，保有之前驱动的 `blank` 不会立即销毁，而是等到主函数结束时才销毁。（如果使用移动赋值运算符，那么换出的 RAII 对象会立即销毁，因为是临时对象。）`blank` 被换掉之后就已经没了用，理应早点销毁，否则仍然有潜在的危险。比如有一天，我们觉得，可以把 `blank` 放在 `window` 的前面：
```cpp
int main()
{
    ASIODriverRAII blank;
    QWindow window;
    auto hWnd = reinterpret_cast<HWND>(window.winId());
    const wchar_t clsid[] = L"{00000000-0000-0000-0000-000000000000}";
    AppASIODriver() = ASIODriverRAII(clsid, hWnd);
    // 进行一些操作

    // 移动赋值运算符
    AppASIODriver().swap(blank);
    return 0;
}
```
那么问题来了：虽然 `AppASIODriver()` 能够成功销毁，但是 `window` 的析构会比 `blank` 的析构早，`blank` 无法正常销毁。典型的按下葫芦浮起瓢。

我觉得这种问题的出现会很烦人，所以还是老实用移动赋值运算符吧。