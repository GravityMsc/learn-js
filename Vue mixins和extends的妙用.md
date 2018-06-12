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

const mixin1 = {
    created(){
        console.log('mixin1 created');
    }
}

const mixin2 = {
    created() {
        console.log('mixin2 created');
    }
}

export default {
    extends: extend,
    mixins: [mixin1, mixin2],
    name: 'app',
    created(){
        console.log('created');
    }
}
```

控制台输出：

> extends created
>
> mixin1 created
>
> mixin2 created
>
> created

* 结论：优先调用mixins和extends继承的父类，extends触发的优先级更高，相当于是队列
* push（extend，mixin1，mixin2，本身的钩子函数）
* 经过测试，watch的值继承规则一样。

###继承methods

```javascript
const extend = {
    data (){
        return {
            name: 'extend name'
        }
    }
}
const mixin1 = {
    data (){
        return {
            name: 'mixin1 name'
        }
    }
}
const mixin2 = {
    data (){
        return {
            name: 'mixin2 name'
        }
    }
}
// name = 'name'
export default {
    mixins: [mixin1, mixin2],
    extends: extend,
    name: 'app',
    data(){
        return {
            name: 'name'
        }
    }
}
```

```javascript
// 只写出子类，name = 'mixin2 name'，extends优先级高会被mixins覆盖
export default {
  mixins: [mixin1, mixin2],
  extends: extend,
  name: 'app'
}
```

```javascript
// 只写出子类，name = 'mixin1 name'，mixins后面继承会覆盖前面的
export default {
  mixins: [mixin2, mixin1],
  extends: extend,
  name: 'app'
}
```

* 结论：子类再次声明，data中的变量都会被重写，以子类为准。
* 如果子类不声明，data中的变量将会以最后继承的父类为准。
* 经过测试，props中属性、methods中的方法和computed的值继承规则一样。

### 写在后面：

> 关于mixins和extends你可以理解为mvc的c（controller）这一层。可见通用的成员变量（包括属性和方法）抽象成为一个父类，提供给子类继承，这样就可以让子类拥有一些通用成员变量，然而子类也可以重写父类的成员变量。这样的编程思想就很面向对象，也就是继承性。