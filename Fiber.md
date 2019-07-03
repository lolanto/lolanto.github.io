# Fiber

---

Fiber是需要应用负责调度的执行单元。Fiber的执行依赖于thread，假如应用需要手动调度thread的执行，那么使用fiber将是个不错的选择。

对于Fiber而言，使用线程相关的内容(比如TLS)，或者调用线程相关的函数时，其所影响的线程是它当前正在依赖的线程。Fiber拥有的数据包括Fiber本身的一些属性，stack以及用于记录当前stack状态还有执行状态的register。

Fiber的调度是由应用决定的，这就意味着应用可以控制Fiber的执行从而降低与其它Fiber的冲突(不像线程由操作系统负责调度)。一个Fiber可以启动其它Fiber，当Fiber所依赖的线程执行时，Fiber也就开始执行。所以Fiber执行仍然依赖系统调度，但Fiber启动可以由应用控制。
