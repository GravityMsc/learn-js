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

所以我们借助这个特性来做动画，先用SVG的rect画一条线，这条线具有宽度和位置的属性，导航栏元素li触发mouseenter的时候就相应地改变这个rect的x坐标（注意mouseenter和mouseover的区别，mouseenter是不会冒泡的，这里要用mouseenter，避免li的子元素冒泡上去）。由于触发只能是SVG的元素才有效，所以需要在每个li里面写一个SVG给盖住。