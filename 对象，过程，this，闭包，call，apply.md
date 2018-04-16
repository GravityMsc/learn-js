# JavaScript面试频繁出现的几个易错点

## 1.  前言

这段时间，金三银四，很多人面试，很多人分享面试题。在前段时间，我也临时担任面试官，为了大概了解面试者的水平，我也写了一份题目，面试了几个前端开发者。所以，今天就总结一下，那些让人掉坑的考点。



## 2.  面向对象编程

> 关于面向对象和面向过程，个人觉得这两者不是绝对独立的，而是相辅相成的关系。至于什么时候用面向对象，什么时候用面向过程，具体情况，具体分析。

针对于面向对象编程的，知乎上有一个高赞回答：

> 面向对象：狗.吃（屎）
>
> 面向过程：吃.(狗，屎)

但是这个例子觉得不太优雅，我改了一下，举一个优雅些的小例子说明一下面向对象和面向过程的区别。

需求：定义  ‘守候吃火锅’

面向对象的思想是：守候.动作（吃火锅）

面向过程的思想是：动作（守候，吃火锅）

代码实现方面：

```javascript
// 面向对象
// 定义人（姓名）
let People = function(name){
    this.name = name;
}
// 动作
People.prototype = {
    eat: function(someThing){
        console.log(`${this.name}吃${someThing}`);
    }
}
// 守候是个人，所以要创建一个人（new一次People）
let shouhou = new People('守候');
shouhou.eat('火锅');

// 面向过程
let eat = function(who, someThing){
    console.log(`${who}吃${someThing}`);
}
eat('守候', '火锅');
```

结果都一样，都是输出 ‘守候吃火锅’。但是万一我现在吃饱了，准备些代码了。这下怎么实现呢？看代码：

```javascript
// 面向对象
shouhou.coding = function(){
    console.log(this.name + '写代码');
}
shouhou.coding();

// 面向过程
let coding = function(who){
    console.log(who + '写代码');
}
coding('守候');
```

结果也是一样： ‘守候写代码’

但是不难发现面向对象更加的灵活，复用性和扩展性更佳。因为面向对象就是针对对象（例子中的：‘守候’）来执行某些动作。这些动作可以自定义扩展。

而面向过程是定义很多的动作，来指定谁来执行这个动作。

好了，面向对象的简单说明就到这里了。至于面向对象的三大特性：继承，封装，多态自行寻找资料学习。



## 3.  this

使用JavaScript开发的时候，很多开发者多多少少会被this的指向搞蒙圈，但是实际上，关于this的指向，记住最核心的一句话：**哪个对象调用函数，函数里面的this指向哪个对象**。

下面分几种情况讨论下：

### 3.1  普通函数调用

这种情况没特殊意外，就是指向全局对象-window。

```javascript
let username = 'faker';
function fn(){
    alert(this.username);	//undefined
}
fn();
```

可能大家会困惑，为什么不是输出faker，但是再仔细一看，这里声明的方式是let，不会是window对象，如果想要输出faker，要这样写：

```javascript
var username = 'faker';
function fn(){
    alert(this.username);	// faker
}
fn();
// 等同于
window.username = 'faker';
function fn(){
    alert(this.username);	// faker
}
fn();
// 可以理解为widnow.fn();
```

###3.2  对象函数调用

这个相信不难理解，就是哪个函数调用，this指向哪里。

```javascript
window.b = 2222;
let obj = {
    a: 111,
    fn: function(){
        alert(this.a);	// 111
        alert(this.b);	// undefined
    }
};
obj.fn();
```

很明显，第一次就是输出obj.a，就是111。而第二次，obj没有b这个属性，所以输出undefined，因为this指向obj。

但是下面这个情况得注意：

```javascript
let obj1 = {
    a: 222
};
let obj2 = {
    a: 111,
    fn: function(){
        alert(this.a);
    }
};
obj1.fn = obj2.fn;
obj1.fn();	// 222
```

这个相信也不难理解，虽然obj1.fn是从obj2.fn赋值而来，但是调用函数的是obj1，所以this指向obj1。

### 3.3  构造函数调用

```javascript
let TestClass = function(){
    this.name = '111';
};
let subClass = new TestClass();
subClass.name = 'faker';
console.log(subClass.name);		// faker
let subClass1 = new TestClass();
console.log(subclass1.name);	// 111
```

这个也是不难理解，回一下[new的四个步骤](https://link.juejin.im/?target=https%3A%2F%2Fwww.cnblogs.com%2Ffaith3%2Fp%2F6209741.html)就可以了。

但是有一个坑，虽然一般不会出现，但是有必要提一下。

在构造函数里面返回一个对象，会直接返回这个对象，而不是执行构造函数后创建的对象。

![构造函数对象](https://user-gold-cdn.xitu.io/2018/2/25/161cd7a39ac14a3a?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### 3.4  apply和call调用

apply和call简单来说就是会改变传入函数的this。

```javascript
let obj1 = {
    a: 222
};
let obj2 = {
    a: 111,
    fn: function(){
        alert(this.a);
    }
};
obj2.fn.call(obj1); // 222
```

此时虽然是obj2调用方法，但是使用了call，动态的把this指向到obj1.相当于这个obj2.fn的执行环境是obj1。apply和call详细内容在下面提及。

### 3.5  箭头函数调用

首先不得不说，ES6提供了箭头函数，增加了我们的开发效率，但是在箭头函数里面，没有this，箭头函数里面的this是继承外面的环境。

一个例子：

```javascript
let obj = {
    a: 222,
    fn: function(){
        setTimeout(function(){console.log(this.a)});
    }
};
obj.fn();	// undefined
```

不难发现，虽然**fn()**里面的**this**是指向**obj**，但是，传给**setTimeout**的是普通函数，**this**指向是**window**，**window**下面没有**a**，所以这里输出**undefined**。

换成箭头函数

```javascript
let obj = {
    a: 222,
    fn: function(){
        setTimeout(() => {console.log(this.a)});
    }
};
obj.fn();	// 222
```

这次输出**222**是因为，传给**setTimeout**的是箭头函数，然后箭头函数里面没有**this**，所以要向上层作用域去找，在这个例子上，**setTimeout**的上层作用域是**fn**。而**fn**里面的**this**指向**obj**，所以**setTimeout**里面的箭头函数的**this**，指向**obj**。所以输出**222**。



## 4.  call和apply

**call**和**apply**的作用，完全一样，唯一的区别就是在参数上面。

**call**接收的**参数不固定**，第一个参数是函数体内**this**的指向，第二个参数以下是依次传入的参数。

**apply**接收**两个参数**，第一个参数也是函数体内**this**的指向，第二个参数是一个集合对象（数组或者类数组）。

```javascript
let fn = function(a, b ,c){
    console.log(a, b ,c);
}
let arr = [1, 2, 3];
```

![call和apply](https://user-gold-cdn.xitu.io/2018/2/25/161cd9377a0bdf41?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

如上面这个例子，fn.call会将arr当作第一个参数a，然后输出a，b和c都没有值，输出undefined。fn.apply则将arr视为数组，然后以数组的值依次匹配参数a，b，c。

```javascript
let obj1 = {
    a: 222
};
let obj2 = {
    a: 111,
    fn: function(){
        alert(this.a);
    }
};
obj2.fn.call(obj1); // 222
```



call和apply两个主要用途就是：

> 1.改变this的指向（把this从obj2指向到obj1）。
>
> 2.方法借用（obj1没有fn，只是借用obj2的fn方法）。



## 5.  闭包

闭包这个可能大家是迷糊，但是必须要征服的概念！下面用一个例子简单说下。

```javascript
let add = (function(){
    let now = 0;
    return {
        doAdd: function(){
            console.log(now);
        }
    };
})();
```

然后执行几次！

![闭包](https://user-gold-cdn.xitu.io/2018/2/25/161cd9ce3c5bca2b?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

上图结果看到，now这个变量，并没有随着函数的执行完毕而被回收，而是接续保存在内存里面。具体原因说下：刚开始进来，因为是自动执行函数，一开始进来会自动执行这一块：

![闭包](https://user-gold-cdn.xitu.io/2018/2/25/161cd9f3184cafbb?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

然后把这个对象赋值给add。由于add里面有函数是依赖于now这个变量。所以now不会被销毁，回收。这就是闭包的用途之一（延续变量周期）。由于now在外面访问不到，这就是闭包的另一个用途（创建局部变量，保护局部变量不会被访问和修改）。

> 可能有人会有疑问，闭包会造成内存泄漏。但是大家想下，上面的例子，如果不用闭包，就要用全局变量。把变量放在闭包里面和放在全局变量里面，影响是一致的。使用闭包又可以减少全局变量，所以上面的例子闭包更好！

