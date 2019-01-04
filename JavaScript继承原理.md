ES6的class语法糖你是否已经用得是否炉火纯青呢？那如果回归到ES5呢？本文将阐述如何用JavaScript来实现类的继承。

通过本文，你将学到：

1. 如何用JavaScript模拟类中的私有变量；
2. 了解常见的几种JavaScript继承方法，原理及其优缺点；
3. 实现一个较为fancy的JavaScript继承方法。

## 类

我们来回顾一下ES6/TypeScript/ES5类的写法以作对比。首先，我们创建一个GithubUser类，它拥有一个login方法，和一个静态方法getPublicServices，用于获取public的方法列表：

```javascript
class GithubUser {
    static getPublicServices() {
        return ['login'];
    }
    constructor(username, password) {
        this.username = username;
        this.password = password;
    }
    login() {
        console.log(this.username + '要登录Github，密码是' + this.password);
    }
}
```

实际上，ES6这个类的写法有一个弊病，实际上，密码password应该是Github用户一个私有变量，接下来，我们用TypeScript重写一下：

```javascript
class GithubUser {
    static getPublicServices() {
        return ['login'];
    }
    public username: string;
	private password: string;
	cosntructor(username, password) {
    	this.username = username;
        this.password = password;
	}
	public login(): void {
    	console.log(this.username + '要登录Github，密码是' + this.password);
	}
}
```

如此一来，password就只能在类的内部访问了。好了，问题来了，如果结合原型讲解那一文的只是，来用ES5实现这个类呢？

```javascript
function GithubUser(username, password) {
    // private属性
    let _password = password 
    // public属性
    this.username = username 
    // public方法
    GithubUser.prototype.login = function () {
        console.log(this.username + '要登录Github，密码是' + _password)
    }
}
// 静态方法
GithubUser.getPublicServices = function () {
    return ['login']
}
```

> 值得注意的是，我们一般都会把共有方法放在类的原型上，而不会采用this.login = function() {}这种写法。因为只有这样，才能让多个实例引用同一个共有方法，从而避免重复创建方法的浪费。

是不是很直观，留下两个疑问：

1. 如何实现private方法呢？
2. 能否实现protected属性/方法呢？

## 继承

用掘金的用户都应该知道，我们可以选择直接使用Github登录，那么，结合上一节，我们如果创建了一个JuejinUser来继承GithubUser，那么JuejinUser及其实例就可以调用Github的login方法了。首先，先写出这个简单JuejinUser类：

```javascript
function JuejinUser(username, password) {
    // TODO need implementation
    this.articles = 3 // 文章数量
    JuejinUser.prototype.readArticle = function () {
        console.log('Read article')
    }
}
```

由于ES6/TS的继承太过直观，本节将忽略。首先概述一下本文将要讲解的几种继承方法：

* 类式继承
* 构造函数式继承
* 组合式继承
* 原型继承
* 寄生式继承
* 寄生组合式继承

看起来很多，我们将一一论述。

## 类式继承

因为我们已经得知：

> 若通过new Parent()创建了Child，则Child.\_\_proto__ = Parent.prototype，而原型链则是顺着\_\_proto__依次向上查找。因此，可以通过修改子类的原型为父类的实例来实现继承。

第一直觉的实现如下：

```javascript
function GithubUser(username, password) {
    let _password = password 
    this.username = username 
    GithubUser.prototype.login = function () {
        console.log(this.username + '要登录Github，密码是' + _password)
    }
}

function JuejinUser(username, password) {
    this.articles = 3 // 文章数量
    JuejinUser.prototype = new GithubUser(username, password)
    JuejinUser.prototype.readArticle = function () {
        console.log('Read article')
    }
}

const juejinUser1 = new JuejinUser('ulivz', 'xxx', 3)
console.log(juejinUser1)
```

在浏览器中查看原型链：

![原型链](https://user-gold-cdn.xitu.io/2018/3/1/161dd7f0b8279b9e?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

诶，不对啊，很明显juejinUser1.\_\_proto__并不是GithubUser的一个实例。

