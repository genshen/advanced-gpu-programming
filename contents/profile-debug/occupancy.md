# 占用率与性能

GPU 为了实现计算和访存的重叠，一个 CU（英伟达的 GPU 叫 SM）上会同时调度进来多个 block 执行。
这样，每个 SIMD 上，就会有多个 wavefront 在同时（并发地）执行。
虽然，我们希望 SIMD 上运行的 wavefront 数量越多越好。
但是，由于寄存器数量是有限的，共享内存数量是有限的，这就会限制调度进来执行的 block 数量。
我们需要有一些定量的技术来计算当前的资源和能运行的 wavefront/block 数量之间的关系，
以评估是否合理地利用了现有的寄存器和共享内存资源。

注：以下如果没有特别说明，相关数据均指 GCN 架构。其他架构上，可能会有所区别。
## 理论的情况
由于硬件的限制，在GCN 架构中，每个 SIMD 上最多可以同时运行 10 个 wavefront（WAVE64）。
一个 CU 上，最多可以调度进来 16 个 block。即，GCN 架构中，每个CU最多可以同时并发地运行 40 个wavefront。
一个 block 中，最多 16 个 wavefront（64*16=1024）。

占用率：CU中（或者 GPU 中）实际的活跃 wavefront 数量 / 理论的 wavefront 数量。占用率是资源利用的水平反应。

由于实际的活跃 wavefront 数量可以有不同的计算方式，所以占用率可以有以下的表示方式：
- **实测占用率（achieved occupancy）**：实测占用率是在 GPU 硬件上实测的占用率，是和时间有关联的（不同时刻，活跃的 wavefront 数量可能不一样，导致实测的占用率数值可能不一样）。
  例如可以是关于时间 t 的一个函数。有时候，也可以是一个时间段内的平均占用率。
  achieved occupancy is measured on the hardware and is a time-dependent metric (as the number of active WF is not constant) （原文见 1.）
- **理论占用率（theoretical occupancy）**：通过 kernel 所请求的资源计算出的占用率。
  （Theoretical occupancy is a calculated metric, derived from the resources requested by the kernel）

相关信息来源见参考链接[^1][^2][^3][^4]。

## 寄存器资源限制
我们知道，GCN 架构下，每个 SIMD 对应 64 KiB 的通用向量通用寄存器（256*64个 32 位的寄存器）；每个 CU 共 12.5 KiB 的标量通用寄存器（800*4 个寄存器）。  
wavefron 内的线程最多可以用 256 个向量通用寄存器，且向量寄存器申请必须以 4 个 dwords为单位申请；
每个 wavefront 最多可用 102 个标量通用寄存器，且标量寄存器申请数量必须是 16 的倍数。
当然，这是上限，会影响占用率的。

接下来，我们以 GCN 架构为例，分析寄存器资源的占用对占用率的影响。
先看向量寄存器，设线程使用的向量寄存器数量为 $V_N$，那么很容易得到一个 SIMD 上最多活跃的 wavefront 数量和占用率的表 (注：这里占用率计算先不考虑其他因素的共同影响，如共享内存)。

表：GCN架构下，向量寄存器用量与占用率的关系
<!-- | $V_N$ | <= 25 | 26~28 | 29~32 | 33~36 | 37~42 | 43~51 | 52~64 | 65~85 | 86~128 | 129~256 | -->
| $V_N$ | <= 24 | 28 | 32 | 36 | 40 | 44,48 | 52~64 | 68~84 | 88~128 | 132~256 |
|-- | --|-- | -- | -- | -- | -- | -- | -- | -- | -- |
| active wavefronts per SIMD ($\min\left(\frac{256}{V_N},10\right)$) |10 | 9 | 8 | 7 | 6 | 5 | 4 | 3 | 2 | 1|
| occupancy | 100% | 90% | 80% | 70% | 60% | 50% | 40% | 30% | 20% | 10% |

标量寄存器，也是类似的情况。
不过，还有一些细节，需要进一步明确:
- wavefront 虽然最多可以用 112 个标量通用寄存器，但是后 6 个 SGPRs 会有特殊用途。如 VCC 寄存器用的就是用的 SGPR 106 和 SGPR 107（其实VCC就是这两个寄存器的别名），当计算过程中获
取到了中断(trap)请求，VCC 保存中断地址。（[来源](https://rocmdocs.amd.com/en/latest/GCN_ISA_Manuals/GCN-ISA-Manuals.html#the-gpr-counting) 及[来源2](https://rocmdocs.amd.com/en/latest/GCN_ISA_Manuals/testdocbook.html#sgpr-allocation-and-storage)， 或者 GCN vega 10 nm 指令集手册 3.6.2）
- 此外，第 112 个寄存器也有特殊用途，因此有 4 个寄存器也没法用上（不是每个 wavefront 都用）。 因此，实际可以用的标量通用寄存器最多只有 102 个（我们编号为0～101）。
- 当中断时，对于每个 wavefront，还需要额外的 16 个 SGPRS 用来保存 PC 和中断 handler（trap handler）等信息 （[来源](https://rocmdocs.amd.com/en/latest/GCN_ISA_Manuals/testdocbook.html#trap-and-exception-registers), 或者 GCN vega 10 nm 指令集手册的 3.10 小节）。 因此，实际可用于存储普通数据的通用寄存器最多只有 102-16=86 个。
- SGRPs 以 16 个 dwards 为单位申请，这里面就包括了特殊用途的各类寄存器。
> The last 6 of these are used for special purposes (such as VCC), and these cannot be used as general purpose scalar registers by user code.
> The 112 case is special; here, 4 additional registers cannot be used, leaving 102 for GPR purposes.
> 来源见参考链接 1。

我们也可以给出一个标量寄存器和占用率的关系:

| 申请标量寄存器数量($N_S$) | 32 | 48 | 64 | 80 | 96 |
| -- | -- | -- | -- | -- | -- |
| 用于存数据的寄存器数量($N_S - 16 - 6$) | 10 | 26 | 42 | 58 | 74 |
| active wavefronts per SIMD (800/$N_S$) | 10  | 10 | 10 | 10 | 8 | 

这里可以看到，硬件提供的标量寄存器还是很充足的，用量不超过 80，都能达到100%的占用率。即使用到 96，也能有很好的占用率。

## 共享内存的限制
限制条件：
- 一个 CU 共 64 KiB 的 LDS。
- 共享内存按 block 进行分配，且 CU 最多只能同时运行 16 个 block（不过有特例，见“核函数配置的限制”部分）（来源见[^1]）。
- 每个 CU 最多同时运行 40 个 wavefront。

我们在稠密矩阵乘（GEMM）、SpGEMM 或者 SpMV 等问题中，都会用 LDS 作为缓存，先把数据加载到 LDS 中，然后使用时从 LDS 中访问。LDS 的用量会影响调度进来执行的 block 数量，进而影响活跃的线程数量。 
考虑一个 kernel，其 BlockDim 设置为 256，且每个 block 需要用 32 KiB 的 LDS。
那么仅考虑 LDS 的限制时，GPU 可以调度两个 block 进来执行（64KiB/32KiB = 2），共计 $2 \times \frac{256}{64} = 8$个 wavefront，则占用率为：8/40= 0.2，可以说非常低了。

## 核函数配置的限制
这里，我们重点考虑 BlockDim 的设置对占用率的影响（为了方便，这里假设线程按照一维的形式进行组织的）。
这里还需要特别说明下，如果 BlockDim = 64，CU 最多只能同时运行 40 个 block，而非限制在 16 个（见参考链接[^1]）。

我们考虑 BlockDim = 128 的核函数配置（不考虑寄存器、LDS资源的限制），最多可以调度进来执行的 block 数量是 $\max\left(16, 40/(128/64) \right) = 16$。
活跃的最多 wavefront 数量是 32，而非打满到 40，占用率为 32/40 = 0.8。
这是因为设置 BlockDim = 128 太小了，如果设置为 256 或者 512，就能把占用率提升到 1.0。

我们知道，为了不出现硬件浪费，BlockDim 肯定需要设置为 64 的倍数的（NV 平台上应为 32 的倍数）。但是是否所有的 64 的倍数（例如 384）都合适呢？
答案是否定的，例如前面讨论过设置为 128 就不是那么合适。再如 BlockDim=384，即一个 block 内有 6 个 wavefront。那么最多只能调度进来 6 个 block 执行，即活跃的 wavefront 数量为 36。
未了使得最大的活跃 wavefront 数量为 40（仅考虑核函数袋配置时），一个必要条件是 block 内的 wavefront 数量必须是 40 的约数，即只能是 1,2,4,5,8。其中 2 的情况是需要排除掉的（前面我们讨论过 BlockDim = 128 的情况）。最终只剩下 1,4,5,8，对应的 BlockDim 的值为 64, 256,320,512。

另外，需要特别说明的是：
1. 为了进行访存-计算的重叠，需要有足够多的活跃的 wavefront。这是我们讨论占用率的主要出发点。
2. **占用率不一定非要搞到 100%**。如果你的算法访存密度不是那么大，可能 0.6 的占用率也能充分掩盖访存延迟。如果你的算法访存密度很大，可能100% 的占用率也不一定能充分掩盖访存延迟（可能最后所有的 wavefront 都在等待访存返回），这时候就应该关注访存性能的优化了，尽量利用访存合并等技术提高访存带宽。 （别把占用率搞得低的离谱就行，例如参考链接里面有个例子，LDS 用量太大，导致占用率只有 0.05）
3. **提高占用率不是唯一目标**。例如，减小 LDS 用量，虽然可以提高占用率，但是有可能会降低算法的性能（如因为处理的批次会变多）。
4. **最大化占用率，并不意味着最好的性能（性能还和访存模式、负载均衡设计、算法等有关）**

## 其他
- 一个有趣的讨论
    > If you schedule 40 workgroups of one wavefront each to each CU, then that will completely fill all the resources assuming you are not limited by other factors (such as local memory usage, regsters, etc.)  Another way to fill the CU is to schedule 10 workgroups of four wavefronts per CU.
    https://community.amd.com/t5/archives-discussions/optimizing-code-for-gcn/m-p/225385
- 另一个 OpenCl 优化教程
https://rocmdocs.amd.com/en/latest/Programming_Guides/Opencl-optimization.html#hiding-memory-latency-with-alu-operations 

## 参考链接
[^1]: https://radeon-compute-profiler-rcp.readthedocs.io/en/latest/occupancy.html#kernel-occupancy-for-amd-radeon-hd-7000-series-or-newer-based-on-graphics-core-next-architecture
[^2]: http://developer.amd.com/wordpress/media/2013/06/2620_final.pdf
[^3]: https://www.olcf.ornl.gov/wp-content/uploads/2019/10/ORNL_Application_Readiness_Workshop-AMD_GPU_Basics.pdf
[^4]: https://community.amd.com/t5/opencl/wavefront-and-kernel-occupancy/m-p/133044
[^5]: [Nvidia-WarpsAndOccupancy](https://developer.download.nvidia.com/CUDA/training/cuda_webinars_WarpsAndOccupancy.pdf)
