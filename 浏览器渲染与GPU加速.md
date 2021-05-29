## 浏览器解析
1. 构建dom：浏览器包含html解析器和css解析器。分别解析html和css代码。html解析器会解析出dom，css解析器会解析出cssDom
2. 构建渲染树：合并树，从Dom根结点开始遍历每一个可见的节点，为其找到适配的cssDom规则并应用。将两个树合并成render tree 
3. 布局(layout): 根据render tree计算出每个节点在屏幕中的位置,当页面元素位置、大小发生变化时需要重新计算，这个过程称为回流（Reflow), 最终会生成layoutTree
4. 渲染(paint): 遍历渲染树，进行每个节点的绘制, 当页面元素位置、大小、颜色等发生改变时，会引起重绘(repaint)，仅当颜色发生改变时或者其他不影响render tree结构改变时，直接重绘，不会回流

5. 渲染层合成（Composite）
    1. 一个DOM节点对应了一个渲染对象，处于相同坐标空间（z 轴空间）的渲染对象，都将归并到同一个渲染层（`RenderLayer`）中，对于满足形成层叠上下文条件的渲染对象，浏览器会自动为其创建新的渲染层，包含一下方式：
        1. 根元素 document
        2. 有明确的定位属性（relative、fixed、sticky、absolute）
        3.opacity < 1
        4. 有 CSS fliter 属性
        1. 有 CSS transform 属性且值不为 none
        1. overflow 不为 visible
    1. 某些特殊的渲染层会被认为是合成层（`Compositing Layers`），合成层拥有单独的图形层 `GraphicsLayer`,而其他不是合成层的渲染层，则和其第一个拥有 `GraphicsLayer` 父层公用一个。变成合成层的一些方式：
        1. 3D transforms：translate3d、translateZ 
        1. video、canvas、iframe 
        1. position: fixe
        1. will-change

总结： 一个dom节点对应一个渲染对象，个多渲染对象会归并到一个渲染层，多个渲染层可能会归并到一个图形层中，每一个图形层拥有内容呈现的模型和上下文，最终交由GPU绘制到屏幕上。而合成层是特殊的渲染层，不会归并到一个现有的图形层中，而是归并到一个独立的、新增的图形层中进行绘制。

GPU加速指的就是把需要关注性能的dom节点提升为单独的合成层，合成层的渲染会交由GPU处理，并且合成层的回流与重绘只会影响当前的层， 不会影响其他层

补充：另一个渲染优化的知识点：`requestAnimationFrame`, 与jquery时代setInterval画动画的方式不同，`requestAnimationFrame`不需要指定执行的时间间隔，由浏览器下一次重绘之前调用,`requestAnimationFrame`采用系统时间间隔，保持最佳绘制效率，不会因为间隔时间过短，造成过度绘制，增加开销；也不会因为间隔时间太长，使用动画卡顿不流畅，让各种网页动画效果能够有一个统一的刷新机制，从而节省系统资源，提高系统性能，改善视觉效果

补充：`Fixed`布局永远都是相对视窗(viewport)来定位的吗？答案是否定的，如果fixed元素的祖先节点是合成层，那么就会变成相对祖先布局






