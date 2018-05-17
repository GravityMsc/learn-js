## 0. 前言

你是不是在前端群见过很多这种场景：这个怎么做？怎么拿到这些数据？怎么更新整个列表？

回答：循环啊！遍历呀！用一个数组保存，遍历！jQuery！Vue！

然后有一些稍微高级的：我想快一点的解决办法。我想用性能好一点的方法。

回单：递归啊！开个新的数组保存中间变量，再遍历！document.querySelectorAll获取全部，缓存一下长度、所有的元素，遍历！快排，小的放左边大的放右边，递归！

然后当你发现水群是解决不了问题的时候，你已经迈出了进阶的一步了。在群里问，如果不是一堆熟人，都是陌生人的话，太简单人家嫌无脑，太难人家嫌麻烦或者压根不会，中等的稍微查一下资料也能做出来。所以说在一个全是陌生人的群，问别人不如自己动手，问了人家也可能是答非所问、要不就是暴力循环解决或者拿一些其他名词装逼（比如一些没什么知名度的js库，查了一下原理是某个小功能的）。

## 1. 我想给每一个li绑定事件，点击哪个打印相应的编号

某路人：循环，给每一个li绑一个事件。大概是这样子：

html：

```html
<ul>
    <li>0</li>
    <li>1</li>
    <li>2</li>
</ul>
```

js：

```javascript
var li = document.querySelectorAll('li');
for(var i = 0; i < li.length; i++){ //大约有一半的人连个length都不缓存的
    li[i].onclick = function(i){	// 然后再顺便扯一波闭包，扯一下经典面试题，再补一句let能秒杀
        return function(){
            console.log(i);
        }
    }(i);
}
```

问题少年：那我动态添加li呢？

某路人：一样啊，你加多少个，我就循环遍历多少个。

问题少年：加入我有一个按钮，按了增加一个li，也要实现这个效果，怎么办？

某路人：哈？一样啊，就是在新增的时候再for循环重新绑事件。

问题少年：

场景还原：

html：

```html
<ul>
    <li>0</li>
    <li>1</li>
    <li>2</li>
</ul>
<button onclick="addli()">
    
</button>
```

js：

```javascript
// 原来的基础上再多一个函数，然后热心地写下友好的注释
function addli(){
    var newli = document.createElement('li');
    newli,innerHTML = document.querySelectorAll('li').length;	// 先获取长度，把序号写进去
    document.querySelectorAll('ul')[0].appendChild(newli);	// 加入
    var li = document.querySelectorAll('li');
    for(var i = 0; i < li.length; i++){ // 再绑依次事件
        li[i].onclick = function(){
            return function(){
                console.log(i);
            };
        }(i);
    }
}
```

问题少年看见功能是实现了，但是他总觉得那么多document是有问题的，感觉浏览器很不舒服，而且也听闻过“操作dom是很大开销的”。于是他翻了一下资料，了解了一下面向对象、事件代码，通宵达旦写出自己的代码。

html：

```html
<input type="number" id="input">
<button onclick="addli()">
    add
</button>
```

js：

```javascript
class Ul{
    constructor(){
        this.bindEvent = this.bindEvent();	// 缓存合适的dom n事件
        this.ul = document.createElement('ul');
        this.bindEvent(this.ul, 'click', (e) => {
            console.log(e.target.innerHTML);
        });
        this.init = this.init();	// 单例模式，只能添加一个ul
        this.init(this.ul);
        this.counts = 0;	// li个数
    }
    
    bindEvent(){
        if(window.addEventListener){
            return function(ele, event, handle, isBunble){
                isBunble = isBunble || false;
                ele.addEventListener(event, handle, isBunble);
            }
        } else if(window.attachEvent){
            return function(ele, event, handle){
                ele.attachEvent('on' + event, handle);
            }
        } else {
            return function(ele, event, handle){
                ele['on' + event] = handle;
            }
        }
    }
    
    appendLi(nodeCounts){
        this.counts += nodeCounts;
        var temp = '';
        while(nodeCounts){
            temp += `<li>${this.counts - nodeCounts + 1}</li>`;
            nodeCounts--;
        }
        this.ul.innerHTML += temp;
    }
    
    init(){ // 单例模式
        var isappend = false;
        return function(node){
            if(!isappend){
                document.body.appendChild(node);
                isappend = true;
            }
        }
    }
}

var ul = new Ul();
function addli(){
    if(!input.value){
        ul.appendLi(1);
    } else {
        ul.appendLi(+input.value);
        input.value = '';
    }
}
```

## 2. 我想让几个div类似于checkbox的效果

也就是，几个div，点哪个亮哪个，点另一个，正在亮着的div马上暗，新点击的那个亮起来。

某路人：每一个div有两个类，click类表示被点。每次点击一个div，循环遍历全部div重置状态为test类，然后把被点击的那个变成click。于是乎，很快写出了一段代码：

html：

```html
<script src="https://cdn.bootcss.com/jquery/3.2.1/jquery.min.js"></script>
<div class="test"></div>
<div class="test"></div>
<div class="test"></div>
```

css：

```css
.test{
    width: 100px;
    height: 100px;
    background-color: #0f0;
    margin: 10px;
}
.click{
    background-color: #f00;
}
```

js：

```javascript
var divs = $('.test');
divs.click(function(){
    divs.each(function(){
        this.className = 'test';
    });
    this.className = 'click';
});
```

问题少年反问：我不知道什么是jQuery，但是我觉得应该这样子写：

```javascript
var divs = document.querySelectorAll('.test');
window.onclick = (function(){
    var pre;
    return function(e){
        if(pre !== undefined){
            divs[pre].className = 'test';
        }
        pre = Array.prototype.indexOf.call(divs, e.target);
        e.target.className = 'click';
    }
})();
```

某路人：我不知道你绑个事件搞个IIFE干啥，而且我建议你去学一下jQuery。

问题少年（心想）：明明console测试的运行时间，他的平均在0.26ms左右，我这个方法就是0.12ms左右，为什么一定要用jQuery？

## 3. 从1000到5000中取出全部每一位数字的和为5的数

问题少年：rt，求一个快一点的方法。

路人甲：

```javascript
Array(4000).fill(1000).map((v,i) => v + i)
    .filter(n => (n + "").split("")
	.reduce(((s, x) => +x+s), 0) == 5);
```

吃瓜群众：哇，大神，方法太简单了，通俗易懂。

路人乙：

```javascript
let temp = [];
for(let i = 1000; i< 5000; i++){
    temp.push(i.toString().split('').map(x => +x).reduce((a, b) => a + b, 0) === 5&i);
}
console.log(temp.filter(x => x !== false));
```

道高一尺魔高一丈，接着又来了一种通用的方法：

```javascript
var f = (s, l) => --l ? Array(s+1).fill(0).map((_,i)=> f(s-i,l).map(v =>i+v)).reduce((x,s)=>[...s,...x],[]):[[s]];
f(5, 4).filter(s=>s[0]>0);
```

估计问题少年也没有完全满足，虽然答案是简短的es6和api的灵活运用，但是方法还是有点简单无脑，做了多余的循环。

稍微优化一下的版本，循环次数只是缩减到600多次：（抛开ES6装逼的光环）

```javascript
function f(_from, _to, _n){
    var arr = [];
    var _to = _to + '';
    var len = _to.length;
    !function q(n, str){
        if(str.length === len){	// 保证4位数
            if(str.split('').reduce((s, x) => +x+s, 0) == _n && str[0] != '0'){ // 每一位加起来等于5
     			arr.push(+str);           
            }
            return;
        }
        for(var i = 0; i < _n - n; i++){	// 分别是0、1、2、3、4、5开头的
            q(i, i + '' + str);	//一位一位地递归
        }
    }(~~_from/1000, '');
    return arr;
}
f(1000, 5000, 5);
```

可读性显然查了很多，但是运行时间就差太多了：

* 一行ES6：16.3310546875ms 
* for循环：40.275146484375ms 
* 优化版本: 1.171826171875ms

## 4. 我从接口拿到了返回的json数据，但是我又要操作这个数据而且不能污染原数据

