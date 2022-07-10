# 英伟达 GPU 概述
从目前（2022年）的市场来看，英伟达仍然是 GPU 市场的老大，无论是桌面级 GPU，还是服务器级的 GPU，无论是在图形图像、AI、科学计算，甚至挖矿，英伟达 GPU 都是首选。
因此十分有必要对其架构进行一些了解。

如无特殊修饰或者说明，本页下文中出现的 “GPU” 均指英伟达的 GPU。

## NVIDIA GPU 架构演进
这篇文章很全面地介绍了英伟达 GPU 的演进历史，人家讲得比我清晰，特此搬运过来。[GPU架构演进十年，从费米到安培](https://mp.weixin.qq.com/s/Jq9CFbCgi5cxVlE6Xvu9uw)

更多的关于各代 GPU 的信息，也可以从[维基百科](https://en.wikipedia.org/wiki/List_of_Nvidia_graphics_processing_units)查到。
总结来说，从2010年开始，NVIDIA GPU 先后经历过 Tesla（2010年前）、Fermi（2010）、Kepler（2012）、Maxwell（2014）、Pascal（2016）、Volta（2017）、Turing（2018）、Ampere（2020）、Hopper（2022）等架构。上面的链接对每一代架构的主要特性都有介绍，在此不再赘述。

虽然架构一直在演进，但是NV GPU 内部的组件基本还是比较稳定。下面会简单对其进行介绍。
下面将针对 GPU 内部的核心单元进行介绍。

## SM
SM，即流多处理器，可以类比于 AMD GPU 的 CU。不同架构、型号的 GPU 中 SM 的数量差别很大，例如[V100 技术白皮书](https://images.nvidia.cn/content/volta-architecture/pdf/volta-architecture-whitepaper.pdf) 里面表1列出了几代 GPU 的 SM 的数量。

SM 中，一般还会分几个块，叫 processing block。每一块会包含若干个 FP32 Cores（计算 单精度浮点的）, 若干个 FP64 Cores（计算双精度浮点的），SFU (Special Function Unit，例如计算 sin、cos 等), LD/ST（用于内存数据 load/store 的）,  warp scheduler,  dispatch units，和寄存器文件，此外还可能包括 L0 指令 cache（不同架构会有所区别，针对特定架构进行优化时查下白皮书就行），tensor core（用于矩阵乘法计算的）。

### SFU
引用自：https://mp.weixin.qq.com/s/Jq9CFbCgi5cxVlE6Xvu9uw
> 每个SFU每个时钟周期只能一个线程执行一条指令。而一个Warp(32线程)就需要执行8个时钟周期。SFU的流水线是从Dispatch Unit解耦的，所以当SFU被占用时，Dispatch Unit会去使用其他的执行单元。

### RT core
从 Turing 架构开始，引入了 RT core，叫射线追踪 core，应该和图像学那边相关性大。

### tensor core
从 Volta 架构开始，在 SM 中引入了 tensor core，以更好地支撑深度学习应用。
网上也有一堆关于 tensor core 的资料。

### cache
Volta 架构中，还引入了 L0 指令 cache，位于 processing block 中。
L1 数据 cache 和 L1 指令 cache 都是 SM 中的多个 processing block 共享的。
L2 数据 cache 是整个 GPU 共享的。

### warp scheduler 和 dispatch units
warp 调度器会根据一定的策略，来在多个 warp 中，选择其中的一个 warp 以进行指令发射。
dispatch unit 是将 warp 中的指令发射给特定的执行单元（如 FP32，FP62，INT32，LD/ST等）进行执行。

> 每个warp scheduler每cycle可以选中一个warp，每个 dispatch unit 每cycle可以issue一个指令。

更多关于调度和指令发射的内容可以关注[这个文章](https://zhuanlan.zhihu.com/p/166180054) 以及[这篇](https://www.zhihu.com/column/p/391238629)，这个应该算十分高阶的内容了（看评论）。

### shared memory
共享内存位于 SM 内部，和 L1 数据 cache 一样，由 SM 中的多个 processing block 共享。在一些架构上，L1 数据 cache 和 shared memory 是合并在一块的，例如 Volta 中，L1 数据cache 和共享内存一共 128 KB，可以利用 CUDA API 进行配置。

## 参考文献
- Tesla 架构资料：https://www.cs.cmu.edu/afs/cs/academic/class/15869-f11/www/readings/lindholm08_tesla.pdf
- Maxwell 架构（2014）： https://www.microway.com/download/whitepaper/NVIDIA_Maxwell_GM204_Architecture_Whitepaper.pdf
- pascal 架构（2016）： https://images.nvidia.com/content/pdf/tesla/whitepaper/pascal-architecture-whitepaper.pdf
- volta 架构（2017）： https://images.nvidia.com/content/volta-architecture/pdf/volta-architecture-whitepaper.pdf
- Turing 架构（2018）： https://images.nvidia.com/aem-dam/en-zz/Solutions/design-visualization/technologies/turing-architecture/NVIDIA-Turing-Architecture-Whitepaper.pdf
- ampere 架构（2020）： https://images.nvidia.com/aem-dam/en-zz/Solutions/geforce/ampere/pdf/NVIDIA-ampere-GA102-GPU-Architecture-Whitepaper-V1.pdf
- Hopper 架构（2022）： https://nvdam.widen.net/s/9bz6dw7dqr/gtc22-whitepaper-hopper
- https://www.nvidia.com/content/PDF/product-specifications/GeForce_GTX_680_Whitepaper_FINAL.pdf
- 另一篇博客讲 NV GPU 架构演进的。https://jcf94.com/2020/05/24/2020-05-24-nvidia-arch/ 或者 https://github.com/jcf94/jcf94.github.io/tree/master/2020/05/24/2020-05-24-nvidia-arch。
