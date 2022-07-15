## 【现代 C++】`std::make_shared` 遇上私有构造函数
### 问题
类 `Widget` 继承了 `std::enable_shared_from_this`，然而要正常使用，需要保证此类由 `std::shared_ptr` 管理。为了防止意外出现，我们将类的构造函数设为 `private`，然后添加一个“工厂”函数，返回由 `std::shared_ptr` 管理的实例：
```cpp
class Widget: public std::enable_shared_from_this
{
private:
    Widget() {}
public:
    std::shared_ptr<Widget> createInstance()
    {
        // 未完成
    }
};
```
`createInstance()` 如何实现？现代 C++ 提供了两种方式。一种是直接使用 `std::shared_ptr` 的构造函数，另一种是 `std::make_shared`：
```cpp
auto widget1 = std::shared_ptr<Widget>();  // 直接使用构造函数
auto widget2 = std::make_shared<Widget>(); // 用 std::make_shared
```
两种方式各有优缺点。我们关心的是以下两点：
- `std::shared_ptr` 的构造函数需要进行两次内存分配，一次是分配对象的空间，另一次分配控制块的空间；C++ 标准推荐 `std::make_shared` 只进行一次内存分配（将对象和控制块的空间一并分配），并且目前的所有已知实现均只进行一次内存分配。
- `std::shared_ptr<T>` 的构造函数可以调用 `T` 的私有构造函数；`std::make_shared` 不行。

我们看中了 `std::make_shared` 的优点，想要在 `createInstance()` 中使用，结果私有的构造函数成了拦路虎。

### 继承能解决问题吗？
通常，派生类不能访问基类的私有成员，所以不行。

### 老机制解决新问题
C++ 中支持在函数中定义类，称为**局部类（local class）**。
```cpp
int main()
{
    class LocalClass {};
    LocalClass localClass;
}
```
对于成员函数中定义的局部类而言，其访问权和成员函数的访问权一致。类的成员函数的访问权和类本身一致。换言之，如果在基类的成员函数中定义了派生类，这个派生类可以调用基类的私有函数，包括构造函数。

局部类的出现打通了访问权的限制。因此我们可以写出以下代码：
```cpp
class Widget: public std::enable_shared_from_this
{
private:
    Widget() {}
public:
    std::shared_ptr<Widget> createInstance()
    {
        class WidgetDerive: public Widget {}; // 局部类
        return std::make_shared<WidgetDerive>();
    }
};
```
这么一来，`std::make_shared` 访问的构造函数成了派生的局部类的构造函数。派生类是基类成员函数中定义的局部类，因此它可以访问基类的私有构造函数。`std::make_shared` 通过一个“转接头”访问了私有的构造函数，完成了应有的功能。