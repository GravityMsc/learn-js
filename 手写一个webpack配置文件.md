# webpack详解

webpack是一个打包工具，它的宗旨是一切静态资源皆可打包。有人就会问为什么要webpack？webpack是现代前端技术的基石，常规的开发方式，比如jquery，html，css静态网页开发已经落后了。 现在是MVVM的时代，数据驱动界面。webpack将现代js开发中的各种新型有用的技术，集合打包。webpack的描述铺天盖地，我就不浪费大家时间了。理解下这个图就知道webpack的生态圈了：

![webpack生态圈](https://user-gold-cdn.xitu.io/2018/7/12/1648c5f3f744e2dd?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

# webpack4.0的配置

```javascript
const path = require('path');	// 引入node的path模块
const webpack = require('webpack');	// 引入的webpack，使用lodash
const HtmlWebpackPlugin = require('html-webpack-plugin');	// 将html打包
const ExtractTextPlugin = require('extract-text-webpack-plugin');	//打包的CSS拆分，将一部分抽离出来
const CopyWebpack
```

