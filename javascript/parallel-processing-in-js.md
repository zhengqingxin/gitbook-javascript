#JavaScript 中的并行处理
## 导语
>[这里查看原文](https://advancedweb.hu/2016/08/09/parallel-processing-in-js/) 
>
> 一直以来，JavaScript 都并没有多线程的能力，而单线程在让开发者方便的同时，也使它一直不能处理计算量复杂的场景，唯一方法就是让服务端去做。幸运的是，随着 Web Worker 的普及，我们终于可以在后台线程来处理资源密集型的计算了。而不好的方面是，目前制定的标准只适用于当前的生态系统，这有时候就比较尴尬了。如果你了解其他从一开始就支持多线程的语言的话，你可能会发现很多的限制，远非仅仅是实例化一个新线程，然后你操控这个实例就能实现多线程。这篇文章主要来介绍 Web Worker，包括什么时候使用，该怎么使用，它有什么奇怪的特性，会介绍在 Webpack 中如何使用它，还有可能遇到的一些坑。

##Web Workers
> 例子参见 [html](https://github.com/sashee/parallel-processing-in-js/blob/master/worker.html), [js](https://github.com/sashee/parallel-processing-in-js/blob/master/worker.js), [demo](https://sashee.github.io/parallel-processing-in-js/worker.html)

Web Worker 可能是在 JavaScript 中唯一可以真正实现多线程的方法了。我们需要按照下面的方式创建 worker ：

```
const worker = newWorker("worker.js");
```

上面就定义了一个 Worker 实例，然后你可以通过 postMessage 与 worker 通信，就像和 iFrame 通信一样，只不过不存在跨域的问题，不需要验证跨域。

```
worker.postMessage(num);
```

在 worker 代码中，你需要监听这些事件：

```
onmessage = (e) => {
	// e.data will contain the value passed
};
```

这种方式是双向的，所以你也可以从 worker 中 postMessage 给我们的主程序。

在 worker 代码中：

```
postMessage(result);
```

在主程序中：

```
worker.onmessage = (e) => {}
```

这就是 worker 最基本的用法。
##异常处理
在你的 worker 代码中，有很多种方式来处理异常，比如你可以 catch 之后通过 postMessage 传递，这样可能需要多些一些代码，但是确实最有效也最安全的。

另一种方式是用 onerror 事件，这种方式可以捕捉所有未处理的异常，并且交给调用方来决定如何处理。调用方式很简单：

```
worker.onerror = (e) => {};
```

为了调试方便，异常对象中还有一些额外的字段比如：filename，lineno，colno.
##回收
将不需要的 worker 回收是非常重要的，worker 会生成真正的操作系统线程，如果你发现与很多 worker 线程同时运行，你可以通过很简单的杀掉浏览器进程。

你有两种方式杀掉 worker 进程：在 worker 里和在 worker 外。我认为最好的处理 worker 生命周期的地方是在主页面里，但这也要取决于你代码的具体情况。

杀掉一个 worker 实例，在外部可以直接调用 `terminate()`方法，这种方法可以立即杀掉它，释放所有它正在使用的资源，如果它正在运行，也会立即终止。

如果你想要让 worker 自己去管理它的生命周期，可以直接在 worker 代码中调用`stop()`方法。

不管使用哪种方法，worker 都会停止，销毁所有资源。

如果你想使用一种“一次性”的 worker，比如需要做一些复杂运算之后就不再使用了，也要确保在 onerror 事件中去销毁它们，这样有利于规避一些难以发现的问题。

```
worker.onerror = (e) => {
	worker.terminate();
	reject(e);
};
worker.onmessage = (e) => {
	worker.terminate();
	resolve(e.data);
}
```

##行内 Workers
> 例子：[code](https://github.com/sashee/parallel-processing-in-js/blob/master/inline-worker.html),[demo](https://sashee.github.io/parallel-processing-in-js/inline-worker.html)

有些时候将 worker 代码写到一个外部文件可能会使原本简单的问题变得复杂，幸运的是，workers 也可以用一个 Blob 来初始化。

写一个行内 worker ，参考如下代码段：

```
<!-- http://stackoverflow.com/a/6454685/2032154 -->
<script id="worker" type="javascript/worker">
	// Put your worker code here
</script>
```

```
const code = URL.createObjectURL(new Blob([
	document.getElementById("worker").textContent
]));
const worker = new Worker(code);
```

这样你就创建了一个全局的 ObjectURL，但别忘了当不需要的时候要销毁它：

```
worker.terminate();
URL.revokeObjectURL(code);
```
## Workers 嵌套
理论上，你可以嵌套使用 worker，就像在主线程中定义一个 worker 一样。这里有一个简单的 [例子](https://html.spec.whatwg.org/multipage/workers.html#delegation)。但是不幸的是在 Chrome 中一直存在一个 [bug](https://bugs.chromium.org/p/chromium/issues/detail?id=31666) ，让我们不能愉快的玩耍，或许以后这个 bug 会修复，但是目前来说还是没有太多进展，所以你最好不要使用。
##数据传递
> 示例：[code](https://github.com/sashee/parallel-processing-in-js/blob/master/data-passing.html),[demo](https://sashee.github.io/parallel-processing-in-js/data-passing.html)

在 worker 数据传递的过程中有些需要注意的边缘情况。你可以传递数值，字符串，数组，也可以传递序列化/反序列化的对象。然而，你却不应该依赖序列化来保持数据结构，实际上，postMessage 用到了一种 [数据克隆算法](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm)，它会生成一些额外的属性比如 RegExps 和 Blobs 以及一些循环引用。

这就是说，你需要将你要传递的数据最小化。你不可以传递 functions ,即使是支持的类型也会有一些限制，这些也很容易产生一些难以发现的 bug。如果你将你的 API 定义为只传递字符串，数值，数组和对象的话，那你可能会避过这些问题。
### 循环引用
如果你有一个很复杂的对象，那么里面很可能存在循环引用，这时如果你将它序列化成 JSON，你将会得到一个 TypeError: Converting circular structure to JSON.

```
let a = {};
let b = {a};
a.b = b;
JSON.stringify({a,b}); // Error
```
然而你可以在 postMessage 中放心的使用，从而你就可以在 worker 中使用。

###Transferable objects
为了防止同时修改同一变量的场景，你传递给 postMessage 的所有变量都会复制一份，这样确保了你多个线程不会修改同一个变量。但如果你想要传一个非常大的数据的话，你就会发现复制操作是很慢的。比如，如果你在做一些图片相关的运算，你可能会传递整个图片信息，就可能会遇到复制性能的瓶颈。

好在有 [transferable object](https://developer.mozilla.org/en-US/docs/Web/API/Transferable) ，用 transfer 来代替 copy,比如ArrayBuffer 是transferable对象，而我们可以把任何类型的对象放在 ArrayBuffer 中。

如果你 transfer 一个对象，之前拥有它的线程被锁定权限，它确保了数据没有复制之前，不会被同时修改。

这时 postMessage 的代码段就有点尴尬了：

```
const ab = new ArrayBuffer(100);
console.log(ab.byteLength); // 100
worker.postMessage(ab, [ab]);
console.log(ab.byteLength); // 0
```
确保在 postMessage 中传递第二个参数，否则数据将会被复制。

```
const ab = new ArrayBuffer(100);
console.log(ab.byteLength); // 100
worker.postMessage(ab);
console.log(ab.byteLength); // 100
```

##Webpack
>实例：[code](https://github.com/sashee/parallel-processing-in-js/blob/master/webpack/src/app.js),[demo](https://sashee.github.io/parallel-processing-in-js/webpack/dist/index.html)

在 Webpack 中使用 Web worker 时，你需要用 [worker-loader](https://github.com/webpack/worker-loader)。将它添加到 package.json 中的 devDependencies，然后运行 npm install，就可以了。

用到 worker 时，只需要 require 它。

```
const workerCode = require("worker!./worker.js");

...

const worker = new workerCode();
```
这样就初始化了 worker，然后就像上面讲的一样使用 worker。

如果需要使用行内 worker，你需要传递 inline 参数给 loader。

实例：[code](https://github.com/sashee/parallel-processing-in-js/blob/master/webpack/src/inline.js),[demo](https://sashee.github.io/parallel-processing-in-js/webpack/dist/inline.html)

```
const workerCode = require("worker?inline!./worker.js");

...

const worker = new workerCode();

```
在 worker 中你也可以 import 模块。

```
import fibonacci from "./fibonacci.js";

...

const result = fibonacci(num);
```
### 缺点
在 Webpack 中使用 worker 很简单，但是在使用时也有一些坑值得你注意。

首先，无法将代码共用部分提取出来。如果你的 worker 中依赖一段共用代码，你只能把代码添加到 worker 中，不管其他地方是否也用到同样的代码。而且如果你多个 worker 要用同样的库，你也需要在每个 worker 中引入它们。

你可能会想如果你不用 worker-loader，然后用[CommonsChunkPlugin](https://webpack.github.io/docs/code-splitting.html)指定一个新的入口,可能会解决这个问题。但是不幸的是 worker 不像是浏览器 window ，一些 feature 不可用，所以一些代码必须要引入。

同时，用行内 worker 也不会解决问题，共用的代码依然会出现在多个地方。

第二点缺点是，行内 worker 可能会导致 [ObjectURLs内存泄露](https://github.com/webpack/worker-loader/blob/master/createInlineWorker.js#L16).它们被创建出来以后就不会被释放。这虽然不是一个大问题，但是如果你有很多“一次性” worker 的话，就会影响性能。

综上所述，我个人建议是使用标准的 worker，注意在 worker 中引入了什么。还要注意使用缓存。

##IFrames
Web worker 和 IFrame 很像，而且印象中 IFrame 也可以实现多线程。但是 IFrame 存在一些不是线程安全 API，比如 DOM 相关，浏览器不能为他们生成新的线程，参考[这里](https://sashee.github.io/parallel-processing-in-js/iframe.html).

在 IFrame 跨域中，很多 API 它都没有权限，也只能通过 postMessage，就像 Web Worker 一样。理论上，浏览器可以在不同的线程中运行 IFrame，也就可以用 IFrame 实现多线程。

但是实际并非如此，它还是单线程的，浏览器不会给它们额外的线程，参考[demo](https://sashee.github.io/parallel-processing-in-js/iframe-crossorigin.html)
##总结
Web Worker 解决了 JavaScript 一直以来的大难题，尽管它的语法有些奇怪而且有很多限制，但是它却可以真真正正的解决问题。从另外一方面来讲，它也还是个婴儿，某些方面还不是很成熟，不能让我们完全依赖，所以这个技术普及还有一段距离，目前适用场景也比较局限。

所以说，如果你需要做多线程，不要再等待其他的什么技术，学习 web worker 的边缘问题，避开它的坑，你就可以很好的提高用户体验。
