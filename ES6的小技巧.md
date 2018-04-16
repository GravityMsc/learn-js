## 1.  强制参数

ES6提供了默认参数的概念，当函数的参数未传入或者传入值为undefined时，会应用参数的默认值。

默认值可以是个表达式，所以我们可以将默认值设置为一个执行函数，如果该参数没有传值，就会执行我们的默认函数：

```javascript
const required = () => { throw new Error('Missing parameter')};

//The below function will throw an error if either "a" or "b" is missing.
const add = (a = required(), b = required()) => a + b;

add(1, 2)	// 3
add(1)	//Error: Missing parameter.
```



## 2.  强大的reduce

reduce是一个多才多艺的数组方法，来看下。

### 2.1 使用reduce替代map + filter

设想你有这么个需求：要把数组中的值进行计算后再过滤掉一些值，然后输出新数组。很显然我们一般使用map和filter方法组合来达到这个目的，但这也意味着你需要迭代这个数组两次。

来看看我们如何使用reduce只迭代数组一次，来完成同样的结果。下面这个例子我们需要把数组中的值乘2，并返回大于50的值：

```javascript
const numbers = [10, 20, 30, 40];
const doubleOver50 = numbers.reduce((finalList, num) => {
    num = num * 2; // double each number (i.e. map)
    
    // filter number > 50
    if(num > 50){
        finalList.push(num);
    }
    return finalList;
}, []);

doubleOver50; // [60, 80]
```

### 2.2 使用reduce检测括号是否对齐封闭

下面这个例子我们用reduce来检测一段string中的括号是否前后对应封闭。

思路是顶一个名为counter的变量，它的初始值为0，然后迭代字符串，迭代过程中碰到（就加1，碰到）就减一，如果括号前后对应的话，最终counter的值会是0。

```javascript
// Returns 0 if balanced.
const isParentsBalanced = (str) => {
    return str.split('').reduce((counter, char) => {
        if(counter < 0){ // matched ")" before "("
            return counter;
        } else if(char === '(') {
            return ++counter;
        } else if(char === ')') {
            return --counter;
        } else { // matched some other char
            return counter;
        }
    }, 0); // <--- starting value of the counter
}

isParensBalanced('(())') // <--- balanced
isParensBalanced('(asdfds') // <--- balanced
isParensBalanced('(()') // <--- not balanced
isParensBalanced(')(') // <--- not balanced
```

### 2.3 使用reduce计算数组中的重复项

如果你想计算数组中的每个值有多少重复值，reduce也可以快速帮到你。下面的例子我们计算数组中每个值的重复数量，并输出一个对象来展示：

```javascript
var cars = ['BMW', 'Benz', 'Benz', 'Tesla', 'BMW', 'Toyota'];
var carsObj = cars.reduce(function (obj, name){
    obj[name] = obj[name] ? ++obj[name]: 1;
    return obj;
}, {});

carsObj; // => { BMW: 2, Benz: 2, Tesla: 1, Toyota: 1 }
```

reduce的确非常强大，建议参考MDN文档仔细看一遍：[Reduce_MDN](https://link.juejin.im/?target=https%3A%2F%2Fdeveloper.mozilla.org%2Fen-US%2Fdocs%2FWeb%2FJavaScript%2FReference%2FGlobal_Objects%2FArray%2FReduce)



## 3.  对象解构

### 3.1  移除对象的多余属性

有时你可能希望移除一个对象中的某些属性，我们一般会通过迭代这个对象（如for..in循环）来移除那些我们不想要的属性。实际上我们可以通过对象解构的方法将不想要的属性提取出来，并将想留下的变量保存在**rest**参数中。

在下面的这个例子中，我们从对象中移除_internal和tooBig这两个属性：

```javascript
let {_internal, tooBig, ...cleanObject} = {el1: '1', 
                                           _internal:"secret", 
                                           tooBig:{},
                                          el2: '2',
                                          el3: '3'};
console.log(cleanObject);	// {el1: '1', el2: '2', el3: '3'}
```

### 3.2  嵌套对象解构

下面的示例中，engine属性是对象car内的嵌套对象。我们可以通过对象解构快速的获取到engine中的属性，比如vin：

```javascript
var car = {
    model: 'bmw 2018',
    engine: {
        v6: true,
        turbo: true,
        vin: 12345
    }
};

const modelAndVIN = ({model, engine: {vin}}) => {
    console.log(`model: ${model} vin: ${vin}`);
}

modelAndVIN(car);	// => model: bmw 2018 vin: 12345
```

### 3.3  合并对象

ES6增加了展开运算符（也就是3个点），展开运算符常常用在处理数组解构上，在对象的解构上它也同样好用。

下面的示例中我们合并两个对象，新对象中相同的属性会被放在后面的对象覆盖：

```javascript
let object1 = { a: 1, b: 2, c: 3};
let object2 = { b: 30, c: 40, d: 50};
let merged = {...object1, ...object2} // spread and re-add into merged

console.log(merged)	// {a: 1, b: 30, c: 40, d: 50}
```



## 4.  使用Sets数组去重

使用Sets可以快速给数组去重，因为Sets中的值不可重复。

```javascript
let arr = [1, 1, 2, 2, 3, 3];
let deduped = [...new Set(arr)]	// [1, 2, 3]
```



## 5.  数组解构

### 5.1  交换变量的值

```javascript
let param1 = 1;
let param2 = 2;

// swap and assign param1 & param2 each other values
[param1, param2] = [param2, param1];
console.log(param1)	// 2
console.log(param2)	// 1
```

### 5.2  从函数接受和分配多个值

很多时候你的函数都会讲多个数据放在数组内，以返回一个单一值（例如Promise函数，它的决议值只能是个单一值），我们可以使用数组解构简便的从返回结果中获取这些值。

下面的这个例子中，我们使用fetch发送了两个请求，并使用Promise.all()将两个结果保存在数组中，函数的执行结果是返回这个数组。

```javascript
function getFullPost(){
    return Promise.all([
        fetch('/post'),
        fetch('/comments')
    ]);
}

// 在async函数中
const [post, comments] = await getFullPost();

(async function xxxx(){
    const [post, comments] = await getFullPost();
    console.log(post, comments);
})();
```


