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

这个算法实现的关键在于用一个队列存储未经处理的节点，每处理一个节点时，就把和这个节点相连的点加入队列，这样新加入队列的节点就会排到当前层级的节点的后面，当把第一层的节点处理完了，就会把第二层的节点都push到队尾，同理当把第二层的节点都出队了，就会把第三层的节点推到队尾。这样就实现了一个广度优先搜索。

在处理每个节点都需要先判断一下当前节点是否已被标记为known，如果是的话就不用处理了。

在pathTable表格里面用一个prevCell记录到这个节点的上一个节点是哪个，为了能够从目的节点一直往前找到到达第一个节点的路径。最后找到这个path返回。

只要有这个path，就能够计算位置画出路径的图，如下图所示：

![画出路径](https://user-gold-cdn.xitu.io/2017/7/31/0f113353f2bb48f473a7383206d705ac?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

这个算法的速度还是很快的，如下图所示：

![最短路径算法的计算速度](https://user-gold-cdn.xitu.io/2017/7/31/89ce6830ce5b834212d4630e4f53b9d9?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

当把迷宫的规模提高到200 * 200时：

![扩大迷宫规模](https://user-gold-cdn.xitu.io/2017/7/31/d19d0653cbf05429aaa384364e1136da?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

生成迷宫的时间就很耗时了，花费了10秒：

![生成迷宫的耗时](https://user-gold-cdn.xitu.io/2017/7/31/2544ce49ce777b90f7d9eb0616412cd4?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

于是想着利用WASM提高生成迷宫的效率，看看能提升多少。我在《[WebAssembly与程序编译](http://fed.renren.com/2017/05/21/webassembly/)》这篇里已经介绍了WASM的一些基础知识，本篇我将用它来生成迷宫。

## 4、用WASM生成迷宫

我在《[WebAssembly与程序编译](http://fed.renren.com/2017/05/21/webassembly/)》提过用JS写很难编译，所以本篇也直接用C来写。上面是用用的class，但是WASM用C写没有class的类型，只支持基本的操作。但是可以用一个struct存放数据，函数名也相应地做修改，如下代码所示：

```c
struct Data{
    int *set;
    int columns;
    int rows;
    int cells;
    int **linkedMap;
} data;

void Set_union(int root1, int root2){
    int *set = data.set;
    if(set[root1] < set[root2]){
        set[root2] = root1;
    } else {
        if(set[root1] == set[root2]){
            set[root2]--;
        }
        set[root1] = root2;
    }
}

int Set_findSet(int x){
    if(data.set[x] < 0) return x;
    else return data.set[x] = Set_findSet(data.set[x]);
}
```

数据类型都是强类型的，函数名以类名Set_开头，类的数据放在一个struct结构里面。主要导出函数为：

```c
#include <emscripten.h>

EMSCRIPTEN_KEEPALIVE //这个宏表示这个函数要作为导出的函数
int **Maze_generate(int columns, int rows){
    Maze_init(columns, rows);
    Maze_doGenerate();
    return data.linkedMap;
    //return Maze_getJSONStr(); 
}
```

传进来列数和行数，返回一个二维数组。其他代码相应地改成C代码，这里不再放出来。需要注意的是，由于这里用到了一些C内置的库，如使用随机数函数rand()，所以不能用上文提到的生成wasm的方法，不然会报rand等库函数没有定义。

把生成wasm的命令改成：

> emcc maze.c -Os -s WASM=1 -o maze-wasm.html

这样它会生成一个maze-wasm.js和maze-wasm.wasm（生成的html文件不需要用到），生成的JS文件是用来自动加载和导入wasm文件的，在html里面引入这个JS：

```html
<script src="maze-wasm.js"></script>
<script src="maze.js"></script>
```

它就会自动去加载maze-wasm.wasm文件，同时会定义一个全局的Module对象，在wasm文件加载好之后会触发onInit，所以调用它的api添加一个监听函数，如下代码所示：

```javascript
var maze = new Maze(column, row, canvas);
Module.addOnInit(function(){
    var ptr = Module._Maze_generate(column, row);
    maze.linkedMap = readInt32Array(ptr, column * row);
    maze.draw();
});
```

有两种方法可以得到导出的函数，一种是在函数名前加_，如Module.\_Maze_generate，第二种是使用它提供的ccall或cwrap函数，如ccall：

```javascript
var linkedMapPtr = Module.ccall("Maze_generate", "number", 
                ["number", "number"], [column, row]);
```

第一个参数表示函数名，第二个返回类型，第三个参数类型，第四个传参，或者用cwrap：

```javascript
var mazeGenerate = Module.cwrap("Maze_generate", "number", 
                        ["number", "number"]);
var linkedMapPtr = mazeGenerate(column, row);
```

三种方法都会返回linkedMap的指针地址，可通过Module.get得到地址里面的值，如下代码所示：

```javascript
function readInt32Array(ptr, length) {
    var linkedMap = new Array(length);
    for(var i = 0; i < length; i++) {
        var subptr = Module.getValue(ptr + (i * 4), 'i32');
        var neiborcells = [];
        for(var j = 0; j < 4; j++){
            var value = Module.getValue(subptr + (j * 4), 'i32');
            if(value !== -1){
                neiborcells.push(value, 'i32');
            }
        }
        linkedMap[i] = neiborcells;
    }
  return linkedMap;
}
```

由于它是一个二维数组，所以数组里面存放的是指向数组的指针，因此需要再对这些指针再做一次get操作，就可以拿到具体的值了。如果取出的值是-1则表示不是有效的相邻元素，因此C里面数组的长度是固定的，无法随便动态push，因此我在C里面都初始化了4个，因为相邻元素最多只有4个，初始化时用-1填充。取出非-1的值push到JS的数组里面，得到一个用WASM计算的linkedMap，然后再用同样的方法去画地图。

最后再比较一下WASM和JS生成迷宫的时间。如下代码所示，运行50次：

```javascript
var count = 50;
console.time("JS generate maze");
for(var i = 0; i < count; i++){
    var maze = new Maze(column, row, canvas);
    maze.generate();
}
console.timeEnd("JS generate maze");

Module.addOnInit(function(){
    console.time("WASM generate maze");
    for(var i = 0; i < count; i++){
        var maze = new Maze(column, row, canvas);
        var ptr = Module._Maze_generate(column, row);
        var linkedMap = readInt32Array(ptr, column * row);
    }
    console.timeEnd("WASM generate maze");
})
```

迷宫的规模为50 * 50，结果如下：

![一般规模的迷宫](https://user-gold-cdn.xitu.io/2017/7/31/6f53b3b89e300714b400b358213279c9?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

可以看到，WASM的时间大概快了25%，并且有时候会观察到WASM的时间甚至要比JS的时间要长，这是因为算法是随机的，有时候拆掉的墙可能会比较多，所以偏差会比较大。但是大部分情况下的25%还是可信的，因为如果把随机选取的墙保存起来，然后让JS和WASM用同样的数据，这个时间差就会固定在25%，如下图所示：

![速度快慢的对比](https://user-gold-cdn.xitu.io/2017/7/31/ea775706811f4da02bef3e42004b7394?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

这个时间要比上面的大，因为保存了一个需要拆的墙比较多的数组。理论上不用产生随机数，时间会更少，不过我们的重点是比较它们的时间差，结果是不管运行多少次，时间差都比较稳定。

所以在这个例子里面WASM节省了25%的时间，虽然提升不是很明显，但还是有效果，很多个25%累积起来还是挺长的。

综上，本文用JS和WASM使用连通集算法生成迷宫，并用最短路径算法求解迷宫的路径。使用WASM在生成迷宫的例子里面可以提升25%的速度。