### 写在前面

> Vue提供了mixins、extends配置项，最近使用中发现很好用。

### 混合mixins和继承extends

![Vue中的mixins](https://user-gold-cdn.xitu.io/2017/12/19/1606e1f691ef3537?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

> 看看官方文档怎么写的，其实两个都可以理解为继承，mixins接收对象数组（可理解为多继承），extends接收的是对象或函数（可理解为单继承）。

### 继承钩子函数

```javascript
const extend = {
    created(){
        console.log('extends created');
    }
}

const 
```

