# Offline：我是谁，我从哪里来？

> 这篇文章是译文，点击[ 查看原文 ](http://www.html5rocks.com/en/tutorials/offline/whats-offline/)

## Offline 介绍

Web 和 Online 一直关系紧密，甚至有些人认为它们是同义词，而为什么我们今天要讨论 Offline ？它究竟是什么意思呢？

有时候我们会讨论完全离线的 Web 应用这样一个神秘的物种，现在在网络上也很少会见到。一般我们经常会访问 file:/// 这样一个 URI，它指向一个硬件驱动，比如 USB key，或者 DVD(如果你还记得这些)。比如说，有时你在会议上要展示U盘里的内容，你会直接在你的 ppt 中加个  file:/// ，然后直接点进去用浏览器打开。更复杂的例子就是像[  TiddlyWiki ](http://tiddlywiki.com/)一样的单页应用(SPA)，它是一个 always-offline 的记事本应用，利用浏览器特性将整个应用保存在本地。

上面有提到 always-offline 整个概念，
其实还有个 online-offline ，也叫 offline apps，这两个概念是有细微差别的。目前很多 app 都是 offline apps，这些 app 在有网络的时候从远程同步数据，然后在断网的时候也可以使用。