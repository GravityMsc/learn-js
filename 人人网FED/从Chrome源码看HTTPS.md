我在《[https连接的前几毫秒发生了什么](https://fed.renren.com/2017/02/03/https/)》详细地介绍了https连接的过程，该篇通过抓包工具分析整个过程，本篇将从Chrome源码的角度着重介绍加密和解密的过程，并补充更多的细节。

Chrome/Chromium是使用[BoringSSL](https://boringssl.googlesource.com/boringssl/)作为TLS层的库，它是[OpenSSL](https://www.openssl.org/)的一个fork，是Chrome改于openssl以适应自己产品的特点，代码位于[src/third_party/boringssl](https://cs.chromium.org/chromium/src/third_party/boringssl/)。

HTTPS连接的第一步——发送Client Hello，浏览器在Client Hello报文里面填充了使用的TLS版本、client随机数、加密列表（cipher suites）和包含了hostname的扩展。

浏览器支持的TLS版本总共有5个：

```c++
#define SSL3_VERSION 0x0300   // 3.0
#define TLS1_VERSION 0x0301   // 3.1
#define TLS1_1_VERSION 0x0302 // 3.2
#define TLS1_2_VERSION 0x0303 // 3.3 (TLS 1.2)
#define TLS1_3_VERSION 0x0304 // 3.4 (TLS 1.3)
```

最新的版本为TLS 1.3，目前只有Chrome和Firefox支持，nginx1.13（非稳定版本）/CloudFlare支持，当前使用比较广泛的还是TLS 1.2版本。Chrome在Client Hello里面设置的TLS为1.2：

```c++
// hs为SSL_HandShake
hs->client_version =
    hs->max_version >= TLS1_2_VERSION ? TLS1_2_VERSION : hs->max_version;
```

除了TLS之外，还有支持UDP的DTLS：

```c++
#define DTLS1_VERSION 0xfeff
#define DTLS1_2_VERSION 0xfefd
```

打印出来的加密列表cipher suite总共有13个：

```c++
TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256
TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256
TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA
TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA
TLS_RSA_WITH_AES_128_GCM_SHA256
TLS_RSA_WITH_AES_256_GCM_SHA384
TLS_RSA_WITH_AES_128_CBC_SHA
TLS_RSA_WITH_AES_256_CBC_SHA
TLS_RSA_WITH_3DES_EDE_CBC_SHA
```

这是浏览器支持的加密方式，放在Client Hello里面发给服务端选择一个。上面的每一个加密方式都是用的两个字节的数字编号表示，如第一个编号为0xc02B，这个是在[RFC5289](https://tools.ietf.org/html/rfc5289)进行的规定。

这一长串的加密名字表示什么呢？以TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256为例，如下图所示：

![加密方式](https://user-gold-cdn.xitu.io/2018/2/26/161d226da06881b7?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

密钥交换使用ECDHE算法，服务身份验证使用RSA算法，数据传输加密使用AES（+GCM），握手使用SHA256检验。

换句话说，证书签名使用RSA，如果证书验证正确，那么将使用ECDHE算法进行密钥交换，保证浏览器和服务拥有相同的私有密钥，然后一方使用这把密钥进行AES数据加密，另一方使用相同的密钥进行AES数据解密。验证证书签名合法性和密钥交换的身份确认都是使用SHA256这个哈希算法机芯检验。具体过程下文展开描述。

接下来，服务端进行Server Hello的响应，包括服务端要使用的TLS版本，我们访问google.com的时候谷歌返回的版本为TLS 1.2（0x303，即十进制的771）：

![服务器选择的TLS版本](https://user-gold-cdn.xitu.io/2018/2/26/161d226da054f275?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

如果服务器返回的TLS版本为1.3，那么Chrome将使用1.3版本。

Server Hello还返回一个32字节的随机数server random，和浏览器发送的随机数client random相似，这种随机数叫做nonce，用于一次性使用，通常会带有时间戳，在后面生成master key的时候用到。

还会返回一个sessionId用于下次复用当前握手的信息，避免短时间内重复握手。

同时返回所选择的加密方式，如下图所示：

![服务器返回选择的加密方式](https://user-gold-cdn.xitu.io/2018/2/26/161d226da0734daa?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

谷歌服务器使用上面举例的加密方式，根据观察，这也是很多服务器选择的方式，这应该是权衡了安全性和计算复杂度的一种比较好的方式。知道了加密方式之后（包括证书是使用RSA签名的），接下来等收到服务器发过来的证书后，读取证书并检验证书的合法性。

证书的校验是Post体格Task给TaskScheduler线程独立检验的，其他的握手操作都是在Chrome的IO线程进行，应该是考虑到证书的检验比较复杂，所以搞成异步的。

证书的检验Chrome没有使用BoringSSL提供的API，而是自己实现的，在src/net.cert目录。这个过程是这样的，首先会检验是否在黑名单里，这个黑名单如下源码的注释：

```c++
  // CloudFlare revoked all certificates issued prior to April 2nd, 2014. Thus
  // all certificates where the CN ends with ".cloudflare.com" with a prior
  // issuance date are rejected.
  //
  // The old certs had a lifetime of five years, so this can be removed April
  // 2nd, 2019.
```

大意是说证书的通用名（通常就是证书的域名）是以.cloudflare.com记为的证书，并且是2014.4.2前签发的，已经被取消掉了，这些证书有5年的有效期，现在仍然处于有效期，所以需要认为是无效的。

接着检验正式签名的合法性，在Mac上Chrome是调用系统函数SecTrustEvaluate做的检验。检验的过程我在《[https连接的前几毫秒发生了什么](https://fed.renren.com/2017/02/03/https/)》已经做了详细介绍，大概来说，先对证书进行SHA256得到一个哈希值，然后用证书的公钥对证书的签名进行解密从中取得另一个哈希值，如果这两个哈希值相等，说明证书没有被篡改过，确实是权威机构颁发。

一般来说，所谓数字签名，就是对所发送的内容做一个哈希，然后接收方用内用计算一个哈希值，如果这个值等于签名里的哈希，就说明内容没有被第三方篡改过。而这个签名通常是加密的，在证书里面，这个签名是使用证书的私钥进行加密，任何人都可以拿证书里提供的公钥进行解密，但是任何人没有私钥无法正确地加密，因为私钥和公钥是一一配对的，如果那另外一把私钥进行加密，再拿原先的公钥进行解密必定不是原先的内容。

所以如果前面校验正确，那么发送的内容即证书是合法的（证书里面有域名、公钥等信息）。如果这一步的检验不合法，将返回CERT_STATUS_AUTHORITY_INVALID的错误。

再接着检验证书里指定的Common Name通用名是否匹配，如下图所示：

![检验通用名匹配](https://user-gold-cdn.xitu.io/2018/2/26/161d226da0458ada?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

当前访问的hostname为www.google.co.kr，而证书里面的通用名为*.google.co.kr：

![通用名检验](https://user-gold-cdn.xitu.io/2018/2/26/161d226da0f00519?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

www.google.co.kr包含在通配符*.google.co.kr里的，所以这个检验是通过的。如果不通过浏览器将会显示CERT_STATUS_COMMON_NAME_INVALID的错误。

关于这个通配符，有一个小细节，如果通配符是*.com这种顶级域名的那么认为是不合法的，只允许私人注册的域：（这种支持泛域名的证书会比只支持固定域名的贵）

```c++
// Do not allow wildcards for public/ICANN registry controlled domains -
// that is, prevent *.com or *.co.uk as valid presented names, but do not
// prevent *.appspot.com (a private registry controlled domain).
```

然后检测证书是否在公共的黑名单里面：

![网址黑名单](https://user-gold-cdn.xitu.io/2018/2/26/161d226da45d6b6f?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

如果是的话返回证书被取消的状态：CERT_STATUS_REVOKED，这些黑名单列表可见[blacklist](https://chromium.googlesource.com/chromium/src/+show/master/net/data/ssl/blacklist/README.md)。这些黑名单包括China Internet Network Information Center（CNNIC）等，因为公钥固定导致不安全的原因，具体可以见文档附上的链接说明。

再接着检查证书是否使用了弱签名算法如SHA1/MD5:

![是否使用了弱签名算法](https://user-gold-cdn.xitu.io/2018/2/26/161d226dcc954d53?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

如果是的话，返回CERT_STATUS_WEAK_SIGNATURE_ALGORITHM，因为SHA1和MD5都被认为是不安全的哈希算法，容易被碰撞攻击（如2017年2月23日，Google宣布了一个成功的SHA-1碰撞攻击，发布了两份内容不同但SHA-1散列值相同的PDF文件作为概念证明，详见[维基百科](https://zh.wikipedia.org/wiki/SHA-1)）。

紧接着检验证书是否是赛门铁克颁发的：

```c++
// Distrust Symantec-issued certificates, as described at
// https://security.googleblog.com/2017/09/chromes-plan-to-distrust-symantec.html
```

如果是Symantec颁发的，将会在Chrome 66版本（2018.4.17发布稳定版本）取消信任，赛门铁克是全球几大证书机构之一，旗下的根证书包括GeoTrust、VeriSign等：

![赛门铁克](https://user-gold-cdn.xitu.io/2018/2/26/161d226dcc8c3dd5?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

为什么谷歌要取消对它的信任，[谷歌的blog](https://security.googleblog.com/2017/09/chromes-plan-to-distrust-symantec.html)是这么说的：

>During the subsequent investigation, it was revealed that Symantec had entrusted several organizations with the ability to issue certificates without the appropriate or necessary oversight, and had been aware of security deficiencies at these organizations for some time.

大意是经过调查，在没有被监督的情况下它随意委任几家机构颁发证书。当我们打开某些网站，控制台提示：

> The SSL certificate used to load resources from https://***.com will be distrusted in M70. Once distrusted, users will be prevented from loading these resources. See https://g.co/chrome/symantecpkicerts for more information.

就是因为它们使用了GeoTrust颁发的证书。

Chrome还会进行其他的检验，包括证书的有效期是否过长，如下源码注释：

```c++
// For certificates issued after 1 July 2012: 60 months.
// For certificates issued after 1 April 2015: 39 months.
// For certificates issued after 1 March 2018: 825 days.
```

还有证书本身的格式是否合法（CERT_STATUS_INVALID）等等。如果是EV增强型证书还有一些特殊的检验，有些证书需要使用在线证书状态协议（OCSP）进行检验。

检验证书的合法性和握手（HandShake）是同步进行的，因为它是运行在独立的线程。正常来说在Server Hello之后服务发送证书给浏览器进行检验，检验成功才进行下一步操作，可能Chrome考虑到检验比较耗时，所以弄成异步的。

不管怎么样，在Server Hello之后便进行密钥交换，密钥交换的目的是为了双方共享密钥，使用同一把密钥进行加密和解密。密钥交换的方式有两种RSA和ECDHE，RSA的方式比较简单，浏览器生成一把密钥，然后使用证书RSA的公钥进行加密发给服务端，服务再使用它的密钥进行解密得到密钥，这样就能够共享密钥了，它的缺点是攻击者虽然在发送的过程中无法破解，但是如果它保存了所有加密的数据，等到证书到期没有被维护之类的原因导致私钥泄漏，那么它就可以使用这把私钥去解密之前传送过的所有数据。而使用ECDHE是一种更安全的密钥交换算法。

ECDHE的全称叫Elliptic Curve Diffie-Hellman Key Exchange椭圆曲线迪非-赫尔曼密钥交换，它是迪非-赫尔曼密钥交换的变种，使用椭圆曲线加密提高安全性。

迪非-赫尔曼密钥交换的过程是这样的：交换密钥双方甲和乙选取一个基数g，例如g = 2，然后甲和乙产生自己的密钥a和b，甲发送A = g ^ a和g给乙，乙收到后计算得到共享密钥K = A ^ b = g ^ (ab)，同时把B = g ^ b发给甲，这样甲也能得到共享密钥K = B ^ a = g ^ (ab)。如下图所示：

 ![椭圆曲线加密算法](https://user-gold-cdn.xitu.io/2018/2/26/161d226dcc7c7709?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

由于a和b通常会很大，做a或b次幂会是一个天文数字，所以需要模以一个大素数p。

窃听者能够知道g、A、B，但是不知道任何一方的密钥a或者b，所以他无法知道共享密钥K是什么。为了保证传递的信息不回被人篡改，密钥交换的数据需要使用证书的RSA进行签名，更详细的说明可参见[维基百科](https://zh.wikipedia.org/wiki/%E8%BF%AA%E8%8F%B2-%E8%B5%AB%E7%88%BE%E6%9B%BC%E5%AF%86%E9%91%B0%E4%BA%A4%E6%8F%9B)。

通过幂方的计算值传递，比较容易被破解得到双方各自的密钥，这种的安全系数不是很高，所以引入了椭圆曲线加密ECC。ECC和RSA一样也可以当做证书的加密算法，ECC和RSA的共同特点是加密步骤很简单，但是解密非常困难，RSA的困难之处在于把一个大数拆成两个素数相乘，而ECC的难点在于找到一个点的系数。不同点是ECC的破解难度要远远大于RSA，举例来说2048位的RSA的破解难度相当于224位的ECC，长度越短就意味着CPU计算消耗越少，速度越快。ECC在很高级别的加密场合有比较广泛的应用。越来越多的证书使用ECC加密，如*.google.com的域名都是使用的ECC加密的证书，相对于其他RSA证书的2048位的公钥，ECC正式只有256位：

![ECC证书](https://user-gold-cdn.xitu.io/2018/2/26/161d226dccaef255?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

具体来说，所谓椭圆曲线就是指以下方程：

y^3 = x^2 + ax + b

如下图所示：

![椭圆曲线图示](https://user-gold-cdn.xitu.io/2018/2/26/161d226dce4eb84e?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

上图由一个起始点P计算2P——先画一条线与P点相切，与曲线的-2P点相交，做这个点的反射与曲线的焦点就是2P，而计算3P就是2P + P，如下图所示，连接P与2P，与曲线的第三个焦点就是-3P，反射一下就得到3P：（任意一条直线与椭圆曲线最多只有3个交点）

![直线与椭圆曲线的交点](https://user-gold-cdn.xitu.io/2018/2/26/161d226dd0200cc5?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

以此类推，4P = 3P + P，连接3P与P与曲线的交点的反射就是4P。如果经过n此后最后连线与x轴垂直，说明所有的点已用完，总共有n（或者叫order）个点，在这个计算过程中会去一个大数P用来做模数，当点的坐标值大于p时就模一下，起始点P(x, y)叫Generator点，再加上方程参数的两个系数ab——（a, b, order, x,  y）就构成了一组椭圆曲线的基本参数。

椭圆曲线难以破解的地方在于——给定点P和Q，Q = kP(1< k < n)，想要推导出k是一件很困难啊的事情（通常n会很大）。

因此使用椭圆曲线加密的密钥交换过程就变成：

![椭圆曲线的密钥交换过程](https://user-gold-cdn.xitu.io/2018/2/26/161d226ded1e03e3?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

中间人或者窃听者能够知道Q1和Q2以及方程系数a、b和起始点P，但是它无法推导出双方各自的密钥x、y，因此它没有办法计算得到共享密钥K = xyP。并且这个破解的难度要远远大于使用幂方的方式，这个就是ECDHE。更详细的信息可以查看这个[视频教程](https://www.youtube.com/watch?v=F3zzNa42-tQ)。

在实际的实现里，基本参数不是在密钥交换中传递的，而是约定的固定的曲线，在调试过程中，我们发现Chrome总共支持3种曲线：

```c++
static const uint16_t kDefaultGroups[] = {
    SSL_CURVE_X25519,
    SSL_CURVE_SECP256R1,
    SSL_CURVE_SECP384R1,
};
```

www.google.co.kr使用的是Curve X25519，[X25519](https://en.wikipedia.org/wiki/Curve25519)使用的曲线方程为：

y^2 = x^ 3 + 486662x2 + x

而*.google.com使用的是Curve secp256r1，简称为P-256，这个是在Server Key Exchange里面指定的：

![选择的特定的椭圆曲线](https://user-gold-cdn.xitu.io/2018/2/26/161d226df0202807?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

它的参数是这样的：

![椭圆曲线的参数](https://user-gold-cdn.xitu.io/2018/2/26/161d226dfae8dd2f?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

如果转换成十进制的话：

```c++
a = 115792089129476408780076832771566570560534619664239564663761773211729002495996
b = 99593677540221402957765480916910020772520766868399186769503856397241456836063
n = 115792089210356248762697446949407573529996955224135760342422259061068512044369
```

我们看到n是一个78位的数字，所以暴力破解k(P = kG, 1 < k < n)基本上是不可能的。

确定基本方程后，双方Q1和Q2值是在Server Key Exchange和Client Key Exchange以公钥的形式的进行交换。

为了确保密钥交换不会被篡改，需要进行签名，如果签名使用的是RSA的话，那么丰富和验证证书有效性一样。如果证书是ECC的帧数，那么会使用ECDSA（ecdsa_secp256r1_sha256(0x0403,)）进行签名：

![签名算法](https://user-gold-cdn.xitu.io/2018/2/26/161d226dfcfd6757?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

具体验证的函数是使用ECDSA_do_verify这个函数，过程说明可参考[维基百科](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm)，步骤比较多，这里不深入讨论。ECC证书也有公钥和密钥，最后验证合法的标准是使用公钥解密的签名里面的r值如果等于手动计算的值，则说明正确。

接着Client Key Exchange，Chrome根据曲线类型（x25519或者P-256）使用相应的参数和算法生成公钥和密钥对，如X25519的密钥是使用随机数生成的：

![密钥对](https://user-gold-cdn.xitu.io/2018/2/26/161d226e3a6b8981?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

有了密钥再计算配套的公钥，然后把公钥保存起来发出去，并计算共享密钥，最核心的代码应该是以下几行：

```c++
// Compute the x-coordinate of |peer_key| * |private_key_|.
EC_POINT_mul(group.get(), result.get(), NULL, peer_point.get(),
                  private_key_.get(), bn_ctx.get()
```

使用对方的公钥private_key_，得到K = yQ1。

紧接着用这个共享密钥经过[PRF](https://tools.ietf.org/html/rfc5246#section-5)计算得到主密钥master key。我们可以把某次握手得到的密钥打印出来，如下所示：

```c++
连接域名：www.google.com
加密方式：TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
曲线名称：SSL_CURVE_X25519
Peer Public Key (64B):   8b4364a862a7a7f19404973237079b692c1208b8ecf7828d9eae2b76e68e5012
Chrome Public Key (64B): cddd4c2d0c9d49903438a953076fb3baebd38cfa4a3b18144365b67756b4c075
Share Key (78B):         653d6e28202ff88dff92db77c91406b7992a0f15325b0192f17a317e7ff71930404dc7d4857f03
Master Key (96B):        eb584819ae738a45fe9a2e60734d0ae833dfb2d63a1900ee820a36db27a3844e5b6259e2c84e06fd1474c7e1857989ad

连接域名：www.baidu.com
加密方式：TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
曲线名称：SSL_CURVE_SECP256R1
Peer Public Key (142B):   04ac277ce63eb420e9e973c96cdf67e37a5956b949af4b053ca5b1b4b1f884b7f6cadbe2d64a91d43a2e280da528d6b6505bc6be10455e70aeabe569562ccc7bdebc7b5df80705
Chrome Public Key (130B): 04941ec80392f0bf13268a9791e7ee673df0a00af6e59335655b0519fbc575bfcb39eabd80f81118dca4906f776c801aee26f8f4fc195917dc94f9c324886bebc4
Share Key (78B):          52d0f6fc4ecd83107fb8c1cc7fa3f978152c0936c58d8d62d6885f7a672cf87c21212121212103
Mater Key (96B):          1e95a25c356a170c6829ec27a0216c50738b758f93606e8503a2e306796fd99db6ec49f65818a125bba6449b07648262
```

密钥交换之后，双方已经有了相同的密钥，然后通过发送Change Cipher Spec通知对方下一个包将会使用之前约定的方式进行加密。由于传送数据指定的是GCM加密，它是一种[AEAD](https://en.wikipedia.org/wiki/Authenticated_encryption)的加密方式，Chrome会在Change Cipher的过程中做AEAD的配置，这个加密方式的特点是会给数据加认证标签，如果标签对得上说明数据完整没有被破坏。

至此整个TLS握手完成，然后就是发送HTTP请求和接收响应数据了。

数据传送使用AES加密的特点是使用一把密钥加密，在使用相同的密钥就可以解密，具体加密和解密的过程比较复杂，这里不深入研究。不过我们可以把加密前和加密后的数据打印出来，如下图所示：

![加密前和加密后数据的对比](https://user-gold-cdn.xitu.io/2018/2/26/161d226e42b12d92?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

可以看到这是一个HTTP请求，加密前的数据有572B，加密后的数据有601B，体积增长了5%。

这个请求收到以下解密后的响应数据：

![解密后的响应数据](https://user-gold-cdn.xitu.io/2018/2/26/161d226e42bc45a4?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

还有紧接着的gzip压缩的数据。

至此整个过程就说明完了，本篇重点说了Chrome是怎么检验证书合法性的、Diff-Hellman算法是怎么样的、椭圆曲线是怎么加密的、怎样使用ECDHE进行密钥交换，等等。本文很多东西没有讲得很深入，都是点到为止，看完本篇应该对HTTPS整一个加密的过程有一个轮廓的了解，并且对一些加密算法原理有所了解。

