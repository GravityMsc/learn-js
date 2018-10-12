最近整理了一份HTML/CSS/JS编码规范，供大家参考。

## 一、HTML编码规范

### 1、img标签要写alt属性

根据W3C标准，img标签要写alt属性，如果没有就写一个空的。但是一般要写一个有内容的，根据图片想要表达的意思，因为alt是在图片无法加载时显式的文字。如下不太好的写法：

```html
<img src="company-logo.svg" alt="ABC Company Logo">
```

更好的写法：

```html
<img src="chime-logo.svg" alt="ABC Company">
```

这里就不用告诉用户它是一个Logo了，直接告诉它是ABC Company就好了。再如：

```html
<img src="user-head.img" alt="User Portrait">
```

可改成：

```html
<img src="user-head.jpg" alt="Bolynga Team">
```

如果图片显示不出来，就直接显示用户的名字。

有些人偷懒就直接写个空的alt那也可以，但是在一些重要的地方还是要写一下，毕竟它还是有利于SEO。

### 2、单标签不要写闭合标签

为什么？因为写了也没用，还显得你不懂html规范，我们不是写XHTML。常见的单标签有img、link、input、hr、br，如下不好的写法：

```html
<img src="test.jpg"></img><br/>
<input type="email" value=""/>
```

应该改成：

```html
<img src="test.jpg"><br>
<input type="email" value="">
```

如果你用React写jsx模板，它就要求每个标签都要闭合，但是它始终不是原生html。

### 3、自定义属性要以data-开头

自己添加的非标准的属性要以data-开头，否则[w3c validator](https://validator.w3.org/)会认为是不规范的，如下不好的写法：

```html
<div count="5"></div>
```

应改成：

```html
<div data-count="5"></div>
```

### 4、td要在tr里面，li要在ul/ol里面

如下不好的写法：

```html
<div>
    <li></li>
    <li></li>
</div>
```

更常见的是td没有写在tr里面：

```html
<table>
    <td></td>
    <td></td>
</table>
```

如果你写得不规范，有些浏览器会帮你矫正，但是有些可能就没有那么幸运。因为标准没有说如果写得不规范应该怎么处理，各家浏览器可能有自己的处理方式。

### 5、ul/ol的直接子元素只能是li

有时候可能会直接在ul里面写一个div或者span，以为这样没关系：

```html
<ol>
    <span>123</span>
    <li>a</li>
    <li>b</li>
</ol>
```

这样写也是不规范的，不能直接在ol里面写span，ol是一个列表吗，它的子元素应该都是display:list-item的，突然冒出来个span，你让浏览器如何自处。所以写得不规范就会导致在不同的浏览器会有不同的表现。

同样，tr的直接子元素都应该是td，你在td里面写tr那就乱了。

### 6、section里面要有标题标签

如果你用了section/aside/article/nav这种标签的话，需要在里面写一个h1/h2/h3之类的标题标签，因为这四个标签可以划分章节，它们都是独立的章节，需要有标题，如果UI里面根本就没有标题呢？那你可以写一个隐藏的标题标签，如果处于SEO的目的，你不能直接display:none，而要用一些特殊的处理方式，如下套一个hidden-text的类：

```html
<style>.hidden-text{position: absolute; left: -9999px; right: -9999px}</style>
<section>
    <h1 class="hidden-text">Listing Detail</h1>
</section>
```

### 7、使用section标签增强SEO

使用section的好处是可以划分章节，如下代码：

```html
<body>
<h1>Listing Detail</h1>
<section>
    <h1>House Infomation</h1>
    <section>
       <h1>LOCATION</h1>
       <p></p>
    </section>
    <section>
        <h1>BUILDING</h1>
        <p></p>
    </section>
</section>
<section>
    <h1>Listing Picture</h1>
</section>
</body>
```

就会被outline成这样的大纲：

Listing Detail

1. House Infomation
   1. LOCATION
   2. BUILDING
2. Listing Picture

可以使用[html5 outliner](https://gsnedders.html5.org/outliner/)进行实验，可以看到，我们很任性地使用了很多h1标签，这个在html4里面是不合法的。

### 8、行内元素里面不可使用块级元素

例如下面的写法是不合法的：

```html
<a href="/listing?id=123">
	<div>
        
    </div>
</a>
```

a标签是一个行内元素，行内元素里面套了一个div的标签，这样可能会导致a标签无法正常点击。再或者是span里面套了div，这种情况下需要把inline元素显式地设置display为block，如下代码：

```html
<a href="/listing?id=123" style="display: block">
    <div></div>
</a>
```

这样就正常了。

### 9、每个页面要写<!DOCType html>

设置页面的渲染模式为标准模式，如果忘记写了会怎么样？忘记写了会变成怪异模式，怪异模式下很多东西渲染会有所不同，怪异模式下input/textarea的默认盒模型会变成border-box，文档高度会变成可视窗口的高度，获取window的高度时就不是期望的文档高度。还有一个区别，父容器行高在怪异模式下将不会影响子元素，如下代码：

```html
<div><img src="test.jpg" style="height:100px"></div>
```

在标准模式下div下方会留点空白，而在怪异模式下会。这个就提醒我们在写邮件模板时需要在顶部加上<!DOCType html>，因为在本地开发邮件模板时是写html片段，没有这个的话就会变成怪异模式。

### 10、要用table布局写邮件模板

由于邮件客户端多种多样，你不知道用户是使用什么看的邮件，有可能是用的网页邮箱，也有可能用的gamil/outlook等客户端。这些客户端多种多样，对html/css的支持也不一，所以我们不能使用高级的布局和排版，例如flex/float/absoulte定位，使用较初级的table布局能够达到兼容性最好的效果，并且还有伸缩的效果。

另外邮件模板里面不能写媒体查询，不能写script，不能写外联样式，这些都会被邮件客户端过滤掉，样式都得用内联style，你可以先写成外联，然后再用一些工具帮你生成内联html。

写完后要实际测一下，可以用QQ邮箱发送，它支持发送html格式文本吗，发完后在不同的客户端打开看一下， 看有没有问题，如手机的客户端，电脑的客户端，以及浏览器。

由于你不知道用户是用手机打开还是电脑打开，所以你不能把邮件内容的宽度写死，但是完全100%也不好，在PC大屏幕上看起来可能会太大，所以一般可以这样写：

```html
<table style="border-collapse:collapse;font-family: Helvetica Neue,Helvetica,Arial;font-size:14px;width:100%;height:100%">
    <tr><td align="center" valign="top"><table style="border:1px solid #ececec;border-top:none; max-width:600px;border-collapse:collapse">
    <tr><td>内容1</td></tr>
    <tr><td>内容2</td></tr>
</table></td></tr></table>

```

最外面的table宽度100%，里面的table有一个max-width：600px，相对于外面的table居中。这样在PC上最大宽度就为600px，而在手机客户端上宽度就为100%。

但是有些客户单如比较老的outlook无法识别max-width的属性，导致在PC上太宽。但是这个没有办法，因为我们不能直接把宽度写死不然在手机上就要左右滑了，也不能写script判断ua之类的方法。所以无法兼容较老版本的outlook。

### 11、html要保持简洁，不要套太多层

需要套很多层的，一般有两种情况，一种是切图不太会，需要套很多层来维持排版，第二种是会切图，但是把UI拆解得太细。像以下布局：

![布局](https://user-gold-cdn.xitu.io/2017/8/24/bc46161a0d07f1db39d5fc7978d5b443?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

我会这么写：

```html
<ul>
    <li>
    	<div class="img-container">
            <img>
        </div>
        <div class="text"></dib>
    </li>
</ul>
```

因为它是一个列表，所以外面用ul/ui作为容器，里面就两个div，一个图片的float:left，另外一个文字的容器，这样就可以了。不用套很多层，但是有一些必要的，你写得太简单的话，扩展性不好，例如它是一个列表那你就应该使用ul/ol，如果你要清除浮动，那你可能需要套一个clearfix的容器。

如果有一块是整体，它整体向右排版，那么这一块也要套一个容器，有时候为了实现一些效果，可能也要多套一个容器，例如让外面的容器float，而里面的容器display:table-cell。但是你套这些容器应该都是由价值，如果你只是为了在功能上看起来整齐划一，区分明细好像没有太大的意义。

### 12、特殊情况下才在html里面写script和style

通常来说，在html里面直接写script和style是一种不好的实践，这样把样式、结构和逻辑都掺杂在一起了。但是有时候为了避免闪屏的问题，可能会直接在相应的html后面跟上调整的script，这种script看起来有点丑陋，但是很实用，是没有办法的办法。

### 13、样式要写在head标签里

样式不能写在body里，写在body里会导致渲染两次，特别是写得越靠后，可能会出现闪屏的情况，例如上面的已经渲染好了，突然遇到一个style标签，导致它要重新渲染，这样就闪了一下，不管是从码农的追求还是用户的体验，在body里面写style终究是一种下策。

同样地script标签不要写在head标签里面，会阻碍页面加载。

而CSS也推荐写成style标签直接嵌套在页面上，因为如果搞个外链，浏览器需要先做域名解析，然后再建立连接，接着才是下载，这一套下来可能已经过了0.5s/1s，甚至2~3秒。而写在页面的CSS虽然无法缓存，但是本身它也不会很大，再加gzip压缩，基本在50k以内。

### 14、html要加上lang的属性

如下，如果是英文的网页，应该这么写：

```html
<html lang="en">
<html lang="en-US">
```

第一种表示它是英文的网页，第二种表示它是美国英语的网页，加上这个的好处是有利于SEO和屏幕阅读器使用者，他可以快速地知道这个网页是什么语言的，如果是中文可以这样写：

```html
<html lang="zh-CN">
```

### 15、要在head标签靠前位置写上charset的meta标签

如下，一般charset的meta标签要写在head标签后的第一个标签：

```html
<head>
   <meta charset="utf-8">
</head>
```

一个原因是避免网页显示unicode符号时乱码，写在前面是因为w3c有规定，语言编码要在html文档的前1024个字节。如果不写的话再老的浏览器会有urf-7攻击的隐患，具体可以自行查阅资料，只是现在的浏览器基本都去掉了对utf-7编码的支持了。

charset的标签写成html5的这种比较简洁的写法就行了，不需要写成html4这种长长的：

```html
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
```

据我所查，就算是IE6也支持那种简短的写法，虽然它不是一个html5浏览器。

### 16、特殊符号使用html实体

不要直接把Unicode的特殊符号直接拷贝到html文档里面，要使用它对应的实体Entity，常用的如下表所示：

| 符号 | 实体编码 |
| ---- | -------- |
| ©    | &copy ;  |
| ¥    | &yen ;   |
| ®    | &reg ;   |
| >    | &gt ;    |
| <    | &lt ;    |
| &    | &amp ;   |

> 注意上面表格中的实体编码中的分号应该和前面的字符连接在一起，因为使用的编辑器Typora会自动将实体字符编码替换为符号，这样就无法展示实体编码的效果了。

特别是像©这种符号，不要从UI里面直接拷贝一个unicode的字符过去，如果直接拷贝过去会比较丑，它取的是用的字体里面的符号。

### 17、img空src的问题

有时候可能你需要写一个空的img标签，然后再JS里面动态地给它赋src，所以你可能会这么写：

```html
<img src="" alt>
```

但是这样写会有问题，如果你写一个空的src，会导致浏览器认为src就是当前页面链接，然后再一次请求当前页面，就跟你写一个a标签的href为空类似。如果是background-image也会有类似的问题。这个时候怎么办呢？如果你随便写一个不存在的url，浏览器会报404错误。

我知道的有两种解决办法，第一种是把src写成about:blank，如下：

```html
<img src="about:blank" alt>
```

这样它会去加载一个空白页面，这个没有兼容问题，不会加载当前页码，也不会报错。

第二种办法是写个1px的透明像素的base64，如下代码所示：

```html
<img src="data:image/gif;base64,R0lGODlhAQABAIAAAAAAAP///yH5BAEAAAAALAAAAAABAAEAAAIBRAA7">
```

第二种可能比较符合规范，但是第一种比较简单，并且没有兼容性问题。

### 18、关于行内元素空格和换行的影响

有时候换行可能会引入空格，如下代码：

```html
<form>
    <label>Email: </label>
    <input type="email">
</form>

```

在label和input中间会有一个空格，这样可能会导致设置label的width和input的width两者的和等于form的时候会导致input换行了，有时候你检查半天没查出原因，最后可能发现，原来是多了一个空格，而这个空格是换行引起的。这个时候你可能有一个问题，为什么\<form>和\<label>之间以及\<input>和\<form>之间的换行为什么没有引入空格？这是因为块级元素开始的空白文本将会被忽略，如下Chrome源码的说明：

> // Whitespace at the start of a block just goes away. Don't even
>
> // make a layout object for this text.

并且，块级元素后面的空白文本节点将不会参与渲染，也就是说像这种：

```html
<div></div>
<div></div>
```

两个div之间有textNode的文本节点，但是不会参与渲染。

要注意的是注释标签也是正常的页面标签，也会给它创建一个相应的节点，只是它不参与渲染。

### 19、类的命名使用小写字母加中划线连接

如下使用-连接，不要使用驼峰式：

```html
<div class="hello-world"></div>
```

### 20、不推荐使用自定义标签

是否可以使用自定义标签，像angular那样是用的自定义标签，如下代码：

```html
<my-cotnainer></my-container>
```

一般不推荐使用自定义标签，angular也有开关可以控制是否要使用自定义标签。虽然使用自定义标签也是合法的，只要你给他display: block，它就像一个div一样了，但是不管从SEO还是规范化的角度，自定义标签还是有点另类，虽然可能你会觉得它的语义化更好。

### 21、重复id和重复属性

我们知道，如果在页面写了两个一模一样的id，那么查DOM的时候只会取第一个，同理重复的属性也会只取第一个，如下：

```html
<input class="books" type="text" name="books" class="valid">
```

第二个class将会被忽略，className重复了又会怎么样？重复的也是无效的，这里主要是注意如果你直接操作原生className要注意避免className重复，如下代码：

```javascript
var input = form.books;
input.className += " valid";
```

如果重复执行的话，className将会有重复的valid类。

### 22、不推荐使用属性设置样式

例如，如果你要设置一个图片的宽高，可能这么写：

```html
<img src="test.jpg" alt width="400" height="300">
```

这个在iOS的Safari上面是不支持的，可以自行实验。

或者table也有一些可以设置：

```html
<table border="1"></table>
```

但是这种能够用CSS设置的就用CSS，但是有一个例外就是canvas的宽高需要写在html上， 如下代码：

```html
<canvas width="800" height="600"></canvas>
```

如果你用CSS设置的话它会变成拉伸，变得比较模糊。

### 23、使用适合的标签

标签使用上不要太单调：

（1）如果内容是表格就使用table，table有自适应的有点；如果是一个列表就使用ol/ul标签，扩展性比较好。

（2）如果是输入框就使用input，而不是写一个p标签，然后设置contenteditable=true，因为这个在iOS Safari上光标定位容易出现问题。如果需要做特殊效果除外。

（3）如果是粗体就使用b/strong，而不是自己设置font-weight。

（4）如果是表单就使用form标签，注意form里面不能嵌套form。

（5）如果是跳转链接就使用a标签，而不是自己写onclick跳转。a标签里面不能嵌套a标签。

（6）使用html5语义化标签，如导航使用nav，侧边栏使用aside，顶部和尾部使用header/footer，页面比较 独立的部分可以使用article，如用户的评论。

（7）如果是按钮就应该写一个button或者\<input type="button">，而不是写一个a标签设置样式，因为使用button可以设置disabled，然后使用CSS的:disabled，还有:active等伪类使用，例如在:active的时候设置按钮被按下去的效果。

（8）如果是标题就应该使用标题标签h1/h2/h3，而不是自己写一个\<p class="title">\</p>，相反如果内容不是标题就不要使用标题标签了。

（9）在手机上使用select标签，会有原生的下拉控件，手机上原生select的下拉效果体验往往比较好，不管是iOS还是Android，而使用\<input type="tel">在手机上回弹一个电话号码的键盘，\<input type="number">，\<input type="email">都会弹相应的键盘。

（10）如果是分隔线就使用hr标签，而不是自己写一个border-bottom的样式，使用hr容易进行检查。

（11）如果是换行文本就应该使用p标签，而不是写br，因为p标签可以用margin设置行间距，但是如果是长文本的话使用div，因为p标签里面不能有p标签，特别是当数据是后端给的，可能会带有p标签，所以这时容器不能使用p标签。

### 24、不要在https的链接里写http的图片

只要https的网页请求了一张http的图片，就会导致浏览器地址栏左边的小锁没有了，一般不要写死，写成根据当前域名的协议去加载，用//开头：

```html
<img src=”//static.chimeroi.com/hello-world.jpg”>
```

## 二、CSS编码规范

### 1、文件名规范

文件名建议用小写字母加中横线的方式。为什么呢？因为这样可读性比较强，看起来比较清爽，像链接也是用这样的方式，例如Stack Overflow的url：

> https://stackoverflow.com/questions/25704650/disable-blue-highlight-when-touch-press-object-with-cursorpointer

或者是github的地址：

> https://github.com/wangjeaf/ckstyle-node

那为什么变量名不用小写字母加下划线的方式，如family_tree，而是推荐用驼峰式的familyTree？C语言就喜欢用这种方式命名变量，但是由于因为下划线比较难敲(shift + -)，所以一般用驼峰式命名变量的居多。

引入CSS文件的link可以不用带type="text/css"，如下代码：

```html
<link rel="stylesheet" href="test.css">
```

因为link里面最重要的是rel这个属性，可以不要type，但是不能没有rel。

JS也是同样的道理，可以不用type，如下代码：

```html
<script src="test.js"></script>
```

没有兼容性问题。

### 2、属性书写顺序

属性的书写顺序对于浏览器来说没有区别，除了优先级覆盖之外。但是如果顺序保持一致的话，扫一眼可以很快地知道这个选择器有什么类型的属性影响了它，所以一般要把比较重要的属性放前面。比较建议的顺序是这样的：

![属性书写顺序](https://user-gold-cdn.xitu.io/2017/8/24/40dd68c5f860f12f01bfc9a0670506fd?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

你可能会觉得我平时差不多就是这么写的，那么说明你有一个比较好的素养。并且我觉得规则不是死，有时候可以灵活，就像你可能会用transform写居中，然后把left/top/transform挨在一起写了，我觉得这也是无可厚非的，因为这样可以让人一眼看出你要干嘛。

### 3、不要使用样式特点命名

有些人可能喜欢用样式的特点命名，例如：

```css
.red-font{
    color: red;
}
.p1{
    font-size: 18px;
}
.p2{
    font-size: 16px;
}
```

然后你在它的html里面就看到嵌套了大量的p1/p2/bold-font/right-wrap之类的类名，这种是不可取的，假设你搞了个red-font，下次UI要改颜色，那你写的这个类名就没用了，或者是在响应式里面在右边的排版在小屏的时候就会跑到下面去，那你取个right就没用了。有些人先把UI整体瞄了一下，发现UI大概用了3种字号18px/16px/14px，于是写3个类p1/p2/p3，不同的字号就套不同的类。这乍一看，好像写的挺通用，但是当你看他的html时，你就疯掉了，这些p1/p2/p3的类加起来写了有二三十个，密密麻麻的。我觉得如果要这样写的话还不如借助标题标签如下：

```css
.house-info h2{
    font-size: 18px;
}
.house-info h3{
    font-size: 16px;
}

```

因为把它的字号加大了，很可能是一个标题，所以为什么不直接用标题标签，不能仅仅单向因为标题标签会有默认样式。

**类的命名应当使用它所表示的逻辑意义**，如singup-success-toast、request-demo、agent-protrait、company-logo等等。

如果有些样式你觉得真的特别通用，那可以把它当作一个类，如clearfix，或者有些动画效果，有几个地方都要用到，我觉得这种较为复杂并且通用的可以单独作为一个类。但是还是趋向于使用意义命名。

### 4、不要使用hack

有些人在写CSS的时候使用一些hack的方法注释，如下：

```css
.agent-name{
    float: left;
    _margin: 10px;
    //padding: 20px;
}

```

这种方法的原理是由于//或者_开头的CSS属性浏览器不认识，于是就被忽略，分号是属性终止符，从//到分号的内容都被浏览器忽略，但是这种注释是不提倡的，要么把.css文件改成.less或者.scss文件，这样就可以愉快地用//注释了。

还有一些专门针对特定浏览器的hack，如*开头的属性是对IE6的hack。不管怎么样都不要使用hack。

### 5、选择器的性能

选择器一般不要写超过3个，有些人写sass或者less喜欢套很多层，如下：

```less
.listings-list{
    ul{
        li{
            .bed-bath{
                p{
                     color: #505050;
                }
            }
        }
    }
}
```

一个容器就套一层，一层一层地套下来，最底层套了七八层，这么长的选择器的性能比较差，因为Chrome里面是用递归从最后一个选择器一直匹配到第一个，选择器越多，匹配的时间就越长，所以时间会比较长，并且代码的可读性也比较差，为了看到最里面的p标签的样式是哪个的，我得一层层地往上看，看是哪里的p。代码里面缩进了7、8层看起来也比较累。

一般只要写两三个比较重要的选择器就好了，不用每个容器都写进去，重要的目标元素套上class或者id。

最后一个选择器的标签应该少用，因为如果你写个.container div{}的话，那么页面上所有的div每一次都匹配到，因为它是从右往左匹配的，这样写的好处是html不用套很多类，但是扩展性不好，所以不要轻易这样用，如果要用需要仔细考虑，合适才使用，最起码不能滥用。

### 6、避免选择器误选

有时候会出现自己的样式受到其他人样式的影响，或者自己的样式不小心影响了别人，有可能是因为类的命名和别人一样，还有可能是选择器写的范围太广，例如有人在他自己的页面写了：

```css
* {
    box-sizing: border-box;
}
```

结果导致在他的页面的公用弹框样式挂了。一方面不要写*全局匹配选择器，不管从性能还是影响范围来说都太大了，例如在一个有3个子选择器的选择器：

```css
.house-info .key-detail .location{}
```

在三个容器里面，*都是适用的，并且有些熟悉是会继承的，像font-size，会导致这三个容器都有font-size，然后一层层地覆盖。

还有一种情况是滥用了:first-child、:nth-of-type这种选择器，使用这种选择器的后果是扩展性不好，只要html改了，就会导致样式不管用了，或者影响到了其他无关元素。但是并不是说这种选择器就不能用，只要用得好还是挺方便的，例如说在所有的li里面要让最后一个li的margin-left小一点，那么可以这么写：

```css
.listing-list li:last-child{
    margin-left: 10px;
}

```

这了能比你直接套一个类强。但是不管怎么样，不能滥用，合适的时候才使用，而不是仅仅为了少写类名。

