# roofline 性能模型

## 计算密度的概念
对于冯诺依曼结构的计算机，其在计算过程中，往往需要将数据从内存中加载到计算单元中，进行计算后，将结果写回内存。
由于，现代计算硬件的计算速度和访存速度存在跨越数个数量级的差异（如现代GPU 双精度浮点计算速度往往都是几十 TFLOPS，而访存也才几百 GB/s或者才刚过 TB/s）。
例如，英伟达在 2022年 GTC上发布最新一代 H100 GPU，其相关性能参数如下：
|参数 | 值 |
| -- | -- |
| FP64 | 30 TFLOPS  |
|GPU memory  | 80GB |
| GPU memory bandwidth | 3TB/s |

参数链接：https://resources.nvidia.com/en-us-tensor-core/nvidia-h100-datasheet。
由于访存速度和计算速度的差异，我们往往需要从访存和计算两个层面开对算法进行性能分析。

我们在考虑 GPU/DCU 上设计的算法的性能时，我们往往有一系列评价指标，例如：
**计算量**：指在给定算例下，整个算法的浮点计算次数。一般我们用 **FLOPs** 了表示（其中，s小些，表示复数）。例如 GEMM，对于两个都是 N*N 的矩阵而言，其计算量为 $2N^3$（加法和乘法算作两次浮点计算次数；注：不过也有时候，将乘法和加法算作一次，因为硬件可以用一条 FMA 指令来完成乘加操作）。
**访存量**：访存量一般指算法计算时所需访问存储单元的字节大小，一般用 **Bytes** 表示。包括 load 的数据量和 store 的数据量。例如，GEMM，对于两个都是 N*N 的矩阵，其访存量是 $3N \times N$。
**时间复杂度、空间复杂度**：这个就是我们学习传统算法提到的时间复杂度、空间复杂度，这里不做赘述。

对于给定的算法，我们可以用其计算量除以访存量，得到算法在单位访存量下所需的计算量。我们称之为该算法的计算密度（Arithmetic Intensity），也有人叫它计算访存比。

计算密度 I = FLOPs/Bytes= 计算量/访存量。

算法的计算密度，可以反映该算法是计算密集型的还是访存密集型的。计算密度是算法的固有属性，不随硬件环境、编译器环境等改变而改变（但可能随算法中的某些参数改变而改变）。

对于计算密度较低的算法，其表明该算法访存量很大，而计算量相对较小。这时，算法的执行时间往往受限于硬件的访存带宽。我一般们称该情况下的算法为访存密集型算法。典型的如 CSR 格式的稀疏矩阵向量乘法（CsrMV）。
相反地，如果计算密度较高，其表明该算法计算量很大，而访存量相对较小。这时算法的执行时间往往受限于硬件的理论计算能力。我们一般称该情况下的算法为计算密集型的算法。典型的如稠密计算乘法 GEMM。

## roofline 性能模型
对于给定的硬件，我们将计算密度作为横坐标，理论上算法可以达到的计算速度（FLOPs/s）为纵坐标，画一个图，可以得到 roofline 性能模型。该模型是用于评估程序或算法在硬件上能达到的性能上界的模型。
考虑到理论访存带宽和理论峰值计算能力的限制，可有：

计算速度(FLOPs/s) = min(理论峰值性能, 理论访存带宽*计算密度)

以 NV H100 为例，其 roofline 性能模型如下：
![](./images/roofline.png)


## Cache 下的 roofline 性能模型
todo：


## 参考资料
- Charlene Yang, Lawrence Berkeley National Laboratory, Jun 16 2019 Frankfurt, [Introduction to the Roofline Model](https://www.nersc.gov/assets/Uploads/Tutorial-ISC2019-Intro-v2.pdf). 
- https://developer.download.nvidia.cn/video/gputechconf/gtc/2019/presentation/s9624-performance-analysis-of-gpu-accelerated-applications-using-the-roofline-model.pdf

### roofline 的原始论文及相关报告
- roofline 的原始论文: WILLIAMS S, WATERMAN A, PATTERSON D. Roofline: An Insightful Visual Performance Model for Multicore Architectures[J/OL]. Commun. ACM, 2009, 52(4): 65-76. DOI: https://10.1145/1498765.1498785.
- WILLIAMS S, PATTERSON D, OLIKER L, et.al. The roofline model: A pedagogical tool for program analysis and optimization[C/OL]//2008 IEEE Hot Chips 20 Symposium (HCS). 2008: 1-71. DOI: https://10.1109/HOTCHIPS.2008.7476531.
- Samuel Williams, A Vision for Integrating Performance Counters into the Roofline model, https://crd.lbl.gov/assets/pubs_presos/pmu08-roofline-talk.pdf
- Samuel Williams, Roofline Performance Modeling for HPC and Deep Learning Applications, https://crd.lbl.gov/assets/Uploads/S21565-Roofline-1-Intro.pdf
