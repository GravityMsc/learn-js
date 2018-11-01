在上一篇《[我是怎么让网站用上HTML5 Manifest](https://fed.renren.com/2017/09/29/manifest/)》介绍了怎么用Manifest做一个离线网页应用，结果被广大网友吐槽说这个东西已经被deprecated，移出web标准了，现在被Service Worker替代了，不管怎么样，Manifest的一些思想还是可以借用的。笔者又将网站升级到了Service Worker，如果是用Chrome等浏览器就用Service Worker做离线缓存，如果是Safari浏览器就还是用Manifest，读者可以打开这个网站fed.renren.com感受一下，断网也是能正常打开。

## 1、什么是Service Worker

Service Worker是谷歌发起的实现PWA（Progressive Web App）的一个关键角色，PWA是为了解决传统Web APP的缺点：

（1）没有桌面入口

（2）离线无法使用

（3）没有Push推送

那么Service Worker的具体表现是怎么样的呢？如下图所示：

![Service Worker](https://user-gold-cdn.xitu.io/2017/10/4/601308123aaf08f8077b8476ae8d2f58?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

Service Worker是在后台启动的一条服务Worker线程，上图我开了两个标签页，所以显示了两个Client，但是不管开多少个页码都只有一个Worker在负责管理。这个Worker的工作就是把一些资源缓存起来，然后拦截页面的请求，先看下缓存库里有没有，如果有的话就从缓存里取，响应200，反之没有的话就走正常的请求。具体来说，Service Worker结合Web App Manifest能完成以下工作（这也是PWA的检测标准）：

![PWA的检测标准](https://user-gold-cdn.xitu.io/2017/10/4/7cc003776b02a85782fba20cc82582da?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

包括能够离线使用、断网时返回200、能提示用户把网站添加一个图标到桌面上等。

## 2、Service Worker的支持情况

Service Worker目前只有Chrome/Firefox/Opera支持：

![Service Worker的支持情况](https://user-gold-cdn.xitu.io/2017/10/4/17e345518d6198ee3342de27183cb4bb?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

Safari和Edge也在准备支持Service Worker，由于Service Worker是谷歌主导的一项标准，对于生态比较封闭的Safari来说也是迫于形势开始准备支持了，在Safari TP版本，可以看到：

![Safari TP版本支持Service Worker](https://user-gold-cdn.xitu.io/2017/10/4/be3faa1a620dba74e427114eff9c8cad?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

在实验功能（Experimental Features）里已经有Service Worker的菜单项了，只是即使打开也是不能用，会提示你还没有实现：

![Safari Service Worker还没有实现](https://user-gold-cdn.xitu.io/2017/10/4/eec491b2d1e3584be6c758c7935744ca?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

但不管如何，至少说明Safari已经准备支持Service Worker了。另外还可以看到在今年2017年9月发布的Safari 11.0.1版本已经支持WebRTC了，所以Safari还是一个上进的孩子。

Edge也准备支持，所以Service Worker的前景十分光明。

## 3、使用Service Worker

Service Worker的使用套路是先注册一个Worker，然后后台启动一条线程，可以在这条线程启动的时候去加载一些资源缓存起来，然后监听fetch事件，在这个事件里拦截页面的请求，先看下缓存里有没有，如果有直接返回，否则正常加载。或者是一开始不缓存，每个资源请求后再拷贝一份缓存起来，然后下一次请求的时候缓存里就有了。

### （1）注册一个Service Worker

Service Worker对象是在window.navigator里面，如下代码：

```javascript
window.addEventListener("load", function() {
    console.log("Will the service worker register?");
    navigator.serviceWorker.register('/sw-3.js')
    .then(function(reg){
        console.log("Yes, it did.");
    }).catch(function(err) {
        console.log("No it didn't. This happened: ", err)
    }); 
});
```

在页面load完之后注册，注册的时候传一个js文件给它，这个js文件就是Service Worker的运行环境，如果不能成功注册的话就会抛出异常，如Safari TP虽然有这个对象，但是会抛出异常无法使用，就饿可以在catch里面处理。这里有个问题是为什么需要在load事件启动呢？因为你要额外启动一个线程，启动之后你可能还会让它去加载资源，这些都是需要占用CPU和带宽的，我们应该保证页面能正常加载完，然后再启动我们的后台线程，不能与正常的页面加载产生竞争，这个在低端移动设备上意义比较大。

还有一点需要注意的是Service Worker和Cookie一样是由Path路径的概念的，如果你设定一个cookie假设叫time的path=/page/A，在/page/B这个页面是不能够获取到这个cookie的，如果设置cookie的path为根目录/，则所有页面都能获取到。类似地，如果注册的时候使用的js路径为/page/sw.js，那么这个Service Worker只能管理/page路径下的页面和资源，而不能够处理/api路径下的，所以一般把Service Worker注册到顶级目录，如上面代码的"sw-3.js"，这样这个Service Worker就能接管页面的所有资源了。

