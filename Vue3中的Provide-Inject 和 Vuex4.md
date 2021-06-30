# Vuex4 原理

知乎上看到的文章，做个记录


#### 数据侦测原理的改变
- 在vuex之前的版本，数据侦测是直接创建一个新的vue实例：`new Vue()`, 在vuex4中则是使用了核心库`reactive`
    ``` js
    function resetStoreState (store, state, hot) {
      store._state = reactive({
        data: state
      })
    }
    ```
#### 插件的注入
- 和以往一样，Vue使用`app.use`注册插件, `app.use`方法接受的第一个参数是一个对象或者一个方法，如果是方法，则注入app实例调用该方法，如果是对象则调用的`plugin.install`方法，同时，一样的注入app实例作为参数。
    ``` js
    use(plugin, ...options) {
      if (plugin && isFunction(plugin.install)) {
        installedPlugins.add(plugin);
        plugin.install(app, ...options);
      }
      else if (isFunction(plugin)) {
        installedPlugins.add(plugin);
        plugin(app, ...options);
      }
      // 支持链式调用
      return app;
    },
    ```
- `Store`中的`install`方法中调用`app.provide`注入`Store`实例
    ```js
    export class Store{
      install (app, injectKey) {
        // 可以传入 injectKey
        app.provide(injectKey || storeKey, this)
        // 为 option API 中使用
        app.config.globalProperties.$store = this
      }
    }
    ```
#### vuex4中是使用`provide`注入`store`实例，然后子组件中使用`inject`获取的。本质上provide只是context上provides上的一个键值对
#### 然后我们在组件中的`setup`中使用`store`是使用 vuex提供的`useStore`方法，其内部是直接用`inject`拿到之间注入的`store`实例：
```js 
export function useStore (key = null) {
  return inject(key !== null ? key : storeKey)
}
```

#### 然后比较有意思的就是`provide`和`inject`的实现：
``` js
function provide(key, value) {
    if (!currentInstance) {
        if ((process.env.NODE_ENV !== 'production')) {
            warn(`provide() can only be used inside setup().`);
        }
    }
    else {
        let provides = currentInstance.provides;
        // by default an instance inherits its parent's provides object
        // but when it needs to provide values of its own, it creates its
        // own provides object using parent provides object as prototype.
        // this way in `inject` we can simply look up injections from direct
        // parent and let the prototype chain do the work.
        const parentProvides = currentInstance.parent && currentInstance.parent.provides;
        if (parentProvides === provides) {
            provides = currentInstance.provides = Object.create(parentProvides);
        }
        // TS doesn't allow symbol as index type
        provides[key] = value;
    }
}
```
```js
function inject(key, defaultValue, treatDefaultAsFactory = false) {
    // fallback to `currentRenderingInstance` so that this can be called in
    // a functional component
    // 如果是被一个函数式组件调用则取 currentRenderingInstance
    const instance = currentInstance || currentRenderingInstance;
    if (instance) {
        // #2400
        // to support `app.use` plugins,
        // fallback to appContext's `provides` if the intance is at root
        const provides = instance.parent == null
            ? instance.vnode.appContext && instance.vnode.appContext.provides
            : instance.parent.provides;
        if (provides && key in provides) {
            // TS doesn't allow symbol as index type
            return provides[key];
        }
        // 如果参数大于1个 第二个则是默认值 ，第三个参数是 true，并且第二个值是函数则执行函数。
        else if (arguments.length > 1) {
            return treatDefaultAsFactory && isFunction(defaultValue)
                ? defaultValue()
                : defaultValue;
        }
    }
}
```

首先的一个前提是`provides`这个存放键值对信息的数据是放在每个组件实例上的，也就是在每个组件的`setup`中使用

其次是在`inject`寻找`provides`是遵循就近原则的，如果`instance.parent`上有`provides`，那么直接就用这个，否则拿`appContext`的`provides`,最后返回`provides[key]`作为结果

然后在`provide`中为`provides`设置值的时候，在第一次执行时，
`parentProvides`和`currentInstance.provides`是一样的， 这个时候重写`currentInstance.provides`：
```js
provides = currentInstance.provides = Object.create(parentProvides);
```
创建一个新的`provides`，让`provides`的`__proto__`指向`parentProvides`,之后在`provides`设置的新的值则是`provides`自身的属性。

所以当前实例的`provides`可能是一个有很长原型链的对象，分别保存着父级组件的`provides`，也正是基于原型链，所以获取一个`key`值时候会直接根据js的特性在原型链上查找，如果存在相同`key`值，还会就近查找，找到最近一个原型链上存在`key`的键，即可返回值。