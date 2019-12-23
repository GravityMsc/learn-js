vue在官方文档中提到了与react的渲染性能对比中，因为其使用了snabbdom而有更优异的性能。

> JavaScript开销直接与求算必要DOM操作的机制相关。尽管Vue和React都使用了Virtual DOM实现这一点，但Vue的Virtual DOM实现（复刻自snabbdom）是更加轻量化的，因此也就比React的实现更高效。

看到火到不行的国产前端框架vue也在使用别人的Virtual DOM开源方案，是不是很好奇snabbdom有何强大之处呢？不过正是解密snabbdom之前，先简单介绍下Virtual DOM。

## 什么是Virtual DOM

Virtual DOM可以看作一棵模拟了DOM树的JavaScript树，其主要是通过VNode，实现一个无状态的组件，当组件状态发生更新时，然后触发Virtual DOM数据的变化，然后通过Virtual DOM和真实DOM的比对，再对真实DOM更新。可以简单认为Virtual DOM是真实DOM的缓存。

## 为什么用Virtual DOM

我们知道，当我们希望实现一个具有复杂状态的界面时，如果我们在每个可能变化的组件上都绑定事件，绑定字段数据，那么很快由于状态太多，我们需要维护的事件和字段将会越来越多，代码也会越来越复杂，于是，我们想我们可不可以将视图和状态分开来，只要视图发生变化，对应状态也发生变化，然后状态变化，我们再重绘整个视图就好了。

这样的想法虽好，但是代价太高了，于是我们又想，能不能只更新状态发生变化的视图？于是Virtual DOM应运而生，状态变化先反馈到Virtual DOM上，Virtual DOM在找到最小更新视图，最后批量更新到真实DOM上，从而达到性能的提升。

除此之外，从移植性上看，Virtual DOM还对真实DOM做了一次抽象，这意味着Virtual DOM对应可以不是浏览器的DOM，而是不同设备的组件，极大的方便了多平台的使用。如果是要实现前后端同构直出方案，使用Virtual DOM的框架实现起来是比较简单的，因为在服务端的Virtual DOM跟浏览器DOM接口并没有绑定关系。

基于Virtual DOM的数据更新与UI同步机制：

![基于Virtual DOM的数据更新与UI同步机制](https://images2015.cnblogs.com/blog/572874/201705/572874-20170505152506554-159618539.png)

初始渲染时，首先将数据渲染为Virtual DOM，然后由Virtual DOM生成真实的DOM。

![生成真实的DOM过程](https://images2015.cnblogs.com/blog/572874/201705/572874-20170505152941757-1328855064.png)

数据更新时，渲染得到新的Virtual DOM，与上一次得到的Virtual DOM进行diff，得到所有需要在DOM上进行的变更，然后再patch过程中应用到DOM上实现UI的同步更新。

Virtual DOM作为数据结构，需要能准确地转换为真实DOM，并且方便进行对比。

介绍完Virtual DOM，我们应该对snabbdom的功用有个认识了，下面具体解剖下snabbdom这只“小麻雀”。

## snabbdom

### vnode

DOM通常被视为一棵树，元素则是这棵树上的节点（node），而Virtual DOM的基础，就是Virtual Node了。

Snabbdom的Virtual DOM则是纯数据对象，通过vnode模块来创建，对象属性包括：

* sel
* data
* children
* text
* elm
* key

可以看到Virtual Node用于创建真实节点的数据包括：

* 元素类型
* 元素属性
* 元素的子节点

源码：

```javascript
//VNode函数，用于将输入转化成VNode
    /**
     *
     * @param sel    选择器
     * @param data    绑定的数据
     * @param children    子节点数组
     * @param text    当前text节点内容
     * @param elm    对真实dom element的引用
     * @returns {{sel: *, data: *, children: *, text: *, elm: *, key: undefined}}
     */
function vnode(sel, data, children, text, elm) {

     var key = data === undefined ? undefined : data.key;
      return { sel: sel, data: data, children: children,
          text: text, elm: elm, key: key };
}
```

snabbdom并没有直接暴露vnode对象给我们用，而是使用h包装器，h的主要功能是处理参数：

```javascript
h(sel,[data],[children],[text]) => vnode
```

从snabbdom的typescript的源码可以看出，其实就是这几种函数的重载：

```javascript
export function h(sel: string): VNode; 
export function h(sel: string, data: VNodeData): VNode; 
export function h(sel: string, text: string): VNode; 
export function h(sel: string, children: Array<VNode | undefined | null>): VNode; 
export function h(sel: string, data: VNodeData, text: string): VNode; 
export function h(sel: string, data: VNodeData, children: Array<VNode | undefined | null>): VNode; 
```

### patch

创建vnode后，接下来就是调用patch方法将Virtual DOM渲染成真实DOM了。patch是snabbdom的init函数返回的。snabbdom.init传入modules数组，module用来扩展snabbdom创建复杂DOM的能力。

不多说了直接上patch的源码：

```javascript
return function patch(oldVnode, vnode) {
    var i, elm, parent;
    //记录被插入的vnode队列，用于批触发insert
    var insertedVnodeQueue = [];
    //调用全局pre钩子
    for (i = 0; i < cbs.pre.length; ++i) cbs.pre[i]();
    //如果oldvnode是dom节点，转化为oldvnode
    if (isUndef(oldVnode.sel)) {
      oldVnode = emptyNodeAt(oldVnode);
    }
    //如果oldvnode与vnode相似，进行更新
    if (sameVnode(oldVnode, vnode)) {
      patchVnode(oldVnode, vnode, insertedVnodeQueue);
    } else {
      //否则，将vnode插入，并将oldvnode从其父节点上直接删除
      elm = oldVnode.elm;
      parent = api.parentNode(elm);

      createElm(vnode, insertedVnodeQueue);

      if (parent !== null) {
        api.insertBefore(parent, vnode.elm, api.nextSibling(elm));
        removeVnodes(parent, [oldVnode], 0, 0);
      }
    }
    //插入完后，调用被插入的vnode的insert钩子
    for (i = 0; i < insertedVnodeQueue.length; ++i) {
      insertedVnodeQueue[i].data.hook.insert(insertedVnodeQueue[i]);
    }
    //然后调用全局下的post钩子
    for (i = 0; i < cbs.post.length; ++i) cbs.post[i]();
    //返回vnode用作下次patch的oldvnode
    return vnode;
    };
```

先判断新虚拟DOM是否是相同层级vnode，是才执行patchVnode，否则创建新DOM删除旧DOM，判断是否相同vnode比较简单：

```javascript
function sameVnode(vnode1, vnode2) {
    //判断key值和选择器
    return vnode1.key === vnode2.key && vnode1.sel === vnode2.sel;
}
```

patch方法里面实现了snabbdom作为一个高效virtual dom库的法宝——高效的diff算法，可以用一张图示意：

![virtual dom的diff算法](https://images2015.cnblogs.com/blog/572874/201705/572874-20170505153024476-62826483.png)

diff算法的核心是比较只会在同层级进行，不会跨层级比较。不是逐层逐层搜索遍历的方式，这样做时间复杂度将会达到O(n^3)的级别，代价非常高。只比较同层级的方式的时间复杂度可以降低到O(n)。

patchvnode函数的主要作用是以打补丁的方式去更新DOM树。

```javascript
function patchVnode(oldVnode, vnode, insertedVnodeQueue) {
    var i, hook;
    //在patch之前，先调用vnode.data的prepatch钩子
    if (isDef(i = vnode.data) && isDef(hook = i.hook) && isDef(i = hook.prepatch)) {
      i(oldVnode, vnode);
    }
    var elm = vnode.elm = oldVnode.elm, oldCh = oldVnode.children, ch = vnode.children;
    //如果oldvnode和vnode的引用相同，说明没发生任何变化直接返回，避免性能浪费
    if (oldVnode === vnode) return;
    //如果oldvnode和vnode不同，说明vnode有更新
    //如果vnode和oldvnode不相似则直接用vnode引用的DOM节点去替代oldvnode引用的旧节点
    if (!sameVnode(oldVnode, vnode)) {
      var parentElm = api.parentNode(oldVnode.elm);
      elm = createElm(vnode, insertedVnodeQueue);
      api.insertBefore(parentElm, elm, oldVnode.elm);
      removeVnodes(parentElm, [oldVnode], 0, 0);
      return;
    }
    //如果vnode和oldvnode相似，那么我们要对oldvnode本身进行更新
    if (isDef(vnode.data)) {
      //首先调用全局的update钩子，对vnode.elm本身属性进行更新
      for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode);
      //然后调用vnode.data里面的update钩子,再次对vnode.elm更新
      i = vnode.data.hook;
      if (isDef(i) && isDef(i = i.update)) i(oldVnode, vnode);
    }
    //如果vnode不是text节点
    if (isUndef(vnode.text)) {
      //如果vnode和oldVnode都有子节点
      if (isDef(oldCh) && isDef(ch)) {
        //当Vnode和oldvnode的子节点不同时，调用updatechilren函数，diff子节点
        if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue);
      }
      //如果vnode有子节点，oldvnode没子节点
      else if (isDef(ch)) {
        //oldvnode是text节点，则将elm的text清除
        if (isDef(oldVnode.text)) api.setTextContent(elm, '');
        //并添加vnode的children
        addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue);
      }
      //如果oldvnode有children，而vnode没children，则移除elm的children
      else if (isDef(oldCh)) {
        removeVnodes(elm, oldCh, 0, oldCh.length - 1);
      }
      //如果vnode和oldvnode都没chidlren，且vnode没text，则删除oldvnode的text
      else if (isDef(oldVnode.text)) {
        api.setTextContent(elm, '');
      }
    }

    //如果oldvnode的text和vnode的text不同，则更新为vnode的text
    else if (oldVnode.text !== vnode.text) {
      api.setTextContent(elm, vnode.text);
    }
    //patch完，触发postpatch钩子
    if (isDef(hook) && isDef(i = hook.postpatch)) {
      i(oldVnode, vnode);
    }
  }
```

patchVnode将新旧虚拟DOM分为几种情况，执行替换textContent还是updateChildren。

updateChildren是实现diff算法的主要地方：

```javascript
function updateChildren(parentElm, oldCh, newCh, insertedVnodeQueue) {
        var oldStartIdx = 0, newStartIdx = 0;
        var oldEndIdx = oldCh.length - 1;
        var oldStartVnode = oldCh[0];
        var oldEndVnode = oldCh[oldEndIdx];
        var newEndIdx = newCh.length - 1;
        var newStartVnode = newCh[0];
        var newEndVnode = newCh[newEndIdx];
        var oldKeyToIdx;
        var idxInOld;
        var elmToMove;
        var before;
        while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
            if (oldStartVnode == null) {
                oldStartVnode = oldCh[++oldStartIdx]; // Vnode might have been moved left
            }
            else if (oldEndVnode == null) {
                oldEndVnode = oldCh[--oldEndIdx];
            }
            else if (newStartVnode == null) {
                newStartVnode = newCh[++newStartIdx];
            }
            else if (newEndVnode == null) {
                newEndVnode = newCh[--newEndIdx];
            }
            else if (sameVnode(oldStartVnode, newStartVnode)) {
                patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue);
                oldStartVnode = oldCh[++oldStartIdx];
                newStartVnode = newCh[++newStartIdx];
            }
            else if (sameVnode(oldEndVnode, newEndVnode)) {
                patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue);
                oldEndVnode = oldCh[--oldEndIdx];
                newEndVnode = newCh[--newEndIdx];
            }
            else if (sameVnode(oldStartVnode, newEndVnode)) {
                patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue);
                api.insertBefore(parentElm, oldStartVnode.elm, api.nextSibling(oldEndVnode.elm));
                oldStartVnode = oldCh[++oldStartIdx];
                newEndVnode = newCh[--newEndIdx];
            }
            else if (sameVnode(oldEndVnode, newStartVnode)) {
                patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue);
                api.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm);
                oldEndVnode = oldCh[--oldEndIdx];
                newStartVnode = newCh[++newStartIdx];
            }
            else {
                if (oldKeyToIdx === undefined) {
                    oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx);
                }
                idxInOld = oldKeyToIdx[newStartVnode.key];
                if (isUndef(idxInOld)) {
                    api.insertBefore(parentElm, createElm(newStartVnode, insertedVnodeQueue), oldStartVnode.elm);
                    newStartVnode = newCh[++newStartIdx];
                }
                else {
                    elmToMove = oldCh[idxInOld];
                    if (elmToMove.sel !== newStartVnode.sel) {
                        api.insertBefore(parentElm, createElm(newStartVnode, insertedVnodeQueue), oldStartVnode.elm);
                    }
                    else {
                        patchVnode(elmToMove, newStartVnode, insertedVnodeQueue);
                        oldCh[idxInOld] = undefined;
                        api.insertBefore(parentElm, elmToMove.elm, oldStartVnode.elm);
                    }
                    newStartVnode = newCh[++newStartIdx];
                }
            }
        }
        if (oldStartIdx > oldEndIdx) {
            before = newCh[newEndIdx + 1] == null ? null : newCh[newEndIdx + 1].elm;
            addVnodes(parentElm, before, newCh, newStartIdx, newEndIdx, insertedVnodeQueue);
        }
        else if (newStartIdx > newEndIdx) {
            removeVnodes(parentElm, oldCh, oldStartIdx, oldEndIdx);
        }
    }
```

updateChildren的代码比较有难度，借助几张图比较好理解些：

![updateChildren的图示](https://images2015.cnblogs.com/blog/572874/201705/572874-20170505153104851-108994539.png)

过程可以概括为：oldCh和newCh各有两个头尾的变量StartIdx和EndIdx，它们的2个变量相互比较，一共有4种比较方式。如果4中比较都没匹配，如果设置了key，就会用key进行比较，在比较的过程中，变量会往中间靠，一旦StartIdx > EndIdx表明oldCh和newCh至少有一个已经遍历完了，就会结束比较。

具体的diff分析：

对于与sameVnode（oldStartVnode， newStartVnode）和sameVnode(oldEndVnode, newEndVnode)为true的情况，不需要对dom进行移动。

有3种需要dom操作的情况：

1. 当oldStartVnode，newEndVnode相同层级时，说明oldStartVnode.el跑到oldEndVnode.el的后边了。

   ![dom需要进行移动](https://images2015.cnblogs.com/blog/572874/201705/572874-20170505153150820-1790639880.png)

2. 当oldEndvnode，newStartVnode相同层级时，说明oldEndVnode.el跑到了newStartVnode.el的前边。

   ![dom需要进行移动](https://images2015.cnblogs.com/blog/572874/201705/572874-20170505153227804-1587358974.png)

3. newCh中的节点oldCh里没有，将新节点插入到oldStartVnode.el的前边。

   ![dom需要进行移动](https://images2015.cnblogs.com/blog/572874/201705/572874-20170505153301070-885467815.png)

在结束时，分为两种情况：

1. oldStartIdx > oldEndIdx，可以认为oldCh先遍历完。当然也有可能newCh此时也正好完成了遍历，统一都归为此类。此时newStartIdx和newEndIdx之间的vnode是新增的，调用addVnodes，把它们全部插进before的后边，before很多时间是为null的。addVnodes调用的是insertBefore操作dom节点，我们看看insertBefore的文档：parentElement.insertBefore(newElement, referenceElement)如果referenceElement为null则newElement将被插入到子节点的末尾。如果newElement已经在DOM树中，newElement首先会从DOM树中移除。所以before为null，newElement将被插入到子节点的末尾。

   ![结束时dom的情况](https://images2015.cnblogs.com/blog/572874/201705/572874-20170505153340648-1168827360.png)

2. newStartIdx > newEndIdx，可以认为newCh先遍历完。此时oldStartIdx和oldEndIdx之间的vnode在新的子节点里已经不存在了，调用removeVnodes将它们从dom里删除。

   ![结束时dom的情况](https://images2015.cnblogs.com/blog/572874/201705/572874-20170505153419007-23640952.png)

### hook

snabbdom主要流程的代码在上面就介绍完毕了，在上面的代码中可能看不出来如果要创建比较复杂的dom，比如有attribute、props、eventlistener的dom怎么办？奥秘就在于snabbdom在各个主要的环节提供了钩子。钩子方法中可以执行扩展模块，attribute、props、eventlistener等可以通过扩展模块实现。

在源码中可以看到hook是在snabbdom初始化的时候注册的。

```javascript
var hooks = ['create', 'update', 'remove', 'destroy', 'pre', 'post'];
var h_1 = require("./h");
exports.h = h_1.h;
var thunk_1 = require("./thunk");
exports.thunk = thunk_1.thunk;
function init(modules, domApi) {
    var i, j, cbs = {};
    var api = domApi !== undefined ? domApi : htmldomapi_1.default;
    for (i = 0; i < hooks.length; ++i) {
        cbs[hooks[i]] = [];
        for (j = 0; j < modules.length; ++j) {
            var hook = modules[j][hooks[i]];
            if (hook !== undefined) {
                cbs[hooks[i]].push(hook);
            }
        }
    }
```

snabbdom在全局下有6种类型的钩子，触发这些钩子时，会调用对应的函数对节点的状态进行更改，首先我们来看看有哪些钩子以及它们触发的时间：

| Name        | Triggered when                                     | Arguments to callback   |
| ----------- | -------------------------------------------------- | ----------------------- |
| `pre`       | the patch process begins                           | none                    |
| `init`      | a vnode has been added                             | `vnode`                 |
| `create`    | a DOM element has been created based on a vnode    | `emptyVnode, vnode`     |
| `insert`    | an element has been inserted into the DOM          | `vnode`                 |
| `prepatch`  | an element is about to be patched                  | `oldVnode, vnode`       |
| `update`    | an element is being updated                        | `oldVnode, vnode`       |
| `postpatch` | an element has been patched                        | `oldVnode, vnode`       |
| `destroy`   | an element is directly or indirectly being removed | `vnode`                 |
| `remove`    | an element is directly being removed from the DOM  | `vnode, removeCallback` |
| `post`      | the patch process is done                          | none                    |

比如在patch的代码中可以看到调用了pre钩子：

```javascript
return function patch(oldVnode, vnode) {
        var i, elm, parent;
        var insertedVnodeQueue = [];
        for (i = 0; i < cbs.pre.length; ++i)
            cbs.pre[i]();
        if (!isVnode(oldVnode)) {
            oldVnode = emptyNodeAt(oldVnode);
        }
```

我们找一个比较简单的class模块来看下其源码：

```javascript
function updateClass(oldVnode, vnode) {
    var cur, name, elm = vnode.elm, oldClass = oldVnode.data.class, klass = vnode.data.class;
    if (!oldClass && !klass)
        return;
    if (oldClass === klass)
        return;
    oldClass = oldClass || {};
    klass = klass || {};
    for (name in oldClass) {
        if (!klass[name]) {
            elm.classList.remove(name);
        }
    }
    for (name in klass) {
        cur = klass[name];
        if (cur !== oldClass[name]) {
            elm.classList[cur ? 'add' : 'remove'](name);
        }
    }
}
exports.classModule = { create: updateClass, update: updateClass };
Object.defineProperty(exports, "__esModule", { value: true });
exports.default = exports.classModule;

},{}]},{},[1])(1)
});
```

可以看出create和update钩子方法调用的时候，可以执行class模块的updateClass：从elm中删除vnode中不存在的或者值为false的类，将vnode中新的class添加到elm上去。

### 总结snabbdom

1. vnode是基础数据结构
2. patch创建或更新DOM树
3. diff算法只比较同层级
4. 通过钩子和扩展模块创建有attribute、props、eventlistener的复杂dom

## 参考

* [snabbdom源代码](https://github.com/snabbdom/snabbdom/)
* [snabbdom源码学习一](https://segmentfault.com/a/1190000009017324)
* [snabbdom源码学习二](https://segmentfault.com/a/1190000009017349)
* [解析vue2.0的diff算法](http://web.jobbole.com/90831/)