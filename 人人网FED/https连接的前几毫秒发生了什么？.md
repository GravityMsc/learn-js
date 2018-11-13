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

![Certificate证书](http://fed.renren.com/wp-content/uploads/2017/02/p13.png)

在WireShark里面展开证书：

![证书的详情](http://fed.renren.com/wp-content/uploads/2017/02/p14.png)

服务器总共是发了三个证书，第一个叫做*.tmall.com，第二个证书叫做GlobalSign Org，第三个叫GlobalSign Root。这三个证书是什么关系呢？这三个证书是相互依赖的关系，在浏览器里面可以看出：

![证书的依赖关系](http://fed.renren.com/wp-content/uploads/2017/02/11-1.png)

tmall的证书是依赖于GlobalSign Org的证书，换句话说，GlobalSign Org的证书为tmall的证书做担保，而根证书GlobalSign Root为GlobalSign Org做担保，形成一条依赖链。明白这点很重要，从技术的角度上来说，GlobalSign为tmall的证书做签名，只要签名验证正确就说明tmall的证书是合法的。

在tmall的证书里面会指明它的上一节证书是啥：

![上一级证书](http://fed.renren.com/wp-content/uploads/2017/02/p15.png)

现在来看下一个证书里面具体有什么内容。

除了上面提到的签名外，每个证书还包含签名的算法，和被签名的证书tbsCertiticate(to be signed Certificate)三部分：

![证书内文](http://fed.renren.com/wp-content/uploads/2017/02/5-4.png)

这个tbsCertificate里面有什么东西呢？在WireShark里面展开可以看到，里面有申请证书时所填写的国家、省份、城市、组织名称：

![tbsCertificate的详情](http://fed.renren.com/wp-content/uploads/2017/02/6-3.png)

以及证书支持的域名，可以看到淘宝就在里面：

![证书支持的域名](http://fed.renren.com/wp-content/uploads/2017/02/20.png)

证书的有效期，可以看到这个整数如果不续费到今年年底就要到期了：

![证书的有效期](http://fed.renren.com/wp-content/uploads/2017/02/p16.png)

还有证书的公钥，GlobalSign Org的公钥为：

![证书的公钥](http://fed.renren.com/wp-content/uploads/2017/02/p17.png)

我们把证书的公钥拷贝出来，它是一串270个字节的数字，16进制540位：

```java
String publicKey = "3082010a0282010100c70e6c3f23937fcc70a59d20c30e533f7ec04ec29849ca47d523ef03348574c8a3022e465c0b7dc9889d4f8bf0f89c6c8c5535dbbff2b3eafbe356e74a46d91322ca36d59bc1a8e3964393f20cbce6f9e6e899c86348787f5736691a191d5ad1d47dc29cd47fe18012ae7aea88ea57d8ca0a0a3a1249a262197a0d24f737ebb473927b05239b12b5ceeb29dfa41402b901a5d4a69c436488def87efee3f51ee5fedca3a8e46631d94c25e918b9895909aee99d1c6d370f4a1e352028e2afd4218b01c445ad6e2b63ab926b610a4d20ed73ba7ccefe16b5db9f80f0d68b6cd908794a4f7865da92bcbe35f9b3c4f927804eff9652e60220e10773e95d2bbdb2f10203010001";
```

这个公钥是由什么组成的呢？这是由N和c组成的：

> publicKey = (N, e)

其中N是一个大整数，由两个质数相乘得到：

> N = p * q

e是一个幂指数。这个就涉及到非对称加密算法，它是针对对称加密算法来说的。什么是对称加密算法呢？所谓对称加密算法是说：会话双方使用相同的加密解密方式，所以会话前需要先传递加密方式或者说是密钥，而这个密钥可能会被中间人截取。所以后来才有了非对称加密算法：加密和解密的方式不一样，加密用的密钥，而解密用的公钥，公钥是公开的，密钥是不会传播的，可能是保存在拥有视网膜扫描和荷枪实弹的警卫守护的机房当中。

第一个非对称加密算法叫Diffie-Hellman密钥交换算法，它是Diffie和Hellman发明的，后来1977年麻省理工的Rivest、Shamir和Adleman提出了一种新的非对称加密算法并以他们的名字首字母命名叫RSA。它的优点就在于：

1. 加密和解密的计算非常简单
2. 破解十分困难，只要密钥的位数够大，以目前的计算能力是无法破解出密钥的

可以说，只要有计算机网络的地方，就会有RSA。RSA加密具体是怎么进行的呢：

### 4、RSA加密和解密

假设发送的信息为Hello，由于Hello的ASCII编码为：104101108108111，所以要发送的信息为：

M = 104101108108111

即先把要发送的文本转成ASCII编码或者是Unicode编码，然后进行加密：

EM = M ^ e % N

就是把M作e次幂，然后除以N取余数，得到EM，EM即为加密后的信息。其中（N, e）就是上文提到的公钥。接下来将EM发送给对方，对方收到后用自己的密钥进行解密：

M = EM ^ d % N

将加密的信息作d次幂，再除以N取模，（N, d）就是对方的密钥，这样就能够将EM还原为M，可以证明，只要密钥和公钥是一一配对的，上式一定成立。不知道密钥的人是无法破译的，上文已经提到的破解密钥是相当困难的。

接下来回到上文提到的证书的公钥，这是一串270个字节的数字，可以拆成两部分N和e：

![公钥的组成](http://fed.renren.com/wp-content/uploads/2017/02/p18.jpg)

灰色的数字时用来作为标志位的。N是一个16进制为512位、二进制位2048位的大数字。普通的证书是1024位，2048位是一个很高安全级别，换算成10进制是617位，如果你能够将这个167位的大整数拆成两个质数相乘，就可以导出GlobalSign的密钥，也就是说你破解了GlobalSign的证书（但这是不可能的）。

e为65537，证书通常取的幂指数都为这个数字。

在证书里面知道证书使用的加密算法为RSA + SHA256，SHA是一种哈希算法，可用来校验证书是否被篡改过：

![校验证书的完整性](http://fed.renren.com/wp-content/uploads/2017/02/p19.png)

我们将encrypted的值拷贝出来就是证书的签名：

```java
String sinature = "3ec0c71903a19be74dca101a01347ac1464c97e6e5be6d3c6677d599938a13b0db7faf2603660d4aec7056c53381b5d24c2bc0217eb78fb734874714025a0f99259c5b765c26cacff0b3c20adc9b57ea7ca63ae6a2c990837473f72c19b1d29ec575f7d7f34041a2eb744ded2dff4a2e2181979dc12f1e7511464d23d1a40b82a683df4a64d84599df0ac5999abb8946d36481cf3159b6f1e07155cf0e8125b17aba962f642e0817a896fd6c83e9e7a9aeaebfcc4adaae4df834cfeebbc95342a731f7252caa2a97796b148fd35a336476b6c23feef94d012dbfe310da3ea372043d1a396580efa7201f0f405401dff00ecd86e0dcd2f0d824b596175cb07b3d";
```

这个签名是一个256个字节的数字，它是GlobalSign Org用它的密钥对TBSCertificate做的签名，接下来用GlobalSign Org的公钥进行解密：

```java
String decode = 

new BigInteger(signature, 16)   //先转成一个大数字
.pow(e)                         //再做e次幂
.mod(new BigInteger(N) )        //再模以N
.toString();

System.out.println(decode);
```

注意在实际计算中，不能直接e次幂，不然将是一个天文数字里的天文数字，计算量将会非常大，需要乘一次就模一次，很次就算出来了。计算出的结果为：

![计算签名](http://fed.renren.com/wp-content/uploads/2017/02/14-1.png)

这个结果又可以拆成几个部分来看：

![计算结果](http://fed.renren.com/wp-content/uploads/2017/02/15.jpg)

第一个字节是00，第一个字节要比其他字节都要小，第二个字节是01，表示这是一个私有密钥操作，中间的ff是用来填充，加大签名的长度，加大破解难度，最后面的64个字节就是SHA哈希的值，如果证书没有被篡改过，那么将tbsCertificate做一个SHA哈希，它的值应该是和签名里面是完全一样的。所以接下来我们手动计算一下tbsCertificate的哈希值和签名里面的哈希进行比对。

这个的计算方式如下：

hash = SHA_256(DER(tbsCertificate))

注意不是讲tbsCertificate直接哈希，是对它的DER编码值进行哈希，DER是一种加密方式。DER值可以在WireShark里面导出来，证书发过来的时候就已经被DER过了：

![比对哈希](http://fed.renren.com/wp-content/uploads/2017/02/16-1.png)

然后再用openssl计算它的哈希值，命令为：

> openssl dgst -sha256 ~/tbsCertificate.bin

计算结果为：

![哈希值](http://fed.renren.com/wp-content/uploads/2017/02/p20.png)

即为：

> bedfc063c41b62e0438bc8c0fff669de1926b5accfb487bf3fa98b8ff216d650

和上面签名里面的哈希值进行对比，即下面的绿色字：

![签名里面的哈希值](http://fed.renren.com/wp-content/uploads/2017/02/19.jpg)

可以发现：校验正确！这个证书没有被篡改过，确实是tmall的证书。那么这个时候问题来了，中间人有没有可能即篡改了证书，还能保证哈希值是对的？首先不同的字符串被SHA256哈希后的值是一样的概率比较小，同时由于密钥和公钥是一一配对的，所以中间人只能把公钥改成它的公钥，这个公钥是一个p *  q的整数，所以它必须得满足两个条件，一个是要更改成一个有意义的公钥，另一个是整个证书的内容被哈希后的值和没改之前是一样的，满足这两个条件就相当困难了。

所以我们有理由相信，只有知道了GlobalSign Org密钥的人，才能对这个证书正确地加密。也就是说，tmall的证书确实是GlobalSign颁发和签名的，我们可以用相同的方式验证GlobalSign Org是GlobalSign Root颁发和签名。但是我们为什么要相信GlobalSign你？

如果上面提到的中间人不篡改证书，而是把整一个证书都换成它自己的证书，假设叫HackSign，证书里面要带上访问的域名，所以HackSign里面也会带上*.taobao.com。这个HackSign和GlobalSign有什么区别呢？为什么我们要相信GlobalSign，而不相信HackSign呢？

因为GlobalSign Root是浏览器或者操作系统自带的证书，在火狐浏览器的证书列表里面可以看到：

![根证书](http://fed.renren.com/wp-content/uploads/2017/02/p21.png)

GlobalSign Root是Builtin的，即内置的。我们绝对相信GlobalSign Root，同时验证了GlobalSign Org的签名是合法的，GlobalSign Org给tmall的证书也是合法的。所以这个就是我们的信任链，从GlobalSign Root一直相信到tmall。而*.taobao.com的域名已经被证书机构注册了，所以另外的人是不能用\*.taobao.com去注册它的证书的。

所以证书为什么会有有效期，证书为什么要购买，从这里就可以知道原因了。

到这里，你可能又会冒出来另外一个问题，既然我不能改证书，也不能换证书，那我难道不能直接克隆你的证书放到我的机器上去，因为证书是完全公开透明的，可以在浏览器或者WireShark里面看到证书的完整内容。然后我们再用域名污染等技术将你访问的域名打到我的机器上，那么我们跟你一暮一妍的证书不也是合法的？证书不就没用了？

这个问题之前也困扰了我许久，为了回答这个问题，我们先继续讲解连接的过程。

上面已经说到浏览器已经验证了证书是合法的，接下来：

### 5、密钥交换Key Exchange

上文提到服务器选择的加密组合方式为：

Cipher Suite: TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 (0xc02f)

这一串加密方式可以分成三个部分：

![加密方式分解](http://fed.renren.com/wp-content/uploads/2017/02/24.png)

服务器选中的密钥交换加密方式为RSA，数据传输加密方式为AES，校验数据是否合法的算法为SHA256。

具体的密钥交换为ECDHE_RSA，什么叫ECDHE呢？

由于证书的密钥和公钥（2048位）都比较大，使用ECDHE算法最大的优势在于，在安全性强度相同的情况下，其所需的密钥长度较短。以DH演算法密钥长度2048位元为例。ECDHE算法的密钥长度仅需要224位元即可。

所以RSA是用来验证身份然后交换密钥的，并不是用来加密数据的，因为它太长了，计算量太大。加密数据是用的ECDHE生成的密钥和公钥。

在服务器发送了它的证书给浏览器之后，就进行了密钥交换Server Key Exchange和Client Key Exchange：

![密钥交换](http://fed.renren.com/wp-content/uploads/2017/02/25.jpg)

在WireShark里面展开，可以看到它发给客户端的公钥：

![公钥](http://fed.renren.com/wp-content/uploads/2017/02/p23.png)

这个公钥只有65个字节，260位，和上面的270个字节的公钥相比，已经短了很多。

同样地，浏览器结合服务器发给它的随机密码（Server Hello），生成它自己的主密钥，然后发送公钥给服务器：

![公钥发送](http://fed.renren.com/wp-content/uploads/2017/02/p24.png)

这个公钥也是只有65个字节。

双方交换密钥之后，浏览器给服务器发了一个明文的Change Cipher Spec的包，告诉服务器我们已经准备好，可以开始传输数据了：

![准备传输数据](http://fed.renren.com/wp-content/uploads/2017/02/28.jpg)

同样地，服务器也会给浏览器发一个Change Cipher Spec的包：

![服务器准备传输数据](http://fed.renren.com/wp-content/uploads/2017/02/29.jpg)

浏览器给服务器回了个ACK，然后就开始传输数据：

![正式开始传输数据](http://fed.renren.com/wp-content/uploads/2017/02/30.jpg)

传输数据时用的http传输的，但是数据是加密的，没有密钥是没办法解密的：

![加密的数据](http://fed.renren.com/wp-content/uploads/2017/02/31.png)

上面已经提到服务器选择的数据传输加密方式为AES，AES是一种高效的加密方式，它会使用主密钥生成另外一把密钥，其加密过程可见维基百科。

到此，整一个连接过程就讲解完了。这个时候就可以回答上文提出的证书被克隆的问题，其实答案很简单，因为是没有意义的。双方采用RSA交换公钥，使用的公钥和密钥是一一配套的，所以只要证书是对的，即公钥是对的，对面没办法知道匹配的密钥是多少，所以即使证书被克隆，对方收到的数据是无法解密的。所以没有人会偷取证书，因为是没有意义的，因为他不知道密钥，从这个角度来说证书可以验证身份的合法性是可以理解的。

这个连接的过程大概多久呢？

## 五、使用https的代价

在WireShark里面可以看到每个包的发送时间：

![包发送时间](http://fed.renren.com/wp-content/uploads/2017/02/32.png)

从最开始的Client Hello，到最后的Change Cipher Spec的包，即从4.99s到5.299s，这个建立https连接的过程未0.3s。所以使用https是需要付出代价的：

1. 建立https需要花费时间（~0.3s）
2. 数据需要加密和解密，占用更多的cpu
3. 数据加密后比原信息更大，占用更多的带宽

## 六、怎样绕过https

使用ssltrip，这个工具它的实现原理是先使用ARP欺骗和用户建立连接，然后强制将用户访问的https替换成http。即中间人和用户之间是http，而和服务器还是用的https。

怎样规避这个问题：

如果经常访问的网站时https的，某一天突然变成了https，那么很可能有问题，最直观的就是浏览器地址栏的小锁没有了：

![浏览器地址栏](http://fed.renren.com/wp-content/uploads/2017/02/39.png)

七、怎样创建一个自签名的证书

证书要么买，要么自己创建一个，可以使用openssl生成一个整数：

openssl req -x509 -nodes -sha256 -days 365 -newkey rsa:2048 -keyout test.com.key -out test.com.crt

如上，使用sha256 + rsa 2048位，有效期为365天，输出为证书test.com.crt和密钥test.com.key，在生成过程中它会让你填相关的信息：

![相关的信息](http://fed.renren.com/wp-content/uploads/2017/02/40.png)

最后会生成两个文件，一个是证书（未被签名），另一个是密钥：

![证书和密钥](http://fed.renren.com/wp-content/uploads/2017/02/41.png)

然后把这个证书添加到浏览器里面，设置为信任，浏览器就不会报NET::ERR_CERT_AUTHORITY_INVALID的错误，就可以正常使用这个证书了。

