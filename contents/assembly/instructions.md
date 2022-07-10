# 汇编指令
具体的每一个汇编指令的相关语法，参数可以在 llvm 编译器的网站上找到。例如，这个[页面](https://www.llvm.org/docs/AMDGPU/AMDGPUAsmGFX9.html) 列出了 GFX9（对应GCN 5微架构，见[这里](https://www.llvm.org/docs/AMDGPUUsage.html#instructions)） 的相关指令。

## 汇编指令格式

具体文档见 https://www.llvm.org/docs/AMDGPUInstructionSyntax.html#amdgpu-syn-instructions。

## 汇编指令的分类
可以将 AMD GPU 上执行的指令分为以下三类：（向量/标量）访存指令、标量指令、向量指令和控制流操作指令。
在与硬件的对应上：
- 标量 ALU 负责 kernel 的控制流相关（包括 if/else，分支，循环等）All kernel control flow is handled using scalar ALU instructions. This includes if/else, branches and looping ([原文](https://rocmdocs.amd.com/en/latest/GCN_ISA_Manuals/testdocbook.html#program-organization)).
- 标量ALU与内存指令作用于整个wavefront的线程。Scalar ALU (SALU) and memory instructions work on an entire wavefront and operate on up to two SGPRs, as well as literal constants.
- 向量内存和向量 ALU指令作用于 wavefront 内的所有线程。Vector memory and ALU instructions operate on all work-items in the wavefront at one time.
- 向量 ALU 通过“执行掩码”来决定某个线程的 if 分支是否真正执行，依照 SIMT执行。即掩码为1，对应线程激活，该线程执行指令；掩码为0，将对应指令替换为空指令（v_nop）。

### 向量指令
在指令格式上，一般向量计算指令多采用 `v` 开头，例如 `v_mul_f32` 表示两个单精度浮点数相乘，向量指令依据操作数，还可以分为 VOP1、VOP2、VOP3、VOP3P、VOPC 等（具体见 llvm 文档）。
例如，指令 `v_cos_f32 vdst, src` 就属于 VOP1，用于计算单精度浮点数的余弦，其有一个原操作数和一个目的操作数。而 `v_min_u32 vdst, src0, vsrc1` 属于 VOP2，用于比较 src0 和 vsrc1 的大小，结果放 vdst，其中原操作数有两个，目的操作数一个。


### 标量指令
标量指令一般用 `s` 开头。如指令 `s_cmp_le_i32 ssrc0, ssrc1` 是比较两个标量寄存器中的值（按 i32 类型进行比较）。结果存 SCC 寄存器（Scalar Condition Code，用途：Result from a scalar ALU comparison instruction.）。

### 访存指令
访存指令又分为好多种，最常用的是访问设备内存（向量访问）、访问 LDS（即共享内存），标量访问设备内存。
其中，访问 LDS 的指令一般以 `ds` 开头，例如，`ds_write_b64` 和 `ds_read_b64`。
访问设备内存（向量访问）的最常用的指令是 `global_load_*`、`global_store_*`、`global_atomic_*` 等，分别表示 load 数据、store 数据、原子操作。这里，向量化访问设备内存是指，wavefront 内的所有线程都会访问设备内存，每个线程的访存地址可以不同。例如，指令 `global_load_dwordx4 v[8:11], v[8:9], off` 表示从向量寄存器 `v[8:9]` 所表示的地址（地址为64位）处 load 4 个 words 的数据（即 4\*4=16 字节，AMDGPU 一个word 是 4 字节，等于一个向量寄存器的大小），访存的结果保存在寄存器 `v[8:11]` 中（注意：这里，wavefront 每个线程都有自己的 v8、v9、v10、v11 寄存器）。

对于标量访问设备内存的指令有 `s_load_*`、`s_atomic_*` 等。

### 控制指令
如分支指令、跳转指令。具体见指令文档中的 [SOPP 这个分类](https://www.llvm.org/docs/AMDGPU/AMDGPUAsmGFX9.html#sopp)。
例如 `s_barrier` 指令，正如指令名称所示，会让 block 中的所有线程在此处同步。`s_waitcnt` 用于等待访存请求完成。`s_branch`指令表示无条件跳转。

## 访存指令详细说明
前述小节仅概要地说了下访存指令，这里再次对其进行更细致的描述。

### Dependency counters 的补充知识
VMCNT, LGKMCNT, EXPCNT 三个寄存器，一般被称为Dependency counters ([来源](https://developer.amd.com/wordpress/media/2013/06/2620_final.pdf))。

**VMCNT 寄存器**  
VMCNT 寄存器，含义是“Counts the number of VMEM instructions issued but not yet completed”，即 ~~（一个wavefront）~~ 还尚未完成向量访存操作（如果是 GCN架构的话，包括 load和store操作）。其大小是 6 bits，这里的 VM 就是 vector memory 的意思。

具体细节如下：GCN架构下，当一条向量访存指令（如向量化load/store，例如	`global_load_dword`指令、`global_store_dword`指令等）被发射时，计数器 VMCNT 会加1。计数器 VMCNT 在如下情况下会减1 : 1）在 load 数据时，数据写到 VGPRs 中时，或者2) 在 store 数据时，数据写到 L2 cache 中时 。（[原文](https://rocmdocs.amd.com/en/latest/GCN_ISA_Manuals/testdocbook.html?highlight=wavefront%20schedule#data-dependency-resolution)）

另外，GPU 内部的机制保证了，load/store 操作是按照指令发射的顺序返回的。
> Ordering: Memory reads and writes return in the order they were issued, including mixing reads and writes.

举例说明下，对于如下代码：
```c
global_load_dword v9, v[9:10], off 
global_load_dword v3, v[11:12], off
s_waitcnt vmcnt(1)
v_add_u32_e32 v10, v9, v5
s_waitcnt vmcnt(0) 
```
假设这段代码执行前，寄存器 VMCNT 的计数为0。第一条指令 `global_load_dword` 发射成功后，VMCNT 加1；然后，发射第二天 memory load 指令，VMCNT 变为 2。
接着，第三条指令，是进行等待对应访存完成。具体为，等待 VMCNT 变为 1 或者低于1。由于 load/store 访存会按照发射的先后顺序返回。所以 `s_waitcnt vmcnt(1)` 就是等待第一条 `global_load_dword` 指令对应的访存完成，把数据存到 v9 寄存器中。因为，下一条指令 `v_add_u32_e32` 就用用到 v9 寄存器了，必须确保数据已经被 load 进来了。执行完 `v_add_u32_e32` 后，继续用 `s_waitcnt` 等待第二个访存指令对应的访存完成。

**RDNA 架构上 VMCNT 指令的区别**  
注意，以上关于 VMCNT 的材料，只针对 GCN架构的，后面的 RDNA 架构有所区别，它的 store 请求数量用另外一个寄存器 VSCNT 记录，详见前面章节或者[这里](https://gpuopen.com/wp-content/uploads/2019/08/RDNA_Architecture_public.pdf) 或者 [RDNA 白皮书](https://developer.amd.com/wp-content/resources/RDNA_Shader_ISA.pdf)）。
在 RDNA 中，VMCNT 只和向量 load 操作有关，即 “VMCNT only counts vector memory loads, image sample instructions, and vector memory atomics that return data.”, 而 "VSCTN" 寄存器是 “the counts of outstanding vector store events: vector memory stores and atomics that DO NOT return data" （这里直接用英文了，保留原汁原味。~~也可能是不想翻译了~~）。

**LGKM_CNT 寄存器**
LGKM_CNT 寄存器，保存的是 LDS, GDS, (K)constant, (M)essage 等低延迟的指令访存请求数量。概念和前面的 VM_CNT 类似，具体什么时候值会加一，什么时候会减一，可以参考具体的[文档](https://rocmdocs.amd.com/en/latest/GCN_ISA_Manuals/testdocbook.html?highlight=wavefront%20schedule#data-dependency-resolution)。差不多就是，LDS/GDS read/write 指令发射时，会加1；当 LDS/GDS read 指令或者有返回的原子操作(这里的原子操作是针对 LDS/GDS 的) 把数据放到 VGPRs 中，或者 LDS/GDS write 指令完成把数据写到对应的存储位置后。

另外，需要注意的是，这里不同类型的指令，是不保证返回的先后顺序。但是相同类型的指令(除了标量内存 load指令)是保证的，返回顺序和发射顺序保持一致。
> Instructions of different types are returned out-of-order.
Instructions of the same type are returned in the order they were issued, except scalar-memory-reads, which can return out-of-order (in which case only S_WAITCNT 0 is the only legitimate value).

下面的两个汇编代码示例，演示了 LDS 操作和 LGKM_CNT 的关系（示例1），以及标量 load 指令和 LGKM_CNT 寄存器的关系（示例2）。
```
ds_read_b64 v[17:18], v14
v_add_u32_e32 v14, 16, v14
s_or_b64 s[26:27], vcc, s[26:27
global_load_dwordx2 v[15:16], v[15:16], off
s_waitcnt vmcnt(0) lgkmcnt(0)
```
```
s_load_dword s0, s[4:5], 0xc
s_load_dword s3, s[6:7], 0x0
s_waitcnt lgkmcnt(0)
```
扩展阅读：https://bartwronski.com/2014/03/27/gcn-two-ways-of-latency-hiding-and-wave-occupancy/ 
备份：[gcn_–_two_ways_of_latency_hiding_and_wave_occupancy__bart_wronski.webarchive](/hpc/advanced-gpu/gcn_–_two_ways_of_latency_hiding_and_wave_occupancy__bart_wronski.webarchive)

**s_waitcnt**  
从上面的例子，我们也可以窥探一二。s_waitcnt 指令可以等待相关计数的寄存器值（包括LGKM_CNT、VMCNT 等）变为特定的值或者比这个值更小。例如，`s_waitcnt lgkmcnt(2)`，只要 LGKM_CNT 寄存器的值变得 <= 2 就可以往下继续执行后面的指令。

延伸链接：
- https://releases.llvm.org/13.0.0/docs/AMDGPU/gfx9_waitcnt.html
- https://bartwronski.com/2014/03/27/gcn-two-ways-of-latency-hiding-and-wave-occupancy/

**对优化的启发**
我们换个说法，`global_load_dword` 等访存指令（包括访问设备内存和共享内存、GDS等），它是一个异步的操作。我们可以通过 `s_waitcnt` 指令等待访存完成。
那么，对于程序性能优化而言，我们希望尽量在发起访存和等待访存结束之间，尽量多地插入一些计算代码。或者，类似于双缓冲的模式，在 wavefront 内部实现访存和计算重叠。
另外，由于访问设备内存和 LDS 采用不同的寄存器进行计数，我们也可以在访问设备内存周期内（执行 load 指令和 s_waitcnt 指令之间），重叠 LDS 的访问操作，实现访问设备内存和 LDS 访存的重叠。

**相关问题讨论：**  
1. 问题: CU 内或者 SIMD 内，多个 wavefront 的访存请求是如何计数的？会用多个 VMCNT 寄存器吗（即每个 wavefront 都会对应一个VMCNT寄存器吗，用于记录该 wavefront 的未完成的访存操作数量）？


**TODO:**

