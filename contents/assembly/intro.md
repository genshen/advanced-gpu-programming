# AMD GPU 汇编入门

## 为什么需要汇编？
GPU 计算利用率不总是100%，主要是由于：
- 采用高级语言写的程序，没有利用到硬件的一些特性/接口。
- 编译器无法生成优化的ISA代码（为了正确性考虑，或者编译器不够好）。

所以我们需要了解汇编，一方面上层的 API 可能未提供一些接口，可能需要我们用汇编进行接口的的调用；另一方面可以了解高级语言写的代码，在编译编译后，是否符合我们所想，如果不是，可能还需要调整代码以迎合编译器或者直接用汇编写。

## 预备知识
你可能需要具备一定的英文文档阅读能力。本文档可能不会面面俱到，但一些额外/扩展的内容均可以在官方的英文文档中找到。
另外，需要知道如何在 C 语言里面写内嵌汇编。

## 相关文档
AMD GPU 的编译器是基于 llvm 开发的，LLVM 文档中有 [关于 AMD GPU 后端部分](https://www.llvm.org/docs/AMDGPUUsage.html)的内容，可以点击前面的链接进行参考。
在编译时，用的是 hipcc 编译器，但是 hipcc 编译器只是一个脚本，里面会用到 llvm 和 clang 来编译 kernel 的代码。

下面对着上面链接内容中的目录部分，挑一些常用的概念或知识点及文档。当然，更多的内容可以前往 llvm 的文档阅读。
### Processors
对应文档在[这里^1](https://www.llvm.org/docs/AMDGPUUsage.html#processors)。其中，GCN 5 （VEGA）所对应的 processor 有 gfx900、gfx902、gfx904、gfx906、gfx909。例如，曙光在昆山超算中心的 DCU 的 processor 就为 gfx906。
为什么会提到 processor呢？因为不同的 processor 对应的汇编指令可能会有所区别，因此针对特定版本（或一系列特定版本）的 AMD GPU 硬件写汇编时，需要参考不同的汇编指令的手册。

例如，对于 processor 为 gfx906的情况，可参考的汇编指令的文档为[Syntax of gfx906 Instructions](https://www.llvm.org/docs/AMDGPU/AMDGPUAsmGFX906.html)。对于其他的 processor 的情况，也可以对照这个[表中列出的链接^1](https://www.llvm.org/docs/AMDGPUUsage.html#instructions)，找对应的汇编指令的文档。

### Address Spaces
LLVM 的关于 AMD GPU 后端的文档中，有一个章节介绍了 [addresss space ^1](https://www.llvm.org/docs/AMDGPUUsage.html#address-spaces)。


### memory model
内存模型的文档见[这里^1](https://www.llvm.org/docs/AMDGPUUsage.html#memory-model)。对于 GFX9 系列，这个链接中也有详细的情况进行说明，具体的内容和前文提到的 GCN 架构类似。如果想了解其他微指令架构的 AMD GPU，其对应的内存模型也在前面链接指向的文档中有说明。

这个文档对于访存优化是很有用的。这里如果继续写，也仅仅是 llvm 文档的搬运翻译工作，意义不大，建议大家直接看相应的内存模型的文档。

### 指令语法
[这个文档](https://www.llvm.org/docs/AMDGPUInstructionSyntax.html) 以及[这个文档](https://www.llvm.org/docs/AMDGPUOperandSyntax.html) 给出了 AMD GPU 汇编指令的语法规则。
这部分核心的后续会有专门的篇幅进行介绍。


注：
- ^1 该标号标示的页面为同一页面，只是对应不同的段落。
