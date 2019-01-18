## 一篇文章和一道面试题

最近，一篇名为《[8张图帮你一步步看清async/await和promise的执行顺序](https://segmentfault.com/a/1190000017224799)》的文章引起了我的关注。

作者用一道2017年【今日头条】的前端面试题作为引子，分步讲解了最终结果的执行原因。其中涉及到了不少概念，比如异步的执行顺序，宏任务，微任务等，同时作者限定了执行范围，以浏览器的event loop机制为准。下面是原题的代码：

```javascript
async function async1() {
    console.log('async1 start');
    await async2();
    console.log('async1 end');
}

async function async2() {
    console.log('async2');
}

console.log('script start');

setTimeout(function() {
    console.log('setTimeout');
}, 0);

async1();

new Promise(function (resolve) {
    console.log('promise1');
    resolve();
}).then(function() {
    console.log('promise2');
});

console.log('script end');
```

紧接着，作者先给出了答案。并希望读者先行自我测试。

```
script start
async1 start
async2
promise1
script end
promise2
async1 end
setTimeout
```

我在看这道题的时候，先按照自己的理解写出了结果。

```
script start
async1 start
async2
promise1
script end
async1 end
promise2
setTimeout
```

## 一些重要的概念

这里需要先简单地说一些event loop的概念。

* JavaScript是单线程的，所有的同步任务都会在主线程中执行。
* 主线程之外，还有一个任务队列。每当一个异步任务有结果了，就往任务队列里面塞一个事件。
* 当主线程中的任务，都执行完之后，系统会“依次”读取任务队列里面的事件。与之相应的异步任务进入主线程，开始执行。
* 异步任务之间，会存在差异，所以它们执行的优先级也会有区别。大致分为 微任务（micro rask，如：Promise、MutationObserver等）和宏任务（macro task，如：setTimeout、setInterval、I/O等）。同一次事件循环中，微任务永远在宏任务之前执行。
* 主线程会不断重复上面的步骤，直到执行完所有任务。

另外，还有async/await的概念。

* async函数，可以理解是Generator函数的语法糖。
* 它建立在promise之上，总是与await一起使用的。
* await会返回一个Promise对象，或者一个表达式的值。
* 其目的是为了让异步操作更优雅，能像同步一样地书写。

## 我的理解

再说说我对这道题的理解。

* 首先，从console的数量上看，会输出8行结果。
* 再瞟了一样代码，看到了setTimeout，于是，默默地把它填入第8行。
* 在setTimeout附近，看到了console.log('script start')和async1()，可以确认它们是同步任务，会先在主线程中执行。所以，妥妥地在第1行填入script start，第2行填入async1方法中的第一行 async1 start。
* 接下来，遇到了await。从字面意思理解，让我们等等。需要等待async2()函数的返回，同时会阻塞后面的代码。所以，第3行填入async2。
* 讲道理，await都执行完了，该轮到console.log( 'async1 end' )的输出了。但是，别忘了下面还有个Promise，有一点需要注意的是：当 new 一个 Promise的时候，其 resolve 方法中的代码会立即执行。如果不是 async1()的 await 横插一杠，promise1 可以排得更前面。所以，现在第4行填入 promise1。
* 再接下来，同步任务console.log( 'script end' ) 执行。第5行填入 script end。
* 还有第6和第7行，未填。回顾一下上面提到 async/await 的概念，其目的是为了让异步能像同步一样地书写。那么，我认为 console.log( 'async1 end' ) 就是个同步任务。所以，第6行填入async1 end。
* 最后，顺理成章地在第7行填入 promise2。

## 与作者答案的不同

回过头对比与作者的答案，发现第6行和第7行的顺序有问题。

再耐心地往下看按文章，反复地看了几遍async1 end和promise2谁先谁后，还是无法理解为何在chrome浏览器中，promise2会先于async1 end输出。

然后，看到评论区，发现也有人提出了相同的疑惑。[@rhinel](https://link.juejin.im?target=https%3A%2F%2Fsegmentfault.com%2Fu%2Frhinel)提出，在他的72.0.3622.0（正式版本）dev（64 位）的chrome中，跑出来的结果是 async1 end 在 promise2 之前。

**随即我想到了一种可能，JS的规范可能会在未来有变化。于是，我用自己的react工程试了一下（工程中的babel-loader版本为7.1.5。.babelrc的presets设置了stage-3），结果与我的理解一致。当前的最新版本 chromeV71，在这里的执行顺序上，的确存在有问题。**

于是，我也在评论区给作者留了言，进行了讨论。@rhinel最后也证实，其实最近才发布通过了这个顺序的改进方案，这篇 [《Faster async functions and promises》](https://link.juejin.im?target=https%3A%2F%2Fv8.dev%2Fblog%2Ffast-async) 详细解释了这个改进，以及实现效果。不久之后，作者也在他文章的最后，补充了我们讨论的结果，供读者参考。



## 理解1

这个问题涉及以下3点：

1. async函数的返回值
2. Promise链式then()的执行时机
3. async函数中的await操作符到底做了什么

下面一一回答：

1. async函数的返回值：

   * 被async操作符修饰的函数必然返回一个Promise对象

   * 当async函数返回一个值时，Promise的resolve方法负责传递这个值

   * 当async函数抛出异常时，Promise的reject方法会传递这个异常值

   * 所以，以示例代码中async2为例，其等价于

     ```javascript
     function async2(){
         console.log('async2');
         return Promise.resolve();
     }
     ```

2. Promise链式then()的执行时机

   * 多个then()链式调用，并不是连续的创建了多个微任务并推入微任务队列，因为then()的返回值必然是一个Promise，而后续的then()是上一步then()返回的Promise的回调

   * 以示例代码为例

     ```javascript
     ...
     
     new Promise(function(resolve){
         console.log('promise1');
         resolve();
     }).then(function(){
         console.log('promise2');
     }).then(function(){
         console.log('promise3');
     })
     
     ...
     ```

     * Promise构造器内部的同步代码执行到resolve()，Promise的状态改变为fulfillment，then中传入的回调函数console.log('promise2')作为一个微任务推入微任务队列
     * 而第二个then中传入的回调函数console.log('promise3')还没有被推入微任务队列，只有上一个then中的console.log('promise2')执行完毕后，console.log('promise3')才会被推入微任务队列，这是一个关键点

3. async函数中的await操作符到底做了什么

   * 按照规范，我们可以做个转化

     ```javascript
     async function async1(){
         console.log('async1 start');
         await async2();
         console.log('async1 end');
     }
     ```

     可以转化为：

     ```javascript
     function async1(){
         console.log('async1 start');
         return RESOLVE(async2())
         	.then(() => { console.log('async1 end') });
     }
     ```

   * 问题关键就出在这个RESOLVE上了，要引用以下知乎上[贺师俊大佬的回答](https://www.zhihu.com/question/268007969/answer/339811998)：

     * RESOLVE(p)接近于Promise.resolve(p)，不过有微妙而重要的差别：p如果本身已经是Promise实例，Promise.resolve会直接返回p而不是产生一个新的promise；
     * 如果RESOLVE(p)严格按照标准，应该产生一个新的promise，尽管该promise确定会resolve 为p，但这个过程本身是异步的，也就是现在进入job队列的是新promise的resolve过程，所以该promise的then不会被立即调用，而要等到当前独立额job队列执行到前述resolve过程才会被调用，然后其回调（也就是继续await之后的语句）才加入job队列，所以时序上就晚了

   * 所以上述的async1函数我们可以进一步转换一下：

     ```javascript
     function aysnc1(){
         console.log('async1 start');
         return new Promise(resolve => resolve(async2()))
             .then(() => {
             	console.log('async1 end');
         });
     }
     ```

   说道最后，最终示例代码近似等价于以下的代码：

   ```javascript
   function async1(){
       console.log('async1 start')
       return new Promise(resolve => resolve(async2()))
           .then(() => {
               console.log('async1 end')
           });
   }
   function async2(){
     console.log('async2');
     return Promise.resolve();
   }
   console.log('script start')
   setTimeout(function(){
     console.log('setTimeout') 
   },0)  
   async1();
   new Promise(function(resolve){
     console.log('promise1')
     resolve();
   }).then(function(){
     console.log('promise2')
   }).then(function() {
     console.log('promise3')
   }).then(function() {
     console.log('promise4')
   }).then(function() {
     console.log('promise5')
   }).then(function() {
     console.log('promise6')
   }).then(function() {
     console.log('promise7')
   }).then(function() {
     console.log('promise8')
   })
   console.log('script end')
   ```



## 理解2

### 基础知识

在你看答案之前，我希望你至少了解

* promise的executor（执行器）里的代码是同步的
* promise的回调是microTask（微任务）而setTimeout的回调是task（任务/宏任务）
* microTask早于task被执行

### 精简题目

我把这道题目精简下，先把同步的代码和setTimeout的代码删掉，再来解释（期间await的部分规范有变动）。

再精简下，问题就是这样：

```javascript
async function async1(){
    await async2();
    console.log('async1 end');
}
async function aysnc2(){
    
}
async1();
new Promise(function(resolve){
  resolve();
}).then(function(){
  console.log('promise2')
}).then(function() {
  console.log('promise3')
}).then(function() {
  console.log('promise4')
})
```

为什么在chrome canary73返回

```
async1 end
promise2
promise3
promise4
```

而在chrome70上返回

```
promise2
promise3
async1 end
promise4
```

### 正文

我对promise稍微熟悉些，其实也不熟，但是把await抓换成promise会相对好理解些，不知道有没有同感？

这道题其实问的是

> await async2()怎么理解？

因为async函数总是返回一个promise，所以其实就是在问

> await promise怎么理解？

那么我们看下规范[Await](https://tc39.github.io/ecma262/#await)。

![Await规范](https://segmentfault.com/img/bVblHuz?w=797&h=558)

根据提示

```javascript
async function async1(){
  await async2()
  console.log('async1 end')
}
```

等价于

```javascript
async function async1() {
  return new Promise(resolve => {
    resolve(async2())
  }).then(() => {
    console.log('async1 end')
  })
}
```

但是到这里，仍然不太好理解。

> 因为 `resolve(async2())` 并不等于 `Promise.resolve(async2())`

之所以这样，因为async2()返回一个promise，是一个thenable对象，resolve(thenable)并不等于Promise.resolve(thenable)，而resolve(non-thenable)等价于Promise.resolve(non-thenable)，具体对照规范的解释请戳

> [What's the difference between resolve(promise) and resolve('non-thenable-object')?](https://stackoverflow.com/questions/53894038/whats-the-difference-between-resolvepromise-and-resolvenon-thenable-object)

结论就是：resolve(thenable)和Promise.resolve(thenable)的转换关系是这样的

```javascript
new Promise(resolve => {
    resolve(thenable);
})
```

会被转换成

```javascript
new Promise(resolve => {
  Promise.resolve().then(() => {
    thenable.then(resolve)
  })
})
```

那么对于resolve(async2())，我们可以根据规范转换成：

```javascript
Promise.resolve().then(() => {
    async2().then(resolve)
})
```

所以async1就变成了这样：

```javascript
async function async1() {
  return new Promise(resolve => {
    Promise.resolve().then(() => {
      async2().then(resolve)
    })
  }).then(() => {
    console.log('async1 end')
  })
}
```

同样，因为resolve()就等价于Promise.resolve()，所以

```javascript
new Promise(function(resolve){
  resolve();
})
```

等价于

```javascript
Promise.resolve()
```

所以，题目

```javascript
async function async1(){
  await async2()
  console.log('async1 end')
}
async function async2(){} 
async1();
new Promise(function(resolve){
  resolve();
}).then(function(){
  console.log('promise2')
}).then(function() {
  console.log('promise3')
}).then(function() {
  console.log('promise4')
})
```

就等价于

```javascript
async function async1 () {
  return new Promise(resolve => {
    Promise.resolve().then(() => {
      async2().then(resolve)
    })
  }).then(() => {
    console.log('async1 end')
  })
}
async function async2 () {}
async1()
Promise.resolve()
  .then(function () {
    console.log('promise2')
  })
  .then(function () {
    console.log('promise3')
  })
  .then(function () {
    console.log('promise4')
  })
```

这是根据当前规范解释的结果，chrome70和chrome canary73上得到的都是一样的。

```
promise2
promise3
async1 end
promise4
```

### Await规范的更新

那么为什么，chrome73现在得到的结果不一样了呢？修改的决议在[这里](https://github.com/tc39/ecma262/pull/1250)，目前是这个状态。

![await修改](https://segmentfault.com/img/bVblHRK?w=763&h=204)

就像你看到的一样，为什么要把async1

```javascript
async function async1(){
  await async2()
  console.log('async1 end')
}
```

转换成

```javascript
async function async1() {
  return new Promise(resolve => {
    Promise.resolve().then(() => {
      async2().then(resolve)
    })
  }).then(() => {
    console.log('async1 end')
  })
}
```

而不是直接

```javascript
async function async1 () {
  async2().then(() => {
    console.log('async1 end')
  })
}
```

这样是不是更简单，容易理解，且提高性能了呢？如果要这样的话，也就是说：

```javascript
async function async1(){
  await async2()
  console.log('async1 end')
}
```

async1不采用new Promise来包装，也就是不走下面这条路：

```javascript
async function async1() {
  return new Promise(resolve => {
    resolve(async2())
  }).then(() => {
    console.log('async1 end')
  })
}
```

而是直接采用Promise.resolve()来包装，也就是

```javascript
async function async1() {
  Promise.resolve(async2()).then(() => {
    console.log('async1 end')
  })
}
```

又因为async2()返回一个promise，根据规范[Promise.resolve](https://tc39.github.io/ecma262/#sec-promise.resolve)：

![Promise.resolve](https://segmentfault.com/img/bVblH1C?w=821&h=122)

所以Promise.resolve(promise)返回promise，即Promise.resolve(async2())等价于async2()，所以最终得到了代码：

```javascript
async function async1 () {
  async2().then(() => {
    console.log('async1 end')
  })
}
```

这就是贺老师在知乎里所说的

> 根据 TC39 最近决议，await将直接使用Promise.resolve()相同语义。

tc39的spec的更改体现在

![await语义更改](https://segmentfault.com/img/bVblHTj?w=667&h=237)

Chrome canary 73采用了这种实现，所以题目

```javascript
async function async1(){
  await async2()
  console.log('async1 end')
}
async function async2(){} 
async1();
new Promise(function(resolve){
  resolve();
}).then(function(){
  console.log('promise2')
}).then(function() {
  console.log('promise3')
}).then(function() {
  console.log('promise4')
})
```

在chrome canary 73及未来可能被解析为：

```javascript
async function async1 () {
  async2().then(() => {
    console.log('async1 end')
  })
}
async function async2 () {}
async1()
new Promise(function (resolve) {
  resolve()
})
  .then(function () {
    console.log('promise2')
  })
  .then(function () {
    console.log('promise3')
  })
  .then(function () {
    console.log('promise4')
  })

//async1 end
//promise2
//promise3
//promise4
```

在chrome 70被解析为

```javascript
async function async1 () {
  return new Promise(resolve => {
    Promise.resolve().then(() => {
      async2().then(resolve)
    })
  }).then(() => {
    console.log('async1 end')
  })
}
async function async2 () {}
async1()
Promise.resolve()
  .then(function () {
    console.log('promise2')
  })
  .then(function () {
    console.log('promise3')
  })
  .then(function () {
    console.log('promise4')
  })

//promise2
//promise3
//async1 end
//promise4
```

##理解3

我也正在思考这个问题，目前从规范中的`PromiseResolveThenableJob`仅可以看出：

* 如果在`Promise`中`resolve`一个`thenable`对象，需要**先将thenable转化为Promsie，然后立即调用thenable的then方法**，并且这个过程需要作为一个JOB加入微任务队列，以保证对then方法的解析发生在其他上下文代码的解析之后（不知我理解的对不对……"the evaluation of the then method occurs after evaluation of any surrounding code has completed"）：

  示例：

  ```javascript
  let thenable = {
    then: function(resolve, reject) {
      console.log('in thenable');
      resolve(42);
    }
  };
  
  new Promise((r) => {
    console.log('in p0');
    r(thenable);
  })
  .then(() => { console.log('0-0') })
  .then(() => { console.log('0-1') })
  .then(() => { console.log('0-2') })
  .then(() => { console.log('0-3') });
  
  new Promise((r) => {
    console.log('in p1');
    r();
  })
  .then(() => { console.log('1-0') })
  .then(() => { console.log('1-1') })
  .then(() => { console.log('1-2') })
  .then(() => { console.log('1-3') });
  ```

  输出：

  ```
  in p0
  in p1
  in thenable
  1-0
  0-0
  1-1
  0-1
  1-2
  0-2
  1-3
  0-3
  ```

  可以看出`in thenable`后于`in p1`而先于`1-0`输出，所以在执行完同步任务后，微任务队列中只有2个JOB：第一个是“转换`thenable`为Promise的过程”，第二个是`console.log('1-0')`；
  在“转换`thenable`为Promise的过程”执行中直接调用`then`，注册了另一个JOB：`console.log('0-0')`，这导致了第一个`Promise`的后续回调被延后了1个时序。

* 如果在Promise中resolve一个Promise实例呢？

  ```javascript
  new Promise((r) => {
    console.log('in p0');
    r(new Promise((r) => {
      console.log('in internal p');
      r();
    }));
  })
  .then(() => { console.log('0-0') })
  .then(() => { console.log('0-1') })
  .then(() => { console.log('0-2') })
  .then(() => { console.log('0-3') });
  
  new Promise((r) => {
    console.log('in p1');
    r();
  })
  .then(() => { console.log('1-0') })
  .then(() => { console.log('1-1') })
  .then(() => { console.log('1-2') })
  .then(() => { console.log('1-3') });
  ```

  输出：

  ```
  in p0
  in internal p
  in p1
  1-0
  1-1
  0-0
  1-2
  0-1
  1-3
  0-2
  0-3
  ```

  能看出来第一个`Promise`的后续回调被延后了2个时序，也就是说在`console.log('0-0')`之前会加入两个隐藏的JOB.

  这令我无法理解，对于不是很熟悉ECMAScript规范和浏览器内部实现的我们来说，这完全是在跟一个黑盒玩耍……

  所以以后如果有人再说`Promise.resolve()`完全等于`new Promise(r => r())`我一定不会同意，如果以后面试再问同步调用多个async函数说说执行顺序这种问题，我一定怼他（规范都没定下来，你就敢问？说不定你自己都没弄懂……）

  猜测：
  我尝试把这个`Promise`实例当成一个`thenable`对象去思考这个问题：

  1. 两个额外的JOB中的第一个：PromiseResolveThenableJob，负责把传递的值转化为`Promise`，虽然我们已经确定了这个要转化的对象就是一个`Promise`，而且还会`resolved`为`undefined`
  2. 第二个JOB，负责传递这个Promise的状态，也就是`<resolved>: undefined`

  只是结合了[《V8 中更快的异步函数和 promises》](https://zhuanlan.zhihu.com/p/51191817)这篇文章之后的猜测……

##理解4

感谢你一致关注这个问题！看了你在blog中提供的两个stackoverflow上的问题和[tc39规范](https://tc39.github.io/ecma262/#sec-promise-abstract-operations)中*25.6.5.4 Promise.prototype.then*和*25.6.5.4.1 PerformPromiseThen*，我有了一些思路：

1. `resolve(promise)`中`resolve`的值是一个`promise`，同样也是一个`thenable`，所以会执行`EnqueueJob("PromiseJobs", PromiseResolveThenableJob, <<promise, resolution, thenAction>>)`，这是进入队列的第一个Job
2. 在第一个Job`<PromiseResolveThenableJob(promiseToResolve, thenable, then)>`执行中，会创建和`promiseToResolve`关联的`resolve function`和`reject function`，然后同步调用`thenable.then(resolve, reject)`
3. 一般的，对于普通的thenable对象（非Promise），2中最后的操作会通过`resolve`或`reject`函数更改Promise的状态，并触发后续通过`then`注册的回调，但如果传入的`thenable`是一个`Promise`，情况就不同了（下面解释...）
4. 传入的`thenable`是一个`Promise`，则2中最后调用的就是`Promise.prototype.then(onfulfilled, onrejected)`，2中的`resolve`作为`onfulfilled`，`reject`作为`onrejected`
5. 后面就是`Promise.prototype.then`的"常规操作"了……`Promise.prototype.then`会在内部创建一个`promiseCapability`，它包含了一个新的`Promise`和相关联的`resolve function`和`reject function`
6. 根据`promiseCapability`和`onfulfilled/onrejected`创建两个分别用于`fulfill`和`reject`的PromiseReaction，也就是 PromiseJobs 里最终要执行的操作
7. 如果当前的`promise(this)`（这里实际上是1中传入的`thenable`）是`pending`状态，则把这两个在6中生成的`reaction`分别插入到`promise`的`[[PromiseFulfillReactions]]`和`[[PromiseRejectReactions]]`队列中。如果`promise`已经`fulfilled`或`rejected`，就从`promise`的`[[PromiseResult]]`取出`result`，作为`fulfilled/reject`的结果，然后`EnqueueJob("PromiseJobs", PromiseReactionJob, <<reaciton, result>>)`插入到Job队列，最后返回 `prjomiseCapability`里存储`promise`.

- 所以最后，回到我们一开始的问题，`new Promise(r => r(new Promise(r => r(50))))`，在第一个`Job(PromiseResolveThenableJob)`执行中，执行了`Promise.prototype.then`，而这时的`Promise`已经是`<resolved>: 50`了，所以在`then`执行中再一次创建了一个Job: `EnqueueJob("PromiseJobs", PromiseReactionJob, <<reaciton, result>>)`，`result = 50`，最终结果就是额外创建了两个Job，被推迟了2个时序，我想这应该就是问题的答案了。

