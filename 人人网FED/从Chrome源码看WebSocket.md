WebSocket是为了解决双向通信的问题，因为一方面HTTP的设计是单向的，只能是一边发另一边收。而另一方面，HTTP等都是建立在TCP连接上的，HTTP请求完就会把TCP给关了，而TCP连接本身就是一个长连接，只要连接双方不关闭连接就会一直是连接态，所以有必要再搞一个WebSocket吗？

我们可以考虑一下，如果不搞WebSocket怎么实现长连接：

（1）HTTP有一个keep-alive的字段，这个字段的作用是复用TCP连接，可以让一个TCP连接用来发多个http请求，重复利用，避免新的TCP连接又得三次握手。这个keep-alive的时间，服务器如[Apache](https://zh.wikipedia.org/wiki/HTTP%E6%8C%81%E4%B9%85%E8%BF%9E%E6%8E%A5)的时间是5s，而[nginx](http://nginx.org/en/docs/http/ngx_http_core_module.html?&_ga=2.19983992.1933946575.1527338224-727450173.1519009174#keepalive_timeout)默认是75s，超过这个时间服务器就会主动把TCP连接关闭了，因为不关闭的话会有大量的TCP连接占用系统资源。所以这个keep-alive也不是为了长连接设计的，只是为了提高http请求的效率，而http请求上面已经提到它是面向单向的，要么是服务端下发数据，要么是客户端上传数据。

（2）使用HTTP轮询，这也是一种很常用的方法，没有WebSocket之前，基本上网页的聊天功能都是这么实现的，每隔几秒就向服务器发个请求拉取新消息。这个方法的问题就在于它也是需要不断地建立TCP连接，同时HTTP头部是很大的，效率低下。

（3）直接和服务器建立一个TCP连接，保持这个连接不中断，这个至少在浏览器端是做不到的，因为没有相关的API。所以就有了WebSocket直接和服务器建立一个TCP连接。

TCP连接是使用套接字建立的，如果你写过Linux服务的话，就知道怎么用系统底层的API（C语言）建立一个TCP连接，它是使用的套接字socket，这个过程大概如下，服务端使用socket创建一个TCP监听：

```c++
// 先创建一个套接字，返回一个句柄，类似于setTimeout返回的tId
// AF_INET是指使用IPv4地址，SOCK_STREAM表示建立TCP连接（相对于UDP）
int sockfd = socket(AF_INET, SOCK_STREAM, 0);
// 把这个套接字句柄绑定到一个地址，比如localhost:9000
bind(sockfd, servaddr, sizeof(servaddr));
// 开始使用这个套接字监听，最大pending的连接数为100
listen(sockfd, 100);
```

客户端也使用的套接字进行连接：

```c++
// 客户端也是创建一个套接字
int sockfd = socket(AF_INET, SOCK_STREAM, 0);
// 用这个套接字连接到一个serveraddr
connect(sockfd, servaddr, sizeof(servaddr));
// 用这个套接字发送数据
send(sockfd, sendline, strlen(sendline), 0);
// 关闭连接
close(sockfd);
```

也就是说TCP和UDP连接都是使用套接字创建的，所以WebSocket的名字就是这个来的，本质上它就是一个套接字，并变成了一个标准，浏览器开放了API，让网页开发人员也能直接创建套接字和服务端进行通信，并且这个套接字什么时候要关闭了由你们去决定，而不像http一样请求完了浏览器或者服务器就自动把TCP的套接字连接关了。

所以说WebSocket并不是一个什么神奇的东西，它就是一个套接字。同时，WebSocket得借助于现有的网络基础，如果它再从头搞一套建立连接的标准代价就会很大。在它之前能够和服务连接的就只有http请求，所以它得借助于http请求来建立一个原生的socket连接，因此才有了协议转换的那些东西。

浏览器建立一个WebSocket连接非常简单，只需要几行代码：

```javascript
// 创建一个套接字
const socket = new WebSocket('ws://192.168.123.20:9090');
// 连接成功
socket.onopen = function(event){
    console.log('opened');
    // 发送数据
    socket.send('hello, this is from client');
};
```

因为浏览器已经按照文档实现好了，而要创建一个WebSocket的服务端该怎么写呢？这里我们先抛开Chrome源码，先研究服务端的实现，然后再反过来看浏览器客户端的实现。准备用Node.js实现一个WebSocket的服务端，来研究整一个连接建立和接收发送数据的过程是怎么样的。

WebSocket已经在[RFC 6455](https://datatracker.ietf.org/doc/rfc6455/?include_text=1)里面进行了标准化，我们只要按照文档的规定进行实现就能和浏览器进行对接，这个文档的说明比较有趣，特别是第一部分，有兴趣的读者可以看看，并且我们发现WebSocket的实现非常简单，读者如果有时间的话可以先尝试自己实现一个，然后再回过头来，对比本文的实现。

### 1. 连接建立

使用Node.js创建一个hello，world的http服务，如下代码index.js所示：

```javascript
let http = require('http');
const hostname = '192.168.123.20';	// 或者是localhost
const port = '9090';

// 创建一个http服务
let server = http.createServer((req, res) => {
    // 收到请求
    console.log('recv request');
    console.log(req.headers);
    // 进行响应，发送数据
    // res.write('hello, world');
    // res.end();
});

// 开始监听
server.listen(port, hostname, () => {
    // 启动成功
    console.log(`Server running at ${hostname}:${port}`);
});
```

注意到这里没有任何的出错和异常处理，被省略了，在实际的代码里面为了提高程序的稳健性需要有异常处理，特别是这种server类的服务，不能让一个请求就把整个server搞挂了。相关出错处理可以参考Node.js的文档。

保存文件，执行node index.js启动这个服务。

然后写一个index.html，请求上面写的服务。



```html
<!DOCType html>
<html>
<head>
    <meta charset="utf-8">
</head>
<body>
<script>
!function() {
    const socket = new WebSocket('ws://192.168.123.20:9090');
    socket.onopen = function (event) {
        console.log('opened');
        socket.send('hello, this is from client');
    };
}();
</script>
</body>
</html>
```

但是我们发现，Node.js代码里的请求响应回调函数并不会执行，查了文档发现是因为Node.js有另外一个upgrade的事件：

```javascript
// 协议升级
server.on('upgrade', (request, socket, head) => {
    console.log(request.headers);
});
```

因为WebSocket需要先协议升级，在upgrade里面就能收到升级的请求。把收到的请求头打印出来，如下所示：

> { host: '192.168.123.20:9090',
> connection: '**Upgrade**',
> pragma: 'no-cache',
> 'cache-control': 'no-cache',
> upgrade: 'websocket',
> origin: 'http://127.0.0.1:8080',
> 'sec-websocket-version': '13',
> 'user-agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.181 Safari/537.36',
> 'accept-encoding': 'gzip, deflate',
> 'accept-language': 'en,zh-CN;q=0.9,zh;q=0.8,zh-TW;q=0.7',
> '**sec-websocket-key**': 'KR6cP3rhKGrnmIY2iu04Uw==',
> 'sec-websocket-extensions': 'permessage-deflate; client_max_window_bits' }

这是我们建立连接收到的第一个请求，里面有两个关键的字段，一个是connection: 'Upgrade'表示它是一个升级协议的请求，另外一个是sec-websocket-key，这是一个用来确认身份的随机的base64字符串，下面将会用到。

