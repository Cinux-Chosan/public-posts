#  Vue-Router 源码快览

大家好，我是 chosan

相信很多朋友和我一样，虽然知道几种路由模式的实现方式，但是如果不看看框架里面的代码，捋一捋它的基本逻辑，心里就总觉得没底，被面试问到还可能有点慌 —— 万一它不是这么实现的呢？

如果答错了，面试官的表情肯定是这样的：

![image-20220723115726722](https://cdn.chosan.cn/posts-meta/image-20220723115726722.png)

所以今天花了点时间看了一下 Vue Router 的源码，不求甚解，只希望花尽可能少的时间了解一下主线逻辑。

这里不会对常见的路由模式做具体深入的讲解，如果您还不熟悉的话，可以看看社区里面的一些优秀文章，这里简单复习一下常见的路由模式：

- `history`：使用传统的 url `path` 部分（域名后面的以 `/` 开始的路径）作为路由，通过 `pushState` 和 `replaceState` 方法修改状态，通过拦截和监听 `popState` 事件监听路由变化。
- `hash`：使用 `url` 中 `segment` 部分（ `#` 后面的部分）作为路由，通过 `onhashchange` 事件来监听路由变化。
- `memory`：和 `url` 无关，在内存中保存对应的路由信息。



看源码最好的方式就是先熟悉一下官方文档，了解其基本用法，而文档中的入门一节又是整个库最精简最核心的部分，于是我从[官方 git](https://github.com/vuejs/router) 拉下来代码并安装好依赖之后，我自然的打开了 Vue-Router 的官方文档，进入到「[入门](https://router.vuejs.org/zh/guide/)」一节，第一眼看到的就是下面这个简单的例子：

![image-20220723101434397](https://cdn.chosan.cn/posts-meta/image-20220723101434397.png)



还是那熟悉的配方，`router-link` 负责页面的路由跳转，而 `router-view` 负责渲染对应的路由组件。我们要看 Vue-Router 的源码，无非就是想看看 `router-view` 是怎么把组件渲染到页面中的。

话不多说，我凭直觉直接找到源码里面的 `RouterView.ts`，扫了一眼，发现这行代码：

```ts
export const RouterView = RouterViewImpl
```

导出的 RouterView 真实的实现是 `RouterViewImpl`，那直接看 `RouterViewImpl` 实现就行了，`CTRL + 左键` 定位到具体实现，发现这是个 `Vue3` 组件，里面有个 `setup` 方法，无疑它返回的函数就是用于页面渲染的 Render 函数，直接看看 Render 函数返回了什么：

![image-20220723102427315](https://cdn.chosan.cn/posts-meta/image-20220723102427315.png)

[源码](https://github.com/vuejs/router/blob/2ed41a1cd3755a7e1ea2e21211abf5778f1b92c8/packages/router/src/RouterView.ts#L198)

这是 Render 函数返回的内容，连蒙带猜，component 肯定就是`router-view`最终要渲染出来的组件了，再定位看看 `component` 是什么，继续 `CTRL + 左键`：

![image-20220723102633701](https://cdn.chosan.cn/posts-meta/image-20220723102633701.png)

[源码](https://github.com/vuejs/router/blob/2ed41a1cd3755a7e1ea2e21211abf5778f1b92c8/packages/router/src/RouterView.ts#L167)

原来是通过 `h` 创建的 VNode，由于 `h` 方法第一个参数就是组件本身，那这里很明显 `component` 就是 `ViewComponent` 的 VNode，我们不用关心什么乱七八糟的 VNode，只要知道最后渲染的是 `ViewComponent` 就 OK 了，继续看 `ViewComponent` 是什么，不用我说吧，`CTRL + 左键` 走起：

![image-20220723103002902](https://cdn.chosan.cn/posts-meta/image-20220723103002902.png)

[源码](https://github.com/vuejs/router/blob/2ed41a1cd3755a7e1ea2e21211abf5778f1b92c8/packages/router/src/RouterView.ts#L143)

发现 `ViewComponent` 就是从 `matchedRoute.components` 里面根据名字取出来的一个组件，而 `matchedRoute = matchedRouteRef.value`，直接看 `matchedRouteRef`：

![image-20220723103417374](https://cdn.chosan.cn/posts-meta/image-20220723103417374.png)

[源码](https://github.com/vuejs/router/blob/2ed41a1cd3755a7e1ea2e21211abf5778f1b92c8/packages/router/src/RouterView.ts#L82)

它是个计算属性，从 `routeToDisplay` 上面去取的值，继续看看 `routeToDisplay`：

![image-20220723103534204](https://cdn.chosan.cn/posts-meta/image-20220723103534204.png)

[源码](https://github.com/vuejs/router/blob/2ed41a1cd3755a7e1ea2e21211abf5778f1b92c8/packages/router/src/RouterView.ts#L63)

`routeToDisplay` 又是一个计算属性，从 `props.router` 和 `injectedRoute.value` 上面取值，而前面使用 `router-view` 并没有提供 `route` 属性，那么我们直接就看 `injectedRoute` 是什么咯，不然还能怎样？

我还没用过 `inject` 方法，但是我对 Vue 里面的  `provide/inject` 还是有所了解，不用再去看官方文档，很明显，我猜这就是上层注入的一个值，提供给子组件使用的，和 React 里面的 `context` 如出一辙。

所以先盲猜这个 `injectedRoute` 就是上层提供的值，通过 `routerViewLocationKey` 来获取，很明显要看在哪里提供的这个值我们就必须看哪些地方用到了 `routerViewLocationKey`，定位看看：

![image-20220723104150489](https://cdn.chosan.cn/posts-meta/image-20220723104150489.png)

[源码](https://github.com/vuejs/router/blob/2ed41a1cd3755a7e1ea2e21211abf5778f1b92c8/packages/router/src/injectionSymbols.ts#L51)

原来它就是一个 `Symbol`，看看哪些地方用到了这个值呢：

![image-20220723104529904](https://cdn.chosan.cn/posts-meta/image-20220723104529904.png)

很幸运，没有很大一坨，而且每个文件里面最多也只有两三处用到了这个值，我大概查看了每个文件里面对这个值的使用情况，发现只有最后一个 `router.ts` 和我们的逻辑比较贴切，那就直接看 `router.ts` 对应的位置，点进去看看：

![image-20220723104714306](https://cdn.chosan.cn/posts-meta/image-20220723104714306.png)

[源码](https://github.com/vuejs/router/blob/2ed41a1cd3755a7e1ea2e21211abf5778f1b92c8/packages/router/src/router.ts#L1230)

果然，这里通过 `provide` 的方式向后代注入了一个 `routerViewLocationKey` ，其值为 `currentRoute`。

另外我还发现这段逻辑就是在整个插件的注册逻辑里面（也就是通过下图这个 `install` 方法，因为 Vue Plugin 就是通过`install`来注册），另外我还看到了 `router-link` 和 `router-view` 的注册逻辑，以及 Vue-Router 注入 `$route` 的地方：

![image-20220723104954394](https://cdn.chosan.cn/posts-meta/image-20220723104954394.png)

[源码](https://github.com/vuejs/router/blob/2ed41a1cd3755a7e1ea2e21211abf5778f1b92c8/packages/router/src/router.ts#L1190)

这简直就是意外收获，惊喜 ing ！！！😍

言归正传，继续看上面的 `currentRoute` 是怎么来的：

![image-20220723105336443](https://cdn.chosan.cn/posts-meta/image-20220723105336443.png)

[源码](https://github.com/vuejs/router/blob/2ed41a1cd3755a7e1ea2e21211abf5778f1b92c8/packages/router/src/router.ts#L376)

定位到这里，原来就是声明了一个 `shallowRef`，如果您不熟悉什么事 `shallowRef` 也没关系，其实我也没咋用过，知道 `ref` 看名字也大概知道 `shallowRef` 是什么意思，如果没用过 Vue 也不要紧，就把它当成是响应式数据，如果 `currentRoute` 发生了改变，依赖它的地方会被通知到。

看看哪些地方用到了 `currentRoute`：

![image-20220723105659921](https://cdn.chosan.cn/posts-meta/image-20220723105659921.png)

乍一看很多的样子，仔细看其实只有两处修改了 `currentRoute` 的值，很明显我们要看的是第一个红框里面的 `toLocation` 是怎么来的，定位过去看看：

![image-20220723105951625](https://cdn.chosan.cn/posts-meta/image-20220723105951625.png)

[源码](https://github.com/vuejs/router/blob/2ed41a1cd3755a7e1ea2e21211abf5778f1b92c8/packages/router/src/router.ts#L949)

定位 `toLocation`：

![image-20220723105835874](https://cdn.chosan.cn/posts-meta/image-20220723105835874.png)

[源码](https://github.com/vuejs/router/blob/2ed41a1cd3755a7e1ea2e21211abf5778f1b92c8/packages/router/src/router.ts#L916)

发现它是 `finalizeNavigation` 的参数，那就看哪里调用了 `finalizeNavigation` 就可以了。

![image-20220723110155098](https://cdn.chosan.cn/posts-meta/image-20220723110155098.png)

除开声明文件，也就只有两个地方，简单看了一下两个地方发现都差不多，第二个提供的信息更多，我们以第二个为准：

![image-20220723110606903](https://cdn.chosan.cn/posts-meta/image-20220723110606903.png)

[源码](https://github.com/vuejs/router/blob/2ed41a1cd3755a7e1ea2e21211abf5778f1b92c8/packages/router/src/router.ts#L1044)

发现是在 `promise.then` 中调用的，而 `promise` 是 `navigate` 方法返回的：

![image-20220723110652847](https://cdn.chosan.cn/posts-meta/image-20220723110652847.png)

[源码](https://github.com/vuejs/router/blob/2ed41a1cd3755a7e1ea2e21211abf5778f1b92c8/packages/router/src/router.ts#L988)

先不看 `navigate` 是怎么实现的，不过从名字看它肯定是路由导航用的，而且既然是个 `promise` 而且是在 `then` 中调用 `finalizeNavigation`，那我大可以先猜测 `navigate` 是先修改 Vue-Router 的状态，而 `then` 中的 `finalizeNavigation` 待会儿会消费这个状态。 

现在我们就是要看看 `toLocation` 是怎么来的了，定位看一下：

![image-20220723110440654](https://cdn.chosan.cn/posts-meta/image-20220723110440654.png)

[源码](https://github.com/vuejs/router/blob/2ed41a1cd3755a7e1ea2e21211abf5778f1b92c8/packages/router/src/router.ts#L960)

发现 `toLocation` 是 `resolve(to)` 的返回值，其类型是 `RouteLocationNormalized`，定位看看类型声明：

![image-20220723111347138](https://cdn.chosan.cn/posts-meta/image-20220723111347138.png)

[源码](https://github.com/vuejs/router/blob/2ed41a1cd3755a7e1ea2e21211abf5778f1b92c8/packages/router/src/types/index.ts#L165)

发现是在 `_RouteLocationBase` 的基础上添加了 `matched` 属性，`matched` 是一个数组，元素类型是 `RouteRecordNormalized`，给大家看看它有哪些属性：

![image-20220723111833172](https://cdn.chosan.cn/posts-meta/image-20220723111833172.png)

注意：属性的实际类型不是 `number`，如果用实际类型会导致显示不下，截原图又太大，故这里只是简单看看 `RouteRecordNormalized` 包含哪些属性。

再看看 `_RouteLocationBase`，包含了 `name`、`path`、`params`、`meta` 以及自己声明的类型：

![image-20220723112151465](https://cdn.chosan.cn/posts-meta/image-20220723112151465.png)

[源码](https://github.com/vuejs/router/blob/2ed41a1cd3755a7e1ea2e21211abf5778f1b92c8/packages/router/src/types/index.ts#L139)

原来如此，`toLocation` 的类型继承自 `_RouteLocationBase`，里面包含了各种路由所需要的参数，并且自身还有一个 `matched` 属性包含了统一格式化后的路由信息。

另外我们还发现 `toLocation` 的获取和 `navigate` 的调用都是在 `routerHistory.listen((to, _from, info) => { })` 这行代码的回调函数中执行的，看到这个我们很容易联想到 `routerHistory.listen` 是监听路由变化的，并且它多半是个抽象方法，是 `history`、`hash` 以及其它路由模式的抽象。

#### 总结

因此，我们似乎已经把主线串起来了：

一开始，Vue-Router 通过导出对象的 `install` 方法被 Vue 调用从而注册了 `router-link` 和 `router-view` 两个组件，并且注入了全局和组件自身的 `$route` 属性，然后开始了它的初始化逻辑。

它通过 `routerHistory.listen` 调用具体的路由实现来监听路由变化，当路由变化时，回调函数被调用并能接受到 `from`、`to` 以及额外的一些参数，根据 `to` 我们可以拿到路由原始配置中具体指向的组件（该过程通过 `resolve(to)`来完成），而最终这些参数会赋值给 `currentRoute.value`，它里面包含了目标路由的所有必要参数，`router-view` 最终渲染的组件 `ViewComponent` 也是来自于它。



### 写在最后

我从开始到结束大概花了十几分钟的时间，因此并没有涉及到诸多细节，比如 `resolve` 是怎么根据 `to` 来查找和格式化数据的，`router-link` 的具体逻辑是什么等等。因为我写这篇文章的最主要目的是想告诉前端新同学两点个人经验总结，可以帮大家节省看源码的时间，提升效率：

- 阅读源码最好是从官方文档的 basic 用例入手（如果可以的话提前想一下这个功能自己会怎么去实现）
- 阅读源码要尽可能大胆的猜测，不要太追求细节，如果一次性非要一行一行的把整个项目从初始化到每种 case 都摸透就可能吃不消。死磕不是目的，循序渐进才是王道，每次花少量的时间厘清部分逻辑，否则步子太大容易扯着蛋。

蛋大的大佬请忽视以上两点！

现在已经厘清大概的主流程了，如果您对其中某部分逻辑比较好奇，比如 `routerHistory` 的实现，或者 `resolve(to)` 是怎么执行的，可以亲自尝试一下上面的方法，记住，每次花少量时间厘清一个功能点就算成功，另外如果发现文中有误欢迎指正，感谢！

好了，祝各位周末愉快 😁！