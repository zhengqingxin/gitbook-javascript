#【JS 小短文】变态版斐波那契

斐波那契数列大家应该再熟悉不过了，平时面试也经常会被问到，然而不知道大家有没有考虑过它的效率呢？

## 普通版
我们一般给出的代码应该是这样的：

```
function fibonacci(n) {
    if(n==0 || n == 1)
        return n;
    return fibonacci(n-1) + fibonacci(n-2);
}
```
这段代码逻辑完全没问题，但是如果你稍测试一下可能就会发现问题了，比如可以试一下 `fibonacci(70)`，这时你可能吃个饭回来它还没执行完。


## 变态版
之所以叫它“变态版”，是因为在`fibonacci(1000)`以下基本可以在 5 毫秒之内完成，效率提升了 Infinity 倍，然而为什么差距这么大？我们先看个图：

![](./image/memorized-fibonacci/1.gif)

这是`n = 5`的时候的计算过程，显而易见，它包含了很多次重复的计算，优化它也很简单，我们只需要加个缓存，代码如下：

```
var fibonacci = (function() {
  // 这里我用了 Map，用普通对象也是可以的
  var cache = new Map();
   return function(n){
       if(n == 0 || n == 1){
       cache.set(n,n);
       return n;
      }
      var v1 = cache.get(n-1) ? cache.get(n-1):fibonacci(n-1);
      var v2 = cache.get(n-2) ? cache.get(n-2):fibonacci(n-2);
      var nVal = v1 + v2;
      cache.set(n, nVal);
      return nVal;
   }
})()
```