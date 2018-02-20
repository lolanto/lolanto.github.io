# Pieces About DirectX

## 1. 创建CubeMap资源

利用长度为6的二维贴图数组作为CubeMap。创建资源需要填写**D3D11_TEXTURE2D_DESC**。其中要注意属性：

* MiscFlags必须为D3D11_RESOURCE_MISC_TEXTURECUBE
* Width/Height必须相同
* ArraySize必须为6

创建**Render Target View**，ViewDimension以及Union中选择2DArray相关

创建**Shader Resource View**, Dimension和Union需要选择Cube相关

绑定RenderTarget时候，所有RenderTarget都必须是同种类型，所以不能和普通的Texture2D资源同时绑定

## 2. MRT && RTA

### Multi-Render Target:

多重渲染对象，DX中最多可以在pixel shader上绑定8个渲染对象，这些渲染对象必须是同种类型的资源，每个渲染对象的数据格式可以不同，记录的内容也可以通过pixel shader决定。

### Render Target Array:

渲染对象数组，渲染对象必须是数组类型的资源(e.g. Texture2DArray)。在Geometry shader中通过指定输出SV_RenderTargetArrayIndex数值，决定输出的图元会被应用到哪一个渲染对象数组元素中。SV_RenderTargetArrayIndex可在GS中设置，在PS中设置和读取。

### 区别:

MRT适用于获取当个镜头下的多个参数(e.g. 生成G-Buffer)，RTA适用于同时记录场景的多个镜头(e.g. 生成cube map)

## 2. GeometryShader

主要作用：输入**图元**产生**新的图元**。