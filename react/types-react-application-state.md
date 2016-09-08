#【译】React中的5种State类型
>[原文地址](http://jamesknelson.com/5-types-react-application-state/)

不知道你有没有遇到和我一样的困惑，当我写 React 的时候，我经常会做[ 思想斗争 ](http://jamesknelson.com/state-react-1-stateless-react-app/)：我的 state 应该放在那里？无论我怎么重新安排`setState`，我都不会觉得很合理。

![](image/types-react-application-state/1.png)

这也是为什么我发现 Redux 以后异常激动的原因，Redux 给我提供了一个[ 统一管理 state 的地方 ](http://jamesknelson.com/state-react-2-inception-redux/)，理论上来说这非常好。

![](image/types-react-application-state/2.png)

但是后来我发现将所有东西都放在同一个地方会让我很难找到其中的一个。

![](image/types-react-application-state/3.png)

最后我发现我需要的不仅是一个统一存储 state 的地方，还需要一个能够组织它们放到正确的位置的系统。

![](image/types-react-application-state/4.png)

对于我来说，这种系统来源于我将我的 state 分成了 5 个种类。这样的话问题就变成了“这个 state 属于什么类型？”的问题，随之问题变得很简单了。

所以说，React apps 有五种类型的 state，每种类型都有它的标准。 It interacts with the other types of state in well defined ways — as long as it follows the rules. And based on these rules, you’ll find that there are ways of storing the state which just make sense.

这五种类型的 state 分别是：

* Data
* Communication
* Control
* Session
* Location

这五种当中，“Data” 可能是最好解释的一个，所以我们下面来说。

## Data
Data state 包括你应用中需要临时存储的数据，也就是你的业务数据。

比方说你现在正在开发一个规模比较大的发票系统。这时候你从服务端获取到的 Data state 可能包括 invoices(发票信息)，
contacts(联系人)，receipts(收据信息)等等。这些所有的数据都来自于外部，it is probably going to have some sort of identifier which you use to request it:

GET the /invoice with ?id=happiness, or something

实际上，我把 Data state 的标准定义为：

