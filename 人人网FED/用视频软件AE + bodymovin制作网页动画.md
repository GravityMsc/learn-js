我们知道，制作动画有多种形式，可以用CSS的animation，也可以用Canvas，或者是用JS控制CSS的属性形成动画。我们经常使用CSS做一些比较简单的动画，像过渡、加载的动画，对于一些比较复杂的，可以会做成gif，或者使用Canvas，使用Canvas的控制粒度可以很细，同时工作量相对也比较大。做动画还有其他的方式，那就是使用After Effect(AE)/Flash/Premiere(Pr)/会声会影等视频软件，这种可视化的制作方式相对于直接写代码来说，会更容易简单自然。做动画本身应该使用工具进行制作，但是这种视频软件做出来的动画最后都是生成视频文件，并且通常体积还很大，没有办法直接移植到网页上去。

然而好消息是，现在我们可以使用AE做动画，然后使用bodymovin插件导出成html文件进行播放。AE是Adobe推出的一个很出名的视频后期处理软件，有些电影就是用AE做的，比如变形金刚，还有人把AE当成加强版PS使用。也就是说假如我们可以用AE作出一些电影级别的效果，然后用html播放，那是一件多么酷炫的事情。

## 1. 安装bodymovin

bodymovin是一个AE的开源第三方扩展，[github地址，现在改名叫lottie](https://github.com/airbnb/lottie-web)，上面可以下载这个插件，然后再安装一个[ZXPInstaller](http://zxpinstaller.com/)来安装这个文件，然后重启AE就可以了，当然前提是你要安装一个AE。它支持一下AE版本：

> After Effects CC 2017, CC 2015.3, CC 2015, CC 2014

安装完之后，点击AE的菜单Window -> Extensions -> Bodymovin就会弹出一个窗口：

![bodymovin](https://user-gold-cdn.xitu.io/2017/8/6/a47fd88e8b51d36e90eb5d801a2c1968?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

## 2. 使用AE制作动画

我相信很多人都没有玩过AE，所以这里我简单地介绍一下。首先新建一个工程（project），然后新建一个合成（composition），选择1080p/29fps，时长为10s，它就会创建一个10s的合成。如下时间轴面板的显示：

![创建一个AE的合成](https://user-gold-cdn.xitu.io/2017/8/6/6de9072de461d69b48b7aeb43c786d82?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

这个时间轴将会是频繁操作的地方。点击文字工具，在上方的预览窗口选中一个位置点击创建文字，然后把它拖到窗口外面，因为我们准备做一个文字从外面进来的动画，所以刚开始它是在外面的。把上图右边的蓝色竖线表示的时间线拖到0s的位置，然后在左边的文字图层的Position竖线打一个关键帧，如下图所示：

![关键帧](https://user-gold-cdn.xitu.io/2017/8/6/d7ea5182ead25d6eaa0392be6e82a608?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

然后把时间线挪到3s的位置，改变文字的Position，把它挪到窗口的中间，这个时候AE会自动在时间线的位置打一个关键帧，如下图所示：

![制造关键帧](https://user-gold-cdn.xitu.io/2017/8/6/8811a8c381f606ef5167331e689e3f6d?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

然后按一下空格键预览，预览窗口就会播放起了我们刚刚设定的动画：

![制作的动画](https://user-gold-cdn.xitu.io/2017/8/6/2d9553848f913291d6f7f55a91585196?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

你会发现，这个过程不是和CSS的keyframe动画一眼的么？没错！动画的原理都是一样，通过设定关键帧制作动画。现在来比较一下用AE和用CSS/Canvas做这个动画的区别。

## 3. 关键帧动画

现在用CSS做这个动画，如下代码所示：

```html
<style>
    .text {
        animation: move 3s linear infinite;
    }
    
    @keyframes move{
        from{
            transform: translateX(-320px);
        }
        to{
            transform: translateX(100px);
        }
    }
</style>
<div class="container">
    <p class="text">
        Hello, frontend
    </p>
</div>
```

我们给animation添加一个动画，这个动画有两个关键帧，分别在0%和100%的位置，需要变化的是transform的属性。这段代码在浏览器运行，救护有刚刚用AE做的动画的效果了。如果用Canvas呢，应该怎么实现呢？

如下代码所示：

```html
<canvas id="text-move" width="600" height="400"></canvas>
<script>
    !function(){
        window.requestAnimationFrame(draw);
        var canvas = document.querySelector("#text-move"),
            ctx = canvas.getContext("2d");
        
        function draw(){
            // 计算文字position
            var textPosition = getPosition();
            drawText();
            window.requestAnimationFrame(draw);
        }
    }();
</script>
```

这个是canvas动画的基本框架，先注册requestAnimationFrame的draw函数，使得浏览器在重写绘制屏幕时会先调一下这个函数，理想情况下1s会绘制60幅图片，也就是说1s为60帧/60fps。

上面diam最关键的地方是在于计算文字位置position，同样地，也是要先设定初始位置和终点位置还有动画时间，从而知道移动的速度v，即每1s多少距离，记录一个动画开始时间，然后再每次draw的时候用Date.now()获取当前时间减掉开始时间，就得到时间t，然后用v*t就可以得到位移。这就是用Canvas做动画的基本原理，我们看到，用Canvas需要自己实现一个关键帧系统。

从抽象级别来看的话，AE > CSS >> Canvas，使用AE我只需要拖一拖，然后打上几个关键帧，而使用CSS，我需要把我的操作写成代码，而使用Canvas我需要你从0开始一点一点去控制，当然你可以使用一些动画和游戏的引擎提高效率。所以如果有一个可视化界面让你去完成一些复杂的操作，和让你一行一行去写代码的方式选择的话，我想大部分人应该会选择前者。当然这两者的区别不仅仅是操作上的简便性，使用AE借用插件还可以快速地制作出一些复杂的效果。

## 4. bodymovin小试牛刀

刚刚已经用AE做了一个最简单的动画，现在用bodymovin把它导出来。打开bodymovin，选中合层，选择输出路径，如下图所示：

![导出动画](https://user-gold-cdn.xitu.io/2017/8/6/bc6a001638d858932d8564e4c9dfa935?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

然后点击Render，完成它会输出一个json文件，打开这个导出的文件：

> {"v":"4.10.1","fr":29.9700012207031,"ip":0,"op":95.0000038694293,"w":1920,"h":1080,"nm":"Comp 1","ddd":0,"assets":[],"fonts":{"list":[{"origin":0,"fPath":"","fClass":"","fFamily":"Myriad Pro","fWeight":"","fStyle":"Regular","fName":"MyriadPro-Regular","ascent":70.9991455078125}]},"layers":[{"ddd":0,"ind":1,"ty":5,"nm":"hello, frontend","sr":1,"ks":{"o":{"a":0,"k":100,"ix":11},"r":{"a":0,"k":0,"ix":10},"p":{"a":1,"k":[{"i":{"x":0.833,"y":0.833},"o":{"x":0.167,"y":0.167},"n":"0p833_0p833_0p167_0p167","t":0,**"s":[-1017,692,0],"e":[458,692,0]**,"to":[245.83332824707,0,0],"ti":[-245.83332824707,0,0]},{"t":90.0000036657751}],"ix":2},"a":{"a":0,"k":[0,0,0],"ix":1},"s":{"a":0,"k":[100,100,100],"ix":6}},"ao":0,"t":{"d":{"k":[{"s":{"s":164,"f":"MyriadPro-Regular","t":"hello, frontend","j":0,"tr":0,"lh":196.8,"ls":0,"fc":[0,0.64,1]},"t":0}]},"p":{},"m":{"g":1,"a":{"a":0,"k":[0,0],"ix":2}},"a":[]},"ip":0,"op":300.00001221925,"st":0,"bm":0}]}

这个文件记录了所有动画的过程，如上加粗字体是我们刚刚打的两个关键帧的位置。然后安装一下bodymovin的引擎，可在github上面下载bodymovin.js或者是npm install一下：

> npm install bodymovin

然后就可以使用bodymovin了，如下html：

```html
<!DOCType html>
<html>
<head>
    <meta charset="utf-8">
</head>
<body>
    <div id="animation-container" style="width:100%"></div>
    <script src="node_modules/bodymovin/build/player/bodymovin.js"></script>
    <script src="index.js"></script>
</body>
</html>
```

index.js如下代码所示：

```javascript
var animation = bodymovin.loadAnimation({
    container: document.getElementById('animation-container'),
    renderer: 'canvas', //svg、html
    loop: true,
    autoplay: true,
    path: 'data.json'
})
```

调用它的loadAnimation的API，传几个参数，它支持canvas/svg/html三种形式，也就是说它可以用canvas做动画，也可以用svg和html，其中canvas的性能最高，但是canvas有很多效果不支持。data.json的位置通过path告诉它。所有的动画就通过改变path指向的data.json文件区分，而其他的参数不用变。也就是说所有的的动画内容和效果都是通过data.json控制的。

现在在浏览器上面运行一下，你会发现报了一个错误：

![运行错误](https://user-gold-cdn.xitu.io/2017/8/6/e0f474c917f971df0756f41dd04b2d45?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

后来发现这个错误是因为文字的原因，如果是用canvas的方式的时候要把文字导出成svg的形式，而不是一个纯文本然后通过设置font-family，这个可以在bodymovin里面进行设置，如下图所示：

![导出字体文件](https://user-gold-cdn.xitu.io/2017/8/6/c482b6046a44a13702ae7d2640b9d587?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

还可以直接导出一个完整的demo，直接打开html就可以运行，这个比较方便。效果如下：

![完整的demo](https://user-gold-cdn.xitu.io/2017/8/6/9c12a5d3c1c118e33b016f3cec2a4379?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

并且我们发现，它的大小和位移都是相对于容器的，当你把窗口拉小，它也会跟着变小。当使用svg的时候，它是用JS控制svg path标签的transform：

![svg动画](https://user-gold-cdn.xitu.io/2017/8/6/f5759ed63f474b61b5ef4d871b9336e0?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

当使用html时，它是控制CSS的transform：

![CSS动画](https://user-gold-cdn.xitu.io/2017/8/6/e446764ea02c769caf4027dad1771888?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

我们的一个hello，world的工程已经可以跑起来，bodymovin能支持多复杂的动画呢？

## 5. AE的摄像机

用AE做动画的时候经常会用到AE的摄像机图层，所谓摄像机就是一个视角，默认情况下这台摄像机是从正前方中间拍过去的，我们可以改版这台摄像机的位置，如把摄像机往前推那么内容就会放大，把摄像机往左右移动，那么看到的内容就会发生倾斜，它有很多仿摄像机的参数可以控制：

![AE的摄像机](https://user-gold-cdn.xitu.io/2017/8/6/5bc15a943fead62cad340b1fac1951e7?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

摄像机属性都可以通过打关键帧做动画，现在我们加上摄像机做3d的动画。做完后，如果还用canvas的话，它会提示你不能使用canvas，因为它目测不支持WebGL转换，如下图所示：

![canvas不支持WebGL](https://user-gold-cdn.xitu.io/2017/8/6/f98a4c18e14750502610ab7612750f67?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

提示说使用了一个3d camera，尝试使用HTML renderer，这里要改成html。最后的效果如下：

![最终的动画效果](https://user-gold-cdn.xitu.io/2017/8/6/4a2176b61038d09c01b1760e9872ee1f?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

通过检查，可以看到摄像机也是用transform的matrix控制的。完整的demo可见[这个链接](https://fed.renren.com/html/bodymovin/simple-demo/index.html)。

然后我们再继续做更复杂的动画。

## 6. 复杂动画

在所有特效里面，笔者最喜欢的是粒子效果，这种效果也是电影里面经常用到的特效，比如冰雪女王的冰雪魔法：

![冰雪魔法](https://user-gold-cdn.xitu.io/2017/8/6/22476ecf9df97323213851078e1bd7ee?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

还有文字的粒子效果：

![文字的粒子效果](https://user-gold-cdn.xitu.io/2017/8/6/9c77dcdb28020e45d2c4cbd3ede4e621?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

但是这个效果我试了一下没有办法导出来，这种效果本身就比较复杂，渲染起来比较耗时，在html实时播放也不太现实。

还有有时候会报一些奇怪的错误，最常报的一个错误是这个：

> bodymovin.js:9249 Uncaught TypeError: this.addTo3dContainer is not a function

可能是使用了一些特定效果，触发了它的bug。

但是不要沮丧，我们还是可以导出一些复杂的效果的，做动画这种关键还是在于idea。

例如可以做一个装饰的小动画，如下所示：

![爆炸动画](https://user-gold-cdn.xitu.io/2017/8/6/9db07d30ff8c91dbb17ac65b8243e35c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

一个完整的demo[见这个网址](https://fed.renren.com/html/bodymovin/comic-decorater/index.html)。

还可以做一个相册视频，效果如下：

![相册视频](https://user-gold-cdn.xitu.io/2017/8/6/dc9d8f870279631dddac4c48d8111e47?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

完整的demo[见这个网址](https://fed.renren.com/html/bodymovin/picture-gallary/index.html)。

如果是做成一个1080p的视频，1.5分钟的MP4格式就算压得比较厉害也要100多MB，但是现在我只要加载一个90kb的json文件和一个240kb的库文件以及10Mb左右的图片就可以了。并且里面的文字和图形还是矢量的。

现在来看一下CPU消耗，可以看到这个相册视频svg动画还是很耗CPU的，但是你开一个视频播放器也同样挺消耗CPU资源的。

![动画的CPU消耗](https://user-gold-cdn.xitu.io/2017/8/6/b890e34c8f5a5ff7455455f3597d32b3?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

不管怎么样，bodymovin提供了另外一种做网页动画的全新形式，摆脱那种纯代码控制的黑暗，甚至你都不用学Canvas和WebGL，也可以做出很酷炫的动画。大师无疑这种方式存在一种弊端，那就是没办法添加参数控制，例如我做一个愤怒的小鸟，我得通过拉弓的方向和力度以及小鸟的重量去计算它的轨迹。但是用AE做的只能是纯动画。所以平时可以两者结合。

