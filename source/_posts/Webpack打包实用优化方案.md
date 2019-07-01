---
title: Webpack打包实用优化方案
date: 2019-07-01 21:03:30
tags: [webpack,vue]
categories: fe
---

 ## 简介: Webpack是当下最热门的前端资源模块化管理和打包工具。如何提升打包效率，成为了大家关注的点之一。下文将分享我们开发到上线的实用优化方案。


目前最火的打包工具莫过于 `Webpack` 了，关于 `Webpack` 的优化方案，网上有很多文章可以供大家参考。查阅前人的资料，总结自己在项目中遇到的问题，最终得出一些比较实用的优化方案，本文将与大家一一分享。既然是打包优化，那么我们需要时刻关注以下几点：
- 减少编译时间
- 减少编译输出文件大小
- 提高页面性能

## Webpack3.0 的 Scope Hoisting (作用域提升)
 
`Webpack3.0` 最大的一个新功能就是 `Scope Hoisting` （作用域提升）。曾经的 `Webpack` 是将每一个模块（每一个被 `import` 或者 `require` 的代码）`bundle` 到每一个独立的闭包函数里。这会导致 `bundle` 的文件里，每一个模块外层都会有一些特殊的闭包包装，导致文件增大，同时它也使得编译后的 js 文件在浏览器中的执行效率降低。

虽然，理论知识已经了解，但我们还是用实际操作来验证一下：

开启 `Scope Hoisting` 非常简单，因为 webpack 已经内置了这个功能，在插件入口处新增 new webpack.optimize.ModuleConcatenationPlugin()。

```  
plugins:[
		new webpack.optimize.ModuleConcatenationPlugin()
	]

```

我们简单的编写两个文件 `app.js` 和 `timer.js`。

```app.js
import timer from './timer'

var time = (new Date()).getTime();
timer();
console.log('hello webpack'+time);
```

```timer.js
export default function bar(){
	console.log('pig');
}

```

```webpack.config.js
var path = require('path');
var webpack = require('webpack');
var config = {
	entry:{
		app:'./src/app.js'
	}, 
	output:{
		filename:'bundle.js',
		path: path.resolve(__dirname,'./build')
	},
	plugins:[
		//new webpack.optimize.ModuleConcatenationPlugin()
	]
}
module.exports = config;
```
执行编译后，结果如下：`bundle.js` 大小是3.03KB

<img src="http://img14.360buyimg.com/uba/jfs/t19813/103/1136417137/71729/e6115fc2/5b1651ebNa772b49a.jpg" width="320" />

<img src="http://img20.360buyimg.com/uba/jfs/t20593/53/712853791/272783/6d73539b/5b1652c7N32dbc218.jpg" width="320" />

我们把 new webpack.optimize.ModuleConcatenationPlugin() 打开，编译结果如下：

<img src="http://img12.360buyimg.com/uba/jfs/t21550/277/735652064/63914/a2417e89/5b165231N0fd97b21.jpg" width="320" />

<img src="http://img20.360buyimg.com/uba/jfs/t21052/363/751826920/189366/32856e8b/5b1652e0N3b34e6d1.jpg" width="320" />

对比两个的区别大家可以发现，在 `bundle.js` 中 `Webpack2.0` 比 `Webpack3.0` 多了以下的代码：

```
(function(module, __webpack_exports__, __webpack_require__) {
...
});
```
`timer.js` 和 `app.js` 都在一个函数中了，`timer.js` 并没有编译在闭包函数中。一个模块就少了一个闭包函数，那么多引用几个，就可以少很多了。从体积上的确可以看出明显减少。除了这一效果之外，这个内置优化能使得编译后的代码在浏览器中执行效率显著提高。在 Aaron Hardy 的 Optimizing Javascript Through Scope Hoisting[1]的文章中表明，在一个真实的 Tubine[2] 的项目中，作者对比了使用 `Scope Hoisting`  和不使用的打包大小和js在浏览器执行效率。`Turbine` 压缩的gzip文件大小减少了约41%，提高了初始化执行时间约12%。
看到这里，是不是很心动，如果能用在我们项目中，那很完美了。可惜现实比较残酷，下面我们看看它的局限性。
我将 demo 例子的引用模块改成 `CommonJs` 的语法，执行效果如下：

```app.js
//import bar from './timer'

var bar = require('./timer');

var time = (new Date()).getTime();
bar();
console.log('hello webpack'+time);
```

```timer.js

// export default function bar(){
// 	console.log('pig');
// }

exports.bar = function(){
	console.log('big');
}
```
<img src="http://img12.360buyimg.com/uba/jfs/t20380/336/705657242/329273/71167868/5b165328Nf8dfa56d.jpg" width="320" />

你会发现用 `CommonJS` 的模块语法在开启 `Scope Hoisting` 时候，编译打包的 `bundle` 并没有发生改变。因为目前 `webpack3.0` 只支持 `ES Module` 的模块语法。联想一下，你在自己的脚手架中，大多的NPM依赖包都还是 `CommonJS` 的语法，`Webpack` 会回退到原打包模式。你在做升级处理的时候，可以使用 `--display-optimization-bailout` 查看被降级原因。

<img src="http://img13.360buyimg.com/uba/jfs/t20368/302/712478610/199566/30035843/5b165ad5Nfc02c132.jpg" width="320" />

除了目前支持 `ES Modlue` 的语法外，也许你的老代码中还有以下几点：
- 使用了ProvidePlugin[3]
- 使用了eva()函数
- 项目有多个entry

就目前前端生态环境来看，`Webpack 3.0` 的 `Scope Hoisting` 的新特性暂时无法使用，不过 `ES Module`  是趋势，未来模块的引用的方式肯定被ES Module所取代。

## CommonsChunkPlugin 的使用

`Webpack3.0` 的 `Scope Hoisting` 在实际项目中用不了，那我们来看看 `CommonsChunkPlugin` 这个插件的使用。 `CommonChunkPlugin`  插件是一个可选的用于建立一个独立文件（又称作 `chunk` ）的功能，这个文件包含多个入口 `Chunk` 的公共模块（ `CommonsChunkPlugin` 已经从 `webpack v4 legato` 
 中移除了，想要了解最新版本中如何处理 chunk ，可以查看 `SplitChunksPlugin` [4]。
`CommonsChunkPlugin` 优化的思路就是通过将公共模块拆出来，最终合成的文件能在最开始的是加载一次，便于后续访问其余页面，直接使用浏览器缓存中的公共代码，这样无疑体验会更好。
理论知识知道了，那么我们就动手来试试，是不是效果不错。我们搭一个基于 `Webpack2.7.0` 的简单的 `Vue` 脚手架：

<img src="http://img14.360buyimg.com/uba/jfs/t20617/12/736812907/120364/877f6e27/5b1665b0Nf0e26481.jpg" width="320" />

```diff 
const path = require('path');
const webpack = require('webpack');
const configw = require('./package.json');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const ExtractTextPlugin = require('extract-text-webpack-plugin');
const CleanWebpackPlugin = require('clean-webpack-plugin');
const { VueLoaderPlugin } = require('vue-loader')
var config = {
	entry:{
		app:'./src/app.js'
	}, 
	output:{
		path: path.resolve(__dirname, 'build'),
	    publicPath: configw.publicPath + '/',
	    filename: 'js/[name].[chunkhash].js'
	},
	plugins:[
		new CleanWebpackPlugin('build'),
		new HtmlWebpackPlugin({
	        template: './src/index.html'
	    }),
	    new ExtractTextPlugin({
	        filename: 'css/app.css'
	    }),
+	    new webpack.optimize.CommonsChunkPlugin({
+	    	name:'vender',
+	    	minChunks: function(module) {
+	        return (
+	          module.resource &&
+	          /\.js$/.test(module.resource) &&
+	          module.resource.indexOf(
+	            path.join(__dirname, './node_modules')
+	          ) === 0
+	        )
+	      }
+
+	    }),
	    new VueLoaderPlugin(),
	],
	module:{
		rules: [
				{
		        test: /\.css$/,
		        use: ExtractTextPlugin.extract({
		            fallback: "style-loader",
		            use: ['css-loader']
		        }),
		    },
		    {
		        test: /\.scss$/,
		        use: ExtractTextPlugin.extract({
		            fallback: 'style-loader',
		            use: ['css-loader', 'sass-loader']
		        })
		    },
		    {
		        test: /\.vue$/,
		        loader: 'vue-loader',
		        options: {
		            
		        }
		    },
		    {
		        test: /\.js$/,
		        loader: 'babel-loader',
		        exclude: /node_modules/,
		        query: { 
		          presets: ['env']
		        }
		      }
	     ]
	}

}
module.exports = config;
```
为了方便查看，并没有引入压缩插件什么的做优化。目前来看，在业务js中的依赖于 `node_modules` 中的 `vue` 、`vue-router` 、`axios` 等第三方公共库都被抽离出 `app.js` 打包在 `vender.js` 中。为了能做到业务js和第三方库js 相分离，做到浏览器端缓存不会频繁更新的js迈出了第一步。可惜的是，如果我们改动业务代码比如，`app.js` 、`app.vue` 、`index.vue`等业务代码，你会发现除了 `app.js` 的哈希值发生了变化，连没有做更改的 `vender.js` 哈希值都变了。

<img src="http://img13.360buyimg.com/uba/jfs/t20794/191/721701067/147081/79c1ead5/5b16661bN89719c5c.jpg" width="320" />

<img src="http://img11.360buyimg.com/uba/jfs/t21715/334/735840956/144136/ad072bee/5b16661dN5b0dd810.jpg" width="320" />

哈希值变了，就说明文件的内容变了。这一定会让你很奔溃，你想做到的 `Webpack` 构建持久化缓存 js 的功能基本是不可能的。找一下原因吧，没有更改依赖文件，为什么只改动业务 js，`vender.js` 也会变呢？因为每次构建时候，`Webpack` 会生成 `webpack runtime` 代码，用来帮助 `Webpack` 完成其工作，比如在模块交互时，链接模块所需的加载和解析逻辑。如下图所示，在我们的脚手架中，编译后的结果对比，就只有一个运行时产生的哈希值的值不一样。

<img src="http://img10.360buyimg.com/uba/jfs/t21577/57/718994465/322573/a098daa4/5b166660N99c0b0d1.jpg" width="320" />

既然差别这么小，那么我们把 `runtime` 这部分代码抽离出来。这样就能持久化缓存 `vender.js` 了。我们试一下。

```webpack.config.js + manifest
new webpack.optimize.CommonsChunkPlugin({
	    	name:'vender',
	    	minChunks: function(module) {
	        return (
	          module.resource &&
	          /\.js$/.test(module.resource) &&
	          module.resource.indexOf(
	            path.join(__dirname, './node_modules')
	          ) === 0
	        )
	      }
	    }),
	    new webpack.optimize.CommonsChunkPlugin({
	    	name:'manifest',
	    	minChunks:Infinity
	    })
```
修改了以下 `app.vue` 中的代码，前后的编译结果如下：

<img src="http://img11.360buyimg.com/uba/jfs/t21175/1/733999587/193561/85a64d60/5b1666f8N34c3f716.jpg" width="320" />

<img src="http://img14.360buyimg.com/uba/jfs/t22192/295/751522277/198424/d7dae77a/5b16672cNa87b4080.jpg" width="320" />

`vender.js` 的哈希值如预期一样，没有发生变化。因为文件内变化的代码已经被抽离出到 `manifest` 这个文件中了。`manifest` 里面存储了`chunks`映射关系，有了`chunks`的映射，我们才知道要加载的`chunk`的真实地址。那么每次修改业务 js，都不需要部署 `vender.js` 了。这样就达到了第三依赖库实现了用户端和服务端持久缓存。每次上线更新也就部署较小的 `app.js` 和 `manifest.js` 。不过需要注意的是 `manifest` 必须先加载。
那么直接在生产环境使用这种部署策略，不，还不行。我们的目标只有一个实现第三方库的在持久化缓存，但不能给我们的上线带来风险。要保证你不上线第三方库，用户直接访问本地浏览器缓存中的第三方库，即使上线更新业务 js 依然可以正常运行。`CommonsChunkPlugin` 这种打包方式是在运行时编译出的代码，如果我们在业务代码里新增或者删除依赖，试想一下，你项目采用此方式上线后，你对项目也许做优化或者功能模块增加，业务 js 中的对模块依赖部分的代码难免有变动。我们来对比一下保留 `index.vue` 中的依赖和删除依赖的结果：

<img src="http://img10.360buyimg.com/uba/jfs/t19894/46/1152230345/192987/1268444a/5b166967Nb1a60f38.jpg" width="320">

<img src="http://img13.360buyimg.com/uba/jfs/t21229/160/713972814/189853/ad244d7e/5b166834N796e1f17.jpg" width="320" />


很明显我修改了业务代码中的模块依赖，导致了 `vender.js` 库也发生了变化，这是没法避免的。因为 `vender.js` 和 `app.js` 是紧密耦合在一块的，你虽然把 `runtime` 的代码抽离到 `manifest` 中。对引入模块的删除和新增会导致在运行时编译的模块的id依赖发生变化。我对比了两个 `vender.js`  的区别，如下图所示，主要是引用的模块 id 发生了变化。

<img src="http://img12.360buyimg.com/uba/jfs/t21037/39/750124658/1071991/cebe6d59/5b16699bN1d4d4b3d.jpg" width="320" />

看到这里，`CommonsChunkPlugin` 虽然能解决问题，但是上线风险无法避免。这样做是不值得，为了利用浏览器缓存，从而只上线 `app.js` 文件。很难保证上线无问题。最主要问题是，我们在 `build` 完提交测试还是把 `vender.js` 一并提交的。当然有人会说你看哈希值不就知道需不需要上线 `verder.js` 了。可是我上面提到的场景在业务代码中的修改还是很常见的，不值得再上线一次 `vender.js`。
说了这么多，就是为了告诉我们都不行！ `vender.js` 实现持久化缓存。

## DllPlugin 和 DllReferencePlugin

这两个 `Webpack` 打包插件，`Dllplugin` 会打包出一个dll文件和一个 `manifest.json` 模块引用的映射文件。dll文件放什么呢，是我们的第三方库依赖。这样就好比是 Windows 的动态链接库。`Dllplugin` 的使用思路是将我们项目中的公共的第三方库打包到一个dll文件中。它是静态的，除非你手动修改在项目中需要引入的库。同时也会编译出一个`manifest.json` 的映射文件。它也是静态的，里面存储了通过id值映射值找到在dll文件中对应的库 js。`DllReferencePlugin` 则是将映射值打包进我们的业务 js 了。这样就可以完完全全的提前抽离了第三方依赖库。之后，只会打包编译业务部分的代码，再也不用去重复构建第三方库 js。构建编译的时间会大大减少。

我们还是通过实践的方式来证明吧：

首先我们配置一份生成dll的 `config` 。

```diff
const path = require("path");
const webpack = require("webpack");
const UglifyJsPlugin = require('uglifyjs-webpack-plugin');
const config = require('./package.json');
const curDate = new Date();
const curTime = curDate.getFullYear() + '/' + (curDate.getMonth() + 1) + '/' + curDate.getDate() + ' ' + curDate.getHours() + ':' + curDate.getMinutes() + ':' + curDate.getSeconds()
const bannerTxt = config.name + ' ' + config.version + ' ' + curTime;
module.exports = {
	//你想要打包的模块数组
	entry:{
		vendor:['vue','axios','vue-router','qs']
	},
	output:{
		path:path.join(__dirname,'/static/'),
		filename:'[name].dll.js',
		library:'[name]_library'
		//vendor.dll.js 中暴露出的全局变量
		//主要是给DllPlugin中的name 使用
		//故这里需要和webpack.DllPlugin 中的 'name :[name]_libray 保持一致
	},
	plugins:[
+		new webpack.DllPlugin({
+			path:path.join(__dirname,'.','[name]-manifest.json'),
+			name:'[name]_library',
+			context:__dirname
+		}),
		new UglifyJsPlugin({
            cache:true,
            sourceMap:false,
            parallel:4,
            uglifyOptions: {
                ecma:8,
                warnings:false,
                compress:{
                    drop_console:true,
                },
                output:{
                    comments:false,
                    beautify:false,
                }
            }
        }),
        new webpack.BannerPlugin(bannerTxt)
	]
}
```
`entry` 配置了常见的 `Vue` 全家桶系列。因为几乎每个页面都需要用到它们，把它们提到公共的 `vender.js`  中是再好不过的事了。我们看一下运行结果。我配置了 npm script 执行代码 npm run dll：

```
"scripts": {
    "dev": "webpack-dev-server -d --open --progress",
    "build": "cross-env NODE_ENV=production webpack --hide-modules --progress",
    "upload": "cross-env NODE_ENV=upload webpack --hide-modules --progress",
    "dll": "webpack --config ./webpack.dll.config.js"
  }
```

<img src="http://img30.360buyimg.com/uba/jfs/t20371/56/758322565/84300/cd2cc419/5b166ad3N54905d1a.jpg" width="320" />

压缩过的 `dll.js` 的大小还是可以接受的。我们看看生成的 `manifest.json` 里面都存储了什么。

<img src="http://img20.360buyimg.com/uba/jfs/t21850/197/722220053/277331/284e712f/5b166b27Naa5f07ef.jpg" width="320" />

跟预期的一样，里面存储了引用映射路径和对应的id值。`dll.js` 和 `manifest.json` 只需要编译一次。之后我们开发业务代码和上线打包都不需要再次编译打包 `vender.dll.js` 了。我们看一下 `webpack.config.js` 中如何配置的。

```diff
const webpack = require('webpack');
const config = require('./package.json');
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const ExtractTextPlugin = require('extract-text-webpack-plugin');
const CopyWebpackPlugin = require('copy-webpack-plugin');
const CleanWebpackPlugin = require('clean-webpack-plugin');
const UglifyJsPlugin = require('uglifyjs-webpack-plugin');
const autoprefixer = require('autoprefixer');
const htmlwebpackincludeassetsplugin = require('html-webpack-include-assets-plugin');
const AddAssetHtmlPlugin = require('add-asset-html-webpack-plugin');

const webpackConfig = module.exports = {};
const isProduction = process.env.NODE_ENV === 'production';
const isUpload = process.env.NODE_ENV === 'upload';
const curDate = new Date();
const curTime = curDate.getFullYear() + '/' + (curDate.getMonth() + 1) + '/' + curDate.getDate() + ' ' + curDate.getHours() + ':' + curDate.getMinutes() + ':' + curDate.getSeconds();
const bannerTxt = config.name + ' ' + config.version + ' ' + curTime; //构建出的文件顶部banner(注释)内容
webpackConfig.entry = {
    app: './src/app.js',
};

webpackConfig.output = {
    path: path.resolve(__dirname, 'build' + '/' + config.version),
    publicPath: config.publicPath + '/'+config.version+'/',
    filename: 'js/[name].js'
};

webpackConfig.module = {
    rules: [{
        test: /\.css$/,
        use: ExtractTextPlugin.extract({
            fallback: "style-loader",
            use: ['css-loader', 'postcss-loader']
        }),
    }, {
        test: /\.scss$/,
        use: ExtractTextPlugin.extract({
            fallback: 'style-loader',
            use: ['css-loader', 'sass-loader', 'postcss-loader']
        })
    }, {
        test: /\.vue$/,
        loader: 'vue-loader',
        options: {
            extractCSS: true,
            postcss: [require('autoprefixer')()]
        }
    }, {
        test: /\.js$/,
        loader: 'babel-loader',
        exclude: /node_modules/,
    }, {
        test: /\.(png|jpg|gif|webp)$/,
        loader: 'url-loader',
        options: {
            limit: 3000,
            name: 'img/[name].[ext]',
        }
    }, ]
};

webpackConfig.plugins = [
    new webpack.optimize.ModuleConcatenationPlugin(),
    new CleanWebpackPlugin('build'),
    new HtmlWebpackPlugin({
        template: './src/index.html'
    }),
    new ExtractTextPlugin({
        filename: 'css/app.css'
    }),
    new CopyWebpackPlugin([
        { from: path.join(__dirname, "./static/"), to: path.join(__dirname, "./build/lib") }
    ]),
+    new webpack.DllReferencePlugin({
+        context:__dirname,
+        manifest:require('./vendor-manifest.json')
+    })
];

if (isProduction || isUpload) {

    webpackConfig.plugins = (webpackConfig.plugins || []).concat([
        new webpack.DefinePlugin({
            'process.env': {
                NODE_ENV: '"production"'
            }
        }),
        new webpack.LoaderOptionsPlugin({
            minimize: true
        }),
        new UglifyJsPlugin({
            cache:true,
            sourceMap:false,
            parallel:4,
            uglifyOptions: {
                ecma:8,
                warnings:false,
                compress:{
                    drop_console:true,
                },
                output:{
                    comments:false,
                    beautify:false,
                }
            }
            
        }),
        new htmlwebpackincludeassetsplugin({
            assets:['/lib/vendor.dll.js'],
            publicPath:config.publicPath,
            append:false
            
        }),
        new webpack.BannerPlugin(bannerTxt)
    ]);
   
} else {
    webpackConfig.output.publicPath = '/';
    webpackConfig.devtool = '#cheap-module-eval-source-map';
    webpackConfig.plugins = (webpackConfig.plugins || []).concat([
         new AddAssetHtmlPlugin({
            filepath:require.resolve('./static/vendor.dll.js'),
            includeSourcemap:false,
            
        })
    ]);
    webpackConfig.devServer = {
        contentBase: path.resolve(__dirname, 'build'),
        compress: true, //gzip压缩
        historyApiFallback: true,
    };
}
```
我们用 `DllReferencePlugin` 把生成好的 `manifest.json` 映射文件引入到正式的业务代码打包中。

<img src="http://img10.360buyimg.com/uba/jfs/t21985/19/728283776/90839/6552f5c6/5b166b78N5e225eb1.jpg" width="320" />

`app.js` 只有7.45KB，`vender.dll.js` 被拷贝到 `build` 目录下 `lib` 文件夹下。

<img src="http://img11.360buyimg.com/uba/jfs/t22381/162/757531011/30871/b1b17e56/5b166b80N5653be55.jpg" width="320" />

所有业务的代码则都在版本控制文件夹下，`vender.dll.js`  放置在 lib 文件夹下。每次上线如果有版本变化只要上线业务js就行。不需要上线 lib 文件夹。只要你不手动修改 `webpack.dll.config.js` 的entry

```
entry:{
		vendor:['vue','axios','vue-router','qs']
	}
```
就永远不会发生变化。
还是对比一样优化之前和优化之后的：

<img src="http://img14.360buyimg.com/uba/jfs/t21859/333/725930307/108242/973b4f23/5b166c4aNac9af0cc.jpg" width="320" />

<img src="http://img10.360buyimg.com/uba/jfs/t21985/19/728283776/90839/6552f5c6/5b166b78N5e225eb1.jpg" width="320" />

优化之前 `app.js` 由于有打包了第三方库所以有116KB，优化之后，抽离第三库 `app.js` 只有7.45KB。只是项目开始时候，需要提前打包一份dll文件。以后每次编译时间都比之前少了将近一半。这个还只是一个脚手架demo。等到用在实际项目中的构建，效果更加明显。

#总结

有很多人不建议使用 `DllPlugin` ，觉得没必要把所有公共的打包在一起，放在首屏就加载，这样使得首屏加载时间过长之类的，还有觉得多了一份`config` 增加了工作量。不过我个人觉得，对于像 `React` 和 `Vue` 这种全家桶系列的，整体性偏强的技术栈。抽离出全家桶放置在 `vender.js` 中还是很有必要的。因为几乎每个页面都会用到。而且，他们是完全跟业务逻辑无关的第三方库。对它们实现持久化缓存，对于开发者和用户的体验都会大大提升。一点脚手架搭建的心得，感谢各位的浏览，有任何问题欢迎讨论哈～


## 扩展阅读：
[1]ptimizing Javascript Through Scope Hoisting:https://medium.com/launch-by-adobe/optimizing-javascript-through-scope-hoisting-47c132ef27e4
[2]Tubine:https://github.com/Adobe-Marketing-Cloud/reactor-turbine
[3]ProvidePlugin:https://webpack.js.org/plugins/provide-plugin/
[4]SplitChunksPlugin:https://webpack.js.org/plugins/split-chunks-plugin