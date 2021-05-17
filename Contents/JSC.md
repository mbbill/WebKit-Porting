# JavaScriptCore移植

JavaScriptCore是个非常复杂的JS引擎。在尝试移植之前我们先来了解一下它的大致架构，这样在遇到问题的时候可以知道去哪里修改。

## 背景知识

打开JavaScriptCore目录我们能发现非常多的子目录。不用着急，我们后面会一个个介绍每一个部分的用途。一般来说对于一个可执行文件的项目我们总是会找到一个`main`函数入口，对JSC来说也是这样。

### 入口

可能有人会奇怪，JSC不是一个库么，在Linux上它会编译成一个`so`文件供上层组件使用，那怎么会有`main`函数？

答案就是JSC的确是个库，但是它可以非常容易的编译成一个可执行文件单独运行 - 只需要外加一个文件`jsc.cpp`。

你可以在JavaScriptCore的根目录找到这个文件。不过这个单独执行的JSC只是设计用来运行测试用例的，并不等同于NodeJS。

现在让我们打开这个`jsc.cpp`然后找到`main`函数。

```c++
int main(int argc, char** argv)
{
#if OS(DARWIN) && CPU(ARM_THUMB2)
    // Enabled IEEE754 denormal support.
    fenv_t env;
    fegetenv( &env );
    env.__fpscr &= ~0x01000000u;
    fesetenv( &env );
#endif

#if OS(WINDOWS)
    // Cygwin calls ::SetErrorMode(SEM_FAILCRITICALERRORS), which we will inherit. This is bad for
    // testing/debugging, as it causes the post-mortem debugger not to be invoked. We reset the
    // error mode here to work around Cygwin's behavior. See <http://webkit.org/b/55222>.
    ::SetErrorMode(0);

    _setmode(_fileno(stdout), _O_BINARY);
    _setmode(_fileno(stderr), _O_BINARY);

#if defined(_DEBUG)
    _CrtSetReportFile(_CRT_WARN, _CRTDBG_FILE_STDERR);
    _CrtSetReportMode(_CRT_WARN, _CRTDBG_MODE_FILE);
    _CrtSetReportFile(_CRT_ERROR, _CRTDBG_FILE_STDERR);
    _CrtSetReportMode(_CRT_ERROR, _CRTDBG_MODE_FILE);
    _CrtSetReportFile(_CRT_ASSERT, _CRTDBG_FILE_STDERR);
    _CrtSetReportMode(_CRT_ASSERT, _CRTDBG_MODE_FILE);
#endif

    timeBeginPeriod(1);
#endif

#if PLATFORM(GTK)
    if (!setlocale(LC_ALL, ""))
        WTFLogAlways("Locale not supported by C library.\n\tUsing the fallback 'C' locale.");
#endif

    // Need to initialize WTF before we start any threads. Cannot initialize JSC
    // yet, since that would do somethings that we'd like to defer until after we
    // have a chance to parse options.
    WTF::initialize();
#if PLATFORM(COCOA)
    WTF::disableForwardingVPrintfStdErrToOSLog();
#endif

    // We can't use destructors in the following code because it uses Windows
    // Structured Exception Handling
    int res = EXIT_SUCCESS;
    TRY
        res = jscmain(argc, argv);
    EXCEPT(res = EXIT_EXCEPTION)
    finalizeStatsAtEndOfTesting();
    if (getenv("JS_SHELL_WAIT_FOR_INPUT_TO_EXIT")) {
        WTF::fastDisableScavenger();
        fprintf(stdout, "\njs shell waiting for input to exit\n");
        fflush(stdout);
        getc(stdin);
    }

    jscExit(res);
}

```

其中除了前面初始化部分以外最关键的一句是`jscmain(argc, argv);`

进入`jscmain`以后又是一堆初始化。注意前面初始化的是JSC的运行环境，例如WTF。这里的初始化的是对JSC内部对象。

同样的方法，我们发现`jscmain`调用了`runJSC`，然后又进入`runInteractive`，最后也是最关键的，它调用了

```c++
JSValue returnValue = evaluate(globalObject, jscSource(source, sourceOrigin), JSValue(), evaluationException);
```

这里再往后就已经不在`jsc.cpp`内部了。

由此可见这个`evaluate`是JSC的入口**之一**。JSC还有一些其他类似的入口，有兴趣的可以在后面我们谈到WebCore IDL绑定的时候研究一下。

我们再回来看这个入口函数。这个函数定义在`runtime/Completion.h`中。

```c++
JS_EXPORT_PRIVATE JSValue evaluate(ExecState*, const SourceCode&, JSValue thisValue, NakedPtr<Exception>& returnedException);
```

我们给他四个参数：

- 执行状态
- 代码
- this
- 异常

返回一个JSValue的值。

如果对JavaScript有一定了解，在看到这几个参数的时候可能会有一种似曾相识的感觉。从用户的角度来看，JavaScript引擎本质上是一个巨大的状态机，它有一个循环在不停的执行者各种回调函数。我们写的每一行代码都会直接或者间接的被某个回调函数调用。我们自己也可以直接注册这种全局的回调。所以JS引擎的状态在每一次循环执行完毕以后都会回到循环的起始点，这时候由于之前一次调用，JS引擎内部的状态已经改变了。例如我们可以新建一个对象挂在`window`这样的全局对象下面。如果我们把所有能够从根对象访问到的东西看成一棵树，那么这棵树就是`ExecState`。

然后我们还需要提供一个`thisValue`。这就得怪JavaScript语言本身的设计了。熟悉JS的同学可能都曾经有过被这种奇怪的this支配的恐惧把。所以因为JS对this的特殊绑定方式，我们不能像C/C++或者其他静态语言一样把执行过程绑定在一个全局的栈上，而需要在每次运行的时候动态绑定不同的this。

其他参数比较显而易见就不多介绍了。

### 执行流程

从[这里](http://www.filpizlo.com/slides/pizlo-splash2018-jsc-compiler-slides.pdf)借用一张图。它显示的是JSC完整的执行流程。

![image-20210516182443787](JSC.assets/image-20210516182443787.png)

这个图里面包含了目前JSC除了WASM以外几乎全部的执行流程。从高处看它主要包含下面几个方面：

- 从源代码到bytecode——Parser, Bytecompiler, Linker, etc.
- 解释器——LLINT(Low Level Interpreter)
- 多级即时编译器JIT——DFG，FTL(B3)，etc

这个架构其实是一个典型的编译器架构。比如目前的大多数编译器基本上都是这么几个阶段：

- 从代码到AST
- 从AST到bytecode或者bitcode，不同的编译器会有略不一样的选择。
- bytecode到assembly。有一次生成的，也有多次优化的。

比如基于LLVM的编译器多多少少都是这样的架构。

与传统编译器不同的是，JSC每一步都是可以执行的，而不像传统编译器大多数都以得到最终的可执行文件为目标。比如在上面图中我们可以单独运行LLInt，或者单独运行DFG，FTL等，对同一段代码他们都会得到同样的结果。可能有人会问那为什么不直接一步到位呢。

答案是需要在延迟和性能之间取一个平衡(Latency vs performance)。

![image-20210516183745068](JSC.assets/image-20210516183745068.png)

有兴趣的可以仔细读一下上面这个PDF。简单地说就是生成解释器需要的bytecode时间非常短，而生成高度优化的二进制需要时间。不仅仅是编译的时间，还有获取profile然后针对性优化的时间。同时JIT还会造成非常大的内存消耗。所以对于一些简单的，只执行一两次的代码，花很多时间和内存去做优化编译是得不偿失的。

### Interpreter & JIT

### LLInt

### JIT

### Runtime

### Yarr

