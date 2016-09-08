## 我是谁？
说 SW 之前，我们先了解一下 Web worker，[MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/Web_Workers_API/Using_web_workers) 上对 Web worker 是这样定义的：

> _Web Workers provide a simple means for web content to run scripts in background threads. The worker thread can perform tasks without interfering with the user interface. In addition, they can perform I/O using XMLHttpRequest (although the responseXML and channel attributes are always null). Once created, a worker can send messages to the JavaScript code that created it by posting messages to an event handler specified by that code (and vice versa.)_

简单来说，Web worker 为我们提供了一种方式，可以在浏览器后台线程运行 javascript 脚本来为它的“主人”提供服务。当然，为了维护世界和平，服务内容是需要有限制的：

* 运行脚本必须同源
* 不能操作 DOM
* 消息传递必须是 [JSON 消息](https://developer.mozilla.org/en-US/docs/Glossary/JSON)

今天的主角 SW 是 Web worker 的一种，所以，现在知道我是谁了么？我是一个可以运行在浏览器后台的一个脚本。

## 从哪里来？
我从哪里来？我来之前的世界是什么样的？在我们普通人印象中，Web App 一定是在线运行的，离线运行？开什么玩笑，即使是全宇宙最牛逼的 Web App，如果你加载不了它，那也是废废的。然而，永远不要低估一颗有梦想的心，我们已经很努力的来扩展 Web App 的离线能力，已经有很多种技术尝试，来解决这一问题，在 SW 提出之前，另外一个提供开发者离线体验的 API 叫做 App Cache。然而 App Cache 有些局限性，在使用的时候，它会给你预先规定一些条条框框的规则，如果你不遵守哪怕某一点规则，那你的 App 中就可能遇到各种问题，还例如它可以很容易地解决单页应用的问题，但是在多页应用上会很麻烦。从而，为了维护正义...

SW 出现了，SW 的语法可能会比 App Cache 会复杂一些，但是你可以更细颗粒度的去处理缓存资源，有了它，你可以解决目前离线应用的问题，同时也可以做更多的事情，下面来看。

## 到哪里去？
我将要解决什么问题？首先说，SW 本质上作为一个代理服务器坐落于 Web 应用和浏览器与网络之间，代理可以做什么？可以控制你的所有请求，所以说，SW 除了离线能力，SW 一定还可以做很多事情，下面是来自 [MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/Service_Worker_API) 的介绍：

* 后台数据同步
* 响应其它源的资源请求
* 集中获取不易计算的数据的更新，比如 地理位置 和 陀螺仪信息，这样多个页面就可以利用同一组数据了
* 在客户端进行 CoffeeScript, less, CJS/AMD 模块等 的编译和依赖管理
* 为监控后台服务提供钩子
* 针对特定 URL 的个性化模板
* 增强性能，比如预取用户将来可能会需要的资源，比如一个相册中的几张新图片
* 后台同步：启动一个 SW 即使没有用户访问特定站点，这样就可以更新缓存
* 消息推送

这只是列举了一些例子，它可以实现的功能还很多，需要我们发挥聪明才智结合项目考虑。

## 总结
了解了以上三大哲学问题后，还需要提示的是，SW 目前还是一个实验中的功能，所以目前大部分版本浏览器还不支持，具体参考[ 这里 ](https://jakearchibald.github.io/isserviceworkerready/)。古人云：“站在风口上 猪都能飞起来”，所以说目前正是我们研究 SW 的好时机，下面我们就来真正的走进 SW，如果感兴趣的话，请持续关注。