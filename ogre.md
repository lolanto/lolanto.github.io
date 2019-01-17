1. 什么是AutoParamDataSource，*它对渲染过程有什么贡献/起到什么作用？？*

   这个结构包括了管线可能用到的mvp矩阵，摄像机位置，雾的参数，光源的信息。以及摄像机的信息，摄像机Frustum的信息，渲染对象的信息，场景管理器，渲染pass信息

SceneManager包含了RenderSystem，摄像机列表，负责阴影的渲染，光源的管理，雾的管理，包括渲染队列，创建管理场景图(SceneNode)，场景图中的Object以及Entity

主要功能包括：

1. 光源和阴影管理
2. 渲染队列管理
3. 场景图管理

渲染开始的入口函数sceneManager::_renderScene

2. Viewport是摄像机和RenderTarget的集合。记录了由哪个摄像机出发渲染，将结果存储到哪个RenderTarget上，以及存储的范围(局部/全部)。