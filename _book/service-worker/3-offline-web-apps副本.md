# 离线 Web 应用

## 我是谁
我们所说的离线 Web 应用，老外叫 offline web app 或者 online-offline app，第一次听说这个词时，我是这样的...

![](image/3-offline-web-apps/1.jpg)

心想这老外就是文艺啊，后来我去了趟武当山，找到了大师，顿时大彻大悟，online-offline web app 其实形容的非常的 very good，就是一会儿 online，一会儿 offline 的 web 应用，或者说是间歇性在线的 web 应用。而其实还有一种叫做 always-offline web app，这个就很简单了，就是全部下载下来以后就不用再需要网络了，ps：我目前还不懂这样的 app 除了做个计算器记事本还有什么用...所以，我们所说的离线 web 应用，一般都是指前一种间歇性在线的 web 应用。

## 从哪里来
为什么离线 web 应用会被提出？我先来举几个例子：

* 可能你是某大型互联网公司的一个小白领，每天按时上下班，爱好在某宝上逛逛逛买买买，最大梦想就是在公司的厕所里能上某宝看看，但是，你懂的，没门，因为网络限制。但是这个时候有了离线 web 应用，当你早上出门前，早在后台偷偷的帮你把今天的内容更新好了，这会不会让你狂喜呢？

* 假如你是个学霸，喜欢在地铁上看一些在线的文档(可能是个伪学霸，真学霸是不会挤地铁的)，比如 MDN 之类。有了离线 web 应用，就会有这样一个场景，在家 wifi 刷刷的时候，我把整个页面以及相关联页面全部缓存，然后点选项，添加到桌面。这样既省了流量，又提供了流畅的体验，我们就可以愉快的挤地铁了。

* 比如我是个大老板，每天到处飞，最大梦想就是能够在飞机上处理邮件，但是我发不出去啊，有了离线 web 应用，妈妈再也不用担心我回复邮件了，在飞的上回复，下飞的有网时，立即嗖嗖嗖发出去咯。

* 我是个程序猿，我公司不限网，所以我不需要什么离线 web 应用？ No！离线 web 应用在有网络的时候，更能够提升访问性能。

总之，离线能力会给 Web 应用带来巨大的好处，也是 Web 发展的方向，伴随着 sw 的提出，离线 Web 应用更加富有现实意义了。

## 需要解决的问题
然而，现实也比较残酷，开发一个离线 Web 应用需要解决的问题也是很多的，比如下面三点：

* 选择缓存策略
* 缓存的时间节点
* 页面上的用户体验

用这三点举例，是因为我们基本都会遇到的问题，下面来一一解释一下。

## 缓存策略
缓存策略大概不用解释大家也知道什么意思，简单来说就是为下面这四个演员分配角色。

![](image/3-offline-web-apps/2.png)

### 仅 Cache

![](image/3-offline-web-apps/3.png)

就是说所有的请求都从 cache 里拿，如果取不到，就会返回错误。代码类似这样：

```
self.addEventListener('fetch', function(event) {
  // If a match isn't found in the cache, the response
  // will look like a connection error
  event.respondWith(caches.match(event.request));
});
```
然而，使用这种方式的场景似乎很少见...

### 仅 Network

![](image/3-offline-web-apps/4.png)

这种方式相当于 on-offline ，所有资源都请求网络，这种场景适用于处理一些刷新非常频繁的资源，实时性非常高的网络资源。代码类似：

```
self.addEventListener('fetch', function(event) {
  event.respondWith(fetch(event.request));
  // or simply don't call event.respondWith, which
  // will result in default browser behaviour
});
```

### 缓存优先

![](image/3-offline-web-apps/51.png)

这种方式是很普遍的，处理请求时，先从缓存中取，取不到的话去请求网络资源，成功后分别分发给页面和缓存，这样新资源只有第一次访问的时候回去请求外部资源，以后都会走缓存了，代码类似：

```
self.addEventListener('fetch', event=> {
  event.respondWith(
    caches.open('my-sw-test').then(function (cache) {
      return cache.match(event.request).then(function (response) {
        return response || fetch(event.request).then(function (response) {
            cache.put(event.request, response.clone());
            return response;
          });
      });
    })
  );
});

```
