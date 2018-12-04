本篇将尝试使用canvas + wasm画一个迷宫，生成算法主要用到连通集算法，使用wasm主要是为了提升运行效率。然后再用一个最短路径算法找到迷宫的出路，最后的效果如下：

![迷宫及其出路](https://user-gold-cdn.xitu.io/2017/7/31/aa343d30472dacf57dab3802c53e23ad?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

## 1、用连通集算法生成迷宫

生成迷宫的算法其实很简单，假设迷宫的大小是10 * 10，即这个迷宫有100个格子，通过不断地随机拆掉这100个格子中间的墙，直到可以从第一个格子走到最后一个格子，也就是说第一个格子和最后一个盒子处于同一个连通集。具体如下操作。

（1）生成100个格子，每个格子都不相通。

（2）随机选取相邻的两个格子，可以是左右相邻或者是上下相邻，判断这两个格子是不是处于同一个连通集，即能否从其中一个格子走到另外一个格子，如果不能，就拆掉它们中间的墙，让它们相连，处于同一个连通集。

（3）重复第二步，直到第一个格子和最后一个格子相连。

那这个连通集应该怎么表示呢？我们用一个一维数组来表示不同的已连通的集合，初始化的时候每个格子的值都为-1，如下图所示，假设迷宫为3 * 3，即有9个格子。

![连通集的表示](https://user-gold-cdn.xitu.io/2017/7/31/93ff2eaad31fd55d532a885a22a48db3?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

每个索引在迷宫的位置：

![索引与迷宫的映射关系](https://user-gold-cdn.xitu.io/2017/7/31/969250b2270c1d7c1d75ffb0016d5f49?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

负数表示它们是不同的连通集，因为我们还没开始拆墙，所以一开始它们都是独立的。

现在把3、4中间的墙拆掉，也是说让3和4连通，把4的值置成3，表示4在3这个连通集，3是它们的根，如下图所示：

![制造连通集](https://user-gold-cdn.xitu.io/2017/7/31/c6a2a7017980e03f7eb7fc4b20b3a676?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

再把5、8给拆掉：

![制造连通集](https://user-gold-cdn.xitu.io/2017/7/31/7106c36a836c1e8cb68c139ebb8a9b47?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

再把4、5给拆了：

![制造连通集](https://user-gold-cdn.xitu.io/2017/7/31/44700d9b24a30f49c9e12e0ee7e19a86?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

这个时候3、4、5、8就处于同一个连通集了，但是0和8依旧是两个不同的连通集，这个时候再把3和0中间的墙给拆了：

![制造连通集](https://user-gold-cdn.xitu.io/2017/7/31/a7220f020b08b76c6f586c906f092a2a?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

由于0的连通集是3，而8的连通集也是3，即它们处于同一个连通集，因此这个时候从第一个格子到最后一个格子的路是相通的，就生成了一个迷宫。

我们用UnionSet的类表示连通集，如下代码所示：

```javascript
class UnionSet{
    constructor(size){
        this.set = new Array(size);
        for(var i = this.set.length - 1; i >= 0; i--){
            this.set[i] = -1;
        }
    }
    union(root1, root2){
        this.set[root1] = root2;
    }
    findSet(x){
        while(this.set[x] >= 0){
            x = this.set[x];
        }
        return x;
    }
    sameSet(x, y){
        return this.findSet(x) === this.findSet(y);
    }
    unionElement(x, y){
        this.union(this.findSet(x), this.findSet(y));
    }
}
```

我们总共用了22行代码就实现了一个连通集。上面的代码应该比较好理解，对照上面的示意图。如findSet函数得到某个元素所在的set的根元素，而根元素存放的是负数，只要存放的值是正数那么它就是指向另一个节点，通过while循环一层层的往上找直到负数。unionElement可以连通两个元素，先找到它们所在的set，然后把它们的set union一下变成同一个连通集。

现在写一个Maze，用来控制画迷宫的操作，它组合一个UnionSet的实例，如下代码所示：

```javascript
class Maze{
    constructor(columns, rows, cavans){
        this.columns = columns;
        this.rows = rows;
        this.cells = columns * rows;
        //存放是连通的格子，{1: [2, 11]}表示第1个格子和第2、11个格子是相通的
        this.linkedMap = {};
        this.unionSets = new UnionSet(this.cells);
        this.canvas = canvas;
    }
}
```

Maze构造函数传三个参数，前两个是迷宫的列数和行数，最后一个是canvas元素。在构造函数里面初始化一个连通集，作为这个Maze的核心模型，还初始化了一个linkedMap，用来存放拆掉的墙，进而提供给canvas绘图。

Maze类再添加一个生成迷宫的函数，如下代码所示：

```javascript
//生成迷宫
generate(){
    //每次任意取两个相邻的格子，如果它们不在同一个连通集，
    //则拆掉中间的墙，让它们连在一起成为一个连通集
    while(!this.firstLastLinked()){
        var cellPairs = this.pickRandomCellPairs();
        if(!this.unionSets.sameSet(cellPairs[0], cellPairs[1])){
            this.unionSets.unionElement(cellPairs[0], cellPairs[1]);
            this.addLinkedMap(cellPairs[0], cellPairs[1]);
        }   
    }   
}
```

生成迷宫的核心逻辑很简单，在while循环里面判断第一个是否与最后一个格子连通，如果不是的话，则每次随机选取两个相邻的格子，如果它们不再同一个连通集，则把它们连通一下，同时记录一下拆掉的墙到linkedMap里面。

怎么随机选取两个相邻的格子呢？这个虽然没什么技术难点，但是实现起来需要动一番脑筋，因为在边上的格子没有完整的上下左右四个相邻格子，有些只有两个，有些有三个。笔者是这么实现的，相对起来比较简单。

```javascript
//取出随机的两个挨着的格子
pickRandomCellPairs(){
    var cell = (Math.random() * this.cells) >> 0;
    //再取一个相邻格子，0 = 上，1 = 右，2 = 下，3 = 左
    var neiborCells = []; 
    var row = (cell / this.columns) >> 0,
        column = cell % this.rows;
    //不是第一排的有上方的相邻元素
    if(row !== 0){ 
        neiborCells.push(cell - this.columns);
    }   
    //不是最后一排的有下面的相邻元素
    if(row !== this.rows - 1){ 
        neiborCells.push(cell + this.columns);
    }   
    if(column !== 0){ 
        neiborCells.push(cell - 1); 
    }   
    if(column !== this.columns - 1){ 
        neiborCells.push(cell + 1); 
    }   
    var index = (Math.random() * neiborCells.length) >> 0;
    return [cell, neiborCells[index]];
}
```

首先随机选一个格子，然后得到它的行数和列数，接着一次判断它的边界情况。如果它不是处于第一排，那么它就有上面一排的相邻格子，如果不是最后一排则有下面一排的相邻格子，同理，如果不是在第一列则有左边的，不是最后一列则有右边的。把符合条件的格子放到一个数组里面，然后再随机取这个数组里的一个元素。这样就得到了两个随机的相邻元素。

另一个addLinkedMap函数的实现较为简单，如下代码所示：

```javascript
addLinkedMap(x, y){
    if(!this.linkedMap[x]) this.linkedMap[x] = [];
    if(!this.linkedMap[y]) this.linkedMap[y] = [];
    if(this.linkedMap[x].indexOf(y) < 0){
        this.linkedMap[x].push(y);
    }
    if(this.linkedMap[y].indexOf(x) < 0){
        this.linkedMap[y].push(x);
    }
}
```

这样生成迷宫的核心逻辑基本完成，但是上面连通集的代码可以优化，一个是findSet函数，可以在findSet的时候把当前连通集里的元素的存放值直接改成根元素，这样就不用形成一条很长的查找链，或者说形成一棵很高的树，可直接一步到位，如下代码所示：

```javascript
findSet(x){
    if(this.set[x] < 0) return x;
    return this.set[x] = this.findSet(this.set[x]);
}
```

这段代码使用了一个递归，在查找的同时改变值。

union函数也可以做一个优化，findSet可以把树的高度改小，但是在没有改小前的union操作也可以做优化，如下代码所示：

```javascript
union(root1, root2){
    if(this.set[root1] < this.set[root2]){
        this.set[root2] = root1;
    } else {
        if(this.set[root1] === this.set[root2]){
            this.set[root2]--;
        }   
        this.set[root1] = root2;
    }   
}
```

这段代码的目的也是为了减少查找链的长度或者说减少树的高度，方法是把一棵比较矮的连通集称为另外一棵比较高的连通集的子树，这样两个连通集，合并起来的高度还是那颗高的。如果两个连通集的高度一样，则选取其中一个作为根，另外一棵树的节点在查找的时候无疑这些节点的查找长度要加上1，因为多了一个新的root，也就是说树的高度要加1，由于存放的是负数，所以进行减减操作。在判断树高度的时候也是一样的，越小就说明越高。

经验证，这样改过之后，代码执行效率快了一半以上。

迷宫生成好之后，现在开始来画。

## 2、用Canvas画迷宫

先写一个canvas的html元素，如下代码所示：

```html
<canvas id="maze" width="800" height="600"></canvas>
```

注意canvas的宽高要用width和height的属性写，如果用style的话就是拉伸了，会出现模糊的情况。

怎么用canvas画线呢？如下代码所示：

```javascript
var canvas = document.getElementById("maze");
var ctx = canvas.getContext("2d");
ctx.moveTo(0, 0);
ctx.lineTo(100, 100);
ctx.stroke();
```

这段代码画了一条线，从（0,0）到（100,100），这也是本篇将用到的canvas的3个基础的api。

上面已经得到了一个linkedMap，对于一个3 * 3的表格，把linkedMap打印一下，可得到以下表格。

![linkedMap](https://user-gold-cdn.xitu.io/2017/7/31/9147baba1df49acaf6b231b057ed7c3f?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

通过上面的表格可知道，0和3中间是没有墙，所以在画的时候0和3中间就不要画横线了，3和4也是相连的，它们中间就不要画竖线了。对每个普通的格子都画它右边的竖线和下面的横线，而对于被拆掉的就不要画，所以得到以下代码：

```javascript
draw(){ 
    var linkedMap = this.linkedMap;
    var cellWidth = this.canvas.width / this.columns,
        cellHeight = this.canvas.height / this.rows;
    var ctx = this.canvas.getContext("2d");
    //translate 0.5个像素，避免模糊
    ctx.translate(0.5, 0.5);
    for(var i = 0; i < this.cells; i++){
        var row = i / this.columns >> 0,
            column = i % this.columns;
        //画右边的竖线
        if(column !== this.columns - 1 && (!linkedMap[i] || linkedMap[i].indexOf(i + 1) < 0)){
            ctx.moveTo((column + 1) * cellWidth >> 0, row * cellHeight >> 0);
            ctx.lineTo((column + 1) * cellWidth >> 0, (row + 1) * cellHeight >> 0);
        }
        //画下面的横线
        if(row !== this.rows - 1 && (!linkedMap[i] || linkedMap[i].indexOf(i + this.columns) < 0)){
            ctx.moveTo(column * cellWidth >> 0, (row + 1) * cellHeight >> 0);
            ctx.lineTo((column + 1) * cellWidth >> 0, (row + 1) * cellHeight >> 0);
        }
    }
    //最后再一次性stroke，提高性能
    ctx.stroke();
    //画迷宫的四条边框
    this.drawBorder(ctx, cellWidth, cellHeight);
}
```

上面的代码也比较好理解，在画右边的竖线的时候，先判断它和右边的格子是否相通，即linkMap[i]里面有没有i + 1元素，如果没有，并且它不是最后一列，就画右边的竖线。因为迷宫的边框放到后面再画，它比较特殊，最后一个格子的竖线是不要画的，因为它是迷宫的出口。每次moveTo和lineTo的位置需要计算一下。

注意上面的代码做了两个优化，一个是先translate0.5个像素，为了让canvas画线的位置刚好在像素上面，因为我们的lineWidth是1，如果不translate，那么它画的位置如下图中间所示，相邻两个像素占了半个像素，显示器显示的时候会变虚，而translate0.5个像素之后，它就会刚好画在像素点上。详见：[HTML5 Canvas - Crisp lines every time](https://mobtowers.com/2013/04/15/html5-canvas-crisp-lines-every-time/)。

![使得画线刚好落在像素上](https://user-gold-cdn.xitu.io/2017/7/31/717befb63859501868673202e338f305?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

第二个优化是所有的moveTo和lineTo都完成之后再stroke，这样它就是一条线，可以极大地提高画图的效率。这个很重要，刚开始的时候没这么做，导致格子数稍多的时候就画不了，改成这样之后，绘制的效率提升很多。

我们还可以再做一个优化，就是使用双缓冲技术，在画的时候别直接画到屏幕上，而是先在内存的画布里面完成绘制，最后再一次性地Paint绘制到屏幕上，这样也可以提高性能。如下代码所示：

```javascript
draw(){
    var canvasBuffer = document.createElement("canvas");
    canvasBuffer.width = this.canvas.width;
    canvasBuffer.height = this.canvas.height;
    var ctx = canvasBuffer.getContext("2d");
    ctx.translate(0.5, 0.5);
    for(var i = 0; i < this.cells; i++){

    }   
    ctx.stroke();
    this.drawBorder(ctx, cellWidth, cellHeight);
    console.log("draw");
    this.canvas.getContext("2d").drawImage(canvasBuffer, 0, 0); 
}
```

先动态地创建一个canvas节点，获取它的context，在上面画图，画好之后再用原先的canvas的context的drawImage把它画到屏幕上去。

然后就可以写驱动代码了，如下画一个50 * 50的迷宫，并统计一下时间：

```javascript
const column = 50,
      row = 50;
var canvas = document.getElementById("maze");
var maze = new Maze(column, row, canvas);

console.time("generate maze");
maze.generate();
console.timeEnd("generate maze");
console.time("draw maze");
maze.draw();
console.timeEnd("draw maze");
```

画出的迷宫：

![生成的迷宫](https://user-gold-cdn.xitu.io/2017/7/31/f2db9bc86cc423f7f695d7e4857451c7?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

运行时间：

![运行时间](https://user-gold-cdn.xitu.io/2017/7/31/0d469ced618fcdbe41efeb7eb0103b55?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

可以看到画一个2500规模的迷宫，draw的时间还是很少的，而生成的时间也不长，但是我们发现一个问题，就是迷宫的有些格子是封闭的：

![迷宫会出现封闭的格子](https://user-gold-cdn.xitu.io/2017/7/31/1bb8717064c4ef8e1ed28202f73d5450?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

这些不能够进去的格子就没用了，这不太符合迷宫的特点。所以不能存在自我封闭的格子，由于生成的时候是判断第一个格子有没有和最后一个连通，现在改成第一个格子和所有的格子都是连通的，也就是说可以从第一个格子到达任意一个格子，这样迷宫的误导性才比较强，如下代码所示：

```javascript
linkedToFirstCell(){
    for(var i = 1; i < this.cells; i++){
        if(!this.unionSets.sameSet(0, i)) 
            return false;
    }   
    return true;
}
```

把while的判断也改一下，这样改完之后，生成的迷宫变成了这样：

![全连通的迷宫](https://user-gold-cdn.xitu.io/2017/7/31/b7aaca44fb01ea8c69d4aadfffba1389?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

这样生成的迷宫看起来就正常多了，生成迷宫的时间也相应地变长：

![增加了生成迷宫的时间](https://user-gold-cdn.xitu.io/2017/7/31/df9990c8ab9b7510cb2e9f4b1c7f7aef?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

但是我们发现还是有一些比较奇怪的格子布局，如下图所示：

![奇怪的格子布局](https://user-gold-cdn.xitu.io/2017/7/31/1685a4e29e251b4578f5bfdc8473d323?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

因为这样布局的其实没太大的意义，如果让你手动设计一个迷宫，你肯定也不会设计这样的布局。所以我们的算法还可以再改进，由于上面是随机选取两个相邻格子，可以把它改成随机选取4个相邻的格子，这样生成的迷宫通道就会比较长，像上图这种比较奇葩的情况就会比较少。

## 3、用最短路径算法找到迷宫的出路

这个模型更为常见的场景是，现在我在A城镇，准备去Z城镇，中间要绕道B、C、D等城镇，并且有很多条路线可选，并且知道每个城镇和它连通的城镇以及两两之间距离，现在要求解一条A到Z的最短的路，如下图所示：

![最短路径算法](https://user-gold-cdn.xitu.io/2017/7/31/beed1ef99202ab0f2923a6c33ce1e647?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

在迷宫的模型里面也是类似的，要求解从第一个格子到最后一个格子的最短路径，并且已经知道格子之间的连通情况。不一样的是相邻格子之间的距离是无权的，都为1，所以这个处理起来会更加简单。

用一个贪婪算法可以解决这个问题，假设从A到Z的最短路径为A->C->G->Z，那么这条路径也是A到G、A到C的最短路径，因为如果A到G还有更短的路径，那么A到Z的距离就还可以更短了，即这条路径不是最短的。因此我们从A开始延伸，一步步地确定A到其他地点的最短路径，直到扩散到Z。

在无权的情况下，如上面任意相邻城镇的距离相等，和A直接相连的节点必定是A到这个节点的最短路径，如上图A到B、C、F的最短路径为A->B、A->C、A->F，这三个点的最短路径可标记为已知。和C直接相邻的是G和D，C是最短的，所以A->C->G和A->C->D也是最短的，再往下一层，和G、D直接相连的分别是E和Z，所以A->C->G->Z和A->C->D->E是到Z和E的一条最短路径，到此就知道了A->Z的最短路线。E也可以到Z，但是由于Z已经被标记为已知最短了，所以通过E的这条路径就被放弃了。

和A直接相连的作为第一层，而和第一层直接相连的作为第二层，由第一层到第二层一直延伸目标节点，先被找到的节点就会被标记为已知。这是一个广度优先搜索。

而在有权的情况下， 刚开始的是A被标记为已知，由于A和C是最短的，所以C也被标记为已知，B和F不会标记，但是它们和A的距离会受到更新，由初始化的无穷大更新为A->B和A->F的距离。在已查到但未标记的两个点里面，A->F的距离是最短的，所以F被标记为已知，这是因为如果存在另外一条更短的未知的路到F，它必定得先经过已经查到的点（因为已经查找过的点是A的必经之路），这里面已经是最短的了，所以不可能还有更短的了。F被标记为已知之后和F直接相连的E的距离得到更新，同样地，在已查找到但未标记的点里面B的距离最短，所以B被标记为已知，然后再更新和B相连的点的距离。重复这个过程，直到Z被标记为已知。

标记起始点为已知，更新表的距离，再标记表里最短的距离为已知，再更新表的距离，重复知道目的点被标记，这个算法也叫Dijkstra算法。

现在来实现一个无权的最短路径，如下代码所示：

```javascript
calPath(){
    var pathTable = new Array(this.cells);
    for(var i = 0; i < pathTable.length; i++){
        pathTable[i] = {known: false, prevCell: -1};
    }   
    pathTable[0].known = true;
    var map = this.linkedMap;
    //用一个队列存储当前层的节点，先进队列的结点优先处理
    var unSearchCells = [0];
    var j = 0;
    while(!pathTable[pathTable.length - 1].known){
        while(unSearchCells.length){
            var cell = unSearchCells.pop();
            for(var i = 0; i < map[cell].length; i++){
                if(pathTable[map[cell][i]].known) continue;
                pathTable[map[cell][i]].known = true;
                pathTable[map[cell][i]].prevCell = cell; 
                unSearchCells.unshift(map[cell][i]);
                if(pathTable[pathTable.length - 1].known) break;
            }   
        }   
    }   
    var cell = this.cells - 1;
    var path = [cell];
    while(cell !== 0){ 
        var cell = pathTable[cell].prevCell;
        path.push(cell);
    }   
    return path;
}
```

