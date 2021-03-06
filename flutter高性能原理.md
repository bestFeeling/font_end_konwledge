# flutter高性能原理

## 架构机制方面：
 1. Hybrid的架构是基础`Webview`, Webview的渲染性能天然低于原生
 1. RN/Weex 的架构中需要实现Bridge桥接层，会多一层转换
 1. Flutter没有中间层损耗，Flutter引擎会直接基于渲染引擎绘skia制UI

 ## 语言层面
 1. dart单线程异步消息机制，拥有单线程的诸多好处(类比于js，对于能够控制视图展现的代码，单线程可以保证更改来为唯一，试想如果两个线程同时修改一个视图，必然会带里锁和调度的问题)
 2. 由于Isolate之间是无法共享内存，可以让Dart实现无锁的快速分配
 3. 新生代半空间的垃圾回收方式，这种方式非常适合大量Widgets对象创建和销毁优化

## 编译优化
 1. 打包时AOT即原生打包 开发时JIT
 1. 自带tree-shaking， 没有反射相关的代码


## UI绘制的高性能
  1. Widget实际对应的渲染中间层是Element,如果Element没有发生变化，对应的渲染对象RenderObject也不会变化。所以绘制时，大量没有发生改变的RenderObject会保持原来的引用，这会减少计算量。
  