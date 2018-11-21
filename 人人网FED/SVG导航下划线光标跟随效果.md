之前看到一篇博文，介绍导航下划线跟随的效果，是用的CSS的hover结合CSS3的选择器做的，总感觉效果不太自然，所以我就想能不能用SVG来做这个效果，试了一下，还是可以的，不过要借助一点JS。先来看一下按照正常思路用JS应当怎么实现。

## 1、正常思路的实现

我的思路是这样的，在导航的下方画一条线，然后当鼠标hover的时候看它是在哪个item上面，然后改变下划线的位置，并给这个位置加一个transition的动画。html结构：

```html
<nav>
    <ul>
        <li id="item1">首页</li>
        <li id="item-2">产品</li>
        <li id="item-3">关于我们</li>
        <li id="item-4">帮助</li>
    </ul>
    <div class="underline"></div>
</nav>
```

动画的CSS：

```css
.underline {
    transition: transform 0.2s linear,
                width 0.2s linear;
}
```

因为不同导航宽度不一样，所以除了transform之外再加个width的动画。

然后监听导航的mouseover事件，在里面改变下划线的transform和width属性，激发transition动画：

```javascript
let $line = document.querySelector(".underline");
document.querySelector("nav").onmouseover = function(event) {
    let node = event.target;
    // 使用mouseover注意事件冒泡目标元素判断
    if (node.nodeName === "LI") {
        // 计算postion，参考jQuery的postion函数的实现
        // 它的实现比这个复杂很多，考虑的情况要多一些
        let left = node.getBoundingClientRect().left;
        if (node.offsetParent) {
            // BFC
            left -= node.offsetParent.getBoundingClientRect().left;
        }
        $line.style.transform = `translateX(${left}px)`;
        $line.style.width = node.clientWidth + "px";
    }
};
```

效果如下图所示（录屏的效果稍微差了点，实际上是挺好的）：

![下划线动画](https://user-gold-cdn.xitu.io/2018/3/31/1627bf1d4d43c06d?imageslim)

一个demo：[underline-move-js.html](https://fed.renren.com/html/demo/underline-move-js.html)。（手机可通过点击触发mouseover事件）

感觉这个的效果已经很好了，而且代码也不复杂，主要是position计算那里稍微复杂一点，如果没哟用jq的话。

用SVG的动画可以怎么实现呢？

## 2、SVG动画效果

SVG有一个animate标签，可以指定需要做动画的属性，还可以通过begin属性指定动画开始的时机，例如某个元素hover的时候才开始动画。如果用CSS的hover配合选择器话，它有一个缺点就是只能选相邻或者子元素，这个限制就导致我们只能把下划线放在导航栏元素里面，例如作为它的border-bottom，或者紧跟在所有元素的后面。而SVG没有这个限制，SVG动画的触发可以是页面任意SVG元素的任意事件。

所以我们借助这个特性来做动画，先用SVG的rect画一条线，这条线具有宽度和位置的属性，导航栏元素li触发mouseenter的时候就相应地改变这个rect的x坐标（注意mouseenter和mouseover的区别，mouseenter是不会冒泡的，这里要用mouseenter，避免li的子元素冒泡上去）。由于触发只能是SVG的元素才有效，所以需要在每个li里面写一个SVG给盖住：

```html
<li>首页
    <svg>
        <rect id="svgitem1" x="0" y="0" width="100%" height="100%" fill="transparent"></rect>
    </svg>
</li>
<li>产品
    <svg>
        <rect id="svgitem2" x="0" y="0" width="100%" height="100%" fill="transparent"></rect>
    </svg>
</li>
```

这里面每个svg元素都是用的一个透明的矩形rect铺满，每个rect都有一个id：svgitem1、svgitem2等。

然后再用一个独立的svg的rect画下划线：

```html
<nav>
    <ul><li>首页...</li></ul>
    <svg id="svg-underline" width="100%" height="1">
        <rect id="nav-underline" x="0" y="0" width="80" height="2" stroke="black" stroke-width="2"/>
    </svg>
</nav>
```

把这个svg绝对定位到导航栏nav第一个元素的下面：

```css
#svg-underline {
    position: absolute;
    left: 0;
    bottom: 0;
}
```

然后给这个svg添加动画元素：

```html
<svg id="svg-underline" width="100%" height="1">
    <rect id="nav-underline" x="0" y="0" width="80" height="2" stroke="black" stroke-width="2"/>
    <animate xlink:href="#nav-underline" attributeName="x" to="0" dur=".2s" begin="svgitem1.mouseenter" fill="freeze"></animate>
    <animate xlink:href="#nav-underline" attributeName="x" to="80" dur=".2s" begin="svgitem2.mouseenter" fill="freeze"></animate>
    <animate xlink:href="#nav-underline" attributeName="x" to="160" dur=".2s" begin="svgitem3.mouseenter" fill="freeze"></animate>
    <animate xlink:href="#nav-underline" attributeName="x" to="240" dur=".2s" begin="svgitem4.mouseenter" fill="freeze"></animate>
</svg>
```

其中attributeName指定要做动画的属性，这里为x坐标，还有from/to属性，表示从哪个值变到哪个值，这里没有from，就会使用当前的值，这里先指定固定的to，我们先假定导航是等宽的，每个为80px，所以to的值递增。dur表示动画的时间，这里为0.2s。fill="freeze"表示动画结束后停留在最后一帧。begin为动画的触发时机，每个animate元素的begin对应每个li里面svg元素，当li的svg触发mouseenter的时候就会开始相应的animate动画。而动画的效果是改变to属性指定的值，这样就实现了x坐标随着光标移动的动画。

缺点是需要写很多animate元素，但是一般我们用模板渲染，所以这个情况应该会好些。效果如下图所示：

![svg动画](https://user-gold-cdn.xitu.io/2018/3/31/1627bf1d4d37aa78?imageslim)

一个完整的demo：[underline-move-svg.html](https://fed.renren.com/html/demo/underline-move-svg.html)。

上面我们假定了导航是等宽的，但实际上导航往往是不等宽的，那怎么办呢？不等宽的情况基本只能借助JS计算元素宽度，然后动态地改变animate元素里面to的值，同时添加一个下划线宽度width的动画，如下示例：

```html
<animate xlink:href="#nav-underline" attributeName="x" to="0" dur=".2s" begin="svgitem1.mouseenter" fill="freeze"></animate>
<animate xlink:href="#nav-underline" attributeName="width" to="0" dur=".2s" begin="svgitem1.mouseenter" fill="freeze"></animate>
```

然后写一点JS改变一个这两个to的值：

```javascript
let $lis = document.querySelectorAll("nav li"),
    $animates = document.querySelectorAll("#svg-underline animate");
let widthSum = 0;
for (let i = 0; i < $lis.length; i++) {
    let width = $lis[i].clientWidth;
    $animates[i * 2].setAttribute("to", widthSum);
    $animates[i * 2 + 1].setAttribute("to", width);
    widthSum += width;
}
```

如果使用Vue/React等框架，可以在mounted的时候，动态地改变to绑定的值，这个代码也是挺简单的。

![导航不等宽的情况](https://user-gold-cdn.xitu.io/2018/3/31/1627bf1d4d2a4dbf?imageslim)

一个完整的demo：[underline-move-svg-2.html](https://fed.renren.com/html/demo/underline-move-svg-2.html)。

这个效果也很好，但是相对来说还是第1点正常思路的实现比较简单一点。有个问题就是重复进入同一个元素会重复动画，它的from还是上一次的值。

但是不管怎么样，当你发现用html/css不太好做动画时，可以往SVG动画的方向思考，SVG做动画还是很灵活的。上面那个例子其实不是很典型，再介绍另一个使用SVG做动画的例子。

## 3、SVG路径动画

