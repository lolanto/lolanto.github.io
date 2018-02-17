# Pieces About DirectX

## 1. 创建CubeMap资源

利用长度为6的二维贴图数组作为CubeMap。创建资源需要填写**D3D11_TEXTURE2D_DESC**。其中要注意属性：

* MiscFlags必须为D3D11_RESOURCE_MISC_TEXTURECUBE
* Width/Height必须相同
* ArraySize必须为6

创建**Render Target View**，ViewDimension以及Union中选择2DArray相关

创建**Shader Resource View**, Dimension和Union需要选择Cube相关

绑定RenderTarget时候，所有RenderTarget都必须是同种类型，所以不能和普通的Texture2D资源同时绑定