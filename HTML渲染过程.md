# HMTL渲染过程
面试有被问到各种HTML渲染过程的细节，直接被问倒了，查阅资料后，从时间线的层面整理了一下HTML的渲染过程。

总前提： `DOM tree + CSSOM tree才能生成render tree,进而进行layout布局计算,然后才是paint绘制到浏览器展示出来`

假设场景是js部分极其简单，js的加载执行是快于css的。


1. HTML加载完毕，开始解析HTML内容。
    - `document.readyState`进入`loading`状态
1. HTML解析时遇到`<link href`时，会并行的去加载css,并不会阻塞html的解析。
    - 不阻塞html解析，但是会阻塞html渲染，没有加载到css会影响css dom的生成，进而影响到render tree的生成。
1. HTML继续继续解析，遇到`<script>`标签时，等待js文件的加载，并等待js脚本执行完成。
    - 现代浏览器基本上会猜测预加载，在等待当前js加载的时候如果嗅探到后续仍有js需要加载，那么后续的js的网络加载也会发起，但是后续的脚本即使加载完成，也需要等待前面的js代码执行之后再执行(暂不考虑async, defer)
1. HTML解析完毕
    - `document.readyState`进入`interactive`状态
    - 此时DOM已经生成,可用通过js代码操作dom
    - 此时css可能尚未加载完成,所以页面不一定被渲染
1. css加载完成,页面开始绘制
    - `document.readyState`进入`complete`状态
    - 此时页面的图片可能仍未加载完成
