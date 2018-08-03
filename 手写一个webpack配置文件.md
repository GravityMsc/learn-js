# webpack详解

webpack是一个打包工具，它的宗旨是一切静态资源皆可打包。有人就会问为什么要webpack？webpack是现代前端技术的基石，常规的开发方式，比如jquery，html，css静态网页开发已经落后了。 现在是MVVM的时代，数据驱动界面。webpack将现代js开发中的各种新型有用的技术，集合打包。webpack的描述铺天盖地，我就不浪费大家时间了。理解下这个图就知道webpack的生态圈了：

![webpack生态圈](https://user-gold-cdn.xitu.io/2018/7/12/1648c5f3f744e2dd?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

# webpack4.0的配置

```javascript
const path = require('path');	// 引入node的path模块
const webpack = require('webpack');	// 引入的webpack，使用lodash
const HtmlWebpackPlugin = require('html-webpack-plugin');	// 将html打包
const ExtractTextPlugin = require('extract-text-webpack-plugin');	//打包的CSS拆分，将一部分抽离出来
const CopyWebpackPlugin = require('copy-webpack-plugin');
// console.log(path.resolve(__dirname, 'dist));	// 物理地址拼接
module.exports = {
    entry: './scr/index.js', // 入口文件， 在vue-cli中是main.js
    output: {	// webpack如何输出
        path: path.resolve(__dirname, 'dist'),	// 定位，输出文件的目标路径
        filename: '[name].js'
    },
    module: {	// 模块的相关配置
        rules: [	// 根据文件的后缀提供一个loader，解析规则
            {
                test: /\.js$/,	// es6 => es5
                include: [
                    path.resolve(__dirname, 'src')
                ],
                // exclude: []，不匹配项（优先级高于test和include）
                use: 'babel-loader'
            },
            {
                test: /\.less$/,
                use: ExtractTextPlugin.extract({
                    fallback: 'style-loader',
                    use: [
                        'css-loader',
                        'less-loader'
                    ]
                })
            },
            {	// 图片loader
                test: /\.(png|jpg|gif)$/,
                use: [
                    {
                        loader: 'file-loader'	// 根据文件地址加载文件
                    }
                ]
            }
        ]
    }，
    resolve: { // 解析模块的可选项
    	// modules: [] // 模块的查找目录，配置其他的css等文件
    	extensions: [".js", ".json", ".jsx", ".less", ".css"],	// 用到的文件的扩展名
    	alias: {	// 模块别名列表
    		utils: path.resolve(__dirname, 'src/utils')
		}
	},
    plugin: [ // 插进的引用，压缩，分离美化
    	new ExtractTextPlugin('[name].css'),	// [name]默认，也可以自定义name声明使用
        new HtmlWebpackPlugin({	// 将模板的头部和尾部添加CSS和JS模板，dist目录发布到服务器上，项目包可以直接上线
            file: 'index.html',	// 打造单页面运用，最后运行的不是这个	
            template: 'src/index.html'	// vue-cli放在根目录下
        }),
        new CopyWebpackPlugin([	// src下其他的文件直接复制到dist目录下
            { 
                from: 'src/assets/favicon.ico',
                to: 'favicon.ico'
            }
        ]),
        new webpack.ProvidePlugin({	// 引用框架jquery。lodash工具库是很多组件会复用的，省去了import
            '_': 'lodash'	// 引用webpack
        })
    ],
    devServer: {	// 服务于webpack-dev-server，内部封装了一个express
        port: '8080',
        before(app) {
            app.get('api/test.json', (req, res) => {
                res.json({
                    code: 200,
                    message: 'Hello World'
                })
            })
        }
    } 
}
```

# 一、前端环境搭建

我们使用npm或yarn来安装webpack

```bash
npm install webpack webpack-cli -g 
# 或者 
yarn global add webpack webpack-cli
```

为什么webpack会分为两个文件呢？在webpack3中，webpack本身和它的cli以前都是在同一个包中，但在第3版中，他们已经将两者分开来更好地管理它们。

新建一个webpack的文件夹，在其他新建一个try-webpack（防止init时项目名和安装包同名）并初始化和配置webpack。

```bash
 npm init -y  //-y 默认所有的配置
 yarn add webpack webpack-cli -D  //-D webpack安装在devDependencies环境中
```

# 二、部署webpack

在上面搭建好的环境项目中，我们来到package.json里配置我们的scripts：

```json
  "scripts": {
    "build": "webpack --mode production" //我们在这里配置，就可以使用npm run build 启动我们的webpack
  },
  "devDependencies": {
    "webpack": "^4.16.0",
    "webpack-cli": "^3.0.8"
  }
```

配置好我们webpack的运行环境时，联想下vue-cli。平时使用vue-cli会自动帮我们配置并生成项目。我们在src下进行项目的开发，最后npm run build打包生成我们的dist目录。

# 三、npm run build发生了什么

在我们的根项目下try-webpack新建一个src目录。在src目下新建一个index.js文件，在这个文件里面我们可以写任意的代码。写完之后我们在终端运行我们的命令npm run build；你就会发现新增了一个dist目录，里面存放着webpack打包好的main.js文件。

# 四、webpack配置流程篇

我们在开发时一般会打包什么文件呢？我们可以回忆一下，其实vue-cli项目src下就以下几点：

* 发布时需要的html，css，js
* css预编译器stylus，less，sass
* es6的高级语法
* 图片资源.png，.gif，.ico，.jpg
* 文件之间的require
* 别名@等修饰符

那么我将会分这几点来讲解webpack中webpack.config.js的配置。

## HTML在webpack中的配置

在项目的根目录try-webpack下新建webpack.config.js文件，以CommonJS模块化机制向外输出，并且新建一个index.html。

配置我们的入口entry（在vue-cli里相当于根目录下的main.js）和我们的出口output。我们可以把webpack理解为一个工厂，进入相当于把各种各样的原料放进我们的工厂了，然后工厂进行一系列的打包操作把打包好的东西向外输出。

```javascript
const path = require('path');	// 引入我们的node模块里的path
// 测试下 const.log(path.resolve(__dirname, 'dist'));	// 物理地址
```

