对浏览器加载资源有很多不确定性，例如：

1. css/font的资源的优先级会比img高，资源的优先级是怎么确定的呢？
2. 资源优先级又是如何影响加载的先后顺序的呢？
3. 有几种情况可能会导致资源被阻止加载？

通过源码可以找到答案。此次源码解读基于Chromium 64（2017年10月28日更新的源码）。

下面通过加载资源的步骤，依次说明。

## 1、开始加载

通过以下命令打开Chromium，同时打开一个网页：

> chromium --renderer-startup-dialog https://www.baidu.com

Chrome会在DocumentLoader.cpp里面通过以下代码去加载：

```c++
main_resource_ =
    RawResource::FetchMainResource(fetch_params, Fetcher(), substitute_data_);
```

页面资源属于MainResource，Chrome把Resource分为以下几种：

```c++
enum Type : uint8_t {
    kMainResource,
    kImage,
    kCSSStyleSheet,
    kScript,
    kFont,
    kRaw,
    kSVGDocument,
    kXSLStyleSheet,
    kLinkPrefetch,
    kTextTrack,
    kImportResource,
    kMedia,  // Audio or video file requested by a HTML5 media element
    kManifest,
    kMock  // Only for testing
};
```

除了常见的image/css/js/font之外，我们发现还有像textTrack的资源，这个是什么东西呢？这个是video的字幕，使用webvtt格式：

```html
<video controls poster="/images/sample.gif">
   <source src="sample.mp4" type="video/mp4">
   <track kind="captions" src="sampleCaptions.vtt" srclang="en">
</video>
```

还有动态请求ajax属于Raw类型。因为ajax可以请求多种资源。

MainResource包括location即导航输入地址得到的页面、使用frame/iframe嵌套的、通过超链接点击的页面以及表单提交这几种。

接着交给稍底层的ResourceFetcher去加载，所有资源都是通过它去加载：

```c++
fetcher->RequestResource(
      params, RawResourceFactory(Resource::kMainResource), substitute_data)
```

在这个里面会先对请求做预处理。

## 2、预处理请求

每发送一个请求都会生成一个ResourceRequest对象，这个对象包含了http请求的所有信息：

![ResourceRequest对象](https://user-gold-cdn.xitu.io/2017/10/29/89d64c265fd1ef64b5ec86c0e4bc97f2?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

包括url、http_loader、http body等，还有请求的优先级信息等：

![页面加载策略](https://user-gold-cdn.xitu.io/2017/10/29/c2041c918bffb1d76378741d6510ba88?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

然后会根据页面的加载策略对这个请求做一些预处理，如下代码：

```c++
  PrepareRequestResult result = PrepareRequest(params, factory, substitute_data,
                                               identifier, blocked_reason);
  if (result == kAbort)
    return nullptr;
  if (result == kBlock)
    return ResourceForBlockedRequest(params, factory, blocked_reason);
```

prepareRequest会做两件事情，一件是检查请求是否合法，第二件是把请求做些修改。如果检查合法性返回kAbort或者kBlock，说明资源被废弃了或者被阻止了，就不去加载了。

被block的原因可能有以下几种：

```c++
enum class ResourceRequestBlockedReason {
  kCSP,              // CSP内容安全策略检查
  kMixedContent,     // mixed content
  kOrigin,           // secure origin
  kInspector,        // devtools的检查器
  kSubresourceFilter,
  kOther,
  kNone
};
```

源码里面会在这个函数里做合法性检查：

```c++
  blocked_reason = Context().CanRequest(/*参数省略*/);
  if (blocked_reason != ResourceRequestBlockedReason::kNone) {
    return kBlock;
  }
```

CanRequest函数会相应地检查以下内容：

### （1）CSP（Content Security Policy）内容安全策略检查

CSP是减少XSS攻击的一个粗略。如果我们只允许加载自己域的图片的话，可以加上下面这个meta标签：

```html
<meta http-equiv="Content-Security-Policy" content="img-src 'self';">
```

或者是后端设置这个http响应头。

self表示本域，如果加载其他域的图片浏览器将会报错：

![CSP禁止加载其他域图片的方法](https://user-gold-cdn.xitu.io/2017/10/29/8b52c15cf22100e52cb42f20f8a15191?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

所以这个可以防止一些XSS注入的跨域请求。

源码里面会检查该请求是否符合CSP的设定要求：

```c++
  const ContentSecurityPolicy* csp = GetContentSecurityPolicy();
  if (csp && !csp->AllowRequest(
                 request_context, url, options.content_security_policy_nonce,
                 options.integrity_metadata, options.parser_disposition,
                 redirect_status, reporting_policy, check_header_type)) {
    return ResourceRequestBlockedReason::kCSP;
  }
```

如果有CSP并且AllowRequest没有通过的话就会返回堵塞的原因。具体的检查过程是根据不同的类型去获取该类资源的CSP设定进行比较。

接着会根据CSP的要求改变请求：

```c++
ModifyRequestForCSP(request);
```

主要是升级http为https。

### （2）upgrade-insecure-requests

如果设定了以下CSP规则：

```html
<meta http-equiv="Content-Security-Policy" content="upgrade-insecure-requests">
```

那么会将网页的http请求强制升级为https，这是通过改变request对象实现的：

```c++
url.SetProtocol("https");
if (url.Port() == 80)
  url.SetPort(443);
resource_request.SetURL(url);
```

包括改变url的协议和端口号。

### （3）Mixed Content混合内容block

在https的网站请求http的内容就是Mixed Content，例如加载一个http的JS脚本，这种请求通常会被浏览器堵塞掉，因为http是没有加密的，容易受到中间人的攻击，如修改JS的内容，从而控制整个https的页面，而图片指类的资源即使内容被修改可能只是展示出问题，所以默认没有block掉。源码里面会检查Mixed Content的内容：

```c++
  if (ShouldBlockFetchByMixedContentCheck(request_context, frame_type,
                                          resource_request.GetRedirectStatus(),
                                          url, reporting_policy))
    return ResourceRequestBlockedReason::kMixedContent;
```

在源码里面，以下4种资源是optionally-blockable（被动混合内容）：

```c++
    // "Optionally-blockable" mixed content
    case WebURLRequest::kRequestContextAudio:
    case WebURLRequest::kRequestContextFavicon:
    case WebURLRequest::kRequestContextImage:
    case WebURLRequest::kRequestContextVideo:
      return WebMixedContentContextType::kOptionallyBlockable;
```

什么叫做被动混合内容呢？[W3C文档](https://w3c.github.io/webappsec-mixed-content/#category-optionally-blockable)是这么说的：那些不会打破页面重要部分，风险比较低的，但是使用频率又比较高的Mixed Content内容。

而剩下的其他所有资源几乎都是blockable的，包括JS/CSS/Iframe/XMLHttpRequest等：

![blockable的资源](https://user-gold-cdn.xitu.io/2017/10/29/91031bdb318e49b6d98dd3cf0f433dab?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

我们注意到img srcset里的资源也是默认会被阻止的，即下面的img会被block：

```html
<img srcset="http://fedren.com/test-1x.png 1x, http://fedren.com/test-2x.png 2x" alt>
```

但是使用src的不会被block：

```html
<img src="http://fedren.com/images/sell/icon-home.png" alt>
```

如下图所示：

![对img的特殊处理](https://user-gold-cdn.xitu.io/2017/10/29/76ddb6ece45c8f04a681bbf48f643451?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

这就是optionally-blockable和blockable资源的区分。

对于被动混合内容，如果设置strict mode：

```html
<meta http-equiv="Content-Security-Policy" content="block-all-mixed-content">
```

那么即使是optionally的也会被block掉：

```c++
    case WebMixedContentContextType::kOptionallyBlockable:
      allowed = !strict_mode;
      if (allowed) {
        content_settings_client->PassiveInsecureContentFound(url);
        client->DidDisplayInsecureContent();
      }
      break;
```

上面代码，如果strict_mode是true，allowed就是false，被动混合内容就会被阻止。

而对于主动混合内容，如果用户设置允许加载：

![主动混合内容的加载策略](https://user-gold-cdn.xitu.io/2017/10/29/ab3f051cef4502c01806698074832f01?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

那么也是可以加载的：

```c++
    case WebMixedContentContextType::kBlockable: {
      // Strictly block subresources that are mixed with respect to their
      // subframes, unless all insecure content is allowed. This is to avoid the
      // following situation: https://a.com embeds https://b.com, which loads a
      // script over insecure HTTP. The user opts to allow the insecure content,
      // thinking that they are allowing an insecure script to run on
      // https://a.com and not realizing that they are in fact allowing an
      // insecure script on https://b.com.

      bool should_ask_embedder =
          !strict_mode && settings &&
          (!settings->GetStrictlyBlockBlockableMixedContent() ||
           settings->GetAllowRunningOfInsecureContent());
      allowed = should_ask_embedder &&
                content_settings_client->AllowRunningInsecureContent(
                    settings && settings->GetAllowRunningOfInsecureContent(),
                    security_origin, url);
      break;
```

代码倒数第4行会去判断当前的client即当前页码的设置是否允许加载blockable的资源。另外源码注释还提到了一种特殊的情况，就是a.com的页面包含了b.com的页面，b.com允许加载blockable的资源，a.com在非strict mode的时候页面是允许加载的，但是如果a.com是strict mode，那么将不允许加载。

并且如果页面设置了strict mode，用户设置的允许blockable资源加载的设置将会失效：

```c++
  // If we're in strict mode, we'll automagically fail everything, and
  // intentionally skip the client checks in order to prevent degrading the
  // site's security UI.
  bool strict_mode =
      mixed_frame->GetSecurityContext()->GetInsecureRequestPolicy() &
          kBlockAllMixedContent ||
      settings->GetStrictMixedContentChecking();
```

### （4）Origin Block

这个主要是svg使用，获取svg资源的时候必须不能跨域，如果跨域以下资源将会被阻塞：

```html
<svg>
    <use href="http://cdn.test.com/images/logo.svg#abc"></use>
</svg>
```

![跨域block](https://user-gold-cdn.xitu.io/2017/10/29/9215f850e74905e7c2f0d8bc4a180d8c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

并且控制台会打印：

![控制台打印的跨域错误](https://user-gold-cdn.xitu.io/2017/10/29/c8be0f1519eced3b1e0fdbe8d6a009b8?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

源码里面会对这种使用 link加载的svg做一个校验：

```c++
    case Resource::kSVGDocument:
      if (!security_origin->CanRequest(url)) {
        PrintAccessDeniedMessage(url);
        return ResourceRequestBlockedReason::kOrigin;
      }
      break;
```

具体检验CanRequest函数主要是检查是否同源：

```c++
  // We call isSameSchemeHostPort here instead of canAccess because we want
  // to ignore document.domain effects.
  if (IsSameSchemeHostPort(target_origin.get()))
    return true;

  return false;
```

如果协议、域名、端口号都一样则通过检查。需要注意这里和同源策略是两码事，这里的源阻塞是连请求都发不出去，而同源策略只是阻塞请求的返回结果。

svg的外链一般是用来做svg的雪碧图的，但是为什么需要同源呢，如果不同源会有什么不安全的因素？这里我也不清楚，暂时没查到，[W3C](https://www.w3.org/TR/SVG2/linking.html#processingURL)只是说明了需要同源，但没有给出原因。

以上就是3种主要的block的原因。在预处理请求里面除了判断资源有没有被block或者abort（abort的原因通常是url不合法），还会计算资源的加载优先级。

## 3、资源优先级

### （1）计算资源加载优先级

通过调用以下函数设定：

```c++
  resource_request.SetPriority(ComputeLoadPriority(
      resource_type, params.GetResourceRequest(), ResourcePriority::kNotVisible,
      params.Defer(), params.GetSpeculativePreloadType(),
      params.IsLinkPreload()));
```

我们来看一下这个函数里面是怎么计算当前资源的优先级的。

首先每个资源都有一个默认的优先级，这个优先级作为初始化值。

```c++
  ResourceLoadPriority priority = TypeToPriority(type);
```

不同类型的资源优先级是这么定义的：

```c++
ResourceLoadPriority TypeToPriority(Resource::Type type) {
  switch (type) {
    case Resource::kMainResource:
    case Resource::kCSSStyleSheet:
    case Resource::kFont:
      // Also parser-blocking scripts (set explicitly in loadPriority)
      return kResourceLoadPriorityVeryHigh;
    case Resource::kXSLStyleSheet:
      DCHECK(RuntimeEnabledFeatures::XSLTEnabled());
    case Resource::kRaw:
    case Resource::kImportResource:
    case Resource::kScript:
      // Also visible resources/images (set explicitly in loadPriority)
      return kResourceLoadPriorityHigh;
    case Resource::kManifest:
    case Resource::kMock:
      // Also late-body scripts discovered by the preload scanner (set
      // explicitly in loadPriority)
      return kResourceLoadPriorityMedium;
    case Resource::kImage:
    case Resource::kTextTrack:
    case Resource::kMedia:
    case Resource::kSVGDocument:
      // Also async scripts (set explicitly in loadPriority)
      return kResourceLoadPriorityLow;
    case Resource::kLinkPrefetch:
      return kResourceLoadPriorityVeryLow;
  }

  return kResourceLoadPriorityUnresolved;
}
```

可以看到优先级总共分为五级：very-high、high、medium、low、very-low，其中MainResource页面、CSS、字体这三个的优先级是最高的，然后就是Script、Ajax这种，而图片、音视频的默认优先级是比较低的，最低的是prefetch预加载的资源。

什么是prefetch的资源呢？有时候你可能需要让一些资源预先加载好等着用，例如用户输入出错的时候在输入框右边显示一个X的图片，如果等要显示的时候再去加载就会有延时，这个时候可以用一个link标签：

```html
<link rel="prefetch" href="image.png">
```

浏览器空闲的时候就会去加载。另外还可以预解析DNS：

```html
<link rel="dns-prefetch" href="https://cdn.test.com">
```

预建立TCP连接：

```html
<link rel="preconnect" href="https://cdn.chime.me">
```

后面这两个不属于加载资源，这里顺便提一下。

注意上面的switch-case设定资源优先级有一个顺序，如果既是script又是prefetch的话得到的优先级是high，而不是prefetch的very low，因为prefetch是最后一个判断。所以在设定了资源默认的优先级之后，会再对一些情况做一些调整，主要是对prefetch/preload的资源。包括：



