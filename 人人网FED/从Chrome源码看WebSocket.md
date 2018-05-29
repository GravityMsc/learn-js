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

我们需要对这个请求进行响应，按照文档的说明，需要包含以下字段：

```javascript
server.on('upgrade', (request, socket, head) => {
    let base64Value = '';
    // 第一行是响应行（Response line），返回状态码101
    socket.write('HTTP/1.1 101 Web Socket Protocol Handshake\r\n' + 
                // http响应头部字段用\r\n隔开
                'Upgrade: WebSocket\r\n' +
                'Connection: Upgrade\r\n' +
                // 这是一个给浏览器确认身份的字符串
                `Sec-WebSocket-Accept: ${base64Value}\r\n` + 
                '\r\n');
});
```

响应报文需要按照http规定的格式，第一行是响应行，包含了http的版本号，状态码101，状态码的解释。每个头部字段用\r\n隔开，这里面最关键的一个字段是Sec-WebSocket-Accept，它需要计算一下返回给浏览器。怎么计算呢？文档是这么规定的：

> GUID(Globally_Unique_Identifier) = '258EAFA5-E914-47DA-95CA-C5AB0DC85B11'
>
> Sec-WebSocket-Accept = base64(sha1(Sec-Websocket-key + GUID))

使用浏览器给我的sec-websocket-key值，拼上一个固定的字符串，这个字符串叫全局唯一标识符，然后取它的sha1值，再进行base64编码，返回给浏览器。如果浏览器发现这个值不对的话，就会抛出异常，拒绝下一步的连接操作：

![WebSocket连接握手检查](https://user-gold-cdn.xitu.io/2018/5/27/1639fd34f81ea5bc?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

因为它发现你是一个假的WebSocket服务，起码不是按照文档实现的，所以不是同一个世界，没有共同语言，下面的交流就没有必要了。

为了计算这个值需要引入一个sha1的库，base64转换可以使用Node.js的Buffer转换， 如下代码所示：

```javascript
let sha1 = require('sha1');
// 协议升级
server.on('upgrade', (request, socket, head) => {
    // 取出浏览器发送的key值
    let secKey = request.headers['sec-websocket-key'];
    // RFC 6455规定的全局标识符（GUID）
    const UNIQUE_IDENTIFIER = '258EAFA5-E914-47DA-95CA-C5AB0DC85B11';
    // 计算sha1和base64的值
    let shaValue = sha1(secKey + UNIQUE_IDENTIFIER),
        base64Value = Buffer.from(shaValue, 'hex').toString('base64');
    socket.write('HTTP/1.1 101 Web Socket Protocol Handshake\r\n' +
                'Upgrade: WebSocket\r\n' +
                'Connection: Upgrade\r\n' +
                `Sec-WebSocket-Accept: ${base64Value}\r\n` +
                '\r\n');
});
```

使用上面浏览器发送的key值计算得到的accept值为：

> RWMSYL3Zmo91ZR+r39JVM2+PxXc= 

把这个值发给浏览器，Chrome就不会报刚刚那个检验出错了，这样WebSocket连接就建立了，没错就是这么简单。Chrome开发者工具NetWork面板里的WebSocket连接将会从pending状态变成101状态，如果连接关闭了就会变成200状态。

上面浏览器的代码在建立连接完成之后还seed了一个数据过来：

```javascript
socket.send('hello, this is from client');
```

怎么读取这个数据呢？

### 2. 接收数据

数据的传送，文档规定了WebSocket数据帧格式，长这个样子

![WebSocket数据帧的格式](https://user-gold-cdn.xitu.io/2018/5/27/1639fd34f7b6176e?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

不要被这个吓到，一个个拆解来看的话，还是挺简单的可以分成两个部分，帧头字段和有效内容或者叫有效负载（Payload Data），帧头字段主要的作用是为了解释这个帧的，如第1位（bit）FIN如果置为1就表示它是一个结束帧，如果数据比较长就会被拆成几个帧发送，FIN为1表示它是当前数据流的最后一个帧。第4到第7位的opcode是用来做一些指令控制的，如果只为1就是表示Payload Data是文本格式的，2则表示二进制内容，8表示连接关闭。第9位到第15位共7位Payload Len表示有效负载的字节数，7位二进制最大表示127，如果有效负载字节数大于127的话就需要用到Extended payload length部分。

第8位的Mask如果设置为1就表示这个帧的有效负载内容被掩码处理过了，客户端向服务端发送的帧需要进行掩码，而服务端向客户端发送的数据帧不需要掩码。为什么要使用掩码，这个掩码计算又是怎么进行的呢？掩码计算很简单，就是把要发送的数据和另一个数字异或一下再放到Payload Data，这个数字就是上面数据帧里额Masking-key，它是一个32位的数字。接收方把Payload Data再和这个数异或一下就能得到原始的数据，因为和同一个数异或两次等于原本的数，即：

a^b^b = a

并且每个帧里面的Masking-key要求都是随机的，不可被（代理）服务所预测的，为什么要这样呢？文档里面是这么说的：

> The unpredictability of the masking key is essential to prevent authors of malicious applications from selecting the bytes that appear on the wire

这个解释有点含糊，[Stackoverflow](https://stackoverflow.com/questions/14174184/what-is-the-mask-in-a-websocket-frame)上有人说是为了避免代理缓存中毒攻击，具体可参考[Http Cache Poinsing](https://stackoverflow.com/questions/14174184/what-is-the-mask-in-a-websocket-frame)。

所以我们需要从这个帧里面取出掩码的key值，还原原始的payload数据。

数据的发送和传输都要靠socket对象，因为它不是走的http请求，所以在http的响应函数里面是收不到数据的，在upgrade事件里面可以拿到这个socket，监听这个socket对象的data事件，就可以得到接收的数据。

```javascript
socket.on('data', buffer => {
    console.log('buffer len = ', buffer.length);
    console.log(buffer);
});
```

返回的数据类型是Node.js里的Buffer对象，把这个buffer打印出来：

> buffer len = 32 <Buffer 81 9a 4c 3f 64 75 24 5a 08 19 23 13 44 01 24 56 17 55 25 4c 44 13 3e 50 09 55 2f 53 0d 10 22 4b> 

这个buffer就是WebSocket客户端给我们发送的数据帧了，总共有32个字节，上面的打印是用的16进制表示，可以改二进制0101表示，和上面的数据帧格式图一一对照，就能够解释这个数据帧是什么意思，有什么内容。把它打印成原始二进制表示：

> 1000000110011010010011000011111101100100011101010010010001011010000010000001100100100011000100110100010000000001001001000101011000010111010101010010010101001100010001000001001100111110010100000000100101010101001011110101001100001101000100000010001001001011

按照报文格式，如下图所示：

![WebSocket报文数据帧格式](https://user-gold-cdn.xitu.io/2018/5/27/1639fd34f7db5f37?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

通过opcode可以知道它是一个文本数据的帧，payload len得到文本长度为26个字节，这个刚好等于上面发送的内容长度：

![WebSocket发送的内容长度](https://user-gold-cdn.xitu.io/2018/5/27/1639fd34f7e77a71?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

同时掩码Mask是打开的，掩码key值存放反问是[16, 16 + 32]，因为这里不需要使用扩展字段，所以Masking-key就直接跟在Payload len后面了，再往后就是Payload Data，范围是[48, 48 + 26 * 8]。

这就是一个完整的数据帧了，还需要把payload data用掩码异或一下，还原原始数据。在Node.js里面进行处理。Node.js里面的Buffer类只能操作字节级别，如读取第n个字节的内容，没办法直接操作位，如读取第n位的数据。所以额外引入一个库，网上找了一个BitBuffer，但是它的实现好像有问题，所以自己实现了一个。

如下代码所示，实现一个能够读取任意位的BitBuffer：

```javascript
class BitBuffer {
    // 构造函数传一个Buffer对象
    constructor (buffer){
        this.buffer = buffer;
    }
    
    // 获取第offset和位的内容
    _getBit (offset){
        let byteIndex = offset / 8 >> 0;
        let byteOffset = offset % 8;
        // readUInt8可以读取第n个字节的数据
        // 取出这个数的第m位即可
        let num = this.buffer.readUInt8(byteIndex) & (1 << (7 - byteOffset))l
        return num >> (7 - byteOffset);
    }
}
```

原理很简单，先调用Node.js的Buffer的readUInt8读取第n个字节的数据，然后计算一下索要读取的位数在这个字节的第几位，通过与运算，把这个位取出来，更多位运算可以参考：[巧用JS位运算](https://fed.renren.com/2018/03/06/js-bit-algorithm/)。

用这个代码取出第8位的Mask Flag是否有设置，如下代码：

```javascript
socket.on('data', buffer => {
	let bitBuffer = new BitBuffer(buffer);
    let maskFlag = bitBuffer._getBit(8);
    console.log('maskFlag = ' + maskFlag);
});
```

代用maskFlag = 1。那么怎么取出连续的n位呢，如opcode，是从第4位到第7位。这个也好办，就是把第4位到第7位分别取出来拼成一个数即可。

```javascript
getBit(offset, len = 1){
    let result = 0;
    for(let i = 0; i < len; i++){
        result += this._getBit(offset +i) << (len - i - 1);
    }
    return result;
}
```

这个代码的效率不是很高，但是容易理解。有个小坑就是JS的位移只支持32位整数的操作，1<<31会变成一个负数，具体不展开讨论。用这个函数去32位的研制就会有问题。

可以利用这个函数取出opcode和payload len：

```javascript
socket.on('data', buffer => {
    let bitBuffer = new BitBuffer(buffer);
    let maskFlag = bitBuffer.getBit(8);
    let opcode = bitBuffer.getBit(4, 4);
    let payloadLen = bitBuffer.getBit(9, 7);
    console.log('maskFlag = ' + maskFlag);
    console.log('opcode = ' + opcode);
    console.log('payloadLen = ' + payloadLen);
});
```

打印如下：

> maskFlag = 1 opcode = 1 payloadLen = 26 

取掩码值单独实现一下，这个掩码是拆成4个数使用的，一个字节表示一个数，借助上面的getBit函数，代码如下：

```javascript
getMaskingKey(offset){
    const BYTE_COUNT = 4;
    let masks = [];
    for(let i = 0; i < BYTE_COUNT; i++){
        masks.push(this.getBit(offset + i * 8, 8));
    }
    return masks;
}
```

这个例子的掩码值是从第16位开始，所以offset是16：

```javascript
let maskKeys = bitBuffer.getMasking(16);
console.log('maskKey = ' + maskKeys);
```

打印出来的maskKey为：

> maskKeys = 76, 63, 100, 117

怎么用这个Mask Key进行异或呢，文档里面是这么规定的：

> j = i MOD 4 
>
> transformed-octet-i = original-octet-i XOR masking-key-octet-j 

也就是把Payload Data里面的第n，n + 1, n + 2, n + 3的字节内容分别与maskKeys数组的第0,  1,  2，3进行异或即可。所以这个实现也比较简单，如下代码所示：

```javascript
getXorString(byteOffset, byteCount, maskingKeys){
    let text = '';
    for(let i = 0; i < byteCount; i++){
        let j = i % 4;
        // 通过异或得到原始的utf-8编码
        let transformedByte = this.buffer.readUInt8(byteOffset + i) ^ maskingKeys[i];
        // 把编码值转成对应的字符
        text += String.fromCharCode(transformedByte);
    }
    return text;
}
```

异或操作之后就可以得到编码值，再借助String.fromCharCode就能得到对应的文本，如根ASCII表，97就会被还原成字母‘a’。

这个例子的payload data的偏移是从第6个字节开始的，这里我们先直接写死：

```javascript
let payloadLen = bitBuffer.getBit(9, 7);
let maskKeys = bitBuffer.getMaskingKey(16);
let payloadText = bitBuffer.getXorString(48 / 8, payloadLen, maskKeys);
console.log('payloadText = ' + payloadText);
```

打印的文本内容为：

> payloadText = hello, this is from client 

到这里，就把接收的数据还原出来了。如果想要发送数据，就是把读取的过程逆一下，按照帧格式去拼一个符合规范的帧发送给对方，区别是服务端的帧数据时不需要Mask的，如果你Mask了，Chrome会报一个异常，说数据不需要Mask，拒绝解析接收到的数据。

我们再从Chrome源码看WebSocket客户端的实现，来补充一些细节。

Chrome的WebSocket代码是在src/net/websockets，例如Chrome在握手的时候是怎么生成一个随机的sec-websocket-key？如下代码所示：

```c++
std::string GenerateHandshakeChallenge(){
    std::string raw_challenge(websockets::kRawChanllengeLength, '\0');
     crypto::RandBytes(base::string_as_array(&raw_challenge),
                    raw_challenge.length());
  	std::string encoded_challenge;
  	base::Base64Encode(raw_challenge, &encoded_challenge);
  	return encoded_challenge;
}
```

它是用的一个crypto::RandBytes生成随机字节，而在检验sec-websocket-accept也是用的同样的计算方法：

```c++
std::string ComputeSecWebSocketAccept(const std::string& key) {
  std::string accept;
  std::string hash = base::SHA1HashString(key + websockets::kWebSocketGuid);
  base::Base64Encode(hash, &accept);
  return accept;
}
```

而在使用掩码计算的时候也是用的一样的方法：

```c++
inline void MaskWebSocketFramePayloadByBytes(
    const WebSocketMaskingKey& masking_key,
    size_t masking_key_offset,
    char* const begin,
    char* const end) {
  for (char* masked = begin; masked != end; ++masked) {
    *masked ^= masking_key.key[masking_key_offset++];
    if (masking_key_offset == WebSocketFrameHeader::kMaskingKeyLength)
      masking_key_offset = 0;
  }
}
```

其他的还有deflate压缩、cookie、扩展extensions等，本文不再展开讨论。

另外还有一个问题，使用一个WebSocket就需要操持一个TCP连接，如果有1000个用户同时在线，那么服务器就需要保持1000个TCP连接，而一个TCP连接通常需要占用一个独立的线程，而线程的开销是很大的，所以WebSocket对服务端的压力特别大？其实也不见得有那么大，因为Linux有一个epoll的服务模型，它是一个事件驱动机制的，能够让一个核心支持并发的很多个连接。

最后一个问题，由于连接是一直操持的，如果连接双方有一方异常退出了，没有发送一个关闭连接的包通知对方，那么对方就会傻傻第操持着这个没用的连接，所以WebSocket又引入了一个ping/pong的消息帧，帧头里的opcode为0x9就表示是一个ping帧，0x10表示pong的响应帧。所以可以让客户端不断地ping，如每隔30秒就ping一次，服务受到了ping就知道当前客户端还活着，给一个pong的响应，如果服务端太久没受到ping了如1分钟，那么久认为这个客户端已经走了直接关闭连接。而客户端如果没受到pong响应那么就认为当前连接已经断了，需要重连。浏览器JS的API没有开放ping/pong，需要自己实现一个消息类型。

本篇主要讨论了WebSocket存在的意义，给浏览器开放一个socket的API，并机芯标准化，除了浏览器，APP等也都可以按照这个标准实现，弥补了HTTP单向传输的缺点。还讨论了WebSocket报文帧的格式，以及怎么用Node.js读取这个报文帧，客户端会把它发送的内容进行掩码处理。服务端收到的也需要进行掩码还原。我们发现Chrome客户端的实现有很多地方是类似的。

怎么保证WebSocket传输的稳定性可能又是另外一个话题了，包括出错重连机制等等。

