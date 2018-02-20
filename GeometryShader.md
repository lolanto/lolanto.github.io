# About Geometry Shader in DirectX

Geometry shader的作用为以图元为单位修改由vertex shader传递下来的顶点信息。

```c++
[maxvertexcount(NumVerts)]
void ShaderName (
  PrimitiveType DataType Name [ NumElements ],
  inout StreamOutputObject);
```

<u>注意点</u>

1. maxvertexcount 

该参数决定了GS能够输出的顶点的数量，`[maxvertexcount(N)]`超出N的输出的顶点会被忽略

2. 输入

决定输入的图元的类型，同时影响NumElements的取值(e.g. PrimitiveType = triangle; NumElements = 3)。DataType为顶点数据类型尽量与VS的输出结构保持一致

3. StreamOutputObject

```c++
inout StreamOutputObject<DataType> Name;
```

输出的顶点流，StreamOutputObject可以是pointStream, lineStream或者triangleStream。

而尖括号中的DataType则是自定义的顶点输出结构，尽量与下一级shader的输入结构保持一致

4. SV_RenderTargetArrayIndex

在GS中可以进行编辑，作为GS的顶点输出数据之一，决定了该图元使用的渲染对象数组中的位置。

5. Append/RestartStrip

输出stream的方法，Append将当前的顶点放入到输出流中作为新图元的其中一个顶点，RestartStrip则表示“清空”输出流，重新开始新图元的构建。

6. 透视除法

透视除法在Geometry shader之后再进行