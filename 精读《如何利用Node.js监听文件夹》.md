## 1. 引言

本期精读的文章是：[How to Watch for Files Changes in Node.js](http://thisdavej.com/how-to-watch-for-files-changes-in-node-js/)，探讨如何监听文件的变化。

如果想使用现成的库，推荐[chokidar](https://www.npmjs.com/package/chokidar)或[node-watch](https://www.npmjs.com/package/node-watch)，如果想了解实现原理，请往下阅读。

## 2. 概述

### 使用fs.watchfile

使用fs内置函数watchfile似乎可以解决问题：

```typescript
fs.watchFile(dir, (curr, prev) => {});
```

但你可能会发现这个回调执行有一定延迟，因为watchfile是通过轮询检测文件变化的，它并不能实时作出反馈，而且只能监听一个文件，存在效率问题。

### 使用fs.watch

使用fs的另一个内置函数watch是更好的选择：

```typescript
fs.watch(dir, (event, filename) => {});
```

watch通过操作系统提供的文件更改通知机制，在Linux操作系统使用inotify，在macOS系统使用FSEvents，在Windows系统使用ReadDirectoryChangesW，而且可以用来监听目录的变化，在监听文件夹的场景中，比创建N个fs.watchfile效率高出很多。

```bash
$ node file-watcher.js
[2018-05-21T00:55:52.588Z] Watching for file changes on ./button-presses.log
[2018-05-21T00:56:00.773Z] button-presses.log file Changed
[2018-05-21T00:56:00.793Z] button-presses.log file Changed
[2018-05-21T00:56:00.802Z] button-presses.log file Changed
[2018-05-21T00:56:00.813Z] button-presses.log file Changed
```

但当我们修改一个文件时，回调却执行了4次！原因是文件被写入时，可能触发多次写操作，即使只保存了一次。但我们不需要这么敏感的回调，因为通常认为一次保存就是一次修改，系统底层写了几次文件我们并不关心。

因而可以进一步判断是否触发状态是change：

```typescript
fs.watch(dir, (event, filename) => {
    if(filename && event === "change"){
        console.log(`${filename} file Changed`);
    }
});
```

这样做可以一定程度解决问题，但作者发现Raspbian系统不支持rename事件，如果归类为change，会导致这样的判断毫无意义。

> 作者要表达的意思是，在不同的平台下，fs.watch的规则可能会不同，原因是fa.watch分别使用了个平白提供的api，所以无法保证这些api实现规则的统一性。

### 优化方案一：对比文件修改时间

基于fs.watch，增加了对修改时间的判断：

```typescript
let previousMTime = new Date(0);

fs.watch(dir, (event, filename) => {
    if(filename){
        const stats = fs.statSync(filename);
        if(stats.mtime.valueOf() === previousMTime.vulueOf()){
            return;
        }
        previousMTime = stats.mtime;
        console.log(`${filename} file Changed`);
    }
});
```

log由4个变成了3个，但依然存在问题。我们认为文件内容变化才算有修改，但操作系统考虑的因素更多，所以我们再尝试对比文件内容是否变化。

> 笔者补充：另外一些开源编辑器可能会先清空文件再写入，也会影响到触发回调的次数。

### 优化方案二：校验文件md5

只有文件内容变化了，才认为触发了改动，这下总可以了吧：

```typescript
let md5Previous = null;

fs.watch(dir, (event, filename) => {
    if(filename){
        const md5Current = md5(fs.readFileSync(buttonPressesLogFile));
        if(md5Current === md5Previous){
            return;
        }
        md5Previous = md5Current;
        console.log(`${filename} file Changed`);
    }
});
```

log终于由3个变成了2个，为什么多出一个？可能的原因是，在文件保存过程中，系统可能会触发多个回调事件，也许存在中间态。

### 优化方案三：加入延迟机制

我们尝试延迟100ms进行判断，也许能避开中间态：

```typescript
let fsWait = false;
fs.watch(dir, (event, filename) => {
    if(filename){
        if(fsWait) return;
        fsWait = setTimeout(() => {
            fsWait = false;
        }, 100);
        console.log(`${filename} file Changed`);
    }
});
```

这下log变成一个了。很多npm包在这里使用了debounce函数控制触发频率，才将触发频率修正。

而且我们需要结合md5与延迟机制共同作用，才能得到相对精准的结果：

```typescript
let md5Previous = null;
let fsWait = false;
fs.watch(dir, (event, filename) => {
    if(filename){
        if(fsWait)return;
        fsWait = setTimeout(() => {
            fsWait = false;
        }, 100);
        const md5Current = md5(fs.readFileSync(dir));
        if(md5Current === md5Previous){
            return;
        }
        md5Previous = md5Current;
        console.log(`${filename} file Changed`);
    }
});
```

## 3. 精读

作者讨论了一些实现文件夹监听的基本方式，可以看出，使用了各平台原生API的fs.watch并不那么靠谱，但这也是我们监听文件的唯一手段，所以需要基于它进行一系列优化。

而实际场景中，还需要考虑区分文件夹与文件、软连接、读写权限等情况。

另外用在生产环境的ku，也基本使用50到100毫秒解决重复触发的问题。

所以无论chokidar或node-watch，都大量使用了文中提及的技巧，再加上对边界条件的处理，对软连接、权限等情况处理，将所有可能情况都考虑到，才能提供较为精确的回调。

比如判断文件写入操作是否完毕，也需要通过轮询的方式：

```typescript
function awaitWriteFinish() {
  // ...省略
  fs.stat(
    fullPath,
    function(err, curStat) {
      // ...省略

      if (prevStat && curStat.size != prevStat.size) {
        this._pendingWrites[path].lastChange = now;
      }

      if (now - this._pendingWrites[path].lastChange >= threshold) {
        delete this._pendingWrites[path];
        awfEmit(null, curStat);
      } else {
        timeoutHandler = setTimeout(
          awaitWriteFinish.bind(this, curStat),
          this.options.awaitWriteFinish.pollInterval
        );
      }
    }.bind(this)
  );
  // ...省略
}
```

可以看出，第三方npm库都采取不信任操作系统回调的方式，根据文件信息完全重写了判断逻辑。

可见，信任操作系统的回调，就无法抹平所有操作系统间的差异，唯有统一重写文件的“写入”、“删除”、“修改”等逻辑，才能保证全平台的兼容性。

## 4. 总结

利用node.js监听文件夹变化很容易，但提供准确的回调却很难，主要难在两点：

1. 抹平操作系统之间的差异，这需要在结合fs.watch的同时，增加一些额外校验机制与延时机制。
2. 分清操作系统预期与用户预期，比如编辑器的额外操作、操作系统的多次读写都应该被忽略，用户的预期不会那么频繁，会忽略极小时间段内的连续触发。

另外还有兼容性、权限、软连接等其他因素要考虑，fs.watch并不是一个开箱可用的工程级别API。

## 5. 更多讨论

> 讨论地址是：[精读《如何利用 Nodejs 监听文件夹》 · Issue #87 · dt-fe/weekly](https://link.juejin.im/?target=https%3A%2F%2Fgithub.com%2Fdt-fe%2Fweekly%2Fissues%2F87)

如果你想参与讨论，请[点击这里](https://github.com/dt-fe/weekly)，每周都有新的主题，周末或周一发布。





