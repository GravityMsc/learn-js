一直想写一篇关于event loop的文章，前不久发现CNode上有位同学写了一篇原理分析的文章很详细，这里我就不献丑了。本文就拿出六道题来补充一下，放出一张我认为非常直观的图。

![event loop](https://pic4.zhimg.com/80/v2-3a59c624e6ff95a7e8c5a23c979f5abe_hd.jpg)

绿色小块是macrotask（宏任务），macrotask中间的粉红箭头是microtask（微任务）。

## 题目一

```javascript
setTimeout(() => {
    console.log('setTimeout');
}, 0);

setImmediate(() => {
    console.log('setImmediate');
});
```

运行结果是：

> setImmediate
>
> setTimeout

或者：

> setTimeout
>
> setImmediate

为什么结果不确定呢？

**解释**：setTimeout/setInterval的第二个参数取值范围是：[1, 2^31 - 1]，如果超过这个范围则会初始化为1，即setTimeout(fn,  0) === setTimeout(fn,  1)。我们知道setTimeout的回调函数在timer阶段执行，setImmediate的回调函数在check阶段执行，event loop的开始会先检查timer阶段，但是在开始之前到timer阶段会消耗一定时间，所以就会出现两种情况：

1. timer前的准备时间超过1ms，满足loop->time >= 1，则执行timer阶段（setTimeout）的回调函数。
2. timer前的准备时间小于1ms，则先执行check阶段（setImmediate）的回调函数，下一次event loop执行timer阶段（setTimeout）的回调函数。

再看个例子：

```javascript
setTimeout(() => {
    console.log('setTimeout');
}, 0);

setImmediate(() =>{
    console.log('setImmediate');
});

const start = Date.now();
while(Date.now() - start < 10);
```

运行结果一定是：

> setTimeout
>
> setImmediate

## 题目二

```javascript
const fs = require('fs');

fs.readFile(__filename, () => {
    setTimeout(() => {
        console.log('setTimeout');
    }, 0);
    
    setImmediate(() => {
        console.log('setImmediate');
    });
});
```

运行结果是：

> setImmediate
>
> setTimeout

**解释**：fs.readFile的回调函数执行完后：

1. 注册setTimeout的回调函数到timer阶段。
2. 注册setImmediate的回调函数到check阶段。
3. event loop从poll阶段出来继续往下一个阶段执行，恰好是check阶段，所以setImmediate的回调函数先执行。
4. 本次event loop结束后，进入下一次event loop，执行setTimeout的回调函数。

所以，在I/O Calllbacks中注册的setTimeout和setImmediate，永远都是setImmediate先执行。

## 题目三

```javascript
setInterval(() => {
    console.log('setInterval');
}, 100);

process.nextTick(function tick() {
    process.nextTick(tick);
});
```

运行结果：setInterval永远不会打印出来。

**解释**：process.nextTick会无限循环，将event loop阻塞在microtask阶段，导致event loop上其他macrotask阶段的回调函数没有机会执行。

解决办法通常是用setImmediate替代process.nextTick，如下：

```javascript
setInterval(() => {
    console.log('setInterval');
}, 100);

setImmediate(function immediate() {
    setImmediate(immediate);
});
```

运行结果：每100ms打印一次setInterval。

**解释**：process.nextTick内执行process.nextTick仍然将tick函数注册到当前microtask的尾部，所以导致microtask永远执行不完；setImmediate内执行setImmediate会将immediate函数注册到下一次event loop的check阶段，而不是当前正在执行的check阶段，所以给了event loop上其他macrotask执行的机会。

再看个例子：

```javascript
setImmediate(() => {
    console.log('setImmediate1');
    setImmediate(() => {
        console.log('setImmediate2');
    });
    process.nextTick(() => {
        console.log('nextTick');
    });
});

setImmediate(() => {
    console.log('setImmediate3');
});
```

运行结果是：

> setImmediate1
>
> setImmediate3
>
> nextTick
>
> setImmediate2

**注意**：并不是说setImmediate可以完全替代process.nextTick，process.nextTick在特定场景下还是无法被替代的，比如我们就想将一些操作放到最近的microtask里执行。

## 题目四

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

**解释**：promise.then类似于process.nextTick，都会讲回调函数注册到microtask阶段。上面代码会导致死循环，类似前面提到的：

```javascript
process.nextTick(function tick(){
    process.nextTick(tick);
});
```

再看个例子：

```javascript
const promise = Promise.resolve();

promise.then(() => {
    console.log('promise');
});

process.nextTick(() => {
    console.log('nextTick');
});
```

运行结果：

> nextTick
>
> promise

**解释**：promise.then虽然和process.nextTick一样，都将回调函数注册到microtask，但优先级不一样。process.nextTick的microtask queue总是优先于promise的microtask queue执行。

## 题目五

```javascript
setTimeout(() => {
    console.log(1);
}, 0);

new Promise((resolve, reject) => {
    console.log(2);
    for(let i = 0; i < 10000; i++){
        i === 9999 && resolve();
    }
    console.log(3);
}).then(() => {
    console.log(4);
});
console.log(5);
```

运行结果：

> 2
>
> 3
>
> 5
>
> 4
>
> 1

**解释**：Promise构造函数是同步执行的，所以先打印2、3，然后打印5，接下来event loop进入执行microtask阶段，执行promise.then的回调函数打印出4，然后执行下一个macrotask，恰好是timer阶段的setTimeout的回调函数，打印出1。

## 题目六

```javascript
setImmediate(() => {
    console.log(1);
    setTimeout(() => {
        console.log(2);
    }, 100);
    setImmediate(() => {
        console.log(3);
    });
    process.nextTick(() => {
        console.log(4);
    });
});
process.nextTick(() => {
    console.log(5);
    setTimeout(() => {
        console.log(6);
    }, 100);
    setImmediate(() => {
        console.log(7);
    });
    process.nextTick(() => {
        console.log(8);
    });
});
console.log(9);
```

运行结果：

> 9
>
> 5
>
> 8
>
> 1
>
> 7
>
> 4
>
> 3
>
> 6
>
> 2

**解释**：

> 1. 首先两个大的函数都是回调，会加入到event loop中执行，而9是立即执行函数，所以先输出9.。
> 2. process.nextTick是加入到最近的microtask（微任务队列中）中执行，由文件最开始的图可知，在timers之前有一个微任务队列，而setImmediate是在check阶段执行。因此先执行第二个大函数，先输出立即执行的5。
> 3. 然后遇到第二个大函数中的setTimeout，它会被加入到timers队列，然后是setImmediate，被加入到check队列，然后是process.nextTick加入到最近的microtask队列，即当前正在运行的队列，因此会输出8。
> 4. 然后是进入event loop的timers阶段，这个阶段的任务队列中有第三步加入的一个100ms延时函数，因此会被延时执行，假如此处是0ms，则会立即执行。然后是event loop进入check阶段，check阶段的任务队列中有两个函数，分别是第二步加入的setImmediate和第三步加入的setImmediate，由队列先进先出原则，先执行第二步加入的setImmediate函数，于是输出第一个大函数的立即执行的1。然后是100ms延时的setTimeout函数加入到下一次event loop的timers队列中，setImmediate加入到下一次event loop的check队列中，然后是process.nextTick加入到最近的microtask队列中。
> 5. 这时check阶段的第一个setImmediate函数执行完毕，开始执行队列中第二个setImmediate函数，就是第三步加入的那个setImmediate函数，因此输出7。
> 6. 然后是最近的microtask队列，其中有在第四步加入的一个process.nextTick函数，因此输出4。
> 7. 然后执行完整个event loop，没有函数在队列中等待执行，重启event loop，开始执行第二次的timers阶段。检查timers队列中有两个setTimeout函数，分别送是第三步和第四步加入的，但是延时都是100ms，此时还没到时间，因此略过，进入第二次的check阶段，此时check队列中有第四步中加入的setImmediate函数，因此输出3。
> 8. 然后event loop执行完毕，重启event loop，重新进入timers阶段，（假如此时，实际上因为没有函数在event loop中，因为100ms实际上对于程式来说是很长的一段时间，event loop可能会空循环，等待100ms的延时到来）时间已到，timers队列中的函数终于可以执行，首先执行在第二步加入的函数，输出6。
> 9. 然后执行第四步加入的函数输出2。