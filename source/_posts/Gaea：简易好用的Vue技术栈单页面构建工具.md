---
title: Gaea：简易好用的Vue技术栈单页面构建工具
date: 2019-06-26 11:39:58
tags: [webpack,vue,nodejs]
categories: fe
---

>简介：想了解我们最新开发Vue技术栈的构建工具 `Gaea4.0` 吗？它拥有许多新的特性：`Webpack4`、`Babel7`、京东特色的 `Vue` 组件库 `NutUI2.0` 等等。
## 前言
我们团队在最近两年慢慢从PC端转移到移动端，技术栈从 `JQuery`、`zepto` 等DOM操作转移到 `Vue`、`React` 等新的技术栈。曾经部门的构建工具 `JDF` 已经慢慢退出舞台，现在对于新的技术栈需要新的构建工具 `Butler` 和 `Gaea4.0`。本文主要介绍`Gaea4.0`，它依然需要秉承 `JDF` 的定位，力求解决部门团队减少配置开发环境的工作量，做到开箱即用。又可以做到自定义配置，方便开发者根据实际情况插件式的配置开发环境。包含从开发、调试、上线整个流程的功能。下面将全面的介绍一下 `Gaea4.0` 构建工具。

`Gaea4.0` 从最初的一个 `Webpack` 的模版，慢慢演变成一个Vue2.0技术栈的工程具备了以下几个功能：
- cli脚手架工具
- Webpack模版
- 核心组件库 `NutUI2.0`
- 辅助功能

## cli脚手架工具
从 `Vue-cli` 的整体设计可以学习到cli构建工具和模版相分离的设计思路，我们可以通过命令行，选择匹配不同的模版下载，用于不同的开发需求。分开维护的好处是，模版的变动更新，并不需要用户端重新发布和安装cli构建工具，并且也保证了一致性，当我们有发现模版bug，修复后，也能及时同步远端。
<img src="http://img13.360buyimg.com/uba/jfs/t1/24305/28/4664/11020/5c346336Eb8e72d19/d7c48e8140334392.png" width="640px">
如上图所示，将不同场景的可直接运行的webpack模版代码放在 `git` 上维护，通过 `nodejs` 命令行交互方式获取用户配置项目信息和插件信息，通过 `metadata` 模版渲染方式，将模版和自定义部门相结合，最后输出用户需要的模版工程代码。

### 先了解一下需要的第三库：
commander.js，可以自动解析命令参数，用于处理用户输入的命令。
download-git-repo，下载并提取git仓库，用于下载模版。
Inquirer.js，通用的命令行用户界面集合，用于和用户进行交互。
ora，下载过程loading动画效果。
chalk，可以给终端字体颜色。
log-symbols，在终端上显示对勾或者叉图案。
latest-version，获取库最新版本号。
metalsmith，静态网站生成器，批量处理模版。
handlebars.js，模版引擎，将用户提交的动态信息填充到文件中。

### 一点一滴搭建cli脚手架
第一步：通过 `commander.js` 实现命令init项目
```bash
program.usage('<name>').parse(process.argv)
let projectName = program.args[0]
if(!projectName){
    program.help()
    return
}
```
`gaea init projectName` 也许你觉得命令太长，想短一点，也可以支持 `g2 init projectName`。因为 `commander` 支持`git` 风格的子命令处理，格式是 `[command]-[subcommand]`，例如：

- gaea init => gaea-init
- g2 init => g2-init

所以我在原先的 `gaea-init.js` 文件基础上新增加了一个 `g2-init.js` 文件。

第二步：通过 `Inquirer.js` 命令行问答交互方式配置项目信息和插件信息，在 `Gaea` 中，除了一些基本信息外，还需要选择需要提前打包的第三库，以及一些开发中常用的功能。这里设置的参数名字也是后面模版生成时候占位的名字。
```bash
 const answer = await  new Promise((resolve,reject)=>{
        if(projectRoot != '.'){
            fs.mkdirSync(projectRoot);
        }
        return resolve( inquirer.prompt([
            {
                name:'projectName',
                message:'项目名称',
                default:'gaea-init'
            },
            {
                name:'projectVersion',
                message:'项目版本号',
                default:'1.0.0'
            },
            {	
                name:'bucket',
                type:'checkbox',
                message:'第三方依赖库(多选)',
                validate:(bucketstr)=>{
                    return new Promise((resolve,reject)=>{
                        if(bucketstr.indexOf('vue') === -1){
                            reject('vue 必选！');
                        }else{
                            resolve(true);
                        }
                    })
                },
                choices:
                [{
                    name:'vue',
                    checked:true
                },{
                    name:'vue-router',
                    checked:true
                }]
            },{
                name:'features',
                type:'checkbox',
                message:'支持的功能选择',
                choices:[
                    {
                        name:'NutUI2',
                        checked:true,
                    },
                    {
                        name:'Carefree',
                        checked:true
                    },
                    
                ]
            }
        ]))
    })
```
第三步：通过 `download-git-repo` 去 `git` 库下载模版集合。这里的 `dev` 分支存储了`vue`、`vuex`、`skeleton`、`typescript`四套模版。这里是统一全部都下载下来，然后根据命令行交互的信息进行筛选和处理。
```bash
const download  = require('download-git-repo');
const ora       = require('ora');
const path      = require('path');
module.exports = function(target){
    target = path.join(target || '.','.download-temp')
    return new Promise (function(resolve,reject){
        const url = 'direct:https://github.com/jdf2e/Gaea4.git#dev'
        const spinner = ora('正在下载模版')
        spinner.start()
        download(url,target,{clone:true},(err)=>{
            if(err){
                spinner.fail()
                reject(err)
            }else{
                spinner.succeed()
                resolve(target)
            }
        })
    })
}
```
第四步：`generator.js` 处理所有信息输出最终工程模版。传入 `metadata` 数据、下载的临时文件夹地址、最终模版生成的目录地址。通过 `metalsmith` 和 `handlebars` 处理模版，删除不需要的文件，填入模版动态数据。
```bash
 const res = await generator(context.metadata,context.downloadTemp,dest);
```
因为download的模版是一个集合，那么我们怎么根据 `Inquirer.js` 选择的工程信息匹配到对应的模版，这里还需要一个 `template.ignore` 文件。该文件配置了文件查找的匹配规则，
```bash
{{#if Skeleton}}
skeleton/**
skeleton/.*
{{else if TypeScript}}
ts/**
ts/.*
{{else if vuex}}
vuex/**
vuex/.*
{{else if vue}}
vue/**
vue/.*
{{/if}}
```
```bash
return new Promise((resolve,reject)=>{
        const metalsmith = Metalsmith(process.cwd()).metadata(metadata).clean(false).source(src).destination(dest);
        //判断是否存在ignore文件
        const ignoreFile = path.join(src,'templates.ignore');
        if(fs.existsSync(ignoreFile)){
            metalsmith.use((files,metalsmith,done) =>{
                const meta = metalsmith.metadata();
                const ignores = Handlebars.compile(fs.readFileSync(ignoreFile).toString())(meta)
                .split('\n').filter(item=>!!item.length)
               //删除文件
                done()
            }).build(err=>{
                metalsmith.use((files, metalsmith,done) => {
                    const meta = metalsmith.metadata()
                    Object.keys(files).forEach(fileName => {
                        if(fileName.indexOf('/src/') == -1){ //忽略业务代码
                            let t = files[fileName].contents.toString()
                            files[fileName].contents = new Buffer(Handlebars.compile(t)(meta))
                        }
                       
                    })
                    done();
                }).build(err => {
                    //移动文件
                })
            })
        }
    })
```
第五步：通过 `ora` 、 `chalk` 和 `log-symbols` 美化脚手架
在下载模版时候，或者获取库版本时候，都会比较慢，如果终端没有给出任何信息似乎体验上会比较差，在下载模版的时候，增加了 `ora` 代码：
```bash
const spinner = ora('正在下载模版')
spinner.start()
download(url,target,{clone:true},(err)=>{
            if(err){
                spinner.fail()
                reject(err)
            }else{
                spinner.succeed()
                resolve(target)
            }
        })
```
在最终工程模版生成的时候，可以给出字体颜色表示是否成功，例如：
```bash
console.log(logSymbols.success,chalk.green('创建成功:)'));
console.log(logSymbols.error,chalk.red(`创建失败：${err.message}`));
```
完成上面5个步骤之后，我们看一下脚手架的实际效果：
<img src="https://img12.360buyimg.com/uba/jfs/t1/24654/7/4795/1474509/5c3581bdE7d3955e1/033abed0cbe735b3.gif" width="640px" />

## Webpack 模版
有了命令行工具，我们还需要 `Webpack` 工程模版，在最开始的时候 `Gaea` 只是一个 `Vue` 单页面的 `Webpack` 的模版。用于项目的开发、打包编译、上传。那 `Gaea4.0` 给我们带来哪些新变化 `Webpack4.0` 和 `Babel7`

### Webpack4.0

趁着这次的工具的开发，直接将Webpack从 `3.x` 升级到了 `4.x`。简单罗列一下升级中需要注意的点：

1. 推荐使用 `node` 版本 >=8.9.0，在使用时候可以有更好的体验
2. 借鉴了 `parcel` 零配置构建思想，新增了模式配置：`mode`，可选值是 `production` 和 `development`。`mode` 不可缺省，需要二选一：

####  production 模式：

默认提供所有可能的优化，如代码压缩/作用域提升/ `tree-shaking` 等
不支持watching
`process.env.NODE_ENV` 的值不需要再定义，默认是 `production`

#### development 模式：

主要优化了增量构建速度和开发体验
`process.env.NODE_ENV` 的值不需要再定义，默认是 `development`

在 `Webpack` 配置文件中，我们往往需要拿到当前的 `npm` 命令执行的模式，可以如下拿到参数值：
```bash
"dev": " webpack-dev-server --mode development   -d --open --hide-modules --progress",
 "build": "webpack --mode production --hide-modules --progress",
 "upload": "webpack --mode production --env.upload --hide-modules --progress",
```
```bash
module.exports = (env,argv)=> {
    if(argv.mode === 'production'){
       //判断是否是生产环境
    }
    if(env && env.upload){
      //判断是否是编译上传
    }
}
```
在 `Webpack` 打包优化方面，对于第三方的依赖库，例如`vue`、`vuex`、`vue-router`等我们会单独提前预先 `Dllplugin` 打包 `vendor` 文件。那么可以对比一下构建 `vendor` 的大小和和时间。

`Webpack3` 效果如下：
<img src="https://img30.360buyimg.com/uba/jfs/t1/8505/12/12511/97151/5c35c4cbE495bbc04/36f2287411aa1ab7.jpg" width="640px" />

`Webpack4` 效果如下：
<img src="https://img12.360buyimg.com/uba/jfs/t1/29927/27/4862/82546/5c35bab1Ecd405c5c/58b9db21ba4f3c2f.jpg" width="640px">

从打包上看，对于Vue第三方库全家桶，`4.x` 比 `3.x` 减少了14%。时间也减少了8%。`4.x`在打包编译构建速度上做了很大优化，这里就不展开了，对于整体项目构建提升效果是很明显的。

3. `MiniCssExtractPlugin` 替代 `ExtractTextWebpackPlugin`，如果还想用 `ExtractTextWebpackPlugin`，可以下载 `next` 版本。

### Babel 7

曾经在做移动端页面的时候，为了使用 `es6` 或者 `es7` 语法，又要兼容比较低版本的 `android` 或者 `ios` 等使用场景，我们只能加载完整版本80kb左右的 `polyfill` 文件。`Babel7` 真正做到按目标浏览器兼容的浏览器按需加载 `polyfill文件`。如何做到呢？我们看一下 `.babelrc` 文件。首先配置 `babel preset env`：将 `ES6` 的代码转成 `ES5` (注意： `babel-preset-es2015` 已经被废弃了)。设置目标浏览器的范围，设置  `useBuiltIns:usage`。`usage`
是新特性，可以只打包业务代码中需要转换的 `polyfill`。
```bash
{
    "presets": [
        ["@babel/preset-env",
        {
           "modules":false,
           "targets":{
            "browsers": ["> 1%", "last 2 versions", "not ie <= 8","Android >= 4","iOS >= 8"]
           },
           "useBuiltIns":"usage"
            
        }]
    ],
    "plugins": [
            ["@nutui/babel-plugin-separate-import",{"style":"css"}],
            "@babel/plugin-syntax-dynamic-import"
            
        ]
}
```
测试源码如下：
```bash
 let m = [3,4];
 console.log([...this.spread,...m]);
 let obj = Object.assign({},this.obj); 
 for(let a of m){
       console.log(a);
  }
```
使用 `useBuiltIns:usage` 打包：
<img src="https://img30.360buyimg.com/uba/jfs/t1/19168/5/4868/157044/5c35d0c8E3ceb38fe/a5a5c35ef2576296.jpg" width="640px" />
这结果是不是很让人兴奋。

### 模版集合
当我们搭建完 `Webpack4` 的工程模版后，为了进一步降低用户的使用的难度，从项目的实践中总结和抽取比较完整的模版方案：`Vue`、`Vuex`、`TypeScript`、`Skeleton` 四套模版。目前暂时支持这些，后续考虑加入活动页模版、PC端模版等
<img src="https://img20.360buyimg.com/uba/jfs/t1/17890/31/4915/16782/5c35de96E56cfe0e0/de7b34e8f914c3d7.png" width="640px"/> 

## 核心组件库NutUI2.0
对于一个 `Vue` 技术栈来说，少不了实用的组件库。现在业界有很多优秀的组件 `iView`、`element` 等。不过我们团队做的`NutUI2.0 Vue` 组件库也同样优秀，不仅服务于部门内部，也服务于公司的其他部门团队。目前有30+数量的组件，大多数来源于京东app、京东me等使用场景的实际项目积累和沉淀。基于京东APP 7.0 视觉规范。`Gaea4.0` 是默认安装 `@nutui/nutui` 组件库 `npm` 包和 `@nutui/babel-plugin-seperate-import` 按需加载插件 `npm` 包。在项目中，推荐使用按需加载方式调用组件代码，减小打包体积。同时支持组件内挂载，也支持全局挂载。
```bash
import { DatePicker,Button,Rating } from '@nutui/nutui';
...
export default {
    data(){
        return{
        }
    },
    components: {
        'nut-datepicker':DatePicker,
        'nut-button':Button,
        'nut-rating':Rating
    }
}
```
```bash
import { Cell,Dialog } from '@nutui/nutui';
Vue.use(Cell);
Vue.use(Dialog);

```
## 辅助功能
我们团队开发了许多提高效率插件和功能，`Gaea4.0` 都进行有效整合，方便用户使用。

- 一键上传
- carefree：webpack插件，开发阶段自动编译上传测试服务器，提供二维码真机调试
- smock：webpack插件，开发阶段基于Swagger的自动化mock假数据
- skeleon：利用骨架屏vue组件和vue-server-reander 手写注入骨架屏html和css


## 小结
最后通过cli脚手架工具、`Webpack` 模版、核心组件库 `NutUI2.0` 和辅助功能四个方面的讲解，大家对 `Gaea4.0` 有所了解吧，它力求做一个简易好用的 `Vue` 技术栈构建工具。整体的设计就如下图：
<img src="https://img10.360buyimg.com/uba/jfs/t1/15571/28/4772/23937/5c35e95fEd334d898/f61c2a58eca88a90.png" width="640px"/>
`Gaea4.0` 学习了很多业界前辈分享的经验，结合项目中的实践的经验，最后得出的一点小成果，希望能给大家一点启发和帮助。
欢迎使用和关注
github 地址：https://github.com/jdf2e/Gaea4/ 
npm 地址：https://www.npmjs.com/package/gaea-cli
NutUI2.0 地址：http://nutui.jd.com/


##  相关资源
Butler:http://butler.jd.com
iView: https://www.iviewui.com/
element: http://element-cn.eleme.io
carefree: http://carefree.jd.com
smock: http://smock.jd.com
commander.js: https://github.com/tj/commander.js/
download-git-repo: https://github.com/flipxfx/download-git-repo
Inquirer.js: https://github.com/SBoudrias/Inquirer.js/
ora: https://github.com/sindresorhus/ora
chalk: https://github.com/chalk/chalk
log-symbols: https://github.com/sindresorhus/log-symbols
latest-version: https://github.com/sindresorhus/latest-version
metalsmith: https://github.com/segmentio/metalsmith
handlebars.js: https://github.com/wycats/handlebars.js/






