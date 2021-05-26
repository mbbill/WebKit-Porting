# WTF移植

WebKit对平台相关代码的管理可以说是有点混乱的。拿`wtf/`来说，平台相关的实现有可能在根目录某个`cpp`文件中以宏的形式出现，或者在某个子目录下面出现。子目录可能是某一个操作系统（如mac），也可能是某一类系统（如posix），甚至是某一个类库（如cocoa）。

例如`unix`, `linux`,`posix`同时出现在`wtf/`目录下面会让人感觉比较困惑。

解决这个问题有几个办法，下面我们就来梳理一下。

## PLATFORM() & OS()

这两个宏是最重要的平台相关的宏。一般来说`OS`宏后面跟操作系统的名字，比如

- `OS(LINUX)`
- `OS(UNIX)`
- `OS(WINDOWS)`
- `OS(DARWIN)`
- `OS(FUCHSIA)`
- `OS(OPENBSD)`
- `OS(NETBSD)`

等等。

`PLATFORM`宏就稍微有点混乱了。它一般指内核之上的东西，也就是用户态的支持库。例如

- `PLATFORM(GTK)`
- `PLATFORM(MAC)`
- `PLATFORM(IOS)`
- `PLATFORM(IOS_FAMILY)`
- `PLATFORM(WIN)`
- `PLATFORM(COCOA)`
- `PLATFORM(APPLETV)`
- `PLATFORM(WATCHOS)`
- `PLATFORM(WPE)`

移植WebKit一般都会从`OS(LINUX)`和`PLATFORM(WPE)`开始。所以解决上述问题的方法之一就是搜索这两个宏来找到所有我们可能会用到的平台相关代码。不过并不是所有平台代码都会被宏包围的，例如`wtf/linux`目录下面的几个文件都没有这个宏。所以这里我们还有第二个办法，就是从`cmake`脚本入手。

## `PlatformXXX.cmake`

我们能在`wtf/`目录下面找到不少`cmake`文件，例如`PlatformWPE.cmake`。打开以后会看到这个cmake文件里主要就是列举了所有WPE平台的头文件和代码。按照这个列表我们可以很方便的知道哪些代码被编译，哪些被忽略。例如：

```cmake
list(APPEND WTF_SOURCES
    generic/MainThreadGeneric.cpp
    generic/MemoryFootprintGeneric.cpp
    generic/WorkQueueGeneric.cpp

    glib/ChassisType.cpp
    glib/FileSystemGlib.cpp
    glib/GLibUtilities.cpp
    glib/GRefPtr.cpp
    glib/GSocketMonitor.cpp
    glib/RunLoopGLib.cpp
    glib/SocketConnection.cpp
    glib/URLGLib.cpp

    linux/CurrentProcessMemoryStatus.cpp

    posix/OSAllocatorPOSIX.cpp
    posix/ThreadingPOSIX.cpp

    text/unix/TextBreakIteratorInternalICUUnix.cpp

    unix/CPUTimeUnix.cpp
    unix/LanguageUnix.cpp
    unix/MemoryPressureHandlerUnix.cpp
    unix/UniStdExtrasUnix.cpp
)
```

可见它不仅用了部分posix和unix的代码，还用了glib。所以我们知道虽然WPE号称是给嵌入式用的移植，它依然是有不少依赖的。这个glib就是GTK的支持库（注意这不是glibc）。

## `Platform.h`

这个头文件的可读性为零。不过虽然我们没法去读它，在特定的时候还是需要参考一下这个文件里定义的东西，因为很多功能的开关都在这个头文件中根据平台设定来决定。

功能开关的*定义*大致有这么几种：

- `ENABLE_xxx`
- `USE_xxx`
- `HAVE_xxx`

他们分别对应下面几种*使用*方式：

- `ENABLE(xxx)`
- `USE(xxx)`
- `HAVE(xxx)`

例如`#define ENABLE_JIT 1`以后`ENABLE(JIT)`宏会返回1。以此类推。

所以当我们需要知道一个功能有没有被打开，办法之一就是在`Platform.h`中查找对应的宏定义。在查找的时候记得把括号替换成下划线。

## `config.h`

WebKit的每一个组件都有一个`config.h`。例如：

- `WTF/config.h`
- `WebCore/config.h`
- `JavaScriptCore/config.h`
- `WebKit/config.h`
- `WebDriver/config.h`

等等。

在代码里面几乎所有的`config.h`都是这样使用的：

`#include "config.h"`

所以在编译WebKit的过程中我们必须保证包含路径的优先级，使得当前组件的`config.h`被使用。一般来说WebKit的编译脚本因为使用Forwarding Headers不存在这个问题。但是如果我们移植的时候修改了编译脚本，或者使用了其他的编译工具来构建WebKit，那么这个问题会非常快的显现出来。

