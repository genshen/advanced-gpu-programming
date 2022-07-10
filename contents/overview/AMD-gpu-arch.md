# AMD GPU 架构概述
## AMD 显卡概述
A 卡相对于 N 卡起步和生态建设要晚。
在 AMD 官网上可以查询到相关显卡的情况，如 AMD Radeon RX。
更全的信息可以在[维基百科](https://en.wikipedia.org/wiki/List_of_AMD_graphics_processing_units)找到，按照桌面、手机、工作站、服务器等分类。

从上面链接的维基百科，其中表 “Features Overview” 给出了其 GPU 发展的一个概览。
我们重点从 2007 年的 RM600 开始关注，其指令集经过了 TeraScale instruction set => GCN instruction set => RDNA instruction set 的过度。
在相同的指令集下，又会有不同的微架构，例如 GCN 指令集经历了 5 代微架构的优化。每一代 GPU 对相关编程语言/库（如经常听说到的 Vulkan、OpenGL、Direct3D、OpenCL）的支持也都不尽相同，详细情况见上述维基百科网页。

而曙光的 DCU（如昆山超算、先导1号），相关技术和 AMD 有很大的渊源，所使用的指令集为GCN，微架构为 GCN 5rd gen （第五代 GCN 微架构，简称 GCN5）(更准确点说是 GCN5.1)。

本教程讲主要以 GCN5.1 ("Vega" 7nm Instruction Set Architecture) 为例，进行讲解。后面有机会会讲到 RDNA 系列指令集/架构与 GCN 的区别。

## 微架构
和 N 卡类似，其每一代的性能提升，和其微架构的关系很大。微架构也会对编程产生一些影响，一般会使得编程更方便、程序性能更好。

Graphics Core Next（GCN） 指令集的显卡最早在 2012 年推出，先后有 5 代 GCN系列的微架构，分别命名为 GCN 1st gen、GCN 2nd gen、GCN 3rd gen、GCN 4th gen、GCN 5th gen。
相关的技术手册可以在以下链接中找到（我是文档的搬运工）：
- [GCN1](http://developer.amd.com/wordpress/media/2012/12/AMD_Southern_Islands_Instruction_Set_Architecture.pdf)
- [GCN2](http://developer.amd.com/wordpress/media/2013/07/AMD_Sea_Islands_Instruction_Set_Architecture.pdf)
- [GCN3, GCN4](http://developer.amd.com/wordpress/media/2013/12/AMD_GCN3_Instruction_Set_Architecture_rev1.1.pdf)
- [GCN5](https://developer.amd.com/wp-content/resources/Vega_Shader_ISA_28July2017.pdf)
- [GCN5.1 或  Vega 7nm Instruction Set Architecture](https://developer.amd.com/wp-content/resources/Vega_7nm_Shader_ISA.pdf)

不同的微架构，其在硬件(LDS、GDS、SIMD、CU scheduler)、标量操作、向量操作、访存及其对应的指令(如新指令)和指令格式（新指令格式或者指令多了个参数）等方面有所区别。

例如，GCN5 相对于之前的 GCN4 微架构，硬件上支持了更高的时钟频率、支持 HBM2（high bandwidth memory 2）、更大的地址空间（来源 https://en.wikipedia.org/wiki/Graphics_Core_Next#fifth）。
指令上增加了 `V_DOT2_F32_F16`等向量指令、Scalar memory 原子操作指令等，
更多具体可在上述技术上手册中的前面部分（第一章或者前言）找到。

注：GCN5 和 GCN5.1 区别不大，主要体现在工艺上（后者是 7nm 工艺），他们都又叫 Vega 指令集架构，后者往往限定是 “Vega 7nm 指令集架构”。
> Vega 10 and Vega 12 use the 14 nm FinFET process, developed by **Samsung** Electronics and licensed to GlobalFoundries. Vega 20 uses the 7 nm FinFET process developed by **TSMC**.
> Ref from: https://en.wikipedia.org/wiki/Graphics_Core_Next#fifth

所以，为了更详细的了解 GPU 的指令集和内部架构，多去啃 AMD 的技术手册和 rocm 官网的相关技术资料是很有价值的。本栏目的大多数内容（特别是本节关于的架构部分的）也都是来自于技术手册（当然也有一些是作者本人的编程经验，是属于自己的思想），通过自己的理解加以整理得到的。
若和原始技术文档有出入，还望理解并帮助完善改正，在此提前感谢。
