在讨论这个话题之前，先提几个问题：

1. 为什么说https是安全的，安全在哪里？
2. https是使用了证书保证它的安全么？
3. 为什么证书需要购买？

我们先来看https要解决什么问题

## 一、https解决什么问题

https要解决的问题就是中间人攻击，什么是中间人攻击（Man In The Middle Attack）呢？如下图所示：

![中间人攻击示意图](http://fed.renren.com/wp-content/uploads/2017/02/5-1.jpg)

你和服务器的连接会经过一个中间人，你以为你和服务器在正常地传输数据，其实这些数据都先经过了一个中间人，这个中间人可以窥探你的数据或者篡改你的数据后再发给服务器，相反也可以把服务器的数据修改了之后再发给你。而这个中间人对你是透明的，你不知道你的数据已经被人窃取或者修改了。

## 二、中间人攻击的方式

常见的有以下两种：

### 1）域名污染

由于我们访问一个域名时需要先进行域名解析，即向DNS服务器请求某个域名的IP地址。例如taobao.com我这边解析的IP地址为：

![taobao的IP地址](http://fed.renren.com/wp-content/uploads/2017/02/818663-20161112195031733-862916446.png)

在经过DNS的中间链接点可能会抢答，返回给你一个错误的IP地址，这个IP地址就指向中间人的机器。

### 2）ARP欺骗

广域网的传输是用的IP地址，而在局域网里面是用的物理地址，例如路由器需要知道连接它的设备的物理地址它才可以把数据包发给你，它会通过一个ARP的广播，向所有设备查询某个IP地址的物理地址是多少，如下所示：

![ARP广播](http://fed.renren.com/wp-content/uploads/2017/02/818663-20161112145435561-1856047295.png)

路由器发了一个广播，询问192.168.1.100的物理地址是多少，由于没有人响应，所以它每隔1秒就重新发了个包。由于这个网络上的所有机器都会受到这个包，所以这个时候就可以欺骗路由器：

![ARP欺骗](http://fed.renren.com/wp-content/uploads/2017/02/818663-20161112161938311-456617134.png)

上面的192.168.1.102就向路由器发了一个响应的包，告诉路由器它的物理地址。

## 三、https是应对中间人攻击的唯一方式

在ssl的源码里面就有一段注释：

```c++
/* cert is OK. This is the client side of an SSL connection.
 * Now check the name field in the cert against the desired hostname.
 * NB: This is our only defense against Man-In-The-Middle (MITM) attacks! 
 */
```

最后一句的意思就是说使用https，是应对中间人攻击的唯一方式。为什么这么说呢，这得从https连接的过程说起。

## 四、https连接的过程

如果对于一个外行人，可以这么解释：

https连接，服务器发送它的证书给浏览器（客户端），浏览器确认证书正确，并检查证书中对应的主机名是否正确，如果正确则双方加密数据后发给对方，对方再进行解密，保证数据时不透明的。

但是如果这个外行人比较聪明，他可能会问你浏览器是怎么校验证书正确的，证书又是什么东西，加密后不会被中间人破解么？

首先证书是个什么东西，可以在浏览器上面看到证书的内容，例如我们访问谷歌，然后点击地址栏的小锁：

![地址栏](http://fed.renren.com/wp-content/uploads/2017/02/39.png)

再点击详情->查看证书，就可以看到整个证书的完整内容：

![证书的完整内容](http://fed.renren.com/wp-content/uploads/2017/02/818663-20161112181852014-1685923180.jpg)

接下来再用一个WireShark的抓包工具，抓取整个https连接的包，并分析这些包的内容。

下面以访问淘宝为例，打开淘宝，可以在Chrome里面看到淘宝的IP

![淘宝的IP](http://fed.renren.com/wp-content/uploads/2017/02/818663-20161112195031733-862916446.png)

然后打开WireShark，设定过滤条件为源IP和目的IP都为上面的IP，就可以观察到整一个连接建立的过程：

![连接建立的过程](http://fed.renren.com/wp-content/uploads/2017/02/32.png)

第一步肯定是要先建立TCP连接，这里就不说了，我们从Client Hello开始说起。

### 1、Client Hello

我们在WireShark里面观察，将Client Hello里面客户端发送给服务器的一些重要信息罗列出来

（1）使用的TLS版本时1.2，TLS有三个版本，1.0, 1.1, 1.2是最新的版本，https的加密就是考的TLS安全传输层协议：

![TLS协议](http://fed.renren.com/wp-content/uploads/2017/02/818663-20161112200532936-1702996581.png)

（2）客户端当前的时间和一个随机密码串，这个时间是距Unix元年（1970.1.1）的秒数，这里是147895117，随机数的作用下面再提及。

![随机密码](http://fed.renren.com/wp-content/uploads/2017/02/p6.png)

（3）sessionId，会话ID，第一次连接时为0，如果有sessionId，则可以恢复会话，而不用重复握手过程：

![sessionId](http://fed.renren.com/wp-content/uploads/2017/02/p7-1.png)

（4）浏览器支持的加密组合方式：可以看到，浏览器一共支持22种加密组合方式，发给服务器，让服务器选一个。具体的加密方式下文再介绍

![加密组合](http://fed.renren.com/wp-content/uploads/2017/02/p8.png)

（5）还有一个比较有趣的东西是域名：

![域名](http://fed.renren.com/wp-content/uploads/2017/02/p9.png)

为什么说张恒比较特别呢，因为域名时工作在应用层http里的，而握手是发生在TLS还在传输层。在传输层里面就把域名信息告诉服务器，好让服务器根据域名发送相应的证书。

可以输，https = http + tls，如下图所示：

![https的构成](http://fed.renren.com/wp-content/uploads/2017/02/p10.png)

数据传输还是用的http，加密用的tls。tls和ssl又是什么关系？ssl是tls的前身，ssl deprecated之后，才开始有了tls1.0、1.1、1.2。

### 2、Server Hello

服务器收到了Client Hello的信息后，就给浏览器发送了一个Server Hello的包，这个包里面有着跟Client Hello类似的消息：

（1）时间、随机数等，注意服务器还发送了一个SessionId给浏览器。

![Server Hello](http://fed.renren.com/wp-content/uploads/2017/02/p11.png)

（2）服务器选中的加密方式：服务器在客户端提供的方式里面选择了下面这种，这种加密方式也是目前很流行的一种方式：

![加密方式](http://fed.renren.com/wp-content/uploads/2017/02/p12.png)

### 3、Certificate证书

接着服务器发送一个证书的包过来：