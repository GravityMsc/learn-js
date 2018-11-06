我在上一篇《[使用Service Worker做一个PWA离线网页应用](https://fed.renren.com/2017/10/04/service-worker/)》已经介绍了怎么做离线缓存，这一篇将介绍怎么用Service Worker发送Push（Notification），或者叫web push。Web push在国外的网站很流行，但在国内几乎没见到，主要还是因为谷歌在境内无法访问，因为web push走的是谷歌FCM通道，需要能接收到谷歌服务器的消息。但正常网络环境下是无法访问谷歌的，使得在国内搞它的意义不是很大，但是毕竟它是一个标准和趋势，作为一个技术人员来研究一下还是挺有用的。

## 1、发送Push过程

给用户浏览器或者客户端发送一个Push，这个过程是这样的：

![给用户发送Push的过程图示](https://user-gold-cdn.xitu.io/2017/10/8/44e962681fa98fdf75c295319b4ff3f6?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

在浏览器端，注册一个Service Worker之后会返回一个注册的对象，调用这个对象的pushManager.subscribe的方法让浏览器弹一个框，询问用户是否允许接受消息通知：

![弹框询问用户是否接受消息通知](https://user-gold-cdn.xitu.io/2017/10/8/1d64ebd1e2d9633654ba72130b680add?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

如果点击允许的haunted，浏览器就会向FCM请求生成一个subscription（订阅）的标志信息，然后把这个subscription发给服务端存起来，用来发Push给当前用户。服务端使用这个subscription的信息调用web push提供的API向FCM发送消息，FCM再下发给对应的浏览器。然后浏览器会触发Service Worker的push事件，让Service Worker调用showNotification显示这个push的内容。操作系统就会显示这个Push：

![push推送的效果](https://user-gold-cdn.xitu.io/2017/10/8/c7c0847ea11e7a0b111e4b0b2f581949?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

点击这个框，就能跳转到指定的url查看内容。这就是整一个过程，具体来说：

### （1）浏览器发起询问，生成subscription

在注册完service worker后，调用subscription询问用户是否允许接收通知，如下代码所示：

```javascript
navigator.serviceWorker.register("sw-4.js").then(function(reg){
    console.log("Yes, it did register service worker.");
    if (window.PushManager) {
        reg.pushManager.getSubscription().then(subscription => {
            // 如果用户没有订阅
            if (!subscription) {
                subscribeUser(reg);
            } else {
                console.log("You have subscribed our notification");
            }       
        });     
    }
}).catch(function(err) {
    console.log("No it didn't. This happened: ", err)
});
```

上面代码在发起订阅前先看一下之前已经有没有订阅过了，如果没有的话再发起订阅。发起订阅的subscribeUser实现如下代码所示：

```javascript
function subscribeUser(swRegistration) {
    const applicationServerPublicKey = "BBlY_5OeDkp2zl_Hx9jFxymKyK4kQKZdzoCoe0L5RqpiV2eK0t4zx-d3JPHlISZ0P1nQdSZsxuA5SRlDB0MZWLw";
    const applicationServerKey = urlB64ToUint8Array(applicationServerPublicKey);
    swRegistration.pushManager.subscribe({
        userVisibleOnly: true,
        applicationServerKey: applicationServerKey
    })
    // 用户同意
    .then(function(subscription) {
        console.log('User is subscribed:', JSON.stringify(subscription));
        jQuery.post("/add-subscription.php", {subscription: JSON.stringify(subscription)}, function(result) {
            console.log(result);
        });
    })
    // 用户不同意或者生成失败
    .catch(function(err) {
        console.log('Failed to subscribe the user: ', err);
    });
}
```

subscribe传两个参数，一个是userVisibleOnly，这个表示消息是否必须要可见，如果设置为不可见，Chrome将会报错：

> Chrome currently only supports the Push API for subscriptions that will result in user-visible messages. You can indicate this by calling pushManager.subscribe({userVisibleOnly: true}) instead. See https://goo.gl/yqv4Q4 for more details.

但其实这个并不影响，我们设置成true，但是收到消息后可以不用弹框，可以调用postMessage去通知页面做相应的操作。

第二个参数applicationServerKey是服务端的公钥，这个可以用[web push的Node包](https://github.com/web-push-libs/web-push)生成，先安装一个：

> npm install web-push --save

让后用以下代码生成：

```javascript
const webpush = require('web-push');
//VAPID keys should only be generated only once.
const vapidKeys = webpush.generateVAPIDKeys();
console.log(vapidKeys.publicKey, vapidKeys.privateKey);
```

每运行一次就会生成一对新的秘钥对，如：

```bash
publicKey:  BMgkd1qfOfI6vFBbxIFMgdxDGC6-j8NYTwF_MXOIZ-St9lPhhMdPuUyFfwg1DLY59WP0FEaX84ZJRwgztdpfBHs
privateKey: LUeSF8DCv-NBxIfaeWeKTux858H45_V75vT0zZQLEbY
```

公密钥只要能配套就好，公钥在浏览器端使用，用来生成subscription，密钥在服务端使用，用来发Push。

如果用户同意浏览器就会向FCM服务请求生成subscription，然后执行Promise链里的then，返回该subscription，在这个then里面把这个subscription发给服务端存起来。反之，如果用户不同意，或者用户无法连到FCM的服务器将会抛出异常：

> DOMException: Registration failed - push service error

生成的subscription大概长这样：

```json
{"endpoint":"https://fcm.googleapis.com/fcm/send/ci3-kIulf9A:APA91bEaQfDU8zuLSKpjzLfQ8121pNf3Rq7pjomSu4Vg-nMwLGfJSvkOUsJNCyYCOTZgmHDTu9I1xvI-dMVLZm1EgmEH0vDA7QFLjPKShG86W2zwX0IbtBPHEDLO0WgQ8OIhZ6yTnu-S","expirationTime":null,"keys":{"p256dh":"BAdAo6ldzRT5oCN8stqYRemoihPGOEJjrUDL6y8zhdA_swao_q-HlY_69mzIVobWX2MH02TzmtRWj_VeWUFMnXQ=","auth":"SS1PBnGwfMXjpJEfnoUIeQ=="}}
```

说了这么久的FCM，FCM到底是什么呢？

### （2）什么是FCM

[FCM官方](https://firebase.google.com/docs/cloud-messaging/?hl=zh-cn)是这么介绍的：

> Firebase 云信息传递 (FCM) 是一种跨平台消息传递解决方案，可供您免费、可靠地传递消息。
>
> 使用 FCM，您可以通知客户端应用存在可同步的新电子邮件或其他数据。您可以发送通知消息以再次吸引用户并促进用户留存。在即时消息传递等使用情形中，一条消息可将最大 4KB 的有效负载传送至客户端应用。

FCM是一种可靠的消息传递平台，它最大的优点是同一套Push机制可以在iOS/Android/Web三端使用：

![FCM使用图示](https://user-gold-cdn.xitu.io/2017/10/8/441f5c0b63c9a67f5a77f14e3d4afe95?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

这个意义是很大的，因为Android的推送一直都比较乱，国内有些APP使用小米的Push服务，有些使用百度的，还有些使用腾讯的信鸽等等，这些Push都需要在后台运行线程，并且不能休眠，这就导致了手机在休眠状态时仍然有很多线程在运行着，使得手机耗电速度很快。最后还直接导致今年工信部出台要成立安卓统一推送联盟。而苹果有一套统一的推送机制，大家把Push发给苹果的服务器，然后再由苹果下发给相应的苹果设备。Safari现在不支持Service Worker，但是可以用Apple Push，缺点是这种推送苹果说不能用来发送重要的数据，并且目前只能弹框显示，没办法在后台处理消息而不弹框。

### （3）发送推送

发送推送可以用FCM提供的web push库，它支持多种语言，包括Node.js/PHP等版本。用Node.js可以这样发送Push：

```javascript
const webpush = require('web-push');
// 从数据库取出用户的subsciption
const pushSubscription = {"endpoint":"https://fcm.googleapis.com/fcm/send/ci3-kIulf9A:APA91bEaQfDU8zuLSKpjzLfQ8121pNf3Rq7pjomSu4Vg-nMwLGfJSvkOUsJNCyYCOTZgmHDTu9I1xvI-dMVLZm1EgmEH0vDA7QFLjPKShG86W2zwX0IbtBPHEDLO0WgQ8OIhZ6yTnu-S","expirationTime":null,"keys":{"p256dh":"BAdAo6ldzRT5oCN8stqYRemoihPGOEJjrUDL6y8zhdA_swao_q-HlY_69mzIVobWX2MH02TzmtRWj_VeWUFMnXQ=","auth":"SS1PBnGwfMXjpJEfnoUIeQ=="}};

// push的数据
const payload = {
    title: '一篇新的文章',
    body: '点开看看吧',
    icon: '/html/app-manifest/logo_512.png',
    data: {url: "https://fed.renren.com"}
  //badge: '/html/app-manifest/logo_512.png'
};

webpush.sendNotification(pushSubscription, JSON.stringify(payload));
```

经实验，在大多数情况下这个延迟基本在1s以内，这边刚按下回车运行完，那边浏览器就收到了，但是有时候会发送失败（国内网络问题？）。如果这个代码要在服务端运行的话，那么你应该需要一台香港服务器。像笔者把发Push的数据和服务放在香港的服务器，需要发Push的时候由华北的服务器做个中转向这台服务器发请求。只要用户能连上FCM那就可以愉快地发Push了，如果用户连不上那就没办法。

### （4）接收推送消息

用运行在后台的Service Worker接收，监听Push事件：

```javascript
this.addEventListener('push', function(event) {
    console.log('[Service Worker] Push Received.');
    console.log(`[Service Worker] Push had this data: "${event.data.text()}"`);

    let notificationData = event.data.json();
    const title = notificationData.title;
    // 可以发个消息通知页面
    //util.postMessage(notificationData); 
    // 弹消息框
    event.waitUntil(self.registration.showNotification(title, notificationData));
});
```

主要是调用showNotification进行弹框，或者是使用postMessage通知页面相应地做些处理。经实验，如果用户关闭了浏览器，在关闭期间如果有Push的话等到用户重新打开浏览器会再弹出来。然后用户可以点击弹出来的框打开一个指定的页面，这个需要监听notificationclick事件：

```javascript
this.addEventListener('notificationclick', function(event) {
    console.log('[Service Worker] Notification click Received.');

    let notification = event.notification;
    notification.close();
    event.waitUntil(
        clients.openWindow(notification.data.url)
    );
});
```

调用clients.openWindow打开一个新的页面。

这样就基本完成了一个push推送的搭建。

Service Worker让我们在Web端也能有像原生APP一样的Push通知，使得Web端越来越像原生APP端，随着HTML5的其他新功能如[WebAssembly](https://fed.renren.com/2017/05/21/webassembly/)提高运行速度，[WebWorker](https://fed.renren.com/2017/05/21/js-threads/)多线程支持，[数据库](https://fed.renren.com/2017/06/11/sql/)支持大量数据的管理和支持，[Websocket](https://fed.renren.com/2017/05/20/websocket-and-tcp-ip/)进行实时通信，WebRTC进行P2P多媒体传输，还有WebGL、新进的WebVR等，使得在浏览器端能够做的事情越来越多，体验越来越丰富，而且这种Web APP还是跨平台的。Web技术日新月异的发展，让我们相信Web有搞头。