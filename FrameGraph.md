# Frame Graph

---

## Goals

* 对整一帧的执行过程有高层次的，抽象的理解(*high-level knowledge*)
  * 从而获得简化资源管理的可能
  * 从而简化渲染管道的配置过程
  * 简化异步计算以及资源barrier的设置过程
* 允许self-contained以及高效的渲染模块
* 可视化地调试复杂地渲染过程

> 一个frame graph中包含了pass以及资源操作，两种元素构成*有向无环图*

## Design(设计决策)

* 移除立即渲染模式(immediate mode rendering) ?
* 渲染代码被拆分成多个passes
* 渲染代码被拆分成三大类(描述三个过程)：
  * setup过程
  * compile过程
  * execute过程
* 每一帧都会重建整个graph
* 代码驱动的架构?

### setup过程

* 定义渲染/计算Pass(的状态？pipeline state object?)
* 定义pass的输入输出(资源依赖关系)
* 定义的过程与之前立即模式的过程类似?

>必须声明一个Pass中使用的所有资源，及其使用方式——read/write/create
>
>外部的/长期存在的资源以“导入”的方式设置在graph中

> 在PPT示例当中，由于使用了代码驱动的方式，所以实际上一个Pass的setup/定义过程，即是将pass的配置代码(资源的创建/使用说明)写入某个函数中，该函数就成为pass的一部分(part of setup)

### compile过程

* 删除没有被引用到的pass或者资源
  * can be a bit more sloppy during declaration phase(能够在setup过程中进行一些预处理?)
  * 从而减少配置过程的复杂度
  * 删除掉一些“conditional pass"，即在某些条件下(比如非debug)不会被执行的pass；或者通过资源的依赖关系，筛选掉不被使用的资源
* 计算资源的声明周期
* 根据资源的使用情况分配GPU资源
  * 可以在这里应用简化的贪心算法完成计算
  * 在第一次使用前保证资源创建完成，并在最后一次使用完成后被释放
  * 假如使用了异步计算，就需要考虑扩展资源的生命周期
  * 根据资源的使用情况，推导资源的绑定flag

### execute过程

* 执行每个pass的execute回调函数
* 使用立即模式下的渲染代码
  * rendercontext的API调用
  * 状态设置，资源绑定，shader等
  * draw call, dispatch等
* 利用Handle索引setup过程中生成的资源

> 总结三个过程就是——声明资源的使用情况；组合所有的pass，确认哪些pass可以被正确组合，哪些可以被筛选掉；按组合后的顺序执行pass内的指令(draw/dispatch call)

## 异步计算

* 能够从Pass的依赖关系中自动决策哪些compute过程可以异步执行
* 也支持手动设置
  * 该方式可能对性能有一定的提高，但更可能损害性能
  * 增加内存的使用
* Opt-in per render pass?(Opt-in: 选择使用)
* Kicked off on main timeline
* 需要在另一个使用异步计算产生的资源的queue上设置合理的fence
* 资源的生命周期要自动扩展到同步点位置

## 使用C++声明Pass

* 每个Pass都可以声明为一个class
  * 该方式会破坏代码流(Break code flow) ?
  * 需要大量的模板？
  * 将当前代码迁移到frame graph会很困难
* 使用Lambda表达式
  * 保留代码执行流(preserves code flow)
  * 原有代码只需要少量修改即可完成迁移
    * 将过去的代码用Lambda表达式进行封装
    * 增加资源的使用声明

---

## Transient Resource System

* Transient Resource指生存周期不超过一帧的资源
  * 深度缓冲区，Render Target等
  * 尽量减少资源在一帧中的生命周期
* 在使用的时候才分配资源
  * Directly in leaf rendering systems
  * 尽可能早地将临时资源占用的内存空间释放
  * make it easier to write self-contained features
* 这是Frame Graph的重要组件

> 该系统的backend实现依赖具体的平台

> Transient Resource System的存在就是为了能发挥Memory Aliasing的作用

### Memory Aliasing considerations

* 这个过程需要非常小心
* Ensure valid resource metadata state(确保资源的元数据(初始数据)，是否需要保留/忽略之前的数据)
* 确保资源的生命周期安排是合理的
  * 要注意同时在compute和graphic上使用的资源
  * 注意异步计算
  * Ensure that physical pages are written to memory before reuse

## 总结

纵观PPT描述，作者介绍的Frame Graph应该也是输入多pass的组合，然后由Graph负责对所有的pass进行处理，筛选出可以进行异步计算的pass，尔后安排资源管理以及内存分配工作。

PPT中实现的Frame Graph使用的是code-driven的设计方式，所有的pass都是预先以代码形式构造结构体，而后将结构体对象组合出graph出来。虽然pass中设置的shader应该可以在外部修改，但实际上假如在pass的setup阶段设置了GPU资源的使用状态，shader的形式基本上就已经固定下来了。

data-driven的实现方式更趋向于用外部文件描述pass的setup，execute部分，在引擎启动时再将这些外部文件实例化成pass对象。同时还可以根据配置文件将这些pass组合成graph。

哪些pass应该被组合应该是由上层传入的pass决定的。因为每一帧渲染的对象都会不同，每个对象使用的pass自然也不同，每一帧构成的graph也不同。所以作者才会在特性中强调frame graph是可以在运行时动态组合的。

既然如此，应用frame graph的renderer对data-stream的处理性能要求很高。