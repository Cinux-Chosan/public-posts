# [http-proxy](https://github.com/http-party/node-http-proxy) 模块原理解析

曾几何时，一直以为 http-proxy 这样的模块内部做了大量的操作来完成请求代理。

也一直以为，它不是通过新建代理请求并转发收到的数据这种我认为没那么高大上的方式来进行代理的，不过最近有一些特殊情况需要自己实现一套代理来绕过 cloudflare 的指纹识别以及其它用途，于是看了一下 http-proxy 的源码，豁然开朗 ... 一切竟然这么简单！

## 基础使用

首先我们来回顾一下 `http-proxy` 的基本用法：

```js
const httpProxy = require("http-proxy");

httpProxy
  .createServer({
    target: "http://localhost:9003",
  })
  .listen(8003);
```

上面的代码通过 `http-proxy` 直接新建了一个 http Server 并且监听了 8003 端口，所有走到 8003 端口的请求都会被转发到 target 指定的地址，即 `http://localhost:9003`

我们也可以通过手动代理的方式来实现，手动的方式可以加入自己的逻辑在里面：

```js
const httpProxy = require("http-proxy");

const proxy = new httpProxy.createProxyServer();

http
  .createServer(function (req, res) {
    if (someCondition) { // 决定是否需要代理，以及可以通过 target 参数修改代理目标等
      proxy.web(req, res, {
        target: "http://localhost:9002",
      });
    }
  })
  .listen(8003);
```

上面这种用法使用我们自己创建的 HTTP Server 并监听 8003 端口，然后在请求中通过 `proxy.web` 将请求代理到 `http://localhost:9002`。

更多用法我们可以参考【[http-proxy 文档](https://github.com/http-party/node-http-proxy#readme)】

## 原理

_我们默认分别使用 req 和 res 来代表直接和客户端交互的请求和响应对象，proxyReq 和 proxyRes 分别代表和目标服务器交互的请求和响应对象。_

`http-proxy` 的实现原理也非常简单，不管是上面的第一种方式还是第二种方式，它都是新发起一个请求，然后从 req 获取数据并转发给目标服务器，当收到响应后，又将数据通过 res 返回给客户端。实际上第一种方式只是第二种方式在特定情况下的简写，内部依然是通过第二种方式来实现的。

用最精简的代码来表示如下：

- 转发从客户端收到的数据到目标服务器：

```js
const proxyReq = http.request({ target });
req.pipe(proxyReq);
```

- 转发目标服务器返回的数据到客户端：

```js
proxyReq.on("response", (proxyRes) => proxyRes.pipe(res));
```

但在真正转发数据之前，`http-proxy` 会对收到的请求和响应进行一系列预处理，如果你使用过 [axios](https://github.com/axios/axios)，那这个预处理的流程就类似于 [Interceptor](https://github.com/axios/axios#interceptors)，但在 `http-proxy` 中被称作 `pass`

## 源码

源码的目录结构如下

![image-20230309141413229](https://cdn.chosan.cn/posts-meta/image-20230309141413229.png)

其中所有的 `pass` 都被放在了 `passes` 目录中，`web-incoming.js` 中包含的是在收到客户端请求之后但在转发到目标服务器之前的预处理函数，`web-outgoing.js` 中包含的是在收到目标服务器响应后但在转发给客户端之前对数据进行预处理的函数。

以本文[基础使用](#基础使用)部分的第二个示例展开，当我们调用 `proxy.web(req, res, { target: "http://localhost:9002" }); ` 之后，`http-proxy` 在函数内部做了参数标准化处理，然后执行到下面这段代码：

![image-20230309142105424](https://cdn.chosan.cn/posts-meta/image-20230309142105424.png)

上面这段代码是对客户端发过来的请求进行预处理，其中 passes 是 `web-incoming.js` 中导出的函数通过 `Object.keys` 得到的集合，每个函数都会对 `req` 进行处理。

### `web-incoming.js`

在 `web-incoming.js` 中的 `pass` 有 4 个，分别是：

![image-20230309142446660](https://cdn.chosan.cn/posts-meta/image-20230309142446660.png)

除了 `XHeaders` ，其它三个都会被应用到所有的请求中。*代码较短的我会贴出来，较多的我就贴图了。*

- `deleteLength`：当请求是 `DELETE` 或者`OPTIONS` ，并且 `header` 中没有 `content-length` 时则将 `content-length`设置为 0，删掉 `transfer-encoding`

```js
  deleteLength: function deleteLength(req, res, options) {
    if((req.method === 'DELETE' || req.method === 'OPTIONS')
       && !req.headers['content-length']) {
      req.headers['content-length'] = '0';
      delete req.headers['transfer-encoding'];
    }
  }
```

- `timeout`：如果参数中设置了 `timeout` 则将其应用到客户端请求对象 `req` 中，在指定时间未完成就超时。

```js
  timeout: function timeout(req, res, options) {
    if(options.timeout) {
      req.socket.setTimeout(options.timeout);
    }
  }
```

- `XHeaders`：对 `header` 中的 `x-forwarded-for`、`x-forwarded-port`、`x-forwarded-proto`以及`x-forwarded-host` 进行预处理

```js
  XHeaders: function XHeaders(req, res, options) {
    if(!options.xfwd) return;

    var encrypted = req.isSpdy || common.hasEncryptedConnection(req);
    var values = {
      for  : req.connection.remoteAddress || req.socket.remoteAddress,
      port : common.getPort(req),
      proto: encrypted ? 'https' : 'http'
    };

    ['for', 'port', 'proto'].forEach(function(header) {
      req.headers['x-forwarded-' + header] =
        (req.headers['x-forwarded-' + header] || '') +
        (req.headers['x-forwarded-' + header] ? ',' : '') +
        values[header];
    });

    req.headers['x-forwarded-host'] = req.headers['x-forwarded-host'] || req.headers['host'] || '';
  }
```



- `stream`：这个函数代码较多，主要逻辑在下图中标注了出来

![image-20230309145757162](https://cdn.chosan.cn/posts-meta/image-20230309145757162.png)

其中 `stream` 核心逻辑包含 4 步骤：

1. 新建指向目标服务器的请求，称为代理请求

2. 通过代理请求转发客户端的数据到目标服务器

3. 监听代理请求的 `response` 事件，得到代理响应对象，并通过 `web-outgoing.js` 中的函数对响应进行预处理

4. 转发目标服务器返回的数据到客户端

### `web-outgoing.js`

在 `web-outgoing.js` 中，包含以下几个函数：

![image-20230309150932289](https://cdn.chosan.cn/posts-meta/image-20230309150932289.png)

- `removeChunked`：如果是 HTTP 1.0 则删除头部的 `transfer-encoding` 字段

```js
  removeChunked: function removeChunked(req, res, proxyRes) {
    if (req.httpVersion === '1.0') {
      delete proxyRes.headers['transfer-encoding'];
    }
  }
```

- `setConnection`：处理代理响应 `headers` 中的 `connection` 字段

```js
  setConnection: function setConnection(req, res, proxyRes) {
    if (req.httpVersion === '1.0') {
      proxyRes.headers.connection = req.headers.connection || 'close';
    } else if (req.httpVersion !== '2.0' && !proxyRes.headers.connection) {
      proxyRes.headers.connection = req.headers.connection || 'keep-alive';
    }
  }
```

- `setRedirectHostRewrite`：对重定向响应头中的 `location` 字段进行处理，如果响应中重定向到相同 `host`，则将 `location` 指向的 `host` 修改成我们指定的 `host`，另外如果参数中包含了 `protocolRewrite` 字段，还会修改 `location` 指向地址的协议。整个函数的功能相当于将目标服务器的重定向应用到我们自己的接口中，但将域名和协议这些修改成了我们自己的，客户端收到重定向之后也会重定向到我们自己的接口中。

```js
  setRedirectHostRewrite: function setRedirectHostRewrite(req, res, proxyRes, options) {
    if ((options.hostRewrite || options.autoRewrite || options.protocolRewrite)
        && proxyRes.headers['location']
        && redirectRegex.test(proxyRes.statusCode)) {
      var target = url.parse(options.target);
      var u = url.parse(proxyRes.headers['location']);

      // make sure the redirected host matches the target host before rewriting
      if (target.host != u.host) {
        return;
      }

      if (options.hostRewrite) {
        u.host = options.hostRewrite;
      } else if (options.autoRewrite) {
        u.host = req.headers['host'];
      }
      if (options.protocolRewrite) {
        u.protocol = options.protocolRewrite;
      }

      proxyRes.headers['location'] = u.format();
    }
  }
```

- `writeHeaders`：将目标服务器响应的 `header` 设置到客户端响应对象 `res` 中，其中可能会对 `cookie` 进行修改，使 `cookie` 对我们自己的代理域名生效。

![image-20230309152906345](https://cdn.chosan.cn/posts-meta/image-20230309152906345.png)

- `writeStatusCode`：将目标服务器返回的状态码设置回客户端响应对象 res 中。

```js
  writeStatusCode: function writeStatusCode(req, res, proxyRes) {
    // From Node.js docs: response.writeHead(statusCode[, statusMessage][, headers])
    if(proxyRes.statusMessage) {
      res.statusCode = proxyRes.statusCode;
      res.statusMessage = proxyRes.statusMessage;
    } else {
      res.statusCode = proxyRes.statusCode;
    }
  }
```



## 整体流程

整体流程分为请求阶段和响应阶段：

请求阶段：

`收到客户端请求 req` -> `httpProxy 新建一个请求 proxyReq` -> `将客户端请求 req 中的 header 做预处理并传递给 proxyReq` -> `将 req 中的数据通过 proxyReq 转发给目标服务器`

响应阶段：

`httpProxy 收到目标服务器的响应 proxyRes` -> `对 proxyRes 中 header 做预处理并传递给 res，同时在 res 上设置相同的请求状态信息` -> `将 proxyRes 中的数据转发给 res 从而到达客户端完成代理`  

## 总结

`http-proxy` 的代理思路很简单，本文简单解读了一下其关于 http 请求代理的部分，另外它也能够代理 `websocket`，但 `websocket` 相关的部分并没出现在本文中。

也许是职业对代码的敏感性，我感觉有些地方并不太妥当，例如：在对请求做预处理的时候，通过 `Object.keys` 来获取预处理函数集合是不太靠谱的，因为在请求的预处理集合中，`stream` 肯定是需要放到其它预处理函数之后被执行的，虽然它被放在了 `web-incoming.js` 模块的最后，但严格来说 `Object.keys` 并不一定能保证返回顺序，在某种程度上这依赖于具体的 JavaScript 引擎实现，而且依赖这种 "隐式" 顺序总给人一种不太可靠的感觉。但对于 `http-proxy` 来说，也许作者明确知道 V8 中的 `Object.keys` 实现一定是按序的呢？另外人家拥有丰富的测试用例，是否说明是我多虑了呢？

哈哈，这一点我不太敢保证，因为 Node.js 的官方文档在[这里](https://nodejs.org/en/docs/guides/abi-stability/#n-api)有说明，将来可能会包含不同的 JavaScript 引擎。

![image-20230309173607945](https://cdn.chosan.cn/posts-meta/image-20230309173607945.png)

总的来说，`http-proxy` 是一个易用且可靠的代理模块，并且到目前为止快 10 年了依然稳健，像 [Webpack](https://webpack.js.org/configuration/dev-server/#devserverproxy)、[Vite](https://vitejs.dev/config/server-options.html#server-proxy) 这种有着广大前端用户的产品，其 `devServer` 中的 `proxy` 底层依然是使用 `http-proxy` 来实现的。













































