### 前言

axios是目前最常用的http请求库，可以用于浏览器和node.js，在github上已有43k star左右。

Axios的主要特性包括：

* 基于Promise
* 支持浏览器和node.js
* 可添加拦截器和转换请求和响应数据
* 请求可以取消
* 自动转换JSON数据
* 客户端支持防范XSRF
* 支持个主流浏览器以及IE8+

本文将带大家一起阅读axios的源码，解析当中的一些封装技巧、具体的功能实现、以及阅读源码的一些思路。

### 环境搭建

阅读源码并不是只是一味单纯的“读”，很多时候面对复杂的前后文依赖关系以及输入输出，经常会如同乱麻一般理不出头绪。这时候往往加上一些log远胜于凭空的想象。而且可以忽略一些不那么重要的环节，使得更容易抓住主干。

工欲善其事必先利其器，我们需要打造一个playground，用来watch变化，观察我们的log输出。在axios中已经包含了一些examples，但是项目构建工具基于grunt并没有watch变化，我们需要自己去添加。

在 ./GruntFile.js 中添加 grunt.loadNpmTasks('grunt-contrib-watch'); 即可。或者我们可以使用构建工具来做到。然后启动examples服务器和watch构建任务即可。

```bash
npm run examples
npm run dev
# open localhost:3000
```

### 项目目录结构

```bash
├── /lib/                      # 项目源码目
│ ├── /adapters/               # 定义发送请求的适配器
│ │ ├── http.js                # node环境http对象
│ │ └── xhr.js                 # 浏览器环境XML对象
│ ├── /cancel/                 # 定义取消功能
│ ├── /helpers/                # 一些辅助方法
│ ├── /core/                   # 一些核心功能
│ │ ├── Axios.js               # axios实例构造函数
│ │ ├── createError.js         # 抛出错误
│ │ ├── dispatchRequest.js     # 用来调用http请求适配器方法发送请求
│ │ ├── InterceptorManager.js  # 拦截器管理器
│ │ ├── mergeConfig.js         # 合并参数
│ │ ├── settle.js              # 根据http响应状态，改变Promise的状态
│ │ └── transformData.js       # 改变数据格式
│ ├── axios.js                 # 入口，创建构造函数
│ ├── defaults.js              # 默认配置
│ └── utils.js                 # 公用工具
```

### 从API入手

分析源码的时候，我们需要先从API入手，尝试着猜想下内部的结构，带着问题再去，看源码会更加有效。

我们来大致把axios的API进行归类：

|                             API                              |                         类型                          |
| :----------------------------------------------------------: | :---------------------------------------------------: |
|                        axios(config)                         |                       发送请求                        |
|                     axios.create(config)                     | 创建新的请求实例，独立的上下文环境，实例继承axios对象 |
| axios.request(get post delete ...) / instance.request(get post delete ...) |    axios以及创建实例的请求别名（包含多种http方法）    |
|               axios.default / instance.default               |           axios以及创建实例的默认config配置           |
|                axios.interceptors / instance                 |             axios以及创建实例拦截器的配置             |
|                  axios.all() / axios.spread                  |                处理并行请求的静态方法                 |
|   axios.Cancel() / axios.CancelToken() / axios.isCancel()    |                   取消请求相关方法                    |

以上大致就是axios中api的大致分类。我们可以看出暴露给我们的axios对象下挂载的除了all和sparead以及取消请求的方法以外都被创建的实例给继承，可以单独使用并保持自己的上下文环境。让我们从入口看是看看是怎么实现的：

```javascript
'use strict';

var utils = require('./utils');
var bind = require('./helpers/bind');
var Axios = require('./core/Axios');
var mergeConfig = require('./core/mergeConfig');
var defaults = require('./defaults');

/**
 * 创建Axios实例
 */
function createInstance(defaultConfig) {
    // new Axios 得到一个上下文环境，包含defaults配置以及拦截器
    var context = new Axios(defaultConfig);
    
    // instance实例为bind返回的一个函数（即是request发送请求方法），此时this绑定到context上下文环境
    var instance = bind(Axios.prototype.request, context);
    // 将Axios构造函数中的原型方法绑定到instance上并且指定this作用域为context上下文环境
    utils.extend(instance, Axios.prototype, context);
    // 把上下文环境中的defaults以及拦截器绑定到instance实例中
    utils.extend(instance, context);
    
    return instance;
}

// axios入口其实就是一个创建好的实例
var axios = createInstance(defaults);
// 这句没太理解，根据作者的注释是：暴露Axios类去让类去继承
axios.Axios = Axios;

// 工厂函数，根据配置创建新的实例
axios.create = function create(instanceConfig) {
    return createInstance(mergeConfig(axios.defaults, instanceConfig));
};

// 绑定取消请求相关方法到入口对象
axios.Cancel = require('./cancel/Cancel');
axios.CancelToken = require('./cancel/CancelToken');
axios.isCancel = require('./cancel/isCancel');

// all和spread两个处理并行的静态方法
axios.all = function all(promises) {
    return Promise.all(promises);
};
axios.spread = require('./helpers/spread');

module.exports = axios;

// 允许使用TS中的default import语法
module.exports.default = axios;
```

通过以上入口方面分析我们可以看出端倪，axios入口其实就是通过createInstance创建出的实例和axios.create()创建出的实例一样。而源码入口中的重中之重就是createInstance这个方法。createInstance流程大致为：

1. 使用Axios函数创建上下文context，包含自己的defaults config和管理拦截器的数组
2. 利用Axios.prototype.request 和上下文创建实例instance，实例为一个request发送请求的函数this指向上下文context
3. 绑定Axios.prototype的其他方法到instance实例，this指向上下文context
4. 把上下文context中的defaults和拦截器绑定到instance实例

### 请求别名

在axios中axios.get、axios.delete、axios.head等别名请求方法其实都是指向同一个方法axios.request，只是把default config中的请求methods进行了修改而已。具体代码在Axios这个构造函数的原型上，让我们来看下源码的实现：

```javascript
utils.forEach(
	['delete', 'get', 'head', 'options'],
    function forEachMethodNoData(method) {
        Axios.prototype[method] = function(url, config) {
            return this.request(
                utils.merge(config || {}, {
                    method: method,
                    url: url
                })
            );
        };
    }
);

utils.forEach(['post', 'put', 'patch'], function forEachMethodWithData(method) {
    Axios.prototype[method] = function(url, data, config) {
        return this.request(
            utils.merge(config || {}, {
                method: method,
                url: url,
                data: data
            })
        );
    };
});
```

因为post、put、patch有请求体，所以要分开处理。请求别名方便用户快速使用各种不同API进行请求。

### 拦截器的实现

首先在我们创建实例中，会去创建上下文实例，也就是new Axios，会得到interceptors这个属性，这个属性分别又有request和response两个属性，它们的值分别是new InterceptorManager构造函数返回的数组。这个构造函数同样负责拦截器数组的添加和移除。让我们看下源码：

```javascript
'use strict';

var utils = require('./../utils');

function InterceptorManager() {
    this.handlers = [];
}

// axios或实例上调用interceptors.request.use或者interceptors.response.use
// 传入的resolve，reject将被添加到数组的尾部
InterceptorManager.prototype.use = function use(fulfilled, rejected){
    this.handlers.push({
        fulfilled: fulfilled,
        rejected: rejected
    });
    return this.handlers.length - 1;
};
// 移除拦截器，将该项在数组中置成null
InterceptorManager.prototype.eject = function eject(id) {
    if(this.handlers[id]) {
        this.handlers[id] = null;
    }
};

// 辅助方法，帮助遍历拦截器数组，跳过被eject置成null的项
InterceptorManager.prototype.forEach = function forEach(fn) {
    utils.forEach(this.handlers, function forEachHandler(h) {
        if(h !== null) {
            fn(h);
        }
    });
};

module.exports = InterceptorManager;
```

上下文环境有了拦截器的数组，又如何去做到多个拦截器请求到响应的顺序处理以及实现呢？为了了解这点我们还需要进一步往下看Axios.prototype.request方法。

### Axios.prototype.request

Axios.prototype.request方法是请求开始的入口，分别处理了请求的config，以及链式处理请求拦截器、请求、响应拦截器，并返回Promise的格式方便我们处理回调。让我们来看下源码部分：

```javascript
Axios.prototype.request = function request(config) {
    // 判断参数类型，支持axios('url', {})以及axios(config)两种形式
    if(typeof config === 'string') {
        config = arguments[1] || {};
        config.url = arguments[0];
    } else {
        config = config || {};
    }
    // 传入参数与axios或实例下的defaults属性合并
    config = mergeConfig(this.defaults, config);
    config.method = config.method ? config.method.toLowerCase() : 'get';
    
    // 创造一个请求序列数组，第一位是发送请求的方法，第二位是空
    var chain = [dispatchRequest, undefined];
    var promise = Promise.resolve(config);
    // 把实例中的请求拦截器数组中的数据依次加入到请求序列数组的头部
    this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor){
        chain.push(interceptor.fulfilled, interceptor.rejected);
    });
    // 遍历请求序列数组形成promise链依次处理并且处理完弹出请求序列数组
    while (chain.length){
        promise = promise.then(chain.shift(), chain.shift());
    }
    // 返回最终promise对象
    return promise;
};
```

我们可以看到Axios.prototype.request中使用了精妙的封装方法，形成promise链去依次挂载在then方法顺序处理，为了更清晰的认识我们可以画个图去方便认识这一过程。

![promise链](https://pic3.zhimg.com/80/v2-6075b1ceb55311820cac33da4334a92f_hd.jpg)

### 取消请求

Axios.prototype.request调用dispatchRequest是最终处理axios发起请求的函数，它的执行过程流程包括了：

1. 取消请求的处理和判断
2. 处理参数和默认参数
3. 使用相对应的环境adapter发送请求（浏览器环境使用XMLRequest对象、Node使用http对象）
4. 返回后抛出取消请求message，根据配置transformData转换响应数据

这一过程除了取消请求的处理，其余的流程都相对十分的简单，所以我们要对取消请求进行详细的分析。我们还是先看调用方式：

```javascript
const CancelToken = axios.CancelToken;
const source = CancelToken.source();

axios.get('/user/12345', {
    cancelToken: source.token
}).catch(function(thrown) {
    if(axios.isCancel(thrown)) {
        console.log('Request canceled', thrown.message);
    } else {
        // handler error
    }
});

source.cancel('Operation canceled bt the user.');
```

从调用方式我们可以看到，我们需要从config传入axios.CancelToken.source().token，并且可以用axios.CancelToken.source().cancel()执行取消请求。我们还可以看出cancel函数不仅是取消了请求，并且使得整个请求走入了rejected。从整个API设计我们就可以看出这块的功能可能有点复杂，让我们一点点来分析，从CancelToken.source整个方法看实现过程：

```javascript
CancelToken.source = function source() {
    var cancel;
    var token = new CancelToken(function executor(c) {
        cancel = c;
    });
    return {
        token: token,
        cancel: cancel
    };
};
```

axios.CancelToken.source().token返回的是一个new CancelToken的实例，axios.CancelToken.source().canel是传入到new CancelToken中的方法的一个参数。再看下CancelToken整个构造函数：

```javascript
function CancelToken(executor) {
    if (typeof executor !== 'function') {
        throw new TypeError('executor must be a function.');
    }
    
    var resolvePromise;
    this.promise = new Promise(function promiseExecutor(resolve) {
        resolvePromise = resolve;
    });
    
    var token = this;
    executor(function cancel(message) {
        if(token.reason) {
            return;
        }
        
        token.reason = new Cancel(message);
        resolvePromise(token.reason);
    });
}
```

我们根据构造函数可以知道axios.CancelToken.source().token最终拿到的实例下挂载了promise和reason两个属性，promise属性是一个处于pending状态的promise实例，reason是指向cancel后传入的message。而axios.CancelToken.source().cancel是一个函数方法，负责判断是否指向，若未执行拿到axios.CancelToken.source().token。promise中executor的resolve参数，作为触发器，触发处于pending状态中的promise并且传入的message挂载在axios.CancelToken.source().token.reason下。若有已经挂载在reason下则返回防止反复触发。而这个pending状态的promise在cancel后又是怎么进入axios总体promise的rejected中的呢？我们需要看看adpater中的处理：

```javascript
// 如果有cancelToken
if(config.cancelToken) {
    config.cancelToken.promise.then(function onCanceled(cancel) {
        if(!request) {
            return;
        }
        // 取消请求
        request.abort();
        // axios的promise进入rejected
        reject(cancel);
        // 清除request请求对象
        request = null;
    });
}
```

取消请求的总体逻辑大体如此，可能理解起来比较困难，需要反复看源码感受内部的流程，让我们大致在屡一下大致流程：

1. axios.CancelToken.source()返回一个对象，tokens属性CancelToken类的实例，cancel是tokens内部promise的resolve触发器
2. axios的config接受了CancelToken类的实例
3. 当cancel触发于pending中的tokens.promise，取消请求，把axios的promise走向rejected状态

### 总结

axios的大体流程介绍完了，我们可以画个图更加直观的梳理一下。

![axios流程图](https://pic1.zhimg.com/80/v2-a35d475ecf0d4ad1029551214a70bca9_hd.jpg)

