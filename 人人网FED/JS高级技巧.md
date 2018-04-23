本篇是看的《JS高级程序设计》第23章《高级技巧》做的读书分享。本篇按照书里的思路根据自己的理解和经验，进行扩展延伸，同时指出书里的一些问题。将会讨论安全的类型检测、惰性载入函数、冻结对象、定时器等话题。

## 1. 安全的类型检测

这个问题是怎么安全地检测一个变量的类型， 例如判断一个变量是否为一个数组。通常的作答是使用instanceof，如下代码所示。

```javascript
let data = [1, 2, 3];
console.log(data instanceof Array); // true
```

但是上面的判断在一定条件下会失败——就是咋ifame里面判断一个父窗口的变量的时候。写个demo验证一下，如下主页面的main.html。

```html
<script>
    window.global = {
        arrayData: [1, 2, 3]
    };
    console.log("parent arrayData installof Array: " + 
               (window.global.arrayData instanceof Array));
</script>
<iframe src="iframe.html">
    
</iframe>
```

在iframe.html判断一下父窗口的变量类型：

```html
<script>
    console.log("iframe window.parent.global.arrayData instanceof Array: " + 
        (window.parent.global.arrayData instanceof Array));
</script>
```

在iframe里面使用window.parent得到父窗口的全局window对象，这个不管跨不跨域都没有问题，进而可以得到父窗口的变量，然后用instanceof判断，最后运行结果如下：

![iframe的instanceof的运行结果](https://user-gold-cdn.xitu.io/2017/9/3/2e2c9ea215b90395fc2b24ebef894a3b?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

可以看到父窗口的判断是正确的，而子窗口的判断是false，因此一个变量明明是Array，但却不是Array，这是为什么呢？既然这个是父子窗口才有的问题，于是试一下把Array改成父窗口的Array，即window.parent.Array，如下图所示：

![父子窗口的判断](https://user-gold-cdn.xitu.io/2017/9/3/7c8beec0e2045b960ff97037d1b9fa82?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

这次返回了true，然后再变换一下其他的判断，如上图，最后可以知道根本原因是上图最后一个判断：

```javascript
Array !== window.parent.Array
```

它们分别是两个函数，父窗口定义了一个，子窗口又定义了一个，内存地址不一样，内存地址不一样的Object等式判断不成立，而window.parent.arrayData.constructor返回的是父窗口的Array，比较的时候是在子窗口，使用的是子窗口的Array，这两个Array不相等，所以导致判断不成立。

那怎么办呢？

由于不能使用Object的内存地址判断，可以使用字符串的形式，因为字符串是基本类型，字符串比较只要每个字符都相等就好了。ES5提供了这个一个方法Object.prototype.toString，我们先小试牛刀，试一下不同变量的返回值：

![toString的用法](https://user-gold-cdn.xitu.io/2017/9/3/e6aec81a7106fff0bf6c3a9dec6d351e?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

可以看到如果是数组返回“[object Array]”，[ES5对这个函数](http://ecma-international.org/ecma-262/5.1/#sec-15.2.4.2)是这么规定的：

![ES5对toString函数的固定](https://user-gold-cdn.xitu.io/2017/9/3/543fbbb30447f0451fc31c65438a08e6?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

也就是说这个函数的返回值是“[object”开头，后面带上变量类型的名称和右括号。因此既然它是一个标准语法规范，所以可以用这个函数安全地判断变量是不是数组。

可以这么写：

```javascript
Object.prototype.toString.call([1, 2, 3]) === "[object Array]"
```

注意要使用call，而不是直接调用，call的第一个参数是context执行上下文，把数组传给它作为执行上下文。

有一个比较有趣的现象是ES6的class也是返回function。

![function的toString](https://user-gold-cdn.xitu.io/2017/9/3/1807983fe0ec8695051d59e47aada38c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

所以可以知道class也是用function实现的原型，也就是说class和function本质上是一样的，只是写法上不一样。

那是不是说不能再使用instanceof判断变量类型了？不是的，当你需要检测父页面的变量类型就得使用这种方法，本页面的变量还是可以使用instanceof或者constructor的方法判断，只要你确保这个变量不会跨页面。因为对于大多数人来说，很少会写iframe的代码，所以没有必要搞一个比较麻烦的方式，还是用简单的方式就好了。

## 2. 惰性载入函数

有时候需要在代码里面做一些兼容性判断，或者是做一些UA的判断，如下代码所示：

```javascript
// UA的类型
getUAType: function(){
    let ua = window.navigator.userAfent;
    if(ua.match(/renren/i)){
        return 0;
    }
    else if(ua.match(/MicroMessenger/i)){
        return 1;
    }
    else if(ua.match(/weibo/i)){
        return 2;
    }
    return -1;
}
```

这个函数的作用是判断用户是在哪个环境打开的网页，以便于统计哪个渠道的效果比较好。

这种类型的判断都有一个特点，就是它的结果是死的，不管执行按断多少次，都会返回相同的结果，例如用户的UA在这个网页不可能会发生变化（除了调试设定的之外）。所以为了优化，才有了惰性战术一说，上面的代码可以改成：

```javascript
// UA的类型
getUAType: function(){
    let ua = window.navigator.userAgent;
    if(ua.match(/renren/i)){
        pageData.getUAType = () => 0;
        return 0;
    } 
    else if(ua.match(/MicroMessenger/i)){
        pageData.getUAType = () => 1;
        return 1;
    }
    else if(ua.match(/weibo/i)){
        pageData.getUAType = () => 2;
        return 2;
    }
    return -1;
}
```

在每次判断之后，把getUAType这个函数重新赋值，变成一个新的function，而这个function直接返回一个确定的变量，这样以后的每次获取都不用再判断了，这就是惰性函数的作用。你可能会说这个几个判断能优化多少时间呢，这么点时间对于用户来说几乎是没有区别的呀。确实如此，但是作为一个有追求的码农，还是会想办法尽可能优化自己的代码，而不是只是为了完成需求完成功能。并且当你的这些优化累积到一个量的时候就会发生质变。我上大学的时候C++的老师举了一个例子，说有个系统比较慢找她去看一下，其中她做的一个优化就是把小数的双精度改成单精度，最后是快了不少。

但其实上面的例子我们有一个更简单的实现，那就是直接搞个变量存起来就好了：

```javascript
let ua = window.navigator.userAgent;
let UAType = ua.match(/renren/i) ? 0 :
				ua.match(/MicroMessenger/i/) ? 1:
                 ua.match(/weibo/i)? 2 : -1;
```

连函数都不用写了，缺点是即使没有使用到UAType这个变量，也会执行一次判断，但是我们认为这个变量被用到的概率还是很高的。

我们再举一个比较有用的例子，由于Safari的无痕浏览会禁掉本地存储，因此需要搞一个兼容性判断：

```javascript
Data.localStorageEnabled = true;
// Safari的无痕浏览会禁用localStorage
try{
    window.localStorage.trySetData = 1;
} catch(e){
    Data.localStorageEnabled = false;
}

setLocalData: function(key, value){
    if(Data.localStorageEnabled){
        window.localStorage[key] = value;
    }
    else {
        util.setCookie('_L_' + key, value, 1000);
    }
}
```

在设置本地数据的时候，需要判断一下是不是支持本地存储，如果是的话就一共localStorage，否则改用cookie。可以用惰性函数改造一下：

```javascript
setLocalData: function(key, value) {
    if(Data.localStorageEnabled) {
        util.setLocalData = function(key, value){
            return window.localStorage[key];
        }
    } else {
        util.setLocalData = function(key, value){
            return util.getCookie("_L_" + key);
        }
    }
    return util.setLocalData(key, value);
}
```

这里可以减少一次if/else的判断，但好像不是特别实惠，毕竟为了减少一次判断，引入了一个惰性函数的概念，所以你可能要权衡一下这种引入是否值得，如果有三五个判断应该还是比较好的。

## 3. 函数绑定

有时候要把一个函数当参数传递给另一个函数执行，此时函数的执行上下文往往会发生变化，如下代码：

```javascript
class DrawTool {
    constructor(){
        this.points = [];
    }
    handlerMouseClick(event){
        this.points.push(event.latLng);
    }
    init(){
        $map.on('click', this.handleMouseClick);
    }
}
```

click事件的执行回调里面this不是指向了DrawTool的实例了，所以里面的this.points将会返回undefined。第一种解决方法是使用闭包，先把this缓存一下，编程that：

```javascript
class DrawTool{
    constructor(){
        this.points = [];
    }
    handleMouseClick(event){
        this.points.push((event.latLng));
    }
    init(){
        let that = this;
        $map.on('click', event => that.handleMouseClick(event));
    }
}
```

由于回调函数是用that执行的，而that是指向DrawTool的实例，因此就没有问题了。相反如果没有that它就用this，那么就要看this指向哪里了。

因为我们用了箭头函数，而箭头函数的this还是指向父级的上下文，因此这里不用自己创建一个闭包，直接用this就可以：

```javascript
init(){
    $map.on('click', event => this.handleMouseClick(event));
}
```

这种方式更加简单，第二种方法是使用ES5的bind函数绑定，如下代码：

```javascript
init(){
    $map.on('click', this.handleMouseClick.bind(this));
}
```

这个bind看起来好像很神奇，但其实只要一行代码就可以实现一个bind函数：

```javascript
Function.prototype.bind = function(context){
    return () => this.call(context);
}
```

就是返回一个函数，这个函数的this是指向的原始函数，然后让它call(context)绑定一下执行上下文就可以了。

## 4. 柯里化

柯里化就是函数和参数值结合产生一个新的函数，如下代码，假设有一个curry的函数：

```javascript
function add(a, b){
    return a + b;
}

let add1 = add.curry(1);
console.log(add1(5));	// 6
console.log(add1(2));	// 3
```

怎么实现这样一个curry的函数？它的重点是要返回一个函数，这个函数有一些闭包的变量记录了创建时的默认参数，然后执行这个返回函数的时候，把新传进来的参数和默认参数拼成完整参数列表去调用原本的函数，所以就有了以下代码：

```javascript
Function.prototype.curry = function(){
    let defaultArgs = arguments;
    let that = this;
    return function(){
        return that.apply(this, defaultArgs.concat(arguments));
    };
};
```

但是由于参数不是一个数组，没有concat函数，所以需要把伪数组转成一个数组，可以用Array.prototype.slice:

```javascript
Function.prototype.curry = function(){
    let slice = Array.prototype.slice;
    let defaultArgs = slice.call(arguments);
    let that = this;
    return function(){
        return that.apply(this, defaultArgs.concat(slice.call(arguments)));
    };
};
```

现在举一个柯里化一个有用的例子，当需要把一个数组降序排序的时候，需要这样写：

```javascript
let data = [1, 5, 2, 3, 10];
data.sort((a, b) => b - a);	// [10, 5, 3 ,2, 1]
```

给sort传一个函数的参数，但是如果你的降序操作比较多，每次都写一个函数参数还是有点烦的，因此可以用柯里化把这个参数固化起来：

```javascript
Array.prototype.sortDescending = Array.prototype.sort.curry((a, b) => b - a);
```

这样就方便多了：

```javascript
let data = [1, 5, 2, 3, 10];
data.sortDescending();
console.log(data);	// [10, 5, 3, 2, 1]
```

## 5. 防止篡改对象

有时候你可能怕你的对象被误改了，所以需要把它保护起来。

（1）Object.seal防止新增和删除属性

如下代码，当把一个对象seal之后，将不能添加和删除属性。

![阻止对象添加和删除属性](https://user-gold-cdn.xitu.io/2017/9/3/9c3b7ae838e7d7cf9cdec8aa98ee61a7?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

当使用严格模式将会抛出异常。

![严格模式下抛出异常](https://user-gold-cdn.xitu.io/2017/9/3/ee58ecb88a3ca505e70fed2e6025548a?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

（2）Object.freeze冻结对象

这个数不能改属性值，如下图所示。

![Object.freeze冻结](https://user-gold-cdn.xitu.io/2017/9/3/eb52073e8e900111de3143b50a038a1a?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

同时可以使用Object.isFrozen、Object.isSealed、Object.isExtensible判断当前对象的状态。

（3）defineProperty冻结单个属性

如下图所示，设置enumerable/writable为false，那么这个属性将不可遍历和写：

![冻结单个属性](https://user-gold-cdn.xitu.io/2017/9/3/4e3c2c849bca4a77e2a0a8eba83ceabf?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

## 6. 定时器

怎么实现一个JS版的sleep函数？因为在C/C++/Java等语言是有sleep函数，但是JS没有。sleep函数的作用是让线程进入休眠，当到了指定时间后再重新唤起。你不能写个while循环然后不断地判断当前时间和开始时间的差值是不是到了指定时间了，因为这样会占用CPU，就不是休眠了。

这个实现比较简单，我们可以使用setTimeout + 回调：

```javascript
function sleep(millionSeconds, callback){
    setTimeout(callback, millionSeconds);
}

// sleep 2 秒
sleep(2000, () => console.log("sleep recover"));
```

但是使用回调让我的代码不能够和平常的代码一样像瀑布流一样写下来，我得搞一个回调函数当作参数传值。于是想到了Promise，现在用Promise改写一下：

```javascript
function sleep(millionSeconds){
    return new Promise(resolve => setTimeout(resolve, millionSeconds));
}
sleep(2000).then(() => console.log("sleep recover"));
```

但好像还是没有办法解决上面的问题，仍然需要传递一个函数参数。

虽然使用Promise本质上是一样的，但是它有一个resolve的参数，方便你告诉它什么时候异步结束，然后它就可以执行then了，特别是在回调比较复杂的时候，使用Promise还是会更加的方便。

ES7新增了两个新的关键字async/await用于处理异步的情况，让异步代码的写法就像同步代码一样，如下async版本的sleep：

```javascript
function sleep(millionSeconds){
    return new Promise(resolve => setTimeOut(resolve, millionSeconds));
}

async function init(){
    await sleep(2000);
    console.log("sleep recover");
}

init();
```

相对于简单的Promise版本，sleep的实现还是没变。不过在调用sleep的前面加一个await，这样只有sleep这个异步完成了，才会接着执行下面的代码。同时需要把代码逻辑包在一个async标记的函数里面，这个函数会返回一个Promise对象，当里面的异步都执行完了就可以then了：

```javascript
init().then(() => console.log("init finished"));
```

ES7的新关键字让我们的代码更加地简洁优雅。

关于定时器还有一个很重要的话题，那就是setTimeout和setInterval的区别。如下图所示。

![setTimeout 和 setInterval的区别](https://user-gold-cdn.xitu.io/2017/9/3/86b8194cd16e39f7512f319d024d0593?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

setTimeout是在当前执行单元都执行完才开始计时，而setInterval是在设定完计时器后就立马计时。可以用一个实际的例子做说明，这个例子我在《[JS与多线程](https://fed.renren.com/2017/05/21/js-threads/)》这篇文章立马提到过，这里用代码实际地运行一下，如下代码所示：

```javascript
let scriptBegin = Date.now();
fun1();
fun2();

// 需要执行20ms的工作单元
function act(functionName){
    console.log(functionName, Date.now() - scriptBegin);
    let begin = Date.now();
    while(Date.now() - begin < 20);
}
function fun1(){
    let fun3 = () => act("fun3");
    setTimeout(fun3, 0);
    act("fun1");
}
function fun2(){
    act("fun2 - 1");
    let fun4 = () => act("fun4");
    setInterval(fun4, 20);
    act("fun2 - 2");
}
```

这个代码的执行模型是这样的：

![代码执行模型](https://user-gold-cdn.xitu.io/2017/9/3/f0a16d027e34ec4c17e3f4c0f8886cd3?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

控制台输出：

![控制台输出](https://user-gold-cdn.xitu.io/2017/9/3/2b2d2d41fb0e19d67dd7ee39093d404b?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

与上面的模型分析一致。

接着再讨论最后一个话题，函数节流。

## 7. 函数节流

节流的目的是为了不想触发执行的太快，比如：

> * 监听input触发搜索
> * 监听resize做响应式调整
> * 监听mousemove调整位置

我们先看一下，resize/mousemove事件1s内能触发多少次，于是写了以下代码：

```javascript
let begin = 0;
let count = 0;
window.resize = function(){
    count++;
    let now = Date.now();
    if(!begin){
        begin = now;
        return;
    }
    if((now - begin) % 1000 < 60){
        console.log(now - begin, count / (now - begin) * 1000);
    } 
};
```

当把窗口拉得比较快的时候，resize事件大概是1s触发40次：

![resize事件触发频率](https://user-gold-cdn.xitu.io/2017/9/3/2ab94229cd6b7e72a43f16cd80d213aa?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

需要注意的是，并不是说你拉得越快，触发得就越快。实际情况是，拉得越快触发得越慢，因为拉动的时候页面需要重绘，变化的越快，重绘的次数也就越多，所以导致触发得更少了。

mousemove事件在我的电脑的Chrome上1s大概触发了60次：

![mousemove事件触发的频率](https://user-gold-cdn.xitu.io/2017/9/3/53980650b625d9f54fe56ca9bab34fab?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

如果你需要监听resize事件做DOM调整的话，这个调整比较费时，1s要调整40次，这样可能响应不过来，并且不需要调整的那么频繁，所以需要节流。

怎么实现一个节流呢，书里是这么实现的。

```javascript
function throttle(method, context){
    clearTimeout(method.tId);
    method.tId = setTimeout(function(){
        method.call(context);
    }, 100);
}
```

每次执行都要setTimeout一下，如果触发得很快就把上一次的setTimeout清掉重新setTimeout，这样就不会执行的很快了。但是这样有个问题，就是这个回调函数可能永远不会执行，因为它一直在触发，一直在清掉tId，这样就有点尴尬，上面代码的本意应该是100ms内最多触发一次，而实际情况是可能永远不会执行。这种实现应该叫防抖，而不是节流。

把上面的代码稍微改造一下：

```javascript
function throttle(method, context){
    if(method.tId){
        return;
    }
    method.tId = setTimeout(function(){
        method.call(context);
        method.tId = 0;
    }, 100);
}
```

这个实现就是正确的，每100ms最多执行一次回调，原理是在setTimeout里面把tId给置成0，这样能让下一次的触发执行。实际实验一下：

![实际结果](https://user-gold-cdn.xitu.io/2017/9/3/0310ddcbed5f2fe42d18cefffa2b3d7c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

大概每100ms就执行一次，这样就达到了我们的目的。

但是这样还有一个小问题，就是每次执行都是要延迟100ms，有时候用户可能就是最大化了窗口，只触发了一次resize事件，但是这次还是得延迟100ms才能执行，假设你的时间是500ms，那就得延迟半秒，因此这个实现不太理想。

需要优化，如下代码所示：

```javascript
function throttle(method, context) {
    // 如果是第一次触发，立刻执行
    if (typeof method.tId === "undefined") {
        method.call(context);
    }
    if (method.tId) {
        return;
    }
    method.tId = setTimeout(function() {
        method.call(context);
        method.tId = 0;
    }, 100);
}
```

先判断是否为第一次触发，如果是的话立刻执行。这样就解决了上面提到的问题，但是这个实现还是有问题，因为它只是全局的第一次，用户最大化之后，隔了一会又取消最大化了就又有延迟了，并且第一次触发会执行两次。那怎么办呢？

笔者想到一个办法：

```javascript
function throttle(method, context) {
    if (!method.tId) {
        method.call(context);
        method.tId = 1;
        setTimeout(() => method.tId = 0, 100);
    }
}
```

每次触发的时候立刻执行，然后再设定一个计时器，把tId置成0，实际的效果如下：

![实际效果](https://user-gold-cdn.xitu.io/2017/9/3/d55b504af7abcd28d31e30a92ecfbd99?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

这个实现比之前的实现还要简洁，并且能够解决延迟的问题。

所以通过节流，把执行次数降到了1s执行10次，节流时间也可以控制，但同时失去了灵敏度，如果你需要高灵敏度就不应该使用节流，例如做一个拖拽的应用，如果节流了会怎么样？用户会发现拖起来一卡一卡的。

笔者重新看了高程的《高级技巧》的章节结合自己的理解和实践总结了这么一篇文章，我的体会是如果看书看博客只是当作睡前读物看一看其实收获不是很大，没有实际地把书里的代码实践一下，没有结合自己的编码经验，就不能用自己的理解去融入这个知识点，从而转化为自己的知识。你可能会说我看了之后就会印象啊，有印象还是好的，但是你花了那么多时间看了那本书只是得到了一个印象，你自己都没有实践过的印象，这个印象又有多靠谱呢。如果别人问到了这个印象，你可能会回答出一些连不起来的碎片，就会给人一种背书的感觉。还有有时候书里可能会有一些错误或者过时的东西，只有实践了才能出真知。