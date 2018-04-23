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

如下图所示，设置enumable/writable为false，那么