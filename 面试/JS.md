## 一、防抖和节流

**相同**：在不影响客户体验的前提下，将频繁的回调函数，进行次数缩减，避免大量计算导致的页面卡顿。

**不同**：防抖是将多次执行变为最后一次执行，节流是将多次执行变为在规定时间内只执行一次。



### 防抖

**定义：**

指触发事件后在**规定时间内**回调函数**只能执行一次**，如果在规定时间内**又**触发了该条件，则会重新开始算规定时间。

网上有这个比喻：函数防抖就是法师释放技能的时候要读条，技能读条没完成再释放技能就会刷新技能，重新进行读条。

四个字总结就是：**延时执行**

**应用场景：**

两个条件：

1. 如果客户连续的操作会导致频繁的事件回调（可能引起页面卡顿）。
2. 客户只关心“最后一次”操作（也可以理解为停止连续操作后）所返回的结果。

例如：

* 输入搜索联想，用户在不断输入值时，用防抖来节约请求资源。
* 按钮点击：收藏，点赞，心标等。

**原理：**

通过定时器将回调函数进行延时，如果在规定时间内继续回调，发现存在之前的定时器，则将该定时器清除，并重新设置定时器，这里有个细节，就是后面所有的回调函数都要能访问到之前设置的定时器，这时就需要用到闭包。

**两种版本：**

防抖分为两种：

* 非立即执行版：事件触发->延时->执行回调函数；如果在延时中，继续触发事件，则会重新进行延时，在延时结束后执行回调函数。常见的例子：就是input搜索框，客户输完过一会就会自动搜索。
* 立即执行版：事件触发->执行回调函数->延时；如果在延时中，继续触发事件，则会重新进行延时，在延时结束后，并不会执行回调函数，常见例子：就是对于按钮防止点击，例如点赞，心标，收藏等有立即反馈的按钮。

**实现代码及思路：**

```javascript
//非立即执行版:
//首先准备我们要使用的回调函数
function shotCat (content) {
  console.log('shotCat出品,必属精品!必须点赞!(滑稽)')
}

//然后准备包装函数:
//1,保存定时器标识 
//2,返回闭包函数: 1)对定时器的判断清除;2)一般还需要保存函数的参数(一般就是事件返回的对象)和上下文(定时器存在this隐式丢失,详情可以看我不知道的js上)
//最后补充一句,这里不建议通过定义一个全局变量来替代闭包保存定时器标识.
function debounce(fun, delay = 500) {
//let timer = null 保存定时器
    return function (args) {
        let that = this
        let _args = args
		//这里对定时器的设置有两种方法,第一种就是将定时器保存在函数(函数也是对象)的属性上,
		//这种写法,很简便,但不是很常用
        clearTimeout(fun.timer)
        fun.timer = setTimeout(function () {
            fun.call(that, _args)
        }, delay)
		//另外一种写法就是我们比较常见的
		//if (timer) clearTimeout(timer);     相比上面的方法,这里多一个判断
		//timer = setTimeout(function () {
        //    fun.call(that, _args)
        //}, delay)
    }
}
 //接着用变量保存保存 debounce 返回的带有延时功能的函数
let debounceShotCat = debounce(shotCat, 500)  

//最后添加事件监听 回调debounceShotCat 并传入事件返回的对象
let input = document.getElementById('debounce')
input.addEventListener('keyup', function (e) {
        debounceShotCat(e.target.value)
})



//带有立即执行选项的防抖函数:
//思路和上面的大致相同,如果是立即执行,则定时器中不再包含回调函数,而是在回调函数执行后,仅起到延时和重置定时器标识的作用
function debounce(fun, delay = 500,immediate = true) {
    let timer = null //保存定时器
    return function (args) {
        let that = this
        let _args = args
		if (timer) clearTimeout(timer);  //不管是否立即执行都需要首先清空定时器
        if (immediate) {
		    if ( !timer) fun.apply(that, _args)  //如果定时器不存在,则说明延时已过,可以立即执行函数
			//不管上一个延时是否完成,都需要重置定时器
            timer = setTimeout(function(){
                timer = null; //到时间后,定时器自动设为null,不仅方便判断定时器状态还能避免内存泄露
            }, delay)
        }
        else {
		//如果是非立即执行版,则重新设定定时器,并将回调函数放入其中
            timer = setTimeout(function(){
                fun.call(that, _args)
            }, delay);
        }
    }
}
```

### 节流

**定义：**

当持续触发事件时，在规定时间段内只能调用一次回调函数。如果在规定时间内**又**触发了该事件，**则什么也不做，也不会重置定时器**。

**与防抖比较：**

防抖是将多次执行变为最后一次执行，节流是将多次执行变为在规定时间内只执行一次，一般**不会重置定时器**，即不会if(timer) clearTimeout(timer); （**时间戳+定时器版除外**）

**应用场景：**

两个条件：

1. 客户连续频繁地触发事件。
2. 客户不再只关心“最后一次”操作后的结果反馈，而是在操作过程中持续的反馈。

例如：

* 鼠标不断点击触发，点击事件在规定时间内只触发一次（单位时间内只触发一次）。
* 监听滚动事件，比如是否滑到底部自动加载更多，用throttle来判断。

**注意：**何为连续频繁地触发事件，就是**事件触发的时间间隔至少是要比规定的时间要短**。

**原理：**

节流有两种实现方式：

1. 时间戳方式：通过闭包保存上一次的时间戳，然后与事件触发的时间戳比较，如果大于规定时间，则执行回调，否则就什么都不处理。

   特点：**一般第一次会立即执行**，之后连续频繁地触发事件，也是超过了规定时间才会执行一次。最后一次触发事件，也不会执行（说明：如果你最后一次触发时间大于规定时间，这样就算不上连续频繁触发了）。

2. 定时器方式：原理与防抖类似，通过闭包保存上一次定时器状态，然后事件触发时，如果定时器为null（即代表此时间隔已经大于规定时间），则设置新的定时器，到时间后执行回调函数，并将定时器置为null。

   特点：**当第一次触发事件时，不会立即执行函数，到了规定时间后才会执行。**之后连续频繁地触发事件，也是到了规定时间才会执行一次（因为定时器）。当最后一次停止触发后，由于定时器的延时，还会执行一次回调函数（那也是上一次成功触发执行的回调，而不是你最后一次触发产生的）。**一句话总结就是延时回调，你能看到的回调都是上次成功触发产生的，而不是你此刻触发产生的**。

**说明：**这两者最大的区别：时间戳版的函数触发是在规定时间开始的时候，而定时器版的函数触发是在规定时间结束的时候。

**实现代码及思路：**

```javascript
//时间戳版：
//这里fun指的就是回调函数,我就不写出来了
function throttle(fun, delay = 500) {
    let previous = 0;  //记录上一次触发的时间戳.这里初始设为0,是为了确保第一次触发产生回调
    return function(args) {
        let now = Date.now(); //记录此刻触发时的时间戳
        let that = this;
        let _args = args;
        if (now - previous > delay) {  //如果时间差大于规定时间,则触发
            fun.apply(that, _args);
            previous = now;
        }
    }
}


//定时器版:
function throttle(fun, delay = 500) {
    let timer;
    return function(args) {
        let that = this;
        let _args = args;
        if (!timer) {  //如果定时器不存在,则设置新的定时器,到时后,才执行回调,并将定时器设为null
            timer = setTimeout(function(){
                timer = null;
                fun.apply(that, _args)
            }, delay)
        }

    }
}




//时间戳+定时器版: 实现第一次触发可以立即响应,结束触发后也能有响应 (该版才是最符合实际工作需求)
//该版主体思路还是时间戳版,定时器的作用仅仅是执行最后一次回调
function throttle(fun, delay = 500) {
     let timer = null;
     let previous = 0;
     return function(args) {
             let now = Date.now();
             let remaining = delay - (now - previous); //距离规定时间,还剩多少时间
             let that = this;
             let _args = args;
             clearTimeout(timer);  //清除之前设置的定时器
              if (remaining <= 0) {
                    fun.apply(that, _args);
                    previous = Date.now();
              } else {
                    timer = setTimeout(function(){
                    fun.apply(that, _args)
            }, remaining); //因为上面添加的clearTimeout.实际这个定时器只有最后一次才会执行
              }
      }
}
```

## 二、JS垃圾回收机制策略

### 标记清除算法

JavaScript中最常用的垃圾收集方式是标记清除（mark-and-sweep）。

这个算法把“对象是否不再需要”简化定义为“对象是否可以获得”。

该算法假定设置一个叫做根（root）的对象（在JavaScript里，根是全局对象）。垃圾回收器将定期从根开始扫描内存中的对象。凡是能从根部到达的对象，都是还需要使用的。那些无法由根部触发触及到的对象被标记为不再使用，稍后进行回收。

此算法可以分为两个阶段，一个是标记阶段（mark），一个是清除阶段（sweep）。

1. 标记阶段，垃圾回收器会从根对象开始遍历。每一个可以从根对象访问到的对象都会被添加一个标识，于是这个对象就被标识为可达到对象。
2. 清除阶段，垃圾回收器会对堆内存从头到尾进行线性遍历，如果发现有对象没有被标识为可达到对象，那么久将此阶段占用的内存回收，并且将原来标记为可达到对象的标识清除，以便进行下一次垃圾回收操作。

![标记清除垃圾回收示意图](https://user-gold-cdn.xitu.io/2019/2/22/169158136a253bc3?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

在标记阶段，从根对象1可以访问到B，从B又可以访问到E，那么B和E都是可到达对象，同样的道理，F、G、J和K都是可到达对象。

在回收阶段，所有未标记为可到达的对象都会被垃圾回收器回收。

**何时开始垃圾回收？**

通常来说，在使用标记清除算法时，未引用对象并不会被立即回收。取而代之的做法是，垃圾对象将一直累积到内存耗尽为止。当内存耗尽时，程序将会被挂起，垃圾回收开始执行。

补充: 从2012年起，所有现代浏览器都使用了标记-清除垃圾回收算法。所有对JavaScript垃圾回收算法的改进都是基于标记-清除算法的改进，并没有改进标记-清除算法本身和它对“对象是否不再需要”的简化定义。

**标记清除算法缺陷**

* 那些无法从根对象查询到的对象都将被清除
* 垃圾收集后有可能会造成大量的内存碎片，像上面的图片所示，垃圾收集后内存中存在三个内存碎片，假设一个方格代表1个单位的内存，如果有一个对象需要占用3个内存单位的话，那么就会导致内存分配器一直处于暂停状态，而垃圾收集器一直在尝试进行垃圾收集，直到内存溢出。

### 引用计数算法

这是最初级的垃圾收集算法，现在已经没有浏览器会用这种算法。

此算法把“对象是否不再需要”简化定义为“对象没有其他对象引用到它”。如果没哟引用指向该对象（零引用），对象将被垃圾回收机制回收。

引用计数的含义是跟踪记录每个值被引用的次数。当声明了一个变量并将一个引用类型值赋给该变量时，则这个值的引用次数就是1。如果同一个值又被赋给另一个变量，则该值的引用次数加1。相反，如果包含对这个值引用的变量又取得了另外一个值，则这个值的引用次数减1。当这个值的引用次数变成0时，则说明没有办法再访问这个值了，因而就可以将其占用的内存空间回收回来。这样，当垃圾收集器下次再运行时，它就会释放那些引用次数为零的值所占用的内存。

**引用计数缺陷**

该算法有个限制：无法处理循环引用。如果两个对象被创建，并互相引用，形成了一个循环。它们被调用之后会离开函数作用域，所以它们已经没有用了，可以被回收了。然而，引用计数算法考虑到它们互相都有至少一次引用，所以它们不会被回收。

### Chrome V8 垃圾回收算法

Chrome 浏览器所使用的 V8 引擎就是采用的分代回收策略。这个和 Java 回收策略思想是一致的。目的是通过区分「临时」与「持久」对象；多回收「临时对象区」（**新生代**younggeneration），少回收「持久对象区」（**老生代** tenured generation），减少每次需遍历的对象，从而减少每次GC的耗时。

####V8的内存限制

在node中JavaScript能使用的内存是有限制的。

> 1. 64位系统下约为1.4GB。
> 2. 32位系统下约为0.7GB。

对应到分代内存中，默认情况下。

> 1. 32位系统新生代内存大小为16MB，老生代内存大小为700MB。
> 2. 64位系统下，新生代内存大小为32MB，老生代内存大小为1.4GB。

新生代平均分成两块相等的内存空间，叫做semispace，每块内存大小8MB（32位）或16MB（64位）。

这个限制在node启动的时候可以通过传递--max-old-space-size 和 --max-new-space-size来调整，如：

```bash
node --max-old-space-size=1700 app.js //单位为MB
node --max-new-space-size=1024 app.js //单位为MB
```

上述参数在V8初始化时生效，一旦生效就不能再动态改变。

**内存限制的原因**

至于V8为何要限制堆的大小，表层原因：V8最初为浏览器而设计，不太可能遇到用大量内存的场景。深层原因：V8的垃圾回收机制的限制。官方说法，以1.5GB的垃圾回收堆内存为例，V8做一次小的垃圾回收需要50毫秒以上，做一次非增量式的垃圾回收甚至要1秒以上。这是垃圾回收中引起JS线程暂停执行的时间，在这样时间花销下，应用的性能和响应能力都会直线下降。

####V8的分代回收（Generation GC）

V8垃圾回收策略主要基于分代式垃圾回收机制。现代的垃圾回收算法中按对象的存活时间将内存的垃圾回收进行不同的分代，然后分别对不同分代的内存施以更高效的算法。

**V8的内存分代**

在V8中，主要将内存分为新生代和老生代，**新生代内存 存储的为存活时间较短的对象**，**老生代内存 存储的为存活时间较长或常驻内存的对象**，如下图：

![V8对象内存进行对象分代](https://user-gold-cdn.xitu.io/2019/2/19/16904df8de47c353?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

**V8堆的整体大小就是新生代所用内存空间加上老生代的内存空间。**

#### V8新生代算法（Scavenge）:

在分代基础上，新生代中的对象主要通过Scavenge算法进行垃圾回收。在Scavenge的具体实现中，主要采用了Cheney算法

Cheney算法是一种采用复制的方式实现的垃圾回收算法。它将堆内存一分为二，每一部分空间称为semispace。在这两个semispace空间中，只有一个处于使用中，另一个处于闲置状态。**处于使用状态的semispace空间称为From空间，处于闲置状态的空间称为To空间。**

当我们分配对象时，先是在From空间中进行分配。当开始进行垃圾回收时，会检查From空间中的存活对象，这些存活对象将被复制到To空间中，而(From空间内的)非存活对象占用的空间将会被释放。完成复制后，From空间和To空间的角色发生对换(**即以前的From空间释放后变为To;To空间在复制存活的对象后,变为From空间**)。简而言之，在垃圾回收过程中，就是通过将存活对象在两个semispace空间之间进行复制。

**Scavenge的缺点:**
 **只能使用堆内存中的一半**，这是由划分空间和复制机制所决定的。

**Scavenge的优点:**
 Scavenge由于只复制存活的对象，并且对于生命周期短的场景存活对象只占少部分，所以它**在时间效率上有优异的表现。** **Scavenge是典型的牺牲空间换取时间的算法，** 所以无法大规模地应用到所有的垃圾回收中。但可以发现，Scavenge非常适合应用在新生代中，因为新生代中对象的生命周期较短，恰恰适合这个算法。

**晋升:**	
 实际使用的堆内存是新生代的两个semispace空间大小和老生代所用内存大小之和。当一个对象经过多次复制依然存活时，它将会被认为是生命周期较长的对象。这种较长生命周期的对象随后会被移动到老生代中，采用新的算法进行管理。**对象从新生代中移动到老生代中的过程称为晋升。**

在单纯的Scavenge过程中，From空间中的存活对象会被复制到To空间中去，然后对From空间和To空间进行角色对换（又称翻转）。但在分代式垃圾回收前提下，**From空间中的存活对象在复制到To空间之前需要进行检查。在一定条件下，需要将存活周期长的对象移动到老生代中，也就是完成对象晋升。**

**晋升条件:**
 对象晋升的条件主要有两个，一个是对象是否经历过Scavenge回收，一个是To空间的内存占用比超过25%限制。

**设置25%这个限制值的原因:**
 当这次Scavenge回收完成后，这个To空间将变成From空间，接下来的内存分配将在这个空间中进行。**如果占比过高，会影响后续的内存分配。** 对象晋升后，将会在老生代空间中作为存活周期较长的对象来对待，接受新的回收算法处理。

![V8 垃圾回收新生代算法](https://user-gold-cdn.xitu.io/2019/2/19/16904ebc54c74a12?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

#### V8老生代算法（Mark-Sweep && Mark-Compact）:

> 对于老生代中的对象，由于存活对象占较大比重，再采用Scavenge的方式会有两个问题：一个是存活对象较多，复制存活对象的效率将会很低；另一个问题依然是浪费一半空间的问题。为此，V8在老生代中主要采用Mark-Sweep和Mark-Compact相结合的方式进行垃圾回收。

**Mark-Sweep:**
 Mark-Sweep是标记清除的意思，它分为标记和清除两个阶段。与Scavenge相比，Mark-Sweep并不将内存空间划分为两半，所以**不存在浪费一半空间的行为**。与Scavenge复制活着的对象不同，**Mark-Sweep在标记阶段遍历堆中所有对象，并标记活着的对象，在随后的清除阶段中，只清除没有被标记的对象。** 可以看出，**Scavenge中只复制活着的对象，而Mark-Sweep只清理死亡对象。** 活对象在新生代中只占较小部分，死对象在老生代中只占较小部分，这是两种回收方式能高效处理的原因。

下图为Mark-Sweep在老生代空间中标记的示意图，黑色部分标记为死亡对象

![老生代](https://user-gold-cdn.xitu.io/2019/2/19/16904ed3e993dca1?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

**Mark-Sweep最大的问题:**
 在进行一次标记清除回收后，内存空间会出现不连续的状态。这种内存碎片会对后续的内存分配造成问题，因为很可能出现**需要分配一个大对象的情况，这时所有的碎片空间都无法完成此次分配，就会提前触发垃圾回收，而这次回收是不必要的。(注意理解这句话,不要把内存想象成液体.而是固体,就像一个个散乱排列的麻将,需要进行排序处理--即后面要讲的 Mark-Compact)**

**Mark-Compact:**
 为了解决Mark-Sweep的内存碎片问题，Mark-Compact被提出来。Mark-Compact是标记整理的意思，是在Mark-Sweep的基础上演变而来的。它们的差别在于对象在标记为死亡后，**在整理的过程中，将活着的对象往一端移动，移动完成后，直接清理掉边界外的内存。** 下图为Mark-Compact完成标记并移动存活对象后的示意图，白色格子为存活对象，深色格子为死亡对象，浅色格子为存活对象移动后留下的空洞。

![老生代对象清除和整理](https://user-gold-cdn.xitu.io/2019/2/19/16904ed8d55fb20a?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

**完成移动后，就可以直接清除最右边的存活对象后面的内存区域完成回收。**

**Mark-Sweep、Mark-Compact、Scavenge三种主要垃圾回收算法的简单对比**

| 回收算法     | Mark-Sweep   | Mark-Compact | Scavenge           |
| ------------ | ------------ | ------------ | ------------------ |
| 速度         | 中等         | 最慢         | 最快               |
| 空间开销     | 少（有碎片） | 少（无碎片） | 双倍空间（无碎片） |
| 是否移动对象 | 否           | 是           | 是                 |

从表格上看，Mark-Sweep和Mark-Compact之间，由于Mark-Compact需要移动对象，所以它的执行速度不可能很快，所以在取舍上，V8主要使用Mark-Sweep，**在空间不足以对从新生代中晋升过来的对象进行分配时才使用Mark-Compact。**

#### 增量式标记回收(Incremental Marking):

为了避免出现js应用逻辑与垃圾回收器看到的不一致的情况，**垃圾回收的3种基本算法都需要将应用逻辑暂停下来**，待执行完垃圾回收后再恢复执行应用逻辑，这种行为被称为“全停顿”（stop-the-world）。在V8的分代式垃圾回收中，一次小垃圾回收只收集新生代，由于新生代默认配置得较小，且其中存活对象通常较少，所以即便它是全停顿的影响也不大。但V8的老生代通常配置得较大，且存活对象较多，全堆垃圾回收（full垃圾回收）的标记、清理、整理等动作造成的停顿就会比较可怕，需要设法改善(PS: 若V8的堆内存为1.5GB，V8做一次小的垃圾回收需要50ms以上，做一次非增量式的垃圾回收甚至要1秒以上。)。

为了降低全堆垃圾回收带来的停顿时间，V8先从标记阶段入手，将原本要一口气停顿完成的动作改为增量标记（incremental marking），也就是**拆分为许多小“步进”，每做完一“步进”就让js应用逻辑执行一小会，垃圾回收与应用逻辑交替执行直到标记阶段完成。**

V8在经过增量标记的改进后，垃圾回收的最大停顿时间可以减少到原本的1/6左右。

V8后续还引入了延迟清理（lazy sweeping）与增量式整理（incremental compaction），让清理与整理动作也变成增量式的。同时还计划引入并行标记与并行清理，进一步利用多核性能降低每次停顿的时间。

### 减少垃圾和回收对性能的影响:

主要注意以下两点:

- 让垃圾回收尽量少地进行，尤其是全堆垃圾回收。这部分我们基本上帮不了什么忙,主要靠v8自己的优化机制.
- 避免内存泄露,让内存及时得到释放. 这部分是我们需要注意的.具体可以查看,本系列的内存泄露章节,有超详细讲解.

##三、MVC和MVVM的区别

MVC，MVP和MVVM都是常见的软件架构设计模式（Architectural Pattern），它通过分离关注点来改进代码的组织方式。不同于设计模式（Design Pattern），只是为了解决一类问题而总结出的抽象方法，一种架构模式往往使用了多种设计模式。

要了解MVC、MVP和MVVM，就要知道它们的相同点和不同点。不同部分是C(Controller)、P(Presenter)、VM(View-Model)，而相同的部分则是MV(Model-View)。

###MVC模式

**MVC模式（Model–view–controller）:**
 是软件工程中的一种软件架构模式，把软件系统分为三个基本部分：模型（Model）、视图（View）和控制器（Controller）。

- 模型（Model） - Model层用于封装和应用程序的业务逻辑相关的数据以及对数据的处理方法。一旦数据发生变化，模型将通知有关的视图。
- 视图（View） - View作为视图层，主要负责数据的展示,并且响应用户操作.
- 控制器（Controller）- 控制器是模型和视图之间的纽带，接收View传来的用户事件并且传递给Model，同时利用从Model传来的最新模型控制更新View.



![img](https://user-gold-cdn.xitu.io/2019/2/21/169102cbc41f9719?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

####**数据关系:**

- View 接受用户交互请求
- View 将请求转交给Controller
- Controller 操作Model进行数据更新
- 数据更新之后，Model通知View更新数据变化.**PS:** 还有一种是View作为Observer监听Model中的任意更新，一旦有更新事件发出，View会自动触发更新以展示最新的Model状态.这种方式提升了整体效率，简化了Controller的功能，不过也导致了View与Model之间的紧耦合。
- View 更新变化数据

**方式:**

所有方式都是单向通信

####**结构实现:**

- View ：使用 组合(Composite)模式
- View和Controller：使用 策略(Strategy)模式
- Model和 View：使用 观察者(Observer)模式同步信息

####**缺点:**

- **View层过重:** View强依赖于Model的,并且可以直接访问Model.所以不可避免的View还要包括一些业务逻辑.导致view过重,后期修改比较困难,且复用程度低.
- **View层与Controller层也是耦合紧密:** View与Controller虽然看似是相互分离，但却是联系紧密.经常View和Controller一一对应的，捆绑起来作为一个组件使用.解耦程度不足.

###MVVM模式

MVVM是Model-View-ViewModel的简写。由Microsoft提出，并经由Martin Fowler布道传播。在 MVVM 中，不需要Presenter手动地同步View和Model.View 是通过数据驱动的，Model一旦改变就会相应的刷新对应的 View，View 如果改变，也会改变对应的Model。这种方式就可以在业务处理中只关心数据的流转，而无需直接和页面打交道。ViewModel 只关心数据和业务的处理，不关心 View 如何处理数据，在这种情况下，View 和 Model 都可以独立出来，任何一方改变了也不一定需要改变另一方，并且可以将一些可复用的逻辑放在一个 ViewModel 中，让多个 View 复用这个 ViewModel。

- Model - Model层仅仅关注数据本身，不关心任何行为（格式化数据由View负责），这里可以把它理解为一个类似json的数据对象。
- View - MVVM中的View通过使用模板语法来声明式的将数据渲染进DOM，当ViewModel对Model进行更新的时候，会通过数据绑定更新到View。
- ViewModel - 类似与Presenter. ViewModel会对View 层的声明进行处理.当 ViewModel 中数据变化，View 层会进行更新;如果是双向绑定,一旦View对绑定的数据进行操作，则ViewModel 中的数据也会进行自动更新.



![img](https://user-gold-cdn.xitu.io/2019/2/21/169102df8742a9cf?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

####**数据关系:**

- View 接收用户交互请求
- View 将请求转交给ViewModel
- ViewModel 操作Model数据更新
- Model 更新完数据，通知ViewModel数据发生变化
- ViewModel 更新View数据

**方式:**

双向绑定。View/Model的变动，自动反映在 ViewModel，反之亦然。

####**实现数据绑定的方式：**

- 数据劫持 (Vue)
- 发布-订阅模式 (Knockout、Backbone)
- 脏值检查 (旧版Angular)

**使用:**

- 可以兼容你当下使用的 MVC/MVP 框架。
- 增加你的应用的可测试性。
- 配合一个绑定机制效果最好。

####**MVVM优点:**

MVVM模式和MVC模式一样，主要目的是分离视图（View）和模型（Model），有几大优点:

- 1,低耦合。View可以独立于Model变化和修改，一个ViewModel可以绑定到不同的”View”上，当View变化的时候Model可以不变，当Model变化的时候View也可以不变。
- 2,可重用性。你可以把一些视图逻辑放在一个ViewModel里面，让很多view重用这段视图逻辑。
- 3, 独立开发。开发人员可以专注于业务逻辑和数据的开发（ViewModel），设计人员可以专注于页面设计，生成xml代码。
- 4, 可测试。界面素来是比较难于测试的，而现在测试可以针对ViewModel来写。

####**MVVM缺点:**

- 类会增多，ViewModel会越加庞大，调用的复杂度增加。

## 四、this的用法

关于this是面试和日常开发中非常常见的概念之一，也是最易弄混的概念之一，this这个名称本身有时也容易让人迷惑。自己之前面试时在this上也是栽过几次跟头，特地梳理一下，目的是彻底吃透，以绝后患。

###常见错误理解

1. **this是指向自身吗？**

   特别是在函数中使用this的时候，this是指的所在的这个函数对象吗？看下面示例：

   ```javascript
    function test() {
    	console.log(this.name);
    }
    test.name='aaaa';
    test();
   ```
   

在上面这个示例里如果this是指向当前函数的话，执行test后是不是应该输出'aaaa'，但实际上输出的是空字符串，因为此时this指向的是window（在浏览器里执行），所以this并不是来指向自身的。（输出的具体原因下面再分析）

2. **this指向函数的作用域吗**

   这个我们同样可以通过代码来验证下，如下：

   ```javascript
    function test() {
    	var name = 'bbbbb'
    	console.log(this.name);
    }
    test();
   ```
   

执行后发现输出的仍然是空字符串，原因和上面一样，this没有指向当前函数的作用域。但this一定会不指向当前函数作用域吗？也不一定，只需知道不能根据所在函数作用域来确定this的指向就对了，应该是确定this的指向，再确定是不是当前函数的作用域。

解决掉常见的理解错误后，我们看下this其实是在运行时（即被调用时）进行绑定的，并不是在声明时绑定，它的上下文取决于函数调用时的各种条件。this 的绑定和函数声明的位置没有任何关系，只取决于函数的调用方式。确定this指向的步骤应该是“确定调用位置->应用规则->确定this指向”。

###寻找调用位置

this既然是函数调用是才绑定的，那么需要首先需要确定函数的调用位置。这个一般是比较容易的，先确认调用栈，然后当前调用栈的前一个就是调用位置了。示例如下：

```javascript
function a() {
  // 调用栈是c->b->a, 调用位置b
  console.log(a);
}

function b() {
  // 调用栈是c->b, 调用位置c
  a()
}

function c() {
  // 调用栈是c, 调用位置是全局作用域
  b();
}
c(); // c的调用位置
```

###应用绑定规则

确定调用位置后，需要应this的绑定规则，有四种绑定规则，判断条件如下：

####默认绑定

一般可以理解为无法应用其他规则时的兜底默认规则，独立函数调用时一般适用。

```javascript
function test(){
	console.log(this.a);
}
var a = 'aaa'; // 或者window.a='aaa'
test(); // 输出aaa
```

上面这种方式便是默认绑定，test()不在任何对象内的独立调用，适用于默认绑定，**默认绑定this指向的全局对象，在浏览器里面就是window，在node里面就是global， ps：严格模式下，全局对象无法使用默认绑定，默认绑定会绑定到undefined上**

####隐式绑定

隐式绑定存在于在调用位置有上下文对象或者说调用时被对象包含或拥有，示例如下：

```javascript
const obj = {
	name: 'oooo',
	say: function(){
		console.log(this.name);
	}
}
obj.say();// oooo
```

看上面函数say的调用，不是say单独调用，而是被对象obj包含着调用，此时this是指向obj对象的。

##### 隐式丢失

有一种情况是看似应该是隐式绑定，但实际却是默认绑定，有两个栗子如下：

```javascript
栗子one：
var name = 'globallll';
var obj = {
	name: 'oooo',
	say: function(){
		console.log(this.name);
	}
}
var copy = obj.say;
copy();// globallll

栗子two:
var name = 'globallll';
var obj = {
	name: 'oooo',
	say: function(){
		console.log(this.name);
	}
}
function b(func){
	func();
}
b(obj.say);// globallll
```

看起来say函数的确是obj对象的一部分呀，但为什么看起来this是指向的window呢？《you don't know JavaScript》里面把这种特殊对待称为是隐式丢失，但我理解是这种情况是不满足隐式函数绑定的，因为隐式函数绑定应当是调用是被对象包含着调用，而不是说只要是对象的其中一部分就可以了，重点在于调用时是否被函数包含着！

我们来看下上述的两个例子，第一个是把obj里的函数say的引用赋值给copy变量，再通过copy来调用，copy调用时并没有被obj包含着调用，这就适用默认绑定规则--独立函数调用，因此此时this是指向window的。第二个例子同理，只不过看起来是调用的obj.say(),但实际过程是：

```javascript
	func = obj.say;
	func();
```

和第一个一样都有一个赋值的过程。

####显式绑定

首先为啥需要显示绑定呢？因为以上两个规则会导致this的指向不稳定，但有时我们需要函数中this稳定指向某个对象的。比如下面这个：

```javascript
var obj = {
	name: 'ooooo',
	say: function(){
		console.log(this.name);
	}
}
var name = 'globalllll';
setTimeout(obj.say, 1000); // globalllll
```

这个例子中我们其实是想让this指向obj然后输出‘ooooo’的，但实际上调用的过程中首先进行了赋值然后进行了调用，导致使用默认绑定，this指向了window。为了能固定this的绑定，才有了显示绑定。

显示绑定有三种方式：apply，call和bind。这三个函数大家应该用的比较多了，其中apply和call只有传参的区别，而apply和bind的区别在于apply绑定后立即执行，而bind可以返回绑定后的函数。上述例子可以这么解决：

```javascript
var obj = {
	name: 'ooooo',
	say: function(){
		console.log(this.name);
	}
}
var test = obj.say.bind(obj);//绑定后this指向不可修改
//或者
//var test = function (){
//	obj.say.call(obj);
//}
var name = 'globalllll';
setTimeout(test, 1000); // oooo
```

思考🤔：如何用apply或者call实现bind？

进阶：实现apply？

####new绑定

new是一个由类新建示例的过程。使用new时会调用构造函数，但是过程中并不是实例化了一些类，而是通过新建一个对象，然后执行原型的链接，新对象绑定函数调用的this，返回这个对象，返回的这个对象我们就称为一个实例。比如： function Person(name){ this.name = name; } var student = new Person('LiMing'); console.log(student.name) // LiMing

上述代码的过程是：

1. 新建一个对象
2. 执行原型的链接（不是本文的重点）
3. 将新建对象绑定到Person中的this上，也就是目前this指向新建对象
4. 返回这个新建对象给student，即student等于新建对象

这种在new的过程中绑定this的方式称为是new绑定。

###优先级

上面我们了解四种绑定规则，但问题是如果符合其中一种以上的情形时应该如果确定是哪种呢？这就要确定四种规则的优先级了。优先级如下： new	绑定>显式绑定>隐式绑定>默认绑定。

###特殊情况

凡事有例外，this也一样。下面介绍下特殊情况。

#### 箭头函数

es6中引入的箭头函数虽然也叫函数，但是却不适用于上面的四规则。先引入一个🌰：

```javascript
function a(){
	return ()=>{
		console.log(this.name);
	}
}
const obj1={
	name: 11111,
}
const obj2={
	name: 22222,
}
var test = a.call(obj1);
test.call(obj2); // 11111
```

上面例子中箭头函数理论上绑定的是obj2，但是实际输出的却是11111。所以箭头函数是不适用于上面的四规则的。箭头函数的具体规则时：箭头函数this是在声明时就确定了，其this就是声明时所在作用域的this确定的。比如上面的例子，箭头函数是在a函数中声明的，所以箭头函数中所用的this就是a的this，而a中的this是根据调用位置和规则确定是obj1，所以箭头函数中this也是指向obj1。



##五、Call，Apply，Bind的用法

###call 和 apply 的共同点

它们的共同点是，都能够**改变函数执行时的上下文**，将一个对象的方法交给另一个对象来执行，并且是立即执行的。

为何要改变执行上下文？举一个生活中的小例子：平时没时间做饭的我，周末想给孩子炖个腌笃鲜尝尝。但是没有适合的锅，而我又不想出去买。所以就问邻居借了一个锅来用，这样既达到了目的，又节省了开支，一举两得。

改变执行上下文也是一样的，A 对象有一个方法，而 B 对象因为某种原因，也需要用到同样的方法，那么这时候我们是单独为 B 对象扩展一个方法呢，还是借用一下 A 对象的方法呢？当然是借用 A 对象的啦，既完成了需求，又减少了内存的占用。

另外，它们的写法也很类似，**调用 call 和 apply 的对象，必须是一个函数 Function**。接下来，就会说到具体的写法，那也是它们区别的主要体现。

###call 和 apply 的区别

它们的区别，主要体现在参数的写法上。先来看一下它们各自的具体写法。

#### call 的写法

```javascript
Function.call(obj,[param1[,param2[,…[,paramN]]]])
```

需要注意以下几点：

- 调用 call 的对象，必须是个函数 Function。
- call 的第一个参数，是一个对象。 Function 的调用者，将会指向这个对象。如果不传，则默认为全局对象 window。
- 第二个参数开始，可以接收任意个参数。每个参数会映射到相应位置的 Function 的参数上。但是如果将所有的参数作为数组传入，它们会作为一个整体映射到 Function 对应的第一个参数上，之后参数都为空。

```javascript
function func (a,b,c) {}

func.call(obj, 1,2,3)
// func 接收到的参数实际上是 1,2,3

func.call(obj, [1,2,3])
// func 接收到的参数实际上是 [1,2,3],undefined,undefined
```

#### apply 的写法

```javascript
Function.apply(obj[,argArray])
```

需要注意的是：

- 它的调用者必须是函数 Function，并且只接收两个参数，第一个参数的规则与 call 一致。
- 第二个参数，必须是数组或者类数组，它们会被转换成类数组，传入 Function 中，并且会被映射到 Function 对应的参数上。这也是 call 和 apply 之间，很重要的一个区别。

```javascript
func.apply(obj, [1,2,3])
// func 接收到的参数实际上是 1,2,3

func.apply(obj, {
    0: 1,
    1: 2,
    2: 3,
    length: 3
})
// func 接收到的参数实际上是 1,2,3
```

#### 什么是类数组？

先说数组，这我们都熟悉。它的特征有：可以通过角标调用，如 array[0]；具有长度属性length；可以通过 for 循环或forEach方法，进行遍历。

那么，类数组是什么呢？顾名思义，就是**具备与数组特征类似的对象**。比如，下面的这个对象，就是一个类数组。

```javascript
let arrayLike = {
    0: 1,
    1: 2,
    2: 3,
    length: 3
};
```

类数组 arrayLike 可以通过角标进行调用，具有length属性，同时也可以通过 for 循环进行遍历。

类数组，还是比较常用的，只是我们平时可能没注意到。比如，我们获取 DOM 节点的方法，返回的就是一个类数组。再比如，在一个方法中使用 arguments 获取到的所有参数，也是一个类数组。

但是需要注意的是：**类数组无法使用 forEach、splice、push 等数组原型链上的方法**，毕竟它不是真正的数组。

###call 和 apply 的用途

下面会分别列举 call 和 apply 的一些使用场景。声明：例子中没有哪个场景是必须用 call 或者必须用 apply 的，只是个人习惯这么用而已。

#### call 的使用场景

**1、对象的继承**。如下面这个例子：

```javascript
function superClass () {
    this.a = 1;
    this.print = function () {
        console.log(this.a);
    }
}

function subClass () {
    superClass.call(this);
    this.print();
}

subClass();
// 1
```

subClass 通过 call 方法，继承了 superClass 的 print 方法和 a 变量。此外，subClass 还可以扩展自己的其他方法。

**2、借用方法**。还记得刚才的类数组么？如果它想使用 Array 原型链上的方法，可以这样：

```javascript
let domNodes = Array.prototype.slice.call(document.getElementsByTagName("*"));
```

这样，domNodes 就可以应用 Array 下的所有方法了。

#### apply 的一些妙用

**1、Math.max**。用它来获取数组中最大的一项。

```javascript
let max = Math.max.apply(null, array);
```

同理，要获取数组中最小的一项，可以这样：

```
let min = Math.min.apply(null, array);
复制代码
```

**2、实现两个数组合并**。在 ES6 的扩展运算符出现之前，我们可以用 Array.prototype.push来实现。

```javascript
let arr1 = [1, 2, 3];
let arr2 = [4, 5, 6];

Array.prototype.push.apply(arr1, arr2);
console.log(arr1); // [1, 2, 3, 4, 5, 6]
```

###bind 的使用

最后来说说 bind。在 MDN 上的解释是：bind() 方法创建一个新的函数，在调用时设置 this 关键字为提供的值。并在调用新函数时，将给定参数列表作为原函数的参数序列的前若干项。

它的语法如下：

```
Function.bind(thisArg[, arg1[, arg2[, ...]]])
复制代码
```

bind 方法 与 apply 和 call 比较类似，也能改变函数体内的 this 指向。不同的是，**bind 方法的返回值是函数，并且需要稍后调用，才会执行**。而 apply 和 call 则是立即调用。

来看下面这个例子：

```javascript
function add (a, b) {
    return a + b;
}

function sub (a, b) {
    return a - b;
}

add.bind(sub, 5, 3); // 这时，并不会返回 8
add.bind(sub, 5, 3)(); // 调用后，返回 8
```

如果 bind 的第一个参数是 null 或者 undefined，this 就指向全局对象 window。

###总结

call 和 apply 的主要作用，是改变对象的执行上下文，并且是立即执行的。它们在参数上的写法略有区别。

bind 也能改变对象的执行上下文，它与 call 和 apply 不同的是，返回值是一个函数，并且需要稍后再调用一下，才会执行。

最后，分享一个在知乎上看到的，关于 call 和 apply 的便捷记忆法：

> 猫吃鱼，狗吃肉，奥特曼打小怪兽。
>
> 有天狗想吃鱼了
>
> 猫.吃鱼.call(狗，鱼)
>
> 狗就吃到鱼了
>
> 猫成精了，想打怪兽
>
> 奥特曼.打小怪兽.call(猫，小怪兽)
>
> 猫也可以打小怪兽了

## 六、算法

### 判断一个单词是否是回文

回文是指把相同的词汇或句子，在下文中调换位置或颠倒过来，产生首尾回环的情趣，叫做回文，也叫回环。比如 mamam redivider .

很多人拿到这样的题目非常容易想到用for 将字符串颠倒字母顺序然后匹配就行了。其实重要的考察的就是对于reverse的实现。其实我们可以利用现成的函数，将字符串转换成数组，这个思路很重要，我们可以拥有更多的自由度去进行字符串的一些操作。

```javascript
function checkPalindrom(str) {  
    return str == str.split('').reverse().join('');
}
```

### 去掉一组整型数组重复的值

比如 输入: [1,13,24,11,11,14,1,2]，  输出: [1,13,24,11,14,2] ，需要去掉重复的11 和 1 这两个元素。

主要考察个人对Object的使用，利用key来进行筛选。

```javascript
**
* unique an array 
**/
let unique = function(arr) {  
  let hashTable = {};
  let data = [];
  for(let i=0,l=arr.length;i<l;i++) {
    if(!hashTable[arr[i]]) {
      hashTable[arr[i]] = true;
      data.push(arr[i]);
    }
  }
  return data

}

module.exports = unique;  
```

###去掉非整型数组重复的值

假如数组并不是整型数组该如何去除重复呢？

假设有一个这样的数组： `let originalArray = [1, '1', '1', 2, true, 'true', false, false, null, null, {}, {}, 'abc', 'abc', undefined, undefined, NaN, NaN];`。后面的方法中的源数组，都是指的这个。

#### 1、ES6 的 Set 对象

ES6 提供了新的数据结构 Set。它类似于数组，但是成员的值都是唯一的，没有重复的值。Set 本身是一个构造函数，用来生成 Set 数据结构。

```javascript
let resultArr = Array.from(new Set(originalArray));

// 或者用扩展运算符
let resultArr = [...new Set(originalArray)];

console.log(resultArr);
// [1, "1", 2, true, "true", false, null, {…}, {…}, "abc", undefined, NaN]
```

Set 并不是真正的数组，这里的 `Array.from` 和 `...` 都可以将 Set 数据结构，转换成最终的结果数组。

这是最简单快捷的去重方法，但是细心的同学会发现，这里的 `{}` 没有去重。可是又转念一想，2 个空对象的地址并不相同，所以这里并没有问题，结果 ok。

#### 2、Map 的 has 方法

把源数组的每一个元素作为 key 存到 Map 中。由于 Map 中不会出现相同的 key 值，所以最终得到的就是去重后的结果。

```javascript
const resultArr = new Array();

for (let i = 0; i < originalArray.length; i++) {
    // 没有该 key 值
    if (!map.has(originalArray[i])) {
        map.set(originalArray[i], true);
        resultArr.push(originalArray[i]);
    }
}

console.log(resultArr);
// [1, "1", 2, true, "true", false, null, {…}, {…}, "abc", undefined, NaN]
```

但是它与 Set 的数据结构比较相似，结果 ok。

#### 3、indexOf 和 includes

建立一个新的空数组，遍历源数组，往这个空数组里塞值，每次 push 之前，先判断是否已有相同的值。

判断的方法有 2 个：indexOf 和 includes，但它们的结果之间有细微的差别。先看 indexOf。

```javascript
const resultArr = [];
for (let i = 0; i < originalArray.length; i++) {
    if (resultArr.indexOf(originalArray[i]) < 0) {
        resultArr.push(originalArray[i]);
    }
}
console.log(resultArr);
// [1, "1", 2, true, "true", false, null, {…}, {…}, "abc", undefined, NaN, NaN]
```

indexOf 并不没处理 `NaN`。

再来看 includes，它是在 ES7 中正式提出的。

```javascript
const resultArr = [];
for (let i = 0; i < originalArray.length; i++) {
    if (!resultArr.includes(originalArray[i])) {
        resultArr.push(originalArray[i]);
    }
}
console.log(resultArr);
// [1, "1", 2, true, "true", false, null, {…}, {…}, "abc", undefined, NaN]
```

includes 处理了 `NaN`，结果 ok。

#### 4、sort

先将原数组排序，生成新的数组，然后遍历排序后的数组，相邻的两两进行比较，如果不同则存入新数组。

```javascript
const sortedArr = originalArray.sort();

const resultArr = [sortedArr[0]];

for (let i = 1; i < sortedArr.length; i++) {
    if (sortedArr[i] !== resultArr[resultArr.length - 1]) {
        resultArr.push(sortedArr[i]);
    }
}
console.log(resultArr);
// [1, "1", 2, NaN, NaN, {…}, {…}, "abc", false, null, true, "true", undefined]
```

从结果可以看出，对源数组进行了排序。但同样的没有处理 `NaN`。

#### 5、双层 for 循环 + splice

双层循环，外层遍历源数组，内层从 i+1 开始遍历比较，相同时删除这个值。

```javascript
for (let i = 0; i < originalArray.length; i++) {
    for (let j = (i + 1); j < originalArray.length; j++) {
        // 第一个等于第二个，splice去掉第二个
        if (originalArray[i] === originalArray[j]) {
            originalArray.splice(j, 1);
            j--;
        }
    }
}

console.log(originalArray);
// [1, "1", 2, true, "true", false, null, {…}, {…}, "abc", undefined, NaN, NaN]
```

splice 方法会修改源数组，所以这里我们并没有新开空数组去存储，最终输出的是修改之后的源数组。但同样的没有处理 `NaN`。

#### 6、原始去重

定义一个新数组，并存放原数组的第一个元素，然后将源数组一一和新数组的元素对比，若不同则存放在新数组中。

```javascript
let resultArr = [originalArray[0]];
for(var i = 1; i < originalArray.length; i++){
    var repeat = false;
    for(var j=0; j < resultArr.length; j++){
        if(originalArray[i] === resultArr[j]){
            repeat = true;
            break;
        }
    }

    if(!repeat){
       resultArr.push(originalArray[i]);
    }
}
console.log(resultArr);
// [1, "1", 2, true, "true", false, null, {…}, {…}, "abc", undefined, NaN, NaN]
```

这是最原始的去重方法，很好理解，但写法繁琐。同样的没有处理 `NaN`。

#### 7、ES5 的 reduce

reduce 是 ES5 中方法，常用于值的累加。它的语法：

```javascript
arr.reduce(callback[, initialValue])
```

reduce 的第一个参数是一个 callback，callback 中的参数分别为： Accumulator(累加器)、currentValue(当前正在处理的元素)、currentIndex(当前正在处理的元素索引，可选)、array(调用 reduce 的数组，可选)。

reduce 的第二个参数，是作为第一次调用 callback 函数时的第一个参数的值。如果没有提供初始值，则将使用数组中的第一个元素。

利用 reduce 的特性，再结合之前的 includes(也可以用 indexOf)，就能得到新的去重方法：

```javascript
const reducer = (acc, cur) => acc.includes(cur) ? acc : [...acc, cur];

const resultArr = originalArray.reduce(reducer, []);

console.log(resultArr);
// [1, "1", 2, true, "true", false, null, {…}, {…}, "abc", undefined, NaN]
```

这里的 `[]` 就是初始值(initialValue)。acc 是累加器，在这里的作用是将没有重复的值塞入新数组（它一开始是空的）。 reduce 的写法很简单，但需要多加理解。它可以处理 `NaN`，结果 ok。

#### 8、对象的属性

每次取出原数组的元素，然后在对象中访问这个属性，如果存在就说明重复。

```javascript
const resultArr = [];
const obj = {};
for(let i = 0; i < originalArray.length; i++){
    if(!obj[originalArray[i]]){
        resultArr.push(originalArray[i]);
        obj[originalArray[i]] = 1;
    }
}
console.log(resultArr);
// [1, 2, true, false, null, {…}, "abc", undefined, NaN]
```

但这种方法有缺陷。从结果看，它貌似只关心值，不关注类型。还把 {} 给处理了，但这不是正统的处理办法，所以 **不推荐使用**。

#### 9、filter + hasOwnProperty

filter 方法会返回一个新的数组，新数组中的元素，通过 hasOwnProperty 来检查是否为符合条件的元素。

```javascript
const obj = {};
const resultArr = originalArray.filter(function (item) {
    return obj.hasOwnProperty(typeof item + item) ? false : (obj[typeof item + item] = true);
});

console.log(resultArr);
// [1, "1", 2, true, "true", false, null, {…}, "abc", undefined, NaN]
```

这 `貌似` 是目前看来最完美的解决方案了。这里稍加解释一下：

- hasOwnProperty 方法会返回一个布尔值，指示对象自身属性中是否具有指定的属性。
- `typeof item + item` 的写法，是为了保证值相同，但类型不同的元素被保留下来。例如：第一个元素为 number1，第二第三个元素都是 string1，所以第三个元素就被去除了。
- `obj[typeof item + item] = true` 如果 hasOwnProperty 没有找到该属性，则往 obj 里塞键值对进去，以此作为下次循环的判断依据。
- 如果 hasOwnProperty 没有检测到重复的属性，则告诉 filter 方法可以先积攒着，最后一起输出。

`看似` 完美解决了我们源数组的去重问题，但在实际的开发中，一般不会给两个空对象给我们去重。所以稍加改变源数组，给两个空对象中加入键值对。

```javascript
let originalArray = [1, '1', '1', 2, true, 'true', false, false, null, null, {a: 1}, {a: 2}, 'abc', 'abc', undefined, undefined, NaN, NaN];
```

然后再用 filter + hasOwnProperty 去重。

然而，结果竟然把 `{a: 2}` 给去除了！！！这就不对了。

所以，这种方法有点去重 `过头` 了，也是存在问题的。

#### 10、lodash 中的 _.uniq

灵机一动，让我想到了 lodash 的去重方法 _.uniq，那就尝试一把：

```javascript
console.log(_.uniq(originalArray));

// [1, "1", 2, true, "true", false, null, {…}, {…}, "abc", undefined, NaN]
```

用法很简单，可以在实际工作中正确处理去重问题。

然后，我在好奇心促使下，看了它的源码，指向了 baseUniq 文件，它的源码如下：

```javascript
function baseUniq(array, iteratee, comparator) {
  let index = -1
  let includes = arrayIncludes
  let isCommon = true

  const { length } = array
  const result = []
  let seen = result

  if (comparator) {
    isCommon = false
    includes = arrayIncludesWith
  }
  else if (length >= LARGE_ARRAY_SIZE) {
    const set = iteratee ? null : createSet(array)
    if (set) {
      return setToArray(set)
    }
    isCommon = false
    includes = cacheHas
    seen = new SetCache
  }
  else {
    seen = iteratee ? [] : result
  }
  outer:
  while (++index < length) {
    let value = array[index]
    const computed = iteratee ? iteratee(value) : value

    value = (comparator || value !== 0) ? value : 0
    if (isCommon && computed === computed) {
      let seenIndex = seen.length
      while (seenIndex--) {
        if (seen[seenIndex] === computed) {
          continue outer
        }
      }
      if (iteratee) {
        seen.push(computed)
      }
      result.push(value)
    }
    else if (!includes(seen, computed, comparator)) {
      if (seen !== result) {
        seen.push(computed)
      }
      result.push(value)
    }
  }
  return result
}
```

有比较多的干扰项，那是为了兼容另外两个方法，_.uniqBy 和 _.uniqWith。去除掉之后，就会更容易发现它是用 while 做了循环。当遇到相同的值得时候，continue outer 再次进入循环进行比较，将没有重复的值塞进 result 里，最终输出。

另外，_.uniqBy 方法可以通过指定 key，来专门去重对象列表。

```javascript
_.uniqBy([{ 'x': 1 }, { 'x': 2 }, { 'x': 1 }], 'x');
// => [{ 'x': 1 }, { 'x': 2 }]
```

_.uniqWith 方法可以完全地给对象中所有的键值对，进行比较。

```javascript
var objects = [{ 'x': 1, 'y': 2 }, { 'x': 2, 'y': 1 }, { 'x': 1, 'y': 2 }];

_.uniqWith(objects, _.isEqual);
// => [{ 'x': 1, 'y': 2 }, { 'x': 2, 'y': 1 }]
```

这两个方法，都还挺实用的。





### 统计一个字符串出现最多的字符

给出一段英文连续的英文字符窜，找出重复出现次数最多的字母

**比如：** 输入：afjghdfraaaasdenas  输出 ： a

前面出现过去重的算法，这里需要是统计重复次数。

```javascript
function findMaxDuplicateChar(str) {  
  if(str.length == 1) {
    return str;
  }
  let charObj = {};
  for(let i=0;i<str.length;i++) {
    if(!charObj[str.charAt(i)]) {
      charObj[str.charAt(i)] = 1;
    }else{
      charObj[str.charAt(i)] += 1;
    }
  }
  let maxChar = '',
      maxValue = 1;
  for(var k in charObj) {
    if(charObj[k] >= maxValue) {
      maxChar = k;
      maxValue = charObj[k];
    }
  }
  return maxChar;

}

module.exports = findMaxDuplicateChar;  
```

### 排序算法

#### 1.冒泡排序

如果说到算法题目的话，应该大多都是比较开放的题目，不限定算法的实现，但是一定要求掌握其中的几种，所以冒泡排序，这种较为基础并且便于理解记忆的算法一定需要熟记于心。冒泡排序算法就是依次比较大小，小的的大的进行位置上的交换。

```javascript
function bubbleSort(arr) {  
    for(let i = 0,l=arr.length;i<l-1;i++) {
        for(let j = i+1;j<l;j++) { 
          if(arr[i]>arr[j]) {
                let tem = arr[i];
                arr[i] = arr[j];
                arr[j] = tem;
            }
        }
    }
    return arr;
}
module.exports = bubbleSort;  
```

#### 2.选择排序

算法参考某个元素值，将小于它的值，放到左数组中，大于它的值的元素就放到右数组中，然后递归进行上一次左右数组的操作，返回合并的数组就是已经排好顺序的数组了。

```javascript
function quickSort(arr) {

    if(arr.length<=1) {
        return arr;
    }

    let leftArr = [];
    let rightArr = [];
    let q = arr[0];
    for(let i = 1,l=arr.length; i<l; i++) {
        if(arr[i]>q) {
            rightArr.push(arr[i]);
        }else{
            leftArr.push(arr[i]);
        }
    }

    return [].concat(quickSort(leftArr),[q],quickSort(rightArr));
}

module.exports = quickSort; 
```

[HTML5 Canvas Demo: Sorting Algorithms](http://math.hws.edu/eck/jsdemo/sortlab.html)，用动画演示算法。

####3.插入排序（Insertion Sort）

#####算法描述

插入排序（Insertion-Sort）的算法描述是一种简单直观的排序算法。它的工作原理是通过构建有序序列，对于未排序数据，在已排序序列中**从后向前扫描**，找到相应位置并插入。插入排序在实现上，通常采用in-place排序（即只需用到O(1)的额外空间的排序），因而在从后向前扫描过程中，需要反复把已排序元素逐步向后挪位，为最新元素提供插入空间。

插入排序核心--扑克牌思想： 就想着自己在打扑克牌，接起来第一张，放哪里无所谓，再接起来一张，比第一张小，放左边，继续接，可能是中间数，就插在中间.后面起的牌从后向前依次比较,并插入.



![插入排序](https://user-gold-cdn.xitu.io/2019/2/24/1691df9f6afc4b04?imageslim)

#####算法步骤及实现代码

**思路:** 插入排序也属于基本排序算法,**大致思路也是两层循环嵌套**.首先,按照其扑克牌的思路.将要排序的数列分为两部分.左边为有序数列(起在手中的牌),刚开始为空.右边部分为待排序的数列(即乱序的扑克牌).

有了上面大致思想后,开始设置循环.首先外循环为你需要起多少张牌.那是多少?毫无疑问就是数列的长度,但是为了方便,我们可以默认让数列第一个数作为有序数列,可以减少一次循环.故外循环次数为数列长度减1;内循环则循环有序数列,并从右往左,比较大小,将较小数插在前面(结合动图)

**步骤:**

- 1.从第一个元素开始，该元素可以认为已经被排序；
- 2.取出下一个元素，在已经排序的元素序列中从后向前扫描；
- 3.如果该元素（已排序）大于新元素，将该元素移到下一位置；
- 4.重复步骤3，直到找到已排序的元素小于或者等于新元素的位置；
- 5.将新元素插入到该位置后；
- 6.重复步骤2~5。

**JS代码实现:**

```
function insertSort(arr) {
    for(let i = 1; i < arr.length; i++) {  //外循环从1开始，默认arr[0]是有序段
        for(let j = i; j > 0; j--) {  //j = i,表示此时你起在手上的那张牌,将arr[j]依次比较插入有序段中
            if(arr[j] < arr[j-1]) {
                [arr[j],arr[j-1]] = [arr[j-1],arr[j]];  //其实这里内循环中,只要比它前一个数小就交换,直到没有更小的,就break退出.这和动图表示的插入还是有点区别的,但最后结果其实是一样的.
            } else {
                break;
            }
        }
    }
    return arr;
}
var arr=[3,44,38,5,47,15,36,26,27,2,46,4,19,50,48];
console.log(insertSort(arr));//[2, 3, 4, 5, 15, 19, 26, 27, 36, 38, 44, 46, 47, 48, 50]
复制代码
```

####4.快速排序（Quick Sort）

#####算法描述

快速排序的名字起的是简单粗暴，因为一听到这个名字你就知道它存在的意义，就是快，而且效率高! 它是处理**大数据**最快的排序算法之一了。

它是在冒泡排序基础上的递归分治法。通过递归的方式将数据依次分解为包含较小元素和较大元素的不同子序列。该算法不断重复这个步骤直至所有数据都是有序的。

**注意:** 快速排序也是面试是**最最最容易考到的算法题**,经常就会让你进行手写.



![快速排序](https://user-gold-cdn.xitu.io/2019/2/24/1691dfaa8bbf0e52?imageslim)

#####算法步骤及实现代码

**思路:** 快速排序属于高级排序算法,此时就不是相似的循环嵌套.它的大概思想就是: 找到一个数作为参考，比这个数字大的放在数字左边，比它小的放在右边； 然后分别再对左边和右变的序列做相同的操作(递归).

**注意:** **涉及到递归的算法,一定要记得设置出口,跳出递归!**

**步骤:**

- 1.从数列中挑出一个元素，称为 “基准”（pivot）;
- 2.重新排序数列，所有元素比基准值小的摆放在基准前面，所有元素比基准值大的摆在基准的后面（相同的数可以到任一边）。在这个分区退出之后，该基准就处于数列的中间位置。这个称为分区（partition）操作；
- 3.递归地（recursive）把小于基准值元素的子数列和大于基准值元素的子数列排序；

**JS代码实现:**

```javascript
function quickSort (arr) {
	if(arr.length <= 1) {
        return arr;  //递归出口
    }
	let left = [],
        right = [],
		//这里我们默认选择数组第一个为基准,PS:其实这样偷懒是不好的,如果数组是已经排好序了的.则很有可能变成最差情况的时间复杂度
		//pivotIndex = Math.floor(arr.length / 2),
	    pivot = arr[0];    //阮一峰版:  arr.splice(pivotIndex, 1)[0];   使用splice在大量数据时,会消耗大量内存;但也不至于被喷得一无是处! 它的思路是没有任何问题的! 
	for (var i = 1; i < arr.length; i++) {
		if (arr[i] < pivot) {
			left.push(arr[i])
		} else {
			right.push(arr[i])
		}
	}
	//concat也不适合大量数据的排序,会消耗大量内存
	return quickSort(left).concat(pivot, quickSort(right))
}
var arr=[3,44,38,5,47,15,36,26,27,2,46,4,19,50,48];
console.log(quickSort(arr));//[2, 3, 4, 5, 15, 19, 26, 27, 36, 38, 44, 46, 47, 48, 50]

//改进版:
function partition2(arr, low, high) {
  let pivot = arr[low];
  while (low < high) {
    while (low < high && arr[high] > pivot) {
      --high;
    }
    arr[low] = arr[high];
    while (low < high && arr[low] <= pivot) {
      ++low;
    }
    arr[high] = arr[low];
  }
  arr[low] = pivot;
  return low;
}

function quickSort2(arr, low, high) {
  if (low < high) {
    let pivot = partition2(arr, low, high);
    quickSort2(arr, low, pivot - 1);
    quickSort2(arr, pivot + 1, high);
  }
  return arr;
}
var arr=[3,44,38,5,47,15,36,26,27,2,46,4,19,50,48];
console.log(quickSort2(arr,0,arr.length-1));//[2, 3, 4, 5, 15, 19, 26, 27, 36, 38, 44, 46, 47, 48, 50]
```

####5.希尔排序（Shell Sort）

#####算法描述

1959年Shell发明； 第一个突破O(n^2)的排序算法；**是简单插入排序的改进版；它与插入排序的不同之处在于，它会优先比较距离较远的元素。** 希尔排序又叫缩小增量排序.并且排序也是不稳定的

希尔排序是基于插入排序的以下两点性质而提出改进方法的：

- 插入排序在对几乎已经排好序的数据操作时，效率高，即可以达到线性排序的效率；
- 但插入排序一般来说是低效的，因为插入排序每次只能将数据移动一位；

希尔排序的基本思想是：先将整个待排序的记录序列分割成为若干子序列分别进行直接插入排序，待整个序列中的记录“基本有序”时，再对全体记录进行依次直接插入排序。

#####算法步骤及实现代码

**思路:**  希尔排序其实大体思路很简单,就是将数组(长度为len)分成间隔为t1的若干数组.进行插入排序;排完后,将数组再分成间隔为t2(逐步减小)的若干数组,进行插入排序;然后继续上述操作,直到分成间隔为1的数组,再进行最后一次插入排序则完成.

方便理解可以查看下图:



![img](https://user-gold-cdn.xitu.io/2019/2/24/1691dfc27c806ac3?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



**步骤:**

- 1,选择一个增量序列t1，t2，…，tk，其中ti>tj，tk=1；
- 2,按增量序列个数k，对序列进行k 趟排序；
- 3,每趟排序，根据对应的增量ti，将待排序列分割成若干长度为m 的子序列，分别对各子表进行直接插入排序。仅增量因子为1 时，整个序列作为一个表来处理，表长度即为整个序列的长度。

**JS代码实现:**

```javascript
function shellSort(arr) {
    var len = arr.length,
        temp,
        gap = 1;
    while(gap < len/5) {          //动态定义间隔序列
        gap =gap*5+1;
    }
    for (gap; gap > 0; gap = Math.floor(gap/5)) {
        for (var i = gap; i < len; i++) {
            temp = arr[i];
            for (var j = i-gap; j >= 0 && arr[j] > temp; j-=gap) {
                arr[j+gap] = arr[j];
            }
            arr[j+gap] = temp;
        }
    }
    return arr;
}
var arr=[3,44,38,5,47,15,36,26,27,2,46,4,19,50,48];
console.log(shellSort(arr));//[2, 3, 4, 5, 15, 19, 26, 27, 36, 38, 44, 46, 47, 48, 50]
```

####6.归并排序（Merge Sort）

#####算法描述

归并排序（Merge sort）是建立在归并操作上的一种有效的排序算法。该算法是**采用分治法**（Divide and Conquer）的一个非常典型的应用。

归并排序是一种稳定的排序方法。将已有序的子序列合并，得到完全有序的序列；即先使每个子序列有序，再使子序列段间有序。若将两个有序表合并成一个有序表，称为2-路归并。

#####算法步骤及实现代码

**思路:** 将数组分为左和右两部分,然后继续将左右两部分继续(递归)拆分,直到拆分成单个为止;然后将拆分为最小的两个数组,进行比较,合并排成一个数组.接着继续递归比较合并.直到最后合并为一个数组.

**步骤:**

- 1.把长度为n的输入序列分成两个长度为n/2的子序列；
- 2.对这两个子序列分别采用归并排序；
- 3.将两个排序好的子序列合并成一个最终的排序序列。

**JS代码实现:**

```javascript
function mergeSort(arr) {  //采用自上而下的递归方法
    var len = arr.length;
    if(len < 2) {
        return arr;
    }
    var middle = Math.floor(len / 2),
        left = arr.slice(0, middle),
        right = arr.slice(middle);
    return merge(mergeSort(left), mergeSort(right));
}

function merge(left, right){
    var result = [];
    while (left.length && right.length) {
        if (left[0] <= right[0]) {
            result.push(left.shift());
        } else {
            result.push(right.shift());
        }
    }

    while (left.length)
        result.push(left.shift());

    while (right.length)
        result.push(right.shift());
    return result;
}
var arr=[3,44,38,5,47,15,36,26,27,2,46,4,19,50,48];
console.log(mergeSort(arr));//[2, 3, 4, 5, 15, 19, 26, 27, 36, 38, 44, 46, 47, 48, 50]
```

####7.堆排序（Heap Sort）

#####算法描述

堆排序（Heapsort）是指利用堆这种数据结构所设计的一种排序算法。堆积是一个近似完全二叉树的结构，并同时满足堆积的性质：即子结点的键值或索引总是小于（或者大于）它的父节点。堆排序可以说是一种利用堆的概念来排序的选择排序。分为两种方法：

- 大顶堆：每个节点的值都大于或等于其子节点的值，在堆排序算法中用于升序排列；
- 小顶堆：每个节点的值都小于或等于其子节点的值，在堆排序算法中用于降序排列；



![归并排序](https://user-gold-cdn.xitu.io/2019/2/24/1691dfd711a56f14?imageslim)



**步骤:**

- 1.将初始待排序关键字序列(R1,R2....Rn)构建成大顶堆，此堆为初始的无序区；
- 2.将堆顶元素R[1]与最后一个元素R[n]交换，此时得到新的无序区(R1,R2,......Rn-1)和新的有序区(Rn),且满足R[1,2...n-1]<=R[n]；
- 3.由于交换后新的堆顶R[1]可能违反堆的性质，因此需要对当前无序区(R1,R2,......Rn-1)调整为新堆，然后再次将R[1]与无序区最后一个元素交换，得到新的无序区(R1,R2....Rn-2)和新的有序区(Rn-1,Rn)。不断重复此过程直到有序区的元素个数为n-1，则整个排序过程完成。

**JS代码实现:**

```javascript
function buildMaxHeap(arr,len) {   // 建立大顶堆
    
    for (var i = Math.floor(len/2); i >= 0; i--) {
        heapify(arr, i,len);
    }
}

function heapify(arr, i,len) {     // 堆调整
    var left = 2 * i + 1,
        right = 2 * i + 2,
        largest = i;

    if (left < len && arr[left] > arr[largest]) {
        largest = left;
    }

    if (right < len && arr[right] > arr[largest]) {
        largest = right;
    }

    if (largest != i) {
        swap(arr, i, largest);
        heapify(arr, largest,len);
    }
}

function swap(arr, i, j) {
    var temp = arr[i];
    arr[i] = arr[j];
    arr[j] = temp;
}

function heapSort(arr) {
    var len = arr.length;
    buildMaxHeap(arr,len);

    for (var i = arr.length-1; i > 0; i--) {
        swap(arr, 0, i);
        len--;
        heapify(arr, 0,len);
    }
    return arr;
}
var arr=[3,44,38,5,47,15,36,26,27,2,46,4,19,50,48];
console.log(heapSort(arr));//[2, 3, 4, 5, 15, 19, 26, 27, 36, 38, 44, 46, 47, 48, 50]
```

####8.计数排序（Counting Sort）

#####算法描述

计数排序几乎是唯一一个不基于比较的排序算法, 该算法于1954年由 Harold H. Seward 提出. 使用它处理一定范围内的整数排序时, 时间复杂度为O(n+k), 其中k是整数的范围, 它几乎比任何基于比较的排序算法都要快`( 只有当O(k)>O(n*log(n))的时候其效率反而不如基于比较的排序, 如归并排序和堆排序)`.

计数排序的核心在于将输入的数据值转化为键存储在**额外开辟的数组空间中。**



![计数排序](https://user-gold-cdn.xitu.io/2019/2/24/1691dfde0eaa754a?imageslim)

#####算法步骤及实现代码

**思路:** 计数排序利用了一个特性, 对于数组的某个元素, 一旦知道了有多少个其它元素比它小(假设为m个), 那么就可以确定出该元素的正确位置(第m+1位)

**步骤:**

- 1, 获取待排序数组A的最大值, 最小值.
- 2, 将最大值与最小值的差值+1作为长度新建计数数组B，并将相同元素的数量作为值存入计数数组.
- 3, 对计数数组B累加计数, 存储不同值的初始下标.
- 4, 从原数组A挨个取值, 赋值给一个新的数组C相应的下标, 最终返回数组C.

注意: **如果原数组A是包含若干个对象的数组，需要基于对象的某个属性进行排序，那么算法开始时，需要将原数组A处理为一个只包含对象属性值的简单数组simpleA, 接下来便基于simpleA进行计数、累加计数, 其它同上.**

**JS代码实现:**

```javascript
//以下实现不仅支持了数值序列的排序，还支持根据对象的某个属性值来排序。
function countSort(array, keyName){
  var length = array.length,
      output = new Array(length),
      max,
      min,
      simpleArray = keyName ? array.map(function(v){
        return v[keyName];
      }) : array; // 如果keyName是存在的，那么就创建一个只有keyValue的简单数组

  // 获取最大最小值
  max = min = simpleArray[0];
  simpleArray.forEach(function(v){
    v > max && (max = v);
    v < min && (min = v);
  });
  // 获取计数数组的长度
  var k = max - min + 1;
  // 新建并初始化计数数组
  var countArray = new Array(k);
  simpleArray.forEach(function(v){
    countArray[v - min]= (countArray[v - min] || 0) + 1;
  });
  // 累加计数，存储不同值的初始下标
  countArray.reduce(function(prev, current, i, arr){
    arr[i] = prev;
    return prev + current;
  }, 0);
  // 从原数组挨个取值(因取的是原数组的相应值，只能通过遍历原数组来实现)
  simpleArray.forEach(function(v, i){
    var j = countArray[v - min]++;
    output[j] = array[i];
  });
  return output;
}
var arr=[3,44,38,5,47,15,36,26,27,2,46,4,19,50,48];
console.log(countSort(arr));//[2, 3, 4, 5, 15, 19, 26, 27, 36, 38, 44, 46, 47, 48, 50]
```

####9.桶排序（Bucket Sort）

#####算法描述

桶排序是计数排序的升级版。它利用了函数的映射关系，高效与否的关键就在于这个映射函数的确定。为了使桶排序更加高效，我们需要做到这两点：

- 在额外空间充足的情况下，尽量增大桶的数量
- 使用的映射函数能够将输入的 N 个数据均匀的分配到 K 个桶中

同时，对于桶中元素的排序，选择何种比较排序算法对于性能的影响至关重要。很显然，桶划分的越小，各个桶之间的数据越少，排序所用的时间也会越少。但相应的空间消耗就会增大。



![桶排序](https://user-gold-cdn.xitu.io/2019/2/24/1691dfe4be472342?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

#####算法步骤及实现代码

**思路:** 桶排序 (Bucket sort)的工作的原理：假设输入数据服从均匀分布，将数据分到有限数量的桶里，每个桶再分别排序（有可能再使用别的排序算法或是以递归方式继续使用桶排序进行排

**步骤:**

- 1.设置一个定量的数组当作空桶；
- 2.遍历输入数据，并且把数据一个一个放到对应的桶里去；
- 3.对每个不是空的桶进行排序；
- 4.从不是空的桶里把排好序的数据拼接起来。

注意: **如果原数组A是包含若干个对象的数组，需要基于对象的某个属性进行排序，那么算法开始时，需要将原数组A处理为一个只包含对象属性值的简单数组simpleA, 接下来便基于simpleA进行计数、累加计数, 其它同上.**

**JS代码实现:**

```javascript
function bucketSort(arr, bucketSize) {
    if (arr.length === 0) {
      return arr;
    }

    var i;
    var minValue = arr[0];
    var maxValue = arr[0];
    for (i = 1; i < arr.length; i++) {
      if (arr[i] < minValue) {
          minValue = arr[i];                // 输入数据的最小值
      } else if (arr[i] > maxValue) {
          maxValue = arr[i];                // 输入数据的最大值
      }
    }

    //桶的初始化
    var DEFAULT_BUCKET_SIZE = 5;            // 设置桶的默认数量为5
    bucketSize = bucketSize || DEFAULT_BUCKET_SIZE;
    var bucketCount = Math.floor((maxValue - minValue) / bucketSize) + 1;   
    var buckets = new Array(bucketCount);
    for (i = 0; i < buckets.length; i++) {
        buckets[i] = [];
    }

    //利用映射函数将数据分配到各个桶中
    for (i = 0; i < arr.length; i++) {
        buckets[Math.floor((arr[i] - minValue) / bucketSize)].push(arr[i]);
    }

    arr.length = 0;
    for (i = 0; i < buckets.length; i++) {
        insertionSort(buckets[i]);                      // 对每个桶进行排序，这里使用了插入排序
        for (var j = 0; j < buckets[i].length; j++) {
            arr.push(buckets[i][j]);                      
        }
    }

    return arr;
}
var arr=[3,44,38,5,47,15,36,26,27,2,46,4,19,50,48];
console.log(bucketSort(arr));//[2, 3, 4, 5, 15, 19, 26, 27, 36, 38, 44, 46, 47, 48, 50]
```

####10.基数排序（Radix Sort）

#####算法描述

基数排序是一种非比较型整数排序算法，其原理是将整数按位数切割成不同的数字，然后按每个位数分别比较。由于整数也可以表达字符串（比如名字或日期）和特定格式的浮点数，所以基数排序也不是只能使用于整数。

按照优先从高位或低位来排序有两种实现方案:

- MSD: 由高位为基底, 先按k1排序分组, 同一组中记录, 关键码k1相等, 再对各组按k2排序分成子组, 之后, 对后面的关键码继续这样的排序分组, 直到按最次位关键码kd对各子组排序后. 再将各组连接起来, 便得到一个有序序列. MSD方式适用于位数多的序列.
- LSD: 由低位为基底, 先从kd开始排序，再对kd-1进行排序，依次重复，直到对k1排序后便得到一个有序序列. LSD方式适用于位数少的序列.

基数排序,计数排序,桶排序.这三种排序算法都利用了桶的概念，但对桶的使用方法上有明显差异：

- 基数排序：根据键值的每位数字来分配桶；
- 计数排序：每个桶只存储单一键值；
- 桶排序：每个桶存储一定范围的数值；



![基数排序](https://user-gold-cdn.xitu.io/2019/2/24/1691dfecee7659e2?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

#####算法步骤及实现代码

**步骤:**

- 1.取得数组中的最大数，并取得位数；
- 2.arr为原始数组，从最低位开始取每个位组成radix数组；
- 3.对radix进行计数排序（利用计数排序适用于小范围数的特点）；

**JS代码实现:**

```javascript
/**
 * 基数排序适用于：
 *  (1)数据范围较小，建议在小于1000
 *  (2)每个数值都要大于等于0
 * @author xiazdong
 * @param  arr 待排序数组
 * @param  maxDigit 最大位数
 */
//LSD Radix Sort

function radixSort(arr, maxDigit) {
    var mod = 10;
    var dev = 1;
    var counter = [];
    console.time('基数排序耗时');
    for (var i = 0; i < maxDigit; i++, dev *= 10, mod *= 10) {
        for(var j = 0; j < arr.length; j++) {
            var bucket = parseInt((arr[j] % mod) / dev);
            if(counter[bucket]== null) {
                counter[bucket] = [];
            }
            counter[bucket].push(arr[j]);
        }
        var pos = 0;
        for(var j = 0; j < counter.length; j++) {
            var value = null;
            if(counter[j]!=null) {
                while ((value = counter[j].shift()) != null) {
                      arr[pos++] = value;
                }
          }
        }
    }
    console.timeEnd('基数排序耗时');
    return arr;
}
var arr = [3, 44, 38, 5, 47, 15, 36, 26, 27, 2, 46, 4, 19, 50, 48];
console.log(radixSort(arr,2)); //[2, 3, 4, 5, 15, 19, 26, 27, 36, 38, 44, 46, 47, 48, 50]
```

####11.二叉树和二叉查找树

#####**树**

树是一种非顺序数据结构，一种分层数据的抽象模型，它对于存储需要快速查找的数据非常有用。

现实生活中最常见的树的例子是家谱，或是公司的组织架构图.

一个树结构包含一系列存在父子关系的节点。每个节点都有一个父节点（除了顶部的第一个 节点）以及零个或多个子节点.

**树常见结构/属性：**

- 节点 
  - 根节点
  - 内部节点：非根节点、且有子节点的节点
  - 外部节点/页节点：无子节点的节点
- 子树：就是大大小小节点组成的树
- 深度：节点到根节点的节点数量
- 高度：树的高度取决于所有节点深度中的最大值
- 层级：也可以按照节点级别来分层

#####**二叉树**

二叉树，是一种特殊的树，即子节点最多只有两个，这个限制可以使得写出高效的插入、删除、和查找数据。在二叉树中，子节点分别叫左节点和右节点。



![二叉树](https://user-gold-cdn.xitu.io/2019/2/24/1691e0043526d05f?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

#####**二叉查找树**

二叉查找树是一种特殊的二叉树，**相对较小的值保存在左节点中，较大的值（或者等于）保存在右节点中，这一特性使得查找的效率很高，对于数值型和非数值型数据，比如字母和字符串，都是如此。** 现在通过JS实现一个二叉查找树。

**节点:**

二叉树的最小元素是节点，所以先定义一个节点

```javascript
function Node(data,left,right) {
    this.left = left;
    this.right = right;
    this.data = data;
    this.show = () => {return this.data}
}
```

这个就是二叉树的最小结构单元

**二叉树**

```javascript
function BST() {
    this.root = null //初始化,root为null
}
```

BST初始化时，只有一个根节点，且没有任何数据。 接下来，我们利用二叉查找树的规则，定义一个插入方法，这个方法的基本思想是:

1. 如果`BST.root === null` ，那么就将节点作为根节点
2. 如果`BST.root !==null` ，将插入节点进行一个比较，小于根节点，拿到左边的节点，否则拿右边，再次比较、递归。

这里就出现了递归了，因为，总是要把较小的放在靠左的分支。换言之

**最左变的叶子节点是最小的数，最右的叶子节点是最大的数**

```javascript
function insert(data) {
    var node = new Node(data,null,null);
    if(this.root === null) {
        this.root = node
    } else {
        var current = this.root;
        var parent;
        while(true) {
            parent = current;
            if(data < current.data) {
                current = current.left; //到左子树
                if(current === null) {  //如果左子树为空，说明可以将node插入在这里
                    parent.left = node;
                    break;  //跳出while循环
                }
            } else {
                current = current.right;
                if(current === null) {
                    parent.right = node;
                    break;
                }
            }
        }
    }
}
```

这里，是使用了一个循环方法，不断的去向子树寻找正确的位置。 循环和递归都有一个核心，就是找到出口，这里的出口就是当current 为null的时候，代表没有内容，可以插入。

接下来，将此方法写入BST即可:

```javascript
function BST() {
    this.root = null;
    this.insert = insert;
}
```

这样子，就可以使用二叉树这个自建的数据结构了:

```javascript
var bst = new BST()；
bst.insert(10);
bst.insert(8);
bst.insert(2);
bst.insert(7);
bst.insert(5);
```

但是这个时候，想要看树中的数据，不是那么清晰，所以接下来，就要用到遍历了。

**树的遍历:**

按照根节点访问的顺序不同，树的遍历分为以下三种：

- 前序遍历 (根节点->左子树->右子树)
- 中序遍历 (左子树->根节点->右子树)
- 后序遍历 (左子树->右子树->根节点)

**先序遍历:**

先序遍历是以优先于后代节点的顺序访问每个节点的。先序遍历的一种应用是打印一个结构化的文档。



![先序遍历](https://user-gold-cdn.xitu.io/2019/2/24/1691e00c3d7c8b47?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



```javascript
function preOrder(node) {
    if(node !== null) {
        //根节点->左子树->右子树
        console.log(node.show());
        preOrder(node.left);
        preOrder(node.right);
    }
}
```

**中序遍历:**

中序遍历是以从最小到最大的顺序访 问所有节点。中序遍历的一种应用就是对树进行排序操作。



![中序遍历](https://user-gold-cdn.xitu.io/2019/2/24/1691e011441b61ab?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



```javascript
function inOrder(node) {
    if(node !== null) {
        //如果不是null，就一直查找左变，因此递归
		//左子树->根节点->右子树
        inOrder(node.left);
        //递归结束，打印当前值
        console.log(node.show());
        //上一次递归已经把左边搞完了，右边
        inOrder(node.right);
    }
}
```

**后序遍历:**

后序遍历则是先访问节点的后代节点，再访问节点本身。后序遍历的一种应用是计算一个目录和它的子目录中所有文件所占空间的大小。



![后序遍历](https://user-gold-cdn.xitu.io/2019/2/24/1691e0164e731d33?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



```javascript
function postOrder(node) {
    if(node !== null) {
        //左子树->右子树->根节点
        postOrder(node.left);
        postOrder(node.right);
        console.log(node.show())
    }
}
```

#####二叉树的查找

在二叉树这种数据结构中进行数据查找是最方便的：

- 最小值： 最左子树的叶子节点
- 最大值： 最右子树的叶子节点
- 特定值： target与current进行比较，如果比current大，在current.right进行查找，反之类似。

清楚思路后，就动手来写：

```javascript
//最小值
function getMin(bst) {
    var current = bst.root;
    while(current.left !== null) {
        current = current.left;
    }
    return current.data;
}

//最大值
function getMax(bst) {
    var current = bst.root;
    while(current.right !== null) {
        current = current.right;
    }
    return current.data;
}
```

最大、最小值都是非常简单的，下面主要看下如何通过

```javascript
function find(target,bst) {
    var current = bst.root;
    while(current !== null) {
        if(target === current.data) {
            return true;
        }
        else if(target > current.data) {
            current = current.right;
        } else if(target < current.data) {
            current = current.left;
        }
    }
    return -1;
}
```

其实核心，仍然是通过一个循环和判断，来不断的向下去寻找，这里的思想其实和二分查找是有点类似的。

#####AVL树：

AVL树是一种自平衡二叉搜索树，AVL树本质上是带了平衡功能的二叉查找树（二叉排序树，二叉搜索树），在AVL树中任何节点的两个子树的高度最大差别为一，也就是说这种树会在添加或移除节点时尽量试着成为一棵完全树，所以它也被称为高度平衡树。查找、插入和删除在平均和最坏情况下都是 `O（log n）`，增加和删除可能需要通过一次或多次树旋转来重新平衡这个树。

#####红黑树：

红黑树和AVL树类似，都是在进行插入和删除操作时通过特定操作保持二叉查找树的平衡，从而获得较高的查找性能；它虽然是复杂的，但它的最坏情况运行时间也是非常良好的，并且在实践中是高效的：它可以在`O(log n)`时间内做查找，插入和删除，这里的 n 是树中元素的数目。

红黑树是每个节点都带有颜色属性的二叉查找树，颜色或红色或黑色。在二叉查找树强制一般要求以外，对于任何有效的红黑树我们增加了如下的额外要求：

- 节点是红色或黑色
- 根节点是黑色
- 每个叶节点（NIL节点，空节点）是黑色的
- 每个红色节点的两个子节点都是黑色。(从每个叶子到根的所有路径上不能有两个连续的红色节点)
- 从任一节点到其每个叶子的所有路径都包含相同数目的黑色节点

这些约束强制了红黑树的关键性质：从根到叶子的最长的可能路径不多于最短的可能路径的两倍长。结果是这个树大致上是平衡的。因为操作比如插入、删除和查找某个值的最坏情况时间都要求与树的高度成比例，这个在高度上的理论上限允许红黑树在最坏情况下都是高效的，而不同于普通的二叉查找树。

红黑树和AVL树一样都对插入时间、删除时间和查找时间提供了最好可能的最坏情况担保。这不只是使它们在时间敏感的应用如即时应用(real time application)中有价值，而且使它们有在提供最坏情况担保的其他数据结构中作为建造板块的价值；例如，在计算几何中使用的很多数据结构都可以基于红黑树。 红黑树在函数式编程中也特别有用，在这里它们是最常用的持久数据结构之一，它们用来构造关联数组和集合，在突变之后它们能保持为以前的版本。除了`O(log n)`的时间之外，红黑树的持久版本对每次插入或删除需要`O(log n)`的空间。

### 不借助临时变量，进行两个整数的交换

**举例：**输入 a = 2, b = 4 输出 a = 4, b =2

　　这种问题非常巧妙，需要大家跳出惯有的思维，利用 a , b进行置换。

　　主要是利用 + - 去进行运算，类似 a = a + ( b - a) 实际上等同于最后 的 a = b;

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```javascript
function swap(a , b) {  
  b = b - a;
  a = a + b;
  b = a - b;
  return [a,b];
}

module.exports = swap;  
```

### 使用canvas绘制一个有限度的斐波那契数列的曲线

![斐波那契数列曲线](https://images2015.cnblogs.com/blog/738658/201707/738658-20170719222015693-961565046.png)

数列长度限定在9.

`斐波那契数列`，又称黄金分割数列，指的是这样一个数列：0、1、1、2、3、5、8、13、21、34、……在数学上，斐波纳契数列主要考察递归的调用。我们一般都知道定义

```
fibo[i] = fibo[i-1]+fibo[i-2];  
```

生成斐波那契数组的方法

```javascript
function getFibonacci(n) {  
  var fibarr = [];
  var i = 0;
  while(i<n) {
    if(i<=1) {
      fibarr.push(i);
    }else{
      fibarr.push(fibarr[i-1] + fibarr[i-2])
    }
    i++;
  }

  return fibarr;
}
```

剩余的工作就是利用canvas `arc`方法进行曲线绘制了。

### 找出数组的最大差值

**比如：** 输入 [10,5,11,7,8,9]  输出 6

这是通过一道题目去测试对于基本的数组的最大值的查找，很明显我们知道，最大差值肯定是一个数组中最大值与最小值的差。

```javascript
function getMaxProfit(arr) {

    var minPrice = arr[0];
    var maxProfit = 0;

    for (var i = 0; i < arr.length; i++) {
        var currentPrice = arr[i];

        minPrice = Math.min(minPrice, currentPrice);

        var potentialProfit = currentPrice - minPrice;

        maxProfit = Math.max(maxProfit, potentialProfit);
    }

    return maxProfit;
}
```

### 随机生成指定长度的字符串

实现一个算法，随机生成指制定长度的字符窜。

**比如：**给定 长度 8 输出 4ldkfg9j

```javascript
function randomString(n) {  
  let str = 'abcdefghijklmnopqrstuvwxyz9876543210';
  let tmp = '',
      i = 0,
      l = str.length;
  for (i = 0; i < n; i++) {
    tmp += str.charAt(Math.floor(Math.random() * l));
  }
  return tmp;
}

module.exports = randomString;  
```

### 实现类似getElementsByClassName的功能

自己实现一个函数，查找某个DOM节点下面的包含某个class的所有DOM节点？不允许使用原生提供的 getElementsByClassName querySelectorAll 等原生提供DOM查找函数。

```javascript
function queryClassName(node, name) {  
  var starts = '(^|[ \n\r\t\f])',
       ends = '([ \n\r\t\f]|$)';
  var array = [],
        regex = new RegExp(starts + name + ends),
        elements = node.getElementsByTagName("*"),
        length = elements.length,
        i = 0,
        element;

    while (i < length) {
        element = elements[i];
        if (regex.test(element.className)) {
            array.push(element);
        }

        i += 1;
    }

    return array;
}
```

### 使用JS实现二叉查找树（Binary Search Tree）

一般叫全部写完的概率比较少，但是重点考察你对它的理解和一些基本特点的实现。 二叉查找树，也称二叉搜索树、有序二叉树（英语：ordered binary tree）是指一棵空树或者具有下列性质的二叉树：

- 任意节点的左子树不空，则左子树上所有结点的值均小于它的根结点的值；
- 任意节点的右子树不空，则右子树上所有结点的值均大于它的根结点的值；
- 任意节点的左、右子树也分别为二叉查找树；
- 没有键值相等的节点。二叉查找树相比于其他数据结构的优势在于查找、插入的时间复杂度较低。为O(log n)。二叉查找树是基础性数据结构，用于构建更为抽象的数据结构，如集合、multiset、关联数组等。

![img](https://images2015.cnblogs.com/blog/738658/201707/738658-20170719222527693-1678554844.png)

 

  在写的时候需要足够理解二叉搜素树的特点，需要先设定好每个节点的数据结构

```javascript
class Node {  
  constructor(data, left, right) {
    this.data = data;
    this.left = left;
    this.right = right;
  }
}
```

树是有节点构成，由根节点逐渐延生到各个子节点，因此它具备基本的结构就是具备一个根节点，具备添加，查找和删除节点的方法.

```javascript
class BinarySearchTree {

  constructor() {
    this.root = null;
  }

  insert(data) {
    let n = new Node(data, null, null);
    if (!this.root) {
      return this.root = n;
    }
    let currentNode = this.root;
    let parent = null;
    while (1) {
      parent = currentNode;
      if (data < currentNode.data) {
        currentNode = currentNode.left;
        if (currentNode === null) {
          parent.left = n;
          break;
        }
      } else {
        currentNode = currentNode.right;
        if (currentNode === null) {
          parent.right = n;
          break;
        }
      }
    }
  }

  remove(data) {
    this.root = this.removeNode(this.root, data)
  }

  removeNode(node, data) {
    if (node == null) {
      return null;
    }

    if (data == node.data) {
      // no children node
      if (node.left == null && node.right == null) {
        return null;
      }
      if (node.left == null) {
        return node.right;
      }
      if (node.right == null) {
        return node.left;
      }

      let getSmallest = function(node) {
        if(node.left === null && node.right == null) {
          return node;
        }
        if(node.left != null) {
          return node.left;
        }
        if(node.right !== null) {
          return getSmallest(node.right);
        }

      }
      let temNode = getSmallest(node.right);
      node.data = temNode.data;
      node.right = this.removeNode(temNode.right,temNode.data);
      return node;

    } else if (data < node.data) {
      node.left = this.removeNode(node.left,data);
      return node;
    } else {
      node.right = this.removeNode(node.right,data);
      return node;
    }
  }

  find(data) {
    var current = this.root;
    while (current != null) {
      if (data == current.data) {
        break;
      }
      if (data < current.data) {
        current = current.left;
      } else {
        current = current.right
      }
    }
    return current.data;
  }

}

module.exports = BinarySearchTree;  
```

## 七、柯里化

最近，朋友T 在准备面试，他为一道编程题所困，向我求助。原题如下：

```javascript
// 写一个 sum 方法，当使用下面的语法调用时，能正常工作
console.log(sum(2, 3)); // Outputs 5
console.log(sum(2)(3)); // Outputs 5
```

这道题要考察的，就是对函数柯里化的理解。让我们先来解析一下题目的要求：

- 如果传递两个参数，我们只需将它们相加并返回。
- 否则，我们假设它是以sum(2)(3)的形式被调用的，所以我们返回一个匿名函数，它将传递给sum()（在本例中为2）的参数和传递给匿名函数的参数（在本例中为3）。

所以，sum 函数可以这样写：

```javascript
function sum (x) {
    if (arguments.length == 2) {
        return arguments[0] + arguments[1];
    }
    
    return function(y) {
        return x + y;
    }
}
```

arguments 的用法挺灵活的，在这里它则用于分割两种不同的情况。当参数只有一个的时候，进行柯里化的处理。

那么，到底什么是函数的柯里化呢？接下来，我们将从概念出发，探究函数柯里化的实现与用途。

###什么是柯里化

柯里化，是函数式编程的一个重要概念。它既能减少代码冗余，也能增加可读性。另外，附带着还能用来装逼。

先给出柯里化的定义：在数学和计算机科学中，柯里化是一种将使用多个参数的一个函数转换成一系列使用一个参数的函数的技术。

柯里化的定义，理解起来有点费劲。为了更好地理解，先看下面这个例子：

```javascript
function sum (a, b, c) {
    console.log(a + b + c);
}
sum(1, 2, 3); // 6
```

毫无疑问，sum 是个简单的累加函数，接受3个参数，输出累加的结果。

假设有这样的需求，sum的前2个参数保持不变，最后一个参数可以随意。那么就会想到，在函数内，是否可以把前2个参数的相加过程，给抽离出来，因为参数都是相同的，没必要每次都做运算。

如果先不管函数内的具体实现，调用的写法可以是这样： `sum(1, 2)(3);` 或这样 `sum(1, 2)(10);` 。就是，先把前2个参数的运算结果拿到后，再与第3个参数相加。

这其实就是函数柯里化的简单应用。

###柯里化的实现

`sum(1, 2)(3);` 这样的写法，并不常见。拆开来看，`sum(1, 2)` 返回的应该还是个函数，因为后面还有 `(3)` 需要执行。

那么反过来，从最后一个参数，从右往左看，它的左侧必然是一个函数。以此类推，如果前面有n个()，那就是有n个函数返回了结果，只是返回的结果，还是一个函数。是不是有点递归的意思？

网上有一些不同的柯里化的实现方式，以下是个人觉得最容易理解的写法：

```javascript
function curry (fn, currArgs) {
    return function() {
        let args = [].slice.call(arguments);

        // 首次调用时，若未提供最后一个参数currArgs，则不用进行args的拼接
        if (currArgs !== undefined) {
            args = args.concat(currArgs);
        }

        // 递归调用
        if (args.length < fn.length) {
            return curry(fn, args);
        }

        // 递归出口
        return fn.apply(null, args);
    }
}
```

解析一下 curry 函数的写法：

首先，它有 2 个参数，fn 指的就是本文一开始的源处理函数 `sum`。currArgs 是调用 curry 时传入的参数列表，比如 `(1, 2)(3)` 这样的。

再看到 curry 函数内部，它会整个返回一个匿名函数。

再接下来的 `let args = [].slice.call(arguments);`，意思是将 arguments 数组化。arguments 是一个类数组的结构，它并不是一个真的数组，所以没法使用数组的方法。我们用了 call 的方法，就能愉快地对 args 使用数组的原生方法了。在这篇 [「干货」细说 call、apply 以及 bind 的区别和用法](https://juejin.im/post/5c493086f265da6115111ce4) 中，有关于 call 更详细的用法介绍。

`currArgs !== undefined` 的判断，是为了解决递归调用时的参数拼接。

最后，判断 args 的个数，是否与 fn (也就是 sum )的参数个数相等，相等了就可以把参数都传给 fn，进行输出；否则，继续递归调用，直到两者相等。

测试一下：

```javascript
function sum(a, b, c) {
    console.log(a + b + c);
}

const fn = curry(sum);

fn(1, 2, 3); // 6
fn(1, 2)(3); // 6
fn(1)(2, 3); // 6
fn(1)(2)(3); // 6
```

都能输出 6 了，搞定！

###柯里化的用途

理解了柯里化的实现之后，让我们来看一下它的实际应用。柯里化的目的是，减少代码冗余，以及增加代码的可读性。来看下面这个例子：

```javascript
const persons = [
    { name: 'kevin', age: 4 },
    { name: 'bob', age: 5 }
];

// 这里的 curry 函数，之前已实现
const getProp = curry(function (obj, index) {
    const args = [].slice.call(arguments);
    return obj[args[args.length - 1]];
});

const ages = persons.map(getProp('age')); // [4, 5]
const names = persons.map(getProp('name')); // ['kevin', 'bob']
```

在实际的业务中，我们常会遇到类似的列表数据。用 getProp 就可以很方便地，取出列表中某个 key 对应的值。

需要注意的是，`const names = persons.map(getProp('name'));` 执行这条语句时 getProp 的参数只有一个 `name`，而定义 getProp 方法时，传入 curry 的参数有2个，`obj` 和 `index`（这里必须写 2 个及以上的参数）。

为什么要这么写？关键就在于 `arguments` 的隐式传参。

```javascript
const getProp = curry(function (obj, index) {
    console.log(arguments);
    // 会输出4个类数组，取其中一个来看
    // {
    //     0: {name: "kevin", age: 4},
    //     1: 0,
    //     2: [
    //         {name: "kevin", age: 4},
    //         {name: "bob", age: 5}
    //     ],
    //     3: "age"
    // }
});
```

map 是 Array 的原生方法，它的用法如下：

```javascript
var new_array = arr.map(function callback(currentValue[, index[, array]]) {
    // Return element for new_array
}[, thisArg]);
```

所以，我们传入的 `name`，就排在了 arguments 的最后。为了拿到 `name` 对应的值，需要对类数组 arguments 做点转换，让它可以使用 Array 的原生方法。所以，最终 getProp 方法定义成了这样：

```javascript
const getProp = curry(function (obj, index) {
    const args = [].slice.call(arguments);
    return obj[args[args.length - 1]];
});
```

当然，还有另外一种写法，curry 的实现更好理解，但是调用的代码却变多了，大家可以根据实际情况进行取舍。

```javascript
const getProp = curry(function (key, obj) {
    return obj[key];
});

const ages = persons.map(item => {
    return getProp(item)('age');
});
const names = persons.map(item => {
    return getProp(item)('name');
});
```

最后，来看一个 Memoization 的例子。它用于优化比较耗时的计算，通过将计算结果缓存到内存中，这样对于同样的输入值，下次只需要中内存中读取结果。

```javascript
function memoizeFunction(func) {
    const cache = {};
    return function() {
        let key = arguments[0];
        if (cache[key]) {
            return cache[key];
        } else {
            const val = func.apply(null, arguments);
            cache[key] = val;
            return val;
        }
    };
}

const fibonacci = memoizeFunction(function(n) {
    return (n === 0 || n === 1) ? n : fibonacci(n - 1) + fibonacci(n - 2);
});

console.log(fibonacci(100)); // 输出354224848179262000000
console.log(fibonacci(100)); // 输出354224848179262000000
```

代码中，第2次计算 fibonacci(100) 则只需要在内存中直接读取结果。

###总结

**函数的柯里化，是 Javascript 中函数式编程的一个重要概念。它返回的，是一个函数的函数。其实现方式，需要依赖参数以及递归，通过拆分参数的方式，来调用一个多参数的函数方法，以达到减少代码冗余，增加可读性的目的。**

## 八、Array的方法介绍

###ES5 中 Array 操作

#### 1、forEach

Array 方法中最基本的一个，就是遍历，循环。基本用法：`[].forEach(function(item, index, array) {});`

```javascript
const array = [1, 2, 3];
const result = [];

array.forEach(function(item) {
    result.push(item);
});
console.log(result); // [1, 2, 3]
```

#### 2、map

map 方法的作用不难理解，“映射”嘛，也就是原数组被 “映射” 成对应的新数组。基本用法跟 forEach 方法类似：`[].map(function(item, index, array) {});` 下面这个例子是数值项求平方：

```javascript
const data = [1, 2, 3, 4];
const arrayOfSquares = data.map(function (item) {
    return item * item;
});
alert(arrayOfSquares); // 1, 4, 9, 16
```

#### 3、filter

filter 为 “过滤”、“筛选” 之意。指数组 filter 后，返回过滤后的新数组。用法跟 map 极为相似：`[].filter(function(item, index, array) {});`

filter 的 callback 函数，需要返回布尔值 true 或 false。返回值只要 **弱等于** Boolean 就行，看下面这个例子：

```javascript
const data = [0, 1, 2, 3];
const arrayFilter = data.filter(function(item) {
    return item;
});
console.log(arrayFilter); // [1, 2, 3]
```

#### 4、some 和 every

some 意指“某些”，指是否 “某些项” 合乎条件。也就是 **只要有1值个让 callback 返回 true**  就行了。基本用法：`[].som(function(item, index, array) {});`

```javascript
const scores = [5, 8, 3, 10];
const current = 7;

function higherThanCurrent(score) {
    return score > current;
}

if (scores.some(higherThanCurrent)) {
    alert("one more");
}
```

every 跟 some 是一对好基友，同样是返回 Boolean 值。但必须满足每 1 个值都要让 callback 返回 true 才行。改动一下上面 some 的例子：

```javascript
if (scores.every(higherThanCurrent)) {
    console.log("every is ok!");
} else {
    console.log("oh no!");        
}
```

#### 5、concat

concat() 方法用于连接两个或多个数组。该方法不会改变现有的数组，仅会返回被连接数组的一个副本。

```javascript
const arr1 = [1,2,3];
const arr2 = [4,5];
const arr3 = arr1.concat(arr2);
console.log(arr1); // [1, 2, 3]
console.log(arr3); // [1, 2, 3, 4, 5]
```

#### 6、indexOf 和 lastIndexOf

indexOf 方法在字符串中自古就有，string.indexOf(searchString, position)。数组这里的 indexOf 方法与之类似。 返回整数索引值，如果没有匹配（严格匹配），返回-1.

fromIndex可选，表示从这个位置开始搜索，若缺省或格式不合要求，使用默认值0。

```javascript
const data = [2, 5, 7, 3, 5];
console.log(data.indexOf(5, "x"));  // 1 ("x"被忽略)
console.log(data.indexOf(5, "3")); // 4 (从3号位开始搜索)

console.log(data.indexOf(4));    // -1 (未找到)
console.log(data.indexOf("5")); // -1 (未找到，因为5 !== "5")
```

lastIndexOf 则是从后往前找。

```javascript
const numbers = [2, 5, 9, 2];
numbers.lastIndexOf(2);     // 3
numbers.lastIndexOf(7);     // -1
numbers.lastIndexOf(2, 3);  // 3
numbers.lastIndexOf(2, 2);  // 0
numbers.lastIndexOf(2, -2); // 0
numbers.lastIndexOf(2, -1); // 3
```

#### 7、reduce 和 reduceRight

reduce 是JavaScript 1.8 中才引入的，中文意思为“归约”。基本用法：`reduce(callback[, initialValue])`

callback 函数接受4个参数：之前值(previousValue)、当前值(currentValue)、索引值(currentIndex)以及数组本身(array)。

可选的初始值(initialValue)，作为第一次调用回调函数时传给previousValue的值。也就是，为累加等操作传入起始值（额外的加值）。

```javascript
var sum = [1, 2, 3, 4].reduce(function (previous, current, index, array) {
    return previous + current;
});
console.log(sum); // 10
```

解析：

- 因为initialValue不存在，因此一开始的previous值等于数组的第一个元素
- 从而 current 值在第一次调用的时候就是2
- 最后两个参数为索引值 index 以及数组本身 array

以下为循环执行过程：

```javascript
// 初始设置
previous = initialValue = 1, current = 2

// 第一次迭代
previous = (1 + 2) =  3, current = 3

// 第二次迭代
previous = (3 + 3) =  6, current = 4

// 第三次迭代
previous = (6 + 4) =  10, current = undefined (退出)
```

有了reduce，我们可以轻松实现二维数组的扁平化：

```javascript
var matrix = [
  [1, 2],
  [3, 4],
  [5, 6]
];

// 二维数组扁平化
var flatten = matrix.reduce(function (previous, current) {
    return previous.concat(current);
});
console.log(flatten); // [1, 2, 3, 4, 5, 6]
```

reduceRight 跟 reduce 相比，用法类似。实现上差异在于reduceRight是从数组的末尾开始实现。

```javascript
const data = [1, 2, 3, 4];
const specialDiff = data.reduceRight(function (previous, current, index) {
    if (index == 0) {
        return previous + current;
    }
    return previous - current;
});
console.log(specialDiff);  // 0
```

#### 8、push 和 pop

push() 方法可向数组的末尾添加一个或多个元素，返回的是新的数组长度，会改变原数组。

```javascript
const a = [2,3,4];
const b = a.push(5);
console.log(a);  // [2,3,4,5]
console.log(b);  // 4
// push方法可以一次添加多个元素push(data1, data2....)
```

pop() 方法用于删除并返回数组的最后一个元素。返回的是最后一个元素，会改变原数组。

```javascript
const arr = [2,3,4];
console.log(arr.pop()); // 4
console.log(arr);  // [2,3]
```

#### 9、shift 和 unshift

shift() 方法用于把数组的第一个元素从其中删除，并返回第一个元素的值。返回第一个元素，改变原数组。

```javascript
const arr = [2,3,4];
console.log(arr.shift()); // 2
console.log(arr);  // [3,4]
```

unshift() 方法可向数组的开头添加一个或更多元素，并返回新的长度。返回新长度，改变原数组。

```javascript
const arr = [2,3,4,5];
console.log(arr.unshift(3,6)); // 6
console.log(arr); // [3, 6, 2, 3, 4, 5]
// tip:该方法可以不传参数,不传参数就是不增加元素。
```

#### 10、slice 和 splice

slice() 返回一个新的数组，包含从 start 到 end （不包括该元素）的 arrayObject 中的元素。返回选定的元素，该方法不会修改原数组。

```javascript
const arr = [2,3,4,5];
console.log(arr.slice(1,3));  // [3,4]
console.log(arr);  // [2,3,4,5]
```

splice() 可删除从 index 处开始的零个或多个元素，并且用参数列表中声明的一个或多个值来替换那些被删除的元素。如果从 arrayObject 中删除了元素，则返回的是含有被删除的元素的数组。splice() 方法会直接对数组进行修改。

```javascript
const a = [5,6,7,8];
console.log(a.splice(1,0,9)); //[]
console.log(a);  // [5, 9, 6, 7, 8]

const b = [5,6,7,8];
console.log(b.splice(1,2,3));  //v[6, 7]
console.log(b); // [5, 3, 8]
```

#### 11、sort 和 reverse

sort() 按照 Unicode code 位置排序，默认升序。

```javascript
const fruit = ['cherries', 'apples', 'bananas'];
fruit.sort(); // ['apples', 'bananas', 'cherries']

const scores = [1, 10, 21, 2];
scores.sort(); // [1, 10, 2, 21]
```

reverse() 方法用于颠倒数组中元素的顺序。返回的是颠倒后的数组，会改变原数组。

```javascript
const arr = [2,3,4];
console.log(arr.reverse()); // [4, 3, 2]
console.log(arr);  // [4, 3, 2]
```

#### 12、join

join() 方法用于把数组中的所有元素放入一个字符串。元素是通过指定的分隔符进行分隔的，默认使用','号分割，不改变原数组。

```javascript
const arr = [2,3,4];
console.log(arr.join());  // 2,3,4
console.log(arr);  // [2, 3, 4]
```

#### 13、isArray

Array.isArray() 用于确定传递的值是否是一个 Array。一个比较冷门的知识点：其实 Array.prototype 也是一个数组。

```javascript
Array.isArray([]); // true
Array.isArray(Array.prototype); // true

Array.isArray(null); // false
Array.isArray(undefined); // false
Array.isArray(18); // false
Array.isArray('Array'); // false
Array.isArray(true); // false
Array.isArray({ __proto__: Array.prototype });
```

在公用库中，一般会这么做 isArray 的判断：

```
Object.prototype.toString.call(arg) === '[object Array]';
复制代码
```

###ES6 新增的 Array 操作

#### 1、find 和 findIndex

find() 传入一个回调函数，找到数组中符合当前搜索规则的第一个元素，返回这个元素，并且终止搜索。

```javascript
const arr = [1, "2", 3, 3, "2"]
console.log(arr.find(n => typeof n === "number")) // 1
```

findIndex() 与 find() 类似，只是返回的是，找到的这个元素的下标。

```javascript
const arr = [1, "2", 3, 3, "2"]
console.log(arr.findIndex(n => typeof n === "number")) // 0
```

#### 2、fill

用指定的元素填充数组，其实就是用默认内容初始化数组。基本用法：`[].fill(value, start, end)`

该函数有三个参数：填充值(value)，填充起始位置(start，可以省略)，填充结束位置(end，可以省略，实际结束位置是end-1)。

```javascript
// 采用一个默认值，填充数组
const arr1 = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11];
arr1.fill(7);
console.log(arr1); // [7,7,7,7,7,7,7,7,7,7,7]

// 制定开始和结束位置填充，实际填充结束位置是前一位。
const arr2 = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11];
arr2.fill(7, 2, 5);
console.log(arr2); // [1,2,7,7,7,6,7,8,9,10,11]

// 结束位置省略，从起始位置到最后。
const arr3 = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11];
arr3.fill(7, 2);
console.log(arr3); // [1,2,7,7,7,7,7,7,7,7,7]
```

#### 3、from

将类似数组的对象（array-like object）和可遍历（iterable）的对象转为真正的数组。

```javascript
const set = new Set(1, 2, 3, 3, 4);
Array.from(set)  // [1,2,3,4]

Array.from('foo'); // ["f", "o", "o"]
```

#### 4、of

Array.of() 方法创建一个具有可变数量参数的新数组实例，而不考虑参数的数量或类型。

Array.of() 和 Array 构造函数之间的区别在于处理整数参数：Array.of(7) 创建一个具有单个元素 7 的数组，而 Array(7) 创建一个长度为7的空数组（注意：这是指一个有7个空位的数组，而不是由7个undefined组成的数组）。

```javascript
Array.of(7);       // [7] 
Array.of(1, 2, 3); // [1, 2, 3]

Array(7);          // [ , , , , , , ]
Array(1, 2, 3);    // [1, 2, 3]
```

#### 5、copyWithin

选择数组的某个下标，从该位置开始复制数组元素，默认从0开始复制。也可以指定要复制的元素范围。基本用法：`[].copyWithin(target, start, end)`

```javascript
const arr = [1, 2, 3, 4, 5];
console.log(arr.copyWithin(3));
 // [1,2,3,1,2] 从下标为3的元素开始，复制数组，所以4, 5被替换成1, 2

const arr1 = [1, 2, 3, 4, 5];
console.log(arr1.copyWithin(3, 1)); 
// [1,2,3,2,3] 从下标为3的元素开始，复制数组，指定复制的第一个元素下标为1，所以4, 5被替换成2, 3

const arr2 = [1, 2, 3, 4, 5];
console.log(arr2.copyWithin(3, 1, 2));
// [1,2,3,2,5] 从下标为3的元素开始，复制数组，指定复制的第一个元素下标为1，结束位置为2，所以4被替换成2
```

#### 6、includes

判断数组中是否存在该元素，参数：查找的值、起始位置，可以替换 ES5 时代的 indexOf 判断方式。

```javascript
const arr = [1, 2, 3];
arr.includes(2); // true
arr.includes(4); // false
```

另外，它还可以用于优化 || 的判断写法。

```javascript
if (method === 'post' || method === 'put' || method === 'delete') {
    ...
}

// 用 includes 优化 `||` 的写法
if (['post', 'put', 'delete'].includes(method)) {
    ...
}
```

#### 7、entries、values 和 keys

entries() 返回迭代器：返回键值对

```javascript
//数组
const arr = ['a', 'b', 'c'];
for(let v of arr.entries()) {
    console.log(v)
}
// [0, 'a'] [1, 'b'] [2, 'c']

//Set
const arr = new Set(['a', 'b', 'c']);
for(let v of arr.entries()) {
    console.log(v)
}
// ['a', 'a'] ['b', 'b'] ['c', 'c']

//Map
const arr = new Map();
arr.set('a', 'a');
arr.set('b', 'b');
for(let v of arr.entries()) {
    console.log(v)
}
// ['a', 'a'] ['b', 'b']
```

values() 返回迭代器：返回键值对的 value

```javascript
//数组
const arr = ['a', 'b', 'c'];
for(let v of arr.values()) {
    console.log(v)
}
//'a' 'b' 'c'

//Set
const arr = new Set(['a', 'b', 'c']);
for(let v of arr.values()) {
    console.log(v)
}
// 'a' 'b' 'c'

//Map
const arr = new Map();
arr.set('a', 'a');
arr.set('b', 'b');
for(let v of arr.values()) {
    console.log(v)
}
// 'a' 'b'
```

keys() 返回迭代器：返回键值对的 key

```javascript
//数组
const arr = ['a', 'b', 'c'];
for(let v of arr.keys()) {
    console.log(v)
}
// 0 1 2

//Set
const arr = new Set(['a', 'b', 'c']);
for(let v of arr.keys()) {
    console.log(v)
}
// 'a' 'b' 'c'

//Map
const arr = new Map();
arr.set('a', 'a');
arr.set('b', 'b');
for(let v of arr.keys()) {
    console.log(v)
}
// 'a' 'b'
```

## 九、Promise的实现

Promise是前端面试中的高频问题，我作为面试官的时候，问Promise的概率超过90%，据我所知，大多数公司，都会问一些关于Promise的问题。如果你能根据PromiseA+的规范，写出符合规范的源码，那么我想，对于面试中的Promise相关的问题，都能够给出比较完美的答案。

我的建议是，对照规范多写几次实现，也许第一遍的时候，是改了多次，才能通过测试，那么需要反复的写，我已经将Promise的源码实现写了不下七遍。

### Promise的源码实现

```javascript
/**
 * 1. new Promise时，需要传递一个 executor 执行器，执行器立刻执行
 * 2. executor 接受两个参数，分别是 resolve 和 reject
 * 3. promise 只能从 pending 到 rejected, 或者从 pending 到 fulfilled
 * 4. promise 的状态一旦确认，就不会再改变
 * 5. promise 都有 then 方法，then 接收两个参数，分别是 promise 成功的回调 onFulfilled, 
 *      和 promise 失败的回调 onRejected
 * 6. 如果调用 then 时，promise已经成功，则执行 onFulfilled，并将promise的值作为参数传递进去。
 *      如果promise已经失败，那么执行 onRejected, 并将 promise 失败的原因作为参数传递进去。
 *      如果promise的状态是pending，需要将onFulfilled和onRejected函数存放起来，等待状态确定后，再依次将对应的函数执行(发布订阅)
 * 7. then 的参数 onFulfilled 和 onRejected 可以缺省
 * 8. promise 可以then多次，promise 的then 方法返回一个 promise
 * 9. 如果 then 返回的是一个结果，那么就会把这个结果作为参数，传递给下一个then的成功的回调(onFulfilled)
 * 10. 如果 then 中抛出了异常，那么就会把这个异常作为参数，传递给下一个then的失败的回调(onRejected)
 * 11.如果 then 返回的是一个promise,那么需要等这个promise，那么会等这个promise执行完，promise如果成功，
 *   就走下一个then的成功，如果失败，就走下一个then的失败
 */

const PENDING = 'pending';
const FULFILLED = 'fulfilled';
const REJECTED = 'rejected';
function Promise(executor) {
    let self = this;
    self.status = PENDING;
    self.onFulfilled = [];//成功的回调
    self.onRejected = []; //失败的回调
    //PromiseA+ 2.1
    function resolve(value) {
        if (self.status === PENDING) {
            self.status = FULFILLED;
            self.value = value;
            self.onFulfilled.forEach(fn => fn());//PromiseA+ 2.2.6.1
        }
    }

    function reject(reason) {
        if (self.status === PENDING) {
            self.status = REJECTED;
            self.reason = reason;
            self.onRejected.forEach(fn => fn());//PromiseA+ 2.2.6.2
        }
    }

    try {
        executor(resolve, reject);
    } catch (e) {
        reject(e);
    }
}

Promise.prototype.then = function (onFulfilled, onRejected) {
    //PromiseA+ 2.2.1 / PromiseA+ 2.2.5 / PromiseA+ 2.2.7.3 / PromiseA+ 2.2.7.4
    onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : value => value;
    onRejected = typeof onRejected === 'function' ? onRejected : reason => { throw reason };
    let self = this;
    //PromiseA+ 2.2.7
    let promise2 = new Promise((resolve, reject) => {
        if (self.status === FULFILLED) {
            //PromiseA+ 2.2.2
            //PromiseA+ 2.2.4 --- setTimeout
            setTimeout(() => {
                try {
                    //PromiseA+ 2.2.7.1
                    let x = onFulfilled(self.value);
                    resolvePromise(promise2, x, resolve, reject);
                } catch (e) {
                    //PromiseA+ 2.2.7.2
                    reject(e);
                }
            });
        } else if (self.status === REJECTED) {
            //PromiseA+ 2.2.3
            setTimeout(() => {
                try {
                    let x = onRejected(self.reason);
                    resolvePromise(promise2, x, resolve, reject);
                } catch (e) {
                    reject(e);
                }
            });
        } else if (self.status === PENDING) {
            self.onFulfilled.push(() => {
                setTimeout(() => {
                    try {
                        let x = onFulfilled(self.value);
                        resolvePromise(promise2, x, resolve, reject);
                    } catch (e) {
                        reject(e);
                    }
                });
            });
            self.onRejected.push(() => {
                setTimeout(() => {
                    try {
                        let x = onRejected(self.reason);
                        resolvePromise(promise2, x, resolve, reject);
                    } catch (e) {
                        reject(e);
                    }
                });
            });
        }
    });
    return promise2;
}

function resolvePromise(promise2, x, resolve, reject) {
    let self = this;
    //PromiseA+ 2.3.1
    if (promise2 === x) {
        reject(new TypeError('Chaining cycle'));
    }
    if (x && typeof x === 'object' || typeof x === 'function') {
        let used; //PromiseA+2.3.3.3.3 只能调用一次
        try {
            let then = x.then;
            if (typeof then === 'function') {
                //PromiseA+2.3.3
                then.call(x, (y) => {
                    //PromiseA+2.3.3.1
                    if (used) return;
                    used = true;
                    resolvePromise(promise2, y, resolve, reject);
                }, (r) => {
                    //PromiseA+2.3.3.2
                    if (used) return;
                    used = true;
                    reject(r);
                });

            }else{
                //PromiseA+2.3.3.4
                if (used) return;
                used = true;
                resolve(x);
            }
        } catch (e) {
            //PromiseA+ 2.3.3.2
            if (used) return;
            used = true;
            reject(e);
        }
    } else {
        //PromiseA+ 2.3.3.4
        resolve(x);
    }
}

module.exports = Promise;
```

有专门的测试脚本可以测试所编写的代码是否符合PromiseA+的规范。

首先，在promise实现的代码中，增加以下代码:

```javascript
Promise.defer = Promise.deferred = function () {
    let dfd = {};
    dfd.promise = new Promise((resolve, reject) => {
        dfd.resolve = resolve;
        dfd.reject = reject;
    });
    return dfd;
}
```

安装测试脚本:

```bash
npm install -g promises-aplus-tests
```

如果当前的promise源码的文件名为promise.js

那么在对应的目录执行以下命令:

```shell
promises-aplus-tests promise.js
```

promises-aplus-tests中共有872条测试用例。以上代码，可以完美通过所有用例。

**对上面的代码实现做一点简要说明(其它一些内容注释中已经写得很清楚):**

1. onFulfilled 和 onFulfilled的调用需要放在setTimeout，因为规范中表示: onFulfilled or onRejected must not be called until the execution context stack contains only platform code。使用setTimeout只是模拟异步，原生Promise并非是这样实现的。
2. 在 resolvePromise 的函数中，为何需要usedd这个flag,同样是因为规范中明确表示: If both resolvePromise and rejectPromise are called, or multiple calls to the same argument are made, the first call takes precedence, and any further calls are ignored. 因此我们需要这样的flag来确保只会执行一次。
3. self.onFulfilled 和 self.onRejected 中存储了成功的回调和失败的回调，根据规范2.6显示，当promise从pending态改变的时候，需要按照顺序去指定then对应的回调。

###PromiseA+的规范(翻译版)

PS: 下面是我翻译的规范，供参考

> 术语

1. promise 是一个有then方法的对象或者是函数，行为遵循本规范
2. thenable 是一个有then方法的对象或者是函数
3. value 是promise状态成功时的值，包括 undefined/thenable或者是 promise
4. exception 是一个使用throw抛出的异常值
5. reason 是promise状态失败时的值

> 要求

#### 2.1 Promise States

Promise 必须处于以下三个状态之一: pending, fulfilled 或者是 rejected

##### 2.1.1 如果promise在pending状态

```
2.1.1.1 可以变成 fulfilled 或者是 rejected
```

##### 2.1.2 如果promise在fulfilled状态

```
2.1.2.1 不会变成其它状态

2.1.2.2 必须有一个value值
```

##### 2.1.3 如果promise在rejected状态

```
2.1.3.1 不会变成其它状态

2.1.3.2 必须有一个promise被reject的reason
```

概括即是:promise的状态只能从pending变成fulfilled，或者从pending变成rejected.promise成功，有成功的value.promise失败的话，有失败的原因

#### 2.2 then方法

promise必须提供一个then方法，来访问最终的结果

promise的then方法接收两个参数

```javascript
promise.then(onFulfilled, onRejected)
```

##### 2.2.1 onFulfilled 和 onRejected 都是可选参数

```
2.2.1.1 onFulfilled 必须是函数类型

2.2.1.2 onRejected 必须是函数类型
```

##### 2.2.2 如果 onFulfilled 是函数:

```
2.2.2.1 必须在promise变成 fulfilled 时，调用 onFulfilled，参数是promise的value
2.2.2.2 在promise的状态不是 fulfilled 之前，不能调用
2.2.2.3 onFulfilled 只能被调用一次
```

##### 2.2.3 如果 onRejected 是函数:

```
2.2.3.1 必须在promise变成 rejected 时，调用 onRejected，参数是promise的reason
2.2.3.2 在promise的状态不是 rejected 之前，不能调用
2.2.3.3 onRejected 只能被调用一次
```

##### 2.2.4 onFulfilled 和 onRejected 应该是微任务

##### 2.2.5 onFulfilled  和 onRejected 必须作为函数被调用

##### 2.2.6 then方法可能被多次调用

```
2.2.6.1 如果promise变成了 fulfilled态，所有的onFulfilled回调都需要按照then的顺序执行
2.2.6.2 如果promise变成了 rejected态，所有的onRejected回调都需要按照then的顺序执行
```

##### 2.2.7 then必须返回一个promise

```
promise2 = promise1.then(onFulfilled, onRejected);

2.2.7.1 onFulfilled 或 onRejected 执行的结果为x,调用 resolvePromise
2.2.7.2 如果 onFulfilled 或者 onRejected 执行时抛出异常e,promise2需要被reject
2.2.7.3 如果 onFulfilled 不是一个函数，promise2 以promise1的值fulfilled
2.2.7.4 如果 onRejected 不是一个函数，promise2 以promise1的reason rejected
```

#### 2.3 resolvePromise

resolvePromise(promise2, x, resolve, reject)

##### 2.3.1 如果 promise2 和 x 相等，那么 reject promise with a TypeError

##### 2.3.2 如果 x 是一个 promsie

```
2.3.2.1 如果x是pending态，那么promise必须要在pending,直到 x 变成 fulfilled or rejected.
2.3.2.2 如果 x 被 fulfilled, fulfill promise with the same value.
2.3.2.3 如果 x 被 rejected, reject promise with the same reason.
```

##### 2.3.3 如果 x 是一个 object 或者 是一个 function

```
2.3.3.1 let then = x.then.
2.3.3.2 如果 x.then 这步出错，那么 reject promise with e as the reason..
2.3.3.3 如果 then 是一个函数，then.call(x, resolvePromiseFn, rejectPromise)
    2.3.3.3.1 resolvePromiseFn 的 入参是 y, 执行 resolvePromise(promise2, y, resolve, reject);
    2.3.3.3.2 rejectPromise 的 入参是 r, reject promise with r.
    2.3.3.3.3 如果 resolvePromise 和 rejectPromise 都调用了，那么第一个调用优先，后面的调用忽略。
    2.3.3.3.4 如果调用then抛出异常e 
        2.3.3.3.4.1 如果 resolvePromise 或 rejectPromise 已经被调用，那么忽略
        2.3.3.3.4.3 否则，reject promise with e as the reason
2.3.3.4 如果 then 不是一个function. fulfill promise with x.
```

##### 2.3.4 如果 x 不是一个 object 或者 function，fulfill promise with x.

###Promise的其他方法

虽然上述的promise源码已经符合PromiseA+的规范，但是原生的Promise还提供了一些其他方法，如:

1. Promise.resolve()
2. Promise.reject()
3. Promise.prototype.catch()
4. Promise.prototype.finally()
5. Promise.all()
6. Promise.race()

下面具体说一下每个方法的实现:

> ### Promise.resolve

Promise.resolve(value) 返回一个以给定值解析后的Promise 对象.

1. 如果 value 是个 thenable 对象，返回的promise会“跟随”这个thenable的对象，采用它的最终状态
2. 如果传入的value本身就是promise对象，那么Promise.resolve将不做任何修改、原封不动地返回这个promise对象。
3. 其他情况，直接返回以该值为成功状态的promise对象。

```javascript
Promise.resolve = function (param) {
        if (param instanceof Promise) {
        return param;
    }
    return new Promise((resolve, reject) => {
        if (param && param.then && typeof param.then === 'function') {
            setTimeout(() => {
                param.then(resolve, reject);
            });
        } else {
            resolve(param);
        }
    });
}
```

thenable对象的执行加 setTimeout的原因是根据原生Promise对象执行的结果推断的，如下的测试代码，原生的执行结果为: 20  400  30;为了同样的执行顺序，增加了setTimeout延时。

测试代码:

```javascript
let p = Promise.resolve(20);
p.then((data) => {
    console.log(data);
});


let p2 = Promise.resolve({
    then: function(resolve, reject) {
        resolve(30);
    }
});

p2.then((data)=> {
    console.log(data)
});

let p3 = Promise.resolve(new Promise((resolve, reject) => {
    resolve(400)
}));
p3.then((data) => {
    console.log(data)
});
```

> ### Promise.reject

Promise.reject方法和Promise.resolve不同，Promise.reject()方法的参数，会原封不动地作为reject的理由，变成后续方法的参数。

```javascript
Promise.reject = function (reason) {
    return new Promise((resolve, reject) => {
        reject(reason);
    });
}
```

> ### Promise.prototype.catch

Promise.prototype.catch 用于指定出错时的回调，是特殊的then方法，catch之后，可以继续 .then

```javascript
Promise.prototype.catch = function (onRejected) {
    return this.then(null, onRejected);
}
```

> ### Promise.prototype.finally

不管成功还是失败，都会走到finally中,并且finally之后，还可以继续then。并且会将值原封不动的传递给后面的then.

```javascript
Promise.prototype.finally = function (callback) {
    return this.then((value) => {
        return Promise.resolve(callback()).then(() => {
            return value;
        });
    }, (err) => {
        return Promise.resolve(callback()).then(() => {
            throw err;
        });
    });
}
```

> ### Promise.all

Promise.all(promises) 返回一个promise对象

1. 如果传入的参数是一个空的可迭代对象，那么此promise对象回调完成(resolve),只有此情况，是同步执行的，其它都是异步返回的。
2. 如果传入的参数不包含任何 promise，则返回一个异步完成.
3. promises 中所有的promise都promise都“完成”时或参数中不包含 promise 时回调完成。
4. 如果参数中有一个promise失败，那么Promise.all返回的promise对象失败
5. 在任何情况下，Promise.all 返回的 promise 的完成状态的结果都是一个数组

```javascript
Promise.all = function (promises) {
    return new Promise((resolve, reject) => {
        let index = 0;
        let result = [];
        if (promises.length === 0) {
            resolve(result);
        } else {
            function processValue(i, data) {
                result[i] = data;
                if (++index === promises.length) {
                    resolve(result);
                }
            }
            for (let i = 0; i < promises.length; i++) {
                  //promises[i] 可能是普通值
                  Promise.resolve(promises[i]).then((data) => {
                    processValue(i, data);
                }, (err) => {
                    reject(err);
                    return;
                });
            }
        }
    });
}
```

测试代码:

```javascript
var promise1 = new Promise((resolve, reject) => {
    resolve(3);
})
var promise2 = 42;
var promise3 = new Promise(function(resolve, reject) {
  setTimeout(resolve, 100, 'foo');
});

Promise.all([promise1, promise2, promise3]).then(function(values) {
  console.log(values); //[3, 42, 'foo']
},(err)=>{
    console.log(err)
});

var p = Promise.all([]); // will be immediately resolved
var p2 = Promise.all([1337, "hi"]); // non-promise values will be ignored, but the evaluation will be done asynchronously
console.log(p);
console.log(p2)
setTimeout(function(){
    console.log('the stack is now empty');
    console.log(p2);
});
```

> ### Promise.race

Promise.race函数返回一个 Promise，它将与第一个传递的 promise 相同的完成方式被完成。它可以是完成（ resolves），也可以是失败（rejects），这要取决于第一个完成的方式是两个中的哪个。

如果传的参数数组是空，则返回的 promise 将永远等待。

如果迭代包含一个或多个非承诺值和/或已解决/拒绝的承诺，则 Promise.race 将解析为迭代中找到的第一个值。

```javascript
Promise.race = function (promises) {
    return new Promise((resolve, reject) => {
        if (promises.length === 0) {
            return;
        } else {
            for (let i = 0; i < promises.length; i++) {
                Promise.resolve(promises[i]).then((data) => {
                    resolve(data);
                    return;
                }, (err) => {
                    reject(err);
                    return;
                });
            }
        }
    });
}
```

测试代码:

```javascript
Promise.race([
    new Promise((resolve, reject) => { setTimeout(() => { resolve(100) }, 1000) }),
    undefined,
    new Promise((resolve, reject) => { setTimeout(() => { reject(100) }, 100) })
]).then((data) => {
    console.log('success ', data);
}, (err) => {
    console.log('err ',err);
});

Promise.race([
    new Promise((resolve, reject) => { setTimeout(() => { resolve(100) }, 1000) }),
    new Promise((resolve, reject) => { setTimeout(() => { resolve(200) }, 200) }),
    new Promise((resolve, reject) => { setTimeout(() => { reject(100) }, 100) })
]).then((data) => {
    console.log(data);
}, (err) => {
    console.log(err);
});
```

## 十、浏览器的Event Loop

### 1. 预备知识

> JavaScript的运行机制:

（1）所有同步任务都在主线程上执行，形成一个执行栈（execution context stack）。

（2）主线程之外，还存在"任务队列"(task queue)。只要异步任务有了运行结果，就在"任务队列"之中放置一个事件。

（3）一旦"执行栈"中的所有同步任务执行完毕，系统就会读取"任务队列"，看看里面有哪些事件。那些对应的异步任务，于是结束等待状态，进入执行栈，开始执行。

（4）主线程不断重复上面的第三步

概括即是: 调用栈中的同步任务都执行完毕，栈内被清空了，就代表主线程空闲了，这个时候就会去任务队列中按照顺序读取一个任务放入到栈中执行。每次栈内被清空，都会去读取任务队列有没有任务，有就读取执行，一直循环读取-执行的操作

> 一个事件循环中有一个或者是多个任务队列

> JavaScript中有两种异步任务:

1. 宏任务: script（整体代码）, setTimeout, setInterval, setImmediate, I/O, UI rendering
2. 微任务: process.nextTick（Nodejs）, Promises, Object.observe, MutationObserver;

### 2. 事件循环(event-loop)是什么？

主线程从"任务队列"中读取执行事件，这个过程是循环不断的，这个机制被称为事件循环。此机制具体如下:主线程会不断从任务队列中按顺序取任务执行，每执行完一个任务都会检查microtask队列是否为空（执行完一个任务的具体标志是函数执行栈为空），如果不为空则会一次性执行完所有microtask。然后再进入下一个循环去任务队列中取下一个任务执行。

> #### 详细说明:

1. 选择当前要执行的宏任务队列，选择一个最先进入任务队列的宏任务，如果没有宏任务可以选择，则会跳转至microtask的执行步骤。
2. 将事件循环的当前运行宏任务设置为已选择的宏任务。
3. 运行宏任务。
4. 将事件循环的当前运行任务设置为null。
5. 将运行完的宏任务从宏任务队列中移除。
6. microtasks步骤：进入microtask检查点。
7. 更新界面渲染。
8. 返回第一步。

**执行进入microtask检查的的具体步骤如下:**

1. 设置进入microtask检查点的标志为true。
2. 当事件循环的微任务队列不为空时：选择一个最先进入microtask队列的microtask；设置事件循环的当前运行任务为已选择的microtask；运行microtask；设置事件循环的当前运行任务为null；将运行结束的microtask从microtask队列中移除。
3. 对于相应事件循环的每个环境设置对象（environment settings object）,通知它们哪些promise为rejected。
4. 清理indexedDB的事务。
5. 设置进入microtask检查点的标志为false。

**需要注意的是:当前执行栈执行完毕时会立刻先处理所有微任务队列中的事件, 然后再去宏任务队列中取出一个事件。同一次事件循环中, 微任务永远在宏任务之前执行。**

图示:



![event-loop2](https://user-gold-cdn.xitu.io/2019/3/22/169a4038c4e156f0?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



### 3. Event-loop 是如何工作的？

先看一个简单的示例：

```javascript
setTimeout(()=>{
    console.log("setTimeout1");
    Promise.resolve().then(data => {
        console.log(222);
    });
});
setTimeout(()=>{
    console.log("setTimeout2");
});
Promise.resolve().then(data=>{
    console.log(111);
});
```

思考一下, 运行结果是什么？

运行结果为:

```
111
setTimeout1
222
setTimeout2
```

我们来看一下为什么？

我们来详细说明一下, JS引擎是如何执行这段代码的:

1. 主线程上没有需要执行的代码
2. 接着遇到setTimeout 0，它的作用是在 0ms 后将回调函数放到宏任务队列中(这个任务在下一次的事件循环中执行)。
3. 接着遇到setTimeout 0，它的作用是在 0ms 后将回调函数放到宏任务队列中(这个任务在再下一次的事件循环中执行)。
4. 首先检查微任务队列, 即 microtask队列，发现此队列不为空，执行第一个promise的then回调，输出 '111'。
5. 此时microtask队列为空，进入下一个事件循环, 检查宏任务队列，发现有 setTimeout的回调函数，立即执行回调函数输出 'setTimeout1',检查microtask 队列，发现队列不为空，执行promise的then回调，输出'222'，microtask队列为空，进入下一个事件循环。
6. 检查宏任务队列，发现有 setTimeout的回调函数, 立即执行回调函数输出'setTimeout2'。

再思考一下下面代码的执行顺序:

```javascript
console.log('script start');

setTimeout(function () {
    console.log('setTimeout---0');
}, 0);

setTimeout(function () {
    console.log('setTimeout---200');
    setTimeout(function () {
        console.log('inner-setTimeout---0');
    });
    Promise.resolve().then(function () {
        console.log('promise5');
    });
}, 200);

Promise.resolve().then(function () {
    console.log('promise1');
}).then(function () {
    console.log('promise2');
});
Promise.resolve().then(function () {
    console.log('promise3');
});
console.log('script end');
```

思考一下, 运行结果是什么？

运行结果为:

```javascript
script start
script end
promise1
promise3
promise2
setTimeout---0
setTimeout---200
promise5
inner-setTimeout---0
```

那么为什么？

我们来详细说明一下, JS引擎是如何执行这段代码的:

1. 首先顺序执行完主进程上的同步任务，第一句和最后一句的console.log
2. 接着遇到setTimeout 0，它的作用是在 0ms 后将回调函数放到宏任务队列中(这个任务在下一次的事件循环中执行)。
3. 接着遇到setTimeout 200，它的作用是在 200ms 后将回调函数放到宏任务队列中(这个任务在再下一次的事件循环中执行)。
4. 同步任务执行完之后，首先检查微任务队列, 即 microtask队列，发现此队列不为空，执行第一个promise的then回调，输出 'promise1'，然后执行第二个promise的then回调，输出'promise3'，由于第一个promise的.then()的返回依然是promise，所以第二个.then()会放到microtask队列继续执行，输出 'promise2';
5. 此时microtask队列为空，进入下一个事件循环, 检查宏任务队列，发现有 setTimeout的回调函数，立即执行回调函数输出 'setTimeout---0',检查microtask 队列，队列为空，进入下一次事件循环.
6. 检查宏任务队列，发现有 setTimeout的回调函数, 立即执行回调函数输出'setTimeout---200'.
7. 接着遇到setTimeout 0，它的作用是在 0ms 后将回调函数放到宏任务队列中，检查微任务队列，即 microtask 队列，发现此队列不为空，执行promise的then回调，输出'promise5'。
8. 此时microtask队列为空，进入下一个事件循环，检查宏任务队列，发现有 setTimeout 的回调函数，立即执行回调函数输出，输出'inner-setTimeout---0'。代码执行结束.

### 4. 为什么会需要event-loop?

因为 JavaScript 是单线程的。单线程就意味着，所有任务需要排队，前一个任务结束，才会执行后一个任务。如果前一个任务耗时很长，后一个任务就不得不一直等着。为了协调事件（event），用户交互（user interaction），脚本（script），渲染（rendering），网络（networking）等，用户代理（user agent）必须使用事件循环（event loops）。

最后有一点需要注意的是：本文介绍的是浏览器的Event-loop，因此在测试验证时，一定要使用浏览器环境进行测试验证，如果使用了node环境，那么结果不一定是如上所说。

类型转换

https://blog.csdn.net/one_and_only4711/article/details/6281581

js基本数据类型和引用数据类型

https://www.cnblogs.com/cxying93/p/6106469.html

作用域链，闭包

http://web.jobbole.com/90524/

http://web.jobbole.com/90640/?utm_source=blog.jobbole.com&utm_medium=relatedPosts

https://zhuanlan.zhihu.com/p/25855075

原型链

http://web.jobbole.com/91207/

this的指向，apply，call，bind

http://www.cnblogs.com/pssp/p/5216085.html

http://www.cnblogs.com/pssp/p/5787116.html

http://www.cnblogs.com/pssp/p/5781090.html

js事件委托

https://www.cnblogs.com/zhoushengxiu/p/5703095.html

浏览器事件循环

https://www.cnblogs.com/rubyxie/articles/9797564.html

vue面试问题

http://web.jobbole.com/95479/

async，await，promise

http://web.jobbole.com/95515/

ES6：

let，const

class，promise，module，proxy，set，map，

解构赋值，扩展运算符，Object.assign（对象合并）

ES7：

 async，await