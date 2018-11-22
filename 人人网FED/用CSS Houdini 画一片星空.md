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