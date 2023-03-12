大家好，我是 chosan

今天我们来看看大菠萝 [Pinia](https://pinia.vuejs.org) 的源码。

还是老样子，首先从官方找个简单的 demo 看看基本用法。

## 基本用法

首先需要通过 `createPinia` 创建一个 pinia 实例并传给 Vue 的实例：

![image-20220724194332178](https://cdn.chosan.cn/posts-meta/image-20220724194332178.png)

接下来通过 `defineStore` 定义一个 store，其中包含了 `state`、`actions` 以及未列出来的 `getters` 等，如下图的 `useCounterStore`：

![image-20220724194122497](https://cdn.chosan.cn/posts-meta/image-20220724194122497.png)

但是第二个参数可以是一个函数，类似于 Vue3 的 Composition API：

![image-20220725083643003](https://cdn.chosan.cn/posts-meta/image-20220725083643003.png)

第一种定义方式可以用类似 `mapActions` 这样的方式来解构到 Vue Optional API 中。不过第二种或许更加简介，同时也更具有封装性。

然后在组件中通过 `useCounterStore` 拿到对应的 store 实例，直接通过实例调用方法或属性进行使用：

![image-20220724194227971](https://cdn.chosan.cn/posts-meta/image-20220724194227971.png)



这就是 pinia 一个最简单的例子，那么的朴实无华 ……

但作为看源码的我们，必须抓重点，这里面核心就涉及到两个 pinia 的函数，分别是 `createPinia` 和 `defineStore`，我本来是想通过 「假设」的方式跳过 `createPinia` 的源码，但是还是看了一眼，发现真的没有几行，就先拿出来讲了。

## createPinia

![image-20220724195412850](https://cdn.chosan.cn/posts-meta/image-20220724195412850.png)

`createPinia` 最核心的代码就是红框中的部分，通过返回一个带有 `install` 方法的对象来注册到 Vue 实例中，其中 `install` 会被 Vue 实例自动调用，在 `install` 执行过程中会通过 `app.provide(piniaSymbol, pinia)` 向后代注入 `pinia` 对象，后代可以通过 `inject(piniaSymbol)` 获取到注入的这个 `pinia` 对象，另外它还将 `pinia` 挂载到了全局的 `$pinia` 属性上。

另外 `pinia` 对象上还有一个 `state` 属性，它其实是一个 `ref` 包装的响应式对象，这是它的声明：

![image-20220725111446780](https://cdn.chosan.cn/posts-meta/image-20220725111446780.png)

通过搜索 `pinia.state.value` 我们可以发现，它是用来挂载每个 store 对应的 state 对象。

总结起来，`createPinia` 主要做了：

- 向后代注入 `pinia` 对象，名称为 `piniaSymbol`，并将 `pinia` 对象挂载到 vue 全局属性中
- 创建一个响应式对象 `state` 用于后续保存每个 `store` 的 `state`
- 搜集所有的插件（plugin）存放到 `pinia._p` 中

本身代码量不大，就不再过多赘述了，直接朝  `defineStore` 出发。

## defineStore

`defineStore` 的用法上面已经给出了官方 demo 中的例子，它的作用就是返回一个能获取到 `store` 对象的函数。

![image-20220724222547631](https://cdn.chosan.cn/posts-meta/image-20220724222547631.png)

第一个红框中只是对传入的参数进行标准化，将传入的参数对象赋值给变量 `options`，而 `defineStore` 中的逻辑主要在返回的 `useStore` 函数中。

下面是 `useStore` 的代码：

![image-20220724223751816](https://cdn.chosan.cn/posts-meta/image-20220724223751816.png)

它的逻辑大致可以分为三部分：

- 通过 `inject` 获取到之前在 `createPinia` 注入的 `pinia` 对象实例
- 如果不存在当前 `store` 对象，则创建 `store` 实例并保存到 `pinia._s` 中，创建规则如下：
  - 如果传递给 `defineStore` 的第二个参数是一个函数则调用 `createSetupStore` 创建
  - 否则调用 `createOptionsStore`
- 从 `pinia._s` 中获取到对应的 `store` 并返回

总结起来，`useStore` 的功能就是找到并返回 `store` 实例，如果不存在则创建。

我们的 demo 中传入的第二个参数是一个包含 `state` 和 `actions` 的配置对象，因此进入到 `createOptionsStore` 逻辑中：

![image-20220724224912650](https://cdn.chosan.cn/posts-meta/image-20220724224912650.png)

`createOptionsStore` 中包装了一个 `setup` 函数，然后将 `setup` 函数传递给  `createSetupStore` 来创建 `store` 并返回，看看 `setup` 做了什么：

![image-20220724234317988](https://cdn.chosan.cn/posts-meta/image-20220724234317988.png)

`setup` 做了三件事，分别与上面三个框中的逻辑相对应：

- 首次进来，调用 `state()` 获得初始 state 数据，然后挂载到 `pinia.state.value` 中
- 通过 Vue 提供的 `toRefs` 函数将刚刚挂载的 state 数据转换为响应式。
- 遍历传入的 `getters` ，将它们包装为计算属性，其内部则是直接调用对应的 getter 来获取值，`getter` 的 `this` 和参数都是 `store` 实例

接下来是调用 `createSetupStore` 创建 store 实例，由于 `createSetupStore` 代码量较大，我们将它单独拿出来讲。

#### createSetupStore

![image-20220725092340907](https://cdn.chosan.cn/posts-meta/image-20220725092340907.png)

我们主要了解前 4 个函数参数，分别为：

- `$id`：我们调用 `defineStore` 时的第一个参数，`store` 的名称
- `setup`：返回初始 `store` 数据的函数，如果我们使用前面 demo 中的第二种 `defineStore` 的方式则该参数就是第二个函数参数，如果采用第一种方式则会被 `createOptionsStore` 包装，上面已经介绍过 `createOptionsStore`
- `options`：我们传入的配置对象
- `pinia`：pinia 实例

createSetupStore 顾名思义，就是要通过传入的 setup 函数返回值来创建最终的 store 对象。这个函数的代码比较长，但它主要完成了这几件事：

- 创建一个响应式对象 `store`
- 给 `store` 添加一系列的操作方法，如：`$patch`、`$reset`、`$subscribe`、`$dispose` 以及 `$onAction`
- 执行 `setup` 函数获取返回值，源码里面称它为 `setupStore`
- 对 `setupStore` 中的 `action` 进行包装，使其能够处理订阅消息
- 将 `setupStore` 中的属性赋值给 `store` 对象，另外给 `store` 对象添加了 `$state` 属性
- 在当前 `store` 的基础上执行 `pinia._p` 中的所有插件
- 返回 `store` 对象

所有的操作都是围绕新创建的 `store` 对象，分别来看看：

##### 创建一个响应式对象 `store`

![image-20220725092304719](https://cdn.chosan.cn/posts-meta/image-20220725092304719.png)

 源码中创建 store 对象分为三步（合并了添加操作方法这一步）：

- 创建一个基本的 `partialStore` 对象，它主要绑定了 store 的各种操作方法（暂不关心具体实现）
- 使用 Vue3 `reactive` API 基于 `partialStore` 创建一个响应式的 `store` 对象
- 将 `store` 挂载到 `pinia._s` 中

##### 执行 `setup` 函数获取返回值，源码里面称它为 `setupStore`

![image-20220725093254807](/Users/chosan/Library/Application Support/typora-user-images/image-20220725093254807.png)

##### 对 `setupStore` 中的 `action` 进行包装，使其能够处理订阅消息

![image-20220725093646190](/Users/chosan/Library/Application Support/typora-user-images/image-20220725093646190.png)

这里包含两部分逻辑，简单介绍一下：

对通过 `setup` 函数返回的对象进行遍历，如果它是 `reactive` 或者 `ref` 包装后的对象但不是计算属性，且是以函数的方式创建的 store（即调用 `defineStore` 时第二个参数是函数，否则已经被 `createOptionsStore` 处理过，无需再处理），则将它存到 `pinia.state.value` 中对应的 store 对象里面。
