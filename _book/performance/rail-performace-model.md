#【译】前端性能——RAIL 模型

>[原文地址](https://developers.google.com/web/tools/chrome-devtools/profile/evaluate-performance/rail#focus-on-the-user)

RAIL 是一个以用户为中心的性能模型，每个 Web App 的生命周期中都有这4个方面，性能优化在这4方面中也以不同的方式体现。

![](./image/rail-performance-model/1.png)
>TL;DR
>
>关注用户：目的不是说让你的网站变快，而是让你的用户开心。
>
>立即响应用户：响应用户 input 在100ms以下。
>
>每帧渲染时间小于16ms，以防用户感觉卡顿。
>
>主线程空闲时间最大化。
>
>吸引用户交互，交互内容要在1s以下。

##Response：在100ms以内响应
用户输入时，在感觉到卡顿之前(100ms)做出响应，这里包括所有的输入，无论是点击按钮，切换，或者开始一个动画。

如果你没有及时响应，action 和 reaction 之间的连接就会被打破，用户就会感觉到。

