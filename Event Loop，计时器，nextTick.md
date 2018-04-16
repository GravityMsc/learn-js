# Event Loop、计时器、nextTick

## 什么是事件循环（Event Loop）

JavaScript是单线程的，有了event loop的加持，Node.js才可以非阻塞地执行I/O操作，把这些操作尽量转移给操作系统来执行。

我们知道大部分现代操作系统都是多线程的，这些操作系统可以在后台执行多个操作。当某个操作结束了，操作系统就会通知Node.js，然后Node.js就（可能）会把对应的回调函数添加到poll（轮询）队列，最终这些回调函数会被执行。下文中我们会阐述其中的细节。

### Event Loop详解

当Node.js启动时，会做这几件事：

1. 初始化event loop；
2. 开始执行脚本（或者进入REPL（read-eval-print-loop，交互式解释器，每一行代码输入，即运行，并给出结果，通常只有解释型语言可以做到这一点，编译型语言无法做到），本文不涉及REPL）。这些脚本有可能会调用一些异步API、设定计时器或者调用process.nextTick()；
3. 开始处理event loop。

如何处理event loop呢？下图给出了一个简单的概览：

```javascript
   ┌───────────────────────┐
┌─>│        timers         │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     I/O callbacks     │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     idle, prepare     │
│  └──────────┬────────────┘      ┌───────────────┐
│  ┌──────────┴────────────┐      │   incoming:   │
│  │         poll          │<─────┤  connections, │
│  └──────────┬────────────┘      │   data, etc.  │
│  ┌──────────┴────────────┐      └───────────────┘
│  │        check          │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
└──┤    close callbacks    │
   └───────────────────────┘
```

其中每个方框都是event loop中的一个阶段。

每个阶段都有一个【先入先出队列】，这个对象存有要执行的回调函数（实际上存的是函数地址）。不过每个阶段都有其特有的使命。一般来说，当event loop达到某个阶段时，会在这个阶段进行一些特殊的操作，然后执行这个阶段的队列里的所有回调。什么时候停止执行这些回调呢？下列两种情况之一会停止：

1. 队列的操作全被执行完了；
2. 执行的回调数目达到指定的最大值，然后，event loop进入下一阶段，然后再下一阶段。

一方面，上面这些操作都有可能添加计时器；另一方面，操作系统会向poll队列中添加新的事件，当poll队列中的事件被处理时可能会有新的poll事件进入poll队列。结果，耗时较长的回调函数可以让event loop在poll阶段停留很久，久到错过了计时器的触发时机。你可以在下文的times章节和poll章节详细了解这其中的细节。

注意，Windows的实现和Uinx/Linux的实现稍有不同，不过对本文内容影响不大。本文囊括了event loop最重要的部分，不同平台可能有七个或八个阶段，但是上面的几个阶段是我们真正关心的阶段，而且是Node.js真正用到的阶段。。

## 各阶段概览

> * times阶段：这个阶段执行setTimeout和setInterval的回调函数。
> * I/O callbacks阶段：不在timers阶段、close callbacks阶段和check阶段这三个阶段执行的回调，都由此阶段负责，这几乎包含了所有回调函数。
> * idle，prepare阶段：event loop内部使用的阶段。
> * poll阶段：获取新的I/O事件。在某些场景下Node.js会阻塞在这个阶段。
> * check阶段：执行setImmediate()的回调函数。
> * close callbacks阶段：执行关闭事件的回调函数，如socket.on('close', fn)里面的fn。

一个Node.js程序结束时，Node.js会检查event loop是否在等待异步I/O操作结束，是否在等待计时器触发，如果没有，就会关掉event loop。

## 各阶段详解

### timers阶段

计时器实际上是在指定多久以后可以执行某个回调函数，而不是指定某个函数的确切执行时间。当指定的时间到达后，计时器的回调函数会尽早被执行。如果操作系统很忙，或者Node.js正在执行一个耗时的函数，那么计时器的回调函数就会被推迟执行。

注意，从原理上来说，poll阶段能控制计时器的回调函数什么时候被执行。

举例来说，你设置了一个计时器在100毫秒后执行，然后你的脚本用了95毫秒来异步读取了一个文件：

```javascript
const fs = require('fs');

function someAsyncOperation(callback){
    // 假设读取这个文件一共花费了95毫秒
    fs.readFile('/path/to/file', callback);
}

const timeoutScheduled = Date.now();

setTimeout(() => {
    const delay = Date.now() - timeoutScheduled;
    
    console.log(`${delay}毫秒后执行力setTimeout的回调`);
}, 100);

// 执行一个耗时95毫秒的异步操作
someAayncOperation(() => {
    const stateCallbakc = Date.now();
    
    // 执行一个耗时10毫秒的同步操作
    while(Date.now() - startCallback < 10){
        // 什么也不做
    }
});
```

当event loop进入poll阶段，发现poll队列为空（因为文件还没有读完），event loop检查了一下最近的计时器，大概还有100毫秒的时间，于是event loop决定这段时间就停在poll阶段。在poll阶段停留了95毫秒之后，fs.readFile操作完成，一个耗时10毫秒的回调函数被系统放入poll队列，于是event loop执行了这个回调函数。执行完毕后，poll队列为空，于是event loop去看了一样最近的计时器（event loop发现，已经超时 95 + 10 - 100 = 5毫秒了），于是经由check阶段、close callbacks阶段绕回到timers阶段，执行timers队列里面的那个回调函数。这个例子中，100毫秒的计时器实际上是在105毫秒后才执行的。

注意：为了防止poll阶段占用了event loop的所有时间，libuv（Node.js用来实现event loop和所有异步行为的C语言写成的库）对poll阶段的最长停留时间做出了限制，具体时间因操作系统而异。

### I/O callbacks阶段

这个阶段会执行一些系统操作的回调函数，比如TCP报错，如果一个TCP socket开始连接时出现了ECONNREFUSED错误，一些*nix系统就会（向Node.js）通知这个错误。这个通知就会被放入I/O callbacks队列。

### poll阶段（轮询阶段）

poll阶段有两个功能：

1. 如果发现计时器的时间到了，就绕回到timers阶段执行计时器的回调。
2. 然后再执行poll队列里的回调。

当event loop进入poll阶段，如果发现没有计时器，就会：

1. 如果poll队列不是空的，event loop就会依次执行队列里的回调函数，直到队列被清空或者到达poll阶段的时间上限。
2. 如果poll队列是空的，就会：
    	1. 如果有setImmediate()任务，event loop就结束poll阶段去往check阶段。
    	2. 如果没有setImmediate()任务，event loop就会等待新的回调函数进入poll队列，并立即执行它。

一旦poll队列为空，event loop就会检查计时器有没有到期，如果有计时器到期了，event loop就会回到timers阶段执行计时器的回调。

### close callbacks阶段

如果一个socket或者handle被突然关闭了（比如socket.destroy()），那么就会有一个close事件进入这个阶段。否则（没有close事件的话）就会进入process.nextTick()。



## setImmediate()和setTimeout()

setImmediate和setTimeout很相似，但是其回调函数的调用时机却不一样。

setImmediate()的作用是在当前poll阶段结束后调用一个函数。setTimeout()的作用是在一段时间后调用一个函数。这两者的回调的执行顺序取决于setTimeout和setImmediate被调用时的环境。

如果setTimeout和setImmediate都是在主模块（main module）中被调用的，那么回调的执行顺序取决于当前进程的性能，这个性能受其他应用程序进程的影响。

举例来说，如果在主模块中运行下面的脚本，那么两个回调的执行顺序是无法判断的：

```javascript
// timeout_vs_immediate.js
setTimeout(() => {
    console.log('timeout');
}, 0);

setImmediate(() => {
    console.log('immediate');
});
```

运行结果如下：

```javascript
$ node timeout_vs_immediate.js
timeout
immediate

$ node timeout_vs_immediate.js
immediate
timeout
```

但是，如果把上面代码放到I/O操作的回调里，setImmediate的回调就总是优于setTimeout的回调：

```javascript
// timeout_vs_immediate.js
const fs = require('fs');

fs.readFile(__filename, () => {
    setTimeout(() => {
        console.log('timeout');
    }, 0);
    setImmediate(() => {
        console.log('immediate');
    }); 
});
```

运行结果如下：

```javascript
$ node timeout_vs_immediate.js
immediate
timeout

$ node timeout_vs_immediate.js
immediate
timeout
```

setImmediate的主要优势就是，如果在I/O操作的回调里，setImmediate的回调总是比setTimeout的回调先执行。

## process.nextTick()

你可能发现process.nextTick()这个重要的异步API没有出现在任何一个阶段里，那是因为从技术上来讲process.nextTick()并不是event loop的一部分。实际上，不管event loop当前处于哪个阶段，nextTick队列都是在当前阶段后就被执行了。

回过头来看我们的阶段图，你在任何一个阶段调用process.nextTick(回调)，回调都会在当前阶段继续运行前被调用。这种行为有的时候会造成不好的后果，因为你可以递归地调用process.nextTick()，这样event loop就会一直停在当前阶段不走......无法进入poll阶段。

为什么Node.js要这样设计process.nextTick()呢？

因为有些异步API需要保证一致性，即使可以同步完成，也要保证异步操作的顺序，看下面代码：

```javascript
function apiCall(arg, callback){
    if(typeof arg !== 'string'){
        return process.nextTick(callback, new TypeError('argument should be string'));
    }
}
```

这段代码检查了参数的类型，如果类型不是string，就会将error传递给callback。

这段代码保证apiCall调用之后的同步代码能在callback之前运行。由于用到了process.nextTick()，所以callback会在event loop进入下一阶段前执行。为了做到这一点，JS的调用栈可以先unwind再执行nextTick的回调，这样无论你递归调用多少次process.nextTick()都不会造成调用栈溢出（V8里对应RangeError：Maximum call stack size exceeded）。

如果不这样设计，会造成一些潜在的问题，比如下面的代码：

```javascript
let bar;

// 这是一个异步API，但是却同步地调用了callback
function someAsyncApiCall(callback){callback();}

// someAsyncApiCall在执行过程中就调用了回调
someAsyncApiCall(() => {
    // 此时bar还没有被赋值为1
    console.log('bar', bar);	// undefined
});

bar = 1;
```

开发者虽然把someAsyncApiCall命名的像一个异步函数，但是实际上这个函数是同步执行的。当someAsyncApiCall被调用时，回调也在同一个event loop阶段被调用了。结果回调中就无法得到bar的值。因为赋值语句还没被执行。

如果把回调放在process.nextTick()中执行，后面的赋值语句就可以先执行了。而且process.nextTick()的回调会在event loop进入下一个阶段前调用。

```javascript
let bar;

function someAsyncApiCall(callback) {
  process.nextTick(callback);
}

someAsyncApiCall(() => {
  console.log('bar', bar); // 1
});

bar = 1;
```

一个更符合现实的例子是这样的：

```javascript
const server = net.createServer(() => {}).listen(8080);

server.on('listening', () => {});
```

.listen(8080)这句代码是同步执行的。问题在于listening回调无法被触发，因为listening的监听代码在.listen(8080)的后面。

为了解决这个问题，.listen(8080)函数可以使用process.nextTick()来执行listening事件的回调。

## process.nextTick()和setImmediate()

这两个函数功能很像，而且名字也很令人迷惑。

process.nextTick()的回调会在当前event loop阶段【立即】执行。setImmediate()的回调会在后续的event loop周期（tick）执行。

二者的名字应该互换才对。process.nextTick()比setImmediate()更immediate（立即）一些。

这是一个历史遗留问题，而且为了保证向后兼容性，也不太可能得到改善。所以就算这两个名字听起来让人很疑惑，也不会在未来有任何变化。

我们推荐开发者在任何情况下都使用setImmediate()，因为它的兼容性最好，而且它更容易理解。

## 什么时候用process.nextTick()

使用的主要理由有两个：

1. 让开发者处理错误、清除无用的资源，或者在event loop当前阶段结束前尝试重新请求资源。
2. 有时候有必要让一个回调在调用栈unwind之后，event loop进入下阶段之前执行。

为了让代码更合理，我们可能会写这样的代码：

```javascript
const server = net.createServer();
server.on('connection', (conn) => {});

server.listen(8080);
server.on('listening', () => {});
```

假设listen()在event loop一启动的时候就执行了，而listening事件的回调被放在了setImmediate()里，listen动作是立即发生的，如果想要event loop执行listening回调，就必须先经过poll阶段，当时poll阶段有可能会停留，以等待连接，这样一来就有可能出现connect事件的回调比listening事件的回调先执行。（这显然不合理，所以我们需要用process.nextTick）

再举一个例子，一个类继承了EventEmitter，而且想在实例化的时候触发一个事件：

```javascript
const EventEmitter = require('events');
const util = require('util');

function MyEmitter(){
    EventEmitter.call(this);
    this.emit('event');
}
util.inherits(MyEmitter, EventEmitter);

const myEmitter = new MyEmitter();
myEmitter.on('event', () => {
    console.log('an event occurred!');
});
```

你不能直接在构造函数里执行this.emit('event')，因为这样的话后面的回调就永远无法执行。把this.emit('event')放在process.nextTick()里，后面的回调就可以执行，这才是我们预期的行为：

```javascript
const EventEmitter = require('events');
const util = require('util');

function MyEmitter() {
  EventEmitter.call(this);

  // use nextTick to emit the event once a handler is assigned
  process.nextTick(() => {
    this.emit('event');
  });
}
util.inherits(MyEmitter, EventEmitter);

const myEmitter = new MyEmitter();
myEmitter.on('event', () => {
  console.log('an event occurred!');
});
```





