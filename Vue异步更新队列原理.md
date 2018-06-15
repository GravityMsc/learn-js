声明：本文章中所有源码取自Version：2.5.13的dev分支上的Vue。

我们目前的技术栈主要采用Vue，而工作中我们碰到了一种情况是当传入某些组件内的props被改变时我们需要重置整个组件的生命周期（比如更改iView中datepicker的type，好消息是目前该组件已经可以不用再使用这么愚蠢的方法来切换时间显示器的类型）。为了达成整个目的，于是我们有了如下代码

```vue
<template>
	<button @click="handleClick">
        btn
    </button>
	<someComponent v-if="show" />
</template>
<script>
    {
        data(){
            return {show: true}
        },
        methods: {
            handleClick(){
                this.show = false;
                this.show = true;
            }
        }
    }
</script>
```

别笑，我们当然知道这段代码有多愚蠢，不用尝试也确定这是错的，但是凭借react的经验我大概知道将this.show = true换成setTimeout(() => {this.show = true}, 0)，就应该可以得到想要的结果，果然，组件重置了其声明周期，但是事情还是有点不对头。我们经过几次点击发现组件总是会闪一下。逻辑上这很好理解，组件先销毁后重建有这种情况很正常的，但是抱歉，我们找到了另一种方式，将setTimeout(() => {this.show = true}, 0)换成this.$nextTick(() => {this.show = true})，神奇的事情来了，组件依然重置了其生命周期，但是组件没有丝毫的闪动。

为了让亲爱的您感到我这段虚无缥缈的描述，我为您贴心准备了此[demo](https://jsfiddle.net/25fdgLgr/14/)，您可以将handle1依次换为handle2与handle3来体验组件在闪动与不闪动之间徘徊的快感。

如果您体验完快感之后仍然选择继续阅读，那么我要跟你说的是，接下来的内容会是比较长的，因为想要完全弄明白这件事，我们必须深入Vue的内部与JavaScript的EventLoop两个方面。

导致此问题的主要原因在于Vue默认采用的是异步更新队列的方式，我们可以从官网上找到以下描述：

> 可能你还没有注意到，Vue异步执行DOM更新。只要观察到数据变化，Vue将开启一个队列，并缓冲在同一事件循环中发生的所有数据改变。如果同一个watcher被多次触发，只会被推入到队列中一次。这种在缓冲时去除重复数据对于避免不必要的计算和DOM操作上非常重要。然后，在下一个的事件循环“tick”中，Vue刷新队列并执行实际（已去重的）工作。Vue在内部尝试对异步队列使用原生的Promise.then和MessageChannel，如果执行环境不支持，会采用setTimeout(fn, 0)代替。

这段话确实精简的描述了整个流程，但是并不能解决我们的困惑，接下来是时候展示真正的技术了。需要说明的是以下核心流程如果您没有阅读过一些介绍源码的blog或者是阅读过源码，那么您可能一脸懵逼。但是没关系在这里我们最终关系的基本上只是第4步，您只需要大概将其记住，然后将这个流程对应我们后面解析的源码就可以了。

Vue的核心流程大体可以分成以下几步：

1. 遍历属性为其增加get，set方法，在get方法中会收集依赖（dev.subs.push(watcher)），而set方法则会调用dev的notify方法，此方法的作用是通知subs中的所有的watcher并调用watcher的update方法，我们可以将此理解为设计模式中的发布与订阅。
2. 默认情况下update方法被调用后会触发queueWatcher函数，此函数的主要功能就是将watcher实例本身加入一个队列中（queue.push(watcher)），然后调用nextTick(flushSchedulerQueue)。
3. flushSchedulerQueue是一个函数，目的是调用queue中所有watcher的watcher.run方法，而run方法被调用后接下来的操作就是通过新的虚拟dom与老的虚拟dom做diff算法后生成新的真实dom。
4. 只是此时我们flushSchedulerQueue并没有执行，第二步的最终做的只是将flushSchedulerQueue又放进一个callbacks队列中（callbacks.push(flushSchedulerQueue)），然后异步的将callbacks遍历并执行（此为异步更新队列）。
5. 如上所说，flushSchedulerQueue在被执行后调用watcher.run()，于是你看到了一个新的画面。

以上所有流程都在vue/src/core文件夹中。

接下来我们按照上面例子中的最后一种情况来分析Vue源码的执行过程，其中一些细节我会有所省略，请记住开始的话，我们这里最关心的只是第四步。

当点击按钮后，绑定在按钮上的回调函数被触发，this.show = false被执行。触发了属性中的set函数，set函数中，dev的notify方法被调用，导致其subs中每个watcher的update都被执行（在本例中subs数组里只有一个watcher~），一起来看下watcher的构造函数。

```javascript
class Watcher{
    constructor (vm) {
        // 将vue实例绑定在watcher的vm属性上
        this.vm = vm;
    }
    update(){
        // 默认情况下都会进入else的分支，同步则直接调用watcher的run方法
        if(this.lazy){
            this.dirty = true;
        } else if(this.sync){
            this.run();
        } else {
            queueWatcher(this);
        }
    }
}
```

再来看下queueWatcher

```javascript
/**
 * 将watcher实例推入queue（一个数组）中
 * 被has对象标记的watcher不会重复被加入到队列
 */
export function queueWatcher(watcher: Watcher){
    const id = watcher.id;
    // 判断watcher是否被标记过，has为一个对象，把方案类似数组去重时利用object保存数组
    if(has[id] == null){
        // 没被标记过的watcher进入分支后被标记上
        has[id] = true;
        if(!flushing){
            // 推入到队列中
            queue.push(watcher);
        } else {
            // 如果是在flush队列时被加入，则根据其watcher的id将其插入到到正确的位置
            // 如果不幸该watcher已经错过了被调用的时机则会被立即调用
            // 稍后看flushSchedulerQueue这个函数会理解这两段注释的意思
            let i = queue.length - 1;
            while(i > index && queue[i].id > watcher.id){
                i--;
            }
            queue.splice(i + 1, 0, watcher);
        }
        // queue the flush
        if(!waiting){
            waiting = true;
            // 我们关系的重点nextTick函数，其实我们写的this.$nextTick也是调用的此函数
            nextTick(flushSchedulerQueue);
        }
    }
}
```

这个桉树运行后，我们的watcher进入到了queue队列中（本例中queue内部也只被添加这一个watcher），然后调用nextTick(flushSchedulerQueue)，这里我们先来看下flushSchedulerQueue函数的源码：

```javascript
/**
 * flush整个队列，调用watcher
 */
function flushSchedulerQueue(){
    // 将flush置为true，请联系上下文
    flushing = true;
    let watcher, id;
    
    // flush队列前先排序
    // 1.Vue中的组件的创建于更新有点类似于事件捕获，都是从最外层向内层延伸，所以要先
    // 调用父组件的创建与更新
    // 2.userWacther比renderWatcher创建要早
    // 3.如果父组件的watcher代用run时将父组件干掉了，那其子组件的watcher也就没必要调用了
    queue.sort((a, b) => a.id - b.id);
    
    // 此处不缓存queue的length，因为在循环过程中queue依然可能被添加watcher导致length长度的改变
    for(index = 0; index < queue.length; index++){
        // 取出每个watcher
        watcher = queue[index];
        id = watcher.id;
        // 清掉标记
        has[id] = null;
        // 更新dom走起
        watcher.run();
        // dev环境下，检测是否为死循环
        if(process.env.NODE_ENV !== 'production' && has[id] != null){
            circular[id] = (circular[id] || 0) + 1;
            if(circular[id] > MAX_UPDATE_COUNT){
                warn(
                	'You may have an infinite update loop' + (
                     watcher.user
                        ? `in watcher with expression "${watcher.expression}"`
                        : `in a component render function.`
                    ),
                    watcher.vm
                )
                break;
            }
        }
    }
}
```

仍然要记得，此时我们的flushSchedulerQueue还没执行，它只是被当作回调传入了nextTick中，接下来我们就来说说我们本次的重点nextTick，建议您整体的看一下[nextTick](https://github.com/vuejs/vue/blob/dev/src/core/util/next-tick.js)的源码，虽然我也都会解释到。

我们首先从next-tick.js中提取出来withMacroTask这个函数来说明，很抱歉我把这个函数放到了最后，因为我想让亲爱的您知道，最重要的东西总是要压轴登场的。但是从整体流程来说当我们点击btn的时候，其实第一步应该是调用此函数。

```javascript
/**
 * 包装参数fn，让其使用marcotask
 * 这里的fn为我们在事件上绑定的回调函数
 */
export function withMacroTask(fn: Function): Function {
    return fn._withTask || (fn._withTask = function(){
        useMacroTask = true;
        const res = fn.apply(null, arguments);
        useMacroTask = false;
        return res;
    });
}
```

没错，其实您绑定在onclick上的回调函数是在这个函数内以apply的形式触发的，请您先去在此处打一个断点来验证。好的，我现在相信您已经证明了我所言非虚，但是其实那不重要，因为重要的是我们在此处立了一个flag，useMacroTask = true，这才是很关键的东西，谷歌翻一下我们可以知道它的具体含义，**用宏任务**。

OK，这就要从我们文章开头所说的第二部分EventLoop讲起了。

其实这部分内容相信对已经看到这里的您来说早就接触过了，如果还真的不太清楚的话推荐您看下阮一峰老师的[这篇文章](http://www.ruanyifeng.com/blog/2014/10/event-loop.html)，我们只会大概的做一个总结：

1. 我们的同步任务的调用形成了一个栈结构。
2. 除此之外我们还有一个任务队列，当一个异步任务有了结果后会向队列中添加一个任务，每个任务都对应着一个回调函数。
3. 当我们的栈结构为空时，就会读取任务队列，同时调用其对应的回调函数。
4. 重复。

这个总结目前来说对于我们比较欠缺的信息就是队列中的任务其实是分为两种的，宏任务（macrotask）与微任务（microtask）。当主线程上执行的所有同步任务结束后会从任务队列中抽取出所有微任务执行，当微任务也执行完毕后一轮事件循环就结束了，然后浏览器会重新渲染（请谨记这一点，因为正是此原因才会导致文章开头所说的闪动问题）。之后再从队列中取出宏任务继续下一轮的事件循环，值的注意的一点是执行微任务时仍然可以继续产生微任务在本轮事件循环中不停的执行。所以本质上微任务的优先级是高于宏任务的。

如果你想更详细的了解宏任务和微任务那么推荐您阅读[这篇文章](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/?utm_source=html5weekly)，这或许是东半球关于这个问题解释的最好，最易懂，最详细的文章了。

宏任务和微任务产生的方式并不相同，浏览器环境下setImmediate，MessageChannel，setTimeout会产生宏任务，而MutationObserver，Promise则会产生微任务。而这也是Vue中采取的异步方式，Vue会根据useMacrotask的布尔值来判断是要产生宏任务还是产生微任务来异步更新队列，我们会稍后看到这部分，现在我们还是走回我们原来的逻辑吧。

当fn在withMacroTask函数中被调用后就产生了我们以上所讲的所有步骤，现在是时候来真正看下nextTick函数都干了什么：

```javascript
export function nextTick(cb?: Function, ctx?: Object){
    let _resolve;
    // callbacks为一个数组，此处将cb推进数组，本例中此cb为刚才还未执行的flushSchedulerQueue
    callbacks.push(() => {
        if(cb){
            try{
                cb.call(ctx);
            } catch(e){
                handleError(e, ctx, 'nextTick');
            }
        } else if(_resolve){
            _resolve(ctx);
        }
    });
    // 标记位，保证之后如果有this.$nextTick之类的操作不会再次执行以下代码
    if(!pending){
        pending = true;
        // 用微任务还是用宏任务，此例中运行到现在为止Vue的选择是用宏任务
        // 其实我们可以理解成所有用v-on绑定事件所直接产生的数据变化都是采用宏任务的方式
        // 因为我们绑定的回调都经过了withMacroTask的包装，withMacroTask中会使useMacrotask为true
        if(useMacroTask){
            macroTimeFunc();
        } else {
            microTimeFunc();
        }
    }
    // $flow-disable-line
    if(!cb && typeof Promise !== 'undefined'){
        return new Promise(resolve => {
            _resolve = resolve;
        });
    }
}
```

执行完以上代码最后只剩下两个结果，调用macroTimerFunc或者microTimerFunc，本例中到目前为止，会调用macroTimerFunc。这两个函数的目的其实都是要以异步的形式去遍历callbacks中的函数，只不过就像我们上文所说的，他们采取的方式并不一样，一个是宏任务达到异步，一个是微任务达到异步。另外我要适时的提醒你引起以上所有流程的原因只是运行了一行代码this.show = false而this.$nextTick(() => {this.show = true})还没开始执行，不过别绝望，也快轮到它了。好的，回到正题来看看macroTimerFunc与microTimerFunc吧。

```javascript
/**
  * macroTimerFunc
  */
// 如果当前环境支持setImmediate，就用此来产生宏任务达到异步效果
if(typeof setImmediate !== 'undefined' && isNative(setImmediate)){
    macroTimerFunc = () => {
        setImmediate(flushCallbacks);
    }
} else if(typeof MessageChannel !== 'undefined' && (
	// 否则MessageChannel
    isNative(MessageChannel) || 
    // PhantomJS
    MessageChannel.toString() === '[object MessageChannnelConstructor]'
)){
    const channel = new MessageChanner();
    const port = channel.port2;
    channel.port1.onmessage = flushCallbacks;
    macroTimerFunc = () => {
        port.postMessage(1);
    };
} else {
    // 再不行的话就只能setTimeout了
    /* istanbul ignore next */
    macroTimerFunc = () => {
        setTimeout(flushCallbacks, 0);
    }
}
```

```javascript
/**
  * microTimerFunc
  */
// 如果支持Promise则用Promise来产生微任务
if(typeof Promise !== 'undefined' && isNative(Promise)){
    const p = Promise.resolve();
    microTimerFunc = () => {
        p.then(flushCallbacks);
        // 对IOS做兼容性处理，（IOS中存在一些问题，具体可以看尤大大自己的解释）
        if(isIOS) setTimeout(noop);
    }
} else {
    // 降级
    microTimerFunc = macroTimerFunc;
}
```

截止到目前为止应该有一个比较清晰的认识了，其实nextTick最终希望达到的效果就是采用异步的方式去调用flushCallbacks，至于使用宏任务还是微任务，Vue内部已经帮我们处理掉了，并不用我们去决定。至于flushCallbacks光看名字就知道是循环刚才的callbacks并执行。

```javascript
function flushCallbacks(){
    pending = flase;
    // 将callbacks做一次复制
    const copies = callbacks.slice(0);
    // 置空callbacks
    callbacks.length = 0;
    // 遍历并执行
    for(let i = 0; i < copies.length; i++){
        copies[i]();
    }
}
```

请注意，虽然我们在这里解释了flushCallbacks是干嘛的，但是要记住它是被异步处理的，而当前同步任务还并没有执行完，所以这个函数此时并没有被调用，真正要做的是走完整个同步任务，也就是我们的this.$nextTick(() => {this.show = true})终于要被调用了，感谢老天爷。当this.\$nextTick被调用后() => {this.show = true}同样被当作参数推入了callbacks中，此时可以理解为callbacks长这样[flushSchedulerQueue, () => { this.show = true }] ，然后再withMacroTask中fn.apply调用完毕useMacrotask被变回false，整个同步任务结束。

此时还记得我们在EventLoop中所讲的吗，我们会从任务队列中寻找所有的微任务，而到目前为止任务队列中并没有微任务，于是一轮事件循环完成了，浏览器重新渲染，不过此时我们的dom结构没有发生丝毫变化，所以就算浏览器没重新渲染也并不会有丝毫影响。接下来就是执行任务队列中的宏任务了，它对应的回调就是我们刚才注册的flushCallbacks。首先执行flushSchedulerQueue，其中的watcher被调用了run方法，由于此时我们的data中的show被改变成了false，所以新老虚拟的dom对比后真实dom中移除掉了绑定v-if="show"的组件。

重点来了，虽然dom中移除掉了该组件，但是其实在浏览器上这个组件是依然显示的，因为我们的事件循环还没有完成，其中还有剩余的同步任务需要被执行，浏览器并没有开始重新绘制。（如果您对此段还有疑问，我个人觉得您可能还是没有搞懂dom与浏览器上显示的区别，您可以将dom理解成控制台中elements模块内所有的节点，浏览器的中显示的内容不是与其时刻保持一致的）。

剩下需要被执行的就是() => { this.show = true }，而当执行this.show = true时我们前文所有的流程又通通执行了一遍，其中只有一些细节是与刚才不同的，我们来看一下。

1. 此函数并没有被withMacroTask包装，它是callbacks被flush时被调用的，所以useMacrotask并没有被改变依然是其默认值false。
2. 由于第一点原因我们在这次执行宏任务macrotask时产生了微任务microtask来处理本次的flushCallbacks（也就是调用了microTimerFunc）。

所以当本次macrotask结束时，本次的事件循环还没有结束，我们还留下了微任务需要处理，依然是调用flushSchedulerQueue，然后watcher.run，因为此次show已经为true了，所以对比新老虚拟dom，重新生成该组件，生命周期完成重置。此时，本轮事件循环结束，浏览器重新渲染。希望您还记得，我们的浏览器本身现在的状态就是该组件显示在可视区内，重新渲染后该组件依然显示，所以自然不会出现组件闪动的情况。

现在我相信您自己也能像清楚为什么我们的例子中使用setTimeout会有闪动，但是我还是说一下原因来看下一您与我的想法是否一致。因为setTimeout产生的是宏任务，当一轮事件循环完成后，宏任务并不会直接处理，中间插入了浏览器的绘制。浏览器重新绘制后会将显示的组件移除掉，所以区域内出现一片空白，紧接着下一次事件循环开始，宏任务被执行组件dom又被重新创建，事件循环结束，浏览器重绘，又在可视区域上将盖组件显示。所以在您的视觉效果上，该组件会有闪动，整个过程结束。

终于我们说的都说完了，如果您能坚持看到这里，十分感谢您。不过还有几点是我们依然要考虑的。

1. Vue干嘛要使用异步队列更新，这明明很麻烦又很绕。

其实文档已经告诉我们了：

> 这种在缓冲时去除重复数据对于避免不必要的计算和DOM操作上非常重要。

我们假设flushSchedulerQueue并没有通过nextTick而是直接被调用，那么第一种写法this.show = false; this.show = true都会触发watcher.run方法，导致的结果就是这种写法也可以重置组件的生命周期，您可以在Vue源码中注释掉nextTick(flushSchedulerQueue)改用flushSchedulerQueue()打断点来更加明确的体验一下流程。要知道这仅仅是一个简单的例子，实际工作中我们可能因为这种问题使dom白白被改变了几百次，我们都知道dom的操作是昂贵的，所以Vue帮我们在框架内优化了该步骤。您不妨再想一下直接flushSchedulerQueue()这种情况，组件会不会闪动，来巩固我们刚才讲过的东西。

2. 既然nextTick使用的微任务时由Promise.then().resolve()生成的，我们可不可以直接在回调函数中写this.show = false; Promise.then().resolve(() => { this.show = true })来代替this.$nextTick？很明显我既然这样问了那就是不行的，只是过程您需要自己思考。



