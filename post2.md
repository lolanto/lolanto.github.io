## !Unknown!
### 函数对象

​	class中包含重载operator()的对象。函数对象比起普通函数能够包含自身状态，相比函数指针更灵活，同时在重载函数时使用内联，可以提高运行效率。

### DXGI

​	[DirectX Graphics Infrastructure](https://msdn.microsoft.com/en-us/library/windows/desktop/bb205075(v=vs.85).aspx)。其存在意义是兼容之后的图形API，作为当前图形API与内核态驱动以及硬件沟通(DXGI's purpose is to communicate with the kernel mode driver and the system hardware)。

### D2D

​	D2D基于D3D10.1，要想D2D与D3D11交互，需要生成可同步的DXGI表面，使得表面能够在多个Device之间共享并输出内容。[Surface Sharing Between Windows Graphics APIs](https://msdn.microsoft.com/en-us/library/windows/desktop/ee913554(v=vs.85).aspx)

### Lambda表达式

​	Lambda表达式在编译时会由编译器生成函数对象，根据捕捉器的类型，会自动生成函数对象的构造函数，并将捕捉的变量作为参数传入。[Lambda Functions in c++11 - the Definitive Guide](https://www.cprogramming.com/c++11/c++11-lambda-closures.html)

### 分层架构

​	每一层是某类工作的高度抽象，关注点分离(separation of concerns)是分层架构的特性。为了方便，可以开放某些层，使得非相邻层能够直接访问彼此，但会使得架构耦合性增加。

​	Architecture Sinkhole Anti-pattern该模式中，大量的请求只是简单地穿过各个层级，在穿过的过程中没有丝毫的逻辑处理；假如这样的请求在应用中大量存在(80%以上)，就要考虑是否需要开放某些层，减少请求调用的复杂度。

### 微内核架构

​	核心仅保留通用(抽象)业务逻辑，插件模块实现特定，具体的业务逻辑。该架构可以附加到某一个架构上，从而增加被附加架构在某一需求上的灵活性。架构的关键在于内核与插件的接口设计上，稳定而兼容。

### ANSI

​	0x20以下的字节作为状态码，控制输出终端部分行为。之后到0x80用来编码空格，标点符号，数字，大小写英文字母。

​	0x80至0xFF用于编码**扩展字符集**，包括新的字母，符号，横线竖线等。

### Unicode

​	Unicode对世界范围内的文字进行了“编码”，即每个字符对应一个数字，对应的方式可能是随机的。Unicode不指定数字的字节表示，比如A对应的是U+0061，至于0061是怎么以字节的形式表达，Unicode不负责。所以，Unicode没有限制支持字符的上限，没有限制实现字节的数量。**Unicode仅定义了字符与数字的映射关系**

### UTF 

​	UTF(Unicode Transformation Formats)是Unicode的编码方式，包括UTF-8，UTF-16。不同的UTF规则会将同一个Unicode中规定的数字映射成不同的字节格式。

### 大段小端

​	大端：*低地址*放高位字节；小段：*低地址*放低位字节

待存储数字0x 12 34 56 78

大端：低地址----->高地址

12 34 56 78

小端：低地址----->高地址

78 56 34 12

**应用：** 跨平台/网络编程时使用，确保两台机器接收的信息不会应为字节存放顺序不同而造成错误

**检验：**

```c++
unsigned int d = 0x12345678;
char* t = &d;
// *t == 0x12 大端
// *t == 0x78 小端
```



### UTF-8不考虑字节顺序

​	UTF-8编码至少一个字节，表达某个数字需要的字节安排如下所示

0~127:			0xxxxxxx

128~2047:		110xxxxx	10xxxxxx

2048~65534:		1110xxxx	10xxxxxx	10xxxxxx

65535~2097150:	11110xxx	10xxxxxx	10xxxxxx	10xxxxxx

​	故解析时，只需判断某个字节开头是10证明该字节是构成某个数字的一部分，保留并继续读取；假如读到字节开头是0/110/1110/11110开头，证明是某个数字的开头字节，结合之前保留的数据即可计算实际数值。



### DPI & DIP

​	DPI:**Dot Per Inch**, DIP:**Device Independent Pixel**，在使用Direct2D时使用到的通常是DIP单位，为的是适应不同DPI。目前的显示器，每个Dot代表一个像素，DPI的大小取决于生产工艺。所以假如使用像素单位规定UI元素的尺寸，在不同DPI的显示设备上，UI元素大小不一，DIP用于解决该问题。

​	 **DIP = 1/96th inch**。故DPI = 96时，一英尺由96个像素构成，故1DIP = 1pixel；DPI = 144时，一英尺由144个像素构成，故1DIP = 1.5pixel。[DPI and Device-Independent Pixels](https://msdn.microsoft.com/en-us/library/windows/desktop/ff684173(v=vs.85).aspx)

### 字重

​	 **font weight**描述字体笔画的粗细，分为：thin, light, bold, heavy等，字重越大，字体笔画就越粗。

### 字体衬线

​	 **Serif**带有衬线字体是笔画开始和结尾位置会有额外的修饰，笔画粗细也会与笔画方向有所不同。**Sans-Serif**则是无衬线字体，字体没有经过额外修饰，横竖笔画一般均衡。

​	衬线字体为早期印刷物阅读使用，无衬线字体起到突出文本的作用

### 传递Lambda表达式作为参数

​	使用**std::function**作为参数类型，声明方式为std::function<*return type* (*parameter list*)>

### static_cast与dynamic_cast

​	两者均可以用于父子指针之间的转换。上行转换即子类指针转换成父类指针是完全可以，安全的；但是下行转换，即父类指针变成子类指针就要注意。static_cast是在编译时进行检查，而dynamic_cast是在运行时进行检查。静态转换假如转换错误，即父类转换成其它子类，那么调用一个父类中没有的方法程序就会崩溃。而动态转换会在运行时进行检查，转换失败会返回void。

​	dynamic_cast做下行转换时，父类必须是纯虚类。

### 私有继承

​	私有继承会将父类中所有成员变成子类的私有成员，子类能够访问父类的public以及protected内容。**不能使用父类指针指向子类**，私有继承存在的意义是**复用父类中的部分代码**，父类与子类之间可能并没有什么逻辑关系，私有继承为“实现”服务，而不为“设计”服务。

### 隐式类型转换操作符

```c++
class SomeClass {
    //... some thing about the class
    operator int() {
        //... do something to return an int value
    }
}
```

 	该函数能够定义一个类(此处为SomeClass)对象隐式转换为int的过程，称谓**隐式类型转换**。

### 代理构造函数

```c++
class A{
public:
    A() {
        // Code to do A
    }
    A(/*some parameters*/) {
        // Code to do A
        // other codes
    }
}
```

​	遇到上面这种情况，"Code to do A"的代码在两个构造函数中都被使用，造成冗余。无论是维护还是阅读都有困难。为了避免冗余**C++11**引入代理构造函数(事实上是将原来不被允许的语法赋予新的语义后使用)，使用方式如下：

```c++
A(/*some paramters*/) : A() {
    // other code
}
```

​	注意点：1) 将部分代码代理给其它构造函数的，该构造函数不允许在初始化列表中再初始化其它变量，只可以在函数体中进行赋值；2) 避免循环代理，即A函数代理给B，B由代理给A，编译过程不会出错，直到栈溢出崩溃，使用者应该自己负责规避该错误。

### Using，代替Typedef

​	在**C++11**中以下两个语句是等价的：

```c++
typedef int aliasesName;
using aliasesName = int;
```

相比于原来的typedef，使用using让类型别名的设置带上了与赋值语句类似的语义，更便于理解而已。

### 虚析构函数

​	虚函数能够在动态绑定的时候准确调用我们希望的子类重新实现的父类方法，虚析构函数也是这个道理。动态绑定时，基类析构函数若非虚函数，则仅会调用当前指针类型的析构函数，子类析构函数无法触发导致内存泄漏。所以，即使基类不需要析构函数定义额外逻辑，也要提供虚析构函数。

### RAII

​	Resource Acquisition is Initialization——资源获取即为初始化。C++中的一种资源管理方法。若某种资源需要OpenXXX/CloseXXX配对的函数调用进行维护，为了能够保证资源被正常释放可以采用如下策略：

* 栈是由程序控制，栈上对象的创建与释放是“全自动”的
* 对象入栈——自动调用对象的构造函数；对象出栈——自动调用对象的析构函数
* 换句话说——构造函数保证资源被分配，析构函数保证资源被释放！

```c++
// a resource that has functions like openXXX/closeXXX
class Resource {
    friend class ResourceProxy;
public:
    void Open(){ }
    void Close() { }
// member data and member functions
}

class ResourceProxy {
public:
    Resource& _resource;
    ResourceProxy(Resource& _r) :_resource(_r) { _resource.Open(); }
    ~ResourceProxy() { _resource.Close(); }
}

void main() {
    // Create Resource object and named robj
    ResourceProxy(robj);
    // do things....
    // ResourceProxy's destructor will be called automatically and robj's Close function is invoked as well!
}
```

### 几种特别的模板友元类的示例

```c++
// 1. 任意类型的ptr都可以作为c1的友元类
class c1 {
    template<typename T> friend class ptr;
}
// 2. T类型的ptr作为c1<T>的友元类
template<typename T>
class c1 {
    friend class ptr<T>;
}
// 3. T类作为c1的友元类
template<typename T>
class c1 {
    friend T; // c++11
}
// 4. X类的ptr作为T类的c1的友元类; X != T
template<typename T>
class c1 {
    template<typename X> friend class ptr;
}
```

### typename的另一个用途——嵌套依赖类型

假设有如下代码：

```c++
template<typename T>
class c {
    T::any x;
}
```

在尚未进行编译时，T::any可能是T类的静态变量，也可能是一种依赖T的类型，编译器无法确定，就把它统一当作静态变量处理。

故假如T::any真的是一个类型，就需要在前面加上typename关键字，即：`typename T::any x;`

**特殊情况：**在继承列表以及构造函数的初始化列表中可以省略，即：

```c++
template<typename T>
class c : public base<T>::any {
    c() : base<T>::any() {}
}
```

### auto与decltype

以上两者是**C++11**中引入了两个类型推导的关键字。

**auto:**变量类型推导，变量声明时，根据赋值的表达式计算变量类型。

```c++
int i1 = 1; const int i2 = 2; int i3[10];
auto a1 = i1; // int a1
auto& a2 = i1; // int& a2
auto* a3 = &i1; // int* a3
const auto a4 = i2; // const int a4
auto& a5 = i3; // int a5[10]
auto a6 = i3; // int* a6
```

注意以上在auto关键字以外额外添加的修饰符——&,*,const。为了能够正确计算类型，需要特别注意表达式的结果是**引用**，**常量**以及**数组**。

**decltype:**同样是类型推导，输入表达式后计算表达式返回类型，**表达式本身不会被执行**，只会在编译阶段转换为类型。

```c++
const int i1 = 0; int& i2 = i1; int* i3 = &i2;
decltype(i1) a1 = i1; // const int a1
decltype(i2) a2 = i1; // int& a2
decltype(i3) a3 = &i2; // int* a3
decltype(*i3) a4 = i2; // int& a4
```

和auto不同，表达式返回的const以及引用通通被保留，同时需要注意表达式一旦**解引用**，推断的会是引用。

### typeid() 类型获取操作符

假若有如下代码

```c++
class BaseA {}
class DeriveA : public BaseA {}
class BaseB { public: virtual void func(); }
class DeriveB : public BaseB {}
```

typeid()的参数可以是类名或者表达式。返回类型是标准库类型type_info的引用，该类型不允许拷贝构造函数，仅支持**operator==**,**operator!=**以及**const char* name()**等的函数。

当参数是类名时自然不必多说，当参数是表达式，则需要考虑表达式返回的类型及其继承关系。

假如表达式返回的类型不带有虚函数，则返回的是表达式当前的类型；假如表达式返回的类型带有虚函数，同时返回的表达式是基类的引用(基类指针指向子类，同时解引用)，则返回的是运行时指针指向的子类对象。

**size_t hash_code()**该方法返回输入类型的哈希码，但不保证同一类型的哈希值相同，同一程序调用间返回值相同。

### std::cout与std::cerr

cout与cerr默认情况下均输出到屏幕缓冲上，但cout支持重定向，能够输出至其它输出流(比如文件中)；而cerr则不允许重定向，只输出至控制台窗口。

### const_cast用法

同样是进行类型转换，该转换是将变量的const修饰符去掉，比如：

```c++
void foo(const int& p) {
    // p++ error!
    int& q = const_cast<int&>(p);
    q++; // ok!
}
```

需要注意的是，该操作是用来将**const T&**或者**const T***一类的const去掉而使用的，引用和指针指向的实体应该是一个非const得变量；假如使用该操作修改原本就是const的变量，则会产生未定义的结果。因为声明为常量的变量在编译时存储的位置与堆栈上的数据位置不相同。

### Lua的表遍历

lua中可以使用`ipair`对表进行遍历，C中可以使用`lua_next`函数进行遍历。但！LUA**不保证**遍历得到的顺序与表定义时的顺序一致。

### 数组作为函数参数

当数组以值传递的方式作为函数参数时，数组会退化为指针，当我们使用模板的时候将无法进行正确的模板推断。故可以引用的方式传递数组。

```c++
void func(int arr[]);
void func(int arr[10]);
void func(int* arr);
// 这三者是等价的，前两个arr最终都会变成int* arr
void func(int (&arr)[]);
// 此时arr为引用
```

### 求数组长度

利用模板的自动推断功能在编译器求数组长度

```c++
template<typename T, int N>
inline unsigned ArrSize(const T(&arr)[N]) { return N; }
```

### 初始化列表

```c++
class A {
    public:
    float a;
    float b;
    A() : b(1.0f), a(2.0f) {}
}
```

初始化列表的执行过程由变量的声明顺序决定，上述初始化列表中虽然b在a前，但由于声明时是a在b前，故a会被先初始化，再到b。

### 获得枚举类型的总数量

```c++
enum {
    a = 0,
    b,
    c,
    totalNumberOfEnum // 该变量一直处于最后，读取该变量值即可知道当前枚举的所有选项总量
};
```

### 成员函数修饰符const &, &, &

该修饰符用来限制不同访问类型的对象调用成员函数

1. const& 修饰的函数当且仅当类是const / non-const / lvalue 可访问
2. & 修饰的函数当且仅当类是 non-const 可访问
3. && 修饰的函数当且仅当类是 rvalue 可访问
4. 常见的修饰符 const 表示该函数仅当类是 const 可访问