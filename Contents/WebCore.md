# WebCore移植

相对于JavaScriptCore来说WebCore的代码量大了很多倍，但是整体编译过程的复杂度却稍好一些，因为没有JSC那么多涉及到汇编和各种伪代码转换的复杂步骤。编译过程中最复杂的一个步骤基本上就是IDL生成C++绑定的过程了。我们还像之前一样先了解一些背景知识。

## 背景知识

### Forwarding headers

如果还没有看过[WebKit的编译过程](Contents/Compilation.md)的话，我建议先读一下这个章节，尤其是里面关于forwarding header的部分。

因为WebCore依赖JavaScriptCore，而WebCore本身又会被上层组件使用，所以WebCore既依赖JSC提供的forwarding headers，又需要对WebKit和WebKitLegacy提供自己的forwarding header。

