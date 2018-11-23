要问2018最让人兴奋的CSS技术是什么，CSS Houdini当之无愧，甚至可以去掉2018这个限定。其实这个技术在2016年就出来了，但是在今年3月发布的Chrome 65才正式支持。

CSS Houdini可以做些什么？[谷歌开发者文档](https://developers.google.com/web/updates/2016/05/houdini)列了几个demo，我们先来看一下这几个demo：

（1）给textarea加一个方格背景（[demo](https://fed.renren.com/html/houdini/houdini-samples/paint-worklet/checkerboard/index.html)）

![textarea的方格背景](https://user-gold-cdn.xitu.io/2018/4/22/162eb83af559da91?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

使用以下CSS代码：

```css
textarea {
    background-image: paint(checkerboard);
}
```

（2）给div添加一个钻石形状背景（[demo](https://fed.renren.com/html/houdini/houdini-samples/paint-worklet/diamond-shape/index.html)）

![div添加钻石形状背景](https://user-gold-cdn.xitu.io/2018/4/22/162eb83af578676b?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

使用以下CSS：

```css
div {
    --top-width: 80;
    --top-height: 20;
    -webkit-mask-image: paint(demo);
}
```

（3）点击圆圈扩散动画（[demo](https://fed.renren.com/html/houdini/houdini-samples/paint-worklet/ripple/index.html)）

![点击圆圈扩散动画](https://user-gold-cdn.xitu.io/2018/4/22/162eb83af6011574?imagesli)

这3个例子都是用了Houdini里面的CSS Paint API。

第1个例子如果使用传统的CSS属性，我们最多可能就是使用渐变来做颜色的变化，但是做不到这种一个格子一个格子的颜色变化的，而第2个例子也是没有办法直接用CSS画个钻石的形状。这个时候你可能会想到用SVG/Canvas的方法，SVG和Canvas的特色是矢量路径，可以画出各种各样的矢量图形，而Canvas还能控制任意像素点，所以用这两种方式也是可以画出来的。

但是Canvas和html相结合的时候就会显得有点笨拙，就像第2个例子画一个钻石形状，用Canvas你需要利用类似于BFC定位的方式，把Canvas调到合适的定位，还要注意z-index的覆盖关系，而使用SVG可能会更简单一点，可以设置background-image为一张钻石的svg图片，但是无法像Canvas一样很方便地做一些变量控制，例如随时改一下钻石边框的颜色粗细等。

而第1个例子给textarea加格子背景，只能使用background-image + svg的方式，但是你不知道这个textarea有多大，svg的格子需要准备多少个呢？当然你可能会说谁会给textarea加一个这样的背景呢，但这只是一个示例，其他的场景可能会遇到类似的问题。

第3个例子点击圆圈扩散动画，这个也可以在div里面absolute定位一个canvas元素，但是我们又遇到另外一个问题：无法很方便复用，假设这种圆圈扩散效果在其他地方也要用到，那就得在每个地方都写一个canvas元素并初始化。

所以传统的方式存在以下问题：

（1）需要调好和其他html元素的定位和z-index关系等

（2）编辑框等不能方便地改背景，不能方便地做变量控制

（3）不能方便地进行复用

其实还有另外一个更重要的问题就是新能问题，用Canvas

画这种效果时需要自己控制好频率，一不小心电脑CPU风扇可能就要呼啸起来，特别是不能把握重绘的时机，如果元素大小没有变化时不需要重绘，如果元素被拉大了，那么需要进行重绘，或者当鼠标hover的时候做动画才需要重绘。

CSS Houdini在解决这种自定义图形图像绘制的问题提供了很好的解决方案，可以用Canvas画一个你想要的图形，然后注册到CSS系统里面，就能在CSS属性里面使用这个图像了。以画一个星空为例，一步步说明这个过程。

## 1、画一个黑色的夜空

CSS Houdini只能工作在localhost域名或者是https的环境，否则的话相关API不可见（undefined）的。如果没有https环境的话，可以装一个http-server的npm包，然后在本地启动，访问localhost:8080就可以了，新建一个index.html，写入：

```html
<!DOCType>
<html>
<head>
    <meta charset="utf-8">
<style>
body {
    background-image: paint(starry-sky);
}
</style>    
</head>
<body>
<script>
    CSS.paintWorklet.addModule('starry-sky.js');
</script>
</body>
</html>
```

通过在JS调用CSS.paintWorklet.addModule注册一个CSS图像starry-sky，然后在CSS里面就可以使用这个图形，写在background-image、border-image或者mask-image等属性里面。如上面代码的：

```css
body {
    background-image: paint(starry-sky);
}
```

注册paint worklet的时候需要给它一个独立的js，作为这个worklet的工作环境，这个环境里面是没有window/document等对象的和web worker一样。如果你不想写管理太多js文件，可以借助blob，blob是可以存放任何类型的数据的，包括JS文件。

Worklet需要的starry-sky.js的代码如下所示：

```javascript
class StarrySky {
    paint (ctx, paintSize, properties) {
        // 使用Canvas的API进行绘制
        ctx.fillRect(0, 0, paintSize.width, paintSize.height);
    }
}
// 注册这个属性
registerPaint('starry-sky', StarrySky);
```

写一个类，实现paint接口，这个接口会传一个canvas的context变量、当前画布的大小即当前dom元素的大小，以及当前dom元素的css属性properties。

在paint函数里面调用canvas的绘制函数fillRect进行填充，默认填充色为黑色。访问index.html，就会看到整个页面变成黑色了。我们的Hello World的CSS Houdini Painter就跑起来了，没错，就是这么简单。

但是有一点需要强调的是，浏览器实现并不是给那个dom元素添加一个Canvas然后隐藏起来，整个Paint Worklet实际上是直接影响了当前dom元素重绘过程，相当于我们给它添加了一个重绘的步骤，下文会继续提及。

如果不想独立写一个js，用blob可以这样：

```javascript
let blobURL = URL.createObjectURL(new Blob(['(', function(){
    class StarrySky{
        paint(ctx, paintSize, properties) {
            ctx.fillRect(0, 0, paintSize.width, paintSize.height);
        }
    }
    registerPaint('starry-sky', StarrySky);
}.toString(),
')()'], {type: 'application/javascript'})
);

CSS.paintWorklet.addMoudule(blobURL);
```

## 2、画星星

Canvas星星效果网上找一个就好了，例如这个[Codepen](https://codepen.io/AlienPiglet/pen/hvekG)，代码如下：

```javascript
paint (ctx, paintSize, poperties) {
    let xMax = paintSize.width;
    let yMax = paintSize.height;
    
    // 黑色夜空
    ctx.fillRect(0, 0, xMax, yMax);
    
    // 星星的数量
    let hmTimes = xMax + yMax;
    for(let i = 0; i <= hmTimes; i++){
        // 星星的xy坐标，随机
        let x = Math.floor((Math.random() * xMax) + 1);
        ley y = Math.floor((Math.random() * yMax) + 1);
        // 星星的大小
        let size = Math.floor((Math.random() * 2) + 1);
        // 星星的亮暗
        let opacityOne = Math.floor((Math.random() * 9) + 1);
        let opacityTwo = Math.floor((Math.random() * 9) + 1);
        let hue = Math.floor((Math.random() * 360) + 1);
        ctx.fillStyle = `hsla(${hue}, 30%, 80%, .${opacityOne + opacityTwo})`;
        ctx.fillRect(x, y, size, size);
    }
}
```

效果如下：

![星空](https://user-gold-cdn.xitu.io/2018/4/22/162eb83af610dda9?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



为什么要用fillRect来画星星呢？星星不应该是圆的么？因为如果用arc的话性能会明显降低。由于星星比较小，所以使用了这种方式，当然改成arc也是可以的，因为我们只是画一次就好了。

## 3、控制星星的密度

现在要做一个可配参数来控制星星的密度，就好像border-radius可以控制一样。借助CSS变量，给body添加一个自定义属性--star-density：

```css
body {
    --star-density: 0.8;
    background-image: paint(starry-sky); 
}
```

规定密度系数从0到1变化，通过paint函数的properties参数获取到属性。但是我们发现body/html的自定义属性无法获取，可以继承给body的子元素，但无法在body上获取，所以改成画在body: bdfore上面：

```css
body:before {
    content: "";
    position: absolute;
    left: 0;
    top: 0;
    width: 100%;
    height: 100%;
    --star-density: 0.5;
    background-image: paint(starry-sky); 
}
```

然后给class StarrySky添加一个静态方法：

```javascript
class StarrySky {
    static get inputProperties() {
        return ['--star-density'];
    }
}
```

告知我们需要获取哪些CSS属性，可以是自定义的，也可以是常规的CSS属性。然后再paint方法的properties里面就可以拿到属性值：

```javascript
class StarrySky {
    paint (ctx, paintSize, properties) {
        // 获取自定义属性值
        let starDensity = +properties.get('--star-density').toString() || 1;
        // 最大只能为1
        starDensity > 1 && (starDensity = 1);
        // 星星的数量剩以这个系数
        let hmTimes = Math.round((xMax + yMax) * starDensity);
    }
}
```

让星星的数量乘以传进来的系数进而达到控制密度的目的。上面设置星星的数量为最大值的一半，效果如下：

![使用系数控制星星的密度](https://user-gold-cdn.xitu.io/2018/4/22/162eb83af545429d?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

## 3、重绘

当拉画面的时候会发现所有星星的位置都发生了变化，这是因为触发了重绘。

在paint函数里面添加一个console.log，拉动页面的时候就可以观察到浏览器在不断地执行paint函数。因为这个CSS属性是写在body: before上面的，占满了body，body大小改变就会触发重绘。而如果写在一个宽度固定的div里面。拉动页面不会触发重绘，观察到paint函数没有执行。如果改了div或者body的任何一个CSS属性也会触发重绘。所以这个很方便，不需要我们自己去监听resize之类的DOM变化。

页面拉大时，右边新拉出来的空间星星没有画大，所以本身需要重绘。而重绘给我们造成的问题是星星的位置发生变化，正常情况下应该是页面拉大拉小，星星的位置应该是要不变的。所以需要记录一下星星的一些相关信息。

## 4、记录星星的数据

可以在SkyStarry这个类里面添加一个成员变量stars，保存所有star的信息，包括位置和透明度等，在paint的时候判断一下stars的长度，如果为0则进行初始化，否则使用直接上一次初始化过的星星，这样就能保证每次重绘都是用的同样的星星了。但是在实际的操作过程中，发现一个问题，它会初始化两次starry-sky.js，在paint的时候也会随机切换，如下图所示：

![随机切换的问题](https://user-gold-cdn.xitu.io/2018/4/22/162eb83af5676890?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

这样就造成了有两个stars的数据，在重绘过程中来回切换。原因可能是因为CSS Houdini的本意并不想让你保存实例数据，但是既然它设计成一个类，使用类的实例数据应该也是合情合理的。这个问题我想到的一个解决方法是把random函数变成可控的，只要随机化种子一样，那么生成的random系列就是一样的，而这个随机化种子由CSS变量传进啦。所以就不能用Math.random了，自己实现一个random，[如下代码](https://stackoverflow.com/questions/521295/seeding-the-random-number-generator-in-javascript)所示：

```javascript
    random () {
        let x = Math.sin(this.seed++) * 10000;
        return x - Math.floor(x);
    }
```

只要初始化seed一样，那么就会生成一样的random系列。seed和星星密度类似，由CSS变量控制：

```css
body:before {
    --starry-sky-seed: 1;
    --star-density: 0.5;
    background-image: paint(starry-sky);
}
```

然后再paint函数里面通过properties拿到seed：

```javascript
paint (ctx, paintSize, properties) {
    if (!this.stars) {
        let starOpacity = +properties.get('--star-opacity').toString();
        // 得到随机化种子，可以不传，默认为0
        this.seed = +(properties.get('--starry-sky-seed').toString() || 0);
        this.addStars(paintSize.width, paintSize.height, starDensity);
    }
}
```

通过addStars函数添加星星，这个函数调用上面自定义的random函数：

```javascript
random () {
    let x = Math.sin(this.seed++) * 10000;
    return x - Math.floor(x);
}

addStars (xMax, yMax, starDensity = 1) {
    starDensity > 1 && (starDensity = 1); 
    // 星星的数量
    let hmTimes = Math.round((xMax + yMax) * starDensity);  
    this.stars = new Array(hmTimes);
    for (let i = 0; i < hmTimes; i++) {
        this.stars[i] = { 
            x: Math.floor((this.random() * xMax) + 1), 
            y: Math.floor((this.random() * yMax) + 1), 
            size: Math.floor((this.random() * 2) + 1), 
            // 星星的亮暗
            opacityOne: Math.floor((this.random() * 9) + 1), 
            opacityTwo: Math.floor((this.random() * 9) + 1), 
            hue: Math.floor((this.random() * 360) + 1)
        };  
    }
}
```

这段代码由Math.random改成this.random保证只要随机化种子一样，生成的数据也都是一样的。这样就能解决上面提到的初始化两次数据的问题，因为种子是一样的，所以两次的数据也是一样的。

但是这样有点单调，每次刷新页面星星都是固定的，少了点灵气。可以给这个随机化种子做下优化，例如实现单个小时内是一样的，过了一个小时后刷新页面就会变。通过以下代码可以实现：

```javascript
const ONE_HOUR = 36000 * 1000;
this.seed = +(properties.get('--starry-sky-seed').toString() || 0)
                    + Date.now() / ONE_HOUR >> 0;
```

这样拉动页面的时候星星就不会变了。

但是在从小拉大的时候，右边会没有星星：

![随机化种子一致，但是会导致拖动时没有及时填充新的星星](https://user-gold-cdn.xitu.io/2018/4/22/162eb83b21bc167c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

因为第一次的画布没有这么大，以后又没有更新星星的数据，所以右边就空了。

## 5、增量更新星星数据

不能全部更新星星的数据，不然第4步就白做了。只能把右边没有的给它补上。所以需要记录一下两次画布的大小，如果第二次的画布大了，则增加星星，否则删掉边界外的星星。

所以需要有一个变量记录上一次画布的大小：

```javascript
class StarrySky {
    constructor () {
        // 初始化
        this.lastPaintSize = this.paintSize = {
            width: 0,
            height: 0
        };
        this.stars = [];
    }
}
```

把相关的操作抽成一个函数，包括从CSS变量获取设置，增量更新星星等，这样可以让主逻辑变得清晰一点：

```javascript
paint (ctx, paintSize, properties) {
    // 更新当前paintSize
    this.paintSize = paintSize;
    // 获取CSS变量设置，把密度、seed等存放到类的实例数据
    this.updateControl(properties);
    // 增量更新星星
    this.updateStars();
    // 黑色夜空
    for (let star of this.stars) {
        // 画星星，略
    }   
}
```

增量更新星星需要做两个判断，一个为是否需要删除掉一些星星，另一个为是否需要添加，根据画布的变化：

```javascript
updateStars () {
    // 如果当前的画布比上一次的要小，则删掉一些星星
    if (this.lastPaintSize.width > this.paintSize.width ||
            this.lastPaintSize.height > this.paintSize.height) {
        this.removeStars();
    }   
    // 如果当前画布变大了，则增加一些星星
    if (this.lastPaintSize.width < this.paintSize.width ||  
            this.lastPaintSize.height < this.paintSize.height) {
        this.addStars();
    }   
    this.lastPaintSize = this.paintSize;
}
```

删除星星removeStar的实现很简单，只要判断x，y坐标是否在当前画布内，如果是的话则保留：

```javascript
removeStars () {
    let stars = []
    for (let star of stars) {
        if (star.x <= this.paintSize.width &&  
                star.y <= this.paintSize.height) {
            stars.push(star);
        }   
    }   
    this.stars = stars;
}
```

添加星星的实现也是类似的道理，判断x，y坐标是否在上一次的画布内，如果是的话则不添加：

```javascript
addStars () {
    let xMax = this.paintSize.width,
        yMax = this.paintSize.height;
    // 星星的数量
    let hmTimes = Math.round((xMax + yMax) * this.starDensity); 
    for (let i = 0; i < hmTimes; i++) {
        let x = Math.floor((this.random() * xMax) + 1), 
            y = Math.floor((this.random() * yMax) + 1); 
        // 如果星星落在上一次的画布内，则跳过
        if (x < this.lastPaintSize.width && y < this.lastPaintSize.height) {
            continue;
        }   

        this.stars.push({
            x: x,
            y: y,
            size: Math.floor((this.random() * 2) + 1), 
            // 星星的亮暗
        }); 
    }   
}
```

这样当拖动页面的时候就会触发重绘，重绘的时候就会调paint更新星星。

## 6、让星星闪起来

通过做星星透明度的动画，可以让星星闪起来。如果用Canvas标签，可以借助window.requestAnimationFrame注册一个函数，然后用当前时间减掉开始的时间模以一个值就得到当前的透明度系数。使用Houdini也可以使用这种方式，区别是我们可以把动态变化透明系数当作当前元素的CSS变量或者叫自定义属性，让后用JS动态改变这个自定义属性，就能够触发重绘，这个已经在第3点重绘部分提到。