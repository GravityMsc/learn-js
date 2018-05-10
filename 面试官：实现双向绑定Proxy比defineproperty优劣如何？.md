## 前言

双向绑定其实已经是一个老掉牙的问题了，只要涉及到MVVM框架就不得不谈的知识点，但它毕竟是Vue的三要素之一。

### Vue三要素

* 响应式：例如如何监听数据变化，其中的实现方法就是我们提到的双向绑定。
* 模板引擎：如何解析模板。
* 渲染：Vue如何将监听到的数据变化和解析后的HTML进行渲染。

可以实现双向绑定的方法有很多，KnockoutJS基于观察者模式的双向绑定，Ember基于数据模型的双向绑定，Angular基于脏检查的双向绑定，本篇文章我们重点讲面试中常见的基于数据劫持的双向绑定。

常见的基于数据劫持的双向绑定有两种实现，一个是目前Vue在用的Object.defineProperty，另一个是ES2015中新增的Proxy，而Vue的作者宣称将在Vue3.0版本后加入Proxy从而代替Object.defineProperty，通过本文你也可以知道为什么Vue未来会选择Proxy。

> 严格来讲Proxy应该被称为【代理】而非【劫持】，不过由于作用有很多相似之处，我们在下文中就不再做区分，统一叫做【劫持】。

我们可以通过下图清楚看到以上两种方法在双向绑定体系中的关系。

![双向绑定图示](https://user-gold-cdn.xitu.io/2018/5/1/1631801840069098?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

> 基于数据劫持的当然还有已经凉透的Object.observe方法，已被废弃。

> **提前声明**：我们没有对传入的参数进行及时判断而规避错误，仅仅对核心方法进行了实现。



## 1.基于数据劫持实现的双向绑定的特点

### 1.1 什么是数据劫持

数据劫持比较好理解， 通常我们利用Object.defineProperty劫持对象的访问器，在属性值发生变化时我们可以获取变化，从而进行进一步操作。

```javascript
// 这是将要被劫持的对象
const data = {
    name: '',
};

function say(name){
    if(name === '古天乐'){
        console.log('给大家推荐一款超好玩的游戏');
    } else if(name === '渣渣辉'){
        console.log('戏我演过很多，可游戏我只玩贪玩懒月');
    } else {
        console.log('来做我的兄弟');
    }
}

// 遍历对象，对其属性值进行劫持
Object.keys(data).forEach(function(key){
    Object.defineProperty(data, key, {
        enumerable: true,
        configurable: true,
        get: function(){
            console.log('get');
        },
        set: function(newVal){
            // 当属性值发生变化时我们可以进行额外操作
            console.log(`大家好，我系${newVal}`);
            say(newVal);
        },
    });
});

data.name = '渣渣辉';
// 大家好，我系渣渣辉
// 戏我演过很多，可游戏我只玩贪玩懒月
```

### 1.2 数据劫持的优势

目前业界分为两个大的流派，一个是以React为首的单向数据绑定，另一个是以Angular、Vue为主的双向数据绑定。

> 其实三大框架都是既可以双向绑定也可以单向绑定，比如React可以手动绑定onChange和value实现双向绑定，也可以调用一些双向绑定库，Vue也加入了props这种单向流的api，不过都非主流卖点。

单向或者双向的优劣不在我们的讨论范围，我们需要讨论一下对比其他双向绑定的实现方法，数据劫持的优势所在。

1. 无需显式调用：例如Vue运用数据劫持 + 发布订阅，直接可以通知变化并驱动视图，上面的例子也是比较简单的实现data.name = '渣渣辉' 后直接触发变更，而比如Angular的脏检测则需要显式调用markForCheck（可以用zone.js避免显式调用，不展开），react则需要显式调用setState。
2. 可精确得知变化数据：还是上面的小例子，我们劫持了属性的setter，当属性值改变，我们可以精确获知变化的内容newVal，因此在这部分不需要额外的diff的操作，否则我们只知道数据发生了变化，而不知道具体哪些数据变化了，这个时候需要大量diff来找出变化值，这是额外性能损耗。

### 1.3 基于数据劫持双向绑定的实现思路

数据劫持是双向绑定各种方案中比较流行的一种，最著名的的实现就是Vue。

基于数据劫持的双向绑定离不开Proxy与Object.defineProperty等方法对对象/对象属性的“劫持”，我们要实现一个完整的双向绑定需要以下几个要点。

1. 利用Proxy或Object.defineProperty生成的Observer针对对象/对象的属性进行“劫持”，在属性发生变化后通知订阅者。
2. 解析器Compile解析模板中的Directive（指令），收集指令所依赖的方法和数据，等待数据变化然后进行渲染。
3. Watcher属性Observer和Compile桥梁，它将接收到的Observer产生的数据变化，并根据Compile提供的指令进行视图渲染，使得数据变化促使视图变化。

![数据劫持实现双向绑定的要点](https://user-gold-cdn.xitu.io/2018/4/11/162b38ab2d635662?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

> 我们看到，虽然Vue运用了数据劫持，到那时依然离不开发布订阅的模式，之所以在系列2做了[Event Bus的实现](https://juejin.im/post/5ac2fb886fb9a028b86e328c)，就是因为我们不管在学习一些框架的原理还是一些流行库（例如Redux、Vuex），基本上都离不开发布订阅模式，而Event模块则是此模式的经典实现，所以如果不熟悉发布订阅模式，建议读一下系列2的文章。

## 2.基于Object.defineProperty双向绑定的特点

关于Object.defineProperty的文章在网络上已经汗牛充栋，我们不想花过多的时间在Object.defineProperty上面，本节我们主要讲解Object.defineProperty的特点，方便我们接下来与Proxy进行对比。

> 对Object.defineProperty还不了解的请阅读[文档](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)

两年前就有人写过基于Object.defineProperty实现的[文章](https://segmentfault.com/a/1190000006599500)，想深入理解Object.defineProperty实现的推荐阅读，本文也做了相关参考。

> 上面我们推荐的文章为比较完整的实现（400行代码），我们在本节只提供一个极简版（20行）和一个简化版（150行）的实现，读者可以循序渐进地阅读。

### 2.1 极简版的双向绑定

我们都知道，Object.defineProperty的作用就是劫持一个对象的属性，通常我们对属性的getter和setter方法进行劫持，在对象的属性发生变化时进行特定的操作。

我们就对对象obj的text属性进行劫持，在获取此属性的值时打印'get val'，在更改属性值的时候对DOM进行操作，这就是一个极简的双向绑定。