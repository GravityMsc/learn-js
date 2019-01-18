JavaScript的异步过程一直被认为是不够快的，更糟糕的是，在NodeJS等实时性要求高的场景下调试堪比噩梦。不过，这一切正在改变，这篇文章会详细解释我们是如何优化V8引擎里的async函数和promises的，以及伴随着的开发体验的优化。

## 异步编程的新方案

### 从callbacks到promises，再到async函数

在promises正式成为JavaScript标准的一部分之前，回调被大量用在异步编程中，下面是个例子：

```javascript
function handler(done) {
  validateParams((error) => {
    if (error) return done(error);
    dbQuery((error, dbResults) => {
      if (error) return done(error);
      serviceCall(dbResults, (error, serviceResults) => {
        console.log(result);
        done(error, serviceResults);
      });
    });
  });
}
```

类似以上深度嵌套的回调通常被称为【回调黑洞】，因为它让代码可读性变差且不易维护。

幸运的是，现在promises成为了JavaScript语言的一部分，以下实现了同上面一样的功能：

```javascript
function handler() {
  return validateParams()
    .then(dbQuery)
    .then(serviceCall)
    .then(result => {
      console.log(result);
      return result;
    });
}
```

最近，JavaScript支持了[async函数](https://developers.google.com/web/fundamentals/primers/async-functions)，上面的异步代码可以写成像下面这样的同步的代码：

```javascript
async function handler() {
  await validateParams();
  const dbResults = await dbQuery();
  const results = await serviceCall(dbResults);
  console.log(results);
  return results;
}
```

借助async函数，代码变得简洁，代码的逻辑和数据流变得更可控，当然其实底层实现还是异步。（注意，JavaScript还是单线程执行，async函数并不会开启新的线程。）

### 从事件监听回调到async迭代器

NodeJS里[ReadableStreams](https://nodejs.org/api/stream.html#stream_readable_streams)作为另一种形式的异步也特别常见，下面是个例子：

```javascript
const http = require('http');

http.createServer((req, res) => {
  let body = '';
  req.setEncoding('utf8');
  req.on('data', (chunk) => {
    body += chunk;
  });
  req.on('end', () => {
    res.write(body);
    res.end();
  });
}).listen(1337);
```

这段代码有一点难理解：只能通过回调去拿chunks里的数据流，而且数据流的结束也必须在回调里处理。如果你没能理解到函数是立即结束但实际处理必须在回调里进行，可能就会引入bug。

同样很幸运，ES2018特性里引入了一个很酷的[async迭代器](http://2ality.com/2016/10/asynchronous-iteration.html)可以简化上面的代码：

```javascript
const http = require('http');

http.createServer(async (req, res) => {
  try {
    let body = '';
    req.setEncoding('utf8');
    for await (const chunk of req) {
      body += chunk;
    }
    res.write(body);
    res.end();
  } catch {
    res.statusCode = 500;
    res.end();
  }
}).listen(1337);
```

你可以把所有数据处理逻辑都放到一个async函数里使用for await...of去迭代chunks，而不是分别在'data'和'end'回调里处理，而且我们还加了try-catch块来避免unhandledRejection问题。

以上这些特性你今天就可以在生产环境使用！async 函数**从 Node.js 8 (V8 v6.2 / Chrome 62) 开始就已全面支持**，async 迭代器**从 Node.js 10 (V8 v6.8 / Chrome 68) 开始支持**。

### async性能优化

从 V8 v5.5 (Chrome 55 & Node.js 7) 到 V8 v6.8 (Chrome 68 & Node.js 10)，我们致力于异步代码的性能优化，目前的效果还不错，你可以放心地使用这些新特性。

![async性能](https://user-gold-cdn.xitu.io/2018/11/15/1671636b00a1b83b?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

上面的是 [doxbee 基准测试](https://link.juejin.im/?target=https%3A%2F%2Fgithub.com%2Fv8%2Fpromise-performance-tests%2Fblob%2Fmaster%2Flib%2Fdoxbee-async.js)，用于反应重度使用 promise 的性能，图中纵坐标表示执行时间，所以越小越好。

另一方面，[parallel 基准测试](https://link.juejin.im/?target=https%3A%2F%2Fgithub.com%2Fv8%2Fpromise-performance-tests%2Fblob%2Fmaster%2Flib%2Fparallel-async.js) 反应的是重度使用 [Promise.all()](https://link.juejin.im/?target=https%3A%2F%2Fdeveloper.mozilla.org%2Fen-US%2Fdocs%2FWeb%2FJavaScript%2FReference%2FGlobal_Objects%2FPromise%2Fall) 的性能情况，结果如下：

![基准测试](https://user-gold-cdn.xitu.io/2018/11/15/1671636b004f74e6?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

Promise.all的性能提高了八倍！

然后，上面的测试仅仅是小的 DEMO 级别的测试，V8 团队更关心的是 [实际用户代码的优化效果](https://link.juejin.im/?target=https%3A%2F%2Fv8.dev%2Fblog%2Freal-world-performance)。

![实际性能](https://user-gold-cdn.xitu.io/2018/11/15/1671636b0099db70?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

上面是基于市场上流行的 HTTP 框架做的测试，这些框架大量使用了 promises 和 `async` 函数，这个表展示的是每秒请求数，所以跟之前的表不一样，这个是数值越大越好。从表可以看出，从 Node.js 7 (V8 v5.5) 到 Node.js 10 (V8 v6.8) 性能提升了不少。

性能提升取决于以下三个因素：

- [TurboFan](https://link.juejin.im?target=https%3A%2F%2Fv8.dev%2Fdocs%2Fturbofan)，新的优化编译器 🎉
- [Orinoco](https://link.juejin.im?target=https%3A%2F%2Fv8.dev%2Fblog%2Forinoco)，新的垃圾回收器 🚛
- 一个 Node.js 8 的 bug 导致 await 跳过了一些微 tick（microticks） 🐛

当我们在 [Node.js 8](https://link.juejin.im?target=https%3A%2F%2Fmedium.com%2Fthe-node-js-collection%2Fnode-js-8-3-0-is-now-available-shipping-with-the-ignition-turbofan-execution-pipeline-aa5875ad3367) 里 [启用 TurboFan](https://link.juejin.im?target=https%3A%2F%2Fv8.dev%2Fblog%2Flaunching-ignition-and-turbofan) 的后，性能得到了巨大的提升。

同时我们引入了一个新的垃圾回收器，叫作 Orinoco，它把垃圾回收从主线程中移走，因此对请求响应速度提升有很大帮助。

最后，Node.js 8引入了一个bug在某些时候会让await跳过一些微tick，这反而让性能变好了。这个bug是因为无意中违反了规范导致的，但是却给了我们优化的一些思路，这里我稍微解释下：

```javascript
const p = Promise.resolve();

(async () => {
  await p; console.log('after:await');
})();

p.then(() => console.log('tick:a'))
 .then(() => console.log('tick:b'));
```

上面代码一开始创建了一个已经完成状态的promise p，然后await出其结果，又同时链了两个then，那最终的console.log打印的结果会是什么呢？

因为p是已经完成的，你可能会认为其会先打印'after:await'，然后是剩下两个tick，事实上Node.js 8 里的结果是：

![Node.js 8的结果](https://user-gold-cdn.xitu.io/2018/11/15/1671636b007dab55?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

虽然以上结果符合预期，但是却不符合规范。Node.js 10纠正了这个行为，会先执行then链里的，然后才是async函数。

![Node.js 10](https://user-gold-cdn.xitu.io/2018/11/15/1671636b01b2dba5?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

这个【正确的行为】看起来并不正常，甚至会让很多JavaScript开发者感到吃惊，还是有必要再详细解释下。在解释之前，我们先从一些基础开始。

### 任务（tasks）vs. 微任务（microtasks）

从某个层面上来说，JavaScript里存在任务和微任务。任务处理I/O和计时器等事件，一次只处理一个。微任务时为了async/await和Promise的延迟执行设计的，每次任务最后执行。在返回事件循环（event loop）前，微任务的队列会被清空。

