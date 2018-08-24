##前言

以一个记账项目举例，常见的场景有首页、记到账页面、选择合同、新建合同、选择客户、新建客户这些页面。

![记账项目的例子](https://user-gold-cdn.xitu.io/2018/8/1/164f31da98b56dfa?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

在这些页面中，很显然，用户的浏览行为应该是逐渐深入的，通俗得讲就是浏览页面在不断前进。

而且这些页面之间还是有互动性存在的，两种互动行为：

* 一、用户前进时，总是进入新的页面。（比如在合同列表页反复加载多次列表之后，进入其中一个合同详情，再返回时，应该仍停留之前里列表页同一个位置，而不是重新刷新列表页。）
* 二、用户后退时，需要能保留前一页数据并继续操作。（比如，记到账时需要选择合同，选择合同时可以新建合同，新建合同时填了一堆数据可以去选择客户，在选择客户时又去创建了客户，那么这一堆操作下来应该能够做到：**创建完客户后继续新建合同，建完合同后继续记该合同的到账**）

![演示](https://user-gold-cdn.xitu.io/2018/8/1/164f31de00795275?imageslim)

上图是demo项目中的真实效果，目前常见的vue开发方案里，一般都会引入vuex或localStorage，在各个页面不断的存储和调用页面内的数据，我觉得，这很不科学很不优雅。

## keep-alive什么问题

vue支持keep-alive组件，如果启用，页面内的所有数据都会被保留，所以，上文的互动行为二：**后退时保留前一页数据继续操作**没有问题。

问题出在互动行为一：**用户前进时总是进入新页面**，然而一旦缓存，你就没法总是进新页面了，你总是进入缓存页，这就很让人头疼了。

官方提供了**include**和**exclude**特性，说你可以决定哪些页面使用缓存哪些页面不用缓存。[链接](https://cn.vuejs.org/v2/api/#keep-alive)

然而问题又回到了原点，并没有解决我们**酌情决定是否使用已缓存的缓存**这一需求。

所以很多人想到了一个办法**在离开页面是销毁这个页面**是不是就可以了，然而并不能，这里出现了bug，组件销毁了缓存还在：

![vue组件缓存](https://user-gold-cdn.xitu.io/2018/8/1/164f31e28abbbb3d?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

于是，就有人提出**希望keep-alive能增加可以动态删除已缓存组件的功能**，[issue](https://github.com/vuejs/vue/issues/6509)

这是个老话题，之前一直没有进展，核心原因就在于keep-alive不能正确处理已销毁的组件。

## 尝试解决这个问题

如果能实现**动态使用缓存**这一功能，那么所有问题也就迎刃而解。

最初，我研究keep-alive的代码，发现了这么一段代码：

![vue的keep-alive实现](https://user-gold-cdn.xitu.io/2018/8/1/164f31e53d834315?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

于是，我想，如果在此处判断**如果组件已被销毁则不使用缓存**，是不是就解决这个问题了，于是我提交了一个[PR](https://github.com/vuejs/vue/pull/7151)：

![当组件被销毁时不使用缓存](https://user-gold-cdn.xitu.io/2018/8/1/164f31e7b4c84517?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

不过这个PR迟迟没有通过，我就放弃了。

## 暴力解决这个问题

我继续研究有没有其他方案，然后我在打印组件变量的时候，发现了这个眼熟的字段：

![组件变量](https://user-gold-cdn.xitu.io/2018/8/1/164f31e9cf277d1f?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

这不就是keep-alive的组件嘛，我赶忙点开看看，发现了更眼熟的东东：

![keep-alive的细节](https://user-gold-cdn.xitu.io/2018/8/1/164f31ec5c36f251?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

于是，这事儿就变得简单了，直接按图索骥，咱在销毁组件之前，寻找路由组件所在父级的keep-alive组件，操控其中的cache列表，强行删除其中的缓存，问题也就迎刃而解，是不是很直接很暴力。

## 结论

keep-alive默认不支持动态销毁已缓存的组件，所以此处给出解决方案是通过直接操作keep-alive组件里的cache列表，暴力移除缓存：

```javascript
//使用Vue.mixin的方法拦截了路由离开事件，并在该拦截方法中实现了销毁页面缓存的功能。
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

## 后语

本文主要围绕如何动态删除keep-alive缓存这一问题进行探索，其他关于**如何设定页面层级、如何在前后页之间进行数据传递**等问题，敬请期待《Vue全站缓存之vue-router-then：前后页数据传递》。