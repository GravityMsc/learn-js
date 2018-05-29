之前遇到过一个场景，页面是哪个有几个d3.js绘制的图形。如果调整浏览器可视区的大小，会引发图形重绘。当图中的节点比较多的时候，页面会显得异常卡顿。为了限制类似于这种短时间内高频率触发的情况，我们可以使用防抖函数。

实际开发过程中，这样的情况其实很多，比如：

* 页面的scroll事件
* input框等的输入事件
* 拖拽事件用到的mousemove等

先说说防抖和节流是个啥，有啥区别

> 防抖：设定一个时间间隔，当某个频繁触发的函数执行一次后，在这个时间间隔内不会再次被触发，如果在此期间尝试触发这个函数，则时间间隔会重新开始计算。

> 节流：设定一个时间间隔，某个频繁触发的函数，在这个时间间隔内只会执行一次。也就是说，这个频繁触发的函数会以一个固定的周期执行。

### debounce（函数防抖）

大致捋一遍代码结构。为了方便阅读，我们先把源码中的Function注释掉。

```javascript
function debounce(func, wait, options){
    // 代码一开始，以闭包的形式定义了一些变量
    var lastArgs,	// 最后一次debounce的arguments，它其实起一个标记为的作用，后面会提到
        lastThis,	// 就是last this，用来修正this的指向
        maxWait,	// 存储option里面传入的maxWait值，最大等待时间
        result,		// 其实这个result始终都是undefined
        timerId,	// setTimeout赋给它，用于表示当前定时器
        lastCallTime,	// 最后一次调用debounce的时刻
        lastInvokeTime = 0,	// 最后一次调用用户传入函数的时刻
        leading = false,	// 是否在一开始就指向用户传入的函数
        maxing = false,		// 是否有最大等待时间
        trailing = true;	// 是否在等待周期结束后执行用户传入的函数
	
    // 用户传入的func必须是个函数，否则报错
    if(typeof func != 'function'){
        throw new TypeError(FUNC_ERROR_TEXT);
    }
    
    // toNumber是lodash封装的一个转类型的方法
    wait = toNumber(wait) || 0;
    
    // 获取用户传入的配置
    if(isObject(options)){
        leading = !!options.leading;
        maxing = 'maxWait' in options;
        maxWait = maxing ? nativeMax(toNumber(options.maxWait) || 0, wait) : maxWait;
        trailing = 'trailing' in options ? !!options.trailing : trailing;
    }
    
    //  执行用户传入的函数
    function invokeFunc(time) {
        // ......
    }

    //  防抖开始时执行的操作
    function leadingEdge(time) {)
        // ......
    }

    //  计算仍然需要等待的时间
    function remainingWait(time) {
        // ......
    }

    //  判断此时是否应该执行用户传入的函数
    function shouldInvoke(time) {
        // ......
    }

    //  等待时间结束后的操作
    function timerExpired() {
        // ......
    }

    //  执行用户传入的函数
    function trailingEdge(time) {
        // ......
    }

    //  取消防抖
    function cancel() {
        // ......
    }

     //  立即执行用户传入的函数
    function flush() {
        // ......
    }

    // 防抖开始的入口
    function debounced() {
        // ......
    }
      
      
    debounced.cancel = cancel;
    debounced.flush = flush;
    return debounced;
}
```

我们先从入口函数开始。函数开始执行后，首先会出现三种情况：

* 时间上达到了可以执行的条件；
* 时间上不满足条件，但是此时的定时器并没有启动；
* 不满足条件，返回undefined

代码中timerId = setTimeout(timerExpired, wait);是用来设置定时器，到时间后触发trailingEdge这个函数。

```javascript
function debounced(){
    var time = now(),
        isInvoking = shouldInvoke(time);	// 判断此时是否可以开始执行用户传入的函数
    
    lastArgs = arguments;
    lastThis = this;
    lastCallTime = time;
    
    if(isInvoking){
        // 如果此时并没有定时器存在，就开始进入防抖阶段
        if(timerId === undefined){
            return leadingEdge(lastCallTime);
        }
        // 如果设置了最大等待时间，便立即执行用户传入的函数
        if(maxing){
            // Handle invocations in a tight loop.
            timerId = setTimeout(timeExpired, wait);
            return invokeFunc(lastCallTime);
        }
    }
    if(timerId === undefined){
        timerId = setTimeout(timerExpired, wait);
    }
    
    // 不满足条件， return undefined
    return result;
}
```

我们先来看看shouldInvoke是如何判断函数是否可以执行的。

```javascript
function shouldInvoke(time){
    // lastCallTime初始值是undefined，lastInvokeTime初始值是0，
    // 防抖函数被手动取消后，这两个值会被设为初始值
    var timeSinceLastCall = time - lastCallTime,
        timeSinceLastInvoke = time - lastInvokeTime;
    
    // Either this is the first call, activity has stopped and we're at the
    // trailing edge, the system time has gone backwards and we're treating
    // it as the trailing edge, or we've hit the `maxWait` limit.
    return (
    		lastCallTime === undefined ||	// 上次执行
        	(timeSinceLastCall >= wait) ||	// 上次调用时刻距离现在已经大于wait值
        	(timeSinceLastCall < 0) ||		// 当前时间 - 上次调用时间小于0，应该只可能是手动修改了系统时间吧
        	(maxing && timeSinceLastInvoke >= maxWait) 	// 设置了最大等待时间，且已超时
    );
}
```

我们继续分析函数开始的阶段leadingEdge。首先重置防抖函数最后调用时间，然后去触发一个定时器，保证wait后按接下来的执行。最后判断如果leading是true的话，立即执行用户传入的函数：

```javascript
function leadingEdge(time){
    // Reset any 'maxWait' timer.
    lastInvokeTime = time;
    // Start the timer for the trailing edge
    timerId = setTimeout(timerExpired, wait);
    // Invoke the leading edge.
    return leading ? invokeFunc(time) : result;
}
```

我们已经不止一次去设定定时器了，我们来探究一下里面到底做了啥。其实很简单，判断时间是否符合执行条件，符合的话触发trailingEdge，也就是后续操作，否则计算需要等待的时间，并重新调用这个函数，其实这里就是防抖的核心所在了。

```javascript
function timerExpired(){
    var time = now();
    if(shouldInvoke(time)){
        return trailingEdge(time);
    }
    // Restart the timer.
    timerId = setTimeout(timerExpired, remainingWait(time));
}
```

至于如何重新计算剩余时间的，这里不作过多解释，大家一看便知。

```javascript
function remainingWait(time){
    var timeSinceLastCall = time - lastCallTime,
        timeSinceLastInvoke = time - lastInvokeTime,
        timeWaiting = wait - timeSinceLastCall;
    
    return maxing ? nativeMin(timeWaiting, maxWait - timeSinceLastInvoke) : timeWaiting;
}
```

我们说说等待时间到了以后的操作。重置了一些本周期的变量。并且，如果trailing是true，而且lastArgs存在时，才会再次执行用户传入的参数。这里解释了本章开头提到的lastArgs只是个标记位，如注释所说，它表示debounce至少执行了一次。

```javascript
function trailingEdge(time){
    timerId = undefined;
    
    // Only invoke if we have 'lastArgs' which means 'func' has been
    // debounced at least once.
    if(trailing && lastArgs){
        return invokeFunc(time)l
    }
    lastArgs = lastThis = undefined;
    return result;
}
```

执行用户传入的函数比较简单，我们知道call和apply是会立即执行的，其实最后的result还是undefined。

```javascript
function invokeFunc(time){
    var args = lastArgs,
        thisArg = lastThis;
    // 重置了一些条件
    lastArgs = lastThis = undefined;
    lastInvokeTime = time;
    // 执行用户传入函数
    result = func.apply(thisArg, args);
    return result;
}
```

最后就是取消防抖和立即执行用户传入函数的过程了，代码一目了然，不做过多解释。

```javascript
function cancel(){
    if(timerId !== undefined){
        clearTimeout(timerId);
    }
    lastInvokeTime = 0;
    lastArgs = lastCallTime  = lastThis = timerId = undefined;
}

function flush(){
    return timerId === undefined ? result : trailingEdge(now());
}
```

### throttle（函数节流）

节流其实原理和防抖是一样的，只不过触发条件不同而已，其实就是maxWait为wait的防抖函数。

```javascript
function throttle(func, wait, options){
    var leading  = true,
        trailing = true;
    if(typeof func != 'function'){
        throw new TypeError(FUNC_ERROR_TEXT);
    }
    if(isObject(options)){
        leading = 'leading' in options ? !!options.leading : leading;
        trailing = 'trailing' in options ? !!options.trailing : trailing;
    }
    return debounce(func, wait, {
        'leading': leading,
        'maxWait': wait,
        'trailing': trailing
    });
}
```

### 总结

我们发现，其实lodash除了在cancel函数中使用了清楚定时器的操作外，其他地方并没有去关心定时器，而是很巧妙的在定时器里加了一个判断条件来判断后续函数是否可以执行。这就避免了手动管理计时器。

lodash替我们考虑到了一些比较少见的情景，而且还有一定的容错性。即便ES6实现了很多目前常用的工具函数，但是面对复杂的情景，我们依然可以以按需引入的方式使用lodash的一些函数来提升开发效率，同时使得我们的程序更加健壮。