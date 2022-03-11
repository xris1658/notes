## 【吐槽】VST2 的一个迷惑设计
日期：2022-02-25

### 如何获取插件类型？
获取一个 VST2 插件的类型有两种方式。
- 如果只关心这个插件是不是合成器：
    ```cpp
    AEffect* effect;
    if(effect->flags & effFlagIsSynth)
    {
        // 是合成器
    }
    else
    {
        // 是音频效果器
    }
    ```
- 如果想知道插件的详细类型：
    ```cpp
    AEffect* effect;
    auto category = effect->dispatcher(effect, effGetPlugCategory, 0, 0, nullptr, 0);
    // 与枚举 VstPlugCategory 中的枚举项比较
    ```
    这是 VST2 中新增的回调。

看起来没什么问题，对吧？接下来看到的代码可能会让你一脸懵。

### 令人迷惑的设计
直接上代码：
```cpp
// 文件：pluginterfaces/vst2.x/aeffectx.h
// 省略部分代码
enum VstPlugCategory;
class AudioEffectX
{
public:
    VstPlugCategory getPlugCategory();
};
// 文件：public.sdk/source/vst2.x/audioeffectx.cpp
// 回调函数的 opcode 为 effGetPlugCategory 时调用此函数
// 省略部分代码
VstPlugCategory AudioEffectX::getPlugCategory ()
{ 
	if (cEffect.flags & effFlagsIsSynth)
		return kPlugCategSynth;
	return kPlugCategUnknown; 
}
```
`getPlugCategory` 就让人感到奇怪。如果插件不是乐器，则不知道插件的具体类型。实际上，Steinberg 在编写这个函数时是希望用户进行重写的：
```cpp
// 文件：vst20spec.pdf
virtual VstPlugCategory getPlugCategory()
{
	if (cEffect.flags & effFlagsIsSynth)
		return kPlugCategSynth;
    
	return kPlugCategUnknown; 
}
// specify a category that fits your plug. See VstPlugCategory enumerator in aeffectx.h.
```
如果用户重写了，那么一切都没问题。
如果用户没有重写，后果是什么呢？顶多是没有指定插件的详细类型罢了，不会影响插件的基本功能。这么一来，很多人可能根本不会注意到这个函数的。

### 影响与危害
假设一个用户要写一个 VST2 插件，打算直接继承 `AudioEffectX` 类。如果用户以前没写过 VST2 插件，那么用户多半只会注意第一种方式。这样一来，如果用户要写的是一个乐器，那么 `AudioEffectX` 会提供一个 `kPlugCategSynth`；如果用户只是写一个简单的增益插件（*估计这是新手写的第一个插件*），那么返回的类型是 `kPlugCategUnknown`。

上面这种情况不只在新手身上出现。假设某人编写了一个 VST1 插件，想要将自己的插件升级到 VST2，这个作者很可能只改了个基类，然后生成了插件：
```cpp
class MyPlugin: public AudioEffect {};  // VST1
class MyPlugin: public AudioEffectX {}; // VST2
```
生成插件的详细类型同样是 `kPlugCategUnknown`。

那么什么时候插件作者会意识到这个问题呢？  
如果这个作者想在一个库文件里面放多个插件（可以参见之前的文章[【VST】诶？有些 VST 插件是好几个插件组成的……](../contents/vst-plugin-container.md)），那么作者会注意到这个函数，意识到自己需要让库文件返回正确的类型。  
如果作者没想这个问题呢？很遗憾，这个作者可能永远都不会意识到这个问题了。

VST2 这种暗地挖坑，还允许用户半桶水晃荡的设计影响了不少插件，既有相当出名，几乎人手必备的付费插件，也有近期一些厂商停止维护并公开的免费插件。这些插件又影响了不少宿主程序：如果宿主程序不给插件作详细分类，那么宿主程序的作者写出的代码可以相当简单，代价就是 VST2 Shell 无法在宿主程序中正常使用。如果宿主程序想给插件作详细分类，想用 `effGetPlugCategory` 获取插件的详细类型，那么有些插件，**包括那些人手必备的插件**，就只能被归入“未知类型”。宿主程序的作者收到了一群用户的抱怨，结果发现是插件的锅，最后往往会决定“去 TMD，我不作详细归类了”。于是不作详细分类的宿主程序越来越多，插件作者更难以意识到详细分类的问题，从而形成了恶性循环。

### VST3 是如何避免这种事情发生的？
 