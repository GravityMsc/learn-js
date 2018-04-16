# JavaScript中的new()到底做了什么？

要创建Person的新实例，必须使用new操作符。以这种方式调用构造函数实际上回经历一下4个步骤：

（1）创建一个新对象；

（2）将构造函数的作用域赋给新对象（因此this就指向了这个心对象）；

（3）执行构造函数中的代码（为这个新对象添加属性）；

（4）返回新对象。



## new操作符

在有上面的基础概念的介绍之后，再加上new操作符，我们就能完成传统面向对象的class + new的方式创建对象，在JavaScript中，我们将这类方式称为Pseudoclassical（伪类）。

基于上面的例子，我们执行如下代码：

```javascript
var obj = new Base();
```

这样的代码结果是什么，我们在JavaScript引擎中看到的对象模型是：

obj对象(id:"base") ==（_\_proto__）=》Base.prototype对象

new操作符具体干了什么呢？其实很简单，就干了三件事情。

```javascript
var obj = {};
obj.__proto__ = Base.prototype;
Base.call(obj);
```

第一行，我们创建了一个空对象obj；

第二行，我们将这个空对象的_\_proto__成员指向了Base函数对象prototype成员对象；

第三行，我们将Base函数对象的this指针替换成obj，然后再调用Base函数，于是我们就给obj对象赋值了一个id成员变量，这个成员变量的值是“base”。

如果我们给Base.prototype的对象添加一些函数会有什么效果呢？

例如代码如下：

```javascript
Base.ptototype.toString = function(){
    return this.id;
}
```

那么当我们使用new创建一个新对象的时候，根据_\_proto__的特性，toString这个方法也可以做新对象的方法被访问到。于是我们看到了：

> 构造子对象中，我们来设置类的成员变量（例如：例子中的id），构造子对象prototype中我们来设置类的公共方法。于是通过函数对象和JavaScript特有的_\_proto__与prototype成员及new操作符，模拟出类和类实例化的效果。

