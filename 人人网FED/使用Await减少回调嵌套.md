在开发的时候，有时候需要发送很多请求，然后经常会面临嵌套回调的问题，即在一个回调里面又嵌套了另一个回调，导致代码层层缩进得很厉害，如下代码所示：

```javascript
ajax({
    url: "/list",
    type: "GET",
    success: function(data) {
       appendToDOM(data);
        ajax({
            url: "/update",
            type: "POST",
            success: function(data) {
                util.toast("Success!");
            })
        });
    }
});
```

这样的代码看起来有点吃力，这种异步回调通常可以用Promise优化一下，可以把上面代码改成：

```javascript
new Promise(resolve => {
    ajax({
        url: "/list",
        type: "GET",
        success: data => resolve(data);
    })
}).then(data => {
   appendToDOM(data);
    ajax({
        url: "/update",
        type: "POST",
        success: function(data) {
            util.toast("Successfully!");
        })  
    }); 
});
```

Promise提供了一个resolve，方便通知什么时候异步结束了，不过本质还是一样的，还是使用回调，只是这个回调放在了then里面。

当需要获取多次异步数据的时候，可以使用Promise.all解决：

```javascript
let orderPromise = new Promise(resolve => {
    ajax("/order", "GET", data => resolve(data));
});
let userPromise = new Promise(resolve => {
    ajax("/user", "GET", data => resolve(data));
});

Promise.all([orderPromise, userPromise]).then(values => {
    let order = values[0],
         user = values[1];
});
```

但是这里也是使用了回调，有没有比较优雅的解决方式呢？

ES7的await/async可以让异步回调的写法跟写同步代码一样。第一个嵌套回调的例子可以用await改成下面的代码：

```javascript
// 使用await获取异步数据
let leadList = await new Promise(resolve => {
    ajax({
        url: "/list",
        type: "GET",
        success: data => resolve(data);
    });
});

// await让代码很自然地像瀑布流一样写下来 
appendToDom(leadList);
ajax({
    url: "/update",
    type: "POST",
    success: () => util.toast("Successfully");
});
```

Await让代码可以像瀑布流一样很自然地写下来。

第二个例子：获取多次异步数据，可以改成这样：

```javascript
let order = await new Promise(
           resolve => ajax("/order", data => resovle(data))),

    user = await new Promise(
           resolve => ajax("/user", data => resolve(data)));

// do sth. with order/user
```

这种写法就好像从本地获取数据一样，就不同套回调函数了。

Await除了用在发请求之外，还适用于其他异步场景，例如我在创建订单前线弹一个小框询问用户是要创建哪种类型的订单，然后再弹具体的设置订单的框，所以按正常思路这里需要传递一个按钮回调的点击函数，如下图所示：

![按钮点击回调](https://user-gold-cdn.xitu.io/2017/10/31/6e6b718b506e8eed65cb89b0a201dcbd?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

但其实可以使用await解决，如下代码所示：

```javascript
let quoteHandler = require("./quote");
// 弹出框询问用户并得到用户的选择
let createType = await quoteHandler.confirmCreate();
```

quote里面返回一个Promise，监听点击事件，并传递createType：

```javascript
let quoteHandler = {
    confirmCreate: function(){
        dialog.showDialog({
            contentTpl: tpl,
            className: "confirm-create-quote"
        });
        let $quoteDialog = $(".confirm-create-quote form")[0];
        return new Promise(resolve => {
            $(form.submit).on("click", function(event){
                resolve(form.createType.value);
            });
        });
    }

}
```

这样外部调用者就可以使用await了，额不用传递一个点击事件的回调函数了。

但是需要注意的是await的一次性执行特点。相对于回调函数来说，await的执行时一次性的，例如监听点击事件，然后使用await，那么点击事件只会执行一次，因为代码从上往下执行完了，所以当希望点击之后出错了还能继续修改和提交就不能使用await，另外使用await获取异步数据，如果出错了，那么成功的resolve就不会执行，后续的代码也不会执行，所以请求出错的时候基本逻辑不会有问题。

要在babel里面使用await，需要：

（1）安装一个Node包

> npm install --save-dev bable-plugin-transform-async-to-generator

（2）在工程的根目录添加一个.babelrc文件，内容为：

```json
{
  "plugins": ["transform-async-to-generator"]
}
```

（3）使用的时候先引入一个模块

```javascript
require("babel-polyfill");
```

然后就可以愉快地使用ES7的await了。

使用await的函数前面需要加上async关键字，如下代码：

```javascript
async showOrderDialog() {
     // 获取创建类型
     let createType = await quoteHandler.confirmCreate();

     // 获取老订单数据 
     let orderInfo = await orderHandler.getOrderData();
}
```

我们再举一个例子：使用await实现JS版的sleep函数，因为原生是没有提供线程休眠函数的，如下代码所示：

```javascript
function sleep (time) {
    return new Promise(resolve => 
                          setTimeout(() => resolve(), time));
}

async function start () {
    await sleep(1000);
}

start();
```

babel的await实现是转成了ES6的Generator，如下关键代码：

```javascript
while (1) {
    switch (_context.prev = _context.next) {
        case 0:
            _context.next = 2;
            // sleep返回一个Promise对象
            return sleep(1000);

        case 2:
        case "end":     
            return _context.stop();
    }
}
```

而babel的Generator也是要用ES5实现的，什么是generator呢？如下图所示：

![generator](https://user-gold-cdn.xitu.io/2017/10/31/f245365863c10adaa2a49c85253d7243?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

生成器用function*定义，每次执行生成器的next函数的时候会返回当前生成器里用yield返回的值，然后生成器的迭代器往后走一步，直到所有yield完了。

有兴趣的可以继续研究babel是如何把ES7转成ES5的，据说原生的实现还是直接基于Promise。

使用await还有一个好处，可以直接try-catch捕获异步过程抛出的异常，因为我们是不能直接捕获异步回调里面的异常的，如下代码：

```javascript
let quoteHandler = {
    confirmCreate: function(){
        $(form.submit).on("click", function(event){
            // 这里会抛undefined异常：访问了undefined的value属性
            callback(form.notFoundInput.value);
        });
    }
}

try {
    // 这里无法捕获到异常
    quoteHandler.confirmCreate();
} catch (e) {

}
```

上面的try-catch是没有办法捕获到异常的，因为try里面的代码已经执行完了，在它执行的过程中没有异常，因此无法在这里捕获，如果使用Promise的话一般是使用Promise链的catch：

```javascript
let quoteHandler = {
    confirmCreate: function(){
        return new Promise(resolve => {
            $(form.submit).on("click", function(event){
                // 这里会抛undefined异常：访问了undefined的value属性
                resolve(form.notFoundInput.value);
            });
        });
    }
}

quoteHandler.confirmCreate().then(createType => {

}).catch(e => {
    // 这里能捕获异常
});
```

而使用await，我们可以直接用同步的catch，就好像它真的变成同步执行了：

```javascript
try {
    createType = await quoteHandler.confirmCreate("order");
}catch(e){
    console.log(e);
    return;
}
```

总之使用await让代码少写了很多嵌套，很方便的逻辑处理，纵享丝滑。

