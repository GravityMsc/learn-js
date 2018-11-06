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

### （2）Service Worker安装和激活

注册完之后，Service Worker就会进行安装，这个时候会触发install事件，在install事件里面可以缓存一些资源，如下sw-3.js：

```javascript
const CACHE_NAME = "fed-cache";
this.addEventListener("install", function(event) {
    this.skipWaiting();
    console.log("install service worker");
    // 创建和打开一个缓存库
    caches.open(CACHE_NAME);
    // 首页
    let cacheResources = ["https://fed.renren.com/?launcher=true"];
    event.waitUntil(
        // 请求资源并添加到缓存里面去
        caches.open(CACHE_NAME).then(cache => {
            cache.addAll(cacheResources);
        })
    );
});
```

通过上面的操作，创建和添加了一个缓存库叫fed-cache，如下Chrome控制台所示：

![Chrome控制台查看缓存库](https://user-gold-cdn.xitu.io/2017/10/4/d852807ee179a0e3956116dcfd167215?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

Service Worker的API基本上都是返回Promise对象避免堵塞，所以要用Promise的写法。上面在安装Service Worker的时候就把首页的请求给缓存起来了。在Service Worker的运行环境里面它有一个caches的全局对象，这个是缓存的入口，还有一个常用的clients的全局对象，一个client对应一个标签页。

在Service Worker里面可以使用fetch等API，它和DOM是隔离的，没有windows/document对象，无法直接操作DOM，无法直接和页面交互，在Service Worker里面无法得知当前页码打开了、当前页码的url是什么，因为一个Service Worker管理当前打开的几个标签页，可以通过clients知道所有页面的url。还有可以通过postMessage的方式和主页面相互传递消息和数据，进而做些控制。

install完之后，就会触发Service Worker的active事件：

```javascript
this.addEventListener("active", function(event) {
    console.log("service worker is active");
});
```

Service Worker激活之后就能够监听fetch事件了，我们希望每获取一个资源就把它缓存起来，就不用像上一篇提到的Manifest需要先生成一个列表。

你可能会问，当我刷新页面的时候是不是又重新注册安装和激活了一个Service Worker？虽然又调了一次注册，但并不会重新注册，它发现“sw-3.js”这个已经注册了，就不会再注册了，进而不会触发install和active事件，因为当前Service Worker已经是active状态了。当需要更新Service Worker时，如变成“sw-4.js”或者改变sw-3.js的文本内容，就会重新注册，新的Service Worker会先install然后进入waiting状态，等到重启浏览器时，老的Service Worker就会被替换掉，新的Service Worker进入active状态，如果不想等到重新启动浏览器可以像上面一样在install里面调用skipWaiting：

```javascript
this.skipWaiting();
```

### （3）fetch资源后cache起来

如下代码，监听fetch事件做些处理：

```javascript
this.addEventListener("fetch", function(event) {
    event.respondWith(
        caches.match(event.request).then(response => {
            // cache hit
            if (response) {
                return response;
            }

            return util.fetchPut(event.request.clone());
        })
    );
});
```

先调用caches.match看一下缓存里面是否有了，如果有直接返回缓存里的response，否则的话正常请求资源并把它放到cache里面。放在缓存里资源的key值是Request对象，在match的时候，需要请求的url和header都一致才是相同的资源，可以设定第二个参数ignoreVary：

```javascript
caches.match(event.request, {ignoreVary: true})
```

表示只要请求url相同就认为是同一个资源。

上面代码的util.fetchPut是这样实现的：

```javascript
let util = {
    fetchPut: function (request, callback) {
        return fetch(request).then(response => {
            // 跨域的资源直接return
            if (!response || response.status !== 200 || response.type !== "basic") {
                return response;
            }
            util.putCache(request, response.clone());
            typeof callback === "function" && callback();
            return response;
        });
    },
    putCache: function (request, resource) {
        // 后台不要缓存，preview链接也不要缓存
        if (request.method === "GET" && request.url.indexOf("wp-admin") < 0 
              && request.url.indexOf("preview_id") < 0) {
            caches.open(CACHE_NAME).then(cache => {
                cache.put(request, resource);
            });
        }
    }
};
```

需要注意的是跨域的资源不能缓存，response.status会返回0，如果跨域的资源支持CORS，那么可以把request的mod改成cors。如果请求失败了，如404或者是超时之类的，那么也直接返回response让主页面处理，否则的话说明加载成功，把这个response克隆一个放到cache里面，然后再返回response给主页面线程。注意能放缓存里的资源一般只能是GET，通过POST获取的是不能缓存的，所以要做个判断（当然你也可以手动把request对象的method改成get），还有把一些个人不希望缓存的资源也做个判断。

这样一旦用户打开过一次也没，Service Worker就安装好了，他刷新页面或者打开第二个页面的时候就能够把请求的资源一一做缓存，包括图片、CSS、JS等，只要缓存里有了不管用户在线或者离线都能够正常访问。这样我们自然会有一个问题，这个缓存空间到底有多大？上一篇我们提到Manifest也算是本地存储，PC端的C和ROM而是5Mb，其实这个说法在新版本的Chrome已经补准确了，在Chrome61版本可以看到本地存储的空间和使用情况：

![Chrome本地存储可使用的空间](https://user-gold-cdn.xitu.io/2017/10/4/50e8f96bbcb3bc644a083a409ce0ce2d?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

其中Cache Storage是指Service Worker和Manifest占用的空间大小和，上图可以看到总的空间大小是20GB，几乎是unlimited，所以基本上不用担心缓存会不够用。

### （4）cache html

上面第（3）步把图片、js、css缓存起来了，但是如果把页面HTML也缓存了，例如把首页缓存了，就会有一个尴尬的问题——Service Worker是在页面注册的，但是现在获取页面的时候是从缓存提取的，每次都是一样的，所以就导致无法更新Service Worker，如变成sw-5.js，但是PWA又要求我们能缓存页面html。那怎么办呢？谷歌的开发者文档只是提到会存在这个问题，但并没有说明怎么解决这个问题。这个问题的解决就要求我们要有一个机制能知道html更新了，从而把缓存里面的html给替换掉。

Manifest更新缓存的机制是去看Manifest的文本内容有没有发生变化，如果发生变化了，则会去更新缓存，Service Worker也是根据sw.js的文本内容有没有发生变化，我们可以借鉴这个思想，如果请求的是html并从缓存里取出来后，再发个请求获取一个文件看html更新时间是否发生变化，如果发生变化了则说明发生更改了，进而把缓存给删除。所以可以在服务端通过控制这个文件从而去更新客户端的缓存。如下代码：

```javascript
this.addEventListener("fetch", function(event) {

    event.respondWith(
        caches.match(event.request).then(response => {
            // cache hit
            if (response) {
                //如果取的是html，则看发个请求看html是否更新了
                if (response.headers.get("Content-Type").indexOf("text/html") >= 0) {
                    console.log("update html");
                    let url = new URL(event.request.url);
                    util.updateHtmlPage(url, event.request.clone(), event.clientId);
                }
                return response;
            }

            return util.fetchPut(event.request.clone());
        })
    );
});
```

通过响应头header的content-type是否为text/html，如果是的话就去发个请求获取一个文件，根据这个文件的内容决定是否需要删除缓存，这个更新的函数util.updateHtmlPage是这么实现的：

```javascript
let pageUpdateTime = {

};
let util = {
    updateHtmlPage: function (url, htmlRequest) {
        let pageName = util.getPageName(url);
        let jsonRequest = new Request("/html/service-worker/cache-json/" + pageName + ".sw.json");
        fetch(jsonRequest).then(response => {
            response.json().then(content => {
                if (pageUpdateTime[pageName] !== content.updateTime) {
                    console.log("update page html");
                    // 如果有更新则重新获取html
                    util.fetchPut(htmlRequest);
                    pageUpdateTime[pageName] = content.updateTime;
                }
            });
        });
    },
    delCache: function (url) {
        caches.open(CACHE_NAME).then(cache => {
            console.log("delete cache " + url);
            cache.delete(url, {ignoreVary: true});
        });
    }
};
```

代码先去获取一个json文件，一个页面会对应一个json文件，这个json的内容是这样的：

```json
{"updateTime":"10/2/2017, 3:23:57 PM","resources": {img: [], css: []}}
```

里面主要有一个updateTime的字段，如果本地内存没有这个页面的updateTime的数据或者是和最新updateTime不一样，则重新去获取html，然后当道缓存里。接着需要通知页面线程数据发生变化了，你刷新下页面吧。这样就不用等用户刷新页面才能生效了。所以当刷新完页面后用postMessage通知页面：

```javascript
let util = {
    postMessage: async function (msg) {
        const allClients = await clients.matchAll();
        allClients.forEach(client => client.postMessage(msg));
    }
};
util.fetchPut(htmlRequest, false, function() {
    util.postMessage({type: 1, desc: "html found updated", url: url.href});
});
```

并规定type：1就表示这是一个更新html的消息，然后再页面监听message事件：

```javascript
if("serviceWorker" in navigator) {
    navigator.serviceWorker.addEventListener("message", function(event) {
        let msg = event.data;
        if (msg.type === 1 && window.location.href === msg.url) {
            console.log("recv from service worker", event.data);
            window.location.reload();
        }   
    }); 
}
```

然后当我们需要更新html的时候就更新json文件，这样用户就能看到最新的页面了。或者是当用户重新启动浏览器的时候会导致Service Worker的运行内存都被清空了，即存储页面更新时间的变量被清空了，这个时候也会重新请求页面。

需要注意的是，要把这个json文件的http cache时间设置成0，这样浏览器就不会缓存了，如下nginx的配置：

```bash
location ~* .sw.json$ {
    expires 0;
}
```

因为这个文件是需要实时获取的，不能被缓存，firefox默认会缓存，Chrome不会，加上http缓存时间为0，firefox也不会缓存了。

还有一种更新是用户更新的，例如用户发表了评论，需要在页面通知service worker把html缓存删除再重新获取，这是一个反过来的消息通知：

```javascript
if ("serviceWorker" in navigator) {
    document.querySelector(".comment-form").addEventListener("submit", function() {
            navigator.serviceWorker.controller.postMessage({
                type: 1, 
                desc: "remove html cache", 
                url: window.location.href}
            );
        }
    });
}
```

Service Worker也监听message事件：

```javascript
const messageProcess = {
    // 删除html index
    1: function (url) {
        util.delCache(url);
    }
};

let util = {
    delCache: function (url) {
        caches.open(CACHE_NAME).then(cache => {
            console.log("delete cache " + url);
            cache.delete(url, {ignoreVary: true});
        });
    }
};

this.addEventListener("message", function(event) {
    let msg = event.data;
    console.log(msg);
    if (typeof messageProcess[msg.type] === "function") {
        messageProcess[msg.type](msg.url);
    }
});
```

根据不同的消息类型调用不同的回调函数，如果是1的话就是删除cache。用户发表完评论后会触发刷新页面，刷新的时候缓存已经被删除了就会重新去请求了。

这样就解决了实时更新的问题。

## 4、Http/Manifest/Service Worker三种cache的关系

要缓存可以使用三种手段，使用Http Cache设置缓存时间，也可以用Manifest的Application Cache，还可以用Service Worker缓存，如果三者都用上了会怎么样呢？

会以Service Worker为优先，因为Service Worker把请求拦截了，它最先做处理，如果它缓存库里有的话直接返回，没有的话正常请求，就相当于没有Service Worker了，这个时候就到了Manifest层，Manifest缓存里如果有的话就取这个缓存，如果没有的话就相当于没有Manifest了，于是就会从Http缓存里取了，如果Http缓存里也没有就会发请求去获取，服务端根据Http的etag或者Modified Time可能会返回304 Not Modified，否则正常返回200和数据内容。这就是整一个获取的过程。

所以如果即用了Manifest又用Service Worker的话应该会导致同一个资源存了两次。但是可以让支持Service Worker的浏览器使用Service Worker，而不支持的使用Manifest。

## 5、使用Web App Manifest添加桌面入口

注意这里说的是另外一个Manifest，这个Manifest是一个json文件，用来放网站icon名称等信息以便在桌面添加一个图标，以及制造一种打开这个网页就像打开App一样的效果。上面一直说的Manifest是被废除的Application Cache的Manifest。

这个Manifest.json文件可以这么写：

```json
{
  "short_name": "人人FED",
  "name": "人人网FED，专注于前端技术",
  "icons": [
    {
      "src": "/html/app-manifest/logo_48.png",
      "type": "image/png",
      "sizes": "48x48"
    },
    {
      "src": "/html/app-manifest/logo_96.png",
      "type": "image/png",
      "sizes": "96x96"
    },
    {
      "src": "/html/app-manifest/logo_192.png",
      "type": "image/png",
      "sizes": "192x192"
    },
    {
      "src": "/html/app-manifest/logo_512.png",
      "type": "image/png",
      "sizes": "512x512"
    }
  ],
  "start_url": "/?launcher=true",
  "display": "standalone",
  "background_color": "#287fc5",
  "theme_color": "#fff"
}
```

icon需要准备多种规格，最大需要512px * 512px的，这样Chrome会自动去选取合适的图片。如果把display改成standalone，从生成的图标打开就会像打开一个App一样，没有浏览器地址栏那些东西了。start_url指定打开之后的入口链接。

然后添加一个link标签指向这个Manifest文件：

```html
<link rel="manifest" href="/html/app-manifest/manifest.json">
```

这样结合Service Worker缓存：

![Service Worker缓存](https://user-gold-cdn.xitu.io/2017/10/4/363df72b0b09917fe918072537ac1e5e?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

把start_url指向的页面用Service Worker缓存起来，这样当用户用Chrome浏览器打开这个网页的时候，Chrome就会在底部弹一个提示，询问用户是否把这个网页添加到桌面，如果点“添加”就会生成一个桌面图标，从这个图标点进去就像打开一个App一样。感受一下：

![桌面App](https://user-gold-cdn.xitu.io/2017/10/4/8972092dabe9ee7b3451f354f89d4ae9?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

比较尴尬的是Manifest目前只有Chrome支持，并且只能在安卓系统上使用，iOS的浏览器无法添加一个桌面图标，因为iOS没有开放这种API，但是自家的Safari却又是可以的。

综上，本文介绍了怎么用Service Worker结合Manifest做一个PWA离线Web APP，主要是用Service Worker控制缓存，由于是写JS，比较灵活，还可以与页面进行通信，另外通过请求页面的更新时间来判断是否需要更新html缓存。Service Worker的兼容性不是特别好，但是前景比较光明，浏览器都在准备支持。现阶段可以结合offline cache的Manifest做离线应用。

