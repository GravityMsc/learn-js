声明：本文章中所有源码取自Version：2.5.13的dev分支上的Vue。

我们目前的技术栈主要采用Vue，而工作中我们碰到了一种情况是当传入某些组件内的props被改变时我们需要重置整个组件的生命周期（比如更改iView中datepicker的type，好消息是目前该组件已经可以不用再使用这么愚蠢的方法来切换时间显示器的类型）。为了达成整个目的，于是我们有了如下代码

```vue
<template>
	<button @click="handleClick">
        btn
    </button>
	<someComponent v-if="show" />
</template>
```



