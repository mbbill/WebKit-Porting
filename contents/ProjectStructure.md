# WebKit的项目结构

首先去下载一份最新的WebKit代码。

```shell
git clone https://github.com/WebKit/WebKit.git
```
根目录大概长这样
```shell
-rw-r--r-- 1 mbbill 197121   2033 Apr 27 19:20 CMakeLists.txt
-rw-r--r-- 1 mbbill 197121 279658 Apr 27 19:20 ChangeLog
-rw-r--r-- 1 mbbill 197121 794798 Mar 23  2016 ChangeLog-2012-05-22
-rw-r--r-- 1 mbbill 197121 782150 Mar 20  2019 ChangeLog-2018-01-01
-rw-r--r-- 1 mbbill 197121  75864 Apr 27 19:20 Introduction.md
drwxr-xr-x 1 mbbill 197121      0 Apr 27 19:21 JSTests
drwxr-xr-x 1 mbbill 197121      0 Apr 27 19:21 LayoutTests
-rw-r--r-- 1 mbbill 197121    607 Apr 27 19:22 Makefile
-rw-r--r-- 1 mbbill 197121   4511 Apr 27 19:22 Makefile.shared
drwxr-xr-x 1 mbbill 197121      0 Apr 27 19:22 ManualTests
drwxr-xr-x 1 mbbill 197121      0 Apr 27 19:22 PerformanceTests
-rw-r--r-- 1 mbbill 197121   6172 Apr 27 19:22 ReadMe.md
drwxr-xr-x 1 mbbill 197121      0 Apr 27 19:22 Source
drwxr-xr-x 1 mbbill 197121      0 Apr 27 19:22 Tools
drwxr-xr-x 1 mbbill 197121      0 Apr 27 19:22 WebDriverTests
drwxr-xr-x 1 mbbill 197121      0 Mar 20  2019 WebKit.xcworkspace
drwxr-xr-x 1 mbbill 197121      0 Oct  5  2020 WebKitBuild
drwxr-xr-x 1 mbbill 197121      0 Apr 27 19:22 WebKitLibraries
drwxr-xr-x 1 mbbill 197121      0 Mar 20  2019 Websites
drwxr-xr-x 1 mbbill 197121      0 Apr 27 19:22 resources
```

## WebKit 的组件和依赖关系

上面这份WebKit代码的根目录里面囊括了几乎所有的组件。很显然我们不可能直接入手一个个目录过一遍，所以先捋一下WebKit最重要的一些组件和他们之间的依赖关系，然后我们再来对照着这些组件整理代码。

### WebKit最重要的几个组件

- *WTF*
- *JavaScriptCore*
- *WebCore*
- *WebKit*
- *WebKitLegacy*

他们的依赖关系基本如下

```
   ┌───────────────┐   ┌────────────────┐
   │    WebKit     │   │   WebKitLegacy │
   └───┬─────────┬─┘   └─┬──────────┬───┘
       │         │       │          │
       │     ┌───▼───────▼──┐       │
       │     │    WebCore   │       │
       │     ├─────────┬────┘       │
       │     │         │            │
 ┌─────▼─────▼────┐    │            │
 │ JavaScriptCore ◄────┼────────────┘
 └───────────────┬┘    │
                 │     │
                ┌▼─────▼┐
                │  wtf  │
                └───────┘
```

#### 依赖关系的定义

在WebKit中如果一部分代码可以单独编译成动态或者静态库，我们称之为一个组件。

一个组件被其他组件依赖，那么这个被依赖的组件需要提供头文件和二进制库文件以满足编译需求。

一个组件需要隐藏它的内部实现。除了暴露给其他组件的头文件以外，所有实现部分只有组件内部可见。

被依赖的组件偶尔也可以是可执行文件，例如JSC中有一些helper tools在编译的过程中帮助生成一些代码或者文件。

关于组件如何提供头文件，隐藏内部实现，以及各种包含路径，链接路径的设置会放在下一个章节[WebKit的编译过程]中详细描述。

#### WTF

WTF(WebKit Template Framework) 起初是JavaScriptCore内置的一个C++模板库，用来补充或者替代STL里的一些功能。后来因为WebCore和其他组件也会用到很多它里面的功能，于是整体搬出了JavaScriptCore成为了一个公用的底层库。从功能的角度来看它也不再仅仅是补充STL模板，而是成为了一个系统功能的抽象层。比如在里面可以找到对于OS线程，进程，文件系统，消息循环等等各种功能的包装。

但是这并不代表我们只需要移植一下WTF就大功告成了，因为在其他组件当中依然有很多对系统API的直接调用。WebKit项目目前并没有一个非常严格的“移植”层，也就是说在其他组件当中并没有严格要求所有系统功能调用都需要经过WTF。

#### JavaScriptCore

JavaScriptCore(下面都会简称JSC)是WebKit自带的JavaScript引擎。在最早的时候WebKit项目里面只有它和WebCore，一个负责JavaScript解释执行，一个负责页面渲染。现代WebKit中的很多组件都是后来逐渐演变分割出来的。

JSC的依赖关系非常简单，它只依赖*wtf*。事实上因为JIT(Just In Time)需要映射可执行页面的关系，JSC还有一个可选依赖*bmalloc*。不过简单起见我们可以暂时把*bmalloc*放一边。

#### WebCore

WebCore是WebKit项目中最重要的组件。从依赖关系图上可以看到它处在最中间的位置。它依赖JSC和WTF以及其他很多第三方库，同时它又被WebKit**或者**WebKitLegacy使用。

WebKit的绝大多数渲染功能以及W3C标准的实现都是在WebCore当中完成。渲染部分包括Layout, Rendering, Compositing等等。这里我使用WebKit代码中对应的英语以避免翻译成中文后寻找对应代码困难。

WebCore依赖JSC而不是和JSC平起平坐的原因是一个网页的渲染和执行始终是从页面的下载和解析开始的。注意这一点不同于NodeJS。类似NodeJS这样单纯的JavaScript引擎，它的起点是一个JavaScript文件，同时它没有访问dom的能力因为本身就不是浏览器。所以NodeJS的执行始终是在JS环境中。相反的是WebCore(或者说所有浏览器)的执行流程是从页面开始的。在页面下载完成以后浏览器会首先把DOM准备好。如果页面中没有JavaScript，比如没有任何script tag，或者没有内嵌script，那么JS引擎甚至不会被初始化。如果碰到页面中的JavaScript，一般情况下就需要同步执行。这时候浏览器会把DOM以及浏览器的其他功能绑定到JavaScript环境中的变量然后启动JS引擎。这也就是为什么在浏览器里可以调用这些我们称之为”**绑定**“的对象，例如window。所以这样就可以理解为什么WebCore需要依赖JSC，因为JS引擎在页面渲染中只处在一个被动的位置。对于没有JS的静态页面JSC不会被初始化。我们甚至可以取消WebCore到JSC的依赖，做一个只能解析静态页面而无法运行JavaScript的浏览器。

#### WebKit和WebKitLegacy

这里需要先解释一下WebKit这个名字以避免歧义。当我们平时提到浏览器的时候，WebKit一般指的是浏览器的内核。比如我们提到Safari或者Epiphany这种“**基于WebKit**”的浏览器的时候WebKit指的就是整个WebKit项目，包括WTF，JSC等等它所有的组件。

但是在这里却又有一个叫做“WebKit”的组件。所以在我们讨论这个项目的时候如果我们提到WebKit，或者**WebKit组件**，那么往往指的是这个上层库，而不是整个项目。这里比较容易产生误解，往往需要根据上下文来判断它指的是什么。在文章里我会用**WebKit项目**和**WebKit组件**来区分两者。

为了便于理解，我们可以先了解一下WebKit和WebKitLegacy这两个组件的来历。

在很多年前WebKit项目里面只有这么几个东西：

- JavaScriptCore，负责JavaScript执行。
- WebCore，负责页面熏染。
- WebKit，负责WebCore与浏览器之间的沟通，类似一个桥梁。
- Safari，（闭源）苹果自己的浏览器外壳。

*题外话，另外WebKit历史上还经历过KJS、KHTML的一些有趣阶段，有兴趣可以搜一搜。*

在这个时候WebKit组件还不是很庞大，没有多进程和其他复杂的功能。它的作用主要是两点：

- 把浏览器的功能反向提供给WebCore，例如浏览器的各种设置，浏览器上下文菜单，Cookie存储等等。
- 向浏览器提供WebCore的功能，例如加载某个页面，或者运行某个脚本。

它还提供一个比较稳定的API可以让浏览器使用，同时也不会像WebCore那样使用过于复杂。

后来发生了一件事情让WebKit开始向多进程方向转变，也就是Chrome的出现。最开始的时候(2012年之前)Chrome也是基于WebKit项目的，那时候Chrome在WebKit项目上包装了一层多进程架构以支持它的安全模型，但是那种多进程架构是在Chrome中而不是在WebKit中。没有多久以后WebKit项目团队就意识到这个项目也需要一个多进程的结构，于是就着手开始创建了一个叫做“*WebKit2*”的组件，也就是支持多进程模型的版本。所以这个时候：

- WebKit组件 - 单进程，被当时所有基于WebKit的浏览器使用。
- WebKit2组件 - 多进程，刚开始。

又过了很多年，几乎所有使用WebKit项目的浏览器都迁移到了WebKit2。这个时候项目团队觉得再叫“WebKit2”就有点奇怪，于是：

- WebKit组件改名成了WebKitLegacy，逐渐进入淘汰日程。
- WebKit2组件改名成了WebKit，处于积极维护中。

至此我们知道了WebKit和WebKitLegacy这两个组件的关系。目前WebKitLegacy虽然还没被删除，但也不久了。这两个组件处于同级别，分别对应两种不同的使用方式——单进程与多进程。我们的移植工作并不需要两个都做，择一即可。

## WebKit项目的代码结构

了解了以上的组件结构以后我们可以对照着这个结构寻找一下对应的代码。

```
-rw-r--r-- 1 mbbill 197121   2033 Apr 27 19:20 CMakeLists.txt       // CMake 脚本
-rw-r--r-- 1 mbbill 197121 279658 Apr 27 19:20 ChangeLog            // 修改记录
-rw-r--r-- 1 mbbill 197121 794798 Mar 23  2016 ChangeLog-2012-05-22 // 修改记录
-rw-r--r-- 1 mbbill 197121 782150 Mar 20  2019 ChangeLog-2018-01-01 // 修改记录
-rw-r--r-- 1 mbbill 197121  75864 Apr 27 19:20 Introduction.md
drwxr-xr-x 1 mbbill 197121      0 Apr 27 19:21 JSTests              // JavaScriptCore测试
drwxr-xr-x 1 mbbill 197121      0 Apr 27 19:21 LayoutTests          // Layout页面排版测试
-rw-r--r-- 1 mbbill 197121    607 Apr 27 19:22 Makefile
-rw-r--r-- 1 mbbill 197121   4511 Apr 27 19:22 Makefile.shared
drwxr-xr-x 1 mbbill 197121      0 Apr 27 19:22 ManualTests          // 人工测试
drwxr-xr-x 1 mbbill 197121      0 Apr 27 19:22 PerformanceTests     // 性能测试
-rw-r--r-- 1 mbbill 197121   6172 Apr 27 19:22 ReadMe.md
drwxr-xr-x 1 mbbill 197121      0 Apr 27 19:22 Source               // 所有代码
drwxr-xr-x 1 mbbill 197121      0 Apr 27 19:22 Tools                // 辅助工具，脚本等
drwxr-xr-x 1 mbbill 197121      0 Apr 27 19:22 WebDriverTests       // WebDriver 测试
drwxr-xr-x 1 mbbill 197121      0 Mar 20  2019 WebKit.xcworkspace
drwxr-xr-x 1 mbbill 197121      0 Oct  5  2020 WebKitBuild          // 某些平台的编译辅助
drwxr-xr-x 1 mbbill 197121      0 Apr 27 19:22 WebKitLibraries      // 某些平台的依赖库
drwxr-xr-x 1 mbbill 197121      0 Mar 20  2019 Websites             // WebKit.com 页面
drwxr-xr-x 1 mbbill 197121      0 Apr 27 19:22 resources            // 一些资源
```

上面我标注了主目录下的各种子目录用途。可以看到大多数都是测试和脚本等。而WebKit项目的核心代码全部在**Source**目录中。

下面我们来看一下Source目录里面的内容

```
-rw-r--r-- 1 mbbill 197121 1115 Apr 27 19:22 CMakeLists.txt
drwxr-xr-x 1 mbbill 197121    0 Apr 27 19:22 JavaScriptCore
-rw-r--r-- 1 mbbill 197121 1100 Apr 27 19:22 Makefile
-rw-r--r-- 1 mbbill 197121   66 Mar 23  2016 Makefile.shared
-rw-r--r-- 1 mbbill 197121 2283 Apr 27 19:22 PlatformGTK.cmake
-rw-r--r-- 1 mbbill 197121   43 Mar 23  2016 PlatformMac.cmake
-rw-r--r-- 1 mbbill 197121 1822 Apr 27 19:22 PlatformWPE.cmake
-rw-r--r-- 1 mbbill 197121  205 Apr 27 19:22 PlatformWin.cmake
drwxr-xr-x 1 mbbill 197121    0 Apr 27 19:22 ThirdParty
drwxr-xr-x 1 mbbill 197121    0 Apr 27 19:22 WTF
drwxr-xr-x 1 mbbill 197121    0 Apr 27 19:22 WebCore
drwxr-xr-x 1 mbbill 197121    0 Apr 27 19:22 WebDriver
drwxr-xr-x 1 mbbill 197121    0 Apr 27 19:22 WebInspectorUI
drwxr-xr-x 1 mbbill 197121    0 Apr 27 19:22 WebKit
drwxr-xr-x 1 mbbill 197121    0 Apr 27 19:22 WebKitLegacy
drwxr-xr-x 1 mbbill 197121    0 Apr 27 19:22 bmalloc
drwxr-xr-x 1 mbbill 197121    0 Apr 27 19:22 cmake
```

这里我们明显可以找到上文提到的各种组件，例如WTF，JavaScriptCore，WebCore，WebKit和WebKitLegacy等等。目录的分布非常清晰工整。剩下的一些目录功能如下：

- ThirdParty，编译WebKit需要的一些第三方库。注意并不是全部第三方库都在这个目录里。WebKit依赖的大多数库都需要平台本身提供。
- WebDriver，提供WebDriver的实现。
- WebInspectorUI，WebKit的Inspector本身是一个HTML页面，它所有的代码都在这个目录下。
- bmalloc，前文提到的一个内存分配库。它根据WebKit的内存使用特征来优化以达到比平台通用malloc更高的性能，同时它还为JSC的JIT编译器提供一些内存安全隔离功能。
- cmake，cmake的各种脚本。
- Platform***.cmake，每个平台特定的cmake文件。

### 各组件的内部构成

因为每个组件内部目录结构层次非常复杂，所以这部分会拆分成多个话题分别讨论。

