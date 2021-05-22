Redux或者Koa中都提供了中间件的机制，这可以使我们有机会检阅我们关心的数据。这样我们修改侦测数据，或者添加辅助功能。

中间件是一系列函数的集合，插件自身会遍历每一个中间件，把用户关心的数据当做参数调用用户注册的函数，用户的函数应当返回新的数据，然后作为下一个中间件的参数调用下一个中间件。

将多个中间件聚合起来的函数就是compose函数了

``` js
const compose1 = (...args) => (val) => {
  args.forEach(e => val = e(val))
  return val
}
```
调用也比较简单

`
compose1([fun1, fun2, fun3])(initialValue)
`

Redux里面提供了一种更为简洁的方式：
``` js
const compose2 = (...lists) => lists.reduce((a, b) => (arg) => a(b(arg)))
```
如果我们把a,b,c,3个函数调用compose2，那么会得到一个新的函数:
`(arg) => a(b(c(arg)))`，可以发现是最后的中间件最先执行。

对于同步顺序执行的中间件函数确实比较简单，但是在`koa`里则不一样，因为一个中间件可能是异步的，这种情况下中间件就传一个`next()`方法给用户，无论是异、同步任务，用户处理完成了了后再调用`next()`表示告诉框架已经完成了。

``` js
function compose(middleware) {
  return function (context) {
    return dispatch(0)
    function dispatch(i) {
      let fn = middleware[i]
      if (!fn) return Promise.resolve()
      try {
        return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
      } catch (err) {
        return Promise.reject(err)
      }
    }
  }
}

let f0 = async (c, n) => {
  console.log(1)
  n();
  console.log(2)
}

let f1 = (c, n) => {
  return new Promise((r) => {
    console.log(3)
    setTimeout(n, 1000);
    console.log(4)
  })
}

let f2 = async (c, n) => {
  console.log('5')
  await n()
  console.log('6')
}

let combine = compose([f0, f1, f2])
combine(null)

[Running] node "/Users/yangjie/project/myPromise/target.js"
1
3
4
2
5
6
```

`compose`函数返回一个方法，第一次调用时会调用内部的`dispatch(0)`方法，这个方法返回一个promise,该pormise的resolve方法会直接执行`middleware[0]`，同时注入两个参数:`context, dispatch.bind(null, i + 1)`, `dispatch.bind(null, i + 1)`也就是next方法，就是调用下一个中间件。


**特别注意中f1方法中，在setTimeout中调用了n，但是当前所在的promise并没有resolve掉，倘若f0中的n()前面加个 await，那么console.log(2)将不会被执行。所以如果中间件存在Promise的写法，既要主要调用next()方法，让下一个中间件执行， 也要记得resolve回去，好让上一个中间件可以被正常await，上面的代码就是错误的示范。**