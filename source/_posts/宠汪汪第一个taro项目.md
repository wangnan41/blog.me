---
title: 宠汪汪第一个taro项目
date: 2019-11-11 10:05:05
tags: [taro,小程序,h5]
categories: fe
---

赶在双十一之前，我们利用Taro开发了一个h5和小程序多端的虚拟宠物养成游戏项目。下面简单的做一些总结和回顾～

## 宠汪汪项目
宠汪汪是一款落地于京东APP-我的 的一款轻量级虚拟宠物养成游戏，由我的京豆频道和京东数科来客有礼联合打造，
集娱乐性+工具性于一体，独具匠心地嵌入了宠物赛跑玩法，是电商+小游戏的进化版本宠汪汪机智的系统将根据用户在京东的注册时间分配给他一只专属狗狗，用户可通过多种形式在宠汪汪完成任务领取狗粮，对虚拟宠物进行喂养，宠物得到投喂后用户积分及宠物等级都会得到提升，还可用积分兑换实物奖励。京东APP入口-我的，京豆-一起宠汪汪：

<img src="https://user-gold-cdn.xitu.io/2019/11/11/16e59d08fee25dea?w=750&h=1334&f=jpeg&s=59777"  width="320px"/>

<img src="https://user-gold-cdn.xitu.io/2019/11/11/16e59d090aeb2da5?w=750&h=1334&f=jpeg&s=93780" width="320px"/>

小程序入口-来客有礼

<img src="https://user-gold-cdn.xitu.io/2019/11/11/16e59d08feb7f6bb?w=750&h=1334&f=jpeg&s=122454" width="320px"/>

<img src="https://user-gold-cdn.xitu.io/2019/11/11/16e59d090c44fe46?w=430&h=430&f=jpeg&s=33807" width="320px"/>

## 技术选型
业务需求需要我们同时在h5和小程序开发一套，既保证App端入口的功能，也要保证小程序端的功能，还要保证App端分享的小程序卡片引导用户进行社交分享，增加游戏体验、和用户参与度。为解决多端开发的问题，市面上有很多类似的开发框架，Vue.js规范的[uni-app](https://uniapp.dcloud.io/)、[mpvue](https://github.com/Meituan-Dianping/mpvue)、[wepy](https://wepyjs.github.io/wepy-docs/)，React.js规范的[Taro](https://taro.aotu.io/home/in.html)等等。
根据我们现有的微信原生小程序开发了来客有礼，此小程序目前有80万左右的UV。宠汪汪项目相当于在来客有礼中开发一个新的页面，起初我们打算写两套，原生小程序一套，h5开发一套。但是这样子开发周期过长，无法保证我们在双十一之前上线，而且维护成本也很高。最后我们选择了Taro来解决这个问题，它可以一套代码，编译成两端运行。并且编译后的小程序代码可以很好的与原生小程序项目融合在一起。

### Taro本地h5开发
首先按照官方文档，`npm install -g @tarojs/cli`，然后创建项目 `taro init jd_dog_web`。在 `app.jsx`中添加pages对应的开发的页面。

```
pages: [
    'pages/jdDog/jdDog',
    'pages/petRace/petRace',
    'pages/exchange/exchange',
    'pages/exchange/exchangeRecord',
    'pages/followShop/followShop'
],
```

我们项目开发的目录如下：
* pages：业务页面
* components：模块组件
* common：公共方法
* assets：图片等静态资源

<img src="https://user-gold-cdn.xitu.io/2019/11/11/16e59d0919845644?w=390&h=972&f=jpeg&s=27640"  width="320px" />

通过`npm run dev:h5`进行本地重构开发，第一步已经成功，下面就是兼容原生小程序。

<img src="https://user-gold-cdn.xitu.io/2019/11/11/16e59d090fc853b7?w=814&h=1450&f=jpeg&s=255563" width="320px" />

### Taro与原生小程序融合
因为我们之前是使用原生小程序开发的项目，项目里面有很多公共的方法和模块，所以如何使得我们新开发的页面能够调用并且正常运行原小程序项目的代码成为关键。其实并没有想象那么复杂。
`process.env.TARO_ENV === 'h5'`是关键，其实我们在编译时候会选择是编译到h5还是weapp，所以Taro做了编译场景判断，如果是h5的话，不会包含特殊的小程序代码。比如，宠汪汪的项目需要登录，在app中登录，在小程序中也需要联合登录，联合登录使用的是小程序登录插件。当我们`npm run build:weapp`后，将代码复制到原小程序项目中`requirePlugin('loginPlugin')`可以直接获取到，并且正常调用小程序的联合登录插件。在场景值判断是在小程序中，正常引用原生小程序的一些公共方法是OK的。只不过调试部分稍微麻烦点，需要编译后小程序代码复制到小程序项目中，才能正常运行。

```
loginJd(cb,isLoginOut){
    //判断是h5 环境还是 小程序环境
    if(process.env.TARO_ENV === 'h5'){
        if(isLoginOut){
            //直接强制登录
            ...
            return;
        }
        //判断是h5否已经登录京东账号
        let url = ``
        Taro.request({
            url:url,
            data:{},
            jsonp:true
        }).then((res)=>{
            if(res.data && res.data.islogin =='1'){
                console.log('已登录');
                this.prepareH5()
                cb && cb(1);
            }else{
                console.log('未登录');
                cb && cb(0);
            }
        })
        
    }else{
        let plugin = requirePlugin('loginPlugin');
        let ptKey = plugin.getStorageSync('jdlogin_pt_key');
        if(isLoginOut){
            if(ptKey){
                plugin.logout({
                    callback: ()=> {}
                    });
            }
            cb && cb('');
            return;
        }
        cb && cb(ptKey);  
    }
},
```

说一说如何能把编译后的代码融合到原生小程序中，第一步就是在原生小程序项目中，添加对应的页面，因为不是首页，所以我们直接把页面放到分包中。注册页面后，我们还需把对应的编译完的页面放入到项目的pages中。

```
"subPackages": [
    {
      "root": "pages/followShop/",
      "pages": [
        "followShop"
      ]
    },
    {
      "root": "pages/jdDog/",
      "name": "jdDog",
      "pages": [
        "jdDog"
      ]
    },
    {
      "root": "pages/exchange/",
      "pages": [
        "exchange",
        "exchangeRecord"
      ]
    },
    {
      "root": "pages/petRace/",
      "name": "petRace",
      "pages": [
        "petRace"
      ]
    },
]
```

我们把如下图的编译后的代码直接pages文件移动到原生小程序的pages内，components内的移动到原生小程序的components内。其他的一些依赖的common等文件直接移动到原生项目的一级目录。这样子，在小程序内就可以跑起来了。

<img src="https://user-gold-cdn.xitu.io/2019/11/11/16e59d0932ca99bf?w=580&h=806&f=jpeg&s=16464" width="375px"/>

<img src="https://user-gold-cdn.xitu.io/2019/11/11/16e59d093d707354?w=754&h=1338&f=jpeg&s=185036" width="375px"/>

## 动画方案
对应一个游戏类的项目，怎么能少了动画呢，对于动画部分，作为前端来说可谓是一大障碍，无论是用css还是js来实现动画，并不能很好的还原设计师的原稿，当然简单的一些动画效果可以利用CSS3实现。但是一些复杂的联动的效果是很难的。动效系数是决定页面动画效果是否自然的关键。所以还是将这部分交还给动效设计师吧。此次项目有动效设计师把持，给予了 `APNG` 的 `PNG` 动画图。比如吃饭、升级、倒狗粮、赛跑等难度大的动效果都由 `APNG` 图完成。很幸运的是APNG在移动端支持还是不错的。

<img src="https://user-gold-cdn.xitu.io/2019/11/11/16e59d0940158a53?w=2516&h=810&f=jpeg&s=89775" width="375px"/>

并且`APNG` 对比 `GIF` 由很多方面的优势
- GIF：
最多支持 8 位 256 色，色阶过渡糟糕，图片具有颗粒感
不支持 Alpha 透明通道，边缘有杂边
- APNG：
支持 24 位真彩色图片
支持 8 位 Alpha 透明通道
向下兼容 PNG

选择APNG动画开发动效大大减少了的前端工作量。

## 项目总结
在11月1日当天10点项目正式上线，截止11月11日宠汪汪项目PV已经达到350万，UV达到28万，并且每天都在持续的增长。利用Taro解决了多端场景的痛点，当然项目中有些场景还是需要单独写h5和小程序的代码，以满足业务需求比如长图保存，打字动效果等等。整体来说，的确提高了开发效率，减少研发周期。同时也感谢Taro的技术团队的支持～。




