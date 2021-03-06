打造一个可移植的JavaScript引擎
============================
一般来说谈到JavaScript引擎我们自然会想到几大浏览器自带的那些——V8，JSC，等等。但是他们往往都是给对应的浏览器量身定做的。在我们希望把JavaScript用到自己的项目上的时候，那些引擎的可移植性表现的并不是很好。

项目对嵌入JS引擎的需求
----------------------
V8的API设计的不错，引擎本身也可以独立工作，比如Node.js，但是如果你把代码拿过来，想编译进自己的项目，可能会遇到一些麻烦；JSC也是如此。这些麻烦来自于引擎本身设计针对的场景和我们希望它适配环境之间的差距，比如：
- 我的项目可能运行在xxx处理器上。而V8/JSC不支持。
- 我的项目有独立的，统一的编译环境，而且我并不想迁就V8/JSC自己的编译系统。
- 我的平台不允许申请可执行内存空间。
- 我的平台内存空间受限。
- 我的平台需要使用自己独特的编译器，或者交叉编译器。

总之就是：我希望这个JS引擎就是一堆C/C++代码，可以直接拿来改改然后编译链接就好。除此以外，我还希望这个JS引擎对新的标准支持好一点，兼容性和稳定性好一点，然后最好可以和几大浏览器内嵌引擎在同一水平线上。

然而现实很残酷，主流的几大JS引擎并没有能直接满足这种要求的。

V8
--
V8的编译系统——GYP，GN并不是主要问题，这两种编译脚本很容易转换或者融合到其他编译体系中去。

V8的问题在于，它对平台的需求比较高：

### *JIT*
首先，V8是没有解释器的。即便是最近新加的Igintion，也只是用来辅助JIT的warm up，离开了JIT是不能工作的。于是这就要求这个平台必须能够支持*可执行内存*的申请，简单的说就是需要可以把PC寄存器指向JIT生成的那段machine code里面去。有些平台因为安全的关系会把这个功能关闭，然后基本上就彻底的封死了在上面跑V8的可能性。以前的IOS就是个例子。

### *对内存的需求*
在没有Ignition之前，V8在执行所有JS代码之前统统要编译一遍，这对内存的压力是很大的。而且在对hotspot持续优化的时候还会陆续的生成更多的代码段，可想而知内存消耗会怎样增长，这还不包括JS代码里面申请的内存。

### *编译器的需求*
V8对编译器的需求也是比较高的。从目前来看，一个支持C++11标准的编译器是少不了的。我不是很确定那些只支持部分C++11功能的老编译起是不是可以通过一些workaround来编V8，但是过程一定是很痛苦的。

### *所以V8不合适？*
也不一定。  
对于第一条——JIT对**可执行内存**的需求，V8其实是可以绕过去的。方法就是用它的simulator，比如ARM simulator。  
编译的时候如果`target_cpu`和`v8_target_cpu`不一致，则会启动target simulator模式：

- V8会生成目标CPU指令。
- JIT生成的代码段并不会用PC寄存器指过去执行。
- JIT生成的代码段会用模拟器去模拟每一条指令。

这里你可能会觉得简直多此一举，直接集成一个现成的ARM模拟器不就可以了。其实这个模拟器主要工作并不是用在生产上，而是测试和验证的时候用的。另外，JIT生成机器指令的时候并不会用到每一条指令，所以这里的模拟器并不是一个完整指令集模拟器，它只支持到JIT生成代码用到的那个集合。

另外，因为这个模拟起本身就是测试用的，所以它的性能**非常差**，而且内存占用也不少——因为它解释执行的是JIT后的机器码，而不是某种bytecode。

综上所述，你如果实在想在这种情况下使用V8的话，是可以的，只是会非常别扭，得不偿失。

JavaScriptCore
--------------
JavaScriptCore（后面简称JSC）的情况更有意思一点。大多数人可能会觉得JSC更像是Safari的一部分，很少看到有单独用它的场景。但是实际上JSC是可以单独编译运行的。

如果你拉下来一份WebKit代码，可以直接执行
```
Tools/Scripts/build-jsc
```
就可以编出来一份单独可执行的JSC。这个脚本会给你编译几个东西：JSC的依赖——WTF库之类的，JSC Framework本身，再加上一个简单的JSC shell。

### *JSC的解释器*
和V8不同，JSC最开始的时候只有一个解释器。（V8设计之初就是直接从JIT开始的）  
早期的JSC代码非常原始简单，从KJS fork出来的时候就寥寥几十个文件，后来JSC慢慢的加入了JIT（DFG），然后出现了更快的FTL，后面又加了更高级的解释器（LLINT）。慢慢的JSC变的很快，也很庞大了。最近加入的B3把LLVM扔了，换成了新写的后端，速度和性能又上去不少。

因为这些缘故，JSC里面生成和解释bytecode的逻辑一直都是存在的，所以可能有人会想，那我把JIT关掉，打开Interpreter不就OK了么？

图样！

### *LLINT*
JSC的解释器实现比较神奇。  
首先它其实并不是最早出现在JSC里的那个解释器，最早的那个当初叫做classic interpreter，在08、09年左右的时候被干掉了，换上了这个现在叫LLINT的新解释器。

从名字上看，它的全称是Low Level Interpreter。顾名思义——它比一般意义的解释器更*低级*一点。低在哪里呢？

**它的opcode handler是拿汇编写的**

虽然是汇编，但它并不是给每个平台各写一遍。LLINT的实现方式比较巧妙。

### *LLINT的实现*
Lexer到Bytecompiler这部分因为和主题关系不大，先掠过。  
我们先看看所谓汇编实现的opcode到底是什么。

LLINT的大多数代码在JSC目录的`llint`子目录下面。在这里你可以找到三个以asm结尾的文件，他们就是LLINT的伪汇编代码。
```
LowLevelInterpreter.asm
LowLevelInterpreter64.asm
LowLevelInterpreter32_64.asm
```
所谓伪汇编代码是指他们并不能直接用某种汇编器编译成机器码，而是要先用offline assembler翻译成某个平台、某种编译起支持的汇编代码，然后再编译成那个平台的机器码。

至于为什么会有三个asm文件，这里带64的是给64位平台用的，32_64是给32位平台用的。解释器本身是按照64位平台设计，所以在32位机器上需要*模拟*一下64位数据操作，于是有了这么奇怪的32_64的名字。

#### 伪汇编对数据的访问
在真实的汇编代码里*一般来说*没有所谓的变量。对”变量“的操作本质上是对一个内存地址的读写，所以一般指令以load或者store这样的名字开始，后面带上地址和寄存器。
在LLINT的伪汇编中，对数据的操作更像是某些内联汇编，可以用类似class::member来访问一个外部符号：
```
.copyArgsDone:
    if ARM64
        move sp, t4
        storep t4, VM::topCallFrame[vm]
    else
        storep sp, VM::topCallFrame[vm]
    end
```
比如这一段代码里的storep指令：把一个东西存到了一个地方，保存的位置是VM类的成员topCallFrame为起始，偏移vm的位置。  
这里VM是一个类，topCallFrame是类成员，那问题来了：把这段代码翻译成真实的汇编代码后，怎么表达这里的类对象偏移地址呢？  
先看一下在Mac上转换出来的汇编代码：
```
"\tmovq %rsp, 18384(%rsi)\n"
"\tmovq %rbp, 18376(%rsi)\n"
```
可以看到上面的类和成员都不见了，变成了一些hardcode在代码里面的数字。下面会详细说一下这些数字，确切地说是这些由成员变量的偏移来计算真实地址的过程。

#### 偏移量从哪里来？
首先我们需要知道这个偏移量为什么需要计算？有很多原因会导致一个类成员在类当中的偏移发生改变，比如：
- 成员的先后顺序发生改变
- 添加或者删除了部分成员
- 结构对齐变化
- 编译器对内存安排的变化
- 数据类型在不同平台的长度不同
- 编译起参数改变
- 等等

举个例子：
```c++
struct X {
    char a;
    int b;
    double c;
} x;
```
这里a，b和c相对于&c的偏移在大多数情况下不到编译完成是不能确定的。  
或许你会觉得用一个宏来计算偏移量不就可以了，比如想这样：
```c++
#define OBJECT_OFFSETOF(class, field) (reinterpret_cast<ptrdiff_t>(&(reinterpret_cast<class*>(0x4000)->field)) - 0x4000)
```
这是一个相对比较严谨的offsetof宏，可以用`OBJECT_OFFSETOF(X, a)`来获取a的相对偏移量。但在这里的问题是——你不能保证你所使用的汇编语言支持这样的宏，而事实上大多数都不支持。  

#### 小目标
讨论到这里，需要再回顾一下最初的目标：我们需要一个高度可移植的JS引擎，对编译器依赖降到最低，最好没有针对某个平台的汇编，最好没有对某种特定CPU的要求，最好没有动态生成的代码，最好在这些情况下性能还过得去。  
所以我们先定一个小目标：  
**一个没有汇编，关闭JIT，启动LLINT，并且支持其他各种最新JS规范的JSC。**  
然后看看这个小目标能不能达成。

#### 解决偏移地址的思路
虽然现在还在讨论汇编的事情，看起来和目标没太大关系，不过最后我们需要这些知识来理解最终的方法。  
如果你看懂了上一节里的问题：怎么在汇编代码里面获取对象真实地址的问题，那么可能你已经有一些思路了：*既然我们需要的是最重编译完成的二进制文件中的偏移量，那么我们先把JSC编译出来，再葱里面提取偏移不就可以？*  
可是，怎么”先把JSC编译出来“呢？这是个鸡生蛋还是蛋生鸡的问题，不过解决也不麻烦，我们要的只是偏移地址，所以先编译一个*迷你版JSC*，只包含偏移，拿到数据以后再去编译真正的JSC不就可以？  
*没错，事实上JSC就是这么干的*

#### JSCLLIntOffsetsExtractor
在JSC里面有一个内嵌的中间target，叫`JSCLLIntOffsetsExtractor`，完成的事情就是我们之前说的*迷你版JSC*的功能。

一方面，它非常小——一个c代码和一个头文件，生成一个binary。另一方面，它又包含了所有需要的偏移地址，而且最终要的是和JSC目标文件一致的偏移地址。

它的生成过程比较蹊跷，我们一步步来看。

#### 生成LLIntDesiredOffsets.h
这个所谓迷你版JSC最重要的功能就是提供对象偏移，所以毫无疑问它需要能包含所有伪汇编代码里访问到的偏移量。所以这里需要一个媒介来完成这件事，这个媒介就是头文件`LLIntDesiredOffsets.h`
```
offlineasm/generate_offset_extractor.rb llint/LowLevelInterpreter.asm
```
这里`generate_offset_extractor.rb`是生成器，它从`LowLevelInterpreter.asm`里面提取所有访问到的对象信息，然后集中写入`LLIntDesiredOffsets.h`。  
这个命令之所以没有把前面提到的另外两个asm文件放进去，是因为这两个文件其实是在主汇编文件中包含进去的。
```
if JSVALUE64
    include LowLevelInterpreter64
else
    include LowLevelInterpreter32_64
end
```
`generate_offset_extractor.rb`是offlineasm的一部分功能，后面我们会说到offlineasm的另一部分功能——生成真实的汇编代码。 

#### 编译JSCLLIntOffsetsExtractor
这一步比较简单，只涉及到一个源代码：`JSCLLIntOffsetsExtractor.cpp`。这里最关键的是要**保证我们前面提到的所有可能导致地址偏移出现变化的部分都必须和最终JSC目标一致**。比如地址对齐需要一致，编译参数必须一致，等等。这也是为什么这一步不能把结果简单的check in进去，而必须在编译的时候动态执行一遍。

这一步生成的结果一般来说是一个可执行文件。不过如果不做链接，直接对obj操作也是可以的，读者可以自己思考下为什么。

#### 提取offset
```makefile
DerivedSources/JavaScriptCore/LLIntAssembly.h: Programs/LLIntOffsetsExtractor$(EXEEXT)
	$(AM_V_GEN)$(RUBY) $(srcdir)/Source/JavaScriptCore/offlineasm/asm.rb $(srcdir)/Source/JavaScriptCore/llint/LowLevelInterpreter.asm Programs/LLIntOffsetsExtractor$(EXEEXT) $@

```
这一步涉及到三个文件：
- *LLIntAssembly.h*  
这是生成的最终汇编文件，每个平台不一样，不同cpu当然也不同。这里虽然是`.h`但是里面是一个巨大的汇编字符串。在Windows上这一步略有不同，用的不是头文件而是masm的代码。
- *LLIntOffsetsExtractor*和*LowLevelInterpreter.asm*  
这俩我们之前就已经谈到过他们的功能，一个是伪汇编，一个提供偏移。
- *asm.rb*  
汇编代码生成器。是offlineasm的另一个功能——用上面两个东西作为输入，提取偏移替换伪汇编中的符号，然后生成真实的汇编。

至此，最终我们需要的汇编代码`LLIntAssembly.h`生成完毕，剩余的JSC代码可以依赖这个头文件来继续编译，直至最终链接成完整的JSC目标。

### *干掉汇编*
分析完上面所有的内容，我们似乎离那个小目标还是挺远。一方面我们还是依赖着汇编，另一方面这个过程也特别麻烦。有没有什么办法可以把这些麻烦事都跳过去呢？  
答案是有的，这要靠LLInt的一个特殊功能——**CLoop**

#### CLoop
CLoop是LLInt众多后端的其中一种。所谓后端，就是对某种特定CPU生成指令的功能。在offlineasm目录下可以找到例如`armv7.rb`，`x86.rb`等，他们都是后端的实现。在`backends.rb`中可以看到这样的列表：
```ruby
BACKENDS =
    [
     "X86",
     "X86_64",
     "ARMv7",
     "C_LOOP"
    ]
```
到这里估计你已经明白CLoop大概是什么了，从它的名字上可以猜出来——它针对的CPU是**C语言编译器**——一个用C语言*虚拟*的CPU。

你可能会好奇这个C语言后端到底是什么样的，贴一段代码理解起来会很容易：
```c++
OFFLINE_ASM_OPCODE_LABEL(op_jneq_ptr)
    t0.i = *CAST<int32_t*>(pcBase.i8p + (pc.i << 3) + intptr_t(0x8));
    t1.i = *CAST<int32_t*>(pcBase.i8p + (pc.i << 3) + intptr_t(0x10));
    t2.i = *CAST<intptr_t*>(cfr.i8p + 16);
    t2.i = *CAST<intptr_t*>(t2.i8p + 8);
    t1.i = *CAST<intptr_t*>(t2.i8p + (t1.i << 3) + intptr_t(0x4f8));
    if (t1.i != *CAST<intptr_t*>(cfr.i8p + (t0.i << 3)))
        goto _offlineasm_opJneqPtrTarget;
    pc.i = pc.i + intptr_t(0x4);
    opcode = *CAST<Opcode*>(pcBase.i8p + (pc.i << 3));
    DISPATCH_OPCODE();
```
这段看起来像“汇编”的东西本质上还是一段C代码。它只不过是用结构和对象来模拟了寄存器。比如pc，t0，t1，t2都是寄存器。

其实这个所谓CLoop后端最开始就是拿armv7后端改出来的，可以看到里面还有一些arm的影子。

不过遗憾的是这里面的偏移量仍然还是magic number。

这个CLoop后端一般来说是不用在生产上的，但是它也是可以完全通过所有测试的。而且根据我的测试，他在性能上大概是汇编后端的75%左右，也是相当可以了。

#### 靠近小目标
我们之前遇到的一大阻碍——各个平台的汇编代码的问题现在看似已经有了解，找到一个跨平台的C语言实现的后端来解决为每个CPU指令集编写单独后端的麻烦。那么剩下的地址问题是不是也可以这样？

*答案也是可以的*

既然它本质上是C代码，那很显然我可以用宏来替换地址，完全摈弃掉前面一堆麻烦的过程。不过官方并没有这么做。首先，CLoop并没有生产需求，只要满足正确性验证即可，所以为他单独添加特殊流程是不值得的。其次，CLoop和其他平台的生成过程保持一致可以更容易的定位错误，同一个平台上那些magic number很可能都是一样的。最后，因为offlineasm的一些设计限制，要做到这一点会比较ugly。这后面会讨论到。

#### 让offlineasm为我所用
官方不去做的事情，我们可以自己做。而且应该也不会比重写一套汇编后端更麻烦，所以值得一试。

回顾一下上面我们提到的整个过程：
1. 先生成需要的偏移地址列表。  
`asm == offsets_extractor ==> LLIntDesiredOffsets.h`
1. 把偏移编译进一个*迷你JSC*。  
`.CPP + .h == compiler ==> LLIntOffsetsExtractor`
1. 提取地址，结合原始伪汇编生成最终代码。  
`LLIntOffsetsExtractor + .asm == asm.rb ==> LLIntAssembly.h`

我们现在既然有了CLoop，而且我们希望最终汇编里带有`OBJECT_OFFSETOF`这样的宏，所以整个过程可以简化成：
1. 直接生成带有OBJECT_OFFSETOF的CLoop后端。  
`.asm == asm.rb ==> LLIntAssembly.h`

#### offlineasm 的实现
到现在，问题的关键剩下最关键的一步，如何让offlineasm跳过前面所有这些麻烦事，直接给我们生成一个我们需要的CLoop代码呢？这里我们可以先看一下它的大致实现方式。

1. 第一步：offlineasm拿伪汇编.asm以后需要先过parser，转成ast。这个ast包含了所有东西，包括我们需要的地址符号信息。

1. 第二步：它会拿这个ast去作为参考，来parse LLIntOffsetsExtractor。生成一个offsetsMap的东西。顾名思义，这就是偏移地址表——从符号到数值的一张表。

1. 第三步：利用offsetsMap再处理一遍ast，生成一个lowLevelAST。从名字上也很好理解，低级的AST，符号信息都被处理过，换成了真实的地址和偏移。

1. 第四步：把lowLevelAST扔给对应的后端去emit code。

可以看出来，从第三步开始，我们需要的symbol就被strip掉了。这也是为什么之前我说这么做一定程度上不符合offlineasm的设计原则——这需要修改ast的生成流程，把符号信息保留到最后扔给后端之前才行。

具体修改的细节不赘述，基本就和上面的思想一样，想办法保留足够的信息给CLoop后端，然后在生成代码的时候用`OBJECT_OFFSETOF(class, field)`来取代立即数。有兴趣可以参考我的项目来了解更多细节。

这里生成出来的LLIntAssembly.h一般来说只要你不去修改原始的伪汇编.asm的话，是不需要重新生成的，所以你也可以把它check in进去。

#### 其他处理
经过了上面的修改，这时候offlineasm生成出来的CLoop汇编代码（其实本质上是C代码）已经是非常的跨平台的了，基本可以无视绝大多数CPU差别，但是还有两点绕不过去：
1. `JSVALUE64`和`JSVALUE32_64`
2. `BIG_ENDIAN`和`LITTLE_ENDIAN`
这两个会影响到汇编代码具体实现的不同。前者是32位和64位平台的区别，后者是大小端的差别。我的办法比较简单粗暴，排列组合一共也就4种可能性，我们把针对这4种不同的配置生成的代码都放进最终的LLIntAssembly.h里面去，再用对应的宏保护起来即可。

到这儿，兴致冲冲的去编译一定会很沮丧，你会看到无数的编译错误。这些编译错误来自同一个问题：**私有成员**。  
没错，你可以用`OBJECT_OFFSETOF`来算偏移，但前提是这个class得答应是不是。所以一个个的去加friend吧，虽然有点累，大概二三十个类，不过是一次性的工作，还不算太麻烦。

总结
----
经过上面所有的努力，我们已经完全实现了一开始的所有希望：

- 我们有了一个不依赖特定汇编代码，纯C/C++实现的JS引擎，非常便于移植。
- 不依赖可执行内存，满足很多安全要求高的场景。
- 摒弃了那些复杂的中间步骤，现在可以抛弃掉它的cmake，很轻松的把这一堆C++代码集成进各种项目。
- 它的LLINT(低级解释器)性能比大多数其他所谓精简的JS引擎都要强大很多。
- 它不做JIT，不会动态编译出各种不同优化等级的二进制代码。而它的解释器只依赖一套相对精简的bytecode，对内存的压力小很多。
- 它仍然是一个功能完整，甚至非常先进的JS引擎。

## 参考项目

https://github.com/mbbill/PortableJavaScriptCore

这个项目有点年头了，而且不是非常完整。以后有空我会把它更新到最新的JSC代码。



