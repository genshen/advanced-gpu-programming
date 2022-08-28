## 目的
本栏目将专门对 GPU 上，特别是 AMD 卡、曙光 DCU 上的**架构、编程优化方法、常见问题**进行介绍。
大部分资料来源于平时的编程经验和对 [AMD ROCm 官方文档](https://rocmdocs.amd.com/en/latest/index.html) 内容的整理。

## 内容
本栏目属于稍微高阶的内容。
不包括基础的 Linux 操作、CUDA/HIP 基础编程、基础的优化手段（如 pinned-memory）。
相关内容基本都可以在互联网上（不过不太建议在中文互联网上找）找到，烂大街了。

## 预备知识：CUDA/HIP 编程基础
### NVidia 的CUDA 官方文档
- https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html.
- (中文版本) https://www.nvidia.cn/docs/IO/51635/NVIDIA_CUDA_Programming_Guide_1.1_chs.pdf

### HIP
- 阿贡实验室的 HIP 的编程手册: https://www.olcf.ornl.gov/calendar/intro-to-amd-gpu-programming-with-hip
- AMD 的相关编程资料： https://developer.amd.com/wp-content/resources/ROCm%20Learning%20Centre/chapter3/HIP-Coding-3.pdf.
- AMD 的 HIP 编程指南: https://rocmdocs.amd.com/en/latest/Programming_Guides/Programming-Guides.html
- 曙光 HIP 计算技术详解 [点击下载](/hpc/hip-programming.pdf)
- 曙光 DCU 开发者社区：https://developer.hpccube.com/tool/

## 目录
- [ ] AMD/nvidia GPU 架构
   - [x] [AMD GPU 架构概述](contents/overview/AMD-gpu-arch.md)
   - [x] [AMD Vega 7nm 指令集架构 (GCN5)](contents/overview/AMD-7nm-isa.md)
   - [x] [英伟达 GPU 架构概述](contents/overview/nvidia-gpu-arch.md)
- [ ] AMD GPU 汇编
   - [x] [Introduction](contents/assembly/intro.md)
   - [x] [相关寄存器](contents/assembly/registers.md)
   - [x] [汇编指令](contents/assembly/instructions.md)
- [ ] shared memory 数据共享
   - [ ] [bank 冲突](contents/shared-memory/bank-conflict.md)
- [ ] 线程通信
   - [ ] wavefront/warp 内的线程通信
   - [x] [在条件语句中谨慎使用 `__syncthreads`](https://blog.gensh.me/cuda-warp-conmunication)
- [ ] Debug & Profile
   - [x] [Introduction](contents/profile-debug/intro.md)
   - [x] [占用率与性能](contents/profile-debug/occupancy.md) 
   - [x] [roofline 性能模型](contents/profile-debug/roofline.md) 
- [ ] 访存优化相关
- [ ] 其他优化经验
   - [x] [优化经验总结](contents/extra-opt-tips)

## 推荐阅读
除了本仓库外，笔者在撰写该系列文章时，也参考或阅读过大量的相关教材和课程。这部分资料都属于高阶点的关于 GPU优化的资料：

供参考：
- UIUC ECE 408: https://ece.illinois.edu/academics/courses/ECE408-120211, https://wiki.illinois.edu/wiki/display/ECE408/Class+Schedule, [2019-fall](https://wiki.illinois.edu/wiki/display/ECE408/Materials+from+prior+semesters#)
- NVIDIA, CUDA C++ Best Practices Guide (CUDA BPG) https://docs.nvidia.com/cuda/cuda-c-best-practices-guide, [pdf](https://docs.nvidia.com/cuda/pdf/CUDA_C_Best_Practices_Guide.pdf)， 


## 协议
本着知识共享的精神，本文档采用电子形式发布。 
协议采用知识共享 [署名-非商业性使用-相同方式共享 BY-NC-SA 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/) 进行许可。

严禁任何商业行为使用！