##1. CSS盒模型

对一个文档进行布局（laying out）的时候，浏览器渲染引擎会根据CSS-box模型将所有元素表示为一个矩形盒子。CSS决定这个盒子的大小，位置以及属性（颜色，背景，边框尺寸）。

每个盒子由content(内容)，padding（内边框区域），border（边框），margin（外边框区域）。

通常width，height，min-width，min-height，max-width，max-height都是指content的属性

通常在box-sizing为默认值为content-box，这个时候盒子模型呈现为W3C的标准盒子模型。

加入box-sizing设置为border-box，这个时候盒子模型呈现为IE的盒子模型，此时width和height是指

content +  padding + border的和。

box-sizing为CSS3属性，IE8+和其他现代浏览器支持。



另外浏览器的标准模式和怪异模式，标准模式时，IE6+和其他现代浏览器会以W3C盒子模型渲染。怪异模式下，IE中只有IE9+会用W3C盒子模型。

##2. JS获取宽高

通过JS获取盒模型对应的宽和高，有以下几种方法：

为了方便书写，以下用dom来表示获取的HTML的节点。

1. dom.style.width/height 

　　这种方式只能取到dom元素内联样式所设置的宽高，也就是说如果该节点的样式是在style标签中或外联的CSS文件中设置的话，通过这种方法是获取不到dom的宽高的。

2. dom.currentStyle.width/height 

　　这种方式获取的是在页面渲染完成后的结果，就是说不管是哪种方式设置的样式，都能获取到。

　　但这种方式只有IE浏览器支持。

3. window.getComputedStyle(dom).width/height

　　这种方式的原理和2是一样的，这个可以兼容更多的浏览器，通用性好一些。

4. dom.getBoundingClientRect().width/height

　　这种方式是根据元素在视窗中的绝对位置来获取宽高的

5. dom.offsetWidth/offsetHeight

　　这个就没什么好说的了，最常用的，也是兼容最好的。

##3. 边距重叠

首先你要知道什么情况下会触发：两个或多个毗邻的普通流中的块元素垂直方向上的 margin 会折叠：

1. 两个或多个说明其数量必须是大于一个，又说明，折叠是元素与元素间相互的行为，不存在 A 和 B 折叠，B 没有和 A 折叠的现象。

2. 毗邻是指没有被非空内容、padding、border 或 clear 分隔开，说明其位置关系。注意一点，在没有被分隔开的情况下，一个元素的 margin-top 会和它普通流中的第一个子元素(非浮动元素等)的 margin-top 相邻； 只有在一个元素的 height 是 “auto” 的情况下，它的 margin-bottom 才会和它普通流中的最后一个子元素(非浮动元素等)的 margin-bottom 相邻。

3. 垂直方向是指具体的方位，只有垂直方向的 margin 才会折叠，也就是说，水平方向的 margin 不会发生折叠的现象。



那么如何使元素上下margin不折叠呢：

1.浮动元素、inline-block 元素、绝对定位元素的 margin 不会和垂直方向上其他元素的 margin 折叠（注意这里指的是上下相邻的元素）

2.创建了块级格式化上下文的元素，不和它的子元素发生 margin 折叠（注意这里指的是创建了BFC的元素和它的子元素不会发生折叠）

我们都知道触发BFC的因素是float（除了none）、overflow（除了visible）、display（table-cell/table-caption/inline-block）、position（除了static/relative）

很明显大家可以看出来相邻元素不发生折叠的因素是触发BFC因素的子集，也就是说如果我为上下相邻的元素设置了overflow:hidden，虽然触发了BFC，但是上下元素的上下margin还是会发生折叠

##4. 元素隐藏

1. display:none;

   隐藏元素，不占网页中的任何空间，让这个元素彻底消失（看不见也摸不着）

2. overflow:hidden;

   让超出的元素隐藏，就是在设置该属性的时候他会根据你设置的宽高把多余的那部分剪掉

   我们都知道每个浏览器对代码的解析都不同，所以我们在做页面的时候会遇到很多bug，在IE里面如果内容的高度超过了该层的高度他会自动地撑开，但火狐等里面的高度是多高这层就只有这么大，内容的高即使超出了也不会影响你设置的高，在这个时候我们有的问题就可以用overflow：hidden；来解决，这是第一点，还有就是我们可以利用它做出很多hove效果

3. visibility:hidden;

   他是把那个层隐藏了，也就是你看不到它的内容但是它内容所占据的空间还是存在的。（看不见但摸得到）

##5. display的属性

我们先来看看常见的四种值：inline,block,inline-block，none.

1. block
   单独占一行，可以设置width,height，maigin四个方向，padding四个方向； 
   元素宽度在不设置的情况下，是它本身父容器的100%（和父元素的宽度一致），除非设定一个宽度；

2. inline
   多个内联元素占同一行，直到放不下才换行。设置width,height,margin-top,margin- bottom,padding-top,padding-bottom无效； 
   元素的宽度就是它包含的文字或图片的宽度，不可改变。

3. inline-block
   像inline一样放置(比如和它相邻的元素处在同一行)，像block一样表现。比如：input,select,img等。

```html
<span>This is a span here
    <span style="display: inline-block; border: solid red 1px; height: 70px; ">
        This is a<br />inline-block<br />element
    </span> 
    This is a span here.
</span>
```

即:和内联元素在同一行，但自身相当于块级元素，可以设置width,height等属性。

**适用情况**
不用添加浮动，多个block可以一行显示（比如导航栏）[不适用ie6/7]
使一个inline元素具有高宽边距等而它依然能够保持inline;
inline和inline-block出现的问题
水平呈现的元素间，换行显示或空格分隔的情况下会有间距

**解决方法** 
● 相邻inline-block元素不分行写，写在同一行并且中间无空格 
● select父元素使用font-size:0

4. none
   隐藏该区域，不占实际空间。但对后台来说真实存在，可以获取被隐藏的元素

**和visibility:hidden的区别** 
hidden占实际空间，其后的元素仍在该有的位置，而none后的元素占none原有的位置

5. flex

   flex是flexible box的缩写，意为“弹性布局”。需要要IE10+才支持此属性。

   设为flex布局之后，子元素的float、clear和vertical-align属性将失效。

   > flex中，最核心的概念就是容器和轴，所有的属性都是围绕容器和轴设置的。其中，容器分为父容器和子容器。轴分为主轴和交叉轴（主轴默认为水平方向，方向向右，交叉轴为主轴顺时针旋转90度）。

   在使用 flex 的元素中，默认存在两根轴：水平的主轴（main axis）和垂直的交叉轴（cross axis）
   主轴开始的位置称为 `main start`，主轴结束的位置称为 `main end`。
   同理，交叉轴开始的位置称为 `cross start`，交叉轴结束的位置称为 `cross end`。
   在使用 flex 的子元素中，占据的主轴空间叫做 `main size`，占据的交叉轴空间叫做 `cross size`。

   ![](https://user-gold-cdn.xitu.io/2017/8/20/7bc6f5073e763bdadba7b3d0b1b4165e?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

   **父容器属性**

   * flex-direction：主轴的方向。
   * flex-wrap：超出父容器的子容器的排列方式。
   * flex-row：flex-direction属性和flex-wrap属性的简写形式。
   * justify-content：子容器在主轴的排列方向。
   * align-items：子容器在交叉轴的排列方向。
   * align-content：多根轴线的对齐方式。

   **flex-direction属性**：row（默认值，主轴为水平方向，方向向右），row-reverse（水平方向，方向向左），column（垂直方向，起点在上，方向向下），column-reverse（垂直方向，起点在下，方向向上）

   **flex-wrap**：决定子容器在一条轴线上排不下时，如何换行。nowrap（默认，不换行），wrap（换行，第一行在上方），wrap-reverse（换行，第一行在下方）

   **justify-content**：定义了子容器在主轴上的对齐方式，flex-start（默认，左对齐），flex-end（右对齐），center（居中），space-between（两端对齐，项目之间的间隔都相等），space-around（每个项目的两侧的间隔相等。所以，项目之间的间隔比项目与边框的间隔大一倍）

   **align-items**：定义子容器在交叉轴上如何对齐，flex-start（交叉轴起点对齐），flex-end（交叉轴终点对齐），center（交叉轴中点对齐），baseline（项目的第一行文字基线对齐），stretch（默认，如果项目未设置高度或设为auto，将占满整个容器的高度）

   **align-content**：定义了多根轴线的对齐方式。若只有一根轴线，该属性无效。 flex-start（交叉轴起点对齐），flex-end（交叉轴终点对齐），center（交叉轴中点对齐），space-between（与交叉两端对齐，轴线之间的间隔平均分布），space-around（每根轴线两侧的间隔都相等。所以，轴线之间的间隔比轴线与边框的间隔大一倍），stretch（默认，轴线占满整个交叉轴）



   **子容器的属性**

   * order：子容器的排列顺序

   * flex-grow：子容器剩余空间的拉伸比例

   * flex-shrink：子容器超出空间的压缩比例

   * flex-basis：子容器在不伸缩情况下的原始尺寸

   * flex：子元素的 `flex` 属性是 `flex-grow`,`flex-shrink` 和  `flex-basis` 的简写

   * align-self

   **order**：定义项目的排列顺序。数值越小，排列越靠前，默认为0。

   **flex-grow**：定义容器的伸缩比例。按照该比例给子容器分配空间，默认为0。

   **flex-shrink**：定义了子容器弹性收缩的比例，默认是0。此属性要生效，父容器的flex-wrap属性须为nowrap。

   flex-basis：定义了子容器在不伸缩情况下的原始尺寸，主轴为横向时代表宽度，主轴为纵向时代表高度，默认为auto。

   **flex 属性**

   子元素的 `flex` 属性是 `flex-grow`,`flex-shrink` 和  `flex-basis` 的简写，默认值为 `0` `1` `auto`。后两个属性可选。

   该属性有两个快捷值：auto (1 1 auto) 和 none (0 0 auto)。

   **align-self 属性**

   子容器的 `align-self` 属性允许单个项目有与其他项目不一样的对齐方式，可覆盖父容器 `align-items` 属性。默认值为 `auto`，表示继承父元素的 `align-items`属性，如果没有父元素，则等同于 `stretch`。

   flex-start（交叉轴的起点对齐），flex-end（交叉轴的终点对齐），center（交叉轴的中点对齐），baseline（项目的第一行文字的基线对齐）

##6. CSS定位和浮动

CSS定位（position）有四种：默认定位（static）、相对定位（relative）、绝对定位（absolute）和固定定位（fixed）

**static**：默认值。没有定位，元素在正常的文档流中，top，right，bottom，left和z-index属性无效。

**relative**：生成相对定位的元素，通过top、bottom、left、right的位置相对于其正常位置进行定位，其中的相对指的是相对于元素在默认文档流中的位置。

​	**注意**：1. 元素偏移之后，它本来的默认文档流中占据的位置仍然存在，其紧挨其后的元素的定位依据它本来的位置定位。
2. 该元素偏移之后，可能存在覆盖其他元素的情况，可以使用z-index属性改变层叠情况。
    例如：![](https://images2015.cnblogs.com/blog/1088731/201701/1088731-20170106145415269-473284752.png)![](https://images2015.cnblogs.com/blog/1088731/201701/1088731-20170106142543097-1435490859.png)

    上图的第二个盒子相对于之前的位置向下平移了20px，向右平移了30px。
    ​    
    ![](https://images2015.cnblogs.com/blog/1088731/201701/1088731-20170106145533362-1743833215.png)![](https://images2015.cnblogs.com/blog/1088731/201701/1088731-20170106145555550-1682047357.png)
    ​ 
    想要第三个盒子展现在最上面，需要更改它的z-index，同时要更改定位属性，否则不能使用z-index属性。

**absolute**：生成绝对定位的元素，相对于static定位的第一个父元素进行定位。

​	**注意**：1. 绝对定位的元素已经脱离了文档流（正常文档流中不再保留它的位置），普通流中其他元素的布局就像此元素不存在一样。

2. 绝对定位的元素的位置是相对于最近的已定位的祖先元素，如果元素没有已定位的祖先元素，它的位置就相对于html。
3. 绝对定位的框可以覆盖页面的其他元素。

![](https://images2015.cnblogs.com/blog/1088731/201701/1088731-20170106151042972-620219825.png)

这种情况是离box2最近的父元素已定位的情况，如果离box2最近的父元素没有定位的话，示例如下：

![](https://images2015.cnblogs.com/blog/1088731/201701/1088731-20170106151107487-1349325815.png)

**fixed**：本质上是一种绝对定位，不为元素预留空间。通过指定相对于屏幕视窗（视口，大小用vm，vh表示）来指定元素的位置，且元素的位置在屏幕滚动时不会发生变化。应用于很多网站顶端的固定导航，下方的广告，或者右边的回到顶部div。

![](https://images2015.cnblogs.com/blog/1088731/201701/1088731-20170106152349534-426090169.png)

![](https://images2015.cnblogs.com/blog/1088731/201701/1088731-20170106152556784-105614574.png)



####absolute、relative、fixed偏移的参考点分别是什么？
absolute偏移的参考点是：相对于最近的已定位的父元素，如果没有，则相对于body元素；
relative偏移的参考点是：相对于元素在普通流中的原来位置；
fixed偏移的参考点是：相对于浏览器窗口。

####z-index 有什么作用? 如何使用?

z-index属性用于设置节点的堆叠顺序，拥有更高堆叠顺序的节点将显示在堆叠顺序较低的节点前面。

 使用方法：示例

![img](https://images2015.cnblogs.com/blog/1088731/201701/1088731-20170106154118769-1187322359.png)

1. z-index仅对定位元素有效（position:relative||absolute||fixed）;

2. z-index只可比较同级元素。



####position:relative和负margin都可以使元素位置发生偏移?二者有什么区别？

position:relative和负margin都可以使元素位置发生偏移，（都不会脱离文档流）二者的区别表现在：

​          1.负margin会使元素在文档流中的位置发生偏移，它会放弃偏移之前占据的空间，紧挨其后的元素会填充这部分空间；

​          2.相对定位后元素位置发生偏移，它仍会坚守原来占据的空间，不会让文档流的其他元素流入。

![img](https://images2015.cnblogs.com/blog/1088731/201701/1088731-20170106155039253-884611157.png)

 ![img](https://images2015.cnblogs.com/blog/1088731/201701/1088731-20170106155159191-1376817903.png)

####浮动元素有什么特征？对其他浮动元素、普通元素、文字分别有什么影响?

浮动元素的特征有：

　　　　　1.块元素优先在一排显示；

　　　　　2.内联元素支持宽高；

　　　　   3.无论是块元素还是内联元素，没有宽度时默认内容撑开宽度；

　　　　   4.脱离文档流；

　　　　　5.提升层级半级（z-index）

　　　造成的影响：

　　　　　对其他浮动元素的影响：后浮动的元素永不会超过先浮动元素的顶端。

　　　　　浮动的元素只会覆盖后面的块级元素，不影响前边的块级元素。

　　　　　对普通元素的影响：浮动元素会脱离正常的文档流，使得紧挨它的元素位置发生偏移，影响布局。

　　　　　对文字的影响：浮动元素向下延伸时，不会影响正常文本的显示，文本会相对于浮动元素进行偏移。但部分文本背景会被浮动元素遮住。

​                 ![img](https://images2015.cnblogs.com/blog/1088731/201701/1088731-20170106161752191-1450674854.png)![img](https://images2015.cnblogs.com/blog/1088731/201701/1088731-20170106161823769-36206330.png)

**浮动造成的影响**

* 首先元素浮动会影响跟他同级元素的布局
* 其次会影响父元素容器的高度，正常父元素的高度时自适应的，高度为其包含的内容总高度，而内部元素的浮动会造成父容器高度塌陷（因为元素浮动之后，父元素就不会计算浮动元素的高度了）
* 父容器高度塌陷了，将会影响和父元素同级的文档布局

怎么解决这个问题呢？

这个问题其实分为两个部分：外部矛盾（即父元素的问题）和内部矛盾（元素本身和同级的元素的问题）。

**解决外部矛盾**

* 第一个是触发bfc，触发bfc后，父元素的高度会包括浮动元素的高度。触发的方式是给父元素设置overflow: auto即可。触发bfc（块级格式上下文）的方法有：1. float的值不是默认值none； 2. position的值不是static或者relative； 3. display的值是inline-block、table-cell、flex、table-caption或者inline-flex； 4. overflow的值不是visible。

* 第二个解决外部矛盾的方法是给父元素增加伪元素:after，给伪元素设置样式clear: both来清除浮动，伪元素清除浮动的核心原理其实是在给父元素增加块级容器，同时对块级容器设置clear属性，使其能够清除自身的浮动，从而正常按照块级容器排列方式那样排列在浮动元素的下面。同时，父元素的同级元素也会正常排列在伪元素形成的块级元素后面，而不受浮动影响。

  给父元素增加一个伪元素来清除浮动的本质，是通过：给父元素再加一个块级子容器，当然这个也就是父元素的最后一个块级子容器了。同时给这个块级子容器设置clear属性来清除其浮动，这样这个子容器就能排列在浮动元素的后面，同时也把父元素的高度撑起来了。那么父元素的同级元素也能正常排列了。所以这个子容器不能有高度和内容，不然会影响父元素的布局。

**解决内部矛盾**

其实，解决内部矛盾的原理和解决外部矛盾的第二种方式的原理是一样的，通过给被浮动影响的第一个元素进行清除浮动，就可以使后面的元素也不会受到浮动影响了。如果在这后面再增加一个浮动元素，可以在此浮动元素后面第一个受影响的元素清除浮动，照此类推。

####清除浮动的方法

　　如果一个父盒子中有一个子盒子，并且父盒子没有设置高，子盒子在父盒子中进行了浮动，那么将来父盒子的高度为0.由于将来父盒子的高度为0，下面的元素会自动补位，所以这个时候要进行浮动的清除。clear:both

　　1.使用额外标签法：

　　在浮动的盒子之下再放一个标签，在这个标签中使用clear:both,来清除浮动对页面的影响。

```html
.clearfix {clear:both;}

<div class="clearfix"></div>
```



　　a.内部标签：会将这个浮动盒子的父盒子高度重新撑开

　　b.外部标签：会将这个浮动盒子的影响清除，但是不会撑开盒子。

　　注意：一般情况下不会使用这一种方式来清除浮动。因为这个清除浮动的方式会增加页面的标签。

　　2.使用overflow属性来清除浮动：

　　先找到浮动盒子的父元素，再在父元素中添加一个属性，就是清除这个父元素中的子元素浮动对页面的影响。

　　overflow：hidden；

　　3.使用伪元素(给父元素)来清除浮动：

```css
.clearfix::after {
　　　　       content: "";//添加内容为空
              height: 0;//内容高度为0
              line-height: 0;//内容文本的高度为0
              display: block;//将文本设置为块级元素
              clear: both;//清除浮动
              visibility: hidden;//将元素隐藏
           }
.clearfix {
   zoom: 1;/*为了兼容ie6*/
}
```

##7. 常用CSS布局

####水平垂直居中（感觉总结的并不是很好）

感觉垂直居中真的是已经被讲烂了，但是在平时做项目时，我都是靠试的，导致面试的时候被问到有时候回答不上来，现在就用自己的方式来总结一下。其实这个方式是有很多的，但就是看的教程太多了，导致最后一个都没有记住，**所以我决定尽可能的将情况考虑完整，然后每一种情况只记住一个最佳实践。**

对于居中，我个人认为不需要背什么“x 种方式实现 xx”这样的例子，我们只需要了解其原理即可写出符合要求的 css。

水平、垂直居中，个人比较喜欢用绝对定位的方法实现，其次就是使用 `table` 布局，因为自带垂直居中。如果是单行的行内元素使用 `line-height` 等于 `height`，对于多行元素的垂直居中，大部分都是使用 `table` 元素（求推荐更好的布局），当然还有 flex 和 grid 布局。

####水平居中

一般水平居中还是比较容易的，我一般都是先看子元素是固定宽度还是宽度未知

##### 固定宽度

这种方式是绝对定位居中，除了使用 `margin`，我们还可以使用 `transform`（注意浏览器兼容性，只适用于 ie9+，移动开发请忽略）

```css
        .container{
            width: 300px;
            height: 200px;
            background: pink;
            position: relative;
        }
        .inner{
            width: 100px;
            height: 50px;
            position: absolute;
            top: 50%;
            left: 50%;
            margin-top: -25px;
            margin-left: -50px;
            background: #fff;
            text-align: center;
        }
        .container{
            width: 300px;
            height: 200px;
            background: pink;
            position: relative;
        }
        .inner{
            width: 100px;
            height: 50px;
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background: #fff;
            text-align: center;
        }
```

或者使用 `magin:0 auto`；但一般情况下我都会使用上一种，因为习惯了(ಥ_ಥ)

##### 宽度未知

将子元素设置为行内元素，然后父元素设置 `text-align: center`。

```css
        .container{
            width: 300px;
            height: 200px;
            background: pink;
            position: relative;
            text-align: center;
        }
        .inner{
            display: inline-block;

        }
```

##### 多个块状元素

上面的方式即使子元素不止一个也想实现水平居中也是有效的，（宽度固定不固定都可，不固定的话就不需要设置宽度，会被自动撑开，但是要考虑到撑爆的情况）例如：

```css
        .container{
            width: 250px;
            height: 200px;
            background: pink;
            position: relative;
            text-align: center;
            padding: 20px;
        }
        .inner{
            display: inline-block;
            width: 50px;
            height: 150px;
            margin: 0 auto;
            background: #fff;
            text-align: center;
        }
```



![多个子元素水平居中](https://user-gold-cdn.xitu.io/2017/8/20/666841916ea971f4705b439282c86658?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)多个子元素水平居中



当然也可以使用我们刚刚介绍的 flex，我们只需要让子元素在主轴上的对齐方式设置为居中就可以

```css
        .container{
            width: 250px;
            height: 200px;
            background: pink;
            display: flex;
            justify-content: center;
            padding: 20px;
        }
        .inner{
            background: #fff;
            width: 50px;
            height: 150px;
            margin-left: 10px;
        }
```

#### 垂直居中

##### 单行行内元素

单行行内元素居中，只需要将子元素的行高等于高度就可以了。

```css
        #container {
            height: 400px;
            background: pink;
        }
        #inner{
            display: inline-block;
            height: 200px;
            line-height: 200px;
        }
```

##### 多行元素

上面的这种方式只能处理单行的行内元素，对于多行的行内元素是处理不了的，因为给每一个子元素都设置了 `line-height`，看了很多方法，要不是没有效果，要不然就是又局限性，提到最多的是使用 `table-cell` 的方式（但是貌似这个方法也有一点弊端，那就是其子元素的表现形式和行内元素类似，子元素不能独占一行），**当然如果你有更好的方式，欢迎提出**：

```css
        .container {
            width: 200px;
            height: 400px;
            background: pink;
            position: absolute;
            display: table;
            vertical-align:middle;
        }

        .inner{
            display: table-cell;
            vertical-align:middle;
        }
```

还有一个方法是设置一个空的行内元素，使其 `height:100%`，`display:inline-block`,`vertical-align: middle;` 并且 `font-size:0`。但是这样方式的原理我还不是很清楚，只是知道要设置一个空元素，高度和父元素相等，并且设置垂直居中的属性。但是，这只是用与所有的行内元素的宽度和不超过父元素的宽度的情况。

```html
<div class="container">
    <img src="WechatIMG110.jpg" width="50px" />
    <img src="WechatIMG110.jpg" width="50px" />
    <img src="WechatIMG110.jpg" width="50px" />
    <p>123</p>
</div>
```

```css
.container{
            width: 400px;
            height: 100px;
            background: pink;
            text-align: center;

        }
        img{
            vertical-align: middle;

        }
        p{
            display: inline-block;
            height: 100%;
            line-height: 100%;
            vertical-align: middle;
            font-size: 0;
        }
```

效果：

![多行行内元素](https://user-gold-cdn.xitu.io/2017/8/20/732fedf7ebd5eeb817f9a522bbdff6ea?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)多行行内元素

另一个一劳永逸的方法就是 flex，但是要注意浏览器的兼容性。



####图片和文字垂直居中

经常有看到设计稿是图片和文字垂直居中的，那么怎么才能让图片和文字垂直居中呢？
只需要给图片一个 `vertical-align: middle;` 属性就可以：

```html
<div class="container">
    <img src="WechatIMG110.jpg" height="260" />
    <p>123456</p>

</div>
```

```css
.container {
            background: pink;
            padding: 20px;
            height: 400px;

        }
        .container img{
            display: inline-block;
            vertical-align: middle;
        }

        .container p{
            display: inline-block;

        }
```

效果如下：

![图片和文字垂直居中](https://user-gold-cdn.xitu.io/2017/8/20/88386947f106680912f2b889917891a6?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)图片和文字垂直居中



####重点来啦~

自我感觉总结的居中问题不是特别的好，我们现在可以来总结一下其中的原理：

- 我们要实现水平或者垂直居中，应该从两方面下手：元素自带居中的效果或者强制让其显示在中间。
- 所以我们先考虑，哪些元素有自带的居中效果，最先想到的应该就是 `text-align:center` 了，但是这个只对行内元素有效，所以我们要使用 `text-align:center` 就必须将子元素设置为 `display: inline;` 或者 `display: inline-block;`；
- 接下来我们可能会想既然有 `text-align` 那么会不会对应也有自带垂直居中的呢，答案是有的 `vertical-align:`，我一直不是很喜欢使用这个属性，因为十次用，9.9 次都没有垂直居中，一度让我怀疑人生。现在貌似也搞得不是很清楚，看了 [张鑫旭的文章](https://link.juejin.im?target=http%3A%2F%2Fwww.zhangxinxu.com%2Fwordpress%2F2010%2F05%2F%25E6%2588%2591%25E5%25AF%25B9css-vertical-align%25E7%259A%2584%25E4%25B8%2580%25E4%25BA%259B%25E7%2590%2586%25E8%25A7%25A3%25E4%25B8%258E%25E8%25AE%25A4%25E8%25AF%2586%25EF%25BC%2588%25E4%25B8%2580%25EF%25BC%2589%2F) 居然看得也不是很懂，笑哭。目前就在 `table` 中设置有效，因为 `table` 元素 的特性，打娘胎里面带的就是好用。还有一种可以有效的方式是前面提到的空元素的方式，不过感觉多设置一个元素还不如使用 `table`。
- 还有一只设置垂直居中的是将行内元素的 `line-height` 和 `height` 设置为相同（只适用于单行行内元素）
- 固定宽度或者固定高度的情况个人认为设置水平垂直居最简单，可以直接使用绝对定位。使用绝对定位就是子元素相对于父元素的位置，所以将父元素设置 `position:reletive` 对应的子元素要设置 `position:absolute`，然后使用 `top:50%;left:50%`，将子元素的左上角和父元素的中点对齐，之后再设置偏移 `margin-top: 1/2 子元素高度;margin-left: 1/2 子元素宽度;`。这种方式也很好理解。
- 上面的绝对定位方法只要将 `margin` 改为 `transform` 就可以实现宽度和高度未知的居中（兼容性啊兄弟们！(ಥ_ಥ)）`transformX:50%;transformY:50%`；

不行，感觉总结的还是很渣，╮(╯▽╰)╭哎，谁有好的方法，求推荐。

#### 圣杯布局

其实我还真是第一次听说圣杯布局这种称呼，看了下这个名字的由来，貌似和布局并没有什么关系，圣杯布局倒是挺常见的三栏式布局。两边定宽，中间自适应的三栏布局。
效果如下：

![圣杯布局](https://user-gold-cdn.xitu.io/2017/8/20/00913c0a5f49e94f13dd4a0bf5eea826?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)圣杯布局



这个布局方式的关键是怎么样才能使得在伸缩浏览器窗口的时候让中间的子元素宽度改变。可以适应浏览器的宽度变化使用百分比设置宽度再合适不过，所以我们要将中间子元素的宽度设置为 `100%`，左边和右边的子元素设置为固定的宽度。

我们就来实现一下这样的布局：

##### 给出HTML结构

HTML 文件就很普通：

```html
<div class="container">
    <div class="middle">测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试</div>
    <div class="left">left</div>
    <div class="right">right</div>
</div>
```

**这里我们要注意的是，中间栏要在放在文档流前面以优先渲染。**

##### 给出每个子元素的样式

然后我们写 CSS，我们现将其三个元素的宽度和高度设置好，然后都设置为 `float:left`:

```css
        .middle{
            width: 100%;
            background: paleturquoise;
            height: 200px;
            float: left;
        }
        .left{
            background: palevioletred;
            width: 200px;
            height: 200px;
            float: left;
            font-size: 40px;
            color: #fff;
        }
        .right{
            width: 200px;
            height: 200px;
            background: purple;
            font-size: 40px;
            float: left;
            color: #fff;
        }
```

这时的效果如下：

![圣杯布局](https://user-gold-cdn.xitu.io/2017/8/20/42def5aa9d3b4df14c230506074b7ee0?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)圣杯布局



##### 使子元素在同一行显示

我们可以看出，现在三个子元素是在一排显示的，因为我们给中间的子元素设置的宽度是 `100%`，并且中间的子元素在文档流的最前面，最先被渲染。

那么我们要使得三个元素在同一排显示。接下来我们要将 `.left` 和 `.right` 向上提。实际上我们是使用 `margin-left` 为 负值来实现的，我们将 `.left` 的 `margin-left` 设置为 `-100%`（负的中间子元素的宽度），这样，左边的元素就会被“提升”到上一层。

然后就是右边子元素了，只需要设置 `margin-left` 设置为负的自身的宽度。

结果如下：

![这里写图片描述](https://user-gold-cdn.xitu.io/2017/8/20/929b5b13e3a9396bc4b43c8daaf275c4?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)这里写图片描述



##### 使得中间子元素不被遮盖

从上一张截图显示中显示中间的子元素被遮挡了，所以说我们要解决这个问题，要怎么解决呢？嗯... 只要使得中间的子元素显示的宽度刚好为左边元素和右边元素显示中间的宽度就可以。同时我们还必须保证是使用的半分比的布局方式。

这样的话有一种方式可以即使中间的宽度减少，又可以使中间的宽度仍然使用 `100%`，那就是设置父元素的 `padding` 值，将父元素的 `padding-left` 设置为左边子元素的宽度，将父元素的 `padding-right` 设置为右边子元素的宽度。

显示效果如下：

![圣杯布局](https://user-gold-cdn.xitu.io/2017/8/20/f4fbde95e4aa19f9ac021138e0106fdf?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)圣杯布局



##### 将左边和右边的子元素向两边移动

嗯... 这貌似也不是我们想要的效果，但是，中间的子元素确实是在中间了，那么我们只需要设置相对位置，将左边的子元素和右边的子元素向两边移动就好。

最终的 CSS 代码如下：

```css
        .container{
            padding: 0 200px;
        }
        .middle{
            width: 100%;
            background: paleturquoise;
            height: 200px;
            float: left;
        }
        .left{
            background: palevioletred;
            width: 200px;
            height: 200px;
            float: left;
            font-size: 40px;
            color: #fff;
            margin-left:-100%;


        }
        .right{
            width: 200px;
            height: 200px;
            background: purple;
            font-size: 40px;
            float: left;
            color: #fff;
            margin-left:-200px;

        }
```

最终效果如下：

![圣杯布局](https://user-gold-cdn.xitu.io/2017/8/20/5cca9436df9669d9abebdf7f8fd49c94?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)圣杯布局



#### 双飞翼布局

其实双飞翼布局是为了解决圣杯布局的弊端提出的，如果你跟我一起将上面的圣杯布局的代码敲了一遍，你就会发现一个问题，当你将浏览器宽度缩短到一定程度的时候，会使得中间子元素的宽度比左右子元素宽度小的时候，这时候布局就会出现问题。所以首先，这提示了我们在使用圣杯布局的时候一定要设置整个容器的最小宽度。



![圣杯布局弊端](https://user-gold-cdn.xitu.io/2017/8/20/f83bca0c0e71b7dfb497853f0f5035ad?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)圣杯布局弊端



##### 双飞翼和圣杯布局区别

> 圣杯布局和双飞翼布局解决问题的方案在前一半是相同的，也就是三栏全部float浮动，但左右两栏加上负margin让其跟中间栏div并排，以形成三栏布局。
>
> 不同在于解决”中间栏div内容不被遮挡“问题的思路不一样：圣杯布局，为了中间div内容不被遮挡，将中间div设置了左右padding-left和padding-right后，将左右两个div用相对布局position: relative并分别配合right和left属性，以便左右两栏div移动后不遮挡中间div。
>
> 双飞翼布局，为了中间div内容不被遮挡，直接在中间div内部创建子div用于放置内容，在该子div里用margin-left和margin-right为左右两栏div留出位置。
>
> 作者：知乎用户
> 链接：[www.zhihu.com/question/21…](https://link.juejin.im?target=https%3A%2F%2Fwww.zhihu.com%2Fquestion%2F21504052%2Fanswer%2F50053054)
> 来源：知乎
> 著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

所以只是一个小小的改动

```html
<div class="container">
    <div class="middle-container">
        <div class="middle">测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试测试</div>

    </div>
    <div class="left">left</div>
    <div class="right">right</div>
</div>
```

```css
.middle-container{
            width: 100%;
            background: paleturquoise;
            height: 200px;
            float: left;

        }

        .middle{
            margin-left: 200px;
            margin-right: 200px;
        }
        .left{
            background: palevioletred;
            width: 200px;
            height: 200px;
            float: left;
            font-size: 40px;
            color: #fff;
            margin-left:-100%;


        }
        .right{
            width: 200px;
            height: 200px;
            background: purple;
            font-size: 40px;
            float: left;
            color: #fff;
            margin-left:-200px;

        }
```

这样，在我们将中间元素宽度调到比两边元素小的时候，也是可以正常显示，但是如果总宽度小于左边元素或者右边元素的时候，还是会有问题。

![双飞翼布局](https://user-gold-cdn.xitu.io/2017/8/20/61d393d131af1b12351d76dd2e2c288c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

##8. div水平垂直居中

**方案一：**

div绝对定位水平垂直居中【margin:auto实现绝对定位元素的居中】，

兼容性：,IE7及之前版本不支持

```css
　　　　div{
            width: 200px;
            height: 200px;
            background: green;
            position:absolute;
            left:0;
            top: 0;
            bottom: 0;
            right: 0;
            margin: auto;
        }
```

 

**方案二：**

div绝对定位水平垂直居中【margin 负间距】     这或许是当前最流行的使用方法。

```css
         div{
            width:200px;
            height: 200px;
            background:green;
            position: absolute;
            left:50%;
            top:50%;
            margin-left:-100px;
            margin-top:-100px;
        }        
```

 

**方案三：**

div绝对定位水平垂直居中【Transforms 变形】

兼容性：IE8不支持；

```css
        div{
            width: 200px;
            height: 200px;
            background: green;
            position:absolute;
            left:50%;    /* 定位父级的50% */
            top:50%;
            transform: translate(-50%,-50%); /*自己的50% */
        }            
```

 

**方案四：**

css不定宽高水平垂直居中

```css
　　　　　.box{

             height:600px;
             display:flex;
             justify-content:center;
             align-items:center;
               /* aa只要三句话就可以实现不定宽高水平垂直居中。 */
        }
        .box>div{
            background: green;
            width: 200px;
            height: 200px;
        }
```



**方案五：**

将父盒子设置为table-cell元素，可以使用text-align:center和vertical-align:middle实现水平、垂直居中。比较完美的解决方案是利用三层结构模拟父子结构

```html
<p class="outerBox tableCell">
  </p><p class="ok">
    </p><p class="innerBox">tableCell</p>
  <p></p>
<p></p>
```

```css
/*
table-cell实现居中
将父盒子设置为table-cell元素，设置
text-align:center,vertical-align: middle;
子盒子设置为inline-block元素
*/
.tableCell{
  display: table;
}
.tableCell .ok{
  display: table-cell;
  text-align: center;
  vertical-align: middle;
}
.tableCell .innerBox{
  display: inline-block;
}
```



**方案六：**

对子盒子实现绝对定位，利用calc计算位置

```css
<p class="outerBox calc">
    </p><p class="innerBox">calc</p>
<p></p>


/*绝对定位，clac计算位置*/
.calc{
  position: relative;
}
.calc .innerBox{
  position: absolute;
  left:-webkit-calc((500px - 200px)/2);
  top:-webkit-calc((120px - 50px)/2);
  left:-moz-calc((500px - 200px)/2);
  top:-moz-calc((120px - 50px)/2);
  left:calc((500px - 200px)/2);
  top:calc((120px - 50px)/2);
}
```

##9. 七种实现左侧固定，右侧自适应两栏布局的方法

总结一下左边固定，右边自适应的两栏布局的七种方法。其中有老生常谈的`float`方法,BFC方法，也有CSS3的`flex`布局与`grid`布局。并非所有的布局都会在开发中使用，但是其中也会涉及一些知识点。关于最终的效果，可以查看[这里](https://zhuqingguang.github.io/vac-works/cssLayout/index1.html)

常用的宽度自适应的方法通常是利用了`block`水平的元素宽度能随父容器调节的流动特性。另外一种思路是利用CSS中的`calc()`方法来动态设定宽度。还有一种思路是，利用CSS中的新型布局`flex layout`与`grid layout`。

首先创建基本的HTML布局和最基本的样式。

```html
<div class="wrapper" id="wrapper">
  <div class="left">
    左边固定宽度，高度不固定 </br> </br></br></br>高度有可能会很小，也可能很大。
  </div>
  <div class="right">
    这里的内容可能比左侧高，也可能比左侧低。宽度需要自适应。</br>
    基本的样式是，两个div相距20px, 左侧div宽 120px
  </div>
</div>
```

基本的样式是，两个盒子相距`20px`, 左侧盒子宽 `120px`，右侧盒子宽度自适应。基本的CSS样式如下:

```css
.wrapper {
    padding: 15px 20px;
    border: 1px dashed #ff6c60;
}
.left {
    width: 120px;
    border: 5px solid #ddd;
}
.right {
    margin-left: 20px;
    border: 5px solid #ddd;
}
```

下面的代码都是基于这套基本代码做覆盖，通过给容器添加不同的类来实现效果。

####双inline-block方案

```css
.wrapper-inline-block {
    box-sizing: content-box;
    font-size: 0;    // 消除空格的影响
}

.wrapper-inline-block .left,
.wrapper-inline-block .right {
    display: inline-block;
    vertical-align: top;    // 顶端对齐
    font-size: 14px;
    box-sizing: border-box;
}

.wrapper-inline-block .right {
    width: calc(100% - 140px);
}
```

这种方法是通过`width: calc(100% - 140px)`来动态计算右侧盒子的宽度。需要知道右侧盒子距离左边的距离，以及左侧盒子具体的宽度(content+padding+border)，以此计算父容器宽度的`100%`需要减去的数值。同时，还需要知道右侧盒子的宽度是否包含`border`的宽度。
在这里，为了简单的计算右侧盒子准确的宽度，设置了子元素的`box-sizing:border-box;`以及父元素的`box-sizing: content-box;`。
同时，作为两个`inline-block`的盒子，必须设置`vertical-align`来使其顶端对齐。
另外，为了**准确地应用**计算出来的宽度，需要消除`div`之间的空格，需要通过设置父容器的`font-size: 0;`,或者用注释消除`html`中的空格等方法。
**缺点:**

- 需要知道左侧盒子的宽度，两个盒子的距离，还要设置各个元素的`box-sizing`
- 需要消除空格字符的影响
- 需要设置`vertical-align: top`满足顶端对齐。

####双float方案

```css
.wrapper-double-float {
    overflow: auto;        // 清除浮动
    box-sizing: content-box;
}

.wrapper-double-float .left,
.wrapper-double-float .right {
    float: left;
    box-sizing: border-box;
}

.wrapper-double-float .right {
    width: calc(100% - 140px);
}
```

本方案和双`float`方案原理相同，都是通过动态计算宽度来实现自适应。但是，由于浮动的`block`元素在有空间的情况下会[依次紧贴，排列在一行](https://www.w3.org/TR/CSS21/visuren.html#bfc-next-to-float)，所以无需设置`display: inline-block;`，自然也就少了顶端对齐，空格字符占空间等问题。

> A floated box is shifted to the left or right until its outer edge touches the containing block edge or the outer edge of another float.

不过由于应用了浮动，父元素需要清除浮动。
**缺点:**

- 需要知道左侧盒子的宽度，两个盒子的距离，还要设置各个元素的`box-sizing`。
- 父元素需要清除浮动。

####float+margin-left方案

```css
.wrapper-float {
    overflow: hidden;        // 清除浮动
}

.wrapper-float .left {
    float: left;
}

.wrapper-float .right {
    margin-left: 150px;
}
```

上面两种方案都是利用了CSS的`calc()`函数来计算宽度值。下面两种方案则是利用了`block`级别的元素盒子的宽度具有**填满父容器，并随着父容器的宽度自适应**的**流动特性**。
但是`block`级别的元素都是独占一行的，所以要想办法让两个`block`排列到一起。
我们知道，`block`级别的元素会认为浮动的元素不存在，但是`inline`级别的元素能识别到浮动的元素。这样，`block`级别的元素就可以和浮动的元素同处一行了。
为了让右侧盒子和左侧盒子保持距离，需要为左侧盒子留出足够的距离。这个距离的大小为左侧盒子的宽度以及两个盒子之间的距离之和。然后将该值设置为右侧盒子的`margin-left`。
**缺点：**

- 需要清除浮动
- 需要计算右侧盒子的`margin-left`

####使用absolute+margin-left方法

另外一种让两个`block`排列到一起的方法是对左侧盒子使用`position: absolute`的绝对定位。这样，右侧盒子也能**无视**掉它。

```css
.wrapper-absolute .left {
    position: absolute;
}

.wrapper-absolute .right {
    margin-left: 150px;
}
```

**缺点:**

- 使用了绝对定位，若是用在某个div中，需要更改父容器的`position`。
- 没有清除浮动的方法，若左侧盒子高于右侧盒子，就会超出父容器的高度。因此只能通过设置父容器的`min-height`来放置这种情况。

####使用float+BFC方法

上面的方法都需要通过左侧盒子的宽度，计算某个值，下面三种方法都是不需要计算的。只需要设置两个盒子之间的间隔。

```css
.wrapper-float-bfc {
    overflow: auto;
}

.wrapper-float-bfc .left {
    float: left;
    margin-right: 20px;
}

.wrapper-float-bfc .right {
    margin-left: 0;
    overflow: auto;
}
```

这个方案同样是利用了左侧浮动，但是右侧盒子通过`overflow: auto;`形成了BFC，因此右侧盒子不会与浮动的元素重叠。

> an element in the normal flow that establishes a new block formatting context (such as an element with 'overflow' other than 'visible') must not overlap the margin box of any floats in the same block formatting context as the element itself。

这种情况下，只需要为左侧的浮动盒子设置`margin-right`，就可以实现两个盒子的距离了。而右侧盒子是`block`级别的，所以宽度能实现自适应。
**缺点:**

- 父元素需要清除浮动

####flex方案

```css
.wrapper-flex {
    display: flex;
    align-items: flex-start;
}

.wrapper-flex .left {
    flex: 0 0 auto;
}

.wrapper-flex .right {
    flex: 1 1 auto;
}

```

flex`可以说是最好的方案了，代码少，使用简单。有朝一日，大家都改用现代浏览器，就可以使用了。
**需要注意**的是，`flex`容器的一个默认属性值:`align-items: stretch;`。这个属性导致了列等高的效果。
为了让两个盒子高度自动，需要设置: `align-items: flex-start;

####grid方案

又一个新型的布局方式。可以满足需求，但这并不是它发挥用处的真正地方。

```css
.wrapper-grid {
    display: grid;
    grid-template-columns: 120px 1fr;
    align-items: start;
}

.wrapper-grid .left,
.wrapper-grid .right {
    box-sizing: border-box;
}

.wrapper-grid .left {
    grid-column: 1;
}

.wrapper-grid .right {
    grid-column: 2;
}
```

**注意:**

- `grid`布局也有列等高的默认效果。需要设置: `align-items: start;`。
- `grid`布局还有一个值得注意的小地方和`flex`不同:在使用`margin-left`的时候，`grid`布局默认是`box-sizing`设置的盒宽度之间的位置。而`flex`则是使用两个div的`border`或者`padding`外侧之间的距离。

####极限情况

最后可以再看一下在父容器极限小的情况下，不同方案的表现。主要分成四种情况：

- 动态计算宽度的情况

  两种方案: 双inline-block方案和双float方案。宽度极限小时，右侧的div宽度会非常小，由于遵循流动布局，所以右侧div会移动到下一行。

- 动态计算右侧margin-left的情况

  两种方案: float+margin-left方案和absolute+margin-left方案。宽度极限小时，由于右侧的div忽略了文档流中左侧div的存在，所以其依旧会存在于这一行，并被隐藏。

- `float+BFC`方案的情况

  这种情况下，由于BFC与float的特殊关系，右侧div在宽度减小到最小后，也会掉落到下一行。

- `flex`和`grid`的情况

  这种情况下，默认两种布局方式都不会放不下的div移动到下一行。不过 flex布局可以通过 flex-flow: wrap;来设置多余的div移动到下一行。 grid布局暂不支持。

##10、HTTP状态码

| 100  | Continue                        | 继续。客户端应继续其请求                                     |
| ---- | ------------------------------- | ------------------------------------------------------------ |
| 101  | Switching Protocols             | 切换协议。服务器根据客户端的请求切换协议。只能切换到更高级的协议，例如，切换到HTTP的新版本协议 |
|      |                                 |                                                              |
| 200  | OK                              | 请求成功。一般用于GET与POST请求                              |
| 201  | Created                         | 已创建。成功请求并创建了新的资源                             |
| 202  | Accepted                        | 已接受。已经接受请求，但未处理完成                           |
| 203  | Non-Authoritative Information   | 非授权信息。请求成功。但返回的meta信息不在原始的服务器，而是一个副本 |
| 204  | No Content                      | 无内容。服务器成功处理，但未返回内容。在未更新网页的情况下，可确保浏览器继续显示当前文档 |
| 205  | Reset Content                   | 重置内容。服务器处理成功，用户终端（例如：浏览器）应重置文档视图。可通过此返回码清除浏览器的表单域 |
| 206  | Partial Content                 | 部分内容。服务器成功处理了部分GET请求                        |
|      |                                 |                                                              |
| 300  | Multiple Choices                | 多种选择。请求的资源可包括多个位置，相应可返回一个资源特征与地址的列表用于用户终端（例如：浏览器）选择 |
| 301  | Moved Permanently               | 永久移动。请求的资源已被永久的移动到新URI，返回信息会包括新的URI，浏览器会自动定向到新URI。今后任何新的请求都应使用新的URI代替 |
| 302  | Found                           | 临时移动。与301类似。但资源只是临时被移动。客户端应继续使用原有URI |
| 303  | See Other                       | 查看其它地址。与301类似。使用GET和POST请求查看               |
| 304  | Not Modified                    | 未修改。所请求的资源未修改，服务器返回此状态码时，不会返回任何资源。客户端通常会缓存访问过的资源，通过提供一个头信息指出客户端希望只返回在指定日期之后修改的资源 |
| 305  | Use Proxy                       | 使用代理。所请求的资源必须通过代理访问                       |
| 306  | Unused                          | 已经被废弃的HTTP状态码                                       |
| 307  | Temporary Redirect              | 临时重定向。与302类似。使用GET请求重定向                     |
|      |                                 |                                                              |
| 400  | Bad Request                     | 客户端请求的语法错误，服务器无法理解                         |
| 401  | Unauthorized                    | 请求要求用户的身份认证                                       |
| 402  | Payment Required                | 保留，将来使用                                               |
| 403  | Forbidden                       | 服务器理解请求客户端的请求，但是拒绝执行此请求               |
| 404  | Not Found                       | 服务器无法根据客户端的请求找到资源（网页）。通过此代码，网站设计人员可设置"您所请求的资源无法找到"的个性页面 |
| 405  | Method Not Allowed              | 客户端请求中的方法被禁止                                     |
| 406  | Not Acceptable                  | 服务器无法根据客户端请求的内容特性完成请求                   |
| 407  | Proxy Authentication Required   | 请求要求代理的身份认证，与401类似，但请求者应当使用代理进行授权 |
| 408  | Request Time-out                | 服务器等待客户端发送的请求时间过长，超时                     |
| 409  | Conflict                        | 服务器完成客户端的PUT请求是可能返回此代码，服务器处理请求时发生了冲突 |
| 410  | Gone                            | 客户端请求的资源已经不存在。410不同于404，如果资源以前有现在被永久删除了可使用410代码，网站设计人员可通过301代码指定资源的新位置 |
| 411  | Length Required                 | 服务器无法处理客户端发送的不带Content-Length的请求信息       |
| 412  | Precondition Failed             | 客户端请求信息的先决条件错误                                 |
| 413  | Request Entity Too Large        | 由于请求的实体过大，服务器无法处理，因此拒绝请求。为防止客户端的连续请求，服务器可能会关闭连接。如果只是服务器暂时无法处理，则会包含一个Retry-After的响应信息 |
| 414  | Request-URI Too Large           | 请求的URI过长（URI通常为网址），服务器无法处理               |
| 415  | Unsupported Media Type          | 服务器无法处理请求附带的媒体格式                             |
| 416  | Requested range not satisfiable | 客户端请求的范围无效                                         |
| 417  | Expectation Failed              | 服务器无法满足Expect的请求头信息                             |
|      |                                 |                                                              |
| 500  | Internal Server Error           | 服务器内部错误，无法完成请求                                 |
| 501  | Not Implemented                 | 服务器不支持请求的功能，无法完成请求                         |
| 502  | Bad Gateway                     | 作为网关或者代理工作的服务器尝试执行请求时，从远程服务器接收到了一个无效的响应 |
| 503  | Service Unavailable             | 由于超载或系统维护，服务器暂时的无法处理客户端的请求。延时的长度可包含在服务器的Retry-After头信息中 |
| 504  | Gateway Time-out                | 充当网关或代理的服务器，未及时从远端服务器获取请求           |
| 505  | HTTP Version not supported      | 服务器不支持请求的HTTP协议的版本，无法完成处理               |

##11、伪元素和伪类

概念上区分：

**伪类：**更多的定义的是状态。常见的伪类有:hover, :active, :focus, :visited, :line, :not, :first-child, :last-child等等。

**伪元素：**不存在于DOM树中的虚拟元素，它们可以像正常的html元素一样定义css，但无法使用JavaScript获取。常见的伪元素有 ::before, ::after, ::first-letter, ::first-line等等。

CSS3明确规定，伪类用一个冒号来表示，伪元素用两个冒号来表示，但目前因为兼容性的问题，它们的写法可以是一致的，都用一个冒号就行。

###实战场景——伪类

#### 表单校验

表单的校验中，常会用到 `:required`、`:valid` 和 `:invalid` 这三个伪类。先来看看它们所代表的含义。

- :required，指定具有 required属性 的表单元素
- :valid，指定一个 匹配指定要求 的表单元素
- :invalid，指定一个 不匹配指定要求 的表单元素

看下面这个例子：

```html
<p>input中类型为email的校验</p>

<p>符合email校验规则</p>
<input type="email" required placeholder="请输入" value="24238477@qq.com" />
<br><br>

<p>不符合email校验规则</p>
<input type="email" required placeholder="请输入" value="lalala" />
<br><br>

<p>有required标识，但未填写</p>
<input type="email" required placeholder="请输入" value="" />

input {
    &:valid {
        border-color: green;
        box-shadow: inset 5px 0 0 green;
    }
    &:invalid {
        border-color: red;
        box-shadow: inset 5px 0 0 red;
    }
    &:required {
        border-color: red;
        box-shadow: inset 5px 0 0 red;
    }
}
```

效果如下：



![img](https://user-gold-cdn.xitu.io/2019/1/12/1683fe563d44a675?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



#### 折叠面板

过去，要实现折叠面板的显示或隐藏，只能用JavaScript来搞定。但是现在，可以用伪类 `:target` 来实现。 :target 是文档的内部链接，即 URL 后面跟有锚名称 #，指向文档内某个具体的元素。

看下面这个例子：

```html
<div class="t-collapse">
    <!-- 在url最后添加 #modal1，使得target生效 —>
    <a class="collapse-target" href="#modal1">target 1</a>
    <div class="collapse-body" id="modal1">
        <!-- 将url的#modal1 变为 #，使得target失效 —>
        <a class="collapse-close" href="#">target 1</a>
        <p>...</p>
    </div>
</div>

.t-collapse {
    >.collapse-body {
        display: none;
        &:target {
            display: block;
        }
    }
}

```

#### 元素的index

当我们要指定一系列标签中的某个元素时，并不需要用JavaScript获取。可以用 `:nth-child(n)` 与 `:nth-of-type(n)` 来找到，并指定样式。但它们有一些小区别，需要注意。

首先，它们的n可以是大于零的数字，或者类似2n+1的表达式，再或者是 even / odd。

另外，还有2个区别：

- :nth-of-type(n) 除了关注n之外，还需要关注最前面的`类型`，也就是标签。
- :nth-child(n) 它关注的是：其父元素下的第n个孩子，与类型无关。

看下面这个例子，注意两者的差异：

```html
<h1>这是标题</h1>
<p>第一个段落。</p>
<p>第二个段落。</p>
<p>第三个段落。</p>
<p>第四个段落。</p>
<p>第五个段落。</p>
```



![img](https://user-gold-cdn.xitu.io/2019/1/12/1683fe60c6b954e0?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

###实战场景——伪元素

#### antd的彩蛋事件

还记得2018年圣诞节的“彩蛋事件”，在整个前端圈，轰动一时。因为按钮上的一朵云，导致不少前端er提前回家过年了。当时，彩蛋事件出现的第一时间，就吓得我赶快打开工程看了一眼，果然也中招了。为了保住饭碗，得赶紧把云朵去掉。

查看了生成的html，发现原来是 button 下藏了一个 `::before`。所以，赶紧把样式覆盖掉，兼容代码如下：

```less
.ant-btn {
    &::before {
        display: none !important;
    }
}
```

#### 美化选中的文本

在网页中，默认的划词效果是，原字色保持不变，划过时的背景变为蓝底色。其实，这是可以用 `::selection` 来进行美化的。看下面这个例子：

```html
<p>Custom text selection color</p>
::selection {
    color: red;
    background-color: yellow;
}
```

效果如下：



![img](https://user-gold-cdn.xitu.io/2019/1/12/1683fe68dda8e5f0?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



划过的部分美化为：红色的字体，并且底色变为了黄色。

###附录

#### CSS3中的伪类

:root 选择文档的根元素，等同于 html 元素

:empty 选择没有子元素的元素

:target 选取当前活动的目标元素

:not(selector) 选择除 selector 元素意外的元素

:enabled 选择可用的表单元素

:disabled 选择禁用的表单元素

:checked 选择被选中的表单元素

:nth-child(n) 匹配父元素下指定子元素，在所有子元素中排序第n

:nth-last-child(n) 匹配父元素下指定子元素，在所有子元素中排序第n，从后向前数

:nth-child(odd) 、 :nth-child(even) 、 :nth-child(3n+1) :first-child 、 :last-child 、 :only-child

:nth-of-type(n) 匹配父元素下指定子元素，在同类子元素中排序第n

:nth-last-of-type(n) 匹配父元素下指定子元素，在同类子元素中排序第n，从后向前数

:nth-of-type(odd) 、 :nth-of-type(even) 、 :nth-of-type(3n+1) :first-of-type 、 :last-of-type 、 :only-of-type

#### CSS3中的伪元素

::after 已选中元素的最后一个子元素

::before 已选中元素的第一个子元素

::first-letter 选中某个款级元素的第一行的第一个字母

::first-line 匹配某个块级元素的第一行

::selection 匹配用户划词时的高亮部分

## 12、HTML行内元素、块状元素、行内块状元素的区别

HTML可以将元素分类方式分为行内元素、块状元素和行内块状元素三种。首先需要说明的是，这三者可以互相转换的，使用display属性能够将三者任意转换：

（1）display: inline; 转换为行内元素

（2）display: block; 转换为块状元素

（3）display: inline-block; 转换为行内块状元素

```html
<!DOCTYPE html>
<html>

    <head>
        <meta charset="utf-8" />
        <title>测试案例</title>
        <style type="text/css">
            span {
                 display: block;
                width: 120px;
                height: 30px;
                background: red;
            }
            
            div {
                display: inline;
                width: 120px;
                height: 200px;
                background: green;
            }
            
            i {
                display: inline-block;
                width: 120px;
                height: 30px;
                background: lightblue;
            }
        </style>
    </head>

    <body>
        <span>行内转块状</span>
        <div>块状转行内 </div>
        <i>行内转行内块状</i>
    </body>

</html>
```

### 1、行内元素

行内元素最常使用的就是span，其他的只在特定功能下使用，修饰字体\<b>和\<i>标签，还有\<sub>和\<sup>这两个标签可以直接做出平方的效果，而不需要类似移动属性的帮助，很实用。

行内元素特征：（1）设置宽高无效

​			     （2）对margin仅设置左右方向有效，上下无效；padding设置上下左右都有效，即会撑大空间

​			     （3）不会自动进行换行

```html
<!DOCTYPE html>
<html>

    <head>
        <meta charset="utf-8" />
        <title>测试案例</title>
        <style type="text/css">
            span {
                width: 120px;
                height: 120px;
                margin: 1000px 20px;
                padding: 50px 40px;
                background: lightblue;
            }
        </style>
    </head>

    <body>
        <i>不会自动换行</i>
        <span>行内元素</span>
    </body>

</html>
```

### 2、块状元素

块状元素代表性的就是div，其他如p、nav、aside、header、footer、section、article、ul-li、address等等，都可以用div来实现。不过为了可以方便程序员解读代码，一般都会使用特定的语义化标签，使得代码可读性强，且便于查错。

块状元素特征：（1）能够识别宽高

​			     （2）margin和padding的上下左右均对其有效

​			     （3）可以自动换行

​			     （4）多个块状元素标签写在一起，默认排列方式为从上至下

```html
<!DOCTYPE html>
<html>

    <head>
        <meta charset="utf-8" />
        <title>测试案例</title>
        <style type="text/css">
            div {
                width: 120px;
                height: 120px;
                margin: 50px 50px;
                padding: 50px 40px;
                background: lightblue;
            }
        </style>
    </head>

    <body>
        <i>自动换行</i>
        <div>块状元素</div>
        <div>块状元素</div>
    </body>

</html>
```

### 3、行内块状元素

行内块状元素综合了行内元素和块状元素的特性，但是各有取舍。因此行内块状元素在日常的使用中，由于其特性，使用的次数也比较多。

行内块状元素特征：（1）不自动换行

　　　　　　　　　（2）能够识别宽高

　　　　　　　　　（3）默认排列方式为从左到右

```html
<!DOCTYPE html>
<html>

    <head>
        <meta charset="utf-8" />
        <title>测试案例</title>
        <style type="text/css">
            div {
                display: inline-block;
                width: 100px;
                height: 50px;
                background: lightblue;
            }
        </style>
    </head>

    <body>
        <div>行内块状元素</div>
        <div>行内块状元素</div>
        
    </body>

</html>
```

## 13、BFC

在一个Web页面的CSS渲染中，[块级格式化上下文](http://www.w3.org/TR/CSS21/visuren.html#block-formatting) (Block Fromatting Context)是按照块级盒子布局的。W3C对BFC的定义如下：

>浮动元素和绝对定位元素，非块级盒子的块级容器（例如 inline-blocks, table-cells, 和 table-captions），以及overflow值不为“visiable”的块级盒子，都会为他们的内容创建新的BFC（块级格式上下文）。

为了便于理解，我们换一种方式来重新定义BFC。一个HTML元素要创建BFC，则满足下列的任意一个或多个条件即可：

1、float的值不是none。
2、position的值不是static或者relative。
3、display的值是inline-block、table-cell、flex、table-caption或者inline-flex
4、overflow的值不是visible

BFC是一个独立的布局环境，其中的元素布局是不受外界的影响，并且在一个BFC中，块盒与行盒（行盒由一行中所有的内联元素所组成）都会垂直的沿着其父元素的边框排列。

### 怎么创建BFC

要显示的创建一个BFC是非常简单的，只要满足上述4个CSS条件之一就行。例如：

```html
<div class="container">
  你的内容
</div>
```

在类container中添加类似 overflow: scroll，overflow: hidden，display: flex，float: left，或 display: table 的规则来显示创建BFC。虽然添加上述的任意一条都能创建BFC，但会有一些副作用：

1、display: table 可能引发响应性问题
2、overflow: scroll 可能产生多余的滚动条
3、float: left 将把元素移至左侧，并被其他元素环绕
4、overflow: hidden 将裁切溢出元素

因而无论什么时候需要创建BFC，都要基于自身的需要来考虑。对于本文，将采用 overflow: hidden 方式：

```css
.container {
    overflow: hidden;
}
```

### 再说两点

#### BFC中盒子怎么对齐

如前文所说，在一个BFC中，块盒与行盒（行盒由一行中所有的内联元素所组成）都会垂直的沿着其父元素的边框排列。W3C给出得规范是：

>在BFC中，每一个盒子的左外边缘（margin-left）会触碰到容器的左边缘(border-left)（对于从右到左的格式来说，则触碰到右边缘）。浮动也是如此（尽管盒子里的行盒子 Line Box 可能由于浮动而变窄），除非盒子创建了一个新的BFC（在这种情况下盒子本身可能由于浮动而变窄）。

![BFC](https://camo.githubusercontent.com/79bf7dc35ecbddea11eab4edc9eacf6bd88af365/687474703a2f2f7365676d656e746661756c742e636f6d2f696d672f62566d327150)

#### 外边距折叠

常规流布局时，盒子都是垂直排列，两者之间的间距由各自的外边距所决定，但不是二者外边距之和。

```html
<div class="container">
  <p>Sibling 1</p>
  <p>Sibling 2</p>
</div>
```

对应的CSS：

```css
.container {
  background-color: red;
  overflow: hidden; /* creates a block formatting context */
}
p {
  background-color: lightgreen;
  margin: 10px 0;
}
```

渲染结果如图：
[![img2](https://camo.githubusercontent.com/9332f52c1e1500ca828443ae15d2282b4c121483/687474703a2f2f7365676d656e746661756c742e636f6d2f696d672f62566d327152)](https://camo.githubusercontent.com/9332f52c1e1500ca828443ae15d2282b4c121483/687474703a2f2f7365676d656e746661756c742e636f6d2f696d672f62566d327152)

在上图中，一个红盒子（div）包含着两个兄弟元素（p），一个BFC已经创建了出来。

理论上，两个p元素之间的外边距应当是二者外边距之和（20px）但实际上却是10px，这是外边距折叠(Collapsing Margins)的结果。

在CSS当中，相邻的两个盒子（可能是兄弟关系也可能是祖先关系）的外边距可以结合成一个单独的外边距。这种合并外边距的方式被称为折叠，并且因而所结合成的外边距称为折叠外边距。折叠的结果按照如下规则计算：

1、两个相邻的外边距都是正数时，折叠结果是它们两者之间较大的值。
2、两个相邻的外边距都是负数时，折叠结果是两者绝对值的较大值。
3、两个外边距一正一负时，折叠结果是两者的相加的和。

产生折叠的必备条件：margin必须是邻接的! (对于不产生折叠的情况，见参考文章的链接)

### BFC可以做什么呢？

#### 利用BFC避免外边距折叠

BFC可能造成外边距折叠，也可以利用它来避免这种情况。BFC产生外边距折叠要满足一个条件：两个相邻元素要处于同一个BFC中。所以，若两个相邻元素在不同的BFC中，就能避免外边距折叠。

改进前面的例子：

```html
<div class="container">
    <p>Sibling 1</p>
    <p>Sibling 2</p>
    <p>Sibling 3</p>
</div>
```

对应的CSS：

```css
.container {
  background-color: red;
  overflow: hidden; /* creates a block formatting context */
}
p {
  background-color: lightgreen;
  margin: 10px 0;
}
```

结果和上面一样，由于外边距折叠，三个相邻P元素之间的垂直距离是10px，这是因为三个 p 标签都从属于同一个BFC。

修改第三个P元素，使之创建一个新的BFC：

```html
<div class="container">
    <p>Sibling 1</p>
    <p>Sibling 2</p>
    <div class="newBFC">
        <p>Sibling 3</p>
    </div>
</div>
```

对应的CSS：

```css
.container {
    background-color: red;
    overflow: hidden; /* creates a block formatting context */
}
p {
    margin: 10px 0;
    background-color: lightgreen;
}
.newBFC {
    overflow: hidden;  /* creates new block formatting context */
}
```

现在的结果如图：
[![img](https://camo.githubusercontent.com/c7c77701b984c7d4f957a7729db2cf534ef92c3d/687474703a2f2f7365676d656e746661756c742e636f6d2f696d672f62566d327153)](https://camo.githubusercontent.com/c7c77701b984c7d4f957a7729db2cf534ef92c3d/687474703a2f2f7365676d656e746661756c742e636f6d2f696d672f62566d327153)

因为第二个和第三个P元素现在分属于不同的BFC，它们之间就不会发生外边距折叠了。

#### BFC包含浮动

浮动元素是会脱离文档流的(绝对定位元素会脱离文档流)。如果一个没有高度或者height是auto的容器的子元素是浮动元素，则该容器的高度是不会被撑开的。我们通常会利用伪元素(:after或者:before)来解决这个问题。BFC能包含浮动，也能解决容器高度不会被撑开的问题。

[![img](https://camo.githubusercontent.com/21d74bf8701804cd465f8f88311d771372f0df4b/687474703a2f2f7365676d656e746661756c742e636f6d2f696d672f62566d327154)](https://camo.githubusercontent.com/21d74bf8701804cd465f8f88311d771372f0df4b/687474703a2f2f7365676d656e746661756c742e636f6d2f696d672f62566d327154)

看个例子：

```html
<div class="container">
    <div>Sibling</div>
    <div>Sibling</div>
</div>

```

对应的CSS：

```css
.container {
  background-color: green;
}
.container div {
  float: left;
  background-color: lightgreen;
  margin: 10px;
}
```

在上面这个例子中，容器没有任何高度，并且它包不住浮动子元素，容器的高度并不会被撑开。为解决这个问题，可以在容器中创建一个BFC：

```css
.container {
    overflow: hidden; /* creates block formatting context */
    background-color: green;
}
.container div {
    float: left;
    background-color: lightgreen;
    margin: 10px;
}
```

现在容器可以包住浮动子元素，并且其高度会扩展至包住其子元素，在这个新的BFC中浮动元素又回归到页面的常规流之中了。

#### 使用BFC避免文字环绕

[![img](https://camo.githubusercontent.com/f09eac6e73acf8ecfcc329b1bff19c778912219e/687474703a2f2f7365676d656e746661756c742e636f6d2f696d672f62566d327159)](https://camo.githubusercontent.com/f09eac6e73acf8ecfcc329b1bff19c778912219e/687474703a2f2f7365676d656e746661756c742e636f6d2f696d672f62566d327159)

如上图所示，对于浮动元素，可能会造成文字环绕的情况(Figure1)，但这并不是我们想要的布局(Figure2才是想要的)。要解决这个问题，我们可以用外边距，但也可以用BFC。

First let us understand why the text wraps. For this we have to understand how the box model works when an element is floated. This is the part I left earlier while discussing the alignment in a block formatting context. Let us understand what is happening in Figure 1 in the diagram below:

[![img](https://camo.githubusercontent.com/d5233a0edc9dd480538f6e75cf65d058c6fd88c7/687474703a2f2f7365676d656e746661756c742e636f6d2f696d672f62566d327130)](https://camo.githubusercontent.com/d5233a0edc9dd480538f6e75cf65d058c6fd88c7/687474703a2f2f7365676d656e746661756c742e636f6d2f696d672f62566d327130)

假设HTML是：

```html
<div class="container">
    <div class="floated">
        Floated div
    </div>
    <p>
        Quae hic ut ab perferendis sit quod architecto, 
        dolor debitis quam rem provident aspernatur tempora
        expedita.
    </p>
</div>
```

上图整个黑色区域表示 p 元素。p 元素没有移位但它叠在了浮动元素之下，而p元素的文本(行盒子)却移位了，行盒子水平变窄来给浮动元素腾出了空间。随着文本的增加，最后文本将环绕在浮动元素之下，因为那时候行盒子不再需要移位，也就成了图Figure1的样子。

再回顾一下W3C的描述：
>在BFC上下文中，每个盒子的左外侧紧贴包含块的左侧（从右到左的格式里，则为盒子右外侧紧贴包含块右侧），甚至有浮动也是如此（尽管盒子里的行盒子 Line Box 可能由于浮动而变窄），除非盒子创建了一个新的BFC（在这种情况下盒子本身可能由于浮动而变窄）。

因而，如果p元素创建一个新的BFC那它就不会再紧贴包含块的左侧了。

#### 在多列布局中使用BFC

如果我们创建一个占满整个容器宽度的多列布局，在某些浏览器中最后一列有时候会掉到下一行。这可能是因为浏览器四舍五入了列宽从而所有列的总宽度会超出容器。但如果我们在多列布局中的最后一列里创建一个新的BFC，它将总是占据其他列先占位完毕后剩下的空间。

例如：

```html
<div class="container">
    <div class="column">column 1</div>
    <div class="column">column 2</div>
    <div class="column">column 3</div>
</div>
```

对应的CSS：

```css
.column {
    width: 31.33%;
    background-color: green;
    float: left;
    margin: 0 1%;
}
/*  Establishing a new block formatting 
    context in the last column */
.column:last-child {
    float: none;
overflow: hidden; 
}
```

现在尽管盒子的宽度稍有改变，但布局不会打破。当然，对多列布局来说这不一定是个好办法，但能避免最后一列下掉。这个问题上弹性盒或许是个更好的解决方案，但这个办法可以用来说明元素在这些环境下的行为。