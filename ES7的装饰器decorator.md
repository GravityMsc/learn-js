官方说法，修饰器（Decorator）函数，用来修改类的行为。这样讲对于初学者来说不是很好理解，通俗点讲就是我们可以用修饰器来修改类的属性和方法，比如我们可以在函数执行之前改变它的行为。因为decorator是在编译时执行的，使得让我们能够在设计时对类、属性等进行标注和修改成为了可能。decorator不仅仅可以在类上面使用，还可以在对象上面使用，但是decorator不能修饰函数，因为函数存在变量提升。decorator相当于给对象内的函数包装一层行为。decorator本身就是一个函数，他有三个参数：target（所要修饰的目标类）、name（所要修饰的属性名）、descriptor（该属性的描述对象）。后面我们会让大家体会到decorator的强大魅力。

大型框架都在使用decorator？

* Angular2中的TypeScript Annotate就是标注装潢器的另一类实现。
* React中redux2也开始利用ES7的Decorators进行了大量重构。
* Vue如果你在使用typescript，你会发现Vue组件也开始用Decorator了，就连vuex也全部用Decorators重构。

接下来让我们举一个简单的readonly的例子：

这是一个Dog类

```javascript
class Dog {
    bark() {
        return '汪汪汪！！';
    }
}
```

让我们给他加上@readonly修饰器后

```javascript
import { readOnly } from "./decorators";

class Dog {
    @readonly
    bark() {
        return '汪汪汪！！';
    }
}

let dog = new Dog();
dog.bark = 'wangwang!!';
// Cannot assign to read only property 'bark' of [object Object]
// 这里readonly修饰器把Dog类的bark方法修改为只读状态
```

让我们看下readonly是怎么实现的，代码很简单

```javascript
/**
 * @param target 目标类Dog
 * @param name 所要修饰的属性名 bark
 * @param descriptor 该属性的描述对象 bark方法
 */
function readonly(target, name, descriptor) {
  // descriptor对象原来的值如下
  // {
  //   value: specifiedFunction,
  //   enumerable: false,
  //   configurable: true,
  //   writable: true
  // };
  descriptor.writable = false;
  return descriptor
}
```

readonly有三个参数，第一个target是目标类Dog，第二个是所要修饰的属性名bark，是一个字符串，第三个是该属性的描述对象，bark方法。这里我们用readonly方法将bark方法修饰为只读。所以当你修改bark方法的时候就是报错了。

decorator实用的decorator库core-decorators.js

> npm install core-decorators --save

```javascript
// 将某个属性或方法标记为不可写。
@readonly   
// 标记一个属性或方法，以便它不能被删除; 也阻止了它通过Object.defineProperty被重新配置
@nonconfigurable  
// 立即将提供的函数和参数应用于该方法，允许您使用lodash提供的任意助手来包装方法。 第一个参数是要应用的函数，所有其他参数将传递给该装饰函数。
@decorate  
// 如果你没有像Babel 6那样的装饰器语言支持，或者甚至没有编译器的vanilla ES5代码，那么可以使用applyDecorators（）助手。
@extendDescriptor
// 将属性标记为不可枚举。
@nonenumerable
// 防止属性初始值设定项运行，直到实际查找修饰的属性。
@lazyInitialize
// 强制调用此函数始终将此引用到类实例，即使该函数被传递或将失去其上下文。
@autobind
// 使用弃用消息调用console.warn（）。 提供自定义消息以覆盖默认消息。
@deprecate
// 在调用装饰函数时禁止任何JavaScript console.warn（）调用。
@suppressWarnings
// 将属性标记为可枚举。
@enumerable
// 检查标记的方法是否确实覆盖了原型链上相同签名的函数。
@override  
// 使用console.time和console.timeEnd为函数计时提供唯一标签，其默认前缀为ClassName.method。
@time
// 使用console.profile和console.profileEnd提供函数分析，并使用默认前缀为ClassName.method的唯一标签。
@profile
```

还有很多这里就不过多介绍，了解更多<https://github.com/jayphelps/core-decorators>。

下面给大家介绍一些我们团队写的一些很实用的decorator方法库。

* noConcurrent：避免并发调用，在上一次操作结果返回之前，不响应重复操作

  ```javascript
  import {noConcurrent} from './decorators';
  methods: {
    @noConcurrent     //避免并发，点击提交后，在接口返回之前无视后续点击
    async onSubmit(){
      let submitRes = await this.$http({...});
      //...
      return;
    }
  }
  ```

* makeMutex：多函数互斥，具有相同互斥标识的函数不会并发执行

  ```javascript
  import {makeMutex} from './decorators';
  let globalStore = {};
  class Navigator {
    @makeMutex({namespace:globalStore, mutexId:'navigate'}) //避免跳转相关函数并发执行
    static async navigateTo(route){...}
  
    @makeMutex({namespace:globalStore, mutexId:'navigate'}) //避免跳转相关函数并发执行
    static async redirectTo(route){...}
  }
  ```

* withErrRoast：捕获async函数中的异常，并进行错误提示

  ```javascript
  methods: {
    @withErrToast({defaultMsg: '网络错误', duration: 2000})
    async pullData(){
      let submitRes = await this.$http({...});
      //...
      return '其他原因'; // toast提示 其他原因
      // return 'ok';   // 正常无提示
    }
  }
  ```

* mixinList：用于分页加载，上拉加载时返回拼接数据及是否还有数据提示

  ```javascript
  methods: {
    @mixinList({needToast: false})
    async loadGoods(params = {}){
      let goodsRes = await this.$http(params);
      return goodsRes.respData.infos;
    },
    async hasMore() {
      let result = await this.loadgoods(params);
      if(result.state === 'nomore') this.tipText = '没有更多了';
      this.goods = result.list;
    }
  }
  // 上拉加载调用hasMore函数，goods数组就会得到所有拼接数据
  // loadGoods可传三个参数 params函数需要参数 ,startNum开始的页码,clearlist清空数组
  // mixinList可传一个参数 needToast 没有数据是否需要toast提示
  ```

* typeCheck：检测函数参数类型

  ```javascript
  methods: {
    @typeCheck('number')
    btnClick(index){ ... },
  }
  // btnClick函数的参数index不为number类型 则报错
  ```

* Buried：埋点处理方案，统计页面展现量和所有methods方法点击量，如果某方法不想设置埋点，可以return 'noBuried'即可

  ```javascript
  @Buried
  methods: { 
    btn1Click() {
      // 埋点为 btn1Click
    },
    btn2Click() {
      return 'noBuried'; // 无埋点
    },
  },
  created() {
    // 埋点为 view
  }
  // 统计页面展现量和所有methods方法点击量
  ```

## decorators.js

```javascript
/**
 * 避免并发调用，在上一次操作结果返回之前，不响应重复操作
 * 如：用户连续多次点击同一个提交按钮，希望只响应一次，而不是同时提交多份表单
 * 说明：
 *    同步函数由于js的单线程特性没有并发问题，无需使用此decorator
 *    异步时序，为便于区分操作结束时机，此decorator只支持修饰async函数
 */
export const noConcurrent = _noConcurrentTplt.bind(null,{mutexStore:'_noConCurrentLocks'});

/**
 * 避免并发调用修饰器模板
 * @param {Object} namespace 互斥函数间共享的一个全局变量，用于存储并发信息，多函数互斥时需提供；单函数自身免并发无需提供，以本地私有变量实现
 * @param {string} mutexStore 在namespace中占据一个变量名用于状态存储
 * @param {string} mutexId   互斥标识，具有相同标识的函数不会并发执行，缺省值：函数名
 * @param target
 * @param funcName
 * @param descriptor
 * @private
 */
function _noConcurrentTplt({namespace={}, mutexStore='_noConCurrentLocks', mutexId}, target, funcName, descriptor) {
  namespace[mutexStore] = namespace[mutexStore] || {};
  mutexId = mutexId || funcName;

  let oriFunc = descriptor.value;
  descriptor.value = function () {
    if (namespace[mutexStore][mutexId]) //上一次操作尚未结束，则无视本次调用
      return;

    namespace[mutexStore][mutexId] = true; //操作开始
    let res = oriFunc.apply(this, arguments);

    if (res instanceof Promise)
      res.then(()=> {
        namespace[mutexStore][mutexId] = false;
      }).catch((e)=> {
        namespace[mutexStore][mutexId] = false;
        console.error(funcName, e);
      }); //操作结束
    else {
      console.error('noConcurrent decorator shall be used with async function, yet got sync usage:', funcName);
      namespace[mutexStore][mutexId] = false;
    }

    return res;
  }
}

/**
 * 多函数互斥，具有相同互斥标识的函数不会并发执行
 * @param namespace 互斥函数间共享的一个全局变量，用于存储并发信息
 * @param mutexId   互斥标识，具有相同标识的函数不会并发执行
 * @return {*}
 */
export function makeMutex({namespace, mutexId}) {
  if (typeof namespace !== "object") {
    console.error('[makeNoConcurrent] bad parameters, namespace shall be a global object shared by all mutex funcs, got:', namespace);
    return function () {}
  }

  return _noConcurrentTplt.bind(null, {namespace, mutexStore:'_noConCurrentLocksNS', mutexId})
}

/**
 * 捕获async函数中的异常，并进行错误提示
 * 函数正常结束时应 return 'ok'，return其它文案时将toast指定文案，无返回值或产生异常时将toast默认文案
 * @param {string} defaultMsg  默认文案
 * @param {number, optional} duration 可选，toast持续时长
 */
export function withErrToast({defaultMsg, duration=2000}) {
  return function (target, funcName, descriptor) {
    let oriFunc = descriptor.value;
    descriptor.value = async function () {
      let errMsg = '';
      let res = '';
      try {
        res = await oriFunc.apply(this, arguments);
        if (res != 'ok')
          errMsg = typeof res === 'string' ? res : defaultMsg;
      } catch (e) {
        errMsg = defaultMsg;
        console.error('caught err with func:',funcName, e);
      }

      if (errMsg) {
        this.$toast({
          title: errMsg,
          type: 'fail',
          duration: duration,
        });
      }
      return res;
    }
  }
}

/**
 * 分页加载
 * @param {[Boolean]} [是否加载为空显示toast]
 * @return {[Function]} [decrotor]
 */
export function mixinList ({needToast = false}) {
  let oldList = [],
      pageNum = 1,
  /**
  * state [string]
  *   hasmore  [还有更多]
  *   nomore   [没有更多了]
  */
  state = 'hasmore',
  current = [];
  return function (target,name,descriptor) {
    const oldFunc  = descriptor.value,
          symbol   = Symbol('freeze');
    target[symbol] = false;
    /**
     * [description]
     * @param  {[Object]}   params={}       [请求参数]
     * @param  {[Number]}   startNum=null   [手动重置加载页数]
     * @param  {[Boolean]}  clearlist=false [是否清空数组]
     * @return {[Object]}   [{所有加载页数组集合,加载完成状态}]
     */
    descriptor.value = async function(params={},startNum=null,clearlist=false) {
      try {
        if (target[symbol]) return;
        // 函数执行前赋值操作
        target[symbol] = true;
        params.data.pageNum = pageNum;
        if (startNum !== null && typeof startNum === 'number') {
          params.data.pageNum = startNum;
          pageNum = startNum;
        }
        if (clearlist) oldList = [];
        // 释放函数，取回list
        let before = current;
        current = await oldFunc.call(this,params);
        // 函数执行结束赋值操作
        (state === 'hasmore' || clearlist) && oldList.push(...current);
        if ((current.length === 0) || (params.data.pageSize > current.length)) {
          needToast && this.$toast({title: '没有更多了',type: 'fail'});
          state = 'nomore';
        } else {
          state = 'hasmore';
          pageNum++;
        }
        target[symbol] = false;
        this.$apply();
        return { list : oldList,state };
      } catch(e) {
        console.error('fail code at: ' + e)
      }
    }
  }
}

/**
 * 检测工具
 */ 
const _toString = Object.prototype.toString;
// 检测是否为纯粹的对象
const _isPlainObject = function  (obj) {
  return _toString.call(obj) === '[object Object]'
}
// 检测是否为正则
const _isRegExp = function  (v) {
  return _toString.call(v) === '[object RegExp]'
}
/**
 * @description 检测函数
 *  用于检测类型action
 * @param {Array} checked 被检测数组
 * @param {Array} checker 检测数组
 * @return {Boolean} 是否通过检测
 */ 
const _check = function (checked,checker) {
  check:
  for(let i = 0; i < checked.length; i++) {
    if(/(any)/ig.test(checker[i]))
      continue check;
    if(_isPlainObject(checked[i]) && /(object)/ig.test(checker[i]))
      continue check;
    if(_isRegExp(checked[i]) && /(regexp)/ig.test(checker[i]))
      continue check;
    if(Array.isArray(checked[i]) && /(array)/ig.test(checker[i]))
      continue check;
    let type = typeof checked[i];
    let checkReg = new RegExp(type,'ig')
    if(!checkReg.test(checker[i])) {
      console.error(checked[i] + 'is not a ' + checker[i]);
      return false;
    }
  }
  return true;
}
/**
 * @description 检测类型
 *   1.用于校检函数参数的类型，如果类型错误，会打印错误并不再执行该函数；
 *   2.类型检测忽略大小写，如string和String都可以识别为字符串类型；
 *   3.增加any类型，表示任何类型均可检测通过；
 *   4.可检测多个类型，如 "number array",两者均可检测通过。正则检测忽略连接符 ；
 */
export function typeCheck() {
  const checker =  Array.prototype.slice.apply(arguments);
  return function (target, funcName, descriptor) {
    let oriFunc = descriptor.value;
    descriptor.value =  function () {
      let checked =  Array.prototype.slice.apply(arguments);
      let result = undefined;
      if(_check(checked,checker) ){
        result = oriFunc.call(this,...arguments);
      }
      return result; 
    }
  }
};

const errorLog = (text) => {
  console.error(text);
  return true;
}
/**
 * @description 全埋点 
 *  1.在所有methods方法中埋点为函数名
 *  2.在钩子函数中'beforeCreate','created','beforeMount','mounted','beforeUpdate','activated','deactivated'依次寻找这些钩子
 *    如果存在就会增加埋点 VIEW
 * 
 * 用法： 
 *   @Buried
 *   在单文件导出对象一级子对象下;
 *   如果某方法不想设置埋点 可以 return 'noBuried' 即可
 */
export function Buried(target, funcName, descriptor) {
  let oriMethods = Object.assign({},target.methods),
      oriTarget = Object.assign({},target);
  // methods方法中
  if(target.methods) {
    for(let name in target.methods) {
      target.methods[name] = function () {
        let result = oriMethods[name].call(this,...arguments);
        // 如果方法中返回 noBuried 则不添加埋点
        if(typeof result === 'string' && result.includes('noBuried')) {
          console.log(name + '方法设置不添加埋点');
        } else if(result instanceof Promise) {
          result.then(res => {
            if(typeof res === 'string' && res.includes('noBuried')) { console.log(name + '方法设置不添加埋点'); return; };
            console.log('添加埋点在methods方法中：' , name.toUpperCase ());
            this.$log(name);
          });
        }else{
          console.log('添加埋点在methods方法中：' , name.toUpperCase ());
          this.$log(name);
        };
        return result;
      }
    }
  }
  // 钩子函数中
  const hookFun = (hookName) => {
    target[hookName] = function() {
      let result =  oriTarget[hookName].call(this,...arguments);
      console.log('添加埋点，在钩子函数' + hookName + '中：', 'VIEW');
      this.$log('VIEW');
      return result;
    }
  }

  const LIFECYCLE_HOOKS = [
    'beforeCreate',
    'created',
    'beforeMount',
    'mounted',
    'beforeUpdate',
    'activated',
    'deactivated',
  ];

  for(let item of LIFECYCLE_HOOKS) {
    if (target[item]) return hookFun(item);
  }
}
```

