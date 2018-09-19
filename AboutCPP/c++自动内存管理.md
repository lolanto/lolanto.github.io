# 关于c++自动内存管理

文章主要讨论c++相关的内存池实现技术与思想。

自动内存管理是希望用户在使用时不需要考虑申请的内存的释放问题。

---

以下是对[Building your own memory manager for C/C++ projects](https://www.ibm.com/developerworks/aix/tutorials/au-memorymanager/index.html)的部分翻译

## 为什么要创建自定义的内存管理器

在感谢内存管理器为提高程序效率所作出的贡献之前，我们先来回顾一下c/c++默认的内存管理。标准库中涉及内存管理的函数包括C的`malloc`,`free`,`calloc`以及`realloc`C++引入的`new`,`new[]`,`delete`,以及`delete []`。

类似`malloc`和`new`是通用的内存分配函数。你的代码可能只支持单线程操作，但`malloc`默认能够支持多线程操作。这种额外提供的功能使得这个函数在你的程序中显得效率不高。

`malloc`和`new`在分配内存时会调用操作系统的核心函数，而在释放内存时也会调用操作系统的核心函数。这就意味着，程序每次进行内存操作时，都要从用户态切换到内核态。所以那些频繁调用`new`或者`malloc`等内存操作函数的程序最终都会运行得很慢。

那些在程序中被申请出来的内存区域往往会被忘记释放，而c/c++又没有提供自动的垃圾回收功能。这就使得程序的内存占用不断增加。在大型程序中，该问题会给程序性能带来致命打击，因为能够使用的内存越来越少，同时系统启用的虚拟内存带来了许多磁盘操作，进一步降低效率。

## 内存管理器的设计目标

内存管理器应该满足以下几个设计目标：

* 速度
* 稳定性
* 易用性
* 可移植性

### 速度

内存管理器必须比编译器提供的内存管理器速度快。重复的内存分配和释放不应该影响程序运行效率。假如可以的话，内存管理器应该对程序中典型且大量进行的内存分配行为进行优化。

### 稳定性

内存管理器必须在程序结束前将申请的内存归还给系统。也就是说内存管理器不应该出现内存泄漏的情况。另外，内存管理器应该能够处理错误情况(比如无法分配过大内存空间)并提供相对稳定的错误恢复方法。

### 便利性

内存管理器的嵌入对原有代码的影响应该尽可能少。

### 可移植性

内存管理器应该能够轻松地在各个操作系统间移植，其内部实现不应该使用于平台有关的特性。

## 创建内存管理器的几个有效策略

以下几个策略对创建内存管理器有一定作用

* 请求足够大的内存区域
* 对一般性请求进行优化
* 收集被删除的内存空间

### 请求足够大的内存区域

这是最广为人知的内存管理器策略——在程序启动时申请一片较大的内存区域，尔后再断断续续地补充申请额外的内存区域。各个对象需要申请内存时，均从内存管理器管理的内存区域中分配内存空间。这个方法可以减少对操作系统核函数的调用，提高程序运行效率。

### 优化一般性请求

程序中可能会频繁申请某一固定大小的内存区域，不同的程序大小可能会有所不同。假如内存管理器能够对这种情况进行特殊优化，程序效率会有显著提高。

### 收集被删除的内存空间

被删除的内存空间应该被收集起来。当有其它内存申请时可以优先从这些“废弃”的内区域中分配，使得弃用的内存空间能够被重用。这个方法能够减少程序对内存的使用。

---

## 自动内存管理中存在的不足

从较为抽象，通用的角度考虑，自动内存管理器不应该针对特定的数据类型，它只考虑内存的大小——用户申请了多少内存，就释放多少内存。但是用户定义的对象中假如包含了指针——一个对象可能与另一个对象有关联。那么在清理时，只关注字节就没办法分辨哪些数据是指针(从而同时释放其所指向的内存)，哪些是基本值。

## New和Delete

参考[cppreference-new expression](https://en.cppreference.com/w/cpp/language/new)，[cppreference-operator new](https://en.cppreference.com/w/cpp/memory/new/operator_new)

提及c++内存管理，就必须深入了解c++自由内存分配的New和Delete。

代码中使用new通常是new表达式，其有两种形式：

```c++
::/*(optional)*/ new (placement_params)/*(optional)*/ ( type ) initializer/*(optional)*/
::/*(optional)*/ new (placement_params)/*(optional)*/ type initializer/*(optional)*/
```

作为表达式类似于赋值表达式，其内部会调用Operator=进行实际的赋值操作一样；new表达式根据表达式的写法不同会调用不同的operator new进行操作。

整个new表达式可以拆成三部分分析——*placement_params*，*type*以及*initializer*。这三个部分分别涉及到new的*placement allocation*，*allocation*以及*construction*

### allocation

```c++
void* operator new(size_t count);
```

当placement_params为空时，会调用该运算符，该运算符的目的是：*寻找count大小的内存块，并将其地址返回*。全局以及类中的new操作符都可以被重载。在类中operator new是static函数，故没有this指针可用。

第一个参数count为`sizeof(type)`，c++默认的全局实现是利用malloc完成的，即在堆上寻找足够大的内存区后返回，供构造函数在内存块上初始化对象。可以通过重载该运算符，使其在栈或者BSS，Data段上分配内存。不建议覆盖默认的全局new操作。

### placement allocation

```c++
void* operator new(size_t count, void* ptr); // no-allocating placement allocation
void* operator new(size_t count, user-args); // user placement allocation
```

当提供了placement_param后，该操作符被调用。该运算符的目的是：*当前或许已有合适的内存区域，某些逻辑需要在构造函数调用前执行*。

参数count依然是由系统默认提供的`sizeof(type)`，后面的参数则是placement_param。一旦placement_param与某个operator new的签名对上，就会调用此operator new。

参数为void*的operator new为默认全局运算符，其内部只是返回ptr而已。但编译时会对ptr只想的空间大小进行检查

类可以重载/实现特定签名的operator new，同样该函数为static函数，没有this指针。

### construction

构造函数应该是由系统默认调用的，构造函数仅仅是*初始化operator new返回的内存区域*。假如不提供initializer，则系统调用默认构造函数(无参)，假如initializer与某个自定义构造函数吻合，则调用该构造函数。

与new相比delete表达式相对简单

```c++
::/*optional*/ delete expression
::/*optional*/ delete [] expression
```

根据使用的new表达式不同，被调用的Operator new也不同，导致其调用的operator delete也会不同。

operator delete在析构函数之前被调用，全局的operator delete为

```c++
void operator delete(void* ptr)
void operator delete(void* ptr, size_t count)
```

(1)是简单的new表达式在delete时调用，默认使用free删除new申请的内存空间；在标准库中对空指针没有具体定义。(2)是在用户定义了placement_new并被调用后才在delete时调用的。不建议对全局操作符进行修改。

类特定的operator delete有以下几类

```c++
void T::operator delete(void* ptr)
void T::operator delete(void* ptr, size_t count)
void T::operator delete(void* ptr, user-args...)
```

在delete某个类对象时，优先调用该类定义中是存在的自定义delete，传入的ptr是即将被delete的对象的指针。若(2)定义而(1)没有被定义，则调用(2)，此时count参数相当于sizeof(type)。(无数据成员的类大小为1字节)

(3)在对应签名的operator new(void* ptr, user-args)被调用，而且构造函数抛出异常时被调用。需要在new表达式外进行异常捕获，不需要显式的delete操作。加入类没有进行定义，则寻找全局范围对应签名的，假如都没有，则不会进行任何析构后者内存释放操作。