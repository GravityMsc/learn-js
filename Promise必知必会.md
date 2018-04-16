Promise想必大家都十分熟悉，想想就那么几个api，可是你真的了解Promise吗？本文根据Promise的一些知识点总结了十道题，看看你能做对几道。

## 题目一

```javascript
const promise = new Promise((resolve, reject) => {
    console.log(1);
    resolve();
    console.log(2);
});
promise.then(() => {
    console.log(3);
});
console.log(4);
```

运行结果：

> 1 
>
> 2
>
> 4
>
> 3

**解释**：Promise构造函数是同步执行的，promise.then中的函数是异步执行的。

## 题目二

```javascript
const promise1 = new Promise((resolve, reject) => {
    setTimeout(() => {
        resolve('success');
    }, 1000);
});
const promise2 = promise1.then(() => {
    throw new Error('error!!!');
});

console.log('pormise1', promise1);
console.log('promise2', promise2);

setTimeout(() => {
    console.log('promise1', promise1);
    console.log('promise2', promise2);
}, 2000);
```

运行结果：

> ```
> promise1 Promise { <pending> }
> promise2 Promise { <pending> }
> (node:50928) UnhandledPromiseRejectionWarning: Unhandled promise rejection (rejection id: 1): Error: error!!!
> (node:50928) [DEP0018] DeprecationWarning: Unhandled promise rejections are deprecated. In the future, promise rejections that are not handled will terminate the Node.js process with a non-zero exit code.
> promise1 Promise { 'success' }
> promise2 Promise {
>   <rejected> Error: error!!!
>     at promise.then (...)
>     at <anonymous> }
> ```

**解释**：promise有3种状态：pending、fulfilled或rejected。状态改变只能是pending->fufilled或者pending->rejected，状态一旦改变则不能再变。上面promise2并不是promise1，而是返回一个新的Promise实例。

## 题目三

```javascript
const promise = new Promise((resolve, reject) => {
    resolve('success1');
    reject('error');
    resolve('success2');
});

promise
    .then((res) => {
    console.log('then: ', res);
})
    .catch((err) => {
    console.log('catch: ', err);
});
```

运行结果：

> then: success1

**解释**：构造函数中的resolve或reject只有第一次执行有效，多次调用没有任何作用，呼应代码二结论：promise状态一旦改变则不能再变。

## 题目四

```javascript
Promise.resolve(1)
    .then((res) => {
    console.log(res);
    return 2;
})
    .catch((err) => {
    return 3;
})
    .then((res) => {
    console.log(res);
});
```

运行结果：

> 1
>
> 2

**解释**：promise可以链式调用。提起链式调用我们通常会想到通过return this实现，不过Promise并不是这样实现的。promise每次调用 .then或者 .catch都会返回一个新的promise，从而实现了链式调用。

## 题目五

```javascript
const promise = new Promise((resolve, reject) => {
    setTimeout(() => {
        console.log('once');
        resolve('success');
    }, 1000);
});

const start = Date.now();
promise.then((res) => {
    console.log(res, Date.now - start);
});
promise.then((res) => {
    console.log(res, Date.now - start);
});
```

运行结果：

> once
>
> success 1005
>
> success 1007

**解释**：promise的 .then或者 .catch可以被调用多次，但这里Promise构造函数只执行一次。或者说promise内部状态一经改变，并且有了一个值，那么后续每次调用 .then或者 .catch都会直接拿到该值。

## 题目六

```javascript
Promise.resolve()
    .then(() => {
    return new Error('error!!!');
	})
    .then((res) => {
    console.log('then: ', res);
	})
    .catch((err) => {
    console.log('catch: ', err);
});
```

运行结果：

> ```
> then: Error: error!!!
>     at Promise.resolve.then (...)
>     at ...
> ```

**解释**：.then或者.catch中return一个error对象并不会抛出错误，所以不会被后续的.catch捕获，需要改成下面两种中的一种才可以。

1. return Promise.reject(new Error('error!!!'))
2. throw new Error('error!!!')

因为返回任意一个非promise的值都会白包裹成promise对象，即return new Error('error!!!')等价于return Promise.resolve(new Error('error!!!'))。

## 题目七

```javascript
const promise = Promise.resolve()
.then(() => {
    return promise;
});
promise.catch(console.error);
```

运行结果：

> ```
> TypeError: Chaining cycle detected for promise #<Promise>
>     at <anonymous>
>     at process._tickCallback (internal/process/next_tick.js:188:7)
>     at Function.Module.runMain (module.js:667:11)
>     at startup (bootstrap_node.js:187:16)
>     at bootstrap_node.js:607:3
> ```

**解释**：.then或.catch返回的值不能是promise本身，否则会造成死循环。类似于：

```javascript
process.nextTick(function tick(){
    console.log('tick');
    process.nextTick(tick);
});
```

## 题目八：

```javascript
Promise.resolve(1)
	.then(2)
	.then(Promise.resolve(3))
	.then(console.log);
```

运行结果：

> 1

**解释**：.then或者.catch的参数期望是函数，传入非函数则会发生值穿透。

## 题目九

```javascript
Promise.resolve()
    .then(function success(res){
    throw new Error('error');
}, function fail1(e){
    console.log('fail1: ', e);
})
    .catch(function fail2(e){
    console.log('fail2: ', e);
});
```

运行结果：

> ```
> fail2: Error: error
>     at success (...)
>     at ...
> ```

**解释**：.then可以接收两个从哪火速，第一个是处理成功的函数，第二个是处理错误的函数。.catch是.then第二个参数的简便写法，但是它们用法上有一点需要注意：.then的第二个处理错误的函数捕获不了第一个处理成功的函数抛出的错误，而后续的.catch可以捕获之前的错误。当然以下代码也可以：

```javascript
Promise.resolve()
  .then(function success1 (res) {
    throw new Error('error')
  }, function fail1 (e) {
    console.error('fail1: ', e)
  })
  .then(function success2 (res) {
  }, function fail2 (e) {
    console.error('fail2: ', e)
  })
```

## 题目十

```javascript
process.nextTick(() => {
    console.log('nextTick');
});
Promise.resolve()
    .then(() => {
    console.log('then');
});
setImmediate(() => {
    console.log('setImmediate');
});
console.log('end');
```

运行结果：

> end
>
> nextTick
>
> then
>
> setImmediate

**解释**：process.nextTick和promise.then都属于microtask，而setImmediate属于macrotask（event loop的check阶段），在事件循环的check阶段执行。事件循环的每个阶段（macrotask）之间都会执行microtask，事件循环的开始会执行一次microtask。