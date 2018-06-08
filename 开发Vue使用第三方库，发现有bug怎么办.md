## 写在前面

> 本文只针对使用vue技术栈，进行讨论。

## 正文

使用vue技术栈开发，难免会使用第三方库，这大大提高了我们开发的效率。然而，如果第三方库有bug怎么办？

### 第一步

阅读第三方库源码，怎么阅读这里就不要展开。阅读源码，找到问题所在。

### 第二步

找到了问题所在，怎么解决，给作者提bug？

这个想法不错。但是，我们有其他方法来解决。

既然代码存在bug，我们可以重写有bug的代码。

没有错，就是重写代码。vue提供了extends和mixins重写代码的方式。关于extends和mixins可以阅读之前的一篇文章：[vue mixins和extends的妙用](https://juejin.im/post/5a38d222f265da4312810a76)。

举个例子：使用mint-ui Swipe组件过程中发现存在的bug

```javascript
import {
    Swipe
} from 'mint-ui';
export default {
    components: {
        imageSwipe: {
            extends: Swipe,
            watch: {
                defaultIndex (val) {
                    this.reInitPages();
                }
            }
        }
    }
}
```

上面代码的做法就是，定义一个imageSwipe，继承mint-ui的Swipe组件，加一个watcher。

这时候使用imageSwipe时，props、event和slots与mint-ui的Swipe组件是一样的。

```vue
<image-swipe></image-swipe>
```

## 写在后面

假设，组件中嵌套为暴露出来的组件，这时没有办法从组件引入，可以在原有组件的基础上重写、继承之后，开发出新功能，不一定是只用来修复bug。

```javascript
import {
  Table
} from 'element-ui'

export default {
  extends: Table.components.TableHeader
}
```

以上是重写Table中的tableHeader组件，tableHeader组件无法从element-ui中获取，通过Table.components.TableHeader去获取。