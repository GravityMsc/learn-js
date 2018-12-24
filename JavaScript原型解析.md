## 前言

ES6的第一个版本发布于15年6月，而本文最早创作于16年，那也是笔者从事前端的早期。在那个时候，ES6的众多特性仍处于stage阶段，也远没有现在这么普及，为了更轻松地写JavaScript，笔者曾花费了整整一天，仔细理解了一下原型——这个对于一个成熟的JavaScript开发者必须要跨越的大山。

ES6带来了太多的语法糖，其中箭头函数掩盖了this的神妙，而class也掩盖了本文要长篇谈论的原型。

最近，我重写了这篇文章，通过本文，你将可以学到：

* 如何用ES5模拟类；
* 理解prototype和\_\_proto__；
* 理解原型链和原型继承；
* 更深入地了解JavaScript这门语言。

## 引入：普通对象与函数对象

在JavaScript中，一直有这么一种说法，万物皆对象。事实上，在JavaScript中，对象也是有区别的，我们可以将其划分为普通对象和函数对象。Object和Function便是JavaScript自带的两个典型的函数对象。而函数对象就是一个纯函数，所谓的函数对象，其实就是使用JavaScript在模拟类。

那么，究竟什么是普通对象，什么又是函数对象呢？请看下方的例子：

首先，我们分别创建了三个Function和Object的实例：

```javascript
function fn1(){}
const fn2 = function() {}
const fn3 = new Function('language', 'console.log(language)')

const ob1 = {}
const ob2 = new Object()
const ob3 = new fn1()
```

打印以下结果，可以得到：

```javascript
console.log(typeof Object); // function
console.log(typeof Function); // function
console.log(typeof ob1); // object
console.log(typeof ob2); // object
console.log(typeof ob3); // object
console.log(typeof fn1); // function
console.log(typeof fn2); // function
console.log(typeof fn3); // function
```

在上述的例子中，ob1、ob2、ob3为普通对象（均为Object的实例），而fn1、fn2、fn3均是Function的实例，称之为函数对象。

如何区分呢？其实记住这句话就行了：

* 所有Function的实例都是函数对象，而其他的都是普通对象。

说道这里，细心的同学会发表一个疑问，一开始，我们已经提到，Object和Function均是函数对象，而这里我们又说：所有Function的实例都是函数对象，难道Function也是Function的实例？

先保留这个疑问。接下来，对这一节的内容做个总结：

![普通对象和函数对象的总结](https://user-gold-cdn.xitu.io/2018/2/27/161d355acfd4d6a5?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

从图中可以看出，对象本身的实现还是要依靠构造函数。那原型链到底是用来干嘛的呢？

众所周知，作为一门面向对象（Object Oriented）的语言，必定具有以下特征：

* 对象唯一性
* 抽象性
* 继承性
* 多态性

而原型链最大的目的，就是为了实现继承。

## 进阶：prototype和\_\_proto__

原型链究竟是如何实现继承的呢？首先，我们要引入介绍两兄弟：prototype和\_\_proto__，这是在JavaScript中无处不在的两个变量（如果你经常调试的话），然而，这两个变量并不是在所有的对象上都存在，先看一张表：

| 对象类型     | `prototype` | `__proto__` |
| ------------ | ----------- | ----------- |
| 普通对象(NO) | ❎           | ✅           |
| 函数对象(FO) | ✅           | ✅           |

首先，我们先给出以下结论：

1. 只有函数对象具有prototype这个属性；
2. prototype和\_\_proto__都是JavaScript在定义一个函数或对象时自动创建的预定义属性。

接下来，我们验证上述的两个结论：

```javascript
function fn() {
    console.log(typeof fn.__proto__);	// function
    console.log(typeof fn.prototype);	// object
}

const ob = {};
console.log(typeof ob.__proto__);	// function
console.log(typeof on.prototype);	// undefined
```

既然是语言层面的内置属性，那么两者究竟有何区别呢？我们依然从结论触发，给出以下两个结论：

1. prototype被实例的\_\_proto__所指向（被动）
2. \_\_proto__指向构造函数的prototype（主动）

也就是说以下代码成立：

```javascript
console.log(fn.__proto__ === Function.prototype);	// true
console.log(ob.__proto__ === Object.prortotype);	// true
```

看起来很酷，结论瞬间被证明，那么问题来了：既然fn是一个函数对象，那么fn.prototype.\_\_proto__到底等于什么？

这是我尝试去解决这个问题的过程：

1. 首先用typeof得到fn.prototype的类型：“object”
2. 既然是“object”，那fn.prototype岂不是Object的实例？根据上述的结论，快速地写出验证代码：

```javascript
console.log(fn.prototype.__proto__ === Object.prototype) // true
```

接下来，如果要你快速地写出，在创建一个函数时，JavaScript对该函数原型的初始化代码，你是不是也能快速地写出：

```javascript
// 实际代码
function fn1() {}

// JavaScript自动执行
fn1.prototype = {
    constructor: fn1,
    __proto__: Object.prototype
}

fn1.__proto__ = Function.prototype
```

到这里，你是否有一丝恍然大悟的感觉？此外，因为普通对象就是通过函数对象实例化（new）得到的，而一个实例不可能再次进行实例化，也就不会让另一个对象的\_\_proto__指向它的prototype，因此本节一开始提到的普通对象没有prototype属性的这个结论似乎非常好理解了。从上述的分析，我们还可以看出，fn1.prototype就是一个普通对象，它也不存在prototype属性。

再回顾上一节，我们还遗留一个以为：

* 难道Function也是Function的实例？

答案当然是肯定的。

```javascript
console.log(Function.__proto__ === Function.prototype)	// true
```

## 重点：原型链

上一节我们详解了prototype和\_\_proto__，实际上，这两兄弟主要就是为了构造原型链而存在的。

先上一段代码：

```javascript
const Person = function(name, age) {
    this.name = name
    this.age = age
} /* 1 */

Person.prototype.getName = function() {
    return this.name
} /* 2 */

Person.prototype.getAge = function() {
    return this.age
} /* 3 */

const ulivz = new Person('ulivz', 24); /* 4 */

console.log(ulivz) /* 5 */
console.log(ulivz.getName(), ulivz.getAge()) /* 6 */
```

解释一下执行细节：

1. 执行1，创建了一个构造函数Person，要注意，前面已经提到，此时Person.prototype已经被自动创建，它包含constructor和\_\_proto__这两个属性；
2. 执行2，给对象Person.prototype增加了一个方法getName()；
3. 执行3，给对象Person.prototype增加了一个方法getAge()；
4. 执行4，由构造函数Person创建了一个实例ulivz，值得注意的是，一个构造函数在实例化时，一定会自动执行该构造函数。
5. 在浏览器得到5的输出，即ulivz应该是：

```javascript
{
     name: 'ulivz',
     age: 24
     __proto__: Object // 实际上就是 `Person.prototype`
}
```

结合上一节的经验，以下等式成立：

```javascript
console.log(ulivz.__proto__ == Person.prototype)  // true
```

6. 执行6的时候，由于在ulivz中找不到getName()和getAge()这两个方法，就会继续朝着原型链向上查找，也就是通过\_\_proto__向上查找，于是，很快在ulviz\.\_\_proto\_\_中，即Person.prototype中找到了这两个方法，于是停止查找并执行得到结果。

这便是JavaScript的原型继承。准确的说，JavaScript的原型继承是通过\_\_proto__并借助prototype来实现的。



