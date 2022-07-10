# 相关寄存器

GCN5 的[技术手册](https://developer.amd.com/wp-content/resources/Vega_7nm_Shader_ISA.pdf) (3.1小节)，
或者[在线文档](https://rocmdocs.amd.com/en/latest/GCN_ISA_Manuals/testdocbook.html#testdocbook) 列出了一些相关的寄存器/存储单元。

| Abbrev.      | Name                 | Size  (bits) | Description                       |
|--|--|--|--|
| PC           | Program Counter      | 48     | Points to the memory address of the next shader instruction to execute.|
| V0-V255      | VGPR                 | 32     | Vector general-purpose register.  |
| S0-S103      | SGPR                 | 32     | Vector general-purpose register.  |
| LDS          | Local Data Share     | 64kB   | Local data share is a scratch RAM with built-in arithmetic capabilities that allow data to be shared between threads in a workgroup.|
| EXEC         | Execute Mask         | 64     | A bit mask with one bit per thread, which is applied to vector instructions and controls that threads execute and that ignore the instruction.           |
| EXECZ        | EXEC is zero         | 1      | A single bit flag indicating that the EXEC mask is all zeros.       |
| VCC          | Vector Condition     | 64     | A bit mask with one bit per thread; it holds the result of a vector compare operation.|
| VCCZ         | VCC is zero          | 1      | A single bit-flag indicating that the VCC mask is all zeros.        |
| SCC          | Scalar Condition Code| 1      | Result from a scalar ALU   comparison instruction.           |
| FLAT\_SCRATC | Flat scratch address | 64     | The base address of scratch memory.                           |
| XNACK\_MASK  | Address translation failure.  | 64     | Bit mask of threads that have failed their address translation. |
| STATUS       | Status               | 32     | Read-only shader status bits.     |
| MODE         | Mode                 | 32     | Writable shader mode bits.        |
| M0           | Memory Reg           | 32     | A temporary register that has  various uses, including GPR indexing and bounds checking.     |
| TRAPSTS      | Trap Status          | 32     | Holds information about exceptions and pending traps.     |
| TBA          | Trap Base Address    | 64     | Holds the pointer to the current  trap handler program.             |
| TMA          | Trap Memory Address  | 64     | Temporary register for shader operations. For example, can hold  a pointer to memory used by the trap handler.|
| TTMP0-TTMP15 | Trap Temporary SGPRs | 32     | 16 SGPRs available only to the Trap Handler for temporary storage.                          |
| VMCNT        | Vector memory instruction count       | 6      | Counts the number of VMEM  instructions issued but not yet  completed.                        |
| EXPCNT       | Export Count         | 3      | Counts the number of Export and  GDS instructions issued but not yet completed. Also counts VMEM  writes that have not yet sent  their write-data to the TC.       |
| LGKMCNT      | LDS, GDS, Constant and Message count | 4      | Counts the number of LDS, GDS, constant-fetch (scalar memory read), and message instructions  issued but not yet completed.     |

## 相关寄存器的解释

### PC
PC 就是我们传统理解的 PC 寄存器，指向内存中下一条将要执行的指令。

### V0-V255 与 S0-S103
通用的向量寄存器和标量寄存器，[上一节](../overview/AMD-7nm-isa)有讲到。

### LDS
local data shared, 即共享内存的概念。[上一节](../overview/AMD-7nm-isa)有讲到。

### EXEC 与 EXECZ
即执行掩码，64 位，对应 wavefront 中的 64 个线程。
最常见的情况，在具有分支结构的核函数中，SIMT 模式的执行就是靠着这个掩码来运作的。
我们都知道，在 GPGPU（通用 GPU）中，一个 warp 或者 wavefront 中的若干线程是同时执行一条指令的。在具有分支的情况中，如 if/else 结构，所有的线程会一起执行 if 中对应的指令，然后所有的线程一起执行 else 中的指令。这也是为什么在 GPU 编程中要减少分支分歧（branch divergence）的原因。
由于所有的线程都同时执行同一条指令，可能从代码逻辑角度看一些线程是不需要执行这些指令的（如当前在执行 if 代码，对于 if 条件为 false 而应该进入 else 代码块中的线程而言，这些线程是不需要执行 if 中的代码的）。为了在 SIMT 模式下，也能执行具有分支结构的代码，EXCE 寄存器的作用就突显出来了，其可以记录哪些线程是“激活”状态的，对于“未激活”状态的线程，会有一套机制让其忽略对应的指令。

而 EXECZ 更为特殊，表示 EXEC 中所有的比特位是否都为 0。

### VCC 和 VCCZ
64 位，waavefront 中每个线程对应 1位，用于保存向量比较操作的结果。
VCCZ 表示 VCC 中是否所有的比特位都是 0。

### VMCNT
这个寄存器是和访存（显存）相关的。
在 AMD GPU 中，无论是 load/store 设备内存，还是 read/write 共享内存，其请求都是异步的（请求队列的形式）。VMCNT 积存器中，会记录这样的还没完成的访存请求有多少个。如果值为 0，则表示该 wavefront 中所有的访存操作都完成了。


注意：在后面的 [RDNA 架构](https://gpuopen.com/wp-content/uploads/2019/08/RDNA_Architecture_public.pdf)中，用了两个寄存器（VMCNT、VSCNT），来的分开统计 load 和 store 操作数量。

### LGKMCNT
和 VMCNT 类似，不过这里表示的是访问 LDS。

