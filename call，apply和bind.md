# 从一行等式理解JS当中的call，apply和bind

关于JS当中的call，apply和bind，相信大家和我一样，已经看过了无数篇相关的文章，都有自己的理解。所以这篇文章并非什么科普类的文章，仅仅是把我自己的理解记录下来。

我的学习习惯，是喜欢把各种看似孤立的知识点串联起来，综合理解并运用，通过最简单最直观的思路把它理解透。所以，这篇文章将通过一段非常简洁的等式，把JS当中一个相对较难的知识点，call，apply和bind给串联起来：

```javascript
cat.call(dog, a, b) = cat.apply(dog, [a, b]) = (cat.bind(dog,a, b))() = dog.cat(a, b)
```

要理解JS当中的这三个关键字，首先的弄清楚它们是用来干嘛的。复杂些来说，可以引用MDN文档中的原文：

> 可以让call()中的对象调用当前对象所拥有的function。你可以使用call()来实现继承：写一个方法，然后让另一个新的对象来继承它（而不是在新对象中再写一次这个方法）。

简单些来说，可以引用大家都看过的一句话：

> 为了动态改变某个函数运行时的上下文（context）。

又或者是

> 为了改变函数体内部this的指向。

上面这些解释都很正确，说的一点问题都没有，但是里面却又引入了继承，上下文，this这些额外的知识点。如果我只想用最直观的方法去理解这三个关键字的作用，也许可以这么去理解：

定义一个猫对象：

```javascript
class Cat {
    constructor (name) {
        this.name = name;
    }
    
    catchMouse(name1, name2){
        console.log(`${this.name} caught 2 mouse! They call ${name1} and ${name2}.`);
    }
}
```

这个猫对象拥有一个抓老鼠的技能catchMouse()。

然后类似的，定义一个狗对象：

```javascript
class Dog {
    constructor(name){
        this.name = name;
    }
    
    biteCriminal(name1, name2){
        console.log(`${this.name} bite 2 criminals! Their name is ${name1} and ${name2}.`);
    }
}
```

这个狗对象能够咬坏人biteCriminal()。

接下来，我们实例化两个对象，分别得到一只脚“Kitty”的猫和叫“Doggy”的狗：

```javascript
const kitty = new Cat('Kitty');
const doggy = new Dog('Doggy');
```

首先让它们彼此发挥自己的才能：

```javascript
kitty.catchMouse('Mickey', 'Minnie');
// Kitty caught mouse! They call Mickey and Minnie.

doggy.biteCirminal('Tom', 'Jerry');
// Doggy bite 2 criminals! Their name is Tom and Jerry.
```

现在，我们希望赋予Doggy抓老鼠的能力，如果不使用这三个关键字，应该怎么做呢？

方案A：修改Dog对象，直接为其定义一个和Cat相同的抓老鼠技能。

方案B：让Doggy吃掉Kitty，直接消化吸收Kitty的所有能力。

其实方案A和方案B的解决方法是类似的，也是需要修改Dog对象，不过方案B会更简单粗暴一点：

```javascript
class Dog {
    constructor (name, kitty){
        this.name = name;
        this.catchMouse = kitty.catchMouse;
    }
    
    biteCriminal(name1, name2){
        console.log(`${this.name} bite 2 criminals! Their name is ${name1} and ${name2}.`);
    }
}

const kitty = new Cat('Kitty');
const doggy = new Dog('Doggy', kitty);

doggy.catchMouse('Mickey', 'Minnie');
// Doggy caught 2 mouse! They call Mickey and Minnie.
```

上面这种方法实在是太不优雅，往往很多时候在定义Dog对象的时候根本就没有打算过要为它添加抓老鼠的方法。那么有没有一种方法能够在不修改Dog对象内容的前提下，让Doggy实例也能够拥有抓老鼠的方法呢？答案就是使用call，apply或者bind 关键字：

```javascript
kitty.catchMouse.call(doggy, 'Mickey', 'Minnie');

kieey.catchMouse.apply(doggy, ['Mickey', 'Minnie']);

const doggyCatchMouse = kitty.catchMouse.bind(doggy, 'Mickey', 'Minnie');
doggyCatchMouse();
```

看到这里，相信读者已经能够明白call，apply和bind的区别及作用。

回到文章开头的等式，其实这里的“等号”并不严谨，因为三个关键字的区别及背后的原理肯定不是区区一个等号就能够概括的，但对于概念的理解以及实际情况下的运用来说，这条等式未必不是一个好的思路。