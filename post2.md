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

​	大端：高位字节放在低地址；小段：高位字节放在高地址

待存储数字0x 12 34 56 78

大端：低地址----->高地址

12 34 56 78

小端：低地址----->高地址

78 56 34 12

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

### Vector的引用失效

​	当某个引用变量引用的是Vector中存储的内容时，需要注意引用失效的问题。一旦引用过程中，Vector由于增加或者删除元素而改变了原来分配的内存区域时，引用就会指向错误的位置。所以，假如需要引用Vector中的某个变量，确保引用过程中Vector的内存区域是不发生变化的。