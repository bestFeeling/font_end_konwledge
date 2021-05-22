最近了解到了浏览器有个微任务方法`queueMicrotask`,想想以前自己用`settimeOut`实现过的Promise，现在正好可以替换掉，可是一时间没有想清楚为什么要用事件队列？应该用在哪儿？

凭借自己对Promise的朴素理解，Promise有两个特点：
 1. `resolve`方法的执行和`then`方法的注册，如果`then`方法先于`resolve`注册，那么`then`方法会被保存在内部的队列里等待`resolve`方法的执行，反之，如果已经`resolve`已经执行了，那么已经处于`pending`状态，`then`方法直接执行
 1. `then`应该返回一个新的`Promise`已提供链式调用

这样一想好像确实不需要用到事件队列：
``` js
function myPromise(func) {
  this._value = null
  this._status = 0
  this._resQueue = []
  const _resolve = (val) => {
    if (this._status != 0) return
    this._status = 1
    this._value = val
    while (this._resQueue.length) this._resQueue.shift()(this._value)
  }
  func(_resolve)
}
myPromise.resolve = function (val) {
  return new myPromise(r => r(val))
}
myPromise.prototype.then = function (onResolve) {
  const p2 = new myPromise((r) => {
    const resolveFn = () => {
      let x = onResolve(this._value)
      x instanceof myPromise ? x.then(r) : r(x)
    }
    this._status == 1 ? resolveFn() : this._resQueue.push(resolveFn)
  })
  return p2
}
```

30行的乞丐版Promise
然后是测试代码
``` js
myPromise.resolve().then(() => {
  console.log(0);
  return myPromise.resolve(4);
}).then((res) => {
  console.log(res + '__')
})

myPromise.resolve().then(() => {
  console.log(1);
  return 2
}).then((r) => {
  console.log(r);
  return 3
}).then(() => {
  return new myPromise(r => setTimeout(() => r(4), 1000))
}).then((r) => {
  console.log('1000ms later ', r)
  console.log(5);
}).then(() => {
  console.log(6);
})

[Running] node "/Users/yangjie/project/myPromise/myPromise.js"
0
4__
1
2
1000ms later  4
5
6
```
很好，满足我平时的对Promise的使用，但是我完全没有使用事件队列，有的只是普通的函数回调而已。
后来参考别人的帖子，才发现了这种写法的错误之处：
``` js
const p2 = new myPromise(r => {
    r(1)
  })
  .then((val) => {
    console.log(val)
    return p2
  })
  .then((val) => {
    console.log(val)
  })

[Running] node "/Users/yangjie/project/myPromise/myPromise.js"
1
/Users/yangjie/project/myPromise/myPromise.js:37
    return p2
    ^

ReferenceError: Cannot access 'p2' before initialization

```

p2这个Promise都还没有初始化完成就被当做已经存在的变量给return了，解决这个问题可真是微事件任务登场的好时机。当前执行栈完成了以后，js就会去把所有的微任务列队里的task拿出来全部执行，然后再去执行下一个宏任务。所以把这个then方法的注册函数用`queueMicrotask`包裹一下即可：
``` js
    const resolveFn = () => {
      queueMicrotask(() => {
        let x = onResolve(this._value)
        x instanceof myPromise ? x.then(r) : r(x)
      })
    }
```

以前虽然一直知道Promise是微任务，可是一直不知道为什么是这样。

而理解Promise执行的重心也不在于微任务，30行顺序执行的的Promise更能让我理解Promise执行的内部逻辑。
