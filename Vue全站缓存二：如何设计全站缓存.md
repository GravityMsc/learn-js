> 在前文[Vue全站缓存之keep-alive：动态移除缓存](https://juejin.im/post/5b610da4e51d45195c07720d)中，我们实现了**在路由离开时动态移除缓存**这一功能，以此为基础，vue全站使用缓存成为可能。

## 前言

从早期粗暴得将css、js资源设定浏览器本地缓存，到后来小图标合并成大图节省请求资源，还有动态请求304状态判断，然后ajax开启web2.0时代，pjax大放光彩，到如今vue.js等前端框架的繁荣盛世，所有的这一系列发展，我认为，**提速**是一个核心驱动力。

### keep-alive

在vue里，支持keep-alive特性，通过keep-alive，不再销毁旧组件为新组件让路，而是缓存起来以待将来复用，当复用缓存组件时，如果数据没有变化甚至可以直接原样回复。[链接](https://cn.vuejs.org/v2/api/#keep-alive)

### keep-alive和vue-router

将router-view放置到keep-alive中，即可粗暴的实现所有路由页的缓存功能。

```vue
<!-- App.vue -->
<keep-alive><router-view class="transit-view"></router-view></keep-alive>
```

## 为什么要使用缓存

最常见的一个场景是**新建订单时选择地址**，新建订单是一个路由页面，去选择用户现有的地址又是一个路由页面，所以理所当然的，我们希望用户选择完地址回到订单页面的时候，订单里的其他数据比如选择的优惠券啊收件日期啊都能继续保持。

![使用缓存的原因](https://user-gold-cdn.xitu.io/2018/8/1/164f32499ce04932?imageslim)

在大量类似的场景里，不断的出现数据保留和复用的需求，所以vuex出现了，通过第三方的一个公共组件来保存和中转数据，这是一个解决方案，而是还是主流的解决方案哦。

然而换个角度，如果订单页在选择地址的时候被缓存了，回到订单页后直接复用前面的订单组件，其他数据都保留此时只要更新下地址数据，让所有的代码逻辑都集中在订单组件之中，这样的开发体验是不是会更直观更友好？

这是见仁见智的思路，各有想法，不好说谁好谁坏，我们就先继续讨论缓存组件的方案吧。

## 出现了一点小问题

如果所有的路由页都被缓存了，那么当你不想使用缓存的时候怎么办？比如又建了一个新订单，进入不同的文章的编辑组件，比如进入不同的用户中心，缓存固然提了速，有时我们也会不想要缓存功能，特别是一些表单场景，我们既希望填写一半进入下一页面时能保留填写的数据，我们又希望新进入的表单是一个全新的表单页。

## 鱼和熊掌可以兼得

**我们既希望填写一半进入下一页面时能保留填写的数据，我们又希望新进入的表单是一个全新的表单页**，换句话说，**回到上一个页面时使用缓存，进入下一个页面时不使用缓存**，再换句话说，**所有页面都用缓存，只在后退（回到上一页）时移除当前页缓存，这样下一次前进（进入当前页）时因为没有缓存就自然使用全新页面**，也就是说，只要实现**后退（回到上一页）是移除当前页缓存**这个功能，就可以了。

## 在路由中定义位置

这是一种缓存复用的思路，为了实现**后退（回到上一页）时移除当前页缓存**，因为想要实现动态确定用户的前进后退行为比较麻烦，所以，我们有个傻瓜式的方案：预测使用场景约定各路由页面的层级关系。

比如，在routes定义里，我们可以这么定义各路由页：

```javascript
// 仅供参考，此处缺少路由组件定义
// router/index.js
routes: [
        {   path: '/', redirect:'/yingshou', },
        {   path: '/yingshou',                meta:{rank:1.5,isShowFooter:true},          },
        {   path: '/contract_list',           meta:{rank:1.5,isShowFooter:true},          },
        {   path: '/customer',                meta:{rank:1.5,isShowFooter:true},          },
        {   path: '/wode',                    meta:{rank:1.5,isShowFooter:true},          },
        {   path: '/yingfu',                  meta:{rank:1.5,isShowFooter:true},          },
        {   path: '/yingfu/pact_list',        meta:{rank:2.5},                            },
        {   path: '/yingfu/pact_detail',      meta:{rank:3.5},                            },
        {   path: '/yingfu/expend_view',      meta:{rank:4.5},                            },
        {   path: '/yingfu/jizhichu',         meta:{rank:5.5},                            },
        {   path: '/yingfu/select_pact',      meta:{rank:6.5},                            },
        {   path: '/yingfu/jiyingfu',         meta:{rank:7.5},                            },
    ]
```

核心的思路是，在定义路由时，在meta中定义一个rank字段来声明该路由的页面优先级，比如1.5标识第1层如首页，2.5标识第2层如商品列表页，3.5标识第3层商品详情页，以此类推。

如果大家同在一层，也可以通过1.4和1.5这样小数位来阅读先后层级。

总之，我们期望的是，从第1层进入第2层是前进，从第3层回到第2层是后退。

## 在路由跳转里动态判断移除缓存

使用Vue.mixin的方法拦截了路由离开事件，并在该拦截方法中实现**后退时销毁页面缓存**。

```javascript
// main.js
Vue.mixin({
    beforeRouteLeave:function(to, from, next){
        if (from && from.meta.rank && to.meta.rank && from.meta.rank>to.meta.rank)
        {//此处判断是如果返回上一层，你可以根据自己的业务更改此处的判断逻辑，酌情决定是否摧毁本层缓存。
            if (this.$vnode && this.$vnode.data.keepAlive)
            {
                if (this.$vnode.parent && this.$vnode.parent.componentInstance && this.$vnode.parent.componentInstance.cache)
                {
                    if (this.$vnode.componentOptions)
                    {
                        var key = this.$vnode.key == null
                                    ? this.$vnode.componentOptions.Ctor.cid + (this.$vnode.componentOptions.tag ? `::${this.$vnode.componentOptions.tag}` : '')
                                    : this.$vnode.key;
                        var cache = this.$vnode.parent.componentInstance.cache;
                        var keys  = this.$vnode.parent.componentInstance.keys;
                        if (cache[key])
                        {
                            if (keys.length) {
                                var index = keys.indexOf(key);
                                if (index > -1) {
                                    keys.splice(index, 1);
                                }
                            }
                            delete cache[key];
                        }
                    }
                }
            }
            this.$destroy();
        }
        next();
    },
});
```

## 这样就行了吗

不行，这样只解决了**动态移除缓存**的情况，让用户既可以进入新页面，也可以回到旧页面。前者进新页面问题不大，后者回到旧页面这件事在还没讨论过呢。

## 缓存是说用就用的吗

即使是路由页面复用了缓存，也只是复用了缓存的组件和数据，在实际场景中，从列表A进入详情B再进入列表C，请问列表C和列表A是同一个路由页，但它们的数据会一样吗？应该一样吗？

## 所以计算式缓存了也要更新数据？

看起来，我们得到了一个新结论，缓存页的数据也不可靠啊。

## 应该有办法的

缓存的组件被复用时会触发activated事件，非缓存组件则会在创建时触发created、mounted等一大堆事件，而同一个页面列表A进列表B，因为url参数不同，则会触发beforeRouteUpdate事件。[链接1](https://cn.vuejs.org/v2/api/#keep-alive) [链接2](https://cn.vuejs.org/v2/guide/instance.html#%E5%AE%9E%E4%BE%8B%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%E9%92%A9%E5%AD%90) [链接3](https://router.vuejs.org/zh/guide/advanced/navigation-guards.html)

看起来，我们能够通过捕捉这些事件来干点啥。

## 不对不对

第一直觉是，我们在捕捉到页面载入的事件后去拉取数据更新页面。仔细一想，我们上面罗里吧嗦废了老半天劲是为了进入缓存页面不再看那loading条的啊。

## 笨办法

这里提供了一个笨办法，事件该捕捉咱还是要捕捉，只是这是否去做拉取数据的动作，咱可以有待商榷。

* 在所有的路由页统一注册一个方法，就叫pageenter吧，不管从啥情况进来的，咱都去触发这个方法。

  ```javascript
  // list.vue
      data(){
          return {
              params        : null,
          }
      },
      methods:{
          pageenter:function(){
              this.params = {
                          'uuid'                 : this.$route.query.uuid   ,
                      };
          },
      }
  ```

* 如果直接watch这个params，这事还是没法玩，这里可以用一个笨办法，咱将它转化成字符串再watch，捕捉到字符串变化再去重新拉取数据，如果字符串没有变化，则啥也不做。

  ```javascript
  // list.vue
      computed:{
          paramsToString(){
              return JSON.stringify(this.params);
          },
      },
      watch:{
          paramsToString:function(){
              this.loadContent();
          },
      },
      methods:{
          loadContent:function(){
              //此处拉取数据
              //this.$http.get(....)
          },
      }
  ```

* 在main.js里，可以用Vue.mixin拦截上文提到的三种事件，来触发pageenter方法。

  ```javascript
  //main.js
  Vue.mixin({
      /*初始化组件时，触发pageenter方法*/
      mounted:function(){
         if (this.pageenter)
         {
             this.pageenter();
         }
      },
      // /*从其他组件返回激活当前组件时*/
      activated:function(){
         if (this.pageenter)
         {
             this.pageenter();
         }
      },
      /*在同一组件中，切换路由（参数变化）时*/
      beforeRouteUpdate:function(to, from, next){
          if (this.pageenter)
          {
              this.$nextTick(()=>{
                  this.pageenter();
              });
          }
          next();
      },
  });
  ```

  # 后语

  至此，全站缓存的框架算是基本搭好了。