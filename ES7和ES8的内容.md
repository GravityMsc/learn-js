## ES7新特性

ES7在ES6的基础上添加了三项内容：求幂运算符（**）、Array.prototype.includes()方法、函数作用域中严格表达式的变更。

### Array.prototype.includes()方法

includes()的作用，是查找一个值在不在数组中，如果存在，则返回true，反之返回false。基本用法：

```javascript
['a', 'b', 'c'].includes('a');		// true
['a', 'b', 'c'].includes('d');		// false
```

Array.prototype.includes()方法接收两个参数：要搜索的值和搜索的开始索引。当第二个参数被传入时，该方法会从索引处开始往后搜索（默认索引值为0）。若搜索值再数组中存在则返回true，否则返回false。且看下面示例：

```javascript
['a', 'b', 'c', 'd'].includes('b');		// true
['a', 'b', 'c', 'd'].includes('b', 1);	// true
['a', 'b', 'c', 'd'].includes('b', 2);	// false
```

那么，我们会联想到ES6里数组的另一个方法indexOf，下面的示例代码是等效的：

```javascript
['a', 'b', 'c'].includes('a');	// true
['a', 'b', 'c'].indexOf('a') > -1;	// true
```

此时，就有必要来比较下两者的优缺点和使用场景了。

* 简便性

  从这一点上来说，includes略胜一筹。熟悉indexOf的同学都知道，indexOf返回的是某个元素在数组中的下标值，若想判断某个元素是否在数组里，我们还需要做额外的处理，即判断该返回值是否 > -1。而includes则不用，它直接返回的便是Boolean型的结果。

* 精确性

  两者使用的都是 === 操作符来做值的比较。但是includes()方法有一点不同，两个NaN被认为是相等的，即使在NaN === NaN结果是false的情况下。这一点和indexOf()的行为不同，indexOf()严格使用 === 判断。请看下面示例代码：

  ```javascript
  let demo = [1, NaN, 2, 3];
  
  demo.indexOf(NaN);		// -1
  demo.includes(NaN);		// true
  ```

  上述代码中。indexOf()方法返回-1，即使NaN存在于数组中，而includes()则返回了true。

  > 提示：由于它对NaN的处理方式与indexOf不同，加入你只想知道某个值是否在数组中而并不关心它的索引位置，建议使用includes()。如果你想获取一个值在数组中的位置，那么你只能使用indexOf方法。

includes()还有一个怪异的点需要指出，在判断+0与-0时，被认为是相同的。

```javascript
[1, +0, 3, 4].includes(-0);	// true
[1, +0, 3, 4].indexOf(-0);	// 1
```

在这一点上，indexOf()与includes()的处理结果是一样的，前者同样会返回+0的索引值。

> 注意：在这里，需要注意一点，includes()只能判断简单类型的数据，对于复杂类型的数据，比如对象类型的数组，二维数组都是无法判断的。

###求幂运算符（**）

基本用法

```javascript
3 ** 2		// 9
```

效果同：

```javascript
Math.pow(3, 2);	// 9
```

** 是一个用于求幂的中缀算子，比较可知，中缀符号比函数符号更简洁，这也使得它更为可取。下面让我们扩展下思路，既然说**是一个运算符，那么它就应该能满足类似加等的操作，我们姑且称之为幂等，例如下面的例子，a的值依然是9：

```javascript
let a = 3;
a **= 2
// 9
```

对比下其他语言的指数运算符：

* Python： x ** y
* CoffeeScript: x ** y
* F#: x ** y
* Ruby: x ** y
* perl: x ** y
* Lua, Basic, MATLAB: x ^ y

不难发现，ES的这个新特性是从其他语言（Python，Ruby等）模仿而来的。

## ES8新特性

### 异步函数（Async functions）

#### 为什么要引入async

众所周知，JavaScript语言的执行环境是“单线程”的，那么异步编程对JavaScript语言来说就显得尤为重要。以前我们大多数的做法是使用回调函数来实现JavaScript语言的异步编程。回调函数本身没有问题，但如果出现多个回调函数嵌套，例如：进入某个页面，需要先登录，拿到用户信息之后，调取用户商品信息，代码如下：

```javascript
this.$http.jsonp('/login', (res) => {
    this.$http.jsonp('getInfo', (info) => {
        // do something
    });
});
```

加入上面还有更多的请求操作，就会出现多重嵌套。代码很快就会乱成一团，这种就被称为“回调地狱”（callback hell）。

于是，我们提出了Promise，它将回调函数的嵌套，改成了链式调用。写法如下：

```javascript
var promise = new Promise((resolve, reject) => {
    this.login(resolve);
})
.then(() => this.getInfo())
.catch(() => { console.log("Error"); });
```

从上面可以看出，Promise的写法只是回调函数的改进，使用then方法，只是让异步任务的两段执行更清楚而已。Promise的最大问题是代码冗余，请求任务多时，一堆的then，也使得原来的语义变得很不清楚。此时我们引入了另外一种异步编程的机制：Generator。

Generator函数是一个普通函数，但是又两个特征。一是：function关键字与函数名之间有一个星号；二是：函数体内部使用yield表达式，定义不同的内部态（yield在英语里的意思就是“产出”）。一个简单的例子来说明它的用法：



