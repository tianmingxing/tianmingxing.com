---
title: 在传统多页面项目上使用VUE
date: 2019-05-28 09:18:16
tags:
- VUE
- 使用VUE
- 多页面应用
categories:
- 前端
---

![](/images/vue_logo.png)

[VUE](https://cn.vuejs.org/index.html)作为一款渐进式JS框架并且易于上手而大受开发者欢迎，哪怕你是一名后端开发人员也一定迫不及待想要来尝试。你也一定想要知道它和自己之前用过的其它框架有什么区别？优势在哪里？

任何一款框架都是为解决问题而生的，不管它如何花枝招展，也就是说它并没有那么神奇，它只是款实现需求的工具，在学习时搞清楚这些本质会让自己更容易理解。

<!-- more -->
VUE其实和你所用到的jQuery一样，都可以实现同样的需求，只不过两者在使用上面不相同，在框架运行原理上也不一样，但最终对DOM的操作过程还是一样的。那既然有不一样肯定会产生优劣，VUE的优肯定大于jQuery，否则也不会那么火了。

# 有什么不一样

其实最大的不同是到达目的地的路线不同，jQuery属于直接派，开发者要修改或获取某一个DOM时，通过jQuery直接获取到对象进行操作。而VUE则采用另一种方式，即数据驱动。可以想一下如果DOM上某个元素和对象进行绑定，当对象有变化时DOM能够自动变化，也是最好不过的了，不需要再手动直接操作DOM进行变化。这一种设计思想让开发变得相对容易许多，只要绑定好关系，维护对象状态即可。

举个例子说明便于理解。

在下面的HTML页面中有一个div，需要将p标签中的 `Hello` 改成 `Hello World`，下面看看用jQuery和VUE该怎么来实现。

```html
<html>
  <body>
    <div id="app">
      <p>Hello</p>
    </div>
  </html>
</body>
```

## jQuery

```javascript
$('#app p').text = 'Hello';
$('#app p').text = 'Hello World';
```


## VUE

```html
<html>
  <body>
    <div id="app">
      <p>{{ message }}</p>
    </div>
  </html>
</body>
```

```javascript
var app = new Vue({
  el: '#app',
  data: {
    message: 'Hello'
  }
});

//变量重新赋值后DOM自动更新到
app.message = 'Hello World';
```

## 思考

上面是一个简单的示例，对比差异不是很明显，但基本将两者的本质区分有演示出来。

* jQuery：数据有变化时要自己主动获取到DOM并进行赋值
* VUE：采用数据绑定DOM元素的设计思想，当数据变化时DOM自动变化，无需人工干预。

可以试想一下如果在一个页面中要变化的对象多、结构复杂的时候，采用jQuery这种方式会非常痛苦。

# 开始

如果要在传统多页面应用中使用VUE是非常简单的，就像使用jQuery一样，在页面上引入一个文件即可。

```html
<!-- 开发环境版本，包含了有帮助的命令行警告 -->
<script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
```
## 初始化VUE实例

```javascript
var app = new Vue({
  el: '#app',
  data: {
    message: 'Hello Vue!'
  }
})
```

在初始化中的el属性对应DOM上的一个元素，上面用的是 `app`，表示在该元素下面需要使用VUE的功能，之外的其它元素上使用VUE的语法是不起作用的。另外同一个页面上可以有多次VUE的实例化，但通常只需要一个就可以了，比如在body元素上应用。

一旦VUE进行实例化过后，我们就可以参考VUE的语法来进行操作，这个过程其实和使用模模板引擎类似，如果有用过的人肯定会有一点感觉。

# 总结

在这里并没有太多VUE的使用介绍，因为[官方文档](https://cn.vuejs.org/v2/guide/#%E5%A3%B0%E6%98%8E%E5%BC%8F%E6%B8%B2%E6%9F%93)真的写的太详细了，只要是跟着一步一步的练习，对于有一点前端知识的人都不是问题。
