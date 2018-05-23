## 对象的隐式转换

当对象之间相加obj1 + obj2或相减obj1 -  obj2或者alert(obj)会发生什么？当某个对象出现在了需要原始类型才能进行操作的上下文时，JavaScript会将对象转换成原始类型。在进行转换中的对象有特殊的方法

* 对于对象而言，不存在布尔转换，因为所有对象在上下文中布尔值都为true，所以只有数字和字符串转换
* 数组转换发生在减去对象或者应用数学函数时，例如，Date对象可以比减去，结果是date1 - date2两个日期之间的时间差
* 字符串转换，当我们alert(obj)在类似的上下文中输出对象时，通常会发生这种情况

### ToPrimitive

当我们在需要原始类型的上下文中使用对象时，例如在alert或者数学运算中，使用ToPrimitive算法将其转换为原始类型值。该算法允许我们使用特殊的对象方法自定义转换，根据上下文，转换具有所谓的提示

* string 当一个操作期望一个字符串时，对于对象到字符串的转换，如alert

  ```javascript
  alert(obj);
  
  // 或者使用对象来作为属性
  anotherObj[obj] = 123;
  ```

* number 当一个操作需要数字时，用于对象到数学的转换。例如

  ```javascript
  let num = Number(obj);
  
  let n = +obj;
  let delta = date1 - date2;
  
  let greater = user1 > user2;
  ```

* default 在少数情况下发生，当操作不确定期望的类型时。+这种运算符借款已进行字符串拼接也可以进行数学运算，所以字符串和数字都可以。或者当一个对象与字符串，数字或符号来判断是否相等时

  ```javascript
  let total = car1 + car2;
  
  if(user == 1){ ... };
  ```

  大于小于运算符<>可以同时处理字符串和数字。不过，它使用number提示，而不是default提示，这是历史原因。在JavaScript中，除了一个特例（Date对象），其他的内置对象都按照与default相同的方式实现转换number。

### 为了进行转换，JavaScript会尝试查找并调用三个对象方法

1. 调用obj[Symbol.toPrimitive]\(init)如果方法存在
2. 否则，如果提示是string
   * 尝试obj.toString()和obj.valueOf()。
3. 否则，如果提示是number或default
   * 尝试obj.valueOf()和obj.toString()

### Symbol.toPrimitive

例子如下：

```javascript
let user = {
    name: 'john',
    money: 1000,
    [Symbol.toPrimitive](hint){
        console.log(hint);
        return hint === 'string' ? this.name: this.money;
    }
};

console.log(`${user}`);
// string
// john
console.log(+user);
// Number
// 1000
console.log(user === 1000);
// default
// true
```

### toString()和valueOf()

toString()一级valueOf从远古时代到来，他们不是符号，而是常规字符串命名的方法。他们提供了一种替代老式的方法来实现的转换。如果没有Symbol.toPrimitive那么JavaScript会尝试查找他们并按顺序尝试：

* toString  ->  valueOf  为字符串提示
* valueOf  ->  toString  除此之外

```javascript
let user = {
    name: 'john',
    money: 1000,
    toString(){
        return this.name;
    },
    valueOf(){
        return this.money;
    }
};
console.log(`${user}` + 1);	// john1
console.log(+user);	// 1000
console.log(user === 1000);	// true
```

最后附上一张JavaScript原始类型转换表

![JavaScript原始类型转换表](https://user-gold-cdn.xitu.io/2018/3/29/16271ed68f1b8236?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

## 对象的遍历

### for...in循环

为了遍历对象的所有键，存在一个特殊的循环形式：for...in。

```javascript
let user = {
    name: 'john',
    age: 30,
    isAdmin: true
};

for(let key in user){
    // keys
    alert(key);	// name, age, isAdmin
    alert(user[key]);	// john, 20 ,true
}
```

对象遍历的顺序并不是按照添加顺序创建的，而是具有一定规则的，先是整数属性排序，其他的则以创建顺序出现。

```javascript
let codes = {
    "49": "Germany",
    "41": "Switzerland",
    "44": "Great Britain",
    // ...
    "1": "USA"
};
for(let code in codes){
    console.log(code);	// 1, 41, 44, 49;
}
```

使用for...in遍历对象是无法直接获取属性值的，因为它实际上遍历的是对象中所有可枚举的属性，你需要手动获取属性值。而在ES6中我们可以借助for...of和Iterator来直接获取属性值。简单介绍下Iterator，在ES6中新添了Map和Set，加上原有的数据结构，用户还可以组合使用它们，因此需要统一的接口机制，来处理不同的数据结构。遍历器（Iterator）就是这样一种结构，只要在数据结构中部署它，就可以完成遍历操作，Iterator的遍历过程是这样的

1. 创建一个指针对象，指向当前数据结构的起始位置

2. 第一次调用指针对象的next方法，可以将指针指向数据结构的第一个成员

3. 第二次调用指针对象的next方法，指正就指向数据结构的第二个成员

4. 不断调用指针对象的next方法，知道它指向数据结构的结束位置，每一次调用next方法就会返回一个包含value和done两个属性的对象。其中，value属性是当前成员的值，done属性是一个布尔值，表示遍历是否结束。当我们使用for...of循环遍历某种数据结构时，该循环会自动去寻找Iterator接口。结合着两点我们可以创建对象的遍历器接口

   ```javascript
   let obj = {
       a: 1,
       b: 2
   };
   Object.defineProperty(obj, Symbol.iterator, {
       value(){
           var o = this;
           var idx = 0;
           var k = Object.keys(o);
           return{
               next(){
                   return {
                       value: o[k[indx++]],
                       done: (idx > k.length)
                   };
               }
           };
       }
   });
   ```

## 对象的克隆

对象与原始类型之间的根本区别之一就是它们通过引用来存储和赋值，原始类型值：string，number，boolean被分配和复制为整体值，例如：

```javascript
let message = "hello";
let pharse = message;
```

因此我们有两个独立的变量，每个变量都存储字符串hello。

![存储字符串](https://user-gold-cdn.xitu.io/2018/3/29/16271ed70123f52e?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

对象不是这样的，对象存储的是其值的内存地址，而非值本身。

```javascript
let user = {
	name: 'john'
};
```

![对象存储](https://user-gold-cdn.xitu.io/2018/3/29/16271ed700eb3299?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

当一个变量被赋值为对象时，赋值的是其值的内存地址（引用），而不是对象本身，我们可以将一个对象想象成一个橱柜，那么变量就是橱柜的钥匙，赋值变量就会赋值钥匙，但不是橱柜本身。

```javascript
let user = {name: "john"};
let admin = user;
```

现在我们有两个变量，每个变量都引用同一个对象

![引用](https://user-gold-cdn.xitu.io/2018/3/29/16271ed70292e89f?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

我们可以使用任何变量来访问控制橱柜并修改内容

```javascript
let user = {name: 'john'};
let admin = user;
   
admin.name = 'pete';
alert(user.name);	// "pete"
```

只有两个对象是同一个对象时，他们才是相等的。

例如两个变量引用同一个对象，他们是相等的。

   ```javascript
let a = {};
let b = a;
   
console.log( a == b) // true
console.log( a === b) // true
   ```

这里两个独立的对象是不相等的，尽管他们都是空的

   ```javascript
let a = {};
let b = {};
   
console.log( a == b); // false
   ```

因此复制一个对象变量会创建对同一个对象的引用。但是如果我们需要复制一个对象呢？我们需要创建一个新对并通过遍历它的属性来复制现有对象的结构，例如：

   ```javascript
let user = {
  name: 'john',
  age: 30
}
   
let clone = {}; 
   
for(let key in user){
  clone[key] = user[key];
}
   
clone.name = 'pete';
   
console.log(user.name);
   ```

我们也可以使用Object.assgin方法。

   ```javascript
let user = {
  name: 'john',
  age: 30
}
   
let clone = Object.assign({}, user);
   ```

到现在为止，我们认为所有的属性user都是原始类型的，但属性可以是对其他对象的引用，如何处理他们，例如这个

   ```javascript
let user = {
    name: 'john',
    sizes: {
        height: 182,
        width: 50
    }
};
   
let clone = Object.assign({}, user);
console.log(user.sizes === clone.sizes);
   
user.sizes.width++;
console.log(clone.sizes.width); // 51
   ```

为了解决这个问题，我们应该递归检查每个值的类型，如果它是一个对象，就复制它的结构，这就是所谓的深度克隆，简单的例子:

   ```javascript
let user = {
    name: 'john',
    size: {
        height: 182,
        width: 50
    }
};
let cloneObj = {};
   
function clone(source, target){
    let keys = Object.keys(source);
    for(let key of keys){
        if(typeof source[key] === 'object'){
            target[key] = clone(source[key], {});
        } else {
            target[key] = source[key];
        }
    }
    return target;
}
   
clone(user, cloneObj);
user.size.width++;
console.log(cloneObj);
   ```

而在HTML5规范中提出了一种用于深层克隆的标准算法看，用于处理上述情况和更复杂的情况，称为[结构化克隆算法](https://developer.mozilla.org/zh-CN/docs/Web/Guide/API/DOM/The_structured_clone_algorithm)

关于结构化克隆的好处是在于他处理循环对象并支持大量内置类型，问题在于算法并不对用户直接暴露，只能作为API的一部分。

###MessageChannel

只要你调用postMessage结构化克隆算法就可以使用，我们可以创建一个MessageChannel并发送消息。在接收端，消息包含我们原始数据对象的结构化克隆。

   ```javascript
function strcuralClone(obj){
    return new Promise(resolve => {
        const {port1, port2} = new MessageChannel();
        port2.ommessage = ev => resolve(ev.data);
        port1.postMessage(obj);
    })
}
const user = {
    a: 1,
    b: {
        c: 2
    }
};
   
user.c = user;
const clone = strucuralClone(user);
clone.then((result) => console.log(result));
   ```

   ### History API

如果你曾经使用history.pushState()，那么您可以提供一个状态来保存URL。事实证明，这个状态对象在结构上被同步克隆。同时我们必须小心，不要混淆可能使用状态对象的任何程序逻辑，所以我们需要在完成克隆后恢复原始状态。为了防止发生任何事件，请使用history.replaceState()而不是history.pushState()；

   ```javascript
function strucuralClone(obj){
    const oldState = history.state;
    history.replaceState(obj, document.title);
    const copy = history.state;
    history.replaceState(oldState, document.title);
    return copy;
}
const user = {
    a: 1,
    b: {
        c: 2,
    }
};
user.c = user;
const clone = strcuralClone(user);
console.log(clone);
   ```

   ### Notification API

   ```javascript
function strucuralClone(obj){
    return new Notification('', {data: obj, slient: true});
}
const user = {
    a: 1, 
    b: {
        c: 2,
    }
}
user.c = user;
user.a = 2;
const clone = strucuralClone(user);
user.a = 3;
console.log(clone);   
   ```

   ###结构化克隆优于JSON的地方

   * 结构化克隆可以复制RegExp对象
   * 结构化克隆可以复制Blob，File以及FileList对象
   * 结构化克隆可以复制ImageData对象
   * 结构化克隆可以正确地复制有循环引用的对象

### 结构化克隆不能做到的

* Error以及Function对象是不能被结构化克隆算法复制的
* 企图克隆DOM节点同样会抛出错误
* 对象的某些特定参数也不会被保留
* 原型链上的属性也不会被追踪以及复制

性能比较

![深复制的性能对比](https://user-gold-cdn.xitu.io/2018/3/29/16271ed7161c601d?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

这些克隆方式只是黑科技，在项目中还是乖乖用lodash提供的clone方法吧。

### 对象的不可变性

有时候你会希望属性或者对象是不可改变的，在ES5中可以通过多种方法来实现

   