声音无法自动播放这个在iOS/Android上面一直是个惯例，桌面版的Safari在2017年的11版本也宣布禁掉带有声音的多媒体自动播放功能，紧接着在2018年4月份发布的Chrome 66也正式关掉了声音自动播放，也就是说\<audio autoplay>\</audio>  \<video autoplay>\</video>在桌面版浏览器也将失效。

最开始移动端浏览器是完全禁止音视频自动播放的，考虑到了手机的带宽以及对电池的消耗。但是后来又该了，因为浏览器厂商发现网页开发人员可能会使用GIF动态图代替视频实现自动播放，正如[iOS文档](https://webkit.org/blog/6784/new-video-policies-for-ios/)所说，使用GIF的带宽流量是Video（h264）格式的12倍，而播放性能消耗是2倍，所以这样对用户反而是不利的。又或者是使用Canvas进行hack，如[Android Chrome文档](https://developers.google.com/web/updates/2016/07/autoplay)提到。因此浏览器厂商放开了多媒体自动播放的限制，只要具备以下条件就能自动播放：

（1）没音频轨道，或者设置了muted属性

（2）在视图里面是可见的，要插入到DOM里面并且不是dispaly: none或者Visibility: hidden的，没有滑出可视区域。

换句话说，只要你不开声音扰民，且对用户可见，就让你自动播放，不需要你去使用GIF的方法进行hack。

桌面版的浏览器在近期也使用了这个策略，如升级后的Safari 11的说明：

![策略](https://user-gold-cdn.xitu.io/2018/5/13/1635520d2d1f384f?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

以及[Chrome文档的说明](https://developers.google.com/web/updates/2017/09/autoplay-policy-changes)：

![漫画说明](https://user-gold-cdn.xitu.io/2018/5/13/1635520d2cee425f?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

这个策略无疑对视频网站的冲击最大，如在Safari打开tudou的提示：

![tudou的提示](https://user-gold-cdn.xitu.io/2018/5/13/1635520d2d0a0c10?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

添加了一个设置向导。Chrome的禁止更加人性化，它有一个[MEI的策略](https://developers.google.com/web/updates/2017/09/autoplay-policy-changes#mei)，这个策略大概是说只要用户在当前网页主动播放超过7s的音视频（视频窗口不能小于200 x 140），就允许自动播放。

对于网页开发人员来说，应当如何有效地规避这个风险呢？

Chrome的文档给了一个最佳实践：先把音视频加一个muted的属性就可以自动播放，然后再显示一个声音被关掉的按钮，提示用户点一下打开声音。对于视频来说，确实可以这样处理，而对于音频来说，很多人是监听页面点击事件，只要点一次了就开始播放声音，一般就是播放个背景音乐。但是如果对于有多个声音资源的页面来说如何自动播放多个声音呢？

首先，如果用户还没进行交互就调用播放声音的API，Chrome会这么提示：

> DOMException: play() failed because the user didn't interact with the document first.

Safari会这么提示：

> NotAllowedError: The request is not allowed by the user agnt or the platform in the current context, possibly because the user denied permission.

Chrome报错提示最为友善，意思是说，用户还没有交互，不能调用play函数。用户的交互包括哪些呢？包括用户触发的touchend，click，doubleclick或者是keydown事件，在这些事件里面就能调用play函数。

所以上面提到很多人是监听整个页面的点击事件进行播放，不管点的哪里，只要点了就行，包括触摸下滑。这种方法只适用于一个声音资源，不适用多个声音，多个声音应该怎么破呢？这里并不是说要和浏览器对着干，“逆天而行”，我们的目的还是为了提升用户体验，因为有些场景如果能自动播放确实比较好，如一些答题的场景，需要听声音进行答题，如果用户在答题的过程中能依次自动播放相应题目的声音，确实比较方便。同时也是讨论声音播放的技术实现。

原生播放视频应该就只能使用video标签，而原生播放音频除了使用audio标签之外，还有另外一个API叫AudioContext，它是能够用来控制声音播放并带了很多丰富的操控接口。调用audio.paly必须在点击事件里面响应，而使用AudioContext的区别在于只要用户点过页面任何一个地方之后就都能播放了。所以可以用AudioContext取代audio标签播放声音。

