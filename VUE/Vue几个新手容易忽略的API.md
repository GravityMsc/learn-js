## 一：实现子组件与父组件双向绑定的【sync】修饰符

一般来说，我们实现父子组件值的传递通常使用的是【props】和自定义事件【$emit】。父组件通过【props】将值传给子组件，子组件通过【\$emit】将值传给父组件，父组件通过【\$on】事件获取子组件传过来的值，如果说想要实现子组件修改父组件传过来的值，最容易的就是这种方法了：

```vue
// 父组件向子组件传值
<template>
	<div>
    	<child-com :value="text"></child-com>
    </div>
</template>

<script>
    export default{
        data(){
            return{
                text: "父组件的值",
            }
        }
    }
</script>
```

```vue
// 子组件向父组件传值
<template>
	<div @cilck="post">
        
    </div>
</template>

<script>
    export default{
        methods:{
            post(){
                this.$emit('getChildValue', "子组件的值");
            }
        }
    }
</script>
```

此时父组件可以通过【\$on】获取子组件的值：

```vue
<template>
	<div>
        <child-com @getChildValue = "getValue"></child-com>
    </div>
</template>

<script>
    export default{
        methods:{
            getValue(child_value){
                this.text = child.value;
            }
        }
    }
</script>
```

这样，就可以实现子组件修改父组件的值。

不过，这种方法有一个弊端——子组件修改父组件的值需要一个传递过程，或者说，两个值并不是同步的。

熟悉Vue1.0的朋友应该知道一个叫【.sync】的修饰符，它可以实现父子组件的双向绑定，不过在Vue2.0倍移除了，知道2.3.0版本发布后才重新回归，所以一些和我一样从2.0开始使用Vue的朋友很有可能不清楚，事实上，【.sync】可以很轻松的实现子组件同步修改父组件的值：

```vue
// 父组件
<template>
	<div>
        <child-com :value.sync="text"></child-com>
    </div>
</template>
<script>
    export default{
        data(){
            return {
                text: "父组件的值",
            }
        }
    }
</script>

==========================================================================================
// 子组件修改父组件的值
<template>
	<div @click="post">
        
    </div>
</template>

<script>
    export default{
        methods:{
            post(){
                this.$emit('update:value', "子组件的值");
            }
        }
    }
</script>
```

我们可以看到，对于子组件来说，仅仅是自定义事件名做了一点改变，但是就代码底层逻辑来说，子组件和父组件真正实现了同步的双向绑定。

当然，正如文档所说：

> .sync修饰符很方便，但也会导致问题，因为破坏了单向数据流。由于子组件改变props的代码和普通的状态改动代码毫无区别，当光看子组件的代码时，你完全不知道它何时悄悄地改变了父组件的状态。这在debug复杂结构的应用时会带来很高的维护成本。

###二：自定义指令：“directives”

关于自定义指令文档其实介绍的比较详细了，而且还举了一个非常详细的例子：[自定义指令](https://cn.vuejs.org/v2/guide/custom-directive.html)

自定义指令其实就是Vue为我们提供直接操作DOM的一系列方法，虽然大部分开发时间都会面向数据，但说不准什么时候确实需要操作DOM本身。

就我而言，自定义指令最大的用处就是可以应用一些第三方的代码插入到Vue项目中，比如有一个操作dom的函数：

```vue
//当然，真实情况第三方的代码要复杂的多
function changeColor(dom){
	dom.style.backgroundColor = "red";
}
```

我们可以注册一个全局的指令来为需要执行changeColor方法的DOM提供指令：

```vue
Vue.directives('color', {
	bind: function(el){
			changeColor(el);
		}
});
```

这样，如果需要这个DOM改变颜色的话，只需要这样既可：

```vue
<div v-color>
    改变颜色
</div>
```

当日常开发遇到跟DOM有关的问题却一筹莫展时们可以想想自定义指令是否有功能可以解决问题。

## 三：inheritAttrs和attrs

前面我已经提到过了，父组件通过props可以向子组件传值，但在日常的开发中，还有一种情况很常见，就是父组件给子组件传值，这个值还要从子组件传给它的子组件，所以，我们可能会看到这样的代码：

```vue
// 父组件
<div>
    <child :text="text"></child>
</div>

// 子组件
<div>
    <my-child :text="text"></my-child>
</div>

// 子组件的子组件
<div>
    <div>
        {{text}}
    </div>
</div>
```

这样做是非常麻烦而且 不易于维护的，通常情况下，我们可以使用vuex来解决。不过，不复杂的项目中如果仅仅为这一个问题就引入vuex实际上是没有必要的，Vue提供了【inheritAttrs】和【attrs】两个功能来解决这样的问题：

```vue
// 父组件
<template>
	<div>
        <child :text="text" :count="count"></child>
    </div>
</template>

<script>
    export default{
        data(){
            return {
                text: "父组件的值",
                count: 123456,
            }
        }
    }
</script>
```

```vue
// 子组件
<template>
	<div>
        {{text}}
    </div>
</template>

<script>
    export default{
        props: ["text"]
    }
</script>
```

注意，这里父组件的count属性仅仅挂在子组件上，并没有使用。此时我们打开浏览器，可以看到子组件的DOM上显式的展示了count="123456"。

![子组件的DOM](https://user-gold-cdn.xitu.io/2018/4/23/162ee4df3a1f2d28?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

此时，我们可以通过设置inheritAttrs: false来取消这种默认行为：

```javascript
data(){
    return {
        .....
    }
}

inheritAttrs: false,
mounted(){
    console.log(this.$attrs);	// {count: 123456}
}
```

这时再看DOM上就没有count属性了。然后，我还打印了this.$attrs的值，值为一个包含着count键值对的Object。

也就是说，父组件没有props的属性值会被保存在一个名为$attrs的属性中供子组件使用，然而这并没有解决开头子组件的子组件获取值的问题。别急，我们只需要在子组件上加个东西就可以了：

```vue
<template>
	<div class="child">
        <my-child v-bind="$attrs"></my-child>
    </div>
</template>
```

这样，子组件的子组件也可以获取这个值了。

## 四：混入——mixins

其实这个功能有些类似于es6中的Object.assign()方法。根据一定的规则合并两个配置，具体的混入策略可以看官方文档：[mixins混入策略](https://cn.vuejs.org/v2/guide/mixins.html)

混入最大的用处是把一些常用的data或者methods等抽出来，比如在我的项目中有许多个模态框，而关闭模态框的代码逻辑是一模一样的，为此我没有必要在多个组件中重复把关闭模态框的逻辑写入methods中，只需要在外面定义一个mixins，在需要的组件中通过：mixins: [myMin]写入即可。

```javascript
var mixin = {
    methods: {
        close: function (){
            this.showModal = false;	//关闭模态框
        },
    }
}

var vm = new Vue({
    mixins: [mixin],
    .....
})
```

## 五：provide / inject

provide/inject方法要比inheritAttrs/attrs更适合用来做父组件给子组件或后代组件传值，先发一个文档的链接：[provide/inject](https://cn.vuejs.org/v2/api/#provide-inject)

```vue
//父组件使用provide
<template>  
	<div class="parent">      
        <child-component></child-component>    
    </div> 
</template>

<script>
    export default { 
        ......  
        provide: {    
        	parent: "父组件的值"  
    	},  
        components:{    
            child-component,  
        },  
           ......
</script>

//此时可以在子组件通过这种方式获取父组件中“parent”的值：
//子组件中
export default {  
	mounted(){      
		console.log(this.parent); //"父组件的值"  
	},  
	inject: ['parent'],
}
```

