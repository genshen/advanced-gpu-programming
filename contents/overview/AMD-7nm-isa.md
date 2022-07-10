# AMD Vega 7nm 指令集架构

AMD Vega 指令集架构（或者 GCN 5）的 GPU 大概长下图这样。下面挑重点讲其中的核心部件。

![gcn-vega-7nm-arch.svg](/hpc/advanced-gpu/gcn-vega-7nm-arch.svg){.align-center}

(原图: https://rocmdocs.amd.com/en/latest/GCN_ISA_Manuals/testdocbook.html#introduction)

**注: 如无特殊说明或特殊修饰(如"英伟达 GPU")，本页中的 GPU 均指 AMD Vega 架构的 GPU。**

## `rocminfo` 命令
用 `rocminfo` 命令，可以查看 GPU 的相关硬件信息，包括计算单元数量、wavefront 限制的数量、cache 等。
例如：
```log
  Name:                    gfx906
  Marketing Name:          Device 66a1
  Vendor Name:             AMD
  Feature:                 KERNEL_DISPATCH
  Profile:                 BASE_PROFILE
  Float Round Mode:        NEAR
  Max Queue Number:        128(0x80)
  Queue Min Size:          4096(0x1000)
  Queue Max Size:          131072(0x20000)
  Queue Type:              MULTI
  Node:                    4
  Device Type:             GPU
  Cache Info:
    L1:                      16(0x10) KB
  Chip ID:                 26273(0x66a1)
  Cacheline Size:          64(0x40)
  Max Clock Freq. (MHz):   1700
  BDFID:                   1024
  Internal Node ID:        4
  Compute Unit:            64
  SIMDs per CU:            4
  Shader Engines:          4
  Shader Arrs. per Eng.:   1
  WatchPts on Addr. Ranges:4
  Features:                KERNEL_DISPATCH
  Fast F16 Operation:      FALSE
  Wavefront Size:          64(0x40)
  Workgroup Max Size:      1024(0x400)
  Workgroup Max Size per Dimension:
    x                        1024(0x400)
    y                        1024(0x400)
    z                        1024(0x400)
  Max Waves Per CU:        40(0x28)
  Max Work-item Per CU:    2560(0xa00)
  Grid Max Size:           4294967295(0xffffffff)
  Grid Max Size per Dimension:
    x                        4294967295(0xffffffff)
    y                        4294967295(0xffffffff)
    z                        4294967295(0xffffffff)
  Max fbarriers/Workgrp:   32
  Pool Info:
    Pool 1
      Segment:                 GLOBAL; FLAGS: COARSE GRAINED
      Size:                    16760832(0xffc000) KB
      Allocatable:             TRUE
      Alloc Granule:           4KB
      Alloc Alignment:         4KB
      Acessible by all:        FALSE
    Pool 2
      Segment:                 GROUP
      Size:                    64(0x40) KB
      Allocatable:             FALSE
      Alloc Granule:           0KB
      Alloc Alignment:         0KB
      Acessible by all:        FALSE
  ISA Info:
    ISA 1
      Name:                    amdgcn-amd-amdhsa--gfx906
      Machine Models:          HSA_MACHINE_MODEL_LARGE
      Profiles:                HSA_PROFILE_BASE
      Default Rounding Mode:   NEAR
      Default Rounding Mode:   NEAR
      Fast f16:                TRUE
      Workgroup Max Size:      1024(0x400)
      Workgroup Max Size per Dimension:
        x                        1024(0x400)
        y                        1024(0x400)
        z                        1024(0x400)
      Grid Max Size:           4294967295(0xffffffff)
      Grid Max Size per Dimension:
        x                        4294967295(0xffffffff)
        y                        4294967295(0xffffffff)
        z                        4294967295(0xffffffff)
      FBarrier Max Size:       32
```

另一个命令`clinfo` （位于`opencl/bin/clinfo`下），也能显示一些类似的信息。

## 核心部件 CU
和英伟达 GPU 类似的，Vega 指令集架构 GPU 最核心的是 CU （英伟达 GPU 中是 SM，流处理器）。
例如 Nvidia Tesla V100 中就有 80 个 SMs（参考英伟达的[技术手册](https://images.nvidia.cn/content/volta-architecture/pdf/volta-architecture-whitepaper.pdf)），AMD Vega 架构的 GPU 内部也包含几十个 CUs，常见的是 64 或者 60。开发者可以通过 `rocminfo` 命令查看具体 CU 的数量。例如，上面的 `rocminfo` 显示 CU 的数量为 64 (Compute Unit 项)，每个 CU 内部还包含 4 个 SIMD。

在一个 CU 内部，包含 SIMD、向量/标量寄存器、LDS (local data share)，L1 cache等部件。下图描述了 CU 内部的详细情况。

![vega-gpu-compute-unit.svg](/hpc/advanced-gpu/vega-gpu-compute-unit.svg){.align-center}
图：一个 CU 的内部构成。（上：主要部件，下：CU 内部详细的部件构成）

下图是官方的原图，基本信息和上图差不多：
![amd_gcn_cu.jpeg](/hpc/advanced-gpu/amd_gcn_cu.jpeg)
(图片来源：https://gpuopen.com/learn/optimizing-gpu-occupancy-resource-usage-large-thread-groups/)

### SIMD
SIMD 即我们常说的单指令多数据流，这里指 GPU 中具有该功能的硬件。AMD Vega 指令集的GPU 的一个 CU 内部包含 4 个 SIMD。
每个 SIMD 内部包含 16 个乘法器 (图中红色竖条；这里的乘法器是比较广义的，它还可以做加法运算、fma 运算、发起访存等)，即一个 SIMD 在同时执行一条向量指令时，但是可以同时处理 16 条数据(如16个浮点数等运算)。除了乘法器外，一个 SIMD 内部还包括 64 KiB 的向量寄存器。

### SIMD 和 wavefront
熟悉 CUDA 的同学应该知道 warp 的概念，wavefront 就是 AMD GPU 上的 wrap。
不同的是，NVIDIA GPU 中，warp 包含 32 个线程，但是AMD GPU 中，wavefront 包含 64 个线程 （上面 `rocminfo` 命令输出的 `Wavefront Size` 项也给出了 64 这个数字）。
在 HIP 线程执行时，64 个线程（或者叫 ***work-items*** ）被组成一捆在 SIMD 上执行（这64个线程不可分散到多个 SIMD上）。可是问题是，SIMD 上面只有 16 个乘法器，如何执行这 64 个线程呢？
我们假设某条指令执行需要一个 cycle，那么 64 个线程需要 4 个 cycle 才能都执行完这条指令。实际上，也是这样的，一次让 16 个线程执行，然后下 16 个线程，分4批才能执行完。

详细的解释见 [这里](https://rocmdocs.amd.com/en/latest/Programming_Guides/Opencl-optimization.html?highlight=wavefront#hiding-alu-and-memory-latency)。

> For most AMD GPUs, each compute unit can execute 16 VLIW instructions on each cycle. Each wavefront consists of 64 work-items; each compute unit executes a quarter-wavefront on each cycle, and the entire wavefront is executed in four consecutive cycles.
>
> 注解：每个 cycle 执行 1/4 个 wavefront；整个 wavefront 需要 4 个 cycle；这4个 cycle 不可被打断 (consecutive)。

### VGPR (向量通用寄存器)
下图给出了更细节的关于 VGPRs 和 SGPRs 的视图。

![](https://rocmdocs.amd.com/en/latest/_images/fig_2_1_vega.png)

上面说了，64 个线程组成一个 wavefront 在一个SIMD 上面执行，
所以，这 64 个线程都会有属于自己的向量通用寄存器。上图就很直观地将 VGPRs 分成 64 组，每组 64 个，每个寄存器大小是 32 bit 的。整个 SIMD 上，总向量通用寄存器的大小为 $256 \times 64 \times 32/8=64$ KiB。
每个线程最多可以用 256 个向量通用寄存器。对于每个线程而言，在其的视角看，可以用的寄存器有 v0, v1, v2, $\cdots$, v255。

向量通用寄存器是用于执行向量指令用的。例如一条向量指令`v_mul_f32_e32 v3, v0, v10`，是将 v10 寄存器的数和 v0 寄存器的数相加，结果存 v3 中。这时，在逻辑上，wavefront 中的 64 个线程，每个线程都有 v0,v3,v10 这些寄存器。但是，在各个线程上，相同编号的向量通用寄存器中的数可以是完全不一样的。

### SGRPs (标量通用寄存器)
所谓标量通用寄存器，一方面是和标量指令相关的。
当然，另一方面，在向量指令中，也可以用标量寄存器（如指令 `v_mul_f32_e32 v5, s0, v10`）。
`s0` 就是标量通用寄存器。
和向量通用寄存器不同，对于 wavefront 内的各个线程而言，其读到的标量通用寄存器(如 `s0` )的值都是一样的。

另外，标量通用寄存器是以 wavefront 为单位分配的（每个wavefront中都可以用 s0 标量寄存器）。其值对于 wavefront 内的 64 个线程是一样的。

上图也给出了 CU 中，SGPRs 的数量。其共 800\*4 个（每个 SIMD 对应 800 个为一组的标量寄存器文件），每个大小 32 位，共计 12.5 KiB (因为除以1000的关系，上面图是算成12.8 KiB的，不过关系不大)。
需要特别说明的是，800\*4 个 SGPRs 不在 SIMD 中，而是在 CU 中的。

### 通用寄存器申请
- SGPRs: 一个wavefront 可以申请 16 到 102 个 SGPRs, 且必须以 16 GPRs 为单位申请，编号 s0-s101。
- VGPRs 必须以 4 个 dwords (一个 dword 即 32位，一个寄存器大小) 为单位申请。

详细的解释见[AMD 的技术手册](
https://rocmdocs.amd.com/en/latest/GCN_ISA_Manuals/testdocbook.html#sgpr-allocation-and-storage)。

### L1 cache
L1 cache 是位于 CU 中的。不同的是，L2 cache 是位于 CU 外面的，所有 CU 共享 L2 cache。而 L1 cache 是 CU “私有”的。
AMD Vega GPU 中，L1 cache 大小为 16 KiB，cache-line 大小为 64 字节。

进一步说明，英伟达 GPU 的 cache 也是类似的的，L1 cache 是 SM 内部的，L2 cache 这是全片共享的。另外，AMD 和 NVIDIA GPU 不想大多数 CPU 那样拥有 3层 cache，它只有两层。

### LDS
LDS 可以理解为可编程控制的 L1 cache。AMD Vega GPU 中，LDS 大小为 64 KiB。
此外，LDS 访存延迟和 L1 cache 也大致相当。

附: 这个链接给出了计算机中不同部件访问的延迟，可供参考对比。
https://colin-scott.github.io/personal_website/research/interactive_latency.html

## GDS
GDS 是全片共享的一块存储区域，大小 64 KiB。但是因为[某些原因](https://github.com/RadeonOpenCompute/ROCm/issues/1296)，暂时没法用。
