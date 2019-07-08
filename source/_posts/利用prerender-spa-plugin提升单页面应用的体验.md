---
title: 利用prerender-spa-plugin提升单页面应用的体验
date: 2019-07-08 14:24:36
tags: [webpack,vue]
categories: fe
---

目前 `Vue`、`React` 在前端界混的风生水起，它们的开发思想使得我们能真正做到前后端分离、解耦。单页面的使用给用户带来了更好体验。不过对于 `Vue` 和 React 这种框架来说，`HTML in JS` 的思路在首屏加载慢、白屏以及 `SEO` 等问题就日益突出了。

不仅需要拼框架的功能、生态，当然还不能忘记“用户至上的原理“，拼体验。孜孜不倦的前端朋友们给出了几个解决方案：1.Server-side rendering（SSR），2.Prerendering。下面我将一一介绍一下。

# 什么是SSR？
SSR 直译就是服务端渲染，通过设置 `SSR`，你就可以在后台的 `Node.js` 环境中完成渲染逻辑，然后将 `HTML` 视图直接返回给客户端。这样你不仅可以使用 `Vue` 和 `React` 技术，而且可以直出页面内容。而非只有一个空壳子在后端那里。这样也方便了搜索引擎的蜘蛛获取页面，解决 `SEO` 问题。

# SSR有什么问题？
你可以回想一下我们在很早之前的前端开发模式，其实就是后端直出页面。前端重构页面，交由后端套页面渲染首页的数据。当然一些异步的数据，则通过 `ajax` 获取数据渲染。这似乎又回到了之前的开发模式。前端和后端还是紧密联系在一起了。给维护和迭代带来的不便。

那它没有好处吗？有的！目前，社会上还是有成熟的框架和线上的产品，比如 `Nuxt.js`[1]，<img src="http://img20.360buyimg.com/uba/jfs/t1/2629/26/105/12144/5b8f6f08Efa34770f/8a3b350966c50335.png" width="300" /> ，
说明它还是具有价值的。它很明显可以解决首屏白屏或者SEO等问题。但是它也引入很多问题，其一，对工程师要求较高，需要同时掌握的前后端知识。其二，要考虑在服务端 `Node.js` 环境中的内存泄露、运行速度、并发压力等问题。node 层的服务可以用“脆弱”两个字来形容。其三，开发成本增加，研发周期变长等。一般来说，是需要一个强大且稳定的基础架构来支撑服务端的压力。

如果你可以应付这些，无疑 `SSR` 对于增强应用体验是非常棒的~，但对于像我这样有点焦虑的人来说，是否有其他解决办法呢？有的， `Prerendering`！

# Prerendering
有时候，我们开发的单页面应用也就几个页面，非常小型，仅仅是为了 `SEO`、首页白屏问题，大家都觉得有点校枉过正了。可以利用第三方插件 `prerender-spa-plugin`[2]，在客户端实现渲染，这样无需将 `Vue` 或者 `React` 代码部署在服务端。`prerender-spa-plugin` 是 `Webpack` 的插件，它可以编译应用中的所有静态页面，轻而易举的建立对应的索引路径。下面结合 `Vue.js` 和 `prerender-spa-plugin` 来解决前面所提出的的问题。



## 安装

```
 npm install prerender-spa-plugin --save-dev
```

## 使用

```webpack.config.js
const PrerenderSPAPlugin = require('prerender-spa-plugin');
const Renderer = PrerenderSPAPlugin.PuppeteerRenderer;

module.exports = {
plugins: [
 new PrerenderSPAPlugin({
      staticDir: path.join(__dirname, 'dist'),
      routes: [ '/', '/about', '/contact' ],
      renderer: new Renderer({
        inject: {
          foo: 'bar'
        },
        renderAfterDocumentEvent: 'render-event'
      })
    })
  ])
]
}
```
`staticDir` 指的是预渲染输出的页面地址，`routes` 指的是需要预渲染的路由地址，`renderer` 则是所采用的渲染引擎是什么，目前用的是 `V3.4.0` 版本支持 `PuppeteerRenderer`。`inject` 则是预渲染过程中都能拿到的值，该值提供给你了机会，让你觉得是否渲染这部分代码。例如下面的代码，是不会被预渲染进  `HTML ` 中的。

```
showMessage(){
      if(window.__PRERENDER_INJECTED && window.__PRERENDER_INJECTED.foo =='bar') return;
      this.message = '我是测试预加载拦截';
    }
```
 `renderAfterDocumentEvent ` 这个则很关键，这个是监听  `document.dispatchEvent ` 事件，决定什么时候开始预渲染。

```
new Vue({
  el: '#app',
  router,
  render: h => h(App),
  mounted () {
    // You'll need this for renderAfterDocumentEvent.
    document.dispatchEvent(new Event('render-event'))
  }
});

```

## 实例

具体可以看一下官方的 `Vue.js 2.0 + vue-router Prerender SPA Example`[3] 实例。

## 原理及缺点
`prerender-spa-plugin` 利用了 `Puppeteer`[4] 的爬取页面的功能。`Puppeteer` 是一个 `Chrome` 官方出品的 `headless Chrome node` 库。它提供了一系列的 API, 可以在无 UI 的情况下调用 `Chrome` 的功能, 适用于爬虫、自动化处理等各种场景。它很强大，所以很简单就能将运行时的 `HTML` 打包到文件中。原理是在  `Webpack` 构建阶段的最后，在本地启动一个 `Puppeteer` 的服务，访问配置了预渲染的路由，然后将 puppeteer 中渲染的页面输出到 `HTML` 文件中，并建立路由对应的目录。

<img src="http://img10.360buyimg.com/uba/jfs/t1/2175/24/636/15603/5b92270cE9eaa9c11/ece6cbda01397a83.jpg" width="300"/>

利用官方的实例进行编译结果如下：
<img src="http://img10.360buyimg.com/uba/jfs/t1/2701/34/630/37704/5b92332aE3717fa17/8ad8c070f67e0a48.jpg"  width="300" />
每个对应的路都有一个对应的静态 `HTML`。每一个 `HTML` 内除了
```
<div id="app"></div>
```
这个 `Vue` 的挂载元素外，还有静态的标签内容。

```
<!DOCTYPE html><html lang="en"><head>
  <meta charset="utf-8">
  <title>PRODUCTION prerender-spa-plugin</title>
<link rel="shortcut icon" href="/favicon.ico"><style type="text/css"></style><style type="text/css">#app{font-family:Avenir,Helvetica,Arial,sans-serif;-webkit-font-smoothing:antialiased;-moz-osx-font-smoothing:grayscale;text-align:center;color:#2c3e50;margin-top:60px}h1,h2{font-weight:400}ul{list-style-type:none;padding:0}li{display:inline-block;margin:0 10px}a{color:#42b983}</style></head>
<body>
  <div id="app"><div><img src="/logo.png?82b9c7a5a3f405032b1db71a25f67021"> <h1>Welcome to your prerender-spa-plugin Vuejs 2.0 demo app!</h1> <p>汪楠大大about页</p> <p><a href="/" class="router-link-active">Home</a> <a href="/about" class="router-link-exact-active router-link-active">About</a> <a href="/contact" class="">Contact</a></p> <ul></ul> <a href="javascript:;">点击我，看看有什么效果</a> <p>最好不要点我</p></div></div>
<script type="text/javascript" src="/build.js"></script>

</body></html>
```

既然有了每个路由对应的 `HTML`，那么对应 `SEO` 优化应该不成问题了。我们可以更改 `title` 、`meta` 。而且页面的内容都已经在 `HTML` 中直接呈现，就不会有因为 `js` 等资源加载慢导致白屏的问题。
`prerender-spa-plugin` 的确在一定程度上解决了我们对于 `SEO` 的诉求和页面加载慢的问题。但是它的缺点还是很明显的。
- 不同的用户看到不同的页面，动态数据页面
- 经常发生变化的页面，数据实时性展示（比如体育比赛等）
- 路由过多，构建时间过长

# 正确使用的姿势
聊了这么多，大家都是为了给用户更好的体验。基本上我们做的页面都是强依赖动态数据展示，如果渲染的静态页面和最后呈现的页面之前切换并不自然，那么体验是很差的。我们不能为了解决 `SEO` 或者加载慢的问题，引入新的问题。那到底该怎么做呢？其实很简单：`Loading` 或者 骨架屏。这两个都是将过渡的 `HTML` 片段插入到 `<div id="app"></div>` 中。一旦js加载完，vue开始渲染正式页面时候，就将把过渡的 `HTML` 片段干掉了。看一下下面的代码

```
body>
  <div id="app"><div><div class="j-loading-wrap"><div class="j-mask"></div> <div class="j-loading"><img src="/joy_loading.gif?b494ac2f480615dc87d8797cb1a712da"></div></div> <!----></div></div>
<script type="text/javascript" src="/build.js"></script>

</body>
```
<img src="http://img13.360buyimg.com/uba/jfs/t1/5395/27/1611/120060/5b9491c9Ec65037fb/adbcbce577c357c5.gif" width="240"/>

```
<skeleton-loading >
    <row 
        :gutter="{
            bottom: '0.1rem'
        }">
        <column :span="'24'">
            <square-skeleton 
                :boxProperties="{
                    height: '0.3rem'
                }"    
            />
        </column>
    </row>
    <row>
        <square-skeleton 
            :boxProperties="{
                height: '3.1rem'
            }"    
        />
    </row>
<skeleton-loading >
```
<img src="http://img10.360buyimg.com/uba/jfs/t1/4754/11/8674/1329268/5ba9db10E5a5f52b8/88a9988ec901c0c1.gif"  width="300"/>

`Loading` 或者骨架屏只是为了增强用户体验，那么跟本文的主体有什么关系呢？有关系，一般在做过渡效果的时候，很多都是手写代码，这样既不利于维护，也不利于统一标准。我们可以使用 `prerender-spa-plugin` 进行操作，它能把整个页面都生成静态文件输出到 `HTML`，那么对于我们的 `Loading` 组件或者页面首屏的`DOM` 和样式自动化的输出到 `HTML` 中，不是轻而易举的事情吗？并且对于骨架屏来说，它只需要展示首屏的内容，所以我们可以利用插件的全局变量，进行判断，是否需要后续的抓取页面动作。这样也解决了骨架屏的体积大小问题。

再则，`SEO`的问题对于目前开发的应用，很少有要求，基本都是入口页。但是用 `prerender-spa-plugin` 也可以完美的解决，生成 `Loading` 的多个路由页面或者骨架屏的多个路由页面，都可以由后端部署到 `vm` 模板中，编写对应的`title` 和`meta`，利用 `SEO` 的优化。

以上是自动化抓取页面生成骨架屏，但目前还是处于研究阶段，目前我们在生产项目中用的仍然是手写的骨架屏组件，可以参考这个 `vue-skeleton-loading`[5] 组件。利用预渲染的插件将骨架屏 `Loading` 组件或者标准的 `Loading` 组件以 `DOM` 形式输出到部署生产的 `HTML` 页面中。


# 总结

本文罗列了单页面体验的痛点：首屏加载慢、白屏的问题以 `SEO`。也给出了渐进的解决方案利用预渲染 `prerender-spa-plugin` 的输出 `Vue` 或者 `React`公用组件（骨架屏组件和  `Loading ` 组件）到各个路由页面 `HTML` 中。后续将依赖预渲染插件进行自动化骨架屏的输出方案，欢迎讨论和交流，敬请关注～

# 扩展阅读
[1] Nuxt.js：https://nuxtjs.org/guide
[2] prerender-spa-plugin：https://github.com/chrisvfritz/prerender-spa-plugin
[3] vue2-webpack-router：https://github.com/chrisvfritz/prerender-spa-plugin/tree/dba55854a95a7a4e9b4aaf4203fb0563739bc58a/examples/vue2-webpack-router
[4] puppeteer： https://github.com/GoogleChrome/puppeteer
[5] vue-skeleton-loading：https://github.com/jiingwang/vue-skeleton-loading


