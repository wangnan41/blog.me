---
title: 为什么在一些情况下WebAssembly比JavaScript更具有优势
date: 2019-06-26 14:07:38
tags: [webAssembley,javascript,v8]
categories: translation
---

作者:Alexander Zlatkov | 译：汪楠
原文地址：[https://blog.sessionstack.com/how-javascript-works-a-comparison-with-webassembly-why-in-certain-cases-its-better-to-use-it-d80945172d79/]

本文是探究`JavaScript`工作原理和构建组件文章的第六篇。在鉴定和描述核心内容过程中，我们同时分享了在开发`SessionStack`时候的一些经验。`SessionStack`其实是一款帮助用户实时查看和重现其Web应用缺陷的轻量级`JavaScript`应用程序。它拥有强大的功能和不错的性能表现。

如果你错过了之前的文章，可以点击下面的链接：
- An overview of the engine, the runtime, and the call stack
- Inside Google’s V8 engine + 5 tips on how to write optimized code
- Memory management + how to handle 4 common memory leaks
- The event loop and the rise of Async programming + 5 ways to better coding with async/await
- Deep dive into WebSockets and HTTP/2 with SSE + how to pick the right path

这次我们将拆分`webAssembly`来阐述其工作原理。重点则是对比`JavaScript`，分析它们在一些性能方面上的差异：加载时间，执行速度，垃圾回收，内存占用，平台API，调试，多线程和可移植性。

我们开发web应用方式正处于变革的边缘，虽然现在来看还为时尚早，可是我们对于web应用的思考正在不断的发生改变。

# 首先，让我们看看WebAssembly是什么

`WebAssembly`（又名wasm）是一个运行在网络浏览器中的高效的、低级别的字节码。`WASM`使得你可以使用JavaScript以外的语言（C,C++,Rust 等），用它们编写代码，然后提前编译成`WebAssembly`。

这所带来的好处的就是可以非常快的加载和运行web程序。

# 加载时间

为了去加载JavaScript，浏览器不得不去加载所有带“.js”后缀的文本文件。

`WebAssembly`在浏览器中加载更快，因为它只需要通过互联网传输已经编译完的`wasm`文件。 并且`wasm`是一个低级别的，非常紧凑的二进制格式的类汇编语言。

# 执行速度

目前`Wasm`执行速度只比原生代码慢20%。这无疑是一个令人惊讶的结果。因为它是一种被编译在沙箱环境中的格式，并且运行在诸多限制条件下或者苛刻的条件下，以确保其安全性。与本地运行的代码相比慢了一些。重点是，可以预见它在未来将会更快。

更具有优势的是，它与浏览器无关，所有浏览器引擎都增加了对`WebAssembly`的支持并且目前执行的时间也很类似。

为了理解`WebAssembly`执行速度相比较`Javascript`有多快，你应该先阅读我们关于`Javascript`引擎是如何工作的那篇文章。

让我们从下图直观看一下Javascript运行在V8引擎中发生什么事。

<img src="http://img11.360buyimg.com/uba/jfs/t16804/209/1074245115/32369/406c0c76/5abb0509Ne7c83bf5.png" width="250" />

在图的左边，我们有一些JavaScript源码，包含了一些Javascript函数。首页需要被解析，将字符串转变为标记符号，然后生成抽象语法树（AST）。AST是你的JavaScrip代码在内存中逻辑表示。一旦生成AST表示，V8引擎可直接将其转成机器码。你基本只能按照这个树，生成机器码，然后执行你的函数。这样根本没有尝试使得它执行的更快。

现在，看一下V8管道是如何做的：

<img src="http://img14.360buyimg.com/uba/jfs/t15856/279/2678048376/22864/ab67d883/5abb0509Ne6cd615e.png" width="250" />

此刻我们有了`TurboFan`，一个V8优化编译器。当你的JavaScript应用正在运行，很多代码都执行在V8中。`TurboFan`可以监控某些代码是否执行过慢，是否存在瓶颈和关键点需要优化。它将推送他们到后台交给已优化的JIT处理，它是专门为优化那些耗CPU内存的代码函数准备。

它解决了这个问题，但是也产生了另外的问题，在分析代码并且决定优化那些内容也同样消耗CPU。反过来说，这将意味需要消耗更多的电池能量，在移动设备上就更明显了。

很明显，`wasm`并不需要这么多操作，它可以被插入到工作流中像下图这样。

<img src="http://img11.360buyimg.com/uba/jfs/t16717/258/1119106469/27472/1db72da3/5abb0509N51a8f2fc.png" width="250" />

`Wasm`已经在编译阶段进行了优化。最重要的是解析也不需要了。你已经有一份优化完的二进制代码可以直接输入到后台生成机器码。所有优化工作都已经在前台编译器中完成。

这样`wasm`可以在运行时候跳过工作流中的一些步骤，使得执行更加高效。

# 内存模型

<img src="http://img10.360buyimg.com/uba/jfs/t17038/205/1037656294/55091/faf0b9e1/5abb0509Nb40723f1.png" width="250" />

例如，被编译成`WebAssembly`的C++程序在内存中是连续的，没有“漏洞”的。`Wasm`的一大特征就是使得执行堆栈和线性内存相分离，这样有助于提高代码安全性。在C++编程中，你有一个堆栈，从底部分配栈，从顶部堆栈。这样，也许你也可以利用一个指针在堆栈内存中查找到一些你不应该访问到的变量。

这是很多恶意软件惯用的手段。

`WebAssembly`采用完全不同的模型。执行堆栈和`WebAssembly`程序本身是分离的，所以你根本没机会在内存里改变它和改变变量之类的。而且，这些函数使用整数偏移而非指针。函数指向一个间接函数表。使用这些计算数字直接跳到模块中对应的函数中。它是以这种方式并排的加载多个`wasm`模块，并且不采用指针索引，功能同样也能正常运转。

想要更多关于内存模型信息和如何在JavaScript中管理，你可以点击post on topic这篇文章。

# 垃圾回收

你肯定对JavaScript的内存管理是用垃圾回收器集中处理的有所了解。`WebAssembly`的情况略有不同。它支持编写代码管理内存。你可以将自己写的`GC`和`wasm`模块打包在一起。当然这会是一个很复杂的任务。

目前，`WebAssembly`是围绕C++和RUST用例设计的。因为`wasm`是非常低级别的，它使得编程语言只是位于汇编语言之上一小步，因此非常容易编译它。C 能够使用标准的`malloc`，C++使用智能指针，Rust 则采用完全不同的模式（完全不同的领域）。这三门语言都不使用GC，它们不需要在运行时候跟踪内存的复杂的东西。`WebAssembly`很显然非常适合它们。

另外，这些语言并不是100%为复杂的JavaScript事物而设计的，例如更改DOM。采用C++来编写完整的HTML应用是有些愚笨的。因为C++并不是为它而设计的一门语言。在大多数情况下，工程师们编写C++或者Rust 主要是为了`WebGL`或者高度优化的库（例如：需要大量数学计算）。

在未来，`WebAssembly`会支持那些不带`GC`的语言。

# 平台API接入

依靠在运行时才执行JavaScript，JavaScript应用程序可以接入访问平台提供的特定的API。例如，你在浏览器中执行JavaScript代码，你会有一组Web API可供Web应用程序调用来控制web浏览器/设备功能像DOM、CSSOM、WebGL、IndexedDB、Web Audio API等。

然而，`WebAssembly`模块还不支持任何平台的API。任何API都是由JavaScript来调用。如果你想在`WebAssembly`模块中调用平台特定的API，只能通过JavaScript调用才行。

举个例子：如果你想使用`console.log`，你不得不使用JavaScript代替C++的代码。而调用JavaScript代码则会导致运行成本的增加。
这不会一直都是这样的，规范提出了在未来将为平台API提供`wasm`，使得你发布跟JavaScript没有任何关系的应用。


# source maps

当你压缩JavaScript代码后，你需要一个可行的方法来调试它。这就是`source maps`用武之地。从基础上来讲，`source maps`是一种将组合和压缩文件与之前未操作状态映射的方法。当你构建生产环境代码时，压缩合并了你的JavaScript文件，也同时生产了一个`source maps`文件，它包含了所有源文件的信息。当你在生成的JavaScript中查询确定行和列的代码时候，在`source maps`中查找返回对应的原始位置。

`WebAssembly`目前不支持`source maps`，还有相关的规范，但相信很快会有。
当你在C++中设置断点，你将会在C++ 代码中看到中断而不是在`WebAssembly`中。至少，这是我们所期望的目标。

# 多线程

JavaScript虽然只能运行在单线程上。不过有方法利用事件循环和异步编程像在此篇文章详细描述的内容这样来实现多线程。
JavaScript也可以使用`Web Workers`，他们有一个非常具体的例子—任何CPU高速的运转都会阻塞UI主线程，可以通过Web Worker下线线程解决。可是，`Web Workers`并不能直接操作DOM。

`WebAssembly`并不支持多线程，但是很快应该可以支持这个。Wasm将来与本地线程非常类似（比如像C++ 线程）。拥有真实的线程在浏览器中会创造出很多新的机会。当然，同样也有可能打开了滥用线程的大门。

# 可移植性

如今，JavaScript可以在任何地方运行，从浏览器到服务端甚至嵌入式系统中。
`WebAssembly`从设计上就保证了安全性和可移植性，就像JavaScript一样，它将运行在任何支持wasm的环境中（任何浏览器）。
`WebAssembly`与Java关于小程序的早期想要获取的目标是一致的。
 
# 在什么场景下使用`WebAssembly`更优于`JavaScript`呢？

在`WebAssembly`第一个版本中，主要关注于需要大量CPU计算（比如处理数学问题）。他想到能被大量使用的场景就是游戏—有非常多的像素点需要计算。你可以用C++或者Rust 编写你熟悉的`OpenGL`代码，并且将其编译成`wasm`。然后它就可以在浏览器中跑起来。

可以点击这个看一下（在`firefox`中运行）
http://s3.amazonaws.com/mozilla-games/tmp/2017-02-21-SunTemple/SunTemple.html。它是运行在虚幻引擎中。

另外比较有意义的事是使用`WebAssembly`（性能方面）实现一些处理CPU高密集工作的库。例如，某些图像处理。
在前面提到，`wasm`大多数处理提早在编译环节已经完成，因此在移动设备上可以减少其运行时候的所需要的电池能耗（很大程度上取决于引擎）。

在未来，你可以使用`WASM`库实现功能，即使你并没有编写代码和编译他们。你可以在`NPM`中找到对应的项目和其使用方法。
对于DOM操作和平台API的使用，使用JavaScript是非常合适，因为它并不会增加额外的开销，并且有本地API可以供使用。

在`SessionStack`中，我们不断的挑战JavaScript性能的边界，为了能够编写高度优化且高效的代码。我们提供的解决方案提供非凡的性能，所以不会降低客户本身的app的性能。将SessionStack集成到用户生产的Web应用或者网站后，它会开始记录任何内容：DOM的改变，用户交互，JavaScript异常，堆栈跟踪，失败的网络请求和调试数据。所有这些都将在生产环境中进行，但是不会影响产品的UX和性能。我们需要大量的优化代码和尽可能的异步操作。

不单单只有这个库而已！当你重放用户的会话在`SessionStack`中，我们必须重现当时出问题的所有现场，并且必须重构整个状态，允许你可以在会话的时间线上来回切换。为了做到这一点，我们大量使用JavaScript提供的异步方案，因为目前还是缺少可以替代的办法。

借助`WebAssembly`，我们可以将一些繁琐的处理和渲染转交给更合适的编程语言，并且将数据收集和DOM操作留给JavaScript来处理。
如果你想尝试使用一个`SessionStack`，你可以在开始时候免费使用。每个月计划将有1000个免费会话提供。

<img src="http://img20.360buyimg.com/uba/jfs/t18622/253/1086788209/116877/fa01d6cf/5abb050aNf5263976.png" width="250" />

视频资源：
https://www.youtube.com/watch?v=6v4E6oksar0
https://www.youtube.com/watch?v=6Y3W94_8scw

# 扩展阅读：

SessionStack:https://www.sessionstack.com/?utm_source=medium&utm_medium=blog&utm_content=Post-6-webassembly-intro

An overview of the engine, the runtime, and the call stack:https://blog.sessionstack.com/how-does-javascript-actually-work-part-1-b0bacc073cf?source=collection_home---2------1----------------

Inside Google’s V8 engine + 5 tips on how to write optimized code:https://blog.sessionstack.com/how-javascript-works-inside-the-v8-engine-5-tips-on-how-to-write-optimized-code-ac089e62b12e?source=collection_home---2------2----------------

Memory management + how to handle 4 common memory leaks:https://blog.sessionstack.com/how-javascript-works-memory-management-how-to-handle-4-common-memory-leaks-3f28b94cfbec?source=collection_home---2------0----------------

The event loop and the rise of Async programming + 5 ways to better coding with async/await:https://blog.sessionstack.com/how-javascript-works-event-loop-and-the-rise-of-async-programming-5-ways-to-better-coding-with-2f077c4438b5

Deep dive into WebSockets and HTTP/2 with SSE + how to pick the right path:https://blog.sessionstack.com/how-javascript-works-deep-dive-into-websockets-and-http-2-with-sse-how-to-pick-the-right-path-584e6b8e3bf7?source=collection_home---4------0----------------

our article on how the JavaScript engine:https://blog.sessionstack.com/how-javascript-works-inside-the-v8-engine-5-tips-on-how-to-write-optimized-code-ac089e62b12e?source=---------3----------------

AST:https://en.wikipedia.org/wiki/Abstract_syntax_tree

post on the topic:https://blog.sessionstack.com/how-javascript-works-memory-management-how-to-handle-4-common-memory-leaks-3f28b94cfbec

Web APIs:https://developer.mozilla.org/en-US/docs/Web/API

DOM:https://developer.mozilla.org/en-US/docs/Web/API/Document_Object_Model

CSSOM:https://developer.mozilla.org/en-US/docs/Web/API/CSS_Object_Model

WebGL:https://developer.mozilla.org/en-US/docs/Web/API/WebGL_API

IndexedDB:https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API

Web Audio API:https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API









