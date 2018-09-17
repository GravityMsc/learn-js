日常工作中经常会发现有大量业务逻辑是重复的，而用别人的插件也不能完美解决一些定制化的需求，所以我决定把一些常用的组件抽离、封装出来，形成一套自己的插件库。同时，我将用这个教程系列记录下每一个插件的开发过程，手把手教你如何一步一步去造出一套实用性、可复用性高的轮子。

> 目前项目使用ES5及UMD规范封装，所以在前端暂时只支持\<script>标签的引入方式，未来会逐步用ES6进行重构。

![分页器的原型演示](https://user-gold-cdn.xitu.io/2018/7/26/164d46cd3fe7c8e5?imageslim)

## JavaScript模块化

要开发一个JavaScript的插件，首先要从JavaScript的模块化讲起。什么是模块化？简单的说就是让JavaScript能够以一个整体的方式去组织和维护代码，当多人开发时可以互相引用对方的代码块又不造成冲突。ECMAScript6标准之前常见的模块化规范有：CommonJS、AMD、UMD等，因为我们的代码暂时是采用ES5语法进行开发，所以我们选用UMD的规范来组织代码。关于模块化的发展过程可以参考：

* [JavaScript模块化编程简史（2009-2016）](https://yuguo.us/weblog/javascript-module-development-history)
* [JavaScript模块演化简史](https://zhuanlan.zhihu.com/p/26231889)

在这种模块规范的标准之上，我们还需要一种机制来加载不同的模块，例如实现了AMD规范的require.js，其用法可以参考阮一峰写的这篇教程：

* [JavaScript模块化编程（三）：require.js的用法](http://www.ruanyifeng.com/blog/2012/11/require_js.html)

因为我们开发的轮子暂时不涉及到多模块加载，所以模块的加载暂时不予过多讨论，读者可以自己进行拓展学习。

回到我们的主题上，在正式开发之前，还需要补充一点其他方面的知识。

## 自执行函数

定义一个函数，ES5一般有三种方式：

1. 函数声明

   ```javascript
   function foo(){}
   ```

   这样声明的函数和变量一样，会被自动提升，所以我们我可以把函数声明放在调用它的语句后面：

   ```javascript
   foo();
   function foo(){}
   ```

2. 函数表达式

   ```javascript
   var foo = function(){}
   ```

   右边其实是一个匿名函数，只不过赋值给了一个变量， 我们可以通过这个变量名来调用它，但是和第一种方式不同的是，通过表达式声明的函数不会被提升。

3. 使用Function构造函数

   ```javascript
   var foo = new Function()
   ```

那么有没有一种办法，可以不写函数名，直接声明一个函数并自动调用它呢？答案是肯定的，那就是使用自执行函数。

自执行函数Immediately-Invoked Function Expression，顾名思义，就是自动执行的函数，有的地方也称为立即调用的函数表达式。它的基本形式如下：

```javascript
(function (){
    console.log('hello');
}());
   
(function (){
    console.log('hello');
})();
```

> 两种写法是等效的，只不过前者让代码看起来更像是一个整体。

可以看到，这两种写法的作用其实就是在()内定义函数，然后又使用()来执行函数，因此它就是自执行的。

IIFE的一些好处如下：

* 避免污染全局变量
* 减少命名冲突
* 惰性加载

最重要的一点，它可以创建一个独立的作用域，而在ES6之前JavaScript是没有块级作用域的。利用这一点，我们可以很轻松的保证多个模块之间的变量不被覆盖了：

```javascript
// libA.js
(function(){
    var num = 1;
})();
   
// libB.js
(function(){
    var num = 2;
})();
```

上面这两个文件模块中的作用域都是独立的，互不影响。（如果模块之间想要互相引用，就需要用到模块的加载器了，例如上面提到的require.js等库）

在此基础上，我们就可以看看一个实现了UMD规范的IIFE模板是什么样子了：

```javascript
// if the module has no denpendencies, the above pattern can be simplified to
(function (root, factory){
    if(typeof define === 'function' && define.amd){
        // AMD. Register as an anonymous module.
        define([], factory);
    } else if(typeof module === 'object' && module.exports){
        // Node. Does not work with strict CommonJS, but
        // only CommonJS-like environments that support module.exports,
        // like Node.
        module.exports = factory();
    } else {
        // Browser globals (root is window)
        root.returnExports = factory();
    }
}(typeof self !== 'undefined' ? self : this, function(){
    // Just return a value to define the module export.
    // This example returns an object, but the module
    // can return a function as the exported value.
    return {};
}));
```

可以看到，UMD规范同时兼容了浏览器、Node环境及AMD规范，这样我们的代码使用UMD包装后就可以在不同的环境中运行了。

## 插件模板

开发插件最重要的一点，就是插件的兼容性，一个插件至少要能同时在几种不同的环境中运行。其次，它还需要满足以下几种功能及条件：

1. 插件自身的作用域与用户当前的作用域相互独立，也就是插件内部的私有变量不能影响使用者的环境变量；
2. 插件需具备默认设置参数；
3. 插件除了具备已实现的基本功能外，需提供部分API，使用者可以通过该API修改插件功能的默认参数，从而实现用户自定义插件效果；
4. 插件支持链式调用；
5. 插件需提供监听入口，及针对指定元素进行监听，使得该元素与插件响应达到插件效果。

第一点我们利用UMD包装的方式已经实现了，现在来看第二和第三点。

通常情况下，一个插件内部会有默认参数，并且会提供一些参数让用户实现部分功能的自定义。那么怎么实现呢，这其实就是一个对象合并的问题，例如：

```javascript
function extend(o, n, override){
    for(var p in n){
        if(n.hasOwnProperty(p) && (!o.hasOwnProperty(p)||override))
            o[p] = n[p];
    }
}
   
// 默认参数
var options = {
    pageNumber: 1,
    pageShow: 2
};
   
// 用户设置
var userOptions = {
    pageShow: 3,
    pageCount: 10
};
   
extend(options, userOptions, true);
   
// 合并后
options = {
    pageNumber: 1,
    pageShow: 3,
    pageCount: 10
};
```

如上，采用一个类似的extend函数就可以实现对象的合并了，这样我们的插件也就实现了设置参数的功能。

> 这里的extend函数为浅拷贝，因为插件的用户参数一般是不会修改的，如果想实现深拷贝可参考jQuery中extend的实现方法。

第四点我们插件暂时不需要这样的功能，可以暂时不支持它。第五点在代码中我们会通过回调函数去逐步实现它。

综上，我们就可以实现出一个基础的插件模板了：

```javascript
;	// JavaScript弱语法的特点，如果前面刚好有个函数没有以“；”结尾，那么可能会有语法错误
(function(root, factory){
    if(typeof define === 'function' && define.amd){
        define([], factory);
    } else if(typeof module === 'object' && module.exports){
        module.exports = factory();
    } else {
        root.Plugin = factory();
    }
}(typeof self !== 'undefined' ? self : this, function(){
	'use strict';
       
    // tool
    function extend(o, n, override){
        for(var p in n){
            if(n.hasOwnProperty(p) && (!o.hasOwnProperty(p) || override))
                o[p] = n[p];
        }
    }
       
    // polyfill
    var EventUtil = {
        addEvent: function(element, type, handler){
            // 添加绑定
            if(element.addEventListener){
                // 使用DOM2级方法添加事件
                element.addEventListener(type, handler, false);
            } else if(element.attachEvent){
                // 使用IE方法添加事件
                element.attachEvent("on" + type, handler);
            } else {
                // 使用DOM0级方法添加事件
                element["on" + type] = handler;
            }
        },
        // 移除事件
        removeEvent: function(element, type, handler){
            if(element.removeEventListener){
                element.removeEventListener(type, handler, false);
            } else if(element.datachEvent){
                element.datachEvent("on" + type, handler);
            } else {
                element["on" + type] = null;
            }
        },
        getEvent: function(event){
            // 返回事件对象引用
            return event ? event: window.event;
        },
        // 获取mouseover和mouseout相关元素
        getRelatedTarget: function(event){
            if(event.relatedTarget){
                return event.relatedTarget;
            } else if(event.toElement){
                // 兼容IE8
                return event.toElement;
            } else if(event.formElement){
                return event.formElement;
            } else {
                return null;
            }
        },
        getTarget: function(event){
            // 返回事件源目标
            return event.target || event.srcElement;
        },
        preventDefault: function(event){
            // 取消默认事件
            if(event.preventDefault){
                event.preventDefault();
            } else {
                event.returnValue = false;
            }
        },
        stopPropagation: function(event){
            if(event.stopPropagation){
                event.stopPropagation();
            } else {
                event.cancelBubble = true;
            }
        },
        // 获取mousedown或mouseup按下或释放的按钮是鼠标中的哪一个
        getButton: function(event){
            if(document.implementation.hasFeature("MouseEvents", "2.0")){
                return event.button;
            } else {
                // 将IE模型下的button属性映射为DOM模型下的button属性
                switch(event.button){
                    case 0:
                    case 1:
                    case 3:
                    case 5:
                    case 7:
                        // 按下的是鼠标主按钮（一般是左键）
                        return 0;
                    case 2:
                    case 6:
                        // 按下的是中间的鼠标按钮
                        return 2;
                    case 4:
                        // 鼠标次按钮（一般是右键）
                        return 1;
                }
            }
        },
       // 获取表示鼠标滚轮滚动方向的数值
        getWheelDelta: function(event){
            if(event.wheelDelta){
                return event.wheelDelta;
            } else {
                return -event.detail * 80;
            }
       	},
        // 以跨浏览器取得相同的字符编码，需在keypress事件中使用
        getCharCode: function(event){
            if(typeof event.charcode == "number"){
                return event.charCode;
            } else {
                return event.keyCode;    
            }
        }
    };
       
    // plugin construct function
    function Plugin(selector, userOptions){
    // Plugin() or new Plugin()
        if(!(this instanceof Plugin)) return new Plugin(selector, userOptions);
        this.init(selector, userOptions);
    }
    Plugin.prototype = {
        constructor: Plugin,
        // default option
        options: {},
        init: function(selector, userOptions){
            extend(this.options, userOptions, true);
        }
    };
       
    return Plugin;
}));
```

这里还使用到了一个EventUtil对象，它主要是针对事件注册的一些兼容性做了一些polyfill封装，具体原理可以参阅：

* [EventUtil——跨浏览器的事件对象](https://www.cnblogs.com/hykun/p/EventUtil.html)
* [跨浏览器的事件对象——EventUtil中的方法及用法](https://www.cnblogs.com/vali/p/5881329.html)

到此，一个插件的基本模板就大致成型了。下一节，我们终于可以正式开始分页插件的开发了！

## 思路分析

> 有人说计算机的本质就是对现实世界的抽象，而编程则是对这个抽象世界规则的制定。

正如上面这句话所说，在实际编码之前我们一般需要对要实现的需求效果进行一个思路的分析，最后再进一步把这个思路过程抽象为有逻辑的代码。我们先看一下要实现的分页效果是什么样的，我把它分成两种情况，显示和不显示省略号的，首先来看第一种：

```javascript
// 总共30页
// 第一种情况：不显示省略号，当前页码前后最多显示2个页码
当前页码为 1，那么显示 1 2 3 4 5
当前页码为 2，那么显示 1 2 3 4 5
当前页码为 3，那么显示 1 2 3 4 5
当前页码为 4，那么显示 2 3 4 5 6
...
当前页码为 15，那么显示 13 14 15 16 17
...
当前页码为 27，那么显示 25 26 27 28 29
当前页码为 28，那么显示 26 27 28 29 30
当前页码为 29，那么显示 26 27 28 29 30
当前页码为 30，那么显示 26 27 28 29 30
```

虽然上面每一个数字在实际应用中都是一个按钮或超链接，但现在既然是分析，我们不妨就把它简化并忽略，于是这个问题就变成了一个简单的字符串输出题。我们先定义一个函数：

```javascript
function showPages(page, total, show){
    
}
```

函数传入的参数分别为：当前页码、总页码数、当前页码面前后最多显示页码数，然后我们需要循环调用这个函数打印分页：

```javascript
var total = 30;
for(var i = 1; i <= total; i++){
    console.log(showPages(i, total));
}
```

这样从页码为1到最后一页的结果就全输出了，最后我们需要完成showPages()函数：

```javascript
function showPages(page, total, show){
    var str = '';
    if(page < show + 1){
        for(var i = 1; i <= show * 2 + 1; i++){
            str = str + ' ' + i;
        }
    } else if(page > total - show){
        for(var i = total - show * 2; i <= total; i++){
            str = str + ' ' + i;
        }
    } else {
        for(var i = page - show; i <= page + show; i++){
            str = str + ' ' + i;
        }
    }
    return str.trim();
}
```

思路是分段拼出页码，打印结果如下：

![不显示省略号正常输出页码](https://user-gold-cdn.xitu.io/2018/7/26/164d43d7da9c6309?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

不显示省略号的页面正常输出了，然后我们来看显示省略号的情况：

```javascript
// 第二种情况：显示省略号，当前页码前后最多显示2个页码
当前页码为 1，那么显示 1 2 3 ... 30
当前页码为 2，那么显示 1 2 3 4 ... 30
当前页码为 3，那么显示 1 2 3 4 5 ... 30
当前页码为 4，那么显示 1 2 3 4 5 6 ... 30
当前页码为 5，那么显示 1 ... 3 4 5 6 7 ... 30
...
当前页码为 15，那么显示 1 ... 13 14 15 16 17 ... 30
...
当前页码为 26，那么显示 1 ... 24 25 26 27 28 ... 30
当前页码为 27，那么显示 1 ... 25 26 27 28 29 30
当前页码为 28，那么显示 1 ... 26 27 28 29 30
当前页码为 29，那么显示 1 ... 27 28 29 30
当前页码为 30，那么显示 1 ... 28 29 30
```

同样需要完成showPages()函数：

```javascript
function showPages(page, length, show){
    var str = '';
    var preIndex = page - (show + 1);
    var aftIndex = page + (show + 1);
    if(page < show + 3){
        for(var i = 1; i <= show * 2 + 3; i++){
            if((i !== preIndex && i !== aftIndex) || (i === 1 || i=== total)){
                str = str + ' ' + i;
            } else {
                str = str + ' ... ' + total;
                break;
            }
        }
    } else if(page > total - (show + 2)){
        for(var i = total; i >= total - (show * 2 + 2); i--){
            if((i !== preIndex && i !== aftIndex) || (i === 1 || i === total)){
                str = i + ' ' + str;
            } else {
                str = '1 ... ' + str;
                break;
            }
        }
    } else {
        for(var i = preIndex + 1; i <= aftIndex - 1; i++){
            str = str + ' ' + i;
        }
        str = '1 ... ' + str + ' ... ' + total;
    }
    return str.trim();
}
```

同样也是采用分段拼的思路，能成功打印出结果：

![加入省略号的分页插件](https://user-gold-cdn.xitu.io/2018/7/26/164d43d7ecd072ca?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

但是仔细看看上面的代码会发现有大量冗余的逻辑了，能不能优化呢？下面是一种更为取巧的思路：

```javascript
function showPages(page, total, show){
    var str = page + '';
    for(var i = 1; i <= show; i++){
        if(page - i > 1){
            str = page - i + ' ' + str;
        }
        if(page + i < total){
            str = str + ' ' + (page + i);
        }
    }
    if(page - (show + 1) > 1){
        str = '... ' + str;
    }
    if(page > 1){
        str = 1 + ' ' + str;
    }
    if(page + show + 1 < total){
        str = str + ' ...';
    }
    if(page < total){
        str = str + ' ' + total;
    }
    return str;
}
```

打印结果是一样的，但代码却大为精简了。

## 基本架构

一个好的插件，代码一定是高复用、低耦合、易拓展的，因此我们需要采用面向对象的方法来搭建这个插件的基本架构：

```javascript
// 模仿jQuery $()
function $(selector, context){
    context = arguments.length > 1 ? context: document;
    return context ? context.querySelectorAll(selector) : null;
}

var Pagination = function(selector, pageOption){
    // 默认配置
    this.options = {
        curr: 1,
        pageShow: 2,
        ellipsis: true,
        hash: false
    };
    // 合并配置
    extend(this.options, pageOption, true);
    // 分页器元素
    this.pageElement = $(selector)[0];
    // 数据总数
    this.dataCount = this.options.count;
    // 当前页码
    this.pageNumber = this.options.curr;
    // 总页数
    this.pageCount = Math.ceil(this.options.count / this.options.limit);
    // 渲染
    this.renderPages();
    // 执行回调函数
    this.options.callback && this.options.callback({
        curr: this.pageNumber,
        limit: this.options.limit,
        isFirst: true
    });
    // 改变页数并触发事件
    this.changePage();
};

Pagination.prototype = {
    constructor: Pagination,
    changePage: function(){}
};

return Pagination;
```

如上，一个采用原型模式的分页器对象就搭建完成了，下面我们对上面的代码进行一一讲解。

### 分页配置

本分页器提供如下基本参数：

```javascript
// 分页元素ID（必填）
var selector = '#pagelist';

// 分页配置
var pageOption = {
    // 每页显示数据条数（必填）
    limit: 5,
    // 数据总数（一般通过后端获取，必填）
    count: 162,
    // 当前页码（选填，默认为1）
    curr: 1,
    // 是否显示省略号（选填，默认显示）
    ellipsis: true,
    // 当前页前后两边可显示的页码个数（选填，默认为2）
    pageShow: 2,
    // 开启location.hash，并自定义hash值（默认关闭）
    // 如果开启，在触发分页时，会自动对url追加：#!hash值={curr} 利用这个，可以在页面载入时就定位到指定页
    hash: false,
    // 页面加载后默认执行一次，然后当分页被切换时再次触发
    callback: function(obj){
        // obj.curr: 获取当前页码
        // obj.limit: 获取每页显示数据条数
        // obj.isFirst: 是否首次加载页面，一般用于初始加载的判断
        
        // 首次不执行
        if(!obj.isFirst){
            // do something
        }
    }
};
```

在构造函数里调用extend()完成了用户参数与插件默认参数的合并。

### 回调事件

通常情况下，在改变了插件状态后（点击事件等），插件需要作出一定的反应。因此我们需要对用户行为进行一定的监听，这种监听习惯上就叫做回调函数。在上面代码中我们可以看到有这么一段：

```javascript
// 执行回调函数
this.options.callback && this.options.callback({
    curr: this.pageNumber,
    limit: this.options.limit,
    isFirst: true
});
```

这种写法是不是有点器官，其实它相当于：

```javascript
if(this.options.callback){
    this.options.callback({
        curr: this.pageNumber,
        limit: this.options.limit,
        isFirst: true
    });
}
```

想必聪明的你已经明白了吧，这里的callback并不是某个具体的东西，而是一个引用。不管callback指向谁，我们只需要判断它有没有存在，如果存在就执行它。

## 事件绑定

接下来需要对分页器进行点击事件的绑定，也就是完成我们的changePage()方法：

```javascript
changePage: function(){
    var self = this;
    var pageElement = self.pageElement;
    EventUtil.addEvent(pageElement, "click", function(ev){
    	var e = ev || window.event;
        var target = e.target || e.srcElement;
        if(target.nodeName.toLocaleLowerCase() == "a"){
            if(target.id === "prev"){
                self.prevPage();
            } else if(target.id === "next"){
                self.nextPage();
            } else if(target.id === "first"){
                self.firstPage();
            } else if(target.id === "last"){
                self.lastPage();
            } else if(target.id === "page"){
                self.goPage(parseInt(target.innetHTML));
            } else {
                return;
            }
            self.renderPage();
            self.options.callback && self.options.callback({
                curr: self.pageNumber,
                limit: self.options.limit,
                isFirst: false
            });
            self.pageHash();
        }
    });
}
```

整体的逻辑大家应该都能轻松看懂，无非就是判断当前点击的是什么，然后执行对应的逻辑操作，但具体的实现方式有的同学可能会有一点陌生。

程式里面的target和srcElement是JavaScript事件委托方面的知识。

[js中的事件委托或是事件代理详解](https://www.cnblogs.com/liugang-vip/p/5616484.html)

插件对象、配置完成了，事件也绑定了，那接下来就应该完成我们页码上显示的DOM节点的渲染了。

## 渲染DOM

渲染的过程其实就是对上面我们封装的那几个字符打印函数的改进，把字符串改为具体的DOM节点，然后添加进页面即可。首先我们需要完成一个createHtml()函数：

```javascript
createHtml: function(elemDatas){
    var self = this;
    var fragment = document.createDocumentFragment();
    var liEle = document.createElement("li");
    var aEle = document.createElement("a");
    elemDatas.forEach(function(elementData, index){
        liEle = liEle.cloneNode(false);
        aEle = aEle.cloneNode(false);
        liEle.setAttribute("class", CLASS_NAME.ITEM);
        aEle.setAttribute("href", "javascript:;");
        aEle.setAttribute("id", elementData.id);
        if(elementData.id !== 'page'){
            aEle.setAttribute("class", CLASS_NAME.LINK);
        } else {
            aEle.setAttribute("class", elementData.className);
        }
        aEle.innerHTML = elementData.content;
        liEle.appendChild(aEle);
        fragment.appendChild(liEle);
    });
    return fragment;
}
```

这个函数的作用很简单，就是生成一个节点：

```html
<li class="pagination-item"><a href="javascript:;" id="page" class="pagination-link current">1</a></li>
```

代码中有涉及到两个性能优化的API，第一个API是document.createDocumentFragment()，它的作用是创建一个临时占位符，然后存放哪些需要插入的节点，可以有效避免页面进行DOM操作时的重绘和回流，减小页面的负担，提升页面性能。相关知识点，可参阅以下文章：

* [JS性能优化之创建文档碎片](https://www.cnblogs.com/leejersey/p/3516603.html)
* [前端性能优化第三篇-documentFragment](https://blog.csdn.net/github_39457740/article/details/80698584)

第二个API是cloneNode()，如果要创建很多元素，就可以利用这个API来减少属性的设置次数，不过必须先提前准备一个样板节点，例如：

```javascript
var frag = document.createDocumentFragment();
for (var i = 0; i < 1000; i++) {
    var el = document.createElement('p');
    el.innerHTML = i;
    frag.appendChild(el);
}
document.body.appendChild(frag);
//替换为：
var frag = document.createDocumentFragment();
var pEl = document.getElementsByTagName('p')[0];
for (var i = 0; i < 1000; i++) {
    var el = pEl.cloneNode(false);
    el.innerHTML = i;
    frag.appendChild(el);
}
document.body.appendChild(frag);
```

完成这个函数后，再进一步封装成两个插入节点的函数：（这一步可省略）

```javascript
addFragmentBefore: function(fragment, datas) {
  fragment.insertBefore(this.createHtml(datas), fragment.firstChild);
}

addFragmentAfter: function(fragment, datas) {
  fragment.appendChild(this.createHtml(datas));
}
```

前者是在最前插入节点，后者是在最后插入节点。一些常量和重复操作也可以进一步抽取：

```javascript
pageInfos: [{
    id: "first",
    content: "首页"
  },
  {
    id: "prev",
    content: "前一页"
  },
  {
    id: "next",
    content: "后一页"
  },
  {
    id: "last",
    content: "尾页"
  },
  {
    id: "",
    content: "..."
  }
]

getPageInfos: function(className, content) {
  return {
    id: "page",
    className: className,
    content: content
  };
}
```

利用上面封装好的对象和方法，我们就可以对最开始那两个字符串函数进行改造了：

```javascript
renderNoEllipsis: function(){
    var fragment = document.createDocumentFragment();
    if(this.pageNumber < this.options.pageShow + 1){
        fragment.appendChild(this.renderDom(1, this.options.pageShow * 2 + 1));
    } else if(this.pageNumber > this.pageCount - this.options.pageShow) {
        fragment.appendChild(this.renderDom(this.pageCount - this.options.pageShow * 2, this.pageCount));
    } else {
        fragment.appendChild(this.renderDom(this.pageNumber - this.options.pageShow, this.pageNumber + this.options.pageShow));
    }
    if(this.pageNumber > 1){
        this.addFragmentBefore(fragment, [
            this.pageInfos[0],
            this.pageInfos[1]
        ]);
    }
    if(this.pageNumber < this.pageCount){
        this.addFragmentAfter(fragment, [this.pageInfos[2], this.pageInfos[3]]);
    }
    return fragment;
}

renderEllipsis: function(){
    var fragment = document.createDocumentFragment();
    this.addFragmentAfter(fragment, [
        this.getPageInfos(CLASS_NAME.LINK + " current", this.pageNumber)
    ]);
    for(var i = 1; i <= this.options.pageShow; i+=){
        if(this.pageNumber - i > 1){
        	this.addFragmentBefore(fragment, [
            	this.getPageInfos(CLASS_NAME.LINK, this.pageNumber - i)
        	]);
    	}
    	if(this.pageNumber + i < this.pageCount){
        	this.addFragmentAfter(fragment, [
            	this.getPageInfos(CLASS_NAME.LINK, this.pageNumber + i);
        	]);
    	}
    }
	if(this.pageNumber - (this.options.pageShow + 1) > 1){
    	this.addFragmentBefore(fragment, [this.pageInfos[4]]);
	}
	if(this.pageNumber >1){
        this.addFragmentBefore(fragment, [
            this.pageInfos[0],
            this.pageInfos[1],
            this.getPageInfos(CLASS_NAME.LINK, 1)
        ]);
	}
	if(this.pageNumber + this.options.pageShow + 1 < this.pageCount){
    	this.addFragmentAfter(fragment, [this.pageInfos[4]]);
	}
	if(this.pageNumber <this.pageCount){
        this.addFragmentAfter(fragment, [
            this.getPageInfos(CLASS_NAME.LINK, this.pageCount),
            this.pageInfos[2],
            this.pageInfos[3]
        ]);
	}
	return fragment;
}

renderDom: function(begin, end){
    var fragment = document.createDocumentFragment();
    var str = "";
    for(var i = begin; i <= end; i++){
        str = this.pageNumber === i ? CLASS_NAME.LINK + " current" : CLASS_NAME.LINK;
        thia.addFragmentAfter(fragment, [this.getPageInfos(str, i)]);
    }
    return fragment;
}
```

逻辑和最开始的`showPages()`完全一样，只是变成了DOM的操作而已。

至此，渲染部分的函数基本也封装完成，最后还剩一些操作页码的函数，比较简单，这里就不作讲解了，可自行参考[源码](https://github.com/csdoker/csdwheels/blob/master/src/es5/pagination/pagination.js)。

## 使用场景

相信大家也看出来了，此分页器只负责分页本身的逻辑，具体的数据请求与渲染需要另外去完成。不过，此分页器不仅能应用咋一般的异步分页上，还可直接对一段已知数据进行分页展现，使用场景如下：

### 前端分页

在callback里对总数据进行处理，然后取出当前页需要展示的数据即可。

### 后端分页

利用url上的页码参数，可以在页面载入时就定位到指定页码，并且可以同时请求后端指定页码下对应的数据在callback回调函数里取得当前页码，可以使用window.location.href改变url，并将当前页码作为url参数，然后进行页面跳转，例如“./test.html?page=”。

## 插件调用

插件的调用也非常方便，首先，我们在页面引入相关的CSS、JS文件：

```html
<link rel="stylesheet" href="pagination.min.css">
<script type="text/javascript" src="pagination.min.js"></script>
```

然后将HTML结构插入文档中：

```html
<ol class="pagination" id="pagelist"></ol>
```

最后，将必填、选填的参数配置好即可完成本分页插件的初始化：

```javascript
// 分页元素ID（必填）
var selector = '#pagelist';

// 分页配置
var pageOption = {
  // 每页显示数据条数（必填）
  limit: 5,
  // 数据总数（一般通过后端获取，必填）
  count: 162,
  // 当前页码（选填，默认为1）
  curr: 1,
  // 是否显示省略号（选填，默认显示）
  ellipsis: true,
  // 当前页前后两边可显示的页码个数（选填，默认为2）
  pageShow: 2,
  // 开启location.hash，并自定义hash值 （默认关闭）
  // 如果开启，在触发分页时，会自动对url追加：#!hash值={curr} 利用这个，可以在页面载入时就定位到指定页
  hash: false,
  // 页面加载后默认执行一次，然后当分页被切换时再次触发
  callback: function(obj) {
    // obj.curr：获取当前页码
    // obj.limit：获取每页显示数据条数
    // obj.isFirst：是否首次加载页面，一般用于初始加载的判断

    // 首次不执行
    if (!obj.isFirst) {
      // do something
    }
  }
};

// 初始化分页器
new Pagination(selector, pageOption);
```

> 两种基础模式之上，还可以开启Hash模式

那么，整个分页器插件的封装到这里就全部讲解完毕了，怎么样，是不是觉得还挺简单？偷偷告诉你，接下来我们会逐渐尝试点更有难度的插件哦！

## ES6-环境配置

2015年，ECMAScript正是发布了它的新版本——ECMAScript6，对JavaScript语言本身来说，这是一次彻彻底底的升级。

经过这次更新，不仅修复了许多ES5时代留下来的“坑”，更是在原有的语法和规则上增加了不少强大的新特性，尽管目前浏览器对新规范支持得不是很完善，但经过一些神奇的工具处理后就能让浏览器“认识”这些新东西，并兼容它们了。

so，我们还有什么理由不使用强大的ES6呢？接下来就让我们先来看看这些神奇的工具是怎么使用的吧。

### Babel

首先，我们需要一个工具来转换ES6的代码，它的芳名叫Babel。Babel是一个编译器，负责将源代码转换成指定语法的目标代码，并使它们很好的执行在运行环境中，所以我们可以利用它来编译我们的ES6代码。

要使用Babel相关的功能，必须先用npm安装它们：

> npm i babel-cli babel-preset-env babel-core babel-loader babel-plugin-transform-runtime babel-polyfill babel-runtime -D

安装完成后，我们就可以手动使用命令编译某个目录下的js文件，并输出它们了。

But，这就是完美方案了吗？显然不是。

在实际的开发环境中，我们还需要考虑更多东西，比如模块化开发、自动编译和构建等，所以我们还需要一个更为强大的工具来升级我们的整套构建流程。

### Webpack

在前端框架以及工程化大行其道的今天，想必大家对Webpack、Gulp等工具并不会感到陌生，配合它们我们可以轻松实现一个大型前端应用的构建、打包、发布的过程。不过现在是2018年了，三大框架三足鼎立，而Gulp已经稍显老态，作为它的晚辈，一个名叫Webpack的少年正在逐渐崛起。这位少年，相信大家在使用Vue、React的过程中已经或多或少接触过它了。简而言之，它和Gulp在项目中的角色是一样的，只不过配置更为简单，构建更为高效，下面就让我们来看看Webpack是怎么使用的吧。

一般来说，我们需要安装它：

> npm i webpack webpack-cli webpack-dev-server -D

使用它也非常简单，只需要建立一个名叫webpack.config.js的配置文件即可：

```javascript
const path = require('path');

module.exports = {
  // 模式配置
  mode: 'development',
  // 入口文件
  entry: {},
  // 出口文件
  output: {},
  // 对应的插件
  plugins: [],
  // 处理对应模块
  module: {}
}
```

这个配置文件的主要部分有：入口、出口、插件、模块。在具体配置它们之前，我们先理一理我们项目的打包构建流程：

1. 寻找到./src/es6/目录下的index.js项目入口文件
2. 使用Babel编译它及它所引用的所有依赖（如Scss、css文件等）
3. 压缩编译完成后的js文件，配置为umd规范，重命名为csdwheeels.min.js
4. 清空dist-es6目录
5. 输出至dist-es6目录下

要使用清空目录、压缩代码、解析css等功能，我们还需要安装一下额外的包：

> npm i clean-webpack-plugin uglifyjs-webpack-plugin css-loader style-loader node-sass sass-loader

要在配置中让babel生效，还需要建立一个.babelrc文件，并在其中指定编码规则：

```javascript
{
  "presets": ["env"]
}
```

最后，我们就能完成这个配置文件了：

```javascript
const path = require('path');
const UglifyJsPlugin = require('uglifyjs-webpack-plugin');
const CleanWebpackPlugin = require('clean-webpack-plugin'); //每次构建清理dist目录

module.exports = {
  // 模式配置
  mode: 'development',
  // 入口文件
  entry: {
    pagination: './src/es6/index.js'
  },
  // 出口文件
  output: {
    path: path.resolve(__dirname, 'dist-es6'),
    filename: "csdwheels.min.js",
    libraryTarget: 'umd',
    library: 'csdwheels'
  },
  // 对应的插件
  plugins: [
    new CleanWebpackPlugin(['dist-es6']),
    new UglifyJsPlugin({
      test: /\.js($|\?)/i
    })
  ],
  // 开发服务器配置
  devServer: {},
  // 处理对应模块
  module: {
    rules: [
      {
        test: /\.js$/,
        include: path.join(__dirname , 'src/es6'),
        exclude: /node_modules/,
        use: ['babel-loader']
      },
      {
        test: /\.scss$/,
        use: [{
          loader: 'style-loader'
        }, {
          loader: 'css-loader'
        }, {
          loader: 'sass-loader'
        }]
      }
    ]
  }
}
```

光配置好还不够，我们总需要用命令来运行它吧，在package.json里配置：

```javascript
"scripts": {
  "test": "node test/test.js",
  "dev": "webpack-dev-server",
  "build": "webpack && gulp mini && npm run test"
}
```

这里使用dev可以启动一个服务器来展示项目，不过这里我们暂时不需要，而运行npm run build命令就饿可以同时将我们的./src/es6目录下的源码打包好输出到指定目录了。

ES6开发所需要的环境终于配置完成，接下来就让我们开始代码的重构吧！

## ES6-代码重构

> 如果你想要入门ES6，强烈推荐阮一峰老师的[教程](http://es6.ruanyifeng.com/)

> 相关的新语法和特性较多，不过我们的项目要重构为ES6暂时还用不了多少比较高级的特性，你只需要着重看完Class部分即可。

ES6引入的新特性中，最重要的一个就是Class了。有了它，我们不需要再像以前那样用构造函数去模拟面向对象的写法，因为它是JavaScript原生支持的一种面向对象的语法糖，虽然底层仍然是原型链，不过至少写出来的代码看上去像是那么一回事了。

拿前面提到的插件模板来说，ES5的时候我们是这样写的：

```javascript
(function(root, factory){
    if(typeof define === 'function' && define.amd){
        define([], factory);
    } else if(typeof module === 'object' && module.exports){
        module.exports = factory();
    } else {
        root.Plugin = factory();
    }
}(typeof self !== 'undefined' ? self : this, function(){
    'use strict';
    
    // tool
    function extend(o, n, override){
        for(var p in n){
            if(n.hasOwnProperty(p) && (!o.hasOwnProperty(p) || override))
                o[p] = n[p];
        }
    }
    
    // plugin construct function
    function Plugin(selector, userOptions){
        // Plugin() or new Plugin()
        if(!(this instanceof Plugin)) return new Plugin(selector, userOptions);
        this.init(selector, userOptions);
    }
    
    Plugin.prototype = {
        constructor: Plugin,
        // default: option
        options: {},
        init: function(selector, userOptions){
            extend(this.options, userOptions, true);
        }
    };
    
    return Plugin;
}));
```

经过Class这种新语法糖的改造后，它变成了下面这样：

```javascript
// ES6插件模板
Class Plugin{
    constructor(selector, options = {}){
        this.options = {};
        Object.assign(this.options, options);
        this.init(selector, options);
    }
    
    init(selector, options){}
}
export default Plugin;
```

改造后的代码，不仅在语法层面直接支持了构造函数的写法，更是去掉了IIFE这种臃肿的写法，可以说不管是看起来还是写起来都更为清晰流畅了。

> 利用内置的Object.assign()方法，可以直接替换掉我们实现的extend函数，功能可以说完全一样，而且更为强大

有了新的模板，我们就能直接开始插件代码的重构了，这里只贴上变动比较大的几个地方，其余部分可参考[源码](https://github.com/csdoker/csdwheels/blob/master/src/es6/pagination/pagination.js)

```javascript
import '../../../style/pagination/pagination.scss'

class Pagination {
  static CLASS_NAME = {
    ITEM: 'pagination-item',
    LINK: 'pagination-link'
  }

  static PAGE_INFOS = [{
      id: "first",
      content: "首页"
    },
    {
      id: "prev",
      content: "前一页"
    },
    {
      id: "next",
      content: "后一页"
    },
    {
      id: "last",
      content: "尾页"
    },
    {
      id: "",
      content: "..."
    }
  ]

  constructor(selector, options = {}) {
    // 默认配置
    this.options = {
      curr: 1,
      pageShow: 2,
      ellipsis: true,
      hash: false
    };
    Object.assign(this.options, options);
    this.init(selector);
  }

  changePage () {
    let pageElement = this.pageElement;
    this.addEvent(pageElement, "click", (ev) => {
      let e = ev || window.event;
      let target = e.target || e.srcElement;
      if (target.nodeName.toLocaleLowerCase() == "a") {
        if (target.id === "prev") {
          this.prevPage();
        } else if (target.id === "next") {
          this.nextPage();
        } else if (target.id === "first") {
          this.firstPage();
        } else if (target.id === "last") {
          this.lastPage();
        } else if (target.id === "page") {
          this.goPage(parseInt(target.innerHTML));
        } else {
          return;
        }
        this.renderPages();
        this.options.callback && this.options.callback({
          curr: this.pageNumber,
          limit: this.options.limit,
          isFirst: false
        });
        this.pageHash();
      }
    });
  }

  init(selector) {
    // 分页器元素
    this.pageElement = this.$(selector)[0];
    // 数据总数
    this.dataCount = this.options.count;
    // 当前页码
    this.pageNumber = this.options.curr;
    // 总页数
    this.pageCount = Math.ceil(this.options.count / this.options.limit);
    // 渲染
    this.renderPages();
    // 执行回调函数
    this.options.callback && this.options.callback({
      curr: this.pageNumber,
      limit: this.options.limit,
      isFirst: true
    });
    // 改变页数并触发事件
    this.changePage();
  }
}
export default Pagination;
```

总结起来，这次改造用到的语法就这么几点：

1. const、let替换var
2. 用constructor实现构造函数
3. 箭头函数替换function

除此之外，在安装了Sass的编译插件后，我们还能直接在这个js文件中把样式import进来，这样打包压缩后的js中也会包含进我们的样式代码，使用的时候就不需要额外再引入样式文件了。最后，由于ES6并不支持类的静态属性，所以还需要用到ES7新提案的static语法。我们可以安装对应的babel包：

> npm i babel-preset-stage-0 -D

安装后，在.babelrc文件中添加它即可：

```javascript
{
  "presets": ["env", "stage-0"]
}
```

现在万事俱备，你只需要运行npm run build，然后就可以看到我们打包完成后的csdwheels.min.js文件了。

打包后，我们还可以发布这个npm包，运行如下命令即可：

> npm login
>
> npm publish

要使用发布后的插件，只需要安装这个npm包，并import对应的插件：

> npm i csdwheels -D

```javascript
import { Pagination } from 'csdwheels';
```

## Vue插件版本

按照原定开发计划，其实是不想马上更新Vue版本的，毕竟这个系列的“卖点”是原生开发， 不过最近用Vue做的项目和自己的博客都恰好用到了分页这个组件，所以我决定一鼓作气把这个插件的Vue版本写出来，正好也利用这个机会学学Vue插件的开发。

### 开发规范

既然是框架，那肯定有它自己的开发规范了，类似于我们自己写的插件一眼，它也会给我们提供各式各样的API接口，让我们能定制自己的插件模块。简单来说，我们的插件在Vue中需要挂载到全局上，这样才能直接在任何地方引入插件：

```javascript
import Pagination from './component/vue-wheels-pagination';

const VueWheelsPagination = {
    install (Vue, options){
        Vue.component(Pagination.name, Pagination);
    }
};

if(typeof window !== 'undefined' && window.Vue){
    window.Vue.use(VueWheelsPagination);
}

export { VueWheelsPagination }
```

vue-wheels-pagination是我们即将要开发的单文件组件，引入后通过install方法把它挂载上去，然后再外部就可以use这个插件，最后导出这个挂载了我们插件的对象。（如果检测到浏览器环境后，可以直接挂载它）这差不多就是一个最简单的插件模板了，更详细的配置可参考[官方文档](https://cn.vuejs.org/v2/guide/plugins.html)。

将这个入口用Webpack打包后，就饿可以在你Vue项目中的main.js中全局加载这个插件了：

```javascript
import { VueWheelsPagination } from 'vue-wheels';
Vue.use( VueWheelsPagination );
```

接下来，就让我们来看看用Vue的方式是怎么完成这个分页插件的吧！

### DOM渲染

利用现代MVVM框架双向绑定的特性，我们已经不必再用原生JS的API去直接操作DOM了，取而代之的，可以在DOM结构上利用框架提供的API间接进行DOM的渲染及交互：

```vue
<template lang="html">
  <nav class="pagination">
    <a href="javascript:;" class="pagination-item first" @click="goFirst()" v-if="pageNumber > 1">{{info.firstInfo}}</a>
    <a href="javascript:;" class="pagination-item prev" @click="goPrev()" v-if="pageNumber > 1">{{info.prevInfo}}</a>
    <ul class="pagination-list" v-if="ellipsis">
      <li class="pagination-item" @click="goFirst()" v-if="pageNumber > 1">1</li>
      <li class="pagination-item ellipsis" v-if="pageNumber - (max + 1) > 1">...</li>
      <li class="pagination-item"
          @click="goPage(pageNumber - pageIndex)"
          v-if="pageNumber - pageIndex > 1"
          v-for="pageIndex in rPageData"
          :key="pageNumber - pageIndex">
        {{pageNumber - pageIndex}}
      </li>
      <li class="pagination-item current" @click="goPage(pageNumber)">{{pageNumber}}</li>
      <li class="pagination-item"
          @click="goPage(pageNumber + pageIndex)"
          v-if="pageNumber + pageIndex < pageCount"
          v-for="pageIndex in pageData"
          :key="pageNumber + pageIndex">
        {{pageNumber + pageIndex}}
      </li>
      <li class="pagination-item ellipsis" v-if="pageNumber + max + 1 < pageCount">...</li>
      <li class="pagination-item" @click="goLast()" v-if="pageNumber < pageCount">{{pageCount}}</li>
    </ul>
    <ul class="pagination-list" v-if="!ellipsis">
      <li :class="pageIndex === pageNumber ? 'pagination-item current' : 'pagination-item'"
          @click="goPage(pageIndex)"
          v-for="pageIndex in pageDataFront"
          v-if="pageNumber < max + 1"
          :key="pageIndex">
        {{pageIndex}}
      </li>
      <li :class="pageIndex === pageNumber ? 'pagination-item current' : 'pagination-item'"
          @click="goPage(pageIndex)"
          v-for="pageIndex in pageDataCenter"
          v-if="pageNumber > pageCount - max"
          :key="pageIndex">
        {{pageIndex}}
      </li>
      <li :class="pageIndex === pageNumber ? 'pagination-item current' : 'pagination-item'"
          @click="goPage(pageIndex)"
          v-for="pageIndex in pageDataBehind"
          v-if="max + 1 <= pageNumber && pageNumber <= pageCount - max"
          :key="pageIndex">
        {{pageIndex}}
      </li>
    </ul>
    <a href="javascript:;" class="pagination-item next" @click="goNext()" v-if="pageNumber < pageCount">{{info.nextInfo}}</a>
    <a href="javascript:;" class="pagination-item last" @click="goLast()" v-if="pageNumber < pageCount">{{info.lastInfo}}</a>
  </nav>
</template>
```

如上，我们直接在单文件组件的template标签中就完成了这个插件大部分的渲染逻辑。相对原生JS实现的版本，不仅轻松省去了事件监听、DOM操作等步骤，而且让我们能只关注插件本省具体的交互逻辑，可以说大大减轻了开发难度，并提升了页面性能。剩下的数据部分的逻辑及交互处理，在JS中完成即可。

### 交互逻辑

```javascript
export default {
    name: 'VueWheelsPagination',
    props: {
        count: {
            type: Number,
            required: true
        },
        limit: {
            type: Number,
            required: true
        },
        curr: {
            type: Number,
            required: false,
            default: 1
        },
        max: {
            type: Number,
            required: false,
            default: 2
        },
        ellipsis: {
            type: Boolean,
            required: false,
            default: true
        },
        info: {
            type: Object,
            required: false,
            default: {
                firstInfo: '首页',
                prevInfo: '前一页',
                nextInfo: '后一页',
                lastInfo: '尾页'
            }
        }
    },
    data(){
        return {
            pageNumber: this.curr;
        }
    },
    watch: {
        curr(newVal){
            this.pageNumber = newVal;
        }
    },
    computed: {
        pageData(){
            let pageData = [];
            for(let index = 1; index <= this.max, index++){
                pageData.push(index);
            }
            return pageData;
        },
        rPageData(){
            return this.pageData.slice(0).reverse();
        },
        pageDataFront(){
            let pageDataFront = [];
            for(let index = 1; index <= this.max * 2 + 1; index++){
                pageDataFront.push(index);
            }
            return pageDataFront;
        },
        pageDataCenter(){
            let pageDataCenter = [];
            for(let index = this.pageCount - this.max * 2; index <= this.pageCount; index++){
                pageDatacenter.push(index);
            }
            return pageDataCenter;
        },
        pageDataBehind(){
            let pageDataBehind = [];
            for(let index = this.pageNumber - this.max; index <= this.pageNumber + this.max; index++){
                pageDataBehind.push(index);
            }
            return pageDataBehind;
        },
        pageCount(){
            return Math.ceil(this.count / this.limit);
        }
    },
    methods: {
        goFirst(){
            this.pageNumber = 1;
            this.$emit('pageChange', 1);
        },
        goPrev(){
            this.pageNumber--;
            this.$emit('pageChange', this.pageNumber);
        },
        goPage(pageNumber){
            this.pageNumber = pageNumber;
            this.$emit('pageChange', this.pageNumber);
        },
        goNext(){
            this.pageNumber++;
            this.$emit('pageChange', this.pageNumber);
        },
        goLast(){
            this.pageNumber = this.pageCount;
            this.$emit('pageChange', this.pageNumber);
        }
    }
}
```

总体分成几个部分：

1. props属性中对父组件传递的参数进行类型、默认值、是否必填等配置的定义
2. 计算属性中对分页器本身所需数据进行初始化
3. 定义操作页码的方法，并向父组件传递当前页码
4. 在watch属性中监听页码的变化（主要应用于不通过分页而在其他地方改变页码的情况）

这样，整个分页插件的开发就完成了。相信大家可以感觉到，关于分页逻辑部分的代码量是明显减少了不少的，并且插件本身的逻辑也更清晰，和我们前面一步一步从底层实现起来的版本比较起来，更易拓展和维护了。

在外层的组件上调用起来大概就像这样：

```vue
<template>
  <div id="app">
    <div class="main">
      <vue-wheels-pagination @pageChange="change" :count="count" :limit="limit" :info="info"></vue-wheels-pagination>
    </div>
  </div>
</template>
```

```javascript
export default {
  name: 'app',
  data () {
    return {
      count: 162,
      limit: 5,
      info: {
        firstInfo: '<<',
        prevInfo: '<',
        nextInfo: '>',
        lastInfo: '>>'
      }
    }
  },
  methods: {
    change (pageNumber) {
      console.log(pageNumber)
    }
  }
}
```

传入必填和选填的参数，再监听到子组件冒泡出来的页码值，最后在你自己定义的change()方法里进行跳转等对应的逻辑处理就行了。

项目的打包流程和上一节提到的差不多，只不过在配置上额外增加了一个本地开发环境服务器的启动，可以参考我的[源码](https://github.com/csdoker/vue-wheels/blob/master/webpack.config.js)。打包完成后，同样可以发布一个npm包，然后就可以在任何Vue项目中引入并使用了。

>  后面开发的轮子不一定都会发布Vue版本，因为已经给大家提供了一种重构和包装插件的思路，如果你有自己的需求，可自行利用框架的规范进行插件开发。

## 参考内容

* [由匿名函数展开的一系列知识点](https://link.juejin.im?target=https%3A%2F%2Fblog.iihaiku.com%2F2016%2F11%2F23%2Fbasic-review-from-anonymous-function-to-iife)

* [自执行函数(IIFE)](https://link.juejin.im?target=https%3A%2F%2Fltaoo.github.io%2F2016%2F08%2F07%2Fimmediately-invoked-function-expression)

* [UMD (Universal Module Definition)](https://link.juejin.im?target=https%3A%2F%2Fgithub.com%2Fumdjs%2Fumd)

* [原生JavaScript插件编写指南](https://link.juejin.im?target=http%3A%2F%2Fgeocld.github.io%2F2016%2F03%2F10%2Fjavascript_plugin)

* [如何定义一个高逼格的原生JS插件](https://link.juejin.im?target=https%3A%2F%2Fwww.jianshu.com%2Fp%2Fe65c246beac1)

* [如何写一个简单的分页](https://link.juejin.im?target=http%3A%2F%2Ff2e.souche.com%2Fblog%2Fru-he-xie-ge-jian-dan-de-fen-ye)

* [我总结的js性能优化的小知识](https://link.juejin.im?target=http%3A%2F%2Fwww.cnblogs.com%2Fliyunhua%2Fp%2F4529086.html)

* [起步 | webpack 中文网](https://link.juejin.im?target=https%3A%2F%2Fwww.webpackjs.com%2Fguides%2Fgetting-started)

* [webpack 项目构建：（一）基本架构搭建](https://link.juejin.im?target=https%3A%2F%2Fsegmentfault.com%2Fa%2F1190000013512471)

* [webpack 项目构建：（二）ES6 编译环境搭建](https://link.juejin.im?target=https%3A%2F%2Fsegmentfault.com%2Fa%2F1190000013542132)

* [Vue-Guide-插件](https://link.juejin.im?target=https%3A%2F%2Fcn.vuejs.org%2Fv2%2Fguide%2Fplugins.html)

* [第一个Vue插件从封装到发布](https://link.juejin.im?target=https%3A%2F%2Fwww.cnblogs.com%2FshapeY%2Fp%2F7875930.html)

* [vue封装插件并发布到npm上](https://link.juejin.im?target=https%3A%2F%2Fblog.csdn.net%2Fapplechu_lu%2Farticle%2Fdetails%2F79329446)

* [vue封装第三方插件并发布到npm](https://link.juejin.im?target=https%3A%2F%2Fwww.cnblogs.com%2Fyesyes%2Fp%2F7588833.html)