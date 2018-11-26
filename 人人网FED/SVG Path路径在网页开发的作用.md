SVG是矢量图形表示，它的一个强大之处在于path标签可以表示任意的矢量形状，利用好这个path可以做出很多传统html/css做不出来的效果。下面举几个例子。

## 1、做路径动画

这个我在《[SVG导航下划线光标跟随效果](https://fed.renren.com/2018/03/31/svg-animate/#svg-move-by-path)》文后补充介绍了这个实现，最后的效果是这样的：

![路径动画](https://user-gold-cdn.xitu.io/2018/6/17/1640d6846e6c49da?imageslim)

实现代码如下：

![路径动画代码如下](https://user-gold-cdn.xitu.io/2018/6/17/1640d6847b35040e?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

利用animateMotion结合path做的动画，具体说明可见上文。

## 2、实现不规则形状的点击

如下图所示，需要实现点到哪个洲就进入哪个洲的效果，例如点到非洲就进入非洲：

![不规则形状的点击](https://user-gold-cdn.xitu.io/2018/6/17/1640d6847b72d249?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

我们可以用div定一个框覆盖在非洲的上面，但是用div的话只能是规则的四方形，没办法实现点到非洲大陆的时候才进入，但是大陆的轮廓又是不规则的，所以用传统html是不能解决这个问题的。但是用SVG的path可以解决这个问题，方法1是监听path的点击事件即可，如下图所示：

![SVG可以制作不规则的路径](https://user-gold-cdn.xitu.io/2018/6/17/1640d6847b2a71ce?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

因为这个轮廓可以跟UI要到，他们一般都是用AI/PS等矢量软件画出来的，让他们导出一个SVG给你就好了。

方法2是可以调用SVG的isPointInFill这个API判断点击的点是否在Path的fill区域里面，这个也可以实现，但是相对于方法1来说比较麻烦，因为还需要把鼠标的位置转换为SVG视图的位置。

## 3、沿着路径拖拽的交互

在第1点沿着路径的动画时自动的过程，有没有办法让用户自己拖拽过去呢，实现如下效果：

![路径拖拽](https://user-gold-cdn.xitu.io/2018/6/17/1640d684873bb4ee?imageslim)

这种的场景有音量控制等需要有百分比控制的。可以先用一个[SVG的在线工具](https://editor.method.ac/)画出一个这样的图形。

![先用工具画出图形](https://user-gold-cdn.xitu.io/2018/6/17/1640d6847b0eadf7?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

就可以拿到SVG的代码：

```html
<svg class="volumn-controller" width="580" height="400" xmlns="http://www.w3.org/2000/svg">
    <path class="volumn-path" stroke="#000" d="m100,247c93,-128 284,-129 388,6" opacity="0.5" stroke-width="1" fill="#fff"/>
    <circle class="drag-button" r="12" cy="247" cx="100" stroke-width="1" stroke="#000" fill="#fff"/>
 </g>
</svg>
```

这里比较关键的是path标签里的d属性，d是data的缩写，定义这个路径的形状，它里面可以用很多属性控制形状的变化，如下图所示：

![控制SVG中path的形状属性](https://user-gold-cdn.xitu.io/2018/6/17/1640d684a0edb38b?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

为了实现这个交互，需要动态地改变circle的圆心位置(cx, cy)到路径上相应的地方。SVG没有直接提供相关的API，但是它提供了一个可以间接利用的API叫getPointAtLength，传递一个长度参数，如下代码所示：

```javascript
let volumnPath = document.querySelector('.volumn-path');
// 输出path在长度为100的位置的点坐标
console.log(volumnPath.getPointAtLength(100));
// 输出当前path的总长度
console.log(volumnPath.getTotalLength());
```

控制台输出：

![路径长度](https://user-gold-cdn.xitu.io/2018/6/17/1640d684aaa4f086?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

把circle的cx/cy改成上面的x/y坐标，圆圈就会跑到对应的位置去了。

![更动坐标位置](https://user-gold-cdn.xitu.io/2018/6/17/1640d684acdf8b0d?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

这里的问题在于这个API传递的length参数是相对于曲线长度的，但是鼠标移动的位置是线性的，没办法直接指导当前鼠标在曲线上距离起始位置是多少。

所以需要计算一下，在这个场景里面我们可以取鼠标的x坐标在曲线上对应的位置就可以了，如下图示意：

![计算鼠标的x坐标在曲线上对应的位置](https://user-gold-cdn.xitu.io/2018/6/17/1640d684b978e1bc?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

到这里就有思路了，可以把这条路径上每隔一个像素长度就算一下它的坐标在哪里，然后存在一个数组里面。由于鼠标移动的时候x坐标是知道的，就可以查一下在这个数组里面相应的x坐标的y坐标是多少，就能得到想要的圆心位置了。

所以先计算一下，保存到一个数组：

```javascript
let $volumnController = document.querySelector('.volumn-controller'),
    $volumnPath = $volumnController.querySelector('.volumn-path');
// 得到当前路径的总长度
let pathTotalLength = $volumnPath.getTotalLength() >> 0;
let points = [];
// 起始位置为长度为0的位置
let startX = Math.round($volumnPath.getPointAtLength(0).x);
// 每隔一个像素距离就保存一下路径上点的坐标
for (let i = 0; i < pathTotalLength; i++) {
    let p = $volumnPath.getPointAtLength(i);
    // 保存的坐标用四舍五入，可以平衡误差
    points[Math.round(p.x) - startX] = Math.round(p.y);
}
```

这里用一个points数组来保存，它的索引index就为x坐标，值为y坐标。在这个例子里面，总长度为451.5px，得到的points数组长度为388，你可以隔0.5px长度就保存一个坐标，不过在这个例子里面1px就够了。

然后监听鼠标事件，得到x坐标，查询y坐标，动态地改变circle的圆心位置，如下代码所示：

```javascript
let $dragButton = $volumnController.querySelector('.drag-button'),
    // 得到起始位置相对当前视窗的位置，相当于jQuery.fn.offset
    dragButtonPos = $dragButton.getBoundingClientRect();
function movePoint (event) {
    // 当前鼠标的位置减去圆心起始位置就得到移位偏差，12是半径值，这里先直接写死
    let diffX = event.clientX - Math.round(dragButtonPos.left + 12);
    // 需要做个边界判断
    diffX < 0 && (diffX = 0);
    diffX >= points.length && (diffX = points.length - 1);
    // startX是在上面的代码得到的长度为0的位置
    $dragButton.setAttribute('cx', diffX + startX);
    // 使用points数组得到y坐标
    $dragButton.setAttribute('cy', points[diffX]);
}
$dragButton.addEventListener('mousedown', function (event) {
    document.addEventListener('mousemove', movePoint);
});
document.addEventListener('mouseup', function () {
    document.removeEventListener('mousemove', movePoint);
});
```

这个实现的代码也是比较简单，需要注意的地方是起始位置的选取，这里有两个坐标系，一个是相对页面的视窗的，它的原点(0, 0)坐标点是当前页面可视区域（client）的左上角，第二个坐标系是SVG的坐标系，它的原点(0, 0)位置是SVG画布的左上角，如下图所示：

![注意坐标系的选取](https://user-gold-cdn.xitu.io/2018/6/17/1640d684dadceac4?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

鼠标的位置是相对于视图client的，所以需要得到圆圈在client的位置，可以通过原生的getBoundingClient获取，然后用鼠标的clientX减掉圆圈的clientX就得到正确的位移偏差diff了，这个diff值加上圆圈的在SVG坐标的起始位置就能得到SVG里的x坐标了，然后去查一下points数组就能得到y坐标，然后去设置circle的cx/cy的值。

这个的实现已经算是十分简单的，大概30行代码。需要注意的是如果SVG缩放了，那么坐标也要相应比例地改一下。所以最好是不要缩放，1:1显示就简单多了。

如果要显示具体的音量值呢？这个也好办，只需要在第一步保存点坐标的时候把在路径上的长度也保存下来就好了，最后的效果如下：

![完整的demo](https://user-gold-cdn.xitu.io/2018/6/17/1640d684e3e3977d?imageslim)

一个完整的demo：[svg-path-drag.html](https://www.yinchengli.com/html/demo/svg-path-drag.html)。

如果路径比较复杂怎么办呢？一个x坐标可能会对应两个点，如下图所示：

![复杂路径](https://user-gold-cdn.xitu.io/2018/6/17/1640d684e41b8ab0?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

这个也是有办法的，计算的方法类似，也是需要把路径上所有每隔1px的点坐标都取出来，然后计算一下鼠标的位置距离哪个点的坐标最近，然后就取那个点就好了。当然在判断哪个点最优时，算法需要优化，不能直接一个for循环，具体可见这个[codepen](https://codepen.io/osublake/pen/YYemRa)。

## 4、路径的变形动画

路径结合关键帧可以做出一些有趣的效果，如这个[codepen](https://codepen.io/chriscoyier/pen/NRwANp)的示例：

![路径变形动画](https://user-gold-cdn.xitu.io/2018/6/17/1640d684e67c22b4?imageslim)

它的实现是hover的时候改变path的d值，然后做d的transition动画，如下代码：

```html
<svg viewBox="0 0 10 10" class="svg-1">
  <path d="M2,2 L8,8" />
</svg>
<style>
.svg-1:hover path {
  d: path("M8,2 L2,8");
}
path {
    transition: d 0.5s linear;
}
</style>
```

这种变形过渡动画时有条件的，就是它的路径数据格式是要一致的，有多少M/L/C属性都要保持一致，否则无法做变形动画。

## 5、结合clip-path做遮罩效果

使用CSS通常只能用border-radius做一些圆角的遮罩，即用border-radius结合overflow：hidden实现，但是使用clip-path + svg的路径能够做出任意形状遮罩，如下做一个心形的：

![任意形状遮罩](https://user-gold-cdn.xitu.io/2018/6/17/1640d684e5deb484?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

如下代码所示：

```html
<div style="width:200px;height:200px">
    <img src="photo.png" alt style="width:100%">
</div>
<style>
img {
    clip-path: url("#heart");
}
</style>
```

style里面的id: #heart是指向了一个SVG的clipPath，如下图所示：

```html
<svg xmlns="http://www.w3.org/2000/svg" width="0" height="0">
    <clipPath id="heart" clipPathUnits="objectBoundingBox">
        <path transform="scale(0.0081967, 0.0101010)" d="m61.18795,24.08746c24.91828,-57.29309 122.5489,0 0,73.66254c-122.5489,-73.66254 -24.91828,-130.95562 0,-73.66254z"/>
    </clipPath>
</svg>
```

为了让这个path刚好能撑起div容器宽度的100%，需要设置：

```javascript
clipPathUnits="objectBoundingBox"
```

这样会导致d属性里面的单位变成比例的0到1，所以需要把它缩小一下，原本的width是122，height是99，需要scale的值为(1 / 122, 1 /  99)。这样就达到100%占满的目的，如果一开始d属性坐标比例就是0到1，就不用这么搞了。

另外clip-path使用svg的path不支持变形动画。

本篇介绍了使用SVG路径path做的几种效果：做一个路径动画、不规则形状的点击、沿着路径拖拽、路径的变形动画以及和clip-path做一些遮罩效果。可以说SVG对的path效果还是很强大的，当你有些效果用html/css无法实现的时候，不妨往SVG的方向思考。