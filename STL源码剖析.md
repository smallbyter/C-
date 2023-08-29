# STL源码剖析

## 一、概论

### **STL简介**

**STL版本**

STL是一个标准，只规定了STL的接口，内部实现没有要求。STL有许多实现版本，PJ STL（被Visual C++采用），RW STL等。

**SGI STL版本**注释丰富，结构清晰，可读性最强，同时它也被GCC采用，所以是最流行的版本。

注意，SGI STL并不是原封不动的被用于GCC，所以在GCC中使用STL可能会和SGI STL有一些微小的区别。

**STL组件**

STL分为六大组件：

- 容器(container)：常用数据结构，大致分为两类，序列容器，如vector，list，deque，关联容器，如set，map。在实现上，是类模板(class template)
- 迭代器(iterator)：一套访问容器的接口，行为类似于指针。它为不同算法提供的相对统一的容器访问方式，使得设计算法时无需关注过多关注数据。（“算法”指广义的算法，操作数据的逻辑代码都可认为是算法）
- 算法(algorithm)：提供一套常用的算法，如sort，search，copy，erase … 在实现上，可以认为是一种函数模板(function template)。
- 配置器(allocator)：为容器提供空间配置和释放，对象构造和析构的服务，也是一个class template。
- 仿函数(functor)：作为函数使用的对象，用于泛化算法中的操作。
- 适配器(adapter)：将一种容器修饰为功能不同的另一种容器，如以容器vector为基础，在其上实现stack，stack的行为也是一种容器。这就是一种配接器。除此之外，还有迭代器配接器和仿函数配接器。

大约可以将头文件分三类：

- STL标准头文件（无文件后缀），如vector, deque, list, map, algorithm, functional ...
- SGI STL内部文件（STL的真正实现），如stl_vector.h, stl_deque.h, stl_list.h, stl_map.h, stl_algo.h, stl_function.h ...
- HP版本（所有stl实现的始祖）规范的头文件，如vector.h, deque.h, list.h, map.h, algo.h, function.h

```c++
//引例
#include <iostream>
#include <vector>
#include <functional>
#include <algorithm>
using namespace std;

int main() {
    int ia[6] = {1,27,19,89,100，12};
    vector<int,allocator<int>> vec(ia,ia+6); 
    cout << count_if(vec.begin(),vec.end(),
                     not1(bind2nd(less<int>(),40)));
}
//使用bind2nd将less仿函数的第二个参数绑定为40。即将二元函数变为一元
//negator的缩写，not1是构造一个与谓词结果相反的一元函数对象。not2是构造一个与谓词结果相反的二元函数对象。
```

## 二、空间配置器

**allocator 标准接口**

STL是一个标准，只对接口进行规范，接口背后的实现可以有不同版本，所以目前流行的STL如Visual C++采用的P. J. Plauger 版本，GCC编译器采用的SGI STL版本等都是常见的STL。SGI STL是最流行的版本，我们主要关注SGI STL的allocator实现。

**SGI标准空间配置器， std::allocator**

SGI中定义了一个符合**部分**标准、名为allocator的配置器，它在文件 defalloc.h 中实现。但不使用它。主要效率不佳，只是简单包装了下`::operator new` 和 `::operator delete`。也不建议我们使用，它存在的意义仅在于为用户提供一个兼容老代码的折衷方法

**SGI特殊的空间配置器， std::alloc**

alloc定义了两级的空间配置器

第一级是对malloc/free简单的封装。 C++内存配置基本操作::operator new ,::operator delete相当于C的malloc()和free()，SGI正是使用malloc()和free()完成空间配置的。

而为了解决小型区块可能造成的内存破碎问题（小型区块额外开销多），alloc采用了第二级空间配置器。第二级空间配置器在分配大块内存(大于128bytes)时，会直接调用第一级空间配置器，而分配小于128bytes的内存时，则使用内存池跟自由链表（free_list）进行内存分配/管理。

在分配内存的时候，补足8bytes的倍数，free_list数组中每个指针分别管理分配大小为8、16、24、32…128bytes的内存。

开始所有指针都为0，没有可分配的区块时(就是free_list[i]==0)。会从内存池中分配内存(默认分配20个区块)插入到free_list[i]中。

然后改变free_list[i]的指向，指向下一个区块(free_list_link指向下一个区块，如果没有则为0)。
