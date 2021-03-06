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

同样的方法，我们发现`jscmain`调用了`runJSC`，然后又进入`runInteractive`。最后也是最关键的，它调用了

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

如果对JavaScript有一定了解，在看到这几个参数的时候可能会有一种似曾相识的感觉。从用户的角度来看，JavaScript引擎本质上是一个巨大的状态机，它有一个循环在不停的执行着各种回调函数。我们写的每一行代码都会直接或者间接的被某个回调函数调用。我们自己也可以直接注册这种全局的回调。所以JS引擎的状态在每一次循环执行完毕以后都会回到循环的起始点。这时候由于之前一次调用，JS引擎内部的状态已经改变了。例如我们可以新建一个对象挂在`window`这样的全局对象下面。如果我们把所有能够从根对象访问到的东西看成一棵树，那么这棵树就是`ExecState`。

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

### 解释器 LLInt

如果你读完上面的部分，你可能会觉得既然JSC的每一个级别（Tier）都可以单独执行，那么我如果能把JSC砍到只剩一个LLInt那不是应该很容易移植吗，因为一般情况下解释器不涉及到动态申请可执行内存，也不会需要平台相关的汇编。它可以看成一个C/C++实现的巨大的`switch-case`。事实上的确是这样的，不过也并不是这么简单，后面我们会详细探讨这个问题。

对嵌入式系统来说很多时候需要在限定的二进制尺寸下完成特定的功能。因为嵌入式设备有各方面的限制，不可能拥有像桌面电脑一样丰富的资源。从这一点来看LLInt似乎非常的合适——它能砍掉非常多的代码，几乎所有JIT相关的都不再需要。它可以节省很多动态内存，因为JIT是吃内存大户。同时它在功能上和高级别的Tier——DFG、FTL JIT几乎没有差别。“**几乎**” 是指像WASM这样必须依赖JIT的功能目前还没办法在JSC中解释执行，但是其他所有我们能想到的JavaScript功能它都是支持。

在深入到LLInt之前，我觉得可以先动手尝试一下JSC，运行一些简单的JavaScript。比如我们可以手动执行一下上面提到的JSC shell (`jsc.cpp`)，打几个命令看看输出。在能够运行JSC以后可以试着用`gdb`启动它，设几个断点然后跟踪一下执行流程。这对于了解代码非常有帮助。

编译和运行JSC shell可以参考WebKit官方的指南，我就不复制粘贴了。`Tools/scripts/run-jsc-stress-tests`是一个非常好的参考，可以看看它是怎么调用JSC来跑测试用例的。比如在这个脚本里面我们会发现JSC Shell支持 `--useLLInt=true`和 `--no-jit`，顾名思义就是关闭JIT然后只使用LLInt。

在看下面内容之前我非常建议先尝试运行和调试一下JSC。

如果试着单步执行的话，可能你会发现当你用gdb挂上了正确的symbol，加载了源代码以后还是会执行到一些看起来像汇编但又不是汇编的东西。像下面这样：

```
OFFLINE_ASM_OPCODE_LABEL(op_enter)
    "\tmovq 16(%rbp), %rdx\n"g
    "\tmovl 24(%rdx), %edx\n"g
    "\tsubq $4, %rdx\n"g
    "\tmovq %rbp, %rsi\n"g
    "\tsubq $32, %rsi\n"g
    "\ttestl %edx, %edx\n"g
    "\tjz " LOCAL_LABEL_STRING(_offlineasm_opEnterDone) "\n"
    "\tmovq $10, %rax\n"g
    "\tnegl %edx\n"g
    "\tmovslq %edx, %rdx\n"g
```

恭喜你已经找到了LLInt的bytecode代码。

#### LLInt目录

我们先打开JavaScriptCore目录下的`llint`子目录。目录里面文件不多，重要的是下面几个：

- LLIntCLoop.cpp // LLInt “CLoop”初始化代码
- LLIntEntrypoint.cpp // JavaScript进入LLInt的入口
- LLIntOffsetExtractor.cpp // 地址提取器
- LLIntSettingsExtractor.cpp // 设置提取器
- LowLevelInterpreter.asm 以及其他asm文件 // bytecode伪汇编实现
- LowLevelInterpreter.cpp // LLInt解释器主循环

上面这些东西看起来会比较奇怪，不用着急我们下面都会说到。

#### Bytecode

上面提到过JSC并不直接解释JavaScript，而是先通过parser把JavaScript转换成AST，然后再用Bytecompiler把AST变成bytecode。这些部分并不是LLInt的一部分，因为他们只是执行之前的解析和转换步骤。

我们可以在JavaScriptCore目录下可以找到`parser`和`bytecode`目录分别对应上述功能。现在我们来看一下bytecode到底是什么样子。

向上翻半页，回到我们之前看到过的那一段奇怪的汇编会发现开头的一段看起来似乎不太一样：

```
OFFLINE_ASM_OPCODE_LABEL(op_enter)
```

这其实是一个宏，它的展开也很简单——它会变成一个label。你们可以在`LowLevelInterpreter.cpp`里找到这段宏的定义。

```c++
#define DISPATCH_OPCODE() goto dispatchOpcode

#define DEFINE_OPCODE(__opcode) \
        case __opcode: \
        __opcode: \
            RECORD_OPCODE_STATS(__opcode);
```

看到这个label是不是忽然想到了什么？解释器大循环和`switch-case`？没错！所有的bytecode最终都会变成这样的汇编，然后被塞到那个超大的`switch-case`循环里面去。

我们再来看这一行，里面还有一个`op_enter`，看起来像字节码。它就是JSC的Opcode(或者bytecode)。不过这里的汇编代码都是生成出来的。你如果在gdb里面追踪这个代码的路径会发现这是个头文件，叫做`LLIntAssembly.h`。并且它是在编译生成的某个目录下面，而不是在源代码目录下面。它的生成过程有点复杂，我们一步步来看。

首先我们到`bytecode`目录下面找一个叫做`BytecodeList.rb`的文件。打开后发现这个文件中罗列了非常多的opcode**声明**。例如：

```
op :enter

op :get_scope,
    args: {
        dst: VirtualRegister
    }

op :create_direct_arguments,
    args: {
        dst: VirtualRegister,
    }
```

我们上面看到的`op_enter`就在这里。它是一个没有参数的opcode。后面的很多opcode在args里面会有不一样数量的参数和定义。所以我们在这儿看到的是opcode的**声明**，也就是所有对于使用者来说需要的信息。而opcode的真正定义在别的地方。为了便于理解，参考下面的图：

```
JavaScript ---> Parser -> AST -> Bytecompiler -> opcode ---> LLInt
// 使用者 <---> opcode <---> 执行者
```

Bytecompiler左边把JavaScript变成opcode，所以它不需要知道每个opcode做什么。

LLInt按顺序执行opcode，所以它需要定义每个opcode的行为。

顺理成章的，我们知道应该去哪里找opcode的定义了——LLInt目录里。

不过在进入LLInt目录之前我们再回来看一下上面这个`BytecodeList.rb`的文件，因为我们还不清楚这个Ruby文件怎么最终会和JSC的其他代码编译到一起去的。如果我们搜索这个文件名，会发现在`JavaScriptCore/CMakeLists.txt`里有这样一行：

```cmake
add_custom_command(
    OUTPUT ${DERIVED_SOURCES_JAVASCRIPTCORE_DIR}/Bytecodes.h ${DERIVED_SOURCES_JAVASCRIPTCORE_DIR}/InitBytecodes.asm ${DERIVED_SOURCES_JAVASCRIPTCORE_DIR}/BytecodeStructs.h ${DERIVED_SOURCES_JAVASCRIPTCORE_DIR}/BytecodeIndices.h
    MAIN_DEPENDENCY ${JAVASCRIPTCORE_DIR}/generator/main.rb
    DEPENDS ${GENERATOR} bytecode/BytecodeList.rb
    COMMAND ${RUBY_EXECUTABLE} ${JAVASCRIPTCORE_DIR}/generator/main.rb --bytecodes_h ${DERIVED_SOURCES_JAVASCRIPTCORE_DIR}/Bytecodes.h --init_bytecodes_asm ${DERIVED_SOURCES_JAVASCRIPTCORE_DIR}/InitBytecodes.asm --bytecode_structs_h ${DERIVED_SOURCES_JAVASCRIPTCORE_DIR}/BytecodeStructs.h --bytecode_indices_h ${DERIVED_SOURCES_JAVASCRIPTCORE_DIR}/BytecodeIndices.h ${JAVASCRIPTCORE_DIR}/bytecode/BytecodeList.rb
    VERBATIM)
```

这里可以看到这个ruby文件会用于生成`Bytecodes.h`。我们打开`Bytecodes.h`会发现它里面用宏的方式定义了每一个opcode。这样一方面可以解耦编译器的前端和后端，另一方面可以很便捷的自动生成大量bytecode相关的代码，而不需要手动一个个的重复添加。

#### Bytecode的实现

上面贴的几个opcode里面有一个是这样的`op :get_scope,`，那我们就搜索一下LLInt目录看看有没有相似的。于是搜索`get_scope`会发现我们找到了竟然不止一个地方：

```asm
// 在LowLevelInterpreter32_64.asm 中

llintOpWithReturn(op_get_scope, OpGetScope, macro (size, get, dispatch, return)
    loadi Callee + PayloadOffset[cfr], t0
    loadp JSCallee::m_scope[t0], t0
    return (CellTag, t0)
end)
```

同时

```asm
// 在LowLevelInterpreter64.asm 中

llintOpWithReturn(op_get_scope, OpGetScope, macro (size, get, dispatch, return)
    loadp Callee[cfr], t0
    loadp JSCallee::m_scope[t0], t0
    return(t0)
end)
```

两个地方的实现看起来相似，却又不太一样。如果我们列出LLInt目录下面的所有`asm`文件，会发现一共有三个：

- LowLevelInterpreter.asm
- LowLevelInterpreter32_64.asm
- LowLevelInterpreter64.asm

这里的`32_64`和`64`与一个非常重要的宏有关：`JSVALUE32_64`

根据ECMA-262规范，JavaScript中的数字必须至少是IEEE754格式双精度浮点。对应的C/C++浮点数类型就是`double`。在JSC中JSValue不仅会编码`double`数值，还会同时包含指针以及整数的payload。在64位系统上`double`和指针都是64位的，同时我们也可以用64位整数运算来得到最大性能，这些都可以很方便的扔进一个寄存器中。但是在32位系统上事情就变得复杂起来。在32位系统中寄存器是32位的，指针也是32位的，double依然是64。为了解决这些问题，JSC定义了三种不同的JSValue格式：

- JSValue32，32位JSValue，适配32位系统，在目前代码中已被删除。
- JSValue32_64，64位JSValue，适配32位系统。
- JSValue64，64位JSValue，适配64位系统。

上面三个`asm`文件就是对应的目前剩余的两种JSValue配置。而`LowLevelInterpreter.asm`则包含大多数不受影响的代码。

我们再回头看一下上面两段代码的区别：

`loadi Callee + PayloadOffset[cfr], t0    //LowLevelInterpreter32_64.asm` 

`loadp Callee[cfr], t0                    // LowLevelInterpreter64.asm`

32位的实现增加了一个`PayloadOffset`的寻址，而这个`PayloadOffset`恰恰就定义在`JavaScriptCore/runtime/JSCJSValue.h`中。

有兴趣的可以继续研究一下这个重要的头文件。

#### 伪汇编到真汇编的转换

我们上面看到的这几个文件都是所谓的LLInt伪汇编。它的主要目的是：

1. 平台无关
2. 可以方便的编译成平台/编译器相关的汇编，甚至到C语言。

既然要做转换，那么肯定会有一个工具来完成这个对应不同平台的转换的任务，这就是`offlineasm`的目标。在JavaScriptCore目录下面你可以找到这个子目录，里面都是Ruby语言编写的脚本。这些脚本在编译JSC的过程中会被CMake调用，然后生成`LLIntAssembly.h`。`LLIntAssembly.h`头文件里都是转换后的内联汇编，在被`LowLevelInterpreter.cpp`的`switch-case`大循环包含以后与其他C++代码编译到一起。

如果我们去检查生成以后的内联汇编会看到它的注释会指明从哪里编译而来：

```c++
  OFFLINE_ASM_LOCAL_LABEL(_offlineasm_llintOpWithReturn__llintOp__commonOp__fn__fn__makeReturn__fn__fn__loadConstantOrVariable__size__k__13_load__constant)
    "\tmovq 16(%rbp), %rdx\n"                                // LowLevelInterpreter64.asm:438
    "\tmovq 184(%rdx), %rdx\n"                               // LowLevelInterpreter64.asm:439
    "\tmovq -128(%rdx, %rsi, 8), %rdx\n"                     // LowLevelInterpreter64.asm:440
```

这三行分别指向了`LowLevelInterpreter64.asm`的438到440行，如下：

```assembly
loadp CodeBlock[cfr], value
loadp CodeBlock::m_constantRegisters + VectorBufferOffset[value], value
loadq -(FirstConstantRegisterIndexNarrow * 8)[value, index, 8], value
```

#### 汇编当中的偏移量

 如果我们回想一下最初的目标——移植JSC，那么我们现在到了哪一步呢？似乎越走越远，而且越来越复杂。即使我们砍掉了JIT只留下解释器，我们依然碰到了复杂的汇编以及各种平台相关的问题。

如果我们移植的是一个只有C代码的普通项目，可能会比较轻松。如果当中嵌着一些汇编，事情会变得稍微麻烦一些。如果这些汇编又是动态生成的，并且通过各种复杂脚本经过很多奇怪的步骤最终得到一个看起来不知道在干什么的东西，这时候如果遇到问题的话调试会变得极其棘手。因为定位错误会变得非常困难。我们拿上面那一段生成的汇编举个例子：

考虑这一行，

```assembly
"\tmovq 184(%rdx), %rdx\n"    // LowLevelInterpreter64.asm:439
```

这里的184是哪里来的呢？

我们知道的是根据后面的注释，它指向`LowLevelInterpreter64.asm:439`行，也就是

````assembly
loadp CodeBlock::m_constantRegisters + VectorBufferOffset[value], value
````

我先来解释一下汇编：

- `loadp`，加载一个指针大小的数到目的寄存器，对应的汇编指令是`movq`
- `loadp`的第一个操作数`CodeBlock::m_constantRegisters + VectorBufferOffset[value]`。意为以value为基地址找到偏移量为`CodeBlock::m_constantRegisters + VectorBufferOffset`的位置。注意这里`VectorBufferOffset` 在`LowLevelInterpreter.asm`中的定义是 `const VectorBufferOffset = Vector::m_buffer`。
- 所以上面这一条伪汇编实质上对应的C++代码就是 `value = *(((*CodeBlock)value)->m_constantRegisters->m_buffer)`.

注意这里为汇编和生成的内联汇编都是AT&T syntax，source before destination.

所以这一行汇编其实就是**通过偏移量**去一个C++的类里面拿了一个数放到了寄存器里。注意我们拿的这个数是`m_buffer`，它是一个指针，指向vector真正存储内容的地方。因为vector的内容是在heap中分配的，我们显然是不可能通过偏移量直接拿到vector内部的数的。用语言描述这段代码就是：拿到value指针，它的类型是`CodeBlock`，我们根据`CodeBlock`中成员变量`m_constantRegisters`的偏移加上`Vector`中数据指针的偏移`m_buffer`找到`m_buffer`相对`CodeBlock`的累计偏移，取到这个指针，然后放到value中去。

偏移量的累加很容易理解：

```c++
struct A {
    int x;
    int y;
};
struct B {
    int z;
    A a;
};
struct B b;
// b.a.y 等同于 &b + (offsetof(B.a) + offsetof(A.y))
```

从上面编译结果来看这个偏移量是**184**，所以很显然`m_constantRegisters`和`m_buffer`各自偏移量相加应该是184，但是我们目前还不知道它是怎么被`offlineasm`这个工具算出来的。

不过可能有些人已经开始隐隐约约感觉到一些问题，因为184这个数是非常有可能变化的，因为一个成员变量在对象内部的偏移量很有可能受到各种条件的影响，包括：

- 编译器优化导致的struct layout变化。
- 对齐设置。
- 如果对象中包含标准库中的对象，那么这些对象尺寸变化也会有影响。
- 如果对象中包含条件编译的部分，在条件满足或者不满足的时候对象尺寸和对其会变化。
- 等等

所以在我们写C/C++的时候是绝对不可能用一个184来代替`offsetof`的，太危险。但是汇编里面我们没有办法这样做，所以只能在编译的时候动态的生成这个数值。

那么问题来了，如果是你的话，你会怎样写`offlineasm`这个工具来转换出184这个数？不要急着往下看，可以先思考一下。

#### 从对象中提取偏移量

可能你已经想到了，如果需要一个非常可靠的方法来提取偏移量，唯一可行的办法就是在最终生成的二进制文件中输出这个偏移。比如上面我们的`struct A`和`struct B`的例子里面，我们可以在别的地方直接把他们内部成员的地址打印出来。但是在这里我们需要的是一个内联汇编，它本身就是用来编译我们最终需要的二进制的一部分，所以这就变成鸡和蛋的问题了——我们需要一个二进制文件来输出偏移量，同时我们需要这个偏移量来编译这个二进制。

这里其实还有一个问题。假设我们现在用的是交叉编译器，比如从X86_64架构的Linux上面交叉编译树莓派上的armv7a版本。这时候我们是没办法直接在Linux host上面运行目标程序的，也就是说“输出偏移”这一步我们需要换一个方法，而不能直接依靠目标平台的可执行文件来给我们打印。毕竟在编译过程中出现这种需要在目标平台执行程序的步骤就太过麻烦了。更何况有些目标平台未必可以做到控制台输出，甚至目标平台硬件都还不存在。

#### LLIntOffsetExtractor

JSC的解决方法是这样的

1. `offlineasm`工具会处理一遍所有伪汇编asm文件，汇整出asm文件当中所有访问到的偏移量，并且整理成一张表。
   1. 实际操作中这张表里不仅仅是偏移(offset)，还会有常量定义(constexpr)和对象大小(sizeof)
2. `offlineasm`会用这张表生成`LLIntDesiredOffsets.h`头文件。
3. JSC会将`LLIntOffsetsExtractor.cpp`编译成目标平台的可执行文件，注意这里会使用完全相同的编译器与编译开关以保证这个二进制文件和编译JSC最终得到的二进制具有相同的layout。
4. `offlineasm`**不会**尝试去执行上面这个可执行文件，因为它是目标平台的二进制。而是会读取这个文件内容，找到上面那张表里面每一项对应的数值。

步骤4可能看起来比较困难，因为我们并没有办法知道目标平台二进制到底是什么格式的，毕竟交叉编译器和操作系统的种类不计其数。所以这里JSC使用了一个手段，就是将这些静态数据用magic number包围起来，然后`offlineasm`只会将目标可执行文件当作二进制数据读取，找到其中的magic number，就可以把中间包含的数据拿出来了。除了大小端导致的magic number顺序问题以外这种方法相对来说是比较可靠的。具体细节可以参考`offlineasm/asm.rb`和`offlineasm/offsets.rb`。

这部分的细节其实还有很多，比如存在另一个工具`LLIntSettingsExtractor`来解决`offlineasm`中的设置问题。有兴趣的可以自己深入研究。

#### CLoop

如果你还记得上面提到过的伪汇编“可以方便的编译成平台/编译器相关的汇编，甚至到C语言。”相信你或许会有些疑惑。我们来看下面这一段代码：

```c++
OFFLINE_ASM_LOCAL_LABEL(_offlineasm_llintOp__commonOp__fn__fn__loadConstantOrVariable__size__k__load__constant)
    t1 = *CAST<intptr_t*>((cfr.i8p() + 16)); // LowLevelInterpreter64.asm:438
    t1 = *CAST<intptr_t*>((t1.i8p() + 184)); // LowLevelInterpreter64.asm:439
    t1 = *CAST<int64_t*>(t1.i8p() + (t0.i() << 3) + intptr_t(-128)); // LowLevelInterpreter64.asm:440
```

同样是上面贴出来的三行汇编，不过这里的写法似乎有一点不同。

我再把上面的为汇编以及编译后的内联汇编贴出来对比一下：

```assembly
loadp CodeBlock[cfr], value
loadp CodeBlock::m_constantRegisters + VectorBufferOffset[value], value
loadq -(FirstConstantRegisterIndexNarrow * 8)[value, index, 8], value
```

```c++
OFFLINE_ASM_LOCAL_LABEL(_offlineasm_llintOpWithReturn__llintOp__commonOp__fn__fn__makeReturn__fn__fn__loadConstantOrVariable__size__k__13_load__constant)
    "\tmovq 16(%rbp), %rdx\n"                                // LowLevelInterpreter64.asm:438
    "\tmovq 184(%rdx), %rdx\n"                               // LowLevelInterpreter64.asm:439
    "\tmovq -128(%rdx, %rsi, 8), %rdx\n"                     // LowLevelInterpreter64.asm:440
```

看出来了吗？

在CLoop的代码中，`t0`,`t1`,`cfr`这些局部变量对应的是汇编中的寄存器。也就是说CLoop用C/C++语言的功能比如局部变量，整数运算，移位，强制转换这些操作来模拟了汇编代码的行为。

如果我们打开`llint/LowLevelInterpreter.cpp`会发现下面这段代码：

```c++
//============================================================================
// CLoopRegister is the storage for an emulated CPU register.
// It defines the policy of how ints smaller than intptr_t are packed into the
// pseudo register, as well as hides endianness differences.

class CLoopRegister {
public:
    ALWAYS_INLINE intptr_t i() const { return m_value; };
    ALWAYS_INLINE uintptr_t u() const { return m_value; }
    ALWAYS_INLINE int32_t i32() const { return m_value; }
    ALWAYS_INLINE uint32_t u32() const { return m_value; }
    ALWAYS_INLINE int8_t i8() const { return m_value; }
    ALWAYS_INLINE uint8_t u8() const { return m_value; }

    ALWAYS_INLINE intptr_t* ip() const { return bitwise_cast<intptr_t*>(m_value); }
    ALWAYS_INLINE int8_t* i8p() const { return bitwise_cast<int8_t*>(m_value); }
    ALWAYS_INLINE void* vp() const { return bitwise_cast<void*>(m_value); }
    ALWAYS_INLINE const void* cvp() const { return bitwise_cast<const void*>(m_value); }
...
```

我们折腾了这么久的汇编，终于又回到C++了，而且这一次通过CLoop似乎我们已经可以做到整个JSC都统一到C++代码上。从可移植性的角度看简直是一个巨大的进步。

但是呢，这里还有一个问题：就是这个神奇的184依然存在：

```c++
t1 = *CAST<intptr_t*>((t1.i8p() + 184)); // LowLevelInterpreter64.asm:439
```

也就是说这一段生成的CLoop“汇编”虽然是C语言模拟的，它依然**不能跨平台**，因为换个不同的硬件这里很可能就不是这184了。如果我们能把这行代码变成下面这样才是完美跨平台的：

```c++
t1 = *CAST<intptr_t*>((t1.i8p() + (OFFSETOF_PRIVATE(CodeBlock, m_constantRegisters) + OFFSETOF_PRIVATE(Vector, m_buffer))));
```

这里假设我们有一个宏，叫做`OFFSETOF_PRIVATE`，它的功能和`offsetof`类似，且能用在`private`成员上。那么我们得到的这一段新的代码才是真正跨平台的，而且我们甚至都不再需要上面所有`offlineasm`的那些繁琐步骤了。

具体怎样实现可以参考附录[打造一个可移植的JS引擎](../Appendix/PortableJSEngine.md)

### JIT

JSC中的JIT(Just In Time compiler)在近十年经历过非常大的改进。目前主要包括：

- DFG(Data Flow Graph)，中等性能，低延迟。
- FTL(Faster Than Light)，高性能，高延迟。

FTL在2014年之前使用的是LLVM后端，14年以后改为了B3(Bare Bones Backend)。后又陆续加入了很多其他的优化以及WASM支持。

因为这些JIT最终都会在运行的时候实时编译可执行代码，所以这样也会带来一些限制：

- 目标平台需要支持可执行内存分配。有些平台因为安全原因这种API是不开放的，这时候所有JIT都没有办法运行。
- 目标平台的处理器必须在JSC的assembler支持范围内。目前assembler支持的架构包括：
  - ARMv7
  - ARM64
  - X86
  - X86_64
  - MIPS

在JSC中添加一个新的CPU汇编器工作量是巨大的，所以如果目标CPU不在上述列表中，那么也就意味着只能依靠解释器了。

### Runtime

众所周知JavaScript是一个需要runtime的语言。这个runtime提供了很多JavaScript语言的规定了，但是却无法用JavaScript本身实现的功能。比如获取系统时间，垃圾收集，异常处理等等。在JSC中Runtime的大多数功能都在`runtime`这个目录下实现了。

例如在JavaScript global namespace下面能访问到的大多数对象都在`runtime`目录下以`JS`开头的文件中定义。例如

- JSFunction
- JSObject
- JSPromise

等等。

不过这里也有一些例外，比如`window`常常被认为是JavaScript的一个全局对象，但实际上它是DOM注入到JavaScript引擎全局环境里去的。它是DOM在JavaScript中的API入口，而不是JavaScript语言本身的一部分。后面章节谈到WebCore和IDL的时候会继续深入这个话题。

#### `*.lut.h`的生成

我们知道JavaScript的runtime中包含着很多内置对象，每个对象都有各种函数或者成员变量。这些内置对象其实本质上都是一个个的hash table。实际上所有JavaScript的object都是hash table，里面存的是key-value pair的成员。

我们同时又知道这些内置的对象一般是不变的，那么对于这些“不变”的hash table最好的办法就是使用静态hash。

在CMakeLists.txt中有这样一行：

```cmake
foreach (_file ${JavaScriptCore_OBJECT_LUT_SOURCES})
    get_filename_component(_name ${_file} NAME_WE)
    GENERATE_HASH_LUT(${CMAKE_CURRENT_SOURCE_DIR}/${_file} ${DERIVED_SOURCES_JAVASCRIPTCORE_DIR}/${_name}.lut.h)
endforeach ()
```

它调用了同级别目录下面的`create_hash_table`脚本来生成这些`.lut.h`头文件。

如果仔细观察的话会发现这些`.lut.h`大多数要么是`constructor`，要么是`prototype`。这其实是对应的JavaScript中的动态和静态成员函数。

在JavaScript中我们知道每个可以new出来的对象都有一个`constructor`和一个`prototype`。`constructor`本质上是一个`Function`对象.我们可以对它使用`new`，所以它是一个构造函数。那么放在这个构造函数里面的成员就是*静态成员*。比如我们不仅可以：

```javascript
new Date();
```

还可以直接使用这个`constructor`的静态成员：

```javascript
Date.now();
// 或者
Date.parse('01 Jan 1970 00:00:00 GMT');
```

这些诸如`Date.now()`的静态成员和其他*成员函数*并不在同一个地方。比如

```javascript
new Date().getDate();
```

我们实际调用的是

```javascript
Date.prototype.getDate();
```

所以对于每一个runtime中的对象来说，我们需要两个对象来支持他们：

- `prototype object` - 存放所有对象成员函数。
- `constructor object` - 存放所有静态函数。

也就是为什么我们能看到生成的lut文件里面有`DateConstructor.lut.h`和`DatePrototype.lut.h`

附录里有一个我画的JavaScript对象关系图以帮助理解，这里涉及到的其他JavaScript细节就不展开了。

#### JIT Profiler & Sampling Profiler

`Profiler`是JSC的JIT需要的一个比较关键的功能。我们知道JIT的内存开销是比较大的。同时JIT的编译过程，尤其是FTL JIT编译是比较慢的。而热点代码往往只占所有代码的一小部分。如果可以知道哪些代码比较“热”，那么编译这些热点很显然会事半功倍。注意这里的Profiler是用来优化JIT的，而不是另一个给inspector用的`sampling profiler`。

关于`sampling profiler`可以参考 https://webkit.org/blog/6539/introducing-jscs-new-sampling-profiler/

`sampling profiler`和JIT没有关系，它的作用是每隔一段时间提取一次当前堆栈，然后汇总信息发送给inspector，这样在inspector的timeline view里面就可以直观的看到那些代码被执行较多次数。这个功能是`runtime`的一部分，因此也就在`runtime`目录下面。

### Yarr

yarr是个正则表达式引擎，它本身也包含一个正则表达式JIT，可以即时编译正则表达式来提供最大性能。代码在`yarr`目录下面。

## 移植

到目前我们基本上已经了解了很多JSC和移植有关的基本知识。下面我们把这些汇总起来，做一个最简单的移植计划。

### 解释器和JIT

对于LLInt和JIT的处理，我们可以先从LLInt入手，暂时关闭所有JIT功能。这样在第一步完成以后我们至少可以有一个能够正常运行解释器的JSC。后续再添加JIT以后出现问题调试起来不那么棘手。

JSC提供很多宏开关可以让我们比较方便的做到这一点：

- `ENABLE_JIT=0`
- `ENABLE_DFG_JIT=0`
- `ENABLE_FTL_JIT=0`
- `ENABLE_B3_JIT=0`

### LLInt的工作模式

对于LLInt我们有两种选择：

- 在JSC支持的CPU上可以将伪汇编代码编译为平台相关的汇编，获得100%解释器性能。
- 使用CLoop获得最大的兼容性，同时牺牲大约30%左右的解释器性能。

30%是我自己在多年前测试得到的结果，可能现在会很不一样。因为CLoop非常依赖C/C++编译器的优化能力，所以这些年编译器也在越来越聪明，可能差距会更小。

如果选择平台汇编，我们需要设置的宏：

- `ENABLE_C_LOOP=0`
- `ENABLE_ASSEMBLER=1`

如果选择CLoop，则

- `ENABLE_C_LOOP=1`
- `ENABLE_ASSEMBLER=0`

注意即使我们关闭了所有的JIT，LLInt依然对assembler是有依赖的，除非使用的是CLoop后端。而assembler目前只支持上一个章节中列举的5种CPU架构，所以如果想要移植JSC到一个不在列表中的CPU上，LLInt+CLoop是唯一的选择。

### CPU相关

如果我们选用LLInt+平台汇编，那么我们需要定义一些CPU相关的宏，例如:

- armv7
  - `WTF_CPU_ARM=1`
  - `WTF_CPU_ARM_VFP_V2=1`
  - `WTF_CPU_NEEDS_ALIGNED_ACCESS=1`
    - 可选
      - `WTF_CPU_ARM_NEON=1`
      - `WTF_CPU_ARM_VFP_V3_D32=1`
      - `WTF_CPU_ARM_THUMB2=1`
- arm64
  - `WTF_CPU_ARM64=1`
- x64
  - `WTF_CPU_X86_64=1`
- mips
  - `WTF_CPU_MIPS=1`

### WebAssembly

JSC的WebAssembly实现依赖JIT。既然我们暂时已经关闭了JIT，那么也需要:

`ENABLE_WEBASSEMBLY=0`

### 多线程的处理

JSC中有不少线程。除了JIT需要的那些以外还有几个比较重要的就是：

- Concurrent GC，执行并行mark-sweep垃圾回收的线程。
- Sampling profiler，定时采样JSC执行堆栈的线程。
- Inspector，支持inspector的线程。

这里面Concurrent GC 和Sampling profiler都需要可以暂停线程的能力。例如Sampling profiler在Linux上是通过给主线程发signal的方式将JavaScript当前执行暂停，然后进行堆栈采样和记录，最后再让主线程继续的方式来收集信息的。在不同的平台上这种暂停和继续线程的功能需要手动移植和实现。

### Allocator

JSC对堆内存也有一些要求，例如它需要能能够按页分配，并且需要可以做到Reserve and commit.

Reserve是对一个地址和一定长度的虚拟内存提出保留，但不一定会立刻占用。

Commit是在真正需要使用某一段内存的时候做“提交”，所以会真实占用系统内存。

很多嵌入式系统并不提供reserve&commit的能力。但是如果使用malloc来替代的话会造成比较大的浪费。在移植JSC的时候这也是需要考虑的一点。

## 附录

### JavaScript对象关系图

![image-20210521183809398](JSC.assets/image-20210521183809398.png)
