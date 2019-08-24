# 记一次H5迁移

之前的项目分两个仓库，一个老得用ejs、gulp、requirejs方式开发、打包；
另一个是vue多页面。两个项目build后的页面会放在一起，压缩下发给app端，
用到了很多Native功能，比如：相机、http请求、图片识别等功能。
由于业务需要，希望一套代码维护，可以在浏览器上运行。

> 项目的基本需求：

1. 尽量共用一套代码；
2. 在浏览器上运行之前在app端h5的业务代码

看起来直观的想法是尽量少动业务逻辑，直接重写native bridge部分的api，不能实现的api再修改交互页面。再之后的迁移中，native api部分难度不算大，h5都有折中的办法。但是app端通过A页面打开B页面，实际是两个webkit，也就是A页面是活着的，而浏览器中A页面是死掉的，back回来页面重新载入执行。这样就会导致在app端B页面回来，native可以通过发布订阅模式触发A页面的监听逻辑，而浏览器做不到。

### 方案

最直观的想法有两个，怎样做到从B页面回来后，A页面能够获取B页面的数据能。

1、通过url带参数或者localStorage，A页面能够获取

2、通过把多页面转换成vue的SPA架构

首先来说说第二种SPA能够很好的解决页面间的通信，所以B回到A能都触发事件，可以利用keep-alive功能，数据传递方式几乎不用改动。看起来挺不错，但是最直接的问题是老项目与新项目页面怎样通信，一个是SPA单页，一个是多页，部署上就要有改动；其次使用keep-alive导致相同path，但是query不同的url不好处理，通过keep-alive组件上加key又会导致页面数据丢失。还有一个很难解决的是headerbar部分，app里这部分是native实现，而现在在浏览器spa中，headerbar中数据更新是一个难点，因为从B回到A时，mounted钩子不会触发，触发的是activated，所以业务中的setHeader逻辑都要在
activated钩子里调用。最终放弃使用SPA的方式。

那么第一种直观上来说架构没有变，都是多页面，新老项目部署不变。但是最大的难点是怎样是A存活呢，可以肯定的说做不到。我们只是希望在A页面加入到订阅的逻辑，在B回到A时能触发即可。把callback不存在sessionStorage或者localStorage不可，因为上下文信息会丢失。这部分最后的办法就是把所有订阅代码提取出来，放在Object里，如下
伪代码：

```javascript
EventMap = {
    a1: function (from, data) {...},
    a2: function (from, data) {...}
}
```

之前的订阅代码如下伪代码：

```javascript
EventListener.on('onBack', function (from, data) {
   ...
})
```

现在改成：

```javascript
EventListener.on('onBack', EventMap.a1, 'a1')
```

当然我们还需要在EventListener.on方法里保存a1到sessionStorage或者localStorage里，因为我们需要维护一个列表，在A页面再次载入后读取这个列表
然后获取EventMap中的function遍历执行。当然这只是经过多次尝试后的方案，
需求刚刚开始，在此记录一下。

2019/8/24