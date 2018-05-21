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

某人不假思索秒回：var data2 = data

另一个路人稍加思考：不是吧，你先要深拷贝一下。

问题少年：那怎么深拷贝呢

某路人：JSON.stringify再JSON.parse

问题少年：谢啦，真好用。

稍微了解的人都知道，一个stringify并不能解决所有的深拷贝问题。第二天，问题少年又来了：我用了转义来拷贝，老板说要弄死我，现在项目里面所有的面向对象都炸了。

某路人：我曹，我就不信还有什么是stringify和parse解决不了的。

另一个路人：原型链断了、undefined变成空值、环引用出错，用了面向对象的全部泡汤了。

问题少年特别无助，也没有人出来帮忙，也许上天有一个稍微好一点点的参考答案，等着他的发掘：

```javascript
function copy(arr){
    var temp;
    if(typeof arr === 'number' || typeof arr === 'boolean' || typeof arr === 'string'){
        return arr;
    }
    if(Object.prototype.toString.call(arr) === "[object Array]"){
        temp = [];
        for(x in arr){
            temp[x] = copy(arr[x]);
        }
        return temp;
    } else if(Object.prototype.toString.call(arr) === "[object RegExp]"){
        temp = arr.valueOf();
        var str = (temp.global ? 'g' : '') + (temp.ignoreCase ? 'i' : '') + (temp.multiline ? 'm' : '');
        return new RegExp(arr.valueOf().source, str);
    } else if(Object.prototype.toString.call(arr) === "[object Function]"){
        var str = arr.toString();
        /^function\s*\w*\s*\(.*\)\s*\{(.*)/m.test(str);
        var str1 = RegExp.$1.slice(0, -1);
        return new Function(str1);	// 函数有换行就出事，求更好的解决办法
    } else if(Object.prototype.toString.call(arr) === "[object Date]"){
        return new Date(arr.valueOf());
    } else if(Object.prototype.toString.call(arr) === "[object Object]"){
        try{
            temp = JSON.parse(JSON.stringify(arr));
        } catch(e){ //环引用解决：取出环引用部分用stringify再放回去
            var temp1 = {},
                circle,
                result,
                reset = false,
                hash;
            function traverse(obj){
                for(x in obj){
                    if(!reset && obj.hasOwnProperty(x)){
                        if(!temp1[x]){
                            temp1[x] = obj[x];
                        } else if(typeof obj[x] === 'object' && typeof temp1[x] === 'object'){
                            try{
                                JSON.stringify(obj[x]);
                            } catch(e){
                                circle = obj[x];
                                hash = new Date().getTime();
                                obj[x] = hash;
                                break;
                            } finally{
                                return traverse(obj[x]);
                            }
                        }
                        if(typeof obj[x] === 'object'){
                            return traverse(obj[x]);
                        }
                    } else if(reset){
                        if(obj[x] === hash){
                            obj[x] = circle;
                            return;
                        }
                        if(typeof obj[x] === 'object'){
                            return traverse(obj[x]);
                        }
                    }
                }
            }
            traverse(arr);
            result = JSON.parse(JSON.stringify(arr));
            reset = true;
            traverse(result);
            traverse(arr);
            temp = result;
        } finally{ // 考虑到原型链和Object.create(null)
            if(arr.__proto__.constructor && arr.__proto__.constructor !== Object){
                temp.__proto__.constructor = arr.__proto__.cosntructor;
            }
            if(!arr.__proto__.constructor){
                temp.__proto__.constructor = null;
            }
            return temp;
        }
    }
    if(!arr){
        return arr;
    }
}
```

## 5. 在数组内动态添加元素，打钩的求和

给出的图片大概是这样的，选取某个li就把它的价格算入sum里面：

![数组示例的图片](https://user-gold-cdn.xitu.io/2018/5/3/16324fe10ad28439?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

相信50%的人都会这样子，某路人：vue，v-for显示，computed。

甚至有的人2分钟把代码撸出来了：

html:

```html
<script src="https://cdn.bootcss.com/vue/2.5.13/vue.min.js"></script>
<div id="app">
    <ul>
        <li v-for="x in list">
        	{{x.price}}
            <input type="checkbox" v-model="x.selected">
        </li>
    </ul>
    <button @click="add">
        add
    </button>
</div>
```

js：

```javascript
new Vue({
    el: '#app',
    data(){
        return {
            list:[{price: 1, selected: false},
                  {price: 2, selected: false},
                  {price: 3, selected: false}]
        }
    },
    methods: {
        add(){
            this.list.push({price: 1, selected: false});
        }
    },
    computed: {
        sum(){
            return this.list.reduce((s, x) => x.selected ? +x.price +s : s, 0);
        }
    }
})
```

问题少年：我们项目不用Vue。

我们项目这个模块就只用面向对象，工具库偶尔用用lodash。

另外lodash深拷贝，目前算是最完美的深拷贝。

我们可以猜想到问题少年想写的代码：

html：

```html
<ul id="app">
    
</ul>
```

js：

```javascript
app.addEventListener('change', (function(){
	var lastPick = 0;
    return function(e){ //相信很多人是每次change后，用循环一个个加起来的
        lastPick += e.target.checked? +e.target.value : -e.target.value;
        res.innerHTML = lastPick;
    }
})());
var list = [{price: 1, selected: false},
                  {price: 2, selected: false},
                  {price: 3, selected: false}];
var append = '';
for(var i = 0; i < list.length; i++){
    append += '<li><input type="checkbox" value=' + list[i].price + '>' + list[i].price + '</li>';
}
app.innerHTML = append;
```

## 6. 后端不帮我分页，前端分页怎么容易一点

问题少年：是个人中心来的，数据不多，而且用户一般都会一页页去浏览全部数据的，因为这些消费数据必须全部看一遍才能了解情况。

路人甲：拿到全部数据后，根据每页数据和第几页for循环取出元素并插入相应的html，每次切换也遍历一次。

路人乙：卧槽，这后端吃屎的吧，居然不分页。

路人丙：当然是按需加载啊，你不应该一下加载完全部。

路人丁：用jQuery分页插件啊。

于是勤劳热心爱解答问题的群友马上写出了答案：

html：

```html
<select id="app">
    <option>2</option>
    <option>3</option>
    <option>4</option>
</select>
<div>
    <button onclick="pre()">上一页</button>
    <button onclick="next()">下一页</button>
    <p>当前是<span id="current"></span>页</p>
    <table>
      	<tbody id="content">
      	</tbody>
      </table>
</div>
```

js：

```javascript
var data = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13]	// 后端拿到的数据
var currentIndex = 1;
var per = 2;
app.onchange = function(e){
    per = e.target.value;
    update(1);	//更改每页数据时候返回第一页
}
function pre(){
    currentIndex = currentIndex > 1 ? currentIndex -1 : currentIndex;
    update(currentIndex, per);
}
function next(){
    currentIndex = currentIndex < Math.floor(data.length / per) + 1 ? currentIndex + 1 : currentIndex;
    update(currentIndex, per);
}
window.onload = function(){
    update();
}
function update(currentPage, perPage){
    var currentPage = currentPage || currentIndex;
    var perPage = perPage || per;
    var ctx = '';
    for(var i = perPage * (currentPage - 1); i < perPage * currentPage; i++){
        if(data[i] !== undefined) ctx += '<tr><td>' + data[i] + '</td></tr>';
    }
    content.innerHTML = ctx;
    content.innerHTML = currentPage;
}
```

当然，后端不帮忙分页的情况下，通常数据量是不大的，上面的做法也是没什么问题。但是，在这个场景下，用户需要浏览器所有的数据，所以不存在什么按需分页。我们不应该在用户需要的时候才循环的，这样子对于用户，时间复杂度是n。如果是先分页再直接拿出来，复杂度就是1。这个场景下，他是全部都浏览的，所以我们先分页再取数据。改进的js：

```javascript
var data = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13];

var currentIndex = 1;
var per = 2;
var list = [];
app.onchange = function(e){
    per = +e.target.value;
    getdatalist(per);
    update(1);
}
function pre(){
    console.time('pre');
    currentIndex = currentIndex > 1 ? currentIndex - 1 : currentIndex;
    update(currentIndex, per);
    console.timeEnd('pre');
}
function next(){
    console.time('next');
    currentIndex = currentIndex < Math.floor(data.length / per) + 1 ? currentIndex + 1 : currentIndex;
    update(currentIndex, per);
    console.timeEnd('next');
}
window.onload = function(){
    getdatalist(2);
    update();
}
function getdatalist(per){
    var per = per || 2;
    var list = [];
    var pages = ~~(data.length/per) + 1;
    for(var i = 0; i < pages; i++){
        var temp = '';
        data.slice(i * per, i * per + per).forEach(function(item){
            temp += '<tr><td>' + item + '</td></tr>';
        });
        list.push(temp);
    }
}
function update(currentPage, perPage){
    currentIndex = currentPage || 1;
    var perPage = perPage || per;
    content.innerHTML = list[currentIndex - 1];
    current.innerHTML = currentIndex;
}
```

时间测试：

用户切换页面的时候时间消耗：

按需循环：

![按需循环](https://user-gold-cdn.xitu.io/2018/5/3/16324fe10ca86e2c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

提前分页：

![提前分页](https://user-gold-cdn.xitu.io/2018/5/3/16324fe10cbde57b?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

随着分页越来越多，提前分页在切换的时间上的优势越来越大。当然，正常的情况下用户一般都不会把全部数据都浏览完的，所以一般也是用按需分页更好。

## 7. 我们要做一个抽奖活动，需要用户的号码存在两个数和为100算中奖

问题少年：随机数字分布得比较均匀（但是乱序），比如3、2、1、4、5、7，而不是5、1、6、7、8

路人甲：一个个循环，再判断。

```javascript
let arr = [12, 40, 60, 80, 62, 13, 58, 87];
let obj = {};
arr.map(item => {
    if(!obj[item]){
        let val = 100 - item;
        obj[item] = 1;
        if(arr.includes(val)){
            obj[val] = 1;
            console.log(item, val);
        }
    }
});
```

可是，这ES6用得有点浪费了。问题少年貌似不满足，说：我们的数据量可能有一点大。

于是，有一个双指针版本。

```javascript
function f(arr, res){
    var l = arr.length;
    var p = ~~(l/2);
    var i = p - 1;
    var j = p;
    var ishasresult = false;
    arr = arr.sort((a, b) => a - b);
    while(i >= 0 && j < l){
        if(arr[i] + arr[j] > res){
            i--;
        } else if(arr[i] + arr[j] < res){
            j++;
        } else {
            ishasresult = true;
            break;
        }
    }
    return ishasresult;
}
```

假设排序为快排，总的时间复杂度就是nlogn + logn。

其实，用ES6的话，这样子更快：

```javascript
var mySet = new Set(arr);
arr.find(function(value, index, arr){
    return mySet.has(100 - value);
});
```

时间测试：

无脑ES6循环：2.419189453125ms    2.35595703125ms   1.330078125ms

双指针： 0.0400390625ms   0.0283203125ms  0.0390625ms

简化ES6：0.1240234375ms    0.10986328125ms    0.092041015625ms

## 8. 数组向头部添加元素，concat和一个个unshift哪个效率高

路人甲：unshift，毕竟它是专门在头部添加的，concat是连接数组的，算法肯定比unshift复杂，es6...（扩展运算符）算是淘汰了concat了吧。

测试结果：

![unshift和concat的比对测试](https://user-gold-cdn.xitu.io/2018/5/3/16324fe10ca7d646?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

很明显的，unshift和扩展运算符是一个个遍历，concat就是直接连起来复杂度为1。但是还是需要看实际场景的，没有绝对的淘汰、取代的这种说法。

## 总结

是不是道出了一些熟悉的经历？循环啊、遍历，再XXX，用Vue啊，用jQuery的xx，用xx插件。对于问问题的人，要先表示清楚需求，尽量讲详细一点，而不是随便截个图就问，能百度能靠文档的，也没有必要问。如果是有意义的问题，那么大家就得好好思考，了解人家的应用场景，而不是无脑循环，也不是直接抛一个xx插件、xx.js给人家，因为人家月懂的，只是想要一个更好的答案或者不是一个无脑的答案。当然，纯小白的话就算了。

如果是大牛，也许朋友圈子里面没有这种事情，也没有进这种群。我也是学了半年的菜鸟，很多应用场景的经验不足。但是有的人，工作了几年还写出一些无脑的代码，代码中又暴露了各种细节没怎么处理。不是说我觉得他们不如我，我只是感觉，这可能就是一些人工作10年1年经验的原因。

另外，面试的时候，是不是经常被问闭包拿来干什么的，上面例子有几个经典的闭包应用所以面试的时候不要只是说出闭包是什么、闭包会内存泄漏，也要知道，闭包用于干什么，无非就是缓存、柯里化、单例模式、模块化，而且能保护内部变量。那么，也问一下自己，究竟有没有用过闭包来干一些有意义的事情，有没有说过“无缘无故搞个IIFE有什么用”这种话？你的ES6仅仅是不是拿来简写或者追求视觉上的代码剪短来装逼的呢，有没有把proxy玩得飞起，解构赋值用的多吗，有没有知道不能捕获异步的try可以考async + await捕捉异步，set用过最多的是不是去重，常用的除了let、const、promise、async-await、...、set、反引号+字符串模板之后是不是没有其他的了？

