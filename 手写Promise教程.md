## 1. Promise的声明

首先呢，promise肯定是一个类，我们就用class来声明。

* 由于new Promise((resolve, reject) => {})，所以传入一个参数（函数），[Promise A+规范](https://promisesaplus.com/)里叫他executor，传入就执行。
* executor里面有两个参数，一个叫resolve（成功），一个叫reject（失败）。
* 由于resolve和reject可执行，所以都是函数，我们用let声明。

```javascript
class Promise{
    // 构造器
    constructor(executor){
        // 成功
        let resolve = () => {};
        // 失败
        let reject = () => {};
        // 立即执行
        executor(resolve, reject);
    }
}
```

## 2. 解决基本状态

[Promise A+规范](https://promisesaplus.com/)对Promise有规定：

* Promise存在三个状态（state）pending、fulfilled、rejected。
* pending（等待态）为初始态，并可以转化为fulfilled（成功态）和rejected（失败态）。
* 成功时，不可转为其他状态，且必须有一个不可改变的值（value）。
* 失败时，不可转为其他状态，且必须有一个不可改变的原因（reason）。
* new Promise((resolve, reject) => {resolve(value)})resolve为成功，接收参数value，状态改变为fulfilled，不可再次改变。
* new Promise((resolve, reject) => {reject(reason)})reject为失败，接收参数reason，状态改变为rejected，不可再次改变。
* 若是executor函数报错，直接执行reject()。

于是乎，我们获得以下代码：

```javascript
class Promise{
    constructor(executor){
        // 初始化state为等待态
        this.state = 'pending';
        // 成功的值
        this.value = undefined;
        // 失败的原因
        this.reason = undefined;
        let resolve = value => {
            // state改变，resolve调用就会失败
            if(this.state === 'pending'){
                // resolve调用后，state转化为成功态
                this.state = 'fulfilled';
                // 储存成功的值
                this.value = value;
            }
        };
        let reject = reason => {
            // state改变，reject调用就会失败
            if(this.state === 'pending'){
                // reject调用后，state转化为失败态
                this.state = 'rejected';
                // 储存失败的原因
                this.reason = reason;
            }
        };
        // 如果executor执行报错，直接执行reject
        try{
            executor(resolve, reject);
        } catch(err){
            reject(err);
        }
    }
}
```

## 3. then方法

[Promise A+规范](https://promisesaplus.com/)规定：Promise有一个叫做then的方法，里面有两个参数：onFulfilled，onRejected，成功有成功的值，失败有失败的原因

* 当状态state为fulfilled，则执行onFulfilled，传入this.value。当状态state为rejected，则执行onRejected，传入this.reason。
* onFulfilled，onRejected如果他们是函数，则必须分别在fulfilled，rejected后被调用，value或reason一次作为它们的第一个参数

```javascript
class Promise{
    constructor(executor){...}
    // then方法，有两个参数onFulfilled，onRejected
    then(onFulfilled, onRejected){
        // 状态为fulfilled，执行onFulfilled，传入成功的值
        if(this.state === 'fulfilled'){
            onFulfilled(this.value);
        };
        // 状态为rejected，执行onRejected，传入失败的原因
        if(this.state === 'rejected'){
            onRejected(this.reason);
        };
    }
}
```

这样写下来，基本的功能实现了，但是对于函数中带有setTimeout的支持还有问题。

## 4. 解决异步实现

现在基本可以实现简单的同步代码，但是当resolve在setTimeout内执行，then时state还是pending等待状态，我们就需要在then调用的时候，将成功和失败存到各自的数组，一旦reject或者resolve，就调用它们。

类似于发布订阅，先将then里面的两个函数存储起来，由于一个promise可以有多个then，所以存在同一个数组内。

```javascript
// 多个then的情况
let p = new Promise();
p.then();
p.then();
```

成功或者失败时，forEach调用它们

```javascript
class Promise{
    constructor(executor){
        this.state = 'pending';
        this.value = undefined;
        this.reason = undefined;
        // 成功存放的数组
        this.onResolvedCallbacks = [];
        // 失败存放的数组
        this.onRejectedCallbacks = [];
        let resolve = value => {
            if(this.state === 'pending'){
                this.state = 'fulfilled';
                this.value = value;
                // 一旦resolve执行，调用成功数组的函数
                this.onResolvedCallbacks.forEach(fn => fn());
            }
        };
        let reject = reason => {
            if(this.state === 'pending'){
                this.state = 'rejected';
                this.reason = reason;
                // 一旦reject执行，调用失败数组的函数
                this.onRejectedCallbacks.forEach(fn => fn());
            }
        };
        try{
            executor(resolve, reject);
        } catch(err){
            reject(err);
        }
    }
    then(onFulfilled, onRejected){
        if(this.state === 'fulfilled'){
            onFulfilled(this.value);
        };
        if(this.state === 'rejected'){
            onRejected(this.reason);
        };
        // 当状态state为pending时
        if(this.state === 'pending'){
            // onFulfilled传入到成功数组
            this.onResolvedCallbacks.push(() => {
                onFulfilled(this.value);
            })
            // onRejected传入到失败数组
            this.onRejectedCallbacks.push(() => {
                onRejected(this.reason);
            })
        }
    }
}
```

## 5. 解决链式调用

我们常常用到new Promise().then().then()，这就是链式调用，用来解决回调地狱。

1. 为了达成链式，我们默认在第一个then里返回一个promise。[Promise A+规范](https://promisesaplus.com/)规定了一种方法，就是在then里面返回一个新的promise，称为promise2：promise2 = new Promise((resolve, reject) => {})。

   * 将这个promise2返回的值传递到下一个then中。
   * 如果返回一个普通的值，则将普通的值传递给下一个then中。

2. 当我们在第一个then中return了一个参数（参数未知，需判断）。这个return出来的新的promise就是onFulfilled()或onRejected()的值。

   [Promise A+规范](https://promisesaplus.com/)则规定onFulfilled()或onRejected()的值，即第一个then返回的值，叫做x，判断x的函数叫做resolvePromise。

   * 首先，要看x是不是promise。
   * 如果是promise，则取它的结果，作为新的promise2成功的结果。
   * 如果是普通值，直接作为promise2成功的结果。
   * 所以要比较x和promise2.
   * resolvePromise的参数由promise2（默认返回的promise）、x（我们自己return的对象）、resolve、reject。
   * resolve和reject是promise2的。

```javascript
class Promise{
    constructor(executor){
        this.state = 'pending';
        this.value = undefined;
        this.reason = undefined;
        this.onResolvedCallbacks = [];
        this.onRejectedCallbacks = [];
        let resolve = value => {
            if(this.state === 'pending'){
                this.state = 'fulfilled';
                this.value = value;
                this.onResolvedCallbacks.forEach(fn => fn());
            }
        };
        let reject = reason => {
            if(this.state === 'pending'){
                this.state = 'rejected';
                this.reason = reason;
                this.onRejectedCallbacks.forEach(fn => fn());
            }
        };
        try{
            executor(resolve, reject);
        } catch(err){
            reject(err);
        }
    }
    then(onFulfilled, onRejected){
        // 声明返回的promise2
        let promise2 = new Promise((resolve, reject) =>{
            if(this.state === 'fulfilled'){
                let x = onFulfilled(this.value);
                // resolvePromise函数，处理自己return的promise和默认的promise2的关系
                resolvePromise(promise2, x, resolve, reject);
            };
            if(this.state === 'rejected'){
                let x = onRejected(this.reason);
                resolvePromise(promise2, x, resolve, reject);
            };
            if(this.state === 'pending'){
                this.onResolvedCallbacks.push(() => {
                    let x = onFulfilled(this.value);
                    resolvePromise(promise2, x, resolve, reject);
                })
                this.onRejectedCallbacks.push(() => {
                    let x = onRejected(this.reason);
                    resolvePromise(promise2, x, resolve, reject);
                })
            }
        }); 
        // 返回promise，完成链式
        return promise2;
    }
}
```

## 6. 完成resolvePromise函数

[Promise A+规范](https://promisesaplus.com/)规定了一段代码，让不同的promise代码互相套用，叫做resolvePromise。

* 如果x === promise2，则是会造成循环引用，自己等待自己完成，则报“循环引用”错误。

  ```javascript
  let p = new Promise(resolve => {
      resolve(0);
  });
  var p2 = p .then(data => {
      // 循环引用，自己等待自己完成，一辈子也完不成
      return p2;
  })
  ```

* 判断x。

  1. x不能是null。
  2. x是普通值，直接resolve(x)。
  3. x是对象或者函数（包括promise），让then = x.then。

* 如果x是对象或者函数（默认promise）

  1. 声明了then。
  2. 如果取then报错，则走reject()。
  3. 如果then是个函数，则用call执行then，第一个参数是this，后面是成功的回调和失败的回调。
  4. 如果成功的回调还是promise，就递归继续解析。

* 成功和失败只能调用一个，所以设定一个called变量来防止多次调用。

  ```javascript
  function resolvePromise(promise2, x, resolve, reject){
      // 循环引用报错
      if(x === promise2){
          // reject报错
          return reject(new TypeError('Chaining cycle detected for promise'));
      }
      // 防止多次调用
      let called;
      // x不是null，且x是对象或者函数
      if(x != null && (typeof x === 'object' || typeof x === 'function')){
          try {
              // A+规定，声明then = x的then方法
              let then = x.then;
              // 如果then是函数，就默认是promise了
              if(typeof then === 'function'){
                  // 就让then执行，第一个参数是this，后面是成功的回调和失败的回调
                  then.call(x, y => {
                      // 成功和失败只能调用一个
                      if(called)return;
                      called = true;
                      // resolve的结果依旧是promise，那就继续解析
                      resolvePromise(promise2, y, resolve, reject);
                  }, err => {
                      // 成功和失败只能调用一个
                      if(called) return;
                      called = true;
                      //失败了就失败了
                      reject(err);
                  })
              } else {
                  resolve(x); // 直接成功即可
              }
          } catch(e){
              // 也属于失败
              if(called) return;
              called = true;
              // 取then出错了那就不要再继续执行了
              reject(e);        
          }
      } else {
          resolve(x);
      }
  }
  ```

## 7. 解决其他问题

[Promise A+规范](https://promisesaplus.com/)规定onFulfilled，onRejected都是可选参数，如果他们不是函数，必须被忽略

* onFulfilled返回一个普通的值，成功时直接等于value => value。
* onRejected返回一个普通的值，失败时如果直接等于value => value，则会跑到下一个then中的onFulfilled中，所以直接扔出一个错误reason => throw err。

[Promise A+规范](https://promisesaplus.com/)规定onFulfilled或onRejected不能同步被调用，必须异步调用。我们就用setTimeout解决异步问题。

* 如果onFulfilled或onRejected报错，则直接返回reject()。

```javascript
class Promise{
    constructor(executor){
        this.state = 'pending';
        this.value = undefined;
        thsi.reason = undefined;
        this.onResolvedCallbacks = [];
        this.onRejectedCallbacks = [];
        let resolve = value => {
            if(this.state === 'pending'){
                this.state = 'fulfilled';
                this.value = value;
                this.onResolvedCallbacks.forEach(fn => fn());
            }
        };
        let reject = reason => {
            if(this.state === 'pending'){
                this.state = 'rejected';
                this.reason = reason;
                this.onRejectedCallbacks.forEach(fn => fn());
            }
        };
        try {
            executor(resolve, reject);
        } catch (err){
            reject(err);
        }
    }
    then(onFulfilled, onRejected){
        // onFulfilled如果不是函数，就忽略onFulfilled，直接返回value
        onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : value => value;
        // onRejected如果不是函数，就忽略onRejected，直接扔出错误
        onRejected = typeof onRejected === 'function' ? onRejected : err => {throw err};
        let promise2 = new Promise((resolve, reject) => {
            if(this.state === 'fulfilled'){
                // 异步
                setTimeout(() => {
                    try {
                        let x = onFulfilled(this.value);
                        resolvePromise(promise2, x, resolve, reject);
                    } catch(e){
                        reject(e);
                    }                
                }, 0);
            };
            if(this.state === 'rejected'){
                // 异步
                setTimeout(() => {
                    // 如果报错
                    try {
                        let x = onRejected(this.reason);
                        resolvePromise(promise2, x, resolve, reject);
                    } catch(e){
                        reject(e);
                    }
                }, 0);
            };
            if(this.state === 'pending'){
                this.onResolvedCallbacks.push(() => {
                    // 异步
                    setTimeout(() => {
                        try{
                            let x = onFulfilled(this.value);
                            resolvePromise(promise2, x, resolve, reject);
                        } catch(e){
                            reject(e);
                        }
                    }, 0);
                });
                this.onRejectedCallbacks.push(() => {
                    // 异步
                    setTimeout(() => {
                        try {
                            let x = onRejected(this.reason);
                            resolvePromise(promise2, x, resolve, reject);
                        } catch(e){
                            reject(e);
                        }
                    }, 0)
                });
            };
        });
        // 返回promise，完成链式
        return promise2;
    }
}
```

## 8. 大功告成

一并实现catch和resolve、reject、race、all方法。

```javascript
class Promise{
    constructor(executor){
        this.state = 'pending';
        this.value = undefined;
        this.reason = undefined;
        this.onResolvedCallbacks = [];
        this.onRejectedCallbacks = [];
        let resolve = value => {
            if(this.state === 'pending'){
                this.state = 'fulfilled';
                this.value = value;
                this.onResolvedCallbacks.forEach(fn => fn());
            }
        };
        let reject = reason => {
            if(this.state === 'pending'){
                this.state = 'rejected';
                this.reason = reason;
                this.onRejectedCallbacks.forEach(fn => fn());
            }
        };
        try{
            executor(resolve, reject);
        } catch(err){
            reject(err);
        }
    }
    then(onFulfilled, onRejected){
        onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : value => value;
        onRejected = typeof onRejected === 'function' ? onRejected : err => {throw err};
        let promise2 = new Promise((resolve, reject) => {
            if(this.state === 'fulfilled'){
                setTimeout(() => {
                    try{
                        let x = onFulfilled(this.value);
                        resolvePromise(promise2, x, resolve, reject);
                    } catch (e){
                        reject(e);
                    }
                }, 0);
            };
            if(this.state === 'rejected'){
                setTimeout(() => {
                    try {
                        let x = onRejected(this.reason);
                        resolvePromise(promise2, x, resolve, reject);
                    } catch(e){
                        reject(e);
                    }
                }, 0);
            };
            if(this.state === 'pending'){
                this.onResolvedCallbacks.push(() => {
                    setTimeout(() => {
                        try{
                            let x = onFulfilled(this.value);
                            resolvePromise(promise2, x, resolve, reject);
                        } catch(e){
                            reject(e);
                        }
                    }, 0);
                });
                this.onRejectedCallbacks.push(() => {
                    setTimeout(() => {
                        try{
                            let x = onRejected(this.reason);
                            resolvePromise(promise2, x, resolve, reject);
                        } catch(e){
                            reject(e);
                        }
                    }, 0); 
                });
            };
        });
        return promise2;
    } catch(fn){
        return this.then(null, fn);
    }
}
function resolvePromise(promise2, x, resolve, reject){
    if(x === promise2){
        return reject(new TypeError('Chaining cycle detected for promise'));
    }
    let called;
    if(x != null && (typeof x === 'object' && typeof x === 'function')){
        try{
            let then = x.then;
            if(typeof then === 'function'){
                then.call(x, y => {
                    if(called) return;
                    called = true;
                    resolvePromise(promise2, y, resolve, reject);
                } ,err => {
                    if(called) return;
                    called = true;
                    reject(err);
                })
            } else {
                resolve(x);
            }
        } catch (e){
            if(called) return;
            called = true;
            reject(e);
        }   
    } else {
        resolve(x);
    }
}
// resolve 方法
Promise.resolve = function(val){
    return new Promise((resolve, reject) => {
        resolve(val);
    });
}
// reject方法
Promise.reject = function(val){
    return new Promise((resolve, reject) => {
        reject(val);
    });
}
// race方法
Promise.race = function(promises){
    return new Promise((resolve, reject) => {
        for(let i = 0; i < promises.length; i++){
            promises[i].then(resolve, reject);
        }
    });
}
// all方法（获取所有的promise，都执行then，把结果放到数组，一起返回）
Promise.all = function(promises){
    let arr = [];
    let i = 0;
    function processData(index, data){
        arr[index] = data;
        i++;
        if(i === promises.length){
            resolve(arr);
        };
    };
    return new Promise((resolve, reject) => {
        for(let i = 0; i < promises.length; i++){
            promises[i].then(data => {
                processData(i, data);
            }, reject);
        };
    });
}
```

## 9. 如何验证我们的promise是否正确

1. 先在我们的代码后面加入下述代码。

   ```javascript
   // 目前是通过测试，他会测试一个对象
   // 语法糖
   Promise.defer = Promise.deferred = function(){
       let dfd = {};
       dfd.promise = new Promise((resolve, reject) => {
           dfd.resolve = resolve;
           dfd.reject = reject;
       });
       return dfd;
   }
   module.exports = Promise;
   ```

2. npm上有一个promises-aplus-tests插件 ，全局安装它。

3. 命令行promises-aplus-tests [js文件名]即可验证。