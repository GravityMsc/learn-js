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

#### a）降低preload的字体的优先级

如下代码：

```c++
  // A preloaded font should not take precedence over critical CSS or
  // parser-blocking scripts.
  if (type == Resource::kFont && is_link_preload)
    priority = kResourceLoadPriorityHigh;
```

会把预加载字体的优先级从very-high变成high。

#### b）降低defer/async的script的优先级

如下代码：

```c++
if (type == Resource::kScript) {
    // Async/Defer: Low Priority (applies to both preload and parser-inserted)
    if (FetchParameters::kLazyLoad == defer_option) {
      priority = kResourceLoadPriorityLow;
    }
}
```

script如果是defer的话，那么它的优先级会变成最低。

#### c）页面底部preload的script优先级变成medium

如下代码：

```c++
if (type == Resource::kScript) {
    // Special handling for scripts.
    // Default/Parser-Blocking/Preload early in document: High (set in
    // typeToPriority)
    // Async/Defer: Low Priority (applies to both preload and parser-inserted)
    // Preload late in document: Medium
    if (FetchParameters::kLazyLoad == defer_option) {
      priority = kResourceLoadPriorityLow;
    } else if (speculative_preload_type ==
                   FetchParameters::SpeculativePreloadType::kInDocument &&
               image_fetched_) {
      // Speculative preload is used as a signal for scripts at the bottom of
      // the document.
      priority = kResourceLoadPriorityMedium;
    }
}
```

如果是defer的script那么优先级调成最低（上面第三小点），否则如果是preload的script，并且如果页面已经加载了一张图片就认为这个script是在页面偏底部的位置，就把它的优先级调成medium。通过一个flag决定是否已经加载过第一张图片了：

```c++
  // Resources before the first image are considered "early" in the document and
  // resources after the first image are "late" in the document.  Important to
  // note that this is based on when the preload scanner discovers a resource
  // for the most part so the main parser may not have reached the image element
  // yet.
  if (type == Resource::kImage && !is_link_preload)
    image_fetched_ = true;
```

资源在第一张非preload的图片前认为是early，而在后面认为是late，late的script的优先级会偏低。

什么叫preload呢？preload不同于prefetch，在早期浏览器，script资源时阻塞加载的，当页面遇到一个script，那么要等这个script下载和执行完了，才会继续解析剩下的DOM结构，也就是说script是串行加载的，并且会堵塞页面其他资源的加载，这样会导致页面整体的加载速度很慢，所以早在2008年的时候浏览器出了一个推测加载（speculative preload）策略，即遇到script的时候，DOM会停止构建，但是会继续去搜索页面需要加载的资源，如看下后续的html有没有img/script标签，先进行预加载，而不用等到构建DOM的时候才去加载。这样大大提高了页面整体的加载速度。

#### 5）把同步即堵塞加载的资源的优先级调成最高

如下代码：

```c++
  // A manually set priority acts as a floor. This is used to ensure that
  // synchronous requests are always given the highest possible priority
  return std::max(priority, resource_request.Priority());
```

如果是同步加载的资源，那么它是request对象里面的优先级最高的，所以本来是high的ajax同步请求在最后return的时候会变成very-high。

这里是取了两个值的最大值，第一个值是上面进行各种判断得到的priority，第二个在初始这个ResourceRequest对象本省就有的一个优先级属性，返回最大值后再重新设置resource_request的优先级属性。

在构建resource request对象时所有资源的优先级都是最低的，这个可以从构造函数里面知道：

```c++
ResourceRequest::ResourceRequest(const KURL& url)
    : url_(url),
      service_worker_mode_(WebURLRequest::ServiceWorkerMode::kAll),
      priority_(kResourceLoadPriorityLowest)
      /* 其它参数略 */ {}
```

但是同步请求在初始化的时候会先设置成最高：

```c++
void FetchParameters::MakeSynchronous() {
  // Synchronous requests should always be max priority, lest they hang the
  // renderer.
  resource_request_.SetPriority(kResourceLoadPriorityHighest);
  resource_request_.SetTimeoutInterval(10);
  options_.synchronous_policy = kRequestSynchronously;
}
```

以上就是基本的资源加载优先级策略。

### （2）转换成Net的优先级

这个是在渲染线程里面进行的，上面提到的资源优先级在发送请求之前会被转换成Net的优先级：

```c++
  resource_request->priority =
      ConvertWebKitPriorityToNetPriority(request.GetPriority());
```

资源优先级对应Net的优先级关系如下所示：

![资源优先级和Net优先级的对应关系](https://user-gold-cdn.xitu.io/2017/10/29/dbd16c869789b07c9fe4b9c3b2c4973b?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

画成一个表：

![表格关系](https://user-gold-cdn.xitu.io/2017/10/29/0500debd627ee8fcba384f23254d5585?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

Net Priority是请求资源的时候使用的，这个是在Chrome的IO线程里面进行的，我在《[JS与多线程](https://fed.renren.com/2017/05/21/js-threads/)》的Chrome的多线程模型里面提到，每个页面都有Renderer线程负责渲染页面，而浏览器有IO线程，用来负责请求资源等。为什么IO线程不是放在每个页面而是放在浏览器框架呢？因为这样的好处是如果两个页面请求了相同的资源的话，如果有缓存的话就能避免重复请求了。

上面的都是在渲染线程里面debug操作得到的数据，为了能够观察资源请求的过程，需要切换到IO线程，而这两个线程间的通行是通过Chrome封装的Mojo框架进行的。在Renderer线程会发一个消息给IO线程通知它：

```c++
  mojo::Message message(
      internal::kURLLoaderFactory_CreateLoaderAndStart_Name, kFlags, 0, 0, nullptr);
 // 对这个message进行各种设置后（代码略），调接收者的Accept函数 
 ignore_result(receiver_->Accept(&message));
```

XCode里面可以看到这是在渲染线程RendererMain里操作的：

![RendererMain](https://user-gold-cdn.xitu.io/2017/10/29/9269e315afdd14b40058aa9695b80356?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

现在要切换到Chrome的IO线程，把debug的方式改一下，如下选择Chromium程序：

![debug Chrome的IO线程](https://user-gold-cdn.xitu.io/2017/10/29/687d5a4d4e588139fbc29f3e3fa9570d?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

之前是使用Attach to Process把渲染进程的PID传进来，因为每个页面都是独立的一个进程，现在要改成debug Chromium进程。然后在content/browser/loader/resource_scheduler.cc这个文件里的ShouldStartRequest函数里面打一个断点，接着在Chromium里面打开一个网页，就可以看到断点生效了。在XCode里面可以看到当前线程名称叫Chrome_IOThread：

![Chrome_IOThread](https://user-gold-cdn.xitu.io/2017/10/29/199e629642e3b8c629d8c0693e1ac852?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

这与上面的描述一致。IO线程是如何利用优先级决定要不要开始加载资源的呢？

### （3）资源加载

上面提到的ShouldStartRequest这个函数是判断当前资源是否能开始加载了，如果能的话就准备记载了，如果不能话就继续把它放到pending request队列里面，如下代码所示：

```c++
  void ScheduleRequest(const net::URLRequest& url_request,
                       ScheduledResourceRequest* request) {
    SetRequestAttributes(request, DetermineRequestAttributes(request));
    ShouldStartReqResult should_start = ShouldStartRequest(request);
    if (should_start == START_REQUEST) {
      // New requests can be started synchronously without issue.
      StartRequest(request, START_SYNC, RequestStartTrigger::NONE);
    } else {
      pending_requests_.Insert(request);
    }
  }
```

一旦受到Mojo的加载资源信息就会调用上面的ScheduleRequest函数，除了收到消息之外，还有一个地方也会调用：

```c++
  void LoadAnyStartablePendingRequests(RequestStartTrigger trigger) {
    // We iterate through all the pending requests, starting with the highest
    // priority one. 
    RequestQueue::NetQueue::iterator request_iter =
        pending_requests_.GetNextHighestIterator();

    while (request_iter != pending_requests_.End()) {
      ScheduledResourceRequest* request = *request_iter;
      ShouldStartReqResult query_result = ShouldStartRequest(request);

      if (query_result == START_REQUEST) {
        pending_requests_.Erase(request);
        StartRequest(request, START_ASYNC, trigger);
      }
  }
```

这个函数的特点是遍历pending requests，每次取出优先级最高的一个request，然后调用ShouldStartRequest判断是否能运行了，如果能的话就就把它从pending requests里面删掉，然后运行。

而这个函数会有三个地方会调用，一个是IO线程的循环判断，只要还有未完成的任务，就会触发加载，第二个是当有请求完成时会调用，第三个是要插入body标签的时候。所以主要总共有三个地方会触发加载：

（1）收到来自渲染线程IPC::Mojo的请求加载资源的消息

（2）每个请求完成之后，触发加载pending requests里还未加载的请求

（3）IO线程定时循环未完成的任务，触发加载

知道了触发加载机制，接着研究具体优先加载的过程，用以下html作为demo：

```html
<!DOCType html>
<html>
<head>
    <meta charset="utf-8">
    <link rel="icon" href="4.png">
    <img src="0.png">
    <img src="1.png">
    <link rel="stylesheet" href="1.css">
    <link rel="stylesheet" href="2.css">
    <link rel="stylesheet" href="3.css">
    <link rel="stylesheet" href="4.css">
    <link rel="stylesheet" href="5.css">
    <link rel="stylesheet" href="6.css">
    <link rel="stylesheet" href="7.css">
</head>
<body>
    <p>hello</p>
    <img src="2.png">
    <img src="3.png">
    <img src="4.png">
    <img src="5.png">
    <img src="6.png">
    <img src="7.png">
    <img src="8.png">
    <img src="9.png">

    <script src="1.js"></script>
    <script src="2.js"></script>
    <script src="3.js"></script>

    <img src="3.png">
<script>
!function(){
    let xhr = new XMLHttpRequest();
    xhr.open("GET", "https://baidu.com");
    xhr.send();
    document.write("hi");
}();
</script>
<link rel="stylesheet" href="9.css">
</body>
</html>
```

然后把Chrome的网络速度调为Fast 3G，让加载速度降低，一遍更好地观察这个过程，结果如下如所示：

![Chrome的加载过程](https://user-gold-cdn.xitu.io/2017/10/29/8f1f1fe1afa78e9a8d75ed5f69421cdb?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

从上图可以发现以下特点：

（1）每个域最多同时加载6个资源（http/1.1）

（2）CSS具有最高的优先级，最先加载，即使是放在最后面的9.css也是比前面资源先开始加载

（3）JS比图片优先加载，即使出现得比图片晚

（4）只有等CSS都加载完了，才能加载其他的资源，即使这个时候没有达到6个的限制

（5）head里面的非高优先级的资源最多能先加载一张（0.png）

（6）xhr的资源虽然具有高优先级，到那时由于它是排在3.j后面的，JS的执行时同步的，所以它排得比较靠后，如果把它排在1.js前面，那么它也会比图片先加载。

为什么是这样的呢？我们从源码寻找答案。

首先认清几个概念，请求可分为delayable和none-delayable两种：

```c++
// The priority level below which resources are considered to be delayable.
static const net::RequestPriority
    kDelayablePriorityThreshold = net::MEDIUM;
```

优先级在Medium以下的为delayable，即可推迟的，而大于等于Medium的为不可delayable的。从刚刚我们总结的表可以看出：css/js是不可推迟的，而图片、preload的js为可推迟加载：

![网络优先级](https://user-gold-cdn.xitu.io/2017/10/29/0500debd627ee8fcba384f23254d5585?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

还有一种是layout-blocking的请求：

```c++
// The priority level above which resources are considered layout-blocking if
// the html_body has not started.
static const net::RequestPriority
    kLayoutBlockingPriorityThreshold = net::MEDIUM;
```

这是当还没有渲染body标签，并且优先级在Medium之上的如CSS的请求。

然后，上面提到的ShouldStartRequest函数，这个函数是规划资源加载顺序最主要的函数，从源码注释可以知道它大概的过程：

```c++
  // ShouldStartRequest is the main scheduling algorithm.
  //
  // Requests are evaluated on five attributes:
  //
  // 1. Non-delayable requests:
  //   * Synchronous requests.
  //   * Non-HTTP[S] requests.
  //
  // 2. Requests to request-priority-capable origin servers.
  //
  // 3. High-priority requests:
  //   * Higher priority requests (> net::LOW).
  //
  // 4. Layout-blocking requests:
  //   * High-priority requests (> net::MEDIUM) initiated before the renderer has
  //     a <body>.
  //
  // 5. Low priority requests
  //
  //  The following rules are followed:
  //
  //  All types of requests:
  //   * Non-delayable, High-priority and request-priority capable requests are
  //     issued immediately.
  //   * Low priority requests are delayable.
  //   * While kInFlightNonDelayableRequestCountPerClientThreshold(=1)
  //     layout-blocking requests are loading or the body tag has not yet been
  //     parsed, limit the number of delayable requests that may be in flight
  //     to kMaxNumDelayableWhileLayoutBlockingPerClient(=1).
  //   * If no high priority or layout-blocking requests are in flight, start
  //     loading delayable requests.
  //   * Never exceed 10 delayable requests in flight per client.
  //   * Never exceed 6 delayable requests for a given host.
```

从上面的注释可以得到以下信息：

（1）高优先级的资源（>=Medium）、同步请求和非http(s)的请求能够立刻加载。

（2）只要有一个layout blocking的资源在加载，最多只加载一个delayable的资源，这个就解释了为什么0.png能够先加载。

（3）只有当layout blocking和high priority的资源加载完了，才能开始加载delayable的资源，这个就解释了为什么要等CSS加载完了才能记载其他的js/图片。

（4）同时加载的delayable资源同一个域只能有6个，同一个client即同一个页面最多只能有10个，否则要进行排队。

注意这里说的开始加载，并不是说能够开始请求建立连接了。源码里面叫in fight，在飞行中，而不是叫in request之类的，能够进行in fight的请求是指那些不用queue的请求，如下图：

![请求图](https://user-gold-cdn.xitu.io/2017/10/29/8f1f1fe1afa78e9a8d75ed5f69421cdb?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

白色条是指queue的时间段，而灰色的是已经in fight了但受到同域只能最多建立6个TCP连接等的影响而进入的stalled状态，绿色是TTFB（Time to First Byte）从开始建立TCP连接到收到第一个字节的时间，蓝色是下载的时间。

我们已经解释了大部分加载的特点的原因，对着上面那张图可以再重述一次：

（1）由于1.css到9.css这几个CSS文件是high priority或者是none delayable的，所以马上in fight，但是还是受到了同一个域最多只能有6个的限制，所以6/7/9.css这三个进入stalled的状态。

（2）1.css到5.css是layout-blocking的，所以作答只能再加载一个delayable的0.png，在它相邻的1.png就得排队了。

（3）等到high priority和layout-blocking的资源7.css/9.css/1.js加载完了，就开始加载delayable的资源，主要是preload的js和图片。

这里有个问题，为什么1.js是high priority的，而2.js和 3.js却是delayable的？为此在源码的ShouldStartRequest函数里面添加一些代码，把每次判断请求的一些关键信息打印出来：

```c++
    LOG(INFO) << "url: " << url_request.url().spec() << " priority: " << url_request.priority()
    << " has_html_body_: " << has_html_body_ << " delayable: "
    << RequestAttributesAreSet(request->attributes(), kAttributeDelayable);
```

把打印出来的信息按顺序画成以下表格：

![请求顺序](https://user-gold-cdn.xitu.io/2017/10/29/216c8971de08c890c1cb35cb117a1b09?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

1.js的优先级一开始是low的，即是delayable的，但是后面又变成了Medium就不是delayable了，是high priority，为什么它的优先级能够提高呢？一开始是low是因为它是推测加载的，所以是优先级比较低，但是当DOM构建到那里的时候它就不是preload的，变成正常的JS加载了，所以它的优先级变成了Medium，这个可以从has_html_body标签进行推测，而2.js要等到1.js下载和解析完，它能算是正常加载，否则还是推测加载，因此它的优先级没有得到提高。

本此解读到这里告一段落，我们得到了有3种原因会组织加载资源，包括CSP、Mixed Content、Origin block，CSP是自己手动设置的一些限制，Mixed Content是https页面不允许加载http的内容，Origin Block主要是svg的href只能是同源的资源。还知道了浏览器把资源归成CSS/Font/JS/Image等几类，总共有5种优先级，从Lowest到Highest，每种资源都会设定一个优先级，总的来说CSS/Font/Frame和同步请求这四种的优先级是最高的，不能推迟加载的，而正常加载的JS属于高优先级，推测加载preload则优先级会比较低，会推迟加载。并且如果有layout blocking的请求的话，那么delayable的资源要等到高优先级的加载完了才能进行加载。已经开始加载的资源还可能会处于stalled的状态，因为每个域同时建立的TCP连接数是有限的。

但是我们还有很多问题没有得到解决，例如：

（1）同源策略具体是怎样处理的？

（2）优先级是如何动态改变的？

（3）http cache/service worker是如何影响资源加载的？

我们将尝试在下一次解读进行回答，看源码是一件比较费时费力的事情，本篇是研究了三个周末四五天的视角猜得到的，而且为了避免错误不会随便进行臆测，基本上每个小点都是实际debug执行和打印console得到的，经过验证了才写出来。但是由于看的深度有限和理解偏差，可能会有一些不全面的地方甚至错误，但是从注释可以看到有些地方为什么有这个判断条码即使是源码的维护者也不太确定。本篇解读尽可能地实事求是。



