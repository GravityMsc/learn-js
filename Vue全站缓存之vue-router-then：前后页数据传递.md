## 前言

在前文[Vue全站缓存之keep-alive：动态移除缓存](https://juejin.im/post/5b610da4e51d45195c07720d)和[Vue全站缓存二：如何设计全站缓存](https://juejin.im/post/5b610f32e51d4519115d3e66)中，我们实现了全站缓存的基础框架，在页面之间后退或同层页面之间跳转时可以复用缓存，减少了请求频率，提升了用户体验，最赞的是，开发逻辑理论上会简单直观很多。

## 出现了一个新需求

在父子组件之间，数据可以通过Prop和$emit事件来互相传递数据。

在本系列里，我们研究的主题是前后页的组件缓存与复用，因为使用了keep-alive的缓存特性，虽然前一页在界面上已经被移除，在系统里，前后页组件对象仍然是同时存在的。

于是，一个新的需求自然而然的出现了，就是**如何在前后页组件之间传递数据**。

## store模式和vuex

前文也曾说过，为了在不同页面之间传递或共用数据，目前主流的方案是使用store模式或引入vuex，作为前后页面之外的公共组件，所有页面都可以直接对其存取数据。[链接](https://cn.vuejs.org/v2/guide/state-management.html#%E7%AE%80%E5%8D%95%E7%8A%B6%E6%80%81%E7%AE%A1%E7%90%86%E8%B5%B7%E6%AD%A5%E4%BD%BF%E7%94%A8)

![store模式](https://user-gold-cdn.xitu.io/2018/8/1/164f33a7b23d3ded?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

这是一个很好的解决方案，可以兼容很多场景，而且很值得继续深挖这个特性哦。

## 懒是人类进步的源泉

就我个人而言，我是很极端的懒，vuex的确能解决前后页数据交流的问题，然而许多时候我仅仅只是打开一个页面选择一个客户回来而已。

如果能写成：

```html
<input v-model='searchText'>
```

我可不想写成：

```html
<input
  v-bind:value="searchText"
  v-on:input="searchText = $event.target.value"
>
```

所以，还是商城的例子，如果在订单页面需要打开一个新页面去选择一个地址，我可不想从vuex那里绕一大圈写一堆 State、Mutation、Action，我希望写成下面这样：

```html
<!-- buySomething.vue -->
<inputSelectedAddress v-model="item.addressID" v-model-link="'/select_address'"  placeholder="选择地址" /> 

<!-- selectAddress.vue -->
<template>
    <ul>
        <li @click="selectOneAddress(1)">北京路南广场东门101</li>
        <li @click="selectOneAddress(2)">上海城下水道左边202</li>
    </ul>
</template>
<script>
    methods:{
        selectOneAddress:function(addressID){
            this.$emit('input',addressID);
            this.$router.go(-1);
        }
    }
</script>
```

打开一个地址选择页，选择好地址后，直接将地址id传回来，甚至直接和v-model兼容起来，然后该干啥干啥，这样的场景想想就很赞啊。

## v-model-link

在上文的示例代码里出现了v-model-link这个自定义指令，这个指令来自插件vue-router-then的一个特性。

[链接](https://github.com/wanyaxing/vue-router-then)

v-model-link的基本用法：

* 在绑定v-model的元素上使用v-mode指令指向一个新页面
* 在新页面里用户互动之后，使用$emit触发一个input事件将数据传递回来即可更新v-model所对应的值。

> 请注意此处的场景差异：$emit本来是用来进行父子组件的事件传递的，在此处被挪用到了前后页组件之间传递事件。

原理简述：

* v-model-link的值是一个url地址，在点击该指令所在的DOM元素后，会跳转到该url页面，并在新的页面里监听input事件反馈给指令所在的DOM元素或组件。
* 因为v-model就是一个input事件的语法糖，所以其对应的值会受input事件影响。
* 因为缓存的存在，前一页组件没有被销毁，这才是vue-router-then这个插件成功使用的基础。
  * 所以，请注意正确设定各路由页面的层级关系，如果离开页面时组件被销毁了，可就没法玩了。

## vue-router-then

[vue-router-then](https://github.com/wanyaxing/vue-router-then)的核心本质就是实现了在当前页操作下一页组件的功能，v-model-link只是该功能基础上的一个语法糖指令，当然也是最重要最常用的特性指令。

### 如何安装

```bash
npm install vue-router-then --save;
```

```javascript
import Vue from 'vue'
import router from './router'

import routerThen from 'vue-router-then';
routerThen.initRouter(router)
Vue.use(routerThen)
```

### 使用方法一：在当前页面操作下一页面的组件对象

该插件名为vue-router-then，顾名思义，其最大的特性是给router的方法(push, repalce, go)返回了一个promise对象，并使用新页面的vm作为参数在then方法里供处理。

```javascript
methods:{
    clickSomeOne:function(){
        this.$routerThen.push('/hello_world').then(vm=>{
            // 此处的 vm 即为新页面hello_world的组件对象
            console.log(vm);
        });
    },
}
```

### 使用方法二：使用$routeThen.modelLink在当前页获得下一页面组件上报的input事件中的值

$routerThen中还新增了一个自定义方法modelLink，用来处理下一页面上报的值，该方法同样可以操作下一页面组件对象。

```javascript

methods:{
    jumpToNextPage:function(value){
        this.$routerThen.modelLink('/select_price',value=>{
            // 此处获得select_price页面$emit出来的 input 事件中的值
            console.log(value);
        }).then(vm=>{
            // 此处的 vm 即为新页面select_price的组件对象
            console.log(vm);
        });
    },
}
```

### 使用方法三：配合v-model使用自定义指令v-model-link

```html
<inputCustomer v-model="item.customerID" v-model-link="'/select_customer'" />
```

## 再举个例子

在我们的记应收这狂记账APP中，有一个选择客户的页面，该页面支持搜索客户，如果搜索不到客户，则应该支持以关键字去建立一个客户并选中新客户。

！[记账APP](https://user-gold-cdn.xitu.io/2018/8/1/164f33b5778c1e0c?imageslim)

也就是说涉及**编辑合同->选择客户->新建客户**这三个页面，其中核心代码如下：

编辑合同：

```html
<!-- InitEditContractDetail.vue -->
<inputCustomer v-model="item.customerID" v-model-link="'/customer/select_customer'"/>
```

选择客户：

```html
<!-- InitSelectCustomer.vue -->
<template>
    <div>
        <input type="text" v-model="keyword" />
        <a @click="newCustomerJumpLink">新增</a>
    </div>
</template>
<script>
    methods:{
        newCustomerJumpLink(){
            this.$routerThen.modelLink('/customer/edit',value=>{
                this.newCustomerAdded(value)
            }).then(vm=>{
                if (this.keyword)
                {
                    vm.$set(vm.item,'name',this.keyword);
                }
            });
        },
        newCustomerAdded:function(customerID){
            this.$emit('input',customerID);
            this.$router.go(-1);
        },
    }
</script>
```

新增客户：

```html
<!-- InitCustomerEdit.vue -->
<template>
    <div>
        <div>客户名称</div>
        <input type="text" v-model="item.name" />
        <div class="btn_submit" @click="submit">提交</div>
    </div>
</template>

<script>
    methods:{
        submit:function(){
            this.$emit('input',customerID);
            this.$router.go(-1);
        }
    }
</script>
```

## 后语

到此时，应付大部分的项目场景应该没问题了。

后续还会发几篇相关的文章，研究vue-router-then的原理，研究全站缓存这种极端情况下用户的数据又会发生什么奇怪的变化呢，又或者还能玩出哪些更有意思的花样呢。



