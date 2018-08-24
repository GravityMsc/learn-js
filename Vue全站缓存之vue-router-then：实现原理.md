## 前言

就效果而言，我很满意**v-model-link**指令带来的开发效率的提升，不过vue-router-then这个插件本身的代码实现得有点粗暴，总觉得自己对vue的理解还是比较肤浅，有时候看别人家的文章总有不明觉厉的感叹，然而正如我在知乎专栏在博客的自我介绍里所说，高楼大厦平地起，咱也想加上一块砖，自己这三板斧该献丑还是献丑，抛砖引玉也行的。

## 活着才有DPS

从第一篇文章[Vue全站缓存之keep-alive：动态移除缓存](https://juejin.im/post/5b610da4e51d45195c07720d)开始，我们就一直在强调如何去实现页面缓存，vue-router-then正是建立在页面缓存成功的基础之上的功能，如果你不能正确的使用页面缓存功能，或者说页面缓存被销毁，vue-router-then也就没有意义了。

## 实现原理

这个插件的原理说开了实现得非常粗暴，说白了就是一句话：在路由跳转事件里给新页面组件绑事件。

## 返回一个promise

在this.$router系列方法的基础之上，包装一个promise，并将resolve方法给存储起来。

```javascript
const routerThen = {
    '$router': null,
    resolve: null,
    // 跳到指定页面，并返回promise
    reuquest: function(requestType='push', location, onComplete=null, onAbort=null){
        if(!location || location == ''){
            throw new Error('location is missing');
        }
        return new Promise((resolve, reject) => {
            if(this.$router){
                console.log('this.$router', this.$router);
                this.resolve = resolve;
                switch(requestType)
                {
                    case 'push':
                        this.$router.push(location, onComplete, onAbort);
                        break;
                    case 'replace':
                        this.$router.replace(location, onComplete, onAbort);
                        break;
                    case 'go':
                        this.$router.go(location);
                        break;
                    default:
                        reject('requestType error:' + requestType);
                        break;
                }
            } else {
                reject('$router missing');
            }
        }).catch(error => {
            this.resolve = null;
            throw new Error(error);
        });
    },
}
```

## 在路由事件里夹点私货

上文里，将resolve存好后，页面就应该开始跳转，此时可以捕捉路由事件，在新页面载入后，将新页面对象vm回调给promise。

```javascript
Vue.mixin({
    // 在路由跳转到下一个页面之前，为下一个页面注册回调事件
    beforeRouteEnter: function(to, from, next){
        if(routerThen.resolve){
            next(vm => {
                routerThen.resolve(vm);
                routerThen.resolve = null;
            });
        } else {
            next();
        }
    },
    beforeRouteUpdate: function(to, from, next){
        if(routerThen.resolve){
            routerThen.resolve(this);
            routerThen.resolve = null;
        }
        next();
    },
});
```

## 拿到页面对象啥都好办了

比如，modelLink方法，其实就是拿到了vm对象给它塞了个input事件。

```javascript
modelLink: function(link, el = null){
    return this.push(link).then(vm => {
        vm.$once('input', value => {
            if(typeof el == 'function'){
                el(value);
            } else if (typeof el == 'object'){
                if(el.$emit){
                    el.$emit('input', value);
                } else if(el.tagName){
                    el.value = value;
                    const e = document.createElement('HTMLEvents');
                    // e.initEvent(binding.modifiers.lazy? 'change' : 'input', true, true);
                    e.initEvent('input', true, true);
                    el.dispatchEvent(e);
                }
            }
        });
        return vm;
    })
},
```

## v-model-link只是一个语法糖

我很喜欢语法糖这个概念，复杂的事情简单化。

```javascript
clickElFun: function(event){
    let link = this.getAttribute('model-link');
    if(link){
        console.log(this);
        return routerThen.modelLink(link, this.vnode && this.vnode.componentInstance ?this.vnode.componentInstance: this);
    }
    return Promise.resolve();
},

    Vue.directive('model-link', function(el, binding, vnode){
        el.binding = binding;
        el.vnode = vnode;
        el.setAttribute('model-link', binding.value);
        el.removeEventListener('click', routerThen.clickElFun);
        el.addEventListener('click', routerThen.clickElFun);
    });
```

## 后语

还是那句话，这个代码有点粗暴，作抛砖引玉，供大家参考。

我很喜欢全站缓存这个理念，在目前vue社区里类似的文章很少看到，希望能有更多朋友参与进来，挖掘其中的亮点。