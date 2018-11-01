Manifest是用来做离线页面的，即使断网也能正常打开页面，用起来简单，但是在实际使用中存在以下问题：

（1）如何自动缓存所有的页面资源？因为manifest不能使用*通配符进行cache

（2）如果网站资源更新，怎么让manifest文件自动更新？不然如果用户不清缓存即使联网也会加载老页面

我觉得很多网站没有使用Manifest是因为上面提到的两个原因，有些人有尝试过，但使用起来比较麻烦，离线应用价值好像不太大。但是使用Manifest还是有很多好处的，特别是像博客等之类的偏向于展示的网站，或者是在线APP，这种万丈的数据动态变化频率比较低，不予要频繁地向服务器请求数据，这样当用户需要频繁退回首页或者频繁地在几个页面来回切换的时候，由于几乎所有的资源都在本地，所以加载起来是瞬时的。

## 1、使用Manifest

使用Manifest很简单，就是在html标签上加一个manifest属性：

```html
<html manifest="/static/manifest/home.appcache">
```

这个属性指向一个manifest的文件，这个文件指明了当前页面哪些资源需要进行离线缓存，如下home.appcache：

```html
CACHE MANIFEST
#9/27/2017, 3:04:25 PM
#html
https://github.com/
#img
https://assets-cdn.github.com/images/modules/site/universe-octoshop.png
https://assets-cdn.github.com/images/modules/site/universe-wordmark.png
#css
https://assets-cdn.github.com/assets/frameworks-bedfc518345231565091.css
#js
https://assets-cdn.github.com/assets/compat-94eba6e3cd1fa18902d9.js
NETWORK:
*

FALLBACK
https://github.com/ /html/manifest/html/home.html
```

这个文件第一行必须以CACHE MANIFEST开头，否则浏览器解析会报错，注释使用#开头，在这一行下面跟着需要缓存的资源，接着的NETWORK表示哪些资源需要联网加载，一般需要写成NETWORK *，表示除了在CACHE外的其他所有资源都需要联网，包括一些动态请求，如果你不是写的\* ，而是写了具体路径，那些既没有在CACHE的，也没有在NETWORK的就会报加载失败的错误，如下所示：

![manifest加载失败](https://user-gold-cdn.xitu.io/2017/9/29/eda55cfdf5cddc0280965d49a4a887c5?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

即使联网也会这样，所以一般写成*。

FALLBACK表示替代静态资源，这些资源加载不到就替代加载哪些资源，如上面的文件https://github.com访问不了就使用一个静态的的html访问：https://github.com/html/manifest/html/home.html。

打开支持Manifest的网站，例如fed.renren.com，可以观察到Chrome控制台cache的过程：

![Manifest缓存的过程](https://user-gold-cdn.xitu.io/2017/9/29/c6ce28fc8086ee9657f7708d6640cc62?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

然后再刷新页面，你会发现页面机会所有资源都是在本地缓存取的，如下图所示：

![Mainfest本地缓存](https://user-gold-cdn.xitu.io/2017/9/29/54f23572e9d7f187636cd512ae0f7308?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

并且你把网端了，刷新页面，页面依旧能够正常加载出来。这个在Chrome/Firefox/Safari等浏览器的支持。

除了Manifest之外，还有另外一个缓存的手段，就是设置HTTP报文头的Cache-Control字段进行缓存，这个可以缓存JS/CSS/图片资源，但是如果你把HTML也缓存就会有一个问题，如果用户不清除缓存，即使你的页面更新了，用户仍然会加载老的页面，知道缓存设定Max-Age时间到了。所以用Manifest可以解决这个问题。

Manifest怎么知道当前页面数据更新了呢？只要把mainfest文件如上面的home.appcache更改一下就可以了，浏览器打开页面时都回去加载这个文件，一旦发现这个文件发生了变化下次刷新的时候就会重新加载所有Cache的文件，最简单的可以把注释里的时间改成当前的时间就可以了：

```html
#9/29/2017, 9:08:49 AM
```

所以当网站的资源发生更改就可以改变这个Manifest的内容，进而联网的浏览器就能进行更新。

使用Manifest需要注意以下问题：

（1）Manifest有大小限制，它其实也算本地存储，本地存储一般每个域有限制使用的空间，PC Chrome是5MB，参考[如下表格](http://grinninggecko.com/2011/02/24/developing-cross-platform-html5-offline-app-1/)：

| Browser                    | Application Cache (AppCache) Storage Limit |
| -------------------------- | ------------------------------------------ |
| Safari Desktop (Mac & Win) | Unlimited                                  |
| Safari Mobile (iOS)        | 10 MB                                      |
| Chrome Desktop (Mac & Win) | 5 MB *                                     |
| Chrome Mobile (Android)    | Unlimited **                               |
| Firefox 4 Beta             | Unlimited (with user prompt)               |
| IE                         | No idea. It sucks. ***                     |

（2）Manifest文件如home.appcache不能跨域，如果跨域需要支持CORS

（3）Manifest Cache的资源不能跨域，同样如果跨域该资源需要支持CORS，一般浏览器会自动处理

## 2、解决Manifest的自动生成和更新问题

由于Manifest不能使用通配符匹配资源，所以需要把要进行cache的资源一个个列出来，而网站的内容经常是动态更新的，所以这个就比较麻烦。为此笔者写了一个自动生成Manifest的NPM包[generate-manifest](https://www.npmjs.com/package/generate-manifest)，用起来非常简单：

```bash
npm install -g generate-manifest
generate-manifest --url=https://github.com
```

它就会生成一个home.appcache的Manifest文件，这个我人家包括页面上的img/js/css的资源链接：

```bash
CACHE MANIFEST
#9/27/2017, 3:04:25 PM
#html
https://github.com/
#img
https://assets-cdn.github.com/images/modules/site/universe-octoshop.png
https://assets-cdn.github.com/images/modules/site/universe-wordmark.png
#css
https://assets-cdn.github.com/assets/frameworks-bedfc518345498ab3204d330c1727cde7e733526a09cd7df6867f6a231565091.css
#js
https://assets-cdn.github.com/assets/compat-91f98c37fc84eac24836eec2567e9912742094369a04c4eba6e3cd1fa18902d9.js
NETWORK:
*

FALLBACK
https://github.com/ /html/manifest/html/home.html
```

还可以支持其他参数定制。详见：[generate-manifest](https://www.npmjs.com/package/generate-manifest)。

这样就就解决了自动生成的问题，自动更新应该怎么办呢？

由于我是一个博客网站，网站内容发生变化的地方主要有：1.发表/更改博客；2.用户发表评论；3.网站的浏览量发生变化，第一个解决的方法写了一个借口，只要发表博客就调一下这个接口去生成一个新的manifest文件：

> https://fed.renren.com/refresh-manifest.php?link=https://fed.renren.com/2017/09/26/manifest/

然后就会调上面的generate-manifest的包，生成一个manifest.appcache的文件，在html里面是根据路径的最后一个部分决定manifest的名字：

```php+HTML
<?php
    $uri = "$_SERVER[REQUEST_URI]";
    $uriArray = explode("/", $uri);
    $uriName = count($uriArray) > 2 ? $uriArray[count($uriArray) - 2] : "home";
?>
<!DOCTYPE html>
<html <?php language_attributes(); ?> manifest=<?php echo "/html/manifest/appcache/$uriName.appcache"?>>
```

这和生成的文字名一一对应。

第二个问题：阅读量数据变化的问题——写一个Linux定时任务，使用crontab添加一个定时任务，执行crontab -e添加：

```shell
0 3 * * * /home/fed/manifest/update-all.sh
```

上面的意思是每天3:00的时候跑一下update-all.sh这个脚本，这个甲苯把所有页面的更新命令都写进去：

```:zero:
generate-manifest --url=https://fed.renren.com
generate-manifest --url=https://fed.renren.com/page/2/
generate-manifest --url=https://fed.renren.com/page/3/
#..其它...
```

第一点提到的发表文章，也会添加一行命令到这个脚本里面。

由于阅读量这个数据不是很重要，所以一填更新一次就好了。这样可以让用户在同一天的操作又缓存。如果第二天再来看的话就更新一下。

因此基本上就解决了自动更新的问题。

还有一个问题是，Manifest改了之后的第一次刷新还是老的页面，只有第二次刷新的时候才是对的，所以我没希望改了Manifest之后能够一刷新就是新的，而不是之前缓存的那个，也不需要刷两次。

那么怎么办呢？Manifest有一个更新的事件，一旦Manifest文件有更新就会触发这个事件，所以我们可以监听这个事件，然后自动刷新页面让页面重新加载就可以了，如下代码：

```javascript
function onUpdateReady() {
    window.location.reload(true);
}
window.applicationCache.addEventListener('updateready', onUpdateReady);
if(window.applicationCache.status === window.applicationCache.UPDATEREADY) {
    onUpdateReady();
}
```

综上，我们很好地利用Manifest做了一个离线页面应用，解决了自动生成和自动更新的问题。即使用户没有离线，第二次加载的资源都是在本地缓存的，所以当用户在几个页面来回切换的时候这个速度是很快的，如很多人可能会在主页的列表和内容页之间来回切换。

虽然Manifest已经被deprecated了，被Service Worker取代了，但是由于它的简单易用以及兼容性好，我们还是可以用一用。

