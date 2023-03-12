## 设置从 vue devtool 打开编辑器中组件对应文件（推荐程度 ☆☆☆☆）

*本文属于个人在项目中完成的小优化点记录，其它小优化导航连接：[关于个人在 Vue 项目上的一些小优化记录](https://juejin.cn/post/7209192211975340088)*

vue devtool 支持在浏览器控制台中快速打开组件在编辑器中对应的文件，在开发或查找问题的过程中出奇地方便，如下面的 1、2 步骤：

![使用控制台打开编辑器对应文件的示例](https://cdn.chosan.cn/posts-meta/vue-project-optimization-1.png)

1. 选中对应的组件
2. 点击右上角的符号直接在编辑器中打开。

官方文档这里有说明：[点击这里查看相关文档](https://devtools.vuejs.org/guide/open-in-editor.html#webpack)

### 支持的编辑器列表

下面先罗列一下支持程度

| Value           | Editor                                                                 | Linux | Windows | OSX |
| --------------- | ---------------------------------------------------------------------- | :---: | :-----: | :-: |
| `appcode`       | [AppCode](https://www.jetbrains.com/objc/)                             |       |         |  ✓  |
| `atom`          | [Atom](https://atom.io/)                                               |   ✓   |    ✓    |  ✓  |
| `atom-beta`     | [Atom Beta](https://atom.io/beta)                                      |       |         |  ✓  |
| `brackets`      | [Brackets](http://brackets.io/)                                        |   ✓   |    ✓    |  ✓  |
| `clion`         | [Clion](https://www.jetbrains.com/clion/)                              |       |    ✓    |  ✓  |
| `code`          | [Visual Studio Code](https://code.visualstudio.com/)                   |   ✓   |    ✓    |  ✓  |
| `code-insiders` | [Visual Studio Code Insiders](https://code.visualstudio.com/insiders/) |   ✓   |    ✓    |  ✓  |
| `codium`        | [VSCodium](https://github.com/VSCodium/vscodium)                       |   ✓   |    ✓    |  ✓  |
| `emacs`         | [Emacs](https://www.gnu.org/software/emacs/)                           |   ✓   |         |     |
| `idea`          | [IDEA](https://www.jetbrains.com/idea/)                                |   ✓   |    ✓    |  ✓  |
| `notepad++`     | [Notepad++](https://notepad-plus-plus.org/download/v7.5.4.html)        |       |    ✓    |     |
| `pycharm`       | [PyCharm](https://www.jetbrains.com/pycharm/)                          |   ✓   |    ✓    |  ✓  |
| `phpstorm`      | [PhpStorm](https://www.jetbrains.com/phpstorm/)                        |   ✓   |    ✓    |  ✓  |
| `rubymine`      | [RubyMine](https://www.jetbrains.com/ruby/)                            |   ✓   |    ✓    |  ✓  |
| `sublime`       | [Sublime Text](https://www.sublimetext.com/)                           |   ✓   |    ✓    |  ✓  |
| `vim`           | [Vim](http://www.vim.org/)                                             |   ✓   |         |     |
| `visualstudio`  | [Visual Studio](https://www.visualstudio.com/vs/)                      |       |         |  ✓  |
| `webstorm`      | [WebStorm](https://www.jetbrains.com/webstorm/)                        |   ✓   |    ✓    |  ✓  |

你也可以前往这里查看最新的支持情况：[点击查看支持的编辑器](https://github.com/vuejs/devtools/issues/821#issuecomment-851621967)

### 配置方式 1：通过环境变量配置

我们需要在 `env` 中指定 `EDITOR` 为我们的编辑器，例如 vscode 可以将 `EDITOR` 指定为 `code`。可以在 `.env.development` 中添加一行 `EDITOR = code` 或者在启动命令那里添加环境变量，推荐使用 `cross-env` 来添加。

### 配置方式 2：手动配置

下面介绍手动配置的情况如何处理：

这个功能需要用到 `launch-editor-middleware` 包，因此我们需要先安装它：`npm i -D launch-editor-middleware`

由于 vue-service 会在开发服务器中自动抢先注册一个路由 `/__open-in-editor`，但是它并没有传入对应的编辑器参数，会导致报错，我们必须抢在它前面注册，否则无法注册成功：

- 如果项目用的是 webpack4，则我们在 `devServer` 中加上配置： `setup: (app) => app.use('/__open-in-editor', require('launch-editor-middleware')('code'))`，如有有问题可以参考这个 [issue](https://github.com/vuejs/devtools/issues/821#issuecomment-851621967)
- 如果项目是 webpack5，则添加到 `onBeforeSetupMiddleware` 中：`onBeforeSetupMiddleware: (devServer) => devServer.app.use('/__open-in-editor', require('launch-editor-middleware')('code'))`

特别说明：在 webpack5 中，使用 `onBeforeSetupMiddleware` 可能会看到如下提示：

![使用onBeforeSetupMiddleware看到的废弃提示](https://cdn.chosan.cn/posts-meta/vue-proj-optimization-2.png)

大意是 `onBeforeSetupMiddleware` 已经被废弃，请使用 `setupMiddlewares` 来代替

但我们不得不用它，因为如果使用 `setupMiddlewares`，我们将无法抢先在 `@vue/cli-service` 前面注册该功能的路由，因为我们传入的参数被 `@vue/cli-service` 包装了一下，放在了它注册逻辑的后面:

![vue-cli-service代码](https://cdn.chosan.cn/posts-meta/vue-proj-optimization.png)

我们可以看到，`@vue/cli-service` 在这里提前注册了 `/__open-in-editor`，然后才会执行我们的逻辑，导致我们无法注册成功。因此如果是手动配置的情况下，我们必须使用 [devServer.onBeforeSetupMiddleware](https://webpack.js.org/configuration/dev-server/#devserveronbeforesetupmiddleware)
