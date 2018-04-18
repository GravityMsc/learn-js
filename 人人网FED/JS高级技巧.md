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