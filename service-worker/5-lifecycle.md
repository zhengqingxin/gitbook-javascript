#【Service Worker】生命周期


> 此处[ 阅读原文 ](https://developers.google.com/web/fundamentals/instant-and-offline/service-worker/lifecycle#updates)


生命周期可能是 Service Worker 中最复杂的一部分了，如果你不知道它什么时间将要做什么，以及它带来的好处，那么你可能会有一种感觉：就是它一直在和你较劲。如果理解它的工作机制，你就可以给用户提供完美的，无感知的更新体验。

## 目的

本文中介绍利用生命周期可以实现的功能大概有如下几点：

* 实现缓存优先(offline-first)
* 在不打断现有 SW 的情况下，准备好一个新的 SW 
* 让注册 SW 的页面同一时间只归属同一个 SW 控制
* 确保你的网站只有一个版本在运行

最后一点尤其重要，一般情况下（没有 SW 的情况），用户浏览你的网站时可能先打开一个 tab，过了一会儿又打开了一个 tab，结果就是在同一时间，你的页面运行了两个版本，大部分时候，这样是没问题的，但是如果你使用了缓存，那么两个 tab 就要面临如何管理缓存的问题，如果处理不好，它可能会造成异常，严重的造成数据丢失。

> 用户非常不喜欢数据丢失，这会让他们非常忧桑。

## 第一个 Service Worker

* `install` 事件是 SW 触发的第一个事件，并且仅触发一次。
* `installEvent.waitUntil()`接收一个 Promise 参数，用它来表示 SW 安装的成功与否。
* SW 在安装成功并激活之前，不会响应 `fetch`或`push`等事件。
* 默认情况下，页面的请求（fetch）不会通过 SW，除非它本身是通过 SW 获取的，也就是说，在安装 SW 之后，需要刷新页面才能有效果。
* `clients.claim()`可以改变这种默认行为。 


<div>
	<iframe id="iframe-1" onload="iframeLoaded('iframe-1')" style="width:100%;border:none;" src="./image/5-lifecycle/index.html">
	</iframe>
</div>


举个例子：

```
<!DOCTYPE html>
3秒后将出现一张图片:
<script>
  navigator.serviceWorker.register('/sw.js')
    .then(reg => console.log('SW registered!', reg))
    .catch(err => console.log('Boo!', err));

  setTimeout(() => {
    const img = new Image();
    img.src = '/dog.svg';
    document.body.appendChild(img);
  }, 3000);
</script>
```
这里注册了一个 SW，并且在 3 秒后在页面上添加一个图片。

这是 SW 的代码：

```
self.addEventListener('install', event => {
  console.log('V1 installing…');

  // 这里缓存一个 cat.svg
  event.waitUntil(
    caches.open('static-v1').then(cache => cache.add('/cat.svg'))
  );
});

self.addEventListener('activate', event => {
  console.log('V1 now ready to handle fetches!');
});

self.addEventListener('fetch', event => {
  const url = new URL(event.request.url);

  //如果是同域并且请求的是 dog.svg 的话，那么返回 cat.svg
  if (url.origin == location.origin && url.pathname == '/dog.svg') {
    event.respondWith(caches.match('/cat.svg'));
  }
});
	
```
这里 SW 缓存了 cat.svg，并当请求 dog.svg 的时候返回它。然而，当你运行[ 这个例子 ](https://cdn.rawgit.com/jakearchibald/80368b84ac1ae8e229fc90b3fe826301/raw/ad55049bee9b11d47f1f7d19a73bf3306d156f43/) 的时候，你会发现第一次出现一只狗，刷新后，猫才会出现。

### 作用域与控制

SW 的默认作用域为基于当前文件 URL 的 `./`。意思就是如果你在`//example.com/foo/bar.js`里注册了一个 SW，那么它默认的作用域为 `//example.com/foo/`。

我们把页面，workers，shared workers 叫做`clients`。SW 只能对作用域内的`clients`有效。一旦一个`client`被“控制”了，那么它的请求都会经过这个 SW。我们可以通过查看`navigator.serviceWorker.controller`是否为 null 来查看一个`client`是否被 SW 控制。

### 下载-解析-执行

当你调用`.register()`的时候，第一个 SW 被下载下来，这过程中如果下载，解析或者在初始化中有错误的话，那么 `register` 的Promise 会返回 reject，然后 SW 会被销毁。

Chrome 开发者工具会展示出来错误，在 Application 中的 Service Workers Tab:

![](./image/5-lifecycle/register-fail.png)

### 安装(install)

SW 首先会触发`install`，每个 SW 只会被触发一次，当你修改你的 SW 后，浏览器会认为这是一个新的 SW，从而会再触发这个新 SW 的`install`事件，在后面会详细说到。

`install`是在 SW 控制 `clients`之前处理缓存很好的时机。在 `event.waitUntil()`传入的 Promise 会让浏览器知道 SW 什么时候安装成功以及是否成功。

当 Promise reject 的时候，代表着安装失败，浏览器将这个 SW 废弃掉，不会控制任何 clients。

### 激活(Activate)

安装成功后并激活(activate)成功后，SW 就可以处理“功能性的事件“了，比如`push`,`sync`。***但这并不代表调用`.register()`的页面会立即生效。***

第一次你请求[ 这个demo ](https://cdn.rawgit.com/jakearchibald/80368b84ac1ae8e229fc90b3fe826301/raw/ad55049bee9b11d47f1f7d19a73bf3306d156f43/)的时候，虽然在 SW 被激活后很久才请求了`dog.svg`（因为这里等待了三秒），但 SW 也并没有处理这个请求，结果你看见了一只狗。当你第二次请求的时候，也就是刷新页面，这时请求被处理了，当前页面和图片都经过了 SW 的 `fetch`事件，所以你看见了一只猫。

### clients.claim

你可以在`activate`事件中通过调用`clients.claim()`来让没被控制的 clients 受控。

比如[ 这个demo ](https://cdn.rawgit.com/jakearchibald/80368b84ac1ae8e229fc90b3fe826301/raw/df4cae41fa658c4ec1fa7b0d2de05f8ba6d43c94/)，可能第一次你就会看见一只猫，这里我说“可能”，是因为这时时间敏感的，仅当 SW 激活并且`clients.claim()`被调用成功在图片请求之前的时候才可以。

所以，可想而知，当你用 SW 加载与正常请求不同资源的时候（比如上面的例子），那用`clients.claim()`可能会遇到一些问题，这时有些资源可能不会通过你的SW。



> 我见过很多人在代码中把`clients.claim()`当做了必选项，但我自己很少这样做，因为仅仅是第一次加载不会通过 SW，而且页面还是都会正常运行的。


## 更新 Service Worker

简单来说：

* 触发更新的几种情况：
	* 第一次导航到作用域范围内页面的时候
	* 当在24小时内没有进行更新检测并且触发功能性时间如`push`或`sync`的时候
	* SW 的 URL 发生变化并调用`.register()`时
* 当 SW 代码发生变化，SW 会做更新（还将包括引入的脚本）
* 更新后的 SW 会和原始的 SW 共同存在，并运行它的`install`
* 如果新的 SW 不是成功状态，比如 404，解析失败，执行中报错或者在 install 过程中被 reject，它将会被废弃，之前版本的 SW 还是激活状态不变。
* 一旦新 SW 安装成功，它会进入`wait`状态直到原始 SW 不控制任何 clients。
* `self.skipWaiting()`可以阻止等待，让新 SW 安装成功后立即激活。

<div>
	<iframe id="iframe-2" onload="iframeLoaded('iframe-2')" style="width:100%;border:none;" src="./image/5-lifecycle/iframe2.html">
	</iframe>
</div>


下面我们来举个 SW 更新的例子，这是一个将猫变成马的故事：

```
const expectedCaches = ['static-v2'];

self.addEventListener('install', event => {
  console.log('V2 installing…');

  // 将 horse.svg 缓存在新的缓存 static-v2 中
  event.waitUntil(
    caches.open('static-v2').then(cache => cache.add('/horse.svg'))
  );
});

self.addEventListener('activate', event => {
  // 删除额外的缓存，static-v1 将被删掉
  event.waitUntil(
    caches.keys().then(keys => Promise.all(
      keys.map(key => {
        if (!expectedCaches.includes(key)) {
          return caches.delete(key);
        }
      })
    )).then(() => {
      console.log('V2 now ready to handle fetches!');
    })
  );
});

self.addEventListener('fetch', event => {
  const url = new URL(event.request.url);

  //如果是同域并且请求的是 dog.svg 的话，那么返回 horse.svg

  if (url.origin == location.origin && url.pathname == '/dog.svg') {
    event.respondWith(caches.match('/horse.svg'));
  }
});
```

如果你打开[ 这个 demo ](https://cdn.rawgit.com/jakearchibald/80368b84ac1ae8e229fc90b3fe826301/raw/ad55049bee9b11d47f1f7d19a73bf3306d156f43/index-v2.html),你会看到一只喵，原因是这样的...

### install
注意这里我们将缓存从 static-v1 换到了 static-v2，这代表了我用了一个新的缓存空间覆盖了之前 SW 正在使用的缓存。

这里新建了一块缓存的做法类似于原生 app 中将每块资源打包到一块指定的执行空间的做法，有时候结合实际情况，你也可以不这么做。
 
### Waiting
一旦新 SW 安装成功，它会进入`wait`状态直到原始 SW 不控制任何 clients。这个状态是`waiting`，这也是浏览器确保在同一时间只有一个版本的 SW 运行的机制。

如果你再次打开[ 这个 demo ](https://cdn.rawgit.com/jakearchibald/80368b84ac1ae8e229fc90b3fe826301/raw/ad55049bee9b11d47f1f7d19a73bf3306d156f43/index-v2.html)，你还是会看到一只喵，因为新的 SW 还是没有被激活，在开发者工具里你依然看到它是 waiting 状态。

![](./image/5-lifecycle/waiting.png)

尽管这个例子中你仅打开了一个 tab，但刷新页面并没有用，这是由于浏览器本身的机制，当你刷新的时候，当前页面不会离开，直到收到了一个响应头，而且即使这样，如果响应中包含`Content-Disposition`的话，当前页面还是不会离开。由于这个时间上的重叠，在刷新的时候当前的 SW 总是控制了一个 client。

为了让 SW 更新，你需要把所有用原始 SW 的页面 tab 关闭或者跳转走，这时你再访问[ 这个 demo ](https://cdn.rawgit.com/jakearchibald/80368b84ac1ae8e229fc90b3fe826301/raw/ad55049bee9b11d47f1f7d19a73bf3306d156f43/index-v2.html)，你就会看到了一匹野马。

这种机制类似于 Chrome 本身的更新机制，Chrome 在后台更新，只有当你重启浏览器的时候才会生效，在这期间你不会被打扰，可以继续使用当前版本。然而，这样可能会使我们开发者比较痛苦，好在开发者工具帮我们解决了这个事情，后面会说到。

### Activate

Activate 在旧的 SW 离开时会被触发，这时新的 SW 可以控制 clients。这时候你可以做一些在老 SW 运行时不能做的事情，比如清理缓存。

在上面的例子中，之前保留的缓存，在`activate`时间执行的时候被清理掉。

> 这里最好不要更新以前的版本，而是直接分配新的缓存空间。

如果你在`event.waitUntil()`中传入了一个 Promise，SW 将会缓存住功能性事件(`fetch`,`push`,`sync`等等)，直到 Promise 返回 resolve 的时候再触发，也就是说，当你的`fetch`事件被触发的时候，SW 已经被完全激活了。

> cache storage API 和 localStorage，IndexedDB 一样是“同域”的。如果你在一个父域下运行多个网站，比如 `yourname.github.io/myapp`,这就要小心你不要把别的网站的缓存删掉了。避免这个问题，你可以将 cache 的 key 设的具有唯一性，比如 myapp-static-v1 并且约束不要碰不以 myapp- 开头的缓存。

### skipWaiting

waiting 意在让你的网站同一时间只有一个 SW 在运行，但如果你不想要这样的话，你可以通过调用`self.skipWaiting()`来让新 SW 立即激活。

这么做会让你的新 SW 踢掉旧的，然后当它变为 waiting 状态时立即激活，注意这里不会跳过 installing，只会跳过 waiting。

在 waiting 之前或者之后调用`skipWaiting()`都可以，一般情况我们在 `install` 事件中调用：

```
self.addEventListener('install', event => {
  self.skipWaiting();

  event.waitUntil(
    // caching etc
  );
});
```

[这个例子](https://cdn.rawgit.com/jakearchibald/80368b84ac1ae8e229fc90b3fe826301/raw/ad55049bee9b11d47f1f7d19a73bf3306d156f43/index-v3.html)中，你可能直接可以看到一只奶牛，和`clients.claim()`一样，这是一场赛跑，仅当你的新 SW 安装，激活等早于你请求图片时，奶牛才会出现。

> `skipWaiting()`意味着新 SW 控制了之前用旧 SW 获取的页面，也就是说你的页面有一部分资源是通过旧 SW 获取，剩下一部分是通过新 SW 获取的，如果这样做会给你带来麻烦，那就不要用`skipWaiting()`,这点我们应该根据具体情况评估。


### 手动更新
像我之前说的，当页面刷新或者执行功能性事件时，浏览器会自动检查更新，其实我们也可以手动的来触发更新：

```
navigator.serviceWorker.register('/sw.js').then(reg => {
  // sometime later…
  reg.update();
});
```
如果你希望你的用户访问页面很长时间而且不用刷新，那么你可以每个一段时间调用一次`update()`。

### 避免改变 SW 的 URL
如果你看过我的文章[缓存最佳实践](https://jakearchibald.com/2016/caching-best-practices/)，你可能会考虑给每个 SW 不同的 URL。**千万不要这么做！**在 SW 中这么做是“最差实践”，要在原地址上修改 SW。

举个例子来说明为什么：

1.`index.html`注册了`sw-v1.js`作为SW。

2.`sw-v1.js`对`index.html`做了缓存，也就是缓存优先（offline-first）。

3.你更新了`index.html`重新注册了在新地址的 SW `sw-v2.js`.

如果你像上面那么做，用户永远也拿不到`sw-v2.js`，因为`index.html`在`sw-v1.js`缓存中，这样的话，如果你想更新为`sw-v2.js`，还需要更改原来的`sw-v1.js`。

在上面的 demo 里，我给每个 SW 用了不同的 URL，这只是为了做演示，不要在生产环境中这么做。


## 让开发更简单

SW 的生命周期是为了用户构建的，但这样难免让我们开发带来一些烦恼，幸亏与一些工具来帮助我们。

### Update on reload

这是我最喜欢的：

![](./image/5-lifecycle/update-on-reload.png)

这样把生命周期变得对开发友好了，每次跳转将会：

1.重新获取 SW

2.尽管字节一致，也会重新安装，也就是说`install`事件被执行并且更新缓存。

3.跳过 waiting，激活新的 SW。

4.导航到这个页面。

这就是说你每次操作都会更新而不用刷新页面或者关闭 tab。

### Skip Waiting
![](./image/5-lifecycle/skip-waiting.png)
如果你有个 SW 在等待状态，你可以点击 skipWaiting 让它立即变为激活状态。

### 强制刷新

如果你强制刷新页面，那么会绕过 SW，变成不受控，这个功能已被定为规范，所以在其他支持 SW 的浏览器中也适用。

## 处理更新

Service Worker 是[ 可扩展web ](https://extensiblewebmanifesto.org/)的一部分。想法初衷是，作为浏览器开发者，有时候我们可能不如 web 开发者更了解 web，所以，我们其实不应该提供仅仅可以解决具体问题的 API，而是应该给 web 开发者更多的权限从而更好的解决问题。

所以，我们尽可能的开放更多，SW 整个生命周期都是可查看的：

```
navigator.serviceWorker.register('/sw.js').then(reg => {
  reg.installing; // 安装中的 SW，或者是undefined
  reg.waiting; // 等待中的 SW，或者是undefined
  reg.active; // 激活中的 SW，或者是undefined

  reg.addEventListener('updatefound', () => {
    // 正在安装的新的 SW
    const newWorker = reg.installing;

    newWorker.state;
    // "installing" - 安装事件被触发，但还没完成
    // "installed"  - 安装完成
    // "activating" - 激活事件被触发，但还没完成
    // "activated"  - 激活成功
    // "redundant"  - 废弃，可能是因为安装失败，或者是被一个新版本覆盖

    newWorker.addEventListener('statechange', () => {
      // newWorker 状态发生变化
    });
  });
});

navigator.serviceWorker.addEventListener('controllerchange', () => {
	// 当 SW controlling 变化时被触发，比如新的 SW skippedWaiting 成为一个新的被激活的 SW
});
```

<script type="text/javascript">
  function iframeLoaded(id) {
      var iFrameID = document.getElementById(id);
      if(iFrameID) {
            iFrameID.height = "";
            iFrameID.height = iFrameID.contentWindow.document.body.scrollHeight + "px";
      }   
  }
</script>  