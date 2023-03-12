# 关于子项目 babel 配置不生效的问题排查记录

![img](https://cdn.chosan.cn/posts-meta/640-20221127155612970-20221127155702162.png)

（最近世界杯加疫情，只能窝在家写写文章，记录记录生活...）

刚加入一家新的公司那会儿（大概两三个月之前），被安排负责一个名字听起来就比较古老但非常重要的的项目 —— **CRM**

刚接触项目准备开发时，发现该项目并不支持一些新的 ES 语法，尤其是 `optional-chaining` 的方式 `?.`，写 `a && a[0] && a[0].xxx` 的方式着实让人难受，于是打算把这些特性加入到项目中来。

项目是 Vue2，本身是通过 Vue CLI 创建的，是支持基本的 `babel` 配置。但今天这个问题并不只是单纯的安装 babel 依赖然后加到 `.babelrc` 里面去那么简单。

过程是这样的，项目结构比较特殊，本身项目根目录叫 `CRM`，在 `src/view` 中有关于工作台相关的代码，假设其目录叫 `workbench`，但是由于悠久的历史，经过了多为前辈之手，workbench 被单独拆分了出来，拥有独立的目录结构和配置文件，它的代码是在编译期间自动从 CRM `src/view/workbench` 中拷贝过来，并且单独打包，其结果会生成到 CRM 的 `public` 目录中，`workbench` 中大多数配置都直接继承自 `CRM`，但 `package.json` 中却没有一个依赖，这应该是利用了 `webpack` 底层模块 `enhanced-resolve` 会模拟 node 寻找模块的方式自动向上查找的特性，因此得以释放双手不用再花时间去处理 `workbench` 中的 ` package.json` 

**但问题恰恰就出现在这里！**且听我细细道来 ...

项目结构大概是这样的：


![image-20221127135502597](https://cdn.chosan.cn/posts-meta/image-20221127135502597.png)



## 问题出现

在 Vue Cli 创建的项目中，其 `babel` 是通过 `vue` 插件 `@vue/cli-plugin-babel` 来加载的。

项目的 `@vue/cli-plugin-babel` 版本为 `3.6.0`，本身版本较低，其依赖的 `@vue/babel-preset-app` 版本也比较低，间接导致 babel 的 `preset` 版本也不高，因此 `preset` 的解析不够智能（也就是无法自动根据 `browserslist` 的 `target` 推断出是否需要自动解析新的语法特性）。需要手动安装并且添加需要的 babel 插件。

最初我直接在 CRM 目录下安装了 `@babel/plugin-syntax-optional-chaining`，并且同步更新到了 CRM 和 workbench 目录的 `.babelrc` 中的 `plugins` 字段下，以为这样就可以稳稳的使用 `?.` 语法特性了。

CRM 没有问题，可以直接使用 `?.` 语法，但 `workbench` 项目启动期间依旧报 `unexpected token` 错误。

![image-20221127150851275](https://cdn.chosan.cn/posts-meta/image-20221127150851275.png)

奇怪就奇怪在这里，相同的配置，为什么外层的 `CRM` 可以，而作为子项目的 `workbench` 却不可以！

![奇怪的情况微信表情包_微信表情_微茶网](https://cdn.chosan.cn/posts-meta/images.jpeg)

查阅了百度、Google、官方文档和 stackoverflow 等依旧没有找到答案，毕竟像这种嵌套子项目的情况估计很少，只有自己动手了。



## 解决问题

要解决这个问题，就得看看在项目启动的过程中 `babel` 相关的地方都做了些啥。于是对 `vue` 打包过程进行断点调试，忽略具体细节...

通过对打包过程源码的调试，发现 `@vue/cli-service` 中有一个名叫 `Service.js` 文件会从项目目录下面读取 `package.json` 文件并拿到其中的依赖项，然后判断是否是 `vue` 对应的插件，这里梳理一下整个过程始末：

- 启动项目，执行 `@vue/cli-service/bin/vue-cli-service.js`，其中会创建 `Service` 实例

![image-20221127145915064](https://cdn.chosan.cn/posts-meta/image-20221127145915064.png)

- 在 `Service` 构造函数中会调用 `this.resolvePlugins` 初始化 `vue` 插件

![image-20221127150032705](https://cdn.chosan.cn/posts-meta/image-20221127150032705.png)

- 初始化过程中会读取项目中的 `package.json` 并合并所有依赖项，然后通过其名称判断是否是 `vue` 插件：

![image-20221127150223272](https://cdn.chosan.cn/posts-meta/image-20221127150223272.png)

`isPlugin` 对传入的字符串做检查，如果它是满足以 `@vue`、`vue-`和`@xxxvue-` 或者 `@xxx-vue-` 开头并且紧跟 `cli-plugin-` 作为名称开头的依赖项都会被当做是 `vue` 插件

![image-20221127150309168](https://cdn.chosan.cn/posts-meta/image-20221127150309168.png)

如果是 `vue` 插件，则会被 `resolve` 出去并加载，如果不是则会被忽略掉。



由于 `CRM` 中的 `package.json` 有整个项目完整的依赖信息，它获取到的内容如下（其中包括了与 `babel` 相关的几个 `vue` 插件）：

![image-20221127150808106](https://cdn.chosan.cn/posts-meta/image-20221127150808106.png)

而 `workbench` 中没有依赖，因此它是一个空对象，没有对应的插件：

![image-20221127150944504](https://cdn.chosan.cn/posts-meta/image-20221127150944504.png)

既然找到了原因，那就很好解决了，我将项目中几个 `vue service` 插件都加入到 `workbench` 的 `package.json` 中

![image-20221127151055490](https://cdn.chosan.cn/posts-meta/image-20221127151055490.png)



然后重新跑项目，报错消失，完美解决！

## 总结

一开始确实没有想到 `vue service` 启动的时候会去读取 `package.json` 中的依赖并进行解析，这种做法太过于隐晦，虽然感觉比较智能，但如果未在文档中出现相关说明，一旦在遇到此类问题时肯定会让人一头雾水，如果不排查源码是很难找出根本原因的。

另外我还发现项目中的另一个问题：**子项目 `workbench` 的 `babel` 其实一直没有生效！**根本原因就是 vue service 通过读取 `package.json` 的方式来加载对应的插件，但是子项目的 `package.json` 并不存在对应的依赖项。这一点其实是很危险的！很有可能由于开发环境用的浏览器版本较高未发现异常，而线上用户浏览器版本较低就出现页面白屏等异常，这种情况将很容易导致严重的线上事故！

又解决了个**大**问题，大快人心！不过，总感觉这个项目中嵌套子项目的做法不太好，不过我已经有了更好的优化思路 ...

![image-20221127153809130](https://cdn.chosan.cn/posts-meta/image-20221127153809130.png)
