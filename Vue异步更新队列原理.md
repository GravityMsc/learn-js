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

其实这部分内容相信对已经看到这里的您来说早就接触过了，如果还真的不太清楚的话推荐您看下阮一峰老师的[这篇文章](http://www.ruanyifeng.com/blog/2014/10/event-loop.html)