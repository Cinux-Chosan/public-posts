# 项目 Vue2.6 升级 2.7 记录

![img](https://cdn.chosan.cn/posts-meta/640.png)

最近受疫情影响一直居家办公，正好趁今天周末有点闲暇，打算把最近手里的项目从 vue 2.6 升级到 2.7，从而能够使用一些新特性（如 `Composition API`、`setup` 和各种 `hook`，具体可以参考官方 [CHANGELOG](https://github.com/vuejs/vue/blob/main/CHANGELOG.md#270-alpha1-2022-05-31)），也能减少对 `mixin` 的使用。


当前环境：

- `@vue/cli-service`: `3.6.0`
- `vue`: `2.6.10`

升级后：

- `@vue/cli-service`: `5.0.8`
- `vue`: `2.7.14`

我将首先升级 `@vue/cli-service`

## 升级 `@vue/cli-service`

参考官方的方案：[Migrate from v3](https://cli.vuejs.org/migrations/migrate-from-v3.html)

1. 执行 `npm install -g @vue/cli`
2. 执行 `vue upgrade` 更新依赖

第一步很顺利，但是在执行第二步的时候由于是老项目，在执行 `vue upgrade` 安装依赖的时候一直提示需要加上 `--legacy-peer-deps` 选项

![image-20221126171416645](https://cdn.chosan.cn/posts-meta/image-20221126171416645.png)

这个版本冲突是 npm 升级之后爆出来的，对于老项目需要加上 `--legacy-peer-deps` 参数来解决，但是翻了一下 cli 的官网没有找到可以添加该参数的相关说明，于是翻看了一下 `@vue/cli` 相关部分的源码，发现其实很多地方都有检测是否需要 `--legacy-peer-deps`

![image-20221126172326703](https://cdn.chosan.cn/posts-meta/image-20221126172326703.png)

![image-20221126172423952](https://cdn.chosan.cn/posts-meta/image-20221126172423952.png)



但唯独 `upgrade` 没有检测是否需要这个参数：

![image-20221126172523421](https://cdn.chosan.cn/posts-meta/image-20221126172523421.png)

于是手动给它加上：

![image-20221126171937858](https://cdn.chosan.cn/posts-meta/image-20221126171937858.png)

此时再执行 `vue upgrade` 就能够成功（`add` 方法是有检测 `needsPeerDepsFix`，但 `runCommand` 中的 `add` 并不是执行该方法，而是直接执行 `add` 命令）。



以上是自动升级方案，也可以参考官方的手动升级方案：[One-By-One Manual Migration](https://cli.vuejs.org/migrations/migrate-from-v3.html#one-by-one-manual-migration)

### 升级影响

此时 `@vue/cli-service` 从 `3.6.0` 升级到了 `5.0.8`，跨度有点大，内置的 `webpack` 也从 `4` 升级到了 `5`，因此需要对项目中的配置做相应调整。

#### webpack node polyfill

在 webpack 4 中默认会 polyfill node 模块，但是在 5 中已经被取消，因此可以自己配置 `resolve.fallback`，我采用了 [node-polyfill-webpack-plugin](https://github.com/Richienb/node-polyfill-webpack-plugin) 包，它自动 polyfill 了部分模块（具体使用方法和哪些模块可以参考其 github 说明）。但在跑项目的时候发现提示找不到 `fs` 模块，这是由于 `node-polyfill-webpack-plugin` 并没有处理 `fs` 模块，由于是前端项目，只是在一些第三方库中可能会有兼容 node 时的代码要用到 `fs`，因此配置了 `resolve.fallback` 来告诉 `webpack` 不 `polyfill` 该模块：

![image-20221126173801769](https://cdn.chosan.cn/posts-meta/image-20221126173801769.png)



#### css 使用 `:export` 导出变量

项目跑起来的时候发现首页的样式不对，通过分析发现是源代码中使用了 `sass` 变量并通过 `:export` 导出：

![image-20221126174123506](https://cdn.chosan.cn/posts-meta/image-20221126174123506.png)

在 webpack 4 中默认所有文件都是当做 `icss` 处理，但是在某些场景下会存在一些问题，因此 webpack 5 取消了该默认配置，需要自己配置。具体信息可以参考[这里](https://webpack.js.org/loaders/css-loader/#separating-interoperable-css-only-and-css-module-features)，因此我们还需要对它进行配置以便向后兼容，红框内是在 `vue.config.js` 中新增的配置，目的是将参数传递给 `css-loader`：

![image-20221126174658828](https://cdn.chosan.cn/posts-meta/image-20221126174658828.png)

#### 其它配置调整

关于 `webpack` 配置改动的地方还有一些不太通用的细节调整，在跑项目的时候可以根据 `webpack` 的报错提示进行修改，这里就不细说了。

## 升级 Vue

直接安装 `vue 2.7` 版本即可：`npm i vue@2.7`，另外在 `2.7` 中可以将 `package.json` 中的 `vue-template-compiler` 删除。

## 总结

本次升级基本比较顺利，不需要我们手动升级 `webpack` 及其各种 `plugin` 和 `loader` 依赖能够节省大部分的精力投入的项目的其它优化中去。如果后续出现其它升级导致的问题将在这里进行更新。

