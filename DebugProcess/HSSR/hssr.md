# 分层屏幕空间光线追踪算法思路及实现详解

核心思想来自论文Hierarchical Multi-Layer Screen-Space Ray Tracing[3]

## 构建G-BUFFER

​	——为之后的光线追踪提供初始数据

​	打开深度测试以及背面提出，记录当前可视范围内的每个像素点的信息，包括摄像机空间下的位置信息，摄像机空间下的法线信息

> 注意: 法线以及位置信息为有符号浮点数，所以对应的缓冲区数据格式需要是RGB32_FLOAT类型
>

**--实现过程--**

```c++
struct VSOutput {
  float4 pos : SV_POSITION;
  float3 csPos : TEXCOORD0;
  float3 csNor : TEXCOORD1;
};

struct PSOutput {
  float4 SSNor : SV_TARGET0;
  float4 SSPos : SV_TARGET1;
};

PSOutput main( VSOutput i ) {
  PSOutput o;
  o.SSNor = float4(i.csNor, 0.0f);
  o.SSPos = float4(i.csPos, 1.0f);

  return o;
}
```




## 记录所有像素

​	——将当前可视范围内的所有物体的每一个面都渲染一遍，尽可能地捕捉每一个可能被反射的细节

​	关闭深度测试，防止遮挡的物体被剔除。关闭背面剔除，背面同样可能被反射。

​	不同的物体在屏幕上的投影可能会占用相同的像素，为了能够保证所有可能的像素都保存下来，需要预先申请足够大的缓冲区，并且考虑缓冲区不足的应对方式。假设当前场景中一个像素位置上平均会有5个片元，则需要申请可以容纳WIDTH * HEIGHT * 5数量的缓冲区。

​	考虑不同物体的绘制时间，投影占用的位置以及不同像素之间的绘制顺序都是不可知的，这里引入Yang[1]的方法——在每个像素位置添加链表，将该像素位置上的所有片元放入到该链表上，之后通过排序还原像素排列。

​	申请与屏幕大小相同的缓冲区，缓冲区中每个像素记录该像素位置上的链表的头指针(ID)信息记为Head Pointer Buffer。

​	申请大缓冲区，缓冲区中每个像素位置上记录链表节点信息，像素位置本身就是节点的ID号。节点信息包括(片源信息+下一个节点ID)，记为Node Buffer。

​	![1523609602308](.\BuildLinkedList.png)

**Figure1: (a)部分为经过当前绘制后Head pointer buffer以及node buffer的情况，可见Head pointer buffer中被绘制位置已经被赋予节点ID，而node buffer中包含了当前被绘制的三个片元的信息，像素位置0,1,2本身就是节点ID；(b)部分黄色三角形片元覆盖之前内容，在node buffer中插入新片元信息，并将next指针指向该位置前一个片元的节点ID，在Head pointer buffer中用新片元的节点ID代替，图片源自[1]**

### --实现过程--

`void BuildLinkedList(float2 uv,  float4 color,  float depth)`

该函数负责创建以及插入节点，

* uv：表示新片元的屏幕坐标(0 ~ width/height)
* color：表示新片元的颜色值(可替换为任意片元信息，或通过增加缓冲区记录更多片元信息)
* depth：表示新片元的深度值(摄像机空间下的位置的z分量，后续通过除以远平面位置得到0~1之间的值)

#### 赋予新片元ID

```c++
int nNewFragmentAddress = RWStructuredCounter.IncrementCounter();
// 假如当前超出容量
if (nNewFragmentAddress >= int(ScreenWidthHeightStorageSlice.z)) return;
```

​	RWStructuredCounter为RWStructuredBuffer\<int\>类型，每个结构化缓存对象中均有隐藏计数器，在App中可以利用函数[ID3D11DeviceContext::OMSetRenderTargetsAndUnorderedAccessViews](https://msdn.microsoft.com/en-us/library/windows/desktop/ff476465(v=vs.85).aspx)设置计数器的初始值。

> 注意：在PS中设置UAV资源只能使用OMSetRenderTargetsAndUnorderedAccessViews函数

​	InCrementCounter函数是原子操作，表示让RWStructuredCounter计数器+1，并将值存储到nNewFragmentAddress中。使用该计数器的值，作为新片元的链表节点的ID。

​	ScreenWidthHeightStorageSlice是外部传入的四维向量，x,y分量表示当前窗口的宽高像素值，z表示整个node Buffer的像素总量，也是当前缓冲区中可以容纳的节点总量(ID上限)。

​	第3行代码意味着当当前节点ID超过了缓冲区容纳上限后，直接结束，不存储新片元。

**ID和像素位置之间有如下转换关系**

```c++
int2 GetAddress(int addr) {
  int2 re;
  re.x = int(ScreenWidthHeightStorageSlice.x);
  re.y = addr / re.x;
  re.x = addr % re.x;
  return re;
}
```

#### 更新head buffer

```c++
  int2 ScreenSpaceAddress = int2(uv);
  uint nOldFragmentAddress = 0;
  InterlockedExchange(tRWFragmentListHead[ScreenSpaceAddress],
    nNewFragmentAddress, nOldFragmentAddress);
```

​	tRWFragmentListHead为RWTexture2D\<uint\>对象，用来记录对应像素位置下的链表头节点的ID(即head buffer)。重复一遍，该对象的大小与屏幕分辨率一致！

​	[InterlockedExchange](https://msdn.microsoft.com/en-us/library/windows/desktop/ff471411(v=vs.85).aspx)是一个原子操作，将新ID放入到head buffer中的当前正在渲染的像素位置(ScreenSpaceAddress)，同时将原来该位置的值，即将被覆盖的节点的ID赋值到nOldFragmentAddress中。

#### 向新节点写入片元信息

```c++
 int2 vAddress = GetAddress(nNewFragmentAddress);
  tRWFragmentColor[vAddress] = color;
  tRWDepthAndLink[vAddress] = uint2(uint(
    (depth / GCameraParam.y) * 4294967295.0f),
    nOldFragmentAddress);
```

​	通过GetAddress函数，计算新节点在node buffer中的存储位置vAddress。tRWFragmentColor以及tRWDepthAndLink均为node buffer。

​	由于一个节点中可能存储许多数据，故可以申请多个相同大小的node buffer，两个缓冲区中相同位置的像素信息记录的是同一个节点的不同数据。

​	tRWFragmentColor为RWTexture2D\<unorm float4\>对象，记录的是当前片元的diffuse颜色值；tRWDepthAndLink为RWTexture2D\<uint2\>对象，即二维uint向量，x分量中的32位记录了当前片元的深度值(进行过float -> uint的格式转换处理)，y分量存储该节点下一个节点的ID，即被覆盖的节点的ID。

​	第3行和第5行代码是进行float->uint的转换，将float值乘以4294967295.0f，相当于保留浮点数的多个位数，再强制转换成uint后存储。GCameraParam.y为远平面位置，depth除以GCameraParam.y将深度值限制在0，1之间，并线性分布

> ​	注意：uint()操作使用的是ftou命令，该命令最后的输出范围是在0 ~ 0xffffffff之间，而depth小于1，故depth与0xffffffff相乘不会超过转换上限



​	经过上述代码，最终输出的应该是head buffer以及两个node buffer。head buffer中记录了每个像素位置上链表的头节点ID，node buffer中记录了每个节点的片元信息(颜色和深度)，以及下一个节点的ID。

> ​	注意：在进行上述过程前，需要对三个缓冲区进行清空，这里是将tRWFragmentColor的信息置为0，将head buffer以及tRWDepthAndLink中的信息置为该缓冲区容纳的像素上限MaxStorage。

​	三个缓存中的空白信息情况如下：

* head buffer中的空白信息意味着场景中没有任何物体投影到该位置上，该位置一直为-1
* tRWFragmentColor中的空白信息意味着当前节点位置尚未被使用，当前场景渲染的像素总量尚未达到node buffer的最大容量。
* tRWDepthAndLink中的空白信息意味着当前节点位置尚未被使用，depth和nextID均为-1
* 每个像素位置上渲染的第一个片元，其nextID永远为MaxStorage，意味着每个像素链表，最终都会索引到ID为MaxStorage的节点上，在ID和像素坐标进行转换时就会被解算为超出当前缓冲区范围的位置。





## 整理凌乱的节点信息

​	——将上述每个像素位置上的链表中的每个片元，按照深度值从前往后进行排序

​	上述过程中生成的链表，其节点顺序与片元的渲染顺序有关，而该顺序完全是随机的，无意义，故需要排序整理。

​	从head buffer中获得一个像素的头节点ID后，从node buffer中可以顺序得到该链表中所有的节点信息，最后一个节点的next ID为MaxStorage。

​	对获得的链表，以节点中的深度信息为依据，进行插入排序，深度值小的片元位于前面，深度值大的片元位于后面(原理摄像机方向，深度值增大)。

​	整理完链表后，将链表信息写入到新的buffer中存储，这里需要创建两个缓冲fragmentDiffuse以及fragmentDepth，前者存储片元的颜色信息，后者存储片元的深度信息。

​	fragmentDiffuse的宽度与屏幕宽度一致，高度则由占据同一像素位置的片元数量决定。此实现假设一个像素位置上最多会有6个片元，故fragmentDiffuse的高度为 屏幕高度 * 6。

​	![1523626023367](.\ArrangeLinkedList.png)
**Figure2: 上图左侧代表排序完成后的，像素位置(x, y)处的链表；右侧为记录排序完成后的信息，假设为fragmentDiffuse缓冲区。**

​	由上图可见，当需要查询某个像素位置上第n层的像素颜色信息时，即可通过读取(x, y + height * n)位置处的颜色信息获得。第0层信息，由每个像素位置上，最靠近摄像机的片元构成，相当于将当前场景打开深度测试，关闭背面剔除渲染的结果。

##### “特殊的”Fragment Depth:

​	Fragment Depth虽然为深度图，但相比普通深度图，多出了一个通道记录深度信息在(向深处)偏移一段距离后的结果。这样做的理由如下：

​	![1523675485601](.\FragmentDepthIllustrate1.png)

**Figure3: (a)部分尚未设置thickness，相当于thickness无穷大;(b)部分设置thickness**

​	反射光线经过屏幕像素位置P(x, y)时，可计算得出当前光线的深度值为D；通过采样深度图的(x, y)位置，可知当前图像中最前方的片元的深度值为d

​	此时若D>d，则判断反射光线与场景中的物体发生碰撞，并以P点的片元信息作为反射信息。

​	从图3(a)中可以知道这种判断是有问题的，光线通过P点时，并没有与物体发生碰撞而是在物体的背面与墙面之间空间中继续移动。但由于深度图只有单一深度值，故无法进行次判断。

​	图3(b)部分，深度图中除了存储深度值以外，还存储了该片元的“厚度”。此时片元的深度是一个范围d ~ d+thickness。当光线通过P点时，只有当D > d 且 D < d+thickness才会判定碰撞生效，否则判定为光线在片元背面穿行。

​	Fragment Depth中存储的便是(depth, depth + thickness)的二元组，以此允许光线的“穿行”。

### --实现过程--

使用compute shader完成

#### 初始化过程

```c++
  uint2 tuv = GID.xy * 10 + GTID.xy;
  uint next = fragmentHead.Load(uint3(tuv, 0));
  float4 colorAndDepth[NUM_CANDIDATE];
  uint2 puv = GetAddress(next);
  uint layer = 0;
```

​	GID, GTID为当前compute shader执行的线程所属的线程(组)ID，由于一个像素开启一个线程，所以可以通过ID计算当前像素的位置(取值0~屏幕分辨率)，即第1行代码的到的结果。

​	fragmentHead是Texture2D\<uint\>对象，记录了屏幕上每个像素位置上的链表的头指针ID。

* tuv: 当前线程处理的像素的位置
* next: 获得当前像素链表的头指针ID
* colorAndDepth: 在这里是一个长度为6的数组，用来存储链表中的每个节点的颜色和深度信息。由于实现中假设每个链表的最大长度为6，故设置6。
* puv: 记录当前节点ID在node buffer中的像素位置
* layer: 由于事先并不知道每个像素上链表的实际长度，所以需要该变量在首次遍历链表的过程中记录

#### 遍历node buffer，获取链表

```c++
  for (;

    (puv.x < ScreenWidthHeightStorageSlice.x &&
    puv.y < ScreenWidthHeightStorageSlice.y);

    puv = GetAddress(next),
    ++layer
  ) {
    colorAndDepth[layer].xyz = fragmentColor.Load(uint3(puv, 0)).xyz;
    uint2 link = fragmentLink.Load(uint3(puv, 0));
    next = link.y;
    
    const float f = 1 / float(uint(0xffffffff));
    colorAndDepth[layer].w = float(link.x) * f;
  }
```

从循环体开始：

​	fragmentColor是输入的Texture2D\<unorm float4\>对象，是之前node buffer中负责记录每个节点的diffuse颜色，fragmentLink则是输入的Texture2D\<uint2\>对象，是之前node buffer中负责记录每个节点深度以及下一节点ID的信息。

​	第9，10行代码，从当前节点位置处(通过当前节点ID换算得到像素位置存储于puv中)读取颜色信息以及深度和下一节点ID信息。

​	next指向刚读出来的下一节点的ID

​	第13，14行代码，还原深度值。在上一部中，深度值从float->uint是先将float * 0xffffffff，再截断成uint；这里将过程逆向进行，不过为了能够保留精度，需要先将uint->float再进行除法(uint除法会截断0后小数)。

迭代更新：

​	一次循环后，增加layer值代表链表中节点数量加1；根据循环体中获得的下一节点的ID，解算出下一节点在node buffer中的像素位置，存于puv中

​	假如新的puv坐标任意一个分量大于或等于node buffer的尺寸，则意味着该puv是无效的，跳出循环。假如下一个节点并不存在，则之前的到的下一节点的ID应该等于MaxStorage，即用来清空head buffer以及node buffer的值，该值恰好等于node buffer的最大容量，则对该值求屏幕坐标，得到的y分量应该等于node buffer的高度。故一旦下一节点不存在，则循环会跳出。

> 注意：尚不清楚该转换对深度精度的影响程度	

#### 对链表节点根据深度值从小到大进行插入排序

```c++
  for (uint curLayer = 0; curLayer < layer; ++curLayer) {
    float minDepth = colorAndDepth[curLayer].w;
    uint minLayer = curLayer;
    for (uint nextLayer = curLayer + 1; nextLayer < layer; ++nextLayer) {
      float curDepth = colorAndDepth[nextLayer].w;
      if (curDepth < minDepth) {
        minDepth = curDepth;
        minLayer = nextLayer;
      }
    }
    float4 temp = colorAndDepth[curLayer];
    colorAndDepth[curLayer] = colorAndDepth[minLayer];
    colorAndDepth[minLayer] = temp;
  }
```

一个显而易见的插入排序，这里不做赘述

#### 将排序结果写入到输出中

```c++
  for(uint i = 0; i < layer && i < NUM_LAYER; 
    ++i, tuv.y += int(ScreenWidthHeightStorageSlice.w)) {
    fragmentDiffuse[tuv] = float4(colorAndDepth[i].xyz, 1.0f);
    fragmentDepth[tuv] = float2(
      colorAndDepth[i].w,
      colorAndDepth[i].w + epsilon(colorAndDepth[i].w, 0));
  }
```

从迭代更新开始：

​	i代表当前的层级，也是当前写入的节点下标，写完一个节点就对i加1

​	tuv.y += int(ScreenWidthHeightStorageSlice.w)意味着，每往后写入一个节点，输出位置就往下偏移(回想Figure2有助理解)，ScreenWidthHeightStorageSlice.w为屏幕高度的像素值。

循环体：

​	第3行代码将节点颜色信息写入到fragmentDiffuse中

​	第4行代码，将节点深度信息写入到fragmentDepth中；其中epsilon函数计算当前深度值的偏移大小thickness

##### epsilon

```c++
// 计算d_min的深度偏移值, level是当前的mipmap level层级
float epsilon(float d_min, uint level) {
  const float inv_far = 1.0f / GCameraParam.y;
  return epsilon_factor * inv_far * w(d_min, level);
}

float w(float depth, uint level) {
  const float base_radius = GCameraParam2.x / GCameraParam.z;
  const float add_radius = (GCameraParam2.z - GCameraParam2.x) / GCameraParam.z;
  return (base_radius + depth * add_radius) * float(1 << level);
}
```

​	epsilon_factor是epsilon的缩放因子，为常量(此处设置为25)；GCameraParam.yz分量分别记录了当前视锥体的远平面位置和当前窗口分辨率的宽度；GCameraParam2.xz两个分量分别记录了近平面的宽度(摄像机空间中的宽度)和远平面的宽度(摄像机空间中的宽度)

​	如上所示，真正决定偏移值大小的应该是函数w；函数w计算结果满足以下几个条件：

* 深度值越大(片元离屏幕越远)，偏移值越大

> 注意：暂时不清楚为何需要满足以上条件

​	GCameraParam2.x / GCameraParam.z为近平面位置，一个像素点在摄像机空间下占据的x方向的大小；GCameraParam2.z / GCaermaParam.z为远平面位置，一个像素点在摄像机空间下占据的x方向的大小；

​	depth取值范围在(0~1)，故相当于根据depth大小在远平面和近平面之间进行插值。最后的1 << level其实是对不同miplevel下图像的宽度除以$2^{MipLevel}$。

​	

​	经过上述过程，最终应该输出两个缓冲区的结果fragmentDiffuse以及fragmentDepth:

* fragmentDiffuse顺序记录了一个像素位置上，深度从小到大的片元diffuse颜色值；由于事先对fragmentDiffuse进行了清空，所以没有着色的地方，颜色值为(0, 0, 0, 0)
* fragmentDepth顺序记录了一个像素位置上，深度值从小到大的片元；同时每个片元除了自身深度值以外，还有深度值加上偏移后的值。同样，没有着色的地方，深度值为(0, 0)

## 手工生成深度图的"mipmap"

​	在屏幕空间进行光线追踪，实质是对光线经过的像素进行遍历，找到发生碰撞的像素点。光线往往会覆盖100+个像素点，故逐一像素遍历没有意义。

​	通过引入步长stride的概念，每经过stride个像素对比一遍以此加速。

​	对于小细节内容可能只占用少量像素(比如10个)，大stride会跳过这些小细节，获得错误结果；太小的stride对性能提升不大，无意义。

​	在光线追踪的过程中，唯一的参考数据即是深度值；通过生成深度图的mipmap，尔后在更高的mipmaplevel上遍历每个像素，此时的高level的深度图一个像素相当于0级深度图的多个像素；

​	计算一定范围内的深度值的综合属性，并付给给高层级的深度图，当光线在高层的深度图中检测到碰撞，便下降深度图的层级，进行更精细的碰撞检测，直到0级的深度图碰撞检测成功，才返回最终的结果。

​	以上遍历过程中，stride大小由当前正在遍历的深度图的mipmap level决定，层级越高stride越大。

![1523690443165](.\DepthMipMapIllustrate.png)

**Figure4: 在多个mipmap level中穿梭，进行光线追踪的示意图。黄色条带包裹的灰色条带的数量，代表了不同层级的一个像素代表0级图像中的像素的数量，条带的高度表示当前像素上的深度值大小；黑色线条为当前正在处理的光线；由图可见，追踪过程从0级开始，一直往上提高，直到某个层级发生碰撞后，层级开始下降，最终在0级处检测碰撞成功并返回；注意：该图只作示意，数据与当前算法存在差异，图片源自[2]**

​	在这个步骤中，需要生成不同层级的深度图。

​	在上一步中生成的fragmentDepth可以作为层级0的深度图，意味着每个层级的深度图的每个像素均包含两个信息(当前像素深度，像素深度+偏移值)。高级的深度图的长宽是前一级深度图的长宽的一半。

​	生成过程如下：

1. 当前需要生成层级L，位置(x,y)的深度图信息。分别取上级(L-1)深度图在位置(2x, 2y), (2x + 1, 2y), (2x, 2y + 1)以及(2x + 1, 2y + 1)的像素值，记为f1, f2, f3和f4
2. f1, f2, f3以及f4像素位置上，均会有多个片元；初始情况下，f1, f2, f3, f4包含的是这四个像素位置上，位置最靠前的四个片元深度值；
3. 从四个深度值上，取最小值d_min，同时d_max = d_min + t；将四个像素位置上，所有片元深度满足d >= d_min && d <= d_max的片元，都归纳为一个片元，同时更新f1, f2, f3, f4，指向各自所属像素位置上的下一个片元
4. 循环步骤3，直到四个像素位置均没有片元方停止

![1523691571894](.\ConstructDepthMipmapIllustrate.png)

**Figure5: 上图展示的即为该过程前两部的结果；像素沿垂直方向排布，从左往右深度增加；同一行的多个矩形区域代表了同一像素位置上，不同深度的片元；左边假设为0级，中间为1级，右边为2级，图片源自[3]**

### --实现过程--

使用compute shader完成

#### 初始化过程

```c++
uint2 tuv = GID.xy * 10 + GTID.xy;
uint2 base[4];
float2 frags[4];
base[0] = tuv * 2;
base[1] = tuv * 2 + uint2(1, 0);
base[2] = tuv * 2 + uint2(0, 1);
base[3] = tuv * 2 + uint2(1, 1);

frags[0] = fragmentDepthLastLevel.Load(uint3(base[0], 0));
frags[1] = fragmentDepthLastLevel.Load(uint3(base[1], 0));
frags[2] = fragmentDepthLastLevel.Load(uint3(base[2], 0));
frags[3] = fragmentDepthLastLevel.Load(uint3(base[3], 0));
```

​	fragmentDepthLastLevel是Texture2D\<float2\>对象，存储上一层深度图的信息

* tuv: 代表当前正在计算的深度图像素位置
* base: 四元素数组，代表即将要合并的上一级深度图的4个相邻片元位置(相当于上文f1, f2, f3, f4的位置)，在初始化过程中，指向4个像素位置上最靠前的4个片元位置
* frags: 上一级深度图的四个深度值对，在初始化过程中，存储的是4个像素位置上最靠前的4个片元的深度值对

#### 检索合并

```c++
  float d_min = min(min(frags[0].x, frags[1].x), min(frags[2].x, frags[3].x));
  if (frags[0].x <= 0 && frags[1].x <= 0 && frags[2].x <= 0 && frags[3].x <= 0) return;
  for(uint cur_lay = 0; cur_lay < NUM_LAYER; ++cur_lay) {
    
    float d_max = d_min + epsilon(d_min, uint(LevelNumSizeLastSlice.x));
    float cur_max = 0;
    for(uint i = 0; i < 4; ++i) {
      while(frags[i].x > 0) {
        // 纳入当前层
        if (frags[i].y <= d_max) {
          cur_max = frags[i].y > cur_max ? frags[i].y : cur_max;
          base[i].y += LevelNumSizeLastSlice.w;
          frags[i] = fragmentDepthLastLevel.Load(uint3(base[i], 0));
        } else{
          break;
        }
      }
    }

    fragmentDepthNextLevel[tuv] = float2(d_min, cur_max);
    d_min = min(min(frags[0].x, frags[1].x), min(frags[2].x, frags[3].x));
    if (frags[0].x <= 0 && frags[1].x <= 0 && frags[2].x <= 0 && frags[3].x <= 0) break;
    tuv.y += LevelNumSizeLastSlice.z;
  }
```

​	NUM_LAYER表示当前一个像素位置上可容纳的片元数量，LevelNumSizeLastSlice.xzw分别表示当前正在生成的深度图层级，当前层级一个layer的高度(一个layer的偏移量，用于索引同一像素位置上下一个片元的存储位置)，上一级深度图一个layer的高度；

​	fragmentDepthNextLevel为RWTexture2D\<float2\>对象，存储当前正在生成的深度图信息

​	第1行和第21行代码，表示从当前指向的四个表层片元中获得最小深度值d_min，也是代表即将写入的layer的最小深度值

​	假如当前指向的4个片元深度值均为0，表示当前指向的4个位置已经超过链表结尾，则之后再无可合并的片元，故直接返回；之前清空深度图的时候将所有数据置为0，而被写入的片元中不可能出现深度值为0的情况，故以0代表结束

​	d_max = d_min + t，计算偏移值的函数epsilon和之前的函数一致

​	d_min~d_max范围内的片元，其最大深度可能并没有d_max大，故使用cur_max记录当前合并的所有片元中的最大深度值

​	遍历4个像素的位置，顺序地遍历链表中的每个节点，凡是满足当前合并条件的片元，指向该片元的base便指向下一节点，frags获取下一个节点的信息；若当前片元不满足合并条件，或者当前片元深度值为0(当前所指片元不存在)则遍历下一个像素位置；

​	当4个像素遍历完成后，将d_min和cur_max写入到fragmentDepthNextLevel中

​	判断当前剩下的节点是否存在，若不存在则结束深度图的生成；若存在，则调整当前layer的指针(tuv.y += LevelNumSizeLastSlice.z)，准备下一个layer的生成。



​	上述步骤生成了比输入深度图fragmentDepthLastLevel更高一层(更粗糙)的深度图并存储到fragmentDepthNextLevel中。

* 在开始生成前，将fragmentDepthNextLevel清空，所有记录置为0；故生成结束后，无片元/节点的位置上的像素值为0
* 上述步骤可循环执行，生成多级深度图

> 注意：在循环生成的过程中，需要更新LevelNumSizeLastSlice中的值

## 追踪开始！

​	第一步中，可以获得每个像素上的法线信息以及位置信息，故可生成反射光线R，包括其起点以及反射方向

​	首先需要将R投影在屏幕上，即将其光栅化，从而获得其所覆盖的像素信息，投影过程如下：

1. 计算R起点位置csOrig，通过投影矩阵，获得R起点位置在屏幕上的像素位置P0，同时可获得起点位置的深度信息K0

2. 计算csOrig + R的方向csDir，将该点通过投影矩阵，获得其在屏幕上的像素位置P

3. 利用P - P0 = d，可以计算R在屏幕上沿光线方向移动的方向

4. 通过P0 + d * t获得投影光线与屏幕边界的交点位置P‘

5. 由于屏幕边界上的点均为视锥体四个侧面上的点，故P’代表着一条从摄像机空间原点沿视锥体侧面的一条射线；故可将P‘重新投影回摄像机空间，并获得视锥体边界上的一条射线L

6. 求出L和R的交点，即可获得光线R与视锥体的交点位置，将该点利用投影矩阵投影到屏幕中，可获得对应的像素位置P1以及深度信息K1

7. 利用P1-P0 = dP，K1 - K0 = dK，可以通过P0 + dP * t; K0 + dK * t的方式获得该光线任意像素位置上的深度值

   t相当于光线前进的步幅，此时结合上一步生成的多个mip level的深度图，便可以在多层深度图中穿梭，进行光线追踪

### --实现过程--

在compute shader中完成

#### 初始化过程

```c++
  uint2 tuv = GID.xy * 10 + GTID.xy;
  // 起点法线,位置以及反射方向
  float3 csNor = normalize(SSNor.Load(uint3(tuv, 0)).xyz);
  float3 csPos = SSPos.Load(uint3(tuv, 0)).xyz;
  float3 csRef = normalize(reflect(
    normalize(csPos), csNor));
  Ray refRay;
  refRay.orig = csPos;
  refRay.dir = csRef;

  RayTracingState state;
  float acceleration_delay = 1;
```

​	SSNor为Texture2D\<float4\>对象，是第一步生成的，记录每个像素位置的法线信息；SSPos为Texture2D\<float4\>对象，同样是第一步生成的记录每个像素的位置信息；

​	第3~4行代码结合法线与位置信息，计算反射方向

​	Ray以及RayTracingState为光线追踪中用到的数据结构，用于记录追踪的状态信息

​	acceleration_delay是之后循环中用到的一个变量，下面会讨论

```c++
struct Ray {
  float3 orig;
  float3 dir;
};

struct RayTracingState {
  float2 P0, dP;
  float k0, dk;
  // 当前插值步长
  float t;
  uint curLevel;
  uint layer;
  float2 pixlen;
  float2 posDir;
  bool done;
};
```

​	Ray用于记录反射光线的状态，包括起点以及反射方向，均处在摄像机空间中

​	RayTracingState用于记录光线追踪的状态:

* P0，dP分别记录反射光线起点在屏幕上的像素位置，dP记录反射光线到达边缘位置时的点(终点)与起点之间的差距
* k0, dk分别记录起点位置的深度值(摄像机空间下)的倒数，以及终点位置的深度值的倒数；只有深度值的倒数可支持屏幕空间的线性插值
* t表示当前的插值情况，0表示当前光线前端处在起点，1表示到达终点
* curLevel表示当前光线正在穿越的深度图的层级——0表示最精细等级，越往上越粗糙
* layer表示与当前光线碰撞的节点，其在链表中的位置
* pixlen = 1 / dP 用于后续计算
* posDir = step(float2(0, 0), dP) 用于后续计算
* done: 当前光线追踪是否完成

#### 初始化追踪状态

`void setupState(Ray ray,inout RayTracingState state);`

​	该函数接收反射光线ray,以及当前正在渲染的像素位置(相当于反射光线起点位置投射到屏幕上的像素位置)

​	返回初始化后的光线状态state

```c++
  float4 H0 = mul(GProjectMatrix, float4(ray.orig, 1.0f));
  state.k0 = 1.0f / H0.w;
  state.P0 = ((H0.xy * state.k0) + 1.0f) / 2.0f;
  state.P0.y = 1.0f - state.P0.y;
  state.P0 *= GCameraParam.zw;
```

​	根据输入的光线，以及像素位置，可以直接设置k0和P0

> 注意：不能将P0直接设置成当前渲染的像素的位置，因为之后需要对插值起点t进行抖动，此处小的偏差可以帮助之后t的起点取值位置，防止t过小而导致光线起始深度值和深度图一致而终止追踪过程

```c++
  float2 P1;
  float4 Hclip = mul(GProjectMatrix, float4(ray.orig + ray.dir, 1.0f));
  float2 Pclip = ((Hclip.xy / Hclip.w + 1.0f) / 2.0f);
  Pclip.y = 1.0f - Pclip.y;
  Pclip = Pclip * GCameraParam.zw;
  Pclip += (distanceSquared(Pclip, state.P0) < 0.000001f) ? float2(0.001f, 0.001f) : float2(0, 0);
  float2 dir_screen = normalize(Pclip - state.P0);
  float2 dest_screen = step(0, dir_screen) * (GCameraParam.zw - uint2(1, 1));
  float2 dist_screen = (dest_screen - state.P0) / dir_screen;
  // P1为光线投影在屏幕上的边界位置
  P1 = state.P0 + dir_screen * min(dist_screen.x, dist_screen.y);
```

​	GProjectMatrix为投影矩阵，GCameraParam.zw分量分别记录了当前窗口的长宽大小

​	ray.orig+ray.dir为光线在摄像机空间中沿反射方向移动一个单位的位置，利用投影矩阵将该点投影到裁剪空间中，为之后求屏幕空间相关信息做准备。

> 注意：假如ray.dir = +-normalize(ray.orig)，则会导致投影结果和P0重合，导致错误的结果

​	第3~5行代码，利用裁剪空间的坐标，求出投影后的点在屏幕的像素位置

​	第6，7行代码，求出光线在屏幕上的移动方向，其中第6行代码是为了防止后面做除法的时候除以0引起错误(就是投影重合导致的错误)，distanceSquared是两个点之间的距离的平方

​	第8~11行代码在求光线与屏幕空间的边界的交点位置：

* 第8行代码实际上是求x,y分量沿着各自的方向最后会抵达的边界。比如若dir_screen.x > 0，则反射光线一定有朝右走的趋势，故反射光线一定会有点(WIDTH, y)，同理，若dir_screen.y > 0则意味着反射光线一定会有点(x, HEIGHT)
* 第9行代码实际上是求x,y分量要分别达到边界需要沿反射方向走多远。比如使用上面的假设，反射光线总会有交点(WIDTH, y)以及(x, HEIGHT)，则分别求出到达两者需要走的距离为dist_screen.xy
* 第11行代码，从dist_screen中求出最小值，意味着反射光线最先到达的边界点，该点也是屏幕的边界

![1523703256673](.\CalcScreenBorder1.png)

**Figure6**

```c++
  float3 csFar = reproject(P1);
  float3 csFarDir = normalize(csFar);
  float3 n = cross(csFarDir, cross(ray.dir, csFarDir));
  float dirN = dot(ray.dir, n);
  dirN += dirN == 0 ? 0.0001f : 0;
  float len = dot(csFar - ray.orig, n) / dirN;
  float3 csEnd = ray.orig + ray.dir * len;
```

​	将边界点重新投影会摄像机空间中，由于只有屏幕的像素位置，该点其实相当于代表摄像机空间中的一条线，而我们下面需要的也是这条线，故可随意设置重投影时候的深度值(假设为1)

`float3 reproject(float2 sspos, float depth = 1.0f);`

该函数接受一个屏幕空间中的像素位置，以及指定的深度值(z坐标)，返回摄像机空间中的点位置

```c++
  // 构建NDC中的坐标 --> 裁剪空间坐标
  sspos /= GCameraParam.zw;
  sspos.y = 1.0f - sspos.y;
  sspos = (sspos * 2.0f - 1.0f) * depth;

  float4 rePos = float4(sspos, 
    (GCameraParam.y * (GCameraParam.x - depth)) / (GCameraParam.x - GCameraParam.y)
    , depth);

  rePos = mul(GProjectMatrixInv, rePos);
  return rePos.xyz;
```

​	GCameraParam.xyzw分别表示摄像机的远近平面的位置，窗口的宽高像素值；GProjectMatrixInv表示投影矩阵的逆矩阵

​	由于投影矩阵的结果是裁剪空间中的坐标，故需要将屏幕空间中的像素坐标转换为裁剪空间中的坐标。

​	求出边界点的位置csFar后，其实就可以构造一条从摄像机空间原点(视点)出发的射线，该射线与反射光线的交点就是我们要求的光线与屏幕空间的交点在摄像机空间中的位置。

​	看第3，4行代码，实际上是求空间中两直线的交点

​	在获得交点之后需要验证该交点的有效性：在视锥体内

```c++
  if (csEnd.z > GCameraParam.y || len < 0)
    csEnd = ray.orig + ray.dir * ((GCameraParam.y - ray.orig.z) / ray.dir.z);
  if (csEnd.z < GCameraParam.x)
    csEnd = ray.orig + ray.dir * ((GCameraParam.x - ray.orig.z) / ray.dir.z);
  float4 H1 = mul(GProjectMatrix, float4(csEnd, 1.0f));
  float k1 = 1.0f / H1.w;
  P1 = ((H1 * k1 + 1.0f) / 2.0f);
  P1.y = 1.0f - P1.y;
  // 光线末端在屏幕上最终的投影位置
  P1 = P1 * GCameraParam.zw;
```

​	csEnd.z > GCameraParam.y意味着交点在远平面之外，此时需要将交点截取在远平面上；len < 0意味着交点位于反射光线的反方向，此时也应该将交点截取在远平面上；csEnd.z < GCameraParam.x意味着交点在近平面之外，此时需要将交点截取在近平面上

​	在修正完交点位置后，利用投影矩阵将交点投影到裁剪空间，进行透视除法求得交点在屏幕空间上的像素位置

​	初始化剩余的state参数

```c++
  // 初始化变量
  state.dP = P1 - state.P0;
  state.dk = k1 - state.k0;
  // xy方向上像素百分比
  state.pixlen = 1.0f / abs(state.dP);
  state.curLevel = 0;
  state.posDir = step(float2(0, 0), state.dP);
  // 初始化抖动
  float2 dt = abs(floor(state.P0 + state.posDir) - state.P0) * state.pixlen;
  state.t = min(dt.x, dt.y) + EPSILON;
  state.done = false;
  state.layer = 0;
```

​	state.pixlen表示x, y方向上移动一个像素，需要插值变量偏移多少；比如dP.x = 3，pixlen.x = 0.33意味着插值变量t每增加0.33，x方向上至少移动一个像素的距离；

> 注意：尚不清楚dt抖动程度的选择标准，此处感觉是利用dt让初始位置尽量靠近下一个采样像素位置

#### 开始步进

```c++
  // star ray marching
  for (int i = 0; i < MAX_STEPS && state.t < 1.0f; ++i) {
    // init
    float div = int(1 << state.curLevel);
    float2 Pcurr = getCurrSSPos(state);
    // 当前level下应该采样的像素位置
    uint2 levelUV = uint2(Pcurr / div);
```

​	步进循环总次数有限，所以选择合理的步幅有助于更快找到正确的碰撞位置；同时插值t不能超过1，1即为光线在屏幕中的末尾端点

​	div是用在需要计算高层级深度时候，通过除0层级的深度图换算当前层级的长宽用的

​	getCurrSSPos获得光线当前在屏幕中的像素位置，由于层级间长宽减半，所以通过除以div可以求出当前光线在高层级的深度中的像素位置

```c++
float2 getCurrSSPos(RayTracingState state) {
  return state.P0 + state.t * state.dP;
}
```

​	该函数很简单，就是通过现在的插值情况直接插值出当前位置

​	在获得当前光线位置之后，就需要考虑光线在当前层级下，通过此处一个像素所覆盖的深度范围

```c++
  // 计算光线和当前像素边界的两个碰撞点
  float2 dt_entry = abs(ceil(Pcurr/div - state.posDir) * div - state.P0) * state.pixlen;
  float2 dt_exit = abs(floor(Pcurr/div + state.posDir) * div - state.P0) * state.pixlen;
  float t_entry = clamp(max(dt_entry.x, dt_entry.y), 0, state.t);
  float t_exit = clamp(min(dt_exit.x, dt_exit.y), t_entry, 1);
  // 计算当前的光线与像素边界碰撞点的深度值(注意深度值的分布)
  // k的值是摄像机深度的倒数
  float rayEntryDepth = state.k0 + state.dk * t_entry;
  rayEntryDepth = 1.0f / (rayEntryDepth * GCameraParam.y);
  float rayExitDepth = state.k0 + state.dk * t_exit;
  rayExitDepth = 1.0f / (rayExitDepth * GCameraParam.y);
  float rayDepthMin = min(rayEntryDepth, rayExitDepth);
  float rayDepthMax = max(rayEntryDepth, rayExitDepth);
```

​	光线从进入当前像素到离开当前像素，在像素的边缘的两个交点决定了光线在经过该像素时候的深度覆盖范围；光线在高层级的深度图中穿梭时，穿过一个像素意味着实际上穿过0级深度图的多个像素，所以不同层级，光线穿过一个像素的深度覆盖范围都不相同

​	代码2~5即在求出光线与当前层级经过的像素的边界上的两个交点(t是两个点的插值结果)，下面只列举一种情况：

![1523763724573](.\TEntryIllustrate.png)

**Figure7: 展示光线与像素求入射交点的过程；P0为光线的起点，P为光线当前位置，P_entry为光线前进方向上进入像素范围的交点，P_exit为光线前进方向上离开像素范围的交点**

​	`ceil(Pcurr/div - state.posDir) * div`意指先求出P点在当前层级中处于的像素位置(除以div)，再按照光线前进方向，大致求出其入射交点位置于像素的左上角，并将该位置还原会层级0中(乘以div)。

​	`-state.P0`即是求出光线起点与该左上角角点之间的差距，由于光线入射必须经过左边或者上边，所以必定会有dx = P_entry.x - P0.x 或者 dy = P_entry.y - P0.y。

​	上图展示的是dx = P_entry.x - P0.x的情况。很显然dx * state.pixlen.x才是我们需要求出的t_entry值，因为此时的P0 + dP * t_entry = P_entry。

​	`float t_entry = clamp(max(dt_entry.x, dt_entry.y), 0, state.t);`即使选出dx * state.pixlen.x和dy * state.pixlen.y中较大的一个作为t_entry，不难发现dx*state.pixlen.x要更大:
$$
pixlen.x = abs(\frac{1}{dP.x}) \\
pixlen.y = abs(\frac{1}{dP.y}) \\
\because \frac{dP.y}{dP.x} > 1 \\
\therefore \frac{pixlen.y}{pixlen.x} < 1 \\
又\because \frac{dx}{dy} > 1 \\
\therefore \frac{dx}{dy} > \frac{pixlen.y}{pixlen.x} \\
\because dy > 0 且 pixlen.x > 0 \\
\therefore dx * pixlen.x > dy * pixlen.y
$$
​	同理可以求出t_exit；

​	第8~11行，是为了将求出的交点的深度值，先还原到摄像机空间下，之后再除以远平面值，得出与深度图中格式一致的深度值，方便之后与深度图中值直接进行比较

#### 判定碰撞情况

```c++
  bool collision = false;
  for (state.layer = 0;
    state.layer < NUM_LAYER;
    ++state.layer, levelUV.y += GCameraParam.w / div) {
    float2 sceneDepth = getLevelData(state.curLevel, levelUV);
    if (sceneDepth.x > rayDepthMax) break;
    if (sceneDepth.y < rayDepthMin) continue;
    collision = true;
    float f = clamp((sceneDepth.x - rayDepthMin) / (rayDepthMax - rayDepthMin), 0, 1);
    if (refRay.dir.z > 0)
      state.t = max(state.t, t_exit * f + t_entry * (1 - f));
    break;
  }
```

​	碰撞检测是针对当前层级(state.curLevel)展开的，从layer = 0即像素位置上最靠前的片元开始检测，直到光线与当前像素发生碰撞，或者遇到提前结束条件，跳出循环；

​	每检查玩一个layer的片元，layer往下移，同时索引片元信息的levelUV也进行偏移(GCameraParam.w是窗口的高度，同时也是0级深度图的高度，除以div获得当前层级深度图一个layer的高度)。

​	退出循环的基本条件state.layer < NUM_LAYER意味着，在检测完该像素位置上所有片元后，光线均未与其发生碰撞，光线位置要更远离摄像机；

​	getLevelData函数从当前层级中，根据索引levelUV获得当前要对比的片元深度信息对sceneDepth，sceneDepth.xy分别记录了像素位置上深度以及加上偏移后的深度值(d_min, d_max)；

​	`if (sceneDepth.x > rayDepthMax) break;`意味着光线的最后端在当前片元的前方，之后的片元均会在当前片元的后方，故光线不可能与当前以及之后的片元发生碰撞，于是提前结束循环

​	`if (sceneDepth.y < rayDepthMin) continue;`当前片元的最深深度还在光线的前方，光线不可能与当前片元碰撞，但可能与之后的片元碰撞，所以跳到下次循环；

​	![1523772766421](.\CollisionIllustrate.png)

**Figure8: 碰撞情况示意图**

​	通过第6，7行代码的筛选后，可以判断当前光线与当前像素是有碰撞情况发生的；故标志collision为true

​	第9~11行代码，旨在针对朝着屏幕里面走的光线，将它们碰撞的位置进一步精确；这个处理只对上图中的a情况有作用，a在经过处理后，光线的位置会被截断到与片元表层深度交点的位置上，而对于剩余情况不做任何处理。

> 注意：暂时不清楚为何其余情况均不做处理

```c++
  // check if done
  if (collision && state.curLevel == 0) {
    state.done = true;
    break;
  }
```

​	假如当前发生碰撞，同时当前深度图层级在0级，则追踪结束

#### 更新追踪状态

```c++
  // update
  state.t = collision ? state.t : t_exit + EPSILON;
  // update current level
  acceleration_delay = max(0, collision ? 3 : acceleration_delay - 1);
  int new_level = clamp(state.curLevel + (collision ? -1 : 1), 0, LEVELS);
  state.curLevel = collision || acceleration_delay == 0 ? new_level : state.curLevel;
}
```

​	假如当前没发生碰撞，则移动光线位置到光线与出像素位置的交点上，并增加少量偏移防止光线落在元像素内重复判断。

> 注意：假如当前发生碰撞，则基本不改变当前光线位置，除非发生上述a情况的碰撞，则state.t会被事先更新

​	acceleration_delay的作用在于，假如之前发生了碰撞，导致当前层级下降，而在之后的检测中发现碰撞失败，又会将层级提高，又进行了与前一次相同的碰撞检测，导致浪费循环次数；

​	![1523774202319](.\accelerationDelay.png)

**Figure9: acceleration_delay示意图：若光线当前位置p1，于大像素块中发生碰撞，则光线会保留在p1位置，同时光线所在层级下降；下一个循环，P1位置上没有发生碰撞，此时存在acceleration_delay阻止层级的提高而使层级停留在当前状态，并使光线移动至P2处，下一个循环，P2位置上检测到碰撞**

​	acceleration_delay为最大为3，意味着可以忍受光线在当前层级移动3个像素而未发生碰撞；这是在上下层级之间差距一半的情况下，有效利用检测次数，减少层级切换的最佳值

​	由于有acceleration_delay的介入，无法判断当前是否需要下降/上升一层，故使用new_level根据碰撞情况计算出下一层的情况，再结合acceleration_delay具体判断是否引用新层级。

#### 写入碰撞结果



```c++
  if (state.done) {
    uint2 pixel = getCurrSSPos(state);
    pixel.y = state.layer ? pixel.y + GCameraParam.w * state.layer : pixel.y;
    result[tuv] = BasicDiffuse.Load(uint3(pixel, 0));
  }
```

​	state.done = true意味着当前光线与场景中的某个像素碰撞成功；

​	先获得光线最终碰撞的位置，根据光线碰撞时候所在layer索引当前碰撞的片元位置

​	BasicDiffuse是记录了当前所有片元颜色信息的缓冲，从其中读取碰撞片元的颜色信息并填入当前渲染的像素中。



​	经过上述步骤，应该会生成一张仅包含当前场景镜面反射结果的图片

------
[1]: https://dl.acm.org/citation.cfm?id=2383624	"Real-Time Concurrent Linked List Construction on the GPU"
[2]: https://dl.acm.org/citation.cfm?id=2614217.2614274	"Screen Space Cone Tracing"
[3]: https://dl.acm.org/citation.cfm?id=3105781	"Hierarchical Multi-Layer Screen-Space Ray Tracing"

