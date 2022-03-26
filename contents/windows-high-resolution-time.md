## 【Windows 和现代 C++】高精度时间戳

日期：2022-03-17

我们需要在 Windows 平台下，在保证时钟获取的时间单调的条件下进行高精度的时间测量。用什么好？

### 现代 C++ 下的已知选择

现代 C++ 添加了不少时间相关的设施，位于 `<chrono>` 头文件，`std::chrono` 命名空间中。C++20 有几个新时钟，不过我们使用 C++17 标准，因此不考虑那些时钟。剩下的选择只有三个：

- `std::chrono::steady_clock`
- `std::chrono::system_clock`
- `std::chrono::high_resolution_clock`

单从名字来看，`high_resolution_clock` 似乎是最满足我们的需求的时钟，然而这个时钟有一些小问题[^1]：

> The `high_resolution_clock` is not implemented consistently across different standard library implementations, and its use should be avoided. It is often just an alias for `std::chrono::steady_clock` or `std::chrono::system_clock`, but which one it is depends on the library or configuration. When it is a `system_clock`, it is not monotonic (e.g., the time can go backwards). For example, for gcc's libstdc++ it is `system_clock`, for MSVC it is `steady_clock`, and for clang's libc++ it depends on configuration.

没错，`high_resolution_clock` 是由实现定义的。如果我们在 Windows 平台用 MSVC，那么使用的实现是 `steady_clock`，能够保证单调性；用 MinGW 的话就比较惨，因为使用的实现是 `system_clock`，而这一时钟不保证时间单调。

这么一来，保证跨平台（包括编译器）一致的选择就只剩下了 `steady_clock`。合适的时钟是选出来了，但事情还没有完。

### `steady_clock` 在 MSVC 下的实现
`steady_clock` 在 MSVC 下是 Windows API 中的函数 `QueryPerformanceCounter`（以下简称为 QPC）的封装[^2]。值得注意的是，我在微软的文档中注意到了一些额外的内容[^3]：
> 3. Compute all timing on a single thread. Computation of timing on multiple threads — for example, with each thread associated with a specific processor — greatly reduces performance of multi-core systems.
> 4. Set that single thread to remain on a single processor by using the Windows API `SetThreadAffinityMask`. Typically, this is the main game thread. While `QueryPerformanceCounter` and `QueryPerformanceFrequency` typically adjust for multiple processors, bugs in the BIOS or drivers may result in these routines returning different values as the thread moves from one processor to another. So, it's best to keep the thread on a single processor.

我们需要让计时操作只在一个线程上运行，同时需要调整计时线程的相关性，让这一线程只在一个处理器上运行。上面的引文还提到，我们可以使用 `SetThreadAffinityMask` 来进行这个固定操作。我们的代码大致如下：
```cpp
#include <Windows.h>

#include <chrono>
#include <cstdint>

void timerThread()
{
    // 获取这个函数当前位于的线程
    auto thread = GetCurrentThread();
    // 64 位
    std::uint64_t procMask = 0;
    std::uint64_t sysMask = 0;
    std::uint64_t timerMask = 1;
    // 获取当前进程的相关性（即允许在哪些逻辑处理器上运行）
    if (GetProcessAffinityMask(GetCurrentProcess(), &procMask, &sysMask) == 0)
    {
        auto error = GetLastError();
        // 错误处理
    }
    if (procMask == 0)
    {
        procMask = 1;
        while ((timerMask & procMask) == 0)
        {
            timerMask <<= 1;
        }
    }
    while (1)
    {
        auto oldMask = SetThreadAffinityMask(thread, timerMask);
        // now 就是我们得到的时刻
        auto now = std::chrono::steady_clock::now();
        SetThreadAffinityMask(thread, oldMask);
    }
}
```

### 脚注
[^1] [`std::chrono::high_resolution_clock` - cppreference.com](https://en.cppreference.com/w/cpp/chrono/high_resolution_clock)  
[^2] [`steady_clock` struct | Microsoft Docs](https://docs.microsoft.com/en-us/cpp/standard-library/steady-clock-struct?view=msvc-160#remarks)  
[^3] [Game Timing and Multicore Processors - Win32 apps | Microsoft Docs](https://docs.microsoft.com/en-us/windows/win32/dxtecharts/game-timing-and-multicore-processors#recommendations)

### 其他参考
[高精度计时器 `QueryPerformanceCounter` 正确的打开方式（windows环境下）_coffeecato的博客-CSDN博客_queryperformancecounter](https://blog.csdn.net/coffeecato/article/details/44656001)