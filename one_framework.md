###统一前端框架的那些事

当前开发场景多种多样，无论是知识产权问题、场景需要还是技术公司的期望、个人技术体现等等原因，

必然会出现不同的框架，对开发效率、性能统一浏览器或者移动端方面考虑大体可以分一类，比如react、vue、angular、快应用等；从跨平台统一可分为一类，如RN、weex、小程序、支付宝小程序、头条小程序、ionic等。

如果你对上面框架有所了解，react、vue、ng算是基础类的框架了，这里就说说React吧。

如果我们想用react开发微信小程序怎么做？直奔主题吧，我们需要babel工具集：

```javascript
import traverse from "@babel/traverse";
import generate from '@babel/generator'
import template from '@babel/template'
import {transformFromAst} from "@babel/core";
import * as t from '@babel/types'
import * as parser from "@babel/parser";
```

使用这些工具足够了，其中特别要熟练了解的是`traverse`及`@babel/types`，我也见了一个[仓库](https://github.com/wuchaofan/planet) 有时间会持续更新，待续。。。