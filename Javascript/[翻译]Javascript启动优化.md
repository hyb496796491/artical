>原文地址：[JavaScript Start-up Optimization](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/javascript-startup-optimization/?utm_source=mybridge&utm_medium=blog&utm_campaign=read_more)

当我们的网站越来越依赖于Javascript，有时我们会不容易发现某些方式带来的性能下降。在本文中，我们将会介绍为什么一些规则能够帮助你提升移动网站的交互和加载速度。提供更少的JavaScript意味着更少的网络传输时间，更少的代码解压缩以及更少的时间解析和编译此JavaScript。

## 网络

当大多数开发人员考虑JavaScript的成本时，下载和执行成本首先映入眼帘。通过线路发送更多字节的JavaScript会延长用户等待的时间。

![image](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/javascript-startup-optimization/images/1_U00XcnhqoczTuJ8NH8UhOw.png)

即使在第一世界国家，这可能也会造成困扰，因为有效的网络连接类型实际上可能不是3G，4G或Wi-Fi。一家咖啡店的Wi-Fi可能只有2G的速度。

您可以减少JavaScript通过网络传输的成本：

- 只发送用户需要的代码。
    - 使用[代码拆分](https://developers.google.com/web/updates/2017/06/supercharged-codesplit)将JavaScript分解为重要的和不重要的。像[webpack](https://webpack.js.org/)这样的模块包装器支持代码拆分。
    - 懒惰地加载非关键代码。
 
- 缩小
    - 使用[UglifyJS](https://github.com/mishoo/UglifyJS)来[缩小](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/optimize-encoding-and-transfer#minification_preprocessing_context-specific_optimizations)ES5代码。
    - 使用[babel- minify](https://github.com/babel/minify)或[uglify -es](https://www.npmjs.com/package/uglify-es)来缩小ES2015+。
    
- 压缩
    - 至少，使用[gzip](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/optimize-encoding-and-transfer#text_compression_with_gzip)来压缩基于文本的资源。
    - 考虑使用 [Brotli 〜Q11](https://www.smashingmagazine.com/2016/10/next-generation-server-compression-with-brotli/)。Brotli在压缩方面的功效超过Gzip。它帮助CertSimple节省了17％的JS字节的大小，帮助LinkedIn节省了4％的加载时间。
    
- 删除未使用的代码。
    - 使用[Chrome DevTools代码覆盖率](https://developers.google.com/web/updates/2017/04/devtools-release-notes#coverage)面板识别可以删除或延迟加载的代码。
    - 使用 [babel-preset-env](https://github.com/babel/babel/tree/master/packages/babel-preset-env) 和browserlist来避免转译那些浏览器已经支持的特性。高级开发人员可能会仔细分析[webpack依赖](https://github.com/webpack-contrib/webpack-bundle-analyzer)来避免引入多余的包。
    - 对于代码的剥离，可以参考[tree-shaking](https://webpack.js.org/guides/tree-shaking/)，[Closuer Compiler](https://developers.google.com/closure/compiler/)的高级优化功能以及一些插件库比如 [lodash-babel-plugin](https://github.com/lodash/babel-plugin-lodash)或者webpack的[ContextReplacementPlugin](https://iamakulov.com/notes/webpack-front-end-size-caching/#moment-js)（比如Moment.js）
    
- 缓存代码以尽量减少网络请求。
    - 使用[HTTP缓存](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching)来确保浏览器有效缓存响应。可以使用决定脚本最佳生命周期的max-age以及避免传送没有改变字节的验证令牌（ETag）
    - [服务工作者](https://developer.mozilla.org/zh-CN/docs/Web/API/Service_Worker_API)缓存可以使您的应用网络具有弹性，并让您可以快速访问[V8的代码缓存](https://v8project.blogspot.com/2015/07/code-caching.html)等功能。
    - 使用长期缓存来避免必须重新获取未更改的资源。如果使用Webpack，请参阅[文件名散列](https://webpack.js.org/guides/caching/)。

## 解析/编译

一旦下载完成，JS引擎对代码的解析/编译是其中一项很大的消耗。在Chrome DevTools中，解析和编译是Performance面板中黄色“Scripting”时间的一部分。

![image](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/javascript-startup-optimization/images/1__4gNDmBlXxOF2-KmsOrKkw.png)

Bottom-Up和Call Tree标签向你显示精确的分析/编译的时间点：

![image](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/javascript-startup-optimization/images/1_GdrVt_BTTzzBOIoyZZsQZQ.png)

通过启用V8的调用统计，我们可以看到分析和编译等阶段花费的时间

> 注意：Performance面板对调用统计的功能目前是实验性的。要启用，请转至
chrome://flags/#enable-devtools-experiments -> 重启 Chrome -> DevTools -> Setting -> Experiments -> 连续敲击6次shift键 -> 勾选 Timeline: V8 Runtime Call Stats on Timeline 选项 -> 重新打开DevTools

但是，为什么这很重要？

![image](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/javascript-startup-optimization/images/1_Dirw7RdQj9Dktc-Ny6-xbA.png)

花费过长的时间去解析/编译代码会严重延迟用户与网站交互。发送的JavaScript越多，在网站交互之前解析和编译所需的时间就越长。

> 浏览器处理JavaScript比处理同等大小的图像或Web字体消耗的更多 - Tom Dale

与JavaScript相比，处理同等大小的图像涉及许多成本（它们仍然需要解码！），但对于移动硬件，平均而言，JS更可能对页面的交互产生负面影响。

![image](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/javascript-startup-optimization/images/1_PRVzNizF9jQ_QADF5lQHpA.png)

处理JavaScript和图像字节具有非常不同的成本。图像通常不会阻塞主线程或阻止接口在解码和栅格化时进行交互。但对于Js而言，解析、编译和执行都会延迟JS的交互。

当我们谈论解析和编译变得缓慢的时候，上下文很重要 - 我们在这里谈论的是一般的手机。对于一般用户而言，他们的手机可能配置处理速度缓慢的CPU和GPU（不存在L2 / L3缓存，甚至可能会受到内存限制）

> 网络功能和设备功能并不总是相匹配。具有光纤网络的用户不一定配备有最好的CPU来分析和评估发送到他们设备的JavaScript。反过来也是如此 —— 糟糕的网络连接，拥有性能卓越的CPU。 - Kristofer Baxter，LinkedIn

如下图，我们可以看到在低端和高端硬件上解析大概1MB JavaScript（已解压缩）的成本。最快的手机和普通手机之间解析/编译代码的时间相差2-5倍。

![image](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/javascript-startup-optimization/images/1_8BQ3bCYu1AVvJWPR1x8Yig.png)

此图强调了跨桌面和不同类别的移动设备对于1MB JavaScript（gzip后约250KB ）的分析时间。

那么真实情况（如CNN）表现如何呢？

在iPhone 8（高端）上，解析/编译CNN的JS只需4秒钟，而普通手机（Moto G4）的时间约为13秒。这可以显着影响用户与本网站完全互动的速度。

![image](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/javascript-startup-optimization/images/1_7ysArXJ4nN0rQEMT9yZ_Sg.png)

以上我们看到，Apple的A11仿生芯片与骁龙617在解析上的对比。

这突出了测试一般硬件（如Moto G4）的重要性，而不仅仅是放在你口袋里的手机。但是，上下文很重要：优化你的用户拥有的设备和网络条件。

![image](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/javascript-startup-optimization/images/1_6oEpMEi_pjRNjmtN9i2TCA.png)

Google Analytics可以帮助你深入了解访问网站的用户所持有的[设备情况](https://crossbrowsertesting.com/blog/development/use-google-analytics-find-devices-customers-use/)。这可以提供机会来了解他们正在使用的真实CPU / GPU约束条件。

我们是否真的发送了太多的JavaScript？额，可能吧:)

使用[HTTP Archive](https://beta.httparchive.org/reports/state-of-javascript#bytesJs)（约500K个站点）来分析移动设备上JavaScript使用的状态，我们可以看到50％的站点需要14秒才能获得交互。这些网站花费长达4秒在解析和编译JS。

![image](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/javascript-startup-optimization/images/1_sVgunAoet0i5FWEI9NSyMg.png)

获取和处理JS或其他资源所花费的长时间，可能会使用户在等待一段时间后选择离开，这一点应该不令人惊讶。不过我们在这里肯定能做得更好。

从网页中去除那些不重要的JavaScript代码可以减少传输时间，CPU的密集解析和编译以及潜在的内存开销。这也有助于让您的网页更快地交互。

## 执行时间

带来成本消耗的不仅仅是解析和编译。JavaScript执行 （解析/编译完成后运行代码）是必须在主线程上发生的操作之一。较长的执行时间也可以阻碍用户与网站互动。

![image](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/javascript-startup-optimization/images/1_ec0wEKKVl7iQidBks3oDKg.png)

> 如果脚本执行时间超过50ms，则交互延迟的时间就是下载，编译和执行JS所需的 全部时间 - Alex Russell

为了解决这个问题，把Javascript分块执行可以避免阻塞主线程。

## 其他成本

JavaScript可以通过其他方式影响页面性能：

- 内存。由于GC（垃圾收集），页面可能会出现频繁的卡顿或暂停。当浏览器回收内存时，JS会被暂停执行，因此频繁的垃圾回收会造成频繁的程序暂停。避免内存泄漏和频繁的GC暂停，以保持页面流程。

- 在运行时期间，长时间运行的JavaScript会阻塞主线程，导致页面无响应。将工作分成更小的部分（使用 requestAnimationFrame() 或requestIdleCallback() 调度）可以最大限度地减少响应问题。

## 减少JavaScript交付成本的模式

当您试图为JavaScript保留解析/编译和网络传输时间时，有一些模式可以帮助像基于路由的分块或 [PRPL](https://developers.google.com/web/fundamentals/performance/prpl-pattern/)。

### PRPL

PRPL（Push，Render，Pre-cache，Lazy-load）是一种通过积极的代码分割和缓存来优化交互性的模式：

![image](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/javascript-startup-optimization/images/1_VgdNbnl08gcetpqE1t9P9w.png)

让我们看看它可能产生的影响。

我们使用V8的调用统计来分析流行移动网站和Progressive Web Apps的加载时间。正如我们所看到的，解析时间（以橙色显示）是许多这些网站花费时间的重要部分：

![image](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/javascript-startup-optimization/images/1_9BMRW5i_bS4By_JSESXX8A.png)

Wego是一家使用PRPL的网站，它设法利用路由来保持较低的解析时间，使得交互速度非常快。上面的许多其他站点都采用代码分离和性能预算来尝试降低JS的成本。

### 逐行引导

许多网站以昂贵的交互性代价来优化内容可视性。为了在有大量JavaScript包时获得快速的第一次绘制，开发人员有时会使用服务器端渲染，然后在JavaScript最终获取时，利用事件处理程序来“升级”它。

小心 - 这有它自己的成本。你1）通常发送一个更大的HTML响应，这可以推动我们的交互性，2）可以让用户进入一个离奇的山谷，在JavaScript完成处理之前，有一半的经验实际上不能互动。

渐进式引导可能是一种更好的方法。发送一个最小功能页面（由当前路由所需的HTML / JS / CSS组成）。随着更多资源到达，该应用程序可以延迟加载和解锁更多功能。

![image](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/javascript-startup-optimization/images/1_zY03Y5nVEY21FXA63Qe8PA.png)

根据视图内容加载合适的代码是就像圣杯。PRPL和渐进式引导是可以帮助实现这一目标的模式。

## 结论

在网络不佳的情况下，传输大小至关重要。解析时间对于CPU绑定的设备很重要。

有团队发现接受严格的性能预算可以保持较低的JavaScript传输和解析/编译时间。有关移动预算的指导，请参阅Alex Russell的“ [您能承担吗？：真实世界的Web性能预算](https://infrequently.org/2017/10/can-you-afford-it-real-world-web-performance-budgets/)”。

![image](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/javascript-startup-optimization/images/1_U8PJVNrA_tYADQ6_S4HUYw.png)

考虑一下我们制作的架构决策可以为应用程序逻辑留下多少JS“空间”。

如果您正在构建一个针对移动设备的网站，请尽可能在代表性硬件上开发，保持较低的JavaScript分析/编译时间，并采用性能预算来确保您的团队能够关注其JavaScript成本。