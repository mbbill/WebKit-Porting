# WebKit 在嵌入式设备上的移植和定制

这个文档的标题是“WebKit 在嵌入式设备上的移植和定制”。这是标题看起来很大，而且似乎不是很容易让人明白到底要做什么，所以我先来简单介绍一下写这一系列文档的目的。

一般来说提到“移植”一个项目，尤其是在嵌入式设备上的“移植”，就是我把一个项目改一改，拿厂商的toolchain编译一下，很多时候还是交叉编译器，然后跑在某个特定的bsp上。这个bsp很多时候是一些特定的Linux内核加上一些用户态的支撑库，同时他们很多都是高度定制的。或者也可以是某个支持POSIX API的RTOS，例如RT-Thread smart。

如果单单是希望在类似Ubuntu这样的Linux发行版上跑WebKit，那太简单了，直接编译WebKitGTK就可以了，但是如果用户态是定制的，那么很多事情就变得棘手， 比如：
- 如果需要的第三方库不存在怎么办？例如bsp没有GTK。
- 如果需要的第三方库和bsp提供的版本不兼容怎么办？
- 那些高度依赖硬件的部分怎么处理？例如音频，视频，渲染加速等等。
- 如果CPU指令集是定制的怎么办？比如遇到一个MIPS的，或者其他更诡异的芯片，我是不是要去重写JIT？那些ARM/X86_64汇编我要不要重写？
- 如果bsp存在一些硬性限制怎么办？比如很多厂商因为安全的缘故禁止申请executable memory，那么很显然所有需要JIT的部分都无法执行了，包括JavaScript JIT, CSS JIT, Regexp JIT等等。那JIT被砍掉以后怎么跑解释器？
- 有些厂商甚至禁止fork，任务必须是单进程，只允许多线程。那么WebKit需要的多进程模型怎么解决？

还有其他许许多多诸如此类的问题。一般来说这样的项目投入都非常的大，我们都不太希望在项目进行到一半的时候遇到一些过不去的坎，以至于要推倒重来，或者导致项目失败。所以这也是我写这一系列文章的目的。

## 为什么选择WebKit

目前三大主流浏览器 —— Chrome，Safari(WebKit)和Firefox各自都有自己的内核，其他浏览器基本上都是从这三个改出来的。
- Chrome在最开始的时候使用的是WebKit，没过几年就分支出来变成了Blink。Blink在Chrome里面的作用是一个Layout engine。一般来说我们平时谈论的网页渲染会经历很多个步骤，从下载，到layout，render，CSS加速，GPU渲染等等。Blink在这里主要是完成的Layout和一小部分渲染，所以它已经不再是像WebKit一样完整的网页渲染引擎。所以单独移植Blink是没有意义的，而且Blink也不能脱离Chrome单独运行。以后如果有时间我可以再写写移植Chrome的难点，但是总的来说Chrome是不适合嵌入式设备的。
- WebKit目前支撑着Safari，Epiphany这一类略显小众的浏览器。虽然WebKit的代码规模比Chrome小很多，但也是千万行级别的。从可移植性的角度来看，WebKit比Chrome好很多，因为：
  - WebKit原生支持Linux，目前官方有WebKitGTK和WebKitWPE可以在Linux上轻松编译。
  - WebKit依赖库比Chrome少很多，而且可以根据需求裁剪，例如可以砍掉Gstreamer如果不需要多媒体支持的话。
  - WebKit可以依赖平台本身提供的第三方库进行编译，而Chrome大多数第三方库都是自带，也就意味着需要自己移植（考虑到需要的资源，我感觉目前除了Google以外基本是不可能做到的）。
  - WebKit可以裁剪到很小的尺寸。我自己的项目可以砍到30M以内的大小。
  - WebKit有一个性能非常不错的JavaScript 引擎，更难能可贵的是这个JavaScript引擎通过一定的修改可以做到完全平台无关的解释执行。这一点后面会详细说。
  - WebKit有一个相对简单的渲染路径，虽然性能不是特别好，但是对某些特定的情况比较友好。
  - 还有很多优点我会在下面每个章节详细说。
- 但是无论如何WebKit都始终是一个极其庞大的项目。如果要用来做项目，那么如果没有对它细致地了解，很多事情是搞不定的。这需要时间和耐心，对着代码思考，然后慢慢的上手。希望像其他项目一样拿来改一下编译脚本就能跑是不可能的。后面我会在每一个段落尽量解释所有难点和细节。


## [WebKit的项目结构](Contents/ProjectStructure.md)

介绍WebKit整个项目的组成结构，包括目录结构，编译组件的依赖关系等等。

## [WebKit的编译过程](Contents/Compilation.md)

WebKit每一个组件的编译过程都非常复杂。这部分会先大致梳理一遍整个项目的编译过程。具体每个组件的细节会在对应的章节详细介绍。


## [WTF移植](Contents/WTF.md)

WTF是Web Template Framework的缩写。这个库原本是JavaScriptCore的一部分，后面独立出来变成一个公共库，为WebKit提供一些基础组件。

## [JavaScriptCore移植](Contents/JSC.md)

JSC是JavaScriptCore的缩写。JSC是WebKit自带的JavaScript引擎。在早年Chrome仍然使用WebKit的时候曾有过两套JavaScript引擎的绑定，一个是JSC，另一个是V8。在Chrome独立出去以后JSC就成为WebKit默认且唯一的JavaScript引擎。

## [WebCore移植](Contents/WebCore.md) [WIP]

WebCore是WebKit的核心组件。它主要提供的功能是排版（Layout），渲染，多媒体，网络，W3C各种标准的实现，等等。虽然其中有些功能并不完全在WebCore中实现，例如网络和渲染有单独的进程负责，但是WebCore依然是处理绝大多数事情的中心。

## [WebKit的图形与渲染](Contents/Graphics.md) [WIP]

WebKit目前有两套相对独立的渲染路径。一个是以Apple体系下的图形库为主，主要依靠*CoreGraphics*和*CoreAnimation*进行渲染。这个渲染路径涉及到的依赖库主要都是苹果系统上独占的闭源库，所以只能在苹果系统下使用，例如macOS或者iOS。第二个渲染路径基于GTK/Cairo/OpenGL，依靠开源的的图形库从而支持Linux系统下的`WebKitGTK`和`WebKitWPE`。这么看的话我们在移植WebKit的时候只能选择这第二条渲染路径。这个章节会讨论GTK和Cairo的2D绘图以及OpenGL支持下的CSS动画加速。

## [WebKit的单进程与多进程架构](Contents/MultiProcessing.md) [WIP]

可能很多人都知道Chrome的多进程架构以及因此造成的内存占用问题，很多人不知道的是目前WebKit也是如此。比如在Mac上打开Safari以后查看进程树就会发现诸如`UIProcess` `WebProcess` `NetworkPocess` 这些子进程。

在Chrome中每个tab都会占用一个子进程，叫做`RenderProcess`负责渲染。WebKit中对应的子进程是`WebProcess`。虽然在功能划分上他们之间有不少区别，但是概念上大同小异，都是把一部分网页渲染的内容放到了这个子进程中。

这一部分会详细介绍WebKit多进程架构的相关内容以及移植要点。


## [WebKit多媒体支持](Contents/MultiMedia.md) [WIP]

音频和视频部分与图形渲染的情形基本相同——Apple自己有一套基于自身生态系统的移植，同时开源社区维护者一套基于Gstreamer的播放器实现。他们都支持传统的url播放和目前比较流行的以MSE/EME为基础的adaptive streaming。

## [WebKit功能模块裁剪](Contents/Features.md) [WIP]

这部分会讨论在嵌入式平台上可以裁剪掉的一些不常用的组件，例如WebRTC等等。

## [Web Inspector](Contents/Inspector.md) [WIP]

Inspector是WebKit的一个非常重要的组件。WebKit的Inspector支持本地和远程模式。本质上Inspector的UI也是一个网页。在本地模式时Inspector页面和被调试页面归属同一个WebKit UI进程管理，而远程模式下Inspector页面会运行在另一个浏览器上，然后通过IPC和被调试浏览器通信。远程模式类似remote gdb，但是和gdb不同的是WebKit的Inspector在Linux上走D-Bus协议，而在Apple平台走XPC协议。这部分会讨论用*WebSocket*实现Inspector来做到减少依赖以及跨平台。

## 附录

下面是我过去编写的一些相关资料。他们可能有一些过时，然后不全是中文，所以在以后有空的时候我会逐步翻译。

- [WebKit Build Process](Appendix/WebKitBuildProcess.md)
- [打造一个可移植的JS引擎](Appendix/PortableJSEngine.md)
- [Ref and RefPtr](Appendix/Ref_RefPtr.md)

