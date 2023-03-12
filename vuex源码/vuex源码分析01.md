大家好，我是 chosan

今天我打算看看 Vuex 的源码。

这里主要演示了我个人阅读源码的流程，希望对您有所启发。

这里假设您已经对 Vuex 的使用有了大致的了解，如果不清楚什么是 `state`、`mutation`、`getter` 的话可以先看看[官方文档](https://vuex.vuejs.org/)，了解最基本的用法即可。

还是老样子，先从官方「[开始](https://vuex.vuejs.org/zh/guide/)」部分入手：

![image-20220723233056685](https://cdn.chosan.cn/posts-meta/image-20220723233056685.png)

这是官方入门示例的代码，红框部分是初始化 `store` 的逻辑，它调用 `createStore` 方法，传入了一个对象参数，其中包含了：

- `state` 方法：用于返回当前模块的初始状态
- `mutations` 对象：用于定义当前模块的所有 mutation

因此我们可以直接先看 `createStore` 中经历了些什么。

从入口文件 `src/index.js` 中我们可以直接定位到 `createStore` 的代码，发现它只是简单的创建并返回了一个 `Store` 实例：

![image-20220723233731869](https://cdn.chosan.cn/posts-meta/image-20220723233731869.png)

顺藤摸瓜，我们找了的 `Store` 类，看看它的构造函数都做了什么：

![image-20220723233921912](https://cdn.chosan.cn/posts-meta/image-20220723233921912.png)

[源码](https://github.com/vuejs/vuex/blob/01f87f0c3d59d0796a2535719dfa8328d1af390d/src/store.js#L19)

首先是一大堆初始化逻辑，大多都是初始化为常见的数据类型，但其中有一个比较特殊，它创建了一个 `ModuleCollection` 的实例，我们看看它做了什么：

![image-20220723234136367](https://cdn.chosan.cn/posts-meta/image-20220723234136367.png)

仅仅是调用了 `this.register` 方法，透传了 `rawRootModule` 参数，而 `rawRootModule` 就是我们传递给 `createStore` 的初始对象，继续看看`this.register`

![image-20220723234405486](https://cdn.chosan.cn/posts-meta/image-20220723234405486.png)

[源码](https://github.com/vuejs/vuex/blob/01f87f0c3d59d0796a2535719dfa8328d1af390d/src/module/module-collection.js#L28)

根据我们的使用场景，进到这里来它只会执行红框中的代码，仅仅是初始化了一个 `Module` 实例，然后赋值给了 `this.root`，看看 `Module` 的初始化流程：

![image-20220723234653003](https://cdn.chosan.cn/posts-meta/image-20220723234653003.png)

[源码](https://github.com/vuejs/vuex/blob/01f87f0c3d59d0796a2535719dfa8328d1af390d/src/module/module.js#L4)

参数 `rawModule` 就是我们传递给 `createStore` 的参数对象，因此这段代码几乎也就只做了一件事 —— 即获取到参数对象中的 `state` 并且根据情况拿到最终的 `state`，然后挂载到自身。

因此最终下来，`Store` 上存在一个 `_modules` 属性，而 `_modules` 属性上存在一个 `root` 属性，而`root` 上挂载了一个 `state` 属性，这个 `state` 就是最终的 `state` 。即整个模块最初的 `state` 存在于 `store._modules.root.state` 。

此刻已经知道了 state 保存的具体位置，我们继续，接下来这几行代码是将 `commit` 和 `dispatch` 绑定到 `store`：

![image-20220724094925093](https://cdn.chosan.cn/posts-meta/image-20220724094925093.png)

将 `dispatch` 和 `commit` 的 `this` 绑定到 `store` 上，从而使得我们可以单独调用这两个方法也能有正确的 `this` 指向，而不用通过 store 来调用。

继续往下是两个函数的调用，根据注释我们可以知道：

- 第一个红框中的 `installModule` 函数功能是初始化 root 模块并递归注册所有的子模块，将所有模块的 getter 放到模块的 `_wrappedGetters` 中。
- 第二个红框中 `resetStoreState` 函数功能是初始化 `state`，为其添加响应式，另外将所有 `_wrappedGetters` 注册为计算属性。 

![image-20220724095333497](https://cdn.chosan.cn/posts-meta/image-20220724095333497.png)

这里有个很重要的点：我们在 vuex 中写的 getter 最终会被处理为计算属性！

先来看看 `installModule` 做了些啥：

![image-20220724095923981](https://cdn.chosan.cn/posts-meta/image-20220724095923981.png)

[源码](https://github.com/vuejs/vuex/blob/01f87f0c3d59d0796a2535719dfa8328d1af390d/src/store-util.js#L89)

原来主要就是分别对 `mutation`、`action` 、`getter` 和子模块进行处理。

mutation 处理：将我们传入的 mutation 包装一下并挂载到 `store._mutations` 对象中

![image-20220724100441201](https://cdn.chosan.cn/posts-meta/image-20220724100441201.png)

[源码](https://github.com/vuejs/vuex/blob/01f87f0c3d59d0796a2535719dfa8328d1af390d/src/store-util.js#L222)

action 处理：将我们传入的 action 简单包装一下挂载到 `store._actions` 中，包装函数内将返回值转换成了 `promise` 对象

![image-20220724100757026](https://cdn.chosan.cn/posts-meta/image-20220724100757026.png)

getter 处理：将我们传入的 getter 挂载到 `store._wrappedGetters` 中，并且向 getter 传递了 4 个参数：当前模块的 `state`、当前模块的所有 `getter`，根模块的 `state` 和根模块的所有 `getter`

![image-20220724100944371](https://cdn.chosan.cn/posts-meta/image-20220724100944371.png)

这就是为什么我们可以在 `getter` 中访问模块的 `state` 和其它 `getter`，就像下面这样：

![image-20220724101404500](https://cdn.chosan.cn/posts-meta/image-20220724101404500.png)

子模块的处理就是简单的递归，这里就不再赘述了。

到目前，我们知道的信息如下：

- 整个模块的 `state` 存在于 `store._modules.root.state`
- 所有的 mutation 都放在 `store._mutations`，所有的 action 都放在 `store._actions`，所有的 `getter` 都放在 `store._wrappedGetters` 中，它们都只是做了简单的包装和一些科里化传参。

看完了 `installModule` 我们再来看看  `resetStoreState`：

![image-20220724102503439](https://cdn.chosan.cn/posts-meta/image-20220724102503439.png)

[源码](https://github.com/vuejs/vuex/blob/01f87f0c3d59d0796a2535719dfa8328d1af390d/src/store-util.js#L89)

这里的代码很简单，从上到下对应每个小框中的逻辑分别为：

- 初始化 `store.getters` 和一些其它参数
- 将上一步在 `installModule` 中处理后的 `store._wrappedGetters` 全部变成计算属性保存到 `store.getters` 中
- 将最开始挂载到 `store._modules.root.state` 中的模块原始 `state` 转换为响应式对象，并挂载到 `store._state.data` 上。





但一般我们使用 `store.state.xxx` 的方法来访问 `xxx` 属性，就像下面这样：

![image-20220724125420161](https://cdn.chosan.cn/posts-meta/image-20220724125420161.png)

因此 `store` 中还需要一个 `state` 的访问器属性，指向 `this._state.data`，看看源码：

![image-20220724125524236](https://cdn.chosan.cn/posts-meta/image-20220724125524236.png)

至此，整个 vuex 的初始化就完成了。整个流程简直不能太简单。



### commit

commit 所做的事情就是从 `store._mutations` 中取出对应的 `mutation`，使用传入的 `payload` 参数调用即可：

![image-20220724125826082](https://cdn.chosan.cn/posts-meta/image-20220724125826082.png)

[源码](https://github.com/vuejs/vuex/blob/01f87f0c3d59d0796a2535719dfa8328d1af390d/src/store.js#L101)

第一个红框中是从 `store._mutations` 获取到对应类型的 `mutation`，由于它是数组，因此第二个红框就是调用每个 `mutation` 即可。 

最后还通知了订阅者数据发生改变，不过我们并没有对数据进行监听，这部分也不属于核心逻辑，因此可以不用在意。

我们上面有看过注册 `mutation` 的逻辑，它仅仅是对我们传入的 mutation 进行包装，将 `this` 绑定到 `store`，然后传入 `state` 和 `commit` 中的 `payload` 参数：

![image-20220724100441201](https://cdn.chosan.cn/posts-meta/image-20220724100441201.png)

这就是 `commit` 的整个流程。

### dispatch

和 commit 一样，`dispatch` 的逻辑则是从 `store._actions` 中取出对应的 `action` 然后调用即可。这里就不在赘述，如果感兴趣可以参考源码[这里](https://github.com/vuejs/vuex/blob/01f87f0c3d59d0796a2535719dfa8328d1af390d/src/store.js#L138)。

### mapGetters

看看官网对 mapGetters 的介绍：

![image-20220724123218420](https://cdn.chosan.cn/posts-meta/image-20220724123218420.png)

它的目的仅仅是将 store 中的 getter 映射到局部计算属性，接收一个数组或者对象，看看源码：

![image-20220724123601299](https://cdn.chosan.cn/posts-meta/image-20220724123601299.png)

[源码](https://github.com/vuejs/vuex/blob/01f87f0c3d59d0796a2535719dfa8328d1af390d/src/helpers.js#L72)

它创建了一个新的对象 `res`，并把每个需要映射的属性包装了一层函数，依旧是从 `store.getters` 中获取值。而由于返回的对象是解构到 vue 的 `computed` 中，因此包装的这个函数实际上就成了 `computed` 中每个属性对应的计算属性函数。它的功能相当于只是简化了我们手动来编写下面这样的代码：
```js
computed: {
  doneTodosCount() {
    return this.$store.getters['doneTodosCount'];
  }
}
```



### 总结

Vuex 的初始化流程包括：

- 根据传入 `createStore` 的参数对模块进行搜集整理，并获取模块的原始 `state`
- 将模块的 `mutation`、`action`、`getter` 分别进行简单包装并挂载到当前模块私有属性上
- 将包装后的 getter 转换为计算属性，挂载到 `store.getters` 中
- 将模块初始 `state` 通过 `reactive` 转换为响应式对象并挂载到 `store._state.data` 上

整个流程比较简单。

另外关于像 `commit` 、`dispatch` 这类的方法只是将原本经过简单包装并挂载在私有属性上的对应方法执行一遍。

关于像 `mapGetters`、`mapMutations` 这类的方法只是完成了简单的数据映射。

### 写在最后

本文源码版本是 `vuex@4.0.2`，是目前最新的版本，不过 Vue3 官方已经推荐使用 [pinia](https://pinia.vuejs.org/)，下次我们对 `pinia` 进行简单分析。

上面贯穿了 `vuex` 的整个初始化流程，然后针对性的对 `commit` 和 `mapGetters` 进行了分析，其它类似的逻辑就没有在进行赘述，如：`dispatch`、`mapMutations` 等，感兴趣的朋友可以自行查看源码。

希望您能和我一起，让看源码变成一件很轻松的事情。 

另外如果感兴趣可以参考我的另一篇文章：[Vue-Router 源码快览（for Vue3）](https://juejin.cn/post/7123431126672080926)
