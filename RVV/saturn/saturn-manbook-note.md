# Reading Notes

## 2.1 Data-parallel Workloads:
1. HPC: scalability towards multi-core, multinode-systmes

2. General-purpose server & PC: Compiler and programmer-friendly ISAs are particularly important for these systems, as they must remain performant on the greatest diversity of workloads.

3. Digital signal processing (DSP): 数字信号处理技术 + 专用处理器，核心是快速、高效地处理音频、视频、传感器等数字信号。

4. Embedded devices: code and data size are more critical architectural considerations.

## 2.2 Data-parallel Implementation Archetypes
### 2.2.1 Long-vector Implementations
> Microarchitecturally, these implementations distribute the vector register file, functional units, and memory access ports across many vector lanes, with scalability towards many lanes being a key requirement. A single core of such a vector unit might have upwards of 32 parallel vector lanes, with a hardware vector length of over 16384 bits.

`lane`: a lane is a replicated, parallel slice of the vector datapath. It is not a single functional unit.

`High-radix memory interconnect`:

`high address generation throughput`:

`Indexed loads and stores`:

`distributed memory access ports`:

`register-register scatters and gathers`:

> 我的想法: memory interconnect可能成为Scalability的瓶颈

### 2.2.2 SIMT GP_GPUs
### 2.2.3 General-purpose ISAs with SIMD
`super scalar`: 在一个周期内发射多条指令给不同的FU

`out of order`: CPU 在执行阶段根据指令依赖和 FU 可用性，动态决定执行顺序，优先执行“已经准备好”的指令。 自动解决不可预测的依赖和延迟（如 cache miss、分支 mispredict）

`code scheduling`: 编译器根据已知指令依赖和目标 CPU 的 pipeline / FU 模型，在编译时重新排列指令顺序，减少 pipeline stall 和 FU 空闲。

`register renaming`: 寄存器重命名让 OoO CPU 可以乱序执行而不破坏程序顺序语义，解决了 WAR/WAW 假依赖问题。

### 2.2.4 VLIW ISAs with SIMD

### 2.2.5 Scalable Vector ISAs
> Saturn targets DSP and domain-specialized cores, and represents a class of designs we call *“short-vector”

### 2.4 Short-Vector Execution

> Saturn’s instruction scheduling mechanism differentiates it from the relevant comparable archetypes
for data-parallel microarchitectures. Fundamentally, Saturn relies on efficient dynamic scheduling
of short-chime short-vectors, without relying on costly register renaming. When LMUL is short (1
or 2), vector chimes may be only 2-4 cycles long, requiring higher throughput scheduling than a
long-chime machine.

`Chime`: 表示一条向量指令在流水线中完成一个元素处理所需的周期节奏。

`Dead time`: stall

> A short-vector machine should fully saturate both the arithmetic and memory pipelines with
such short vector lengths and chimes. Instruction throughput requirements are moderate, but can
still be fulfilled with a modest superscalar in-order scalar core. Notably, some degree of out-of-order
execution, beyond just chaining, is necessary to enable saturating both the memory and arithmetic
pipelines.

如果完全in-order -> 无法填满流水线

完全Out-of-Order -> renaming too expensive

### 2.4.1 Compared to Long-Vector Units
> Given the very long vector lengths, vector instructions are executed in a deeply temporal manner,
even across many parallel vector lanes. Thus, instruction throughput is less critical for maintain-
ing high utilization of functional units. Instead, long-vector microarchitectures typically derive
efficiency and high utilization by amortizing overheads over fewer long-chime inflight instructions.

长向量in order 就可以解决，而且可以分销overheads

# 3 System Organization
## 3.1 Organization
> The Vector Frontend (VFU) integrates into the pipeline of the host RISC-V core. In in-order cores, the stages of the VFU align with the pre-commit pipeline stages of the host core.

VFU的stages是与标量对齐的

Fetch → Decode → Rename → Issue → Execute → Writeback → Commit


> All vector instructions are passed into the vector frontend. The VFU performs early decode of
vector instructions and checks them for the possibility of generating a fault by performing address
translation of vector memory address operands. Instructions that do not fault are passed to the ther vector components once they have also passed the commit stage of the scalar core.

Post-commit execution: 软件看到 vector指令已经提交了，但实际上硬件正在进行运算。

## 3.2 Key Principle
> Saturn relies on post-commit execution of all vector instructions. That is, the VLSU and
VU only receive committed instructions from the VFU. Physical addresses are passed from the
VFU into the VLSU to ensure that the VLSU will never fault on memory accesses. This simplifies
the microarchitecture of the VLSU and VU, as all operations in these units are effectively non-
speculative. To enforce precise faults ahead of commit, Saturn’s VFU implements an aggressive,
but precise mechanism that validates instructions are free-of-fault and reports any faults precisely
if present.

> Saturn’s microarchitecture is designed around in-order execution with many latency-insensitive
interfaces. Notably, the load and store paths are designed as pipelines of blocks with latency-
insensitive decoupled interfaces. The load-response and store-data interfaces into the VU are also
latency-insensitive decoupled interfaces. Within the VU, each execution path (load, store, or arithmetic) executes instructions in-order. The in-order execution of the load/store paths aligns with
the in-order load-response and store-data ports.

> Saturn is organized as a decoupled access-execute (DAE) [15] architecture, where the VLSU
acts as the “access processor” and the VU acts as the “execute processor”. Shallow instruction
queues in the VU act as “decoupling” queues, enabling the VLSU’s load-path to run many in-
structions ahead of the VU. Similarly, the VLSU’s store path can run many cycles behind the VU
through the decoupling enabled by the VSIQ. This approach can tolerate high memory latencies
with minimal hardware cost

> Saturn still supports a limited, but sufficient capability for out-of-order execution. The load,
store, and execute paths in the VU execute independently, dynamically stalling for structural and data hazards without requiring full in-order execution. Allowing dynamic “slip” between these
paths naturally implies out-of-order execution. To track data hazards, all vector instructions in the
VU and VLSU are tagged with a “vector age tag (VAT)”. The VATs are eagerly allocated and
freed, and referenced in the machine wherever the relative age of two instructions is ambiguous.



> Saturn is designed around two key parameters. VLEN and DLEN. VLEN is the vector length of
each register file, as defined in the architecture specification. DLEN is a micro-architectural detail
that describes the datapath width for each of the SIMD-stype datapaths in Saturn. Specifically,
the load pipe, store pipe, and SIMD arithmetic pipes are all designed to fulfill DLEN bits per cycle,
regardless of element width. Future versions of Saturn may allow a narrower memory interface
width (MLEN) than DLEN.

> To decode vector instructions, the Saturn generator implements a decode-database-driven
methodology for vector decode. The Saturn generator tabularly describes a concise list of all
vector control signals for all vector instructions. Within the generator of the VU, control signals
are extracted from the pipeline stages using a generator-time query into the instruction listings.
The results from this query are used to construct a smaller decode table for only the relevant
signals, which is passed to a logic minimizer in Chisel that generates the actual decode circuit.
This approach reduces the prevalence of hand-designed decode circuits in the VU and provides a
centralized location in the generator where control signal settings can be referenced.

# 4 Frontend

## 4.1 Core Integration
> Rocket and Shuttle expose nearly identical interfaces to the vector unit. The scalar pipeline
dispatches vector instructions to the VFU pipeline at the X stage, immediately after scalar operands
are read from the scalar register files. The scalar pipeline also must provide these instructions with
up-to-date values of vtype, vl, and vstart. 

> The M stage exposes a port through which the VFU
can access the host core’s TLB and MMU. The WB stage contains interfaces for synchronizing
committing or faulting vector instructions with the scalar core’s instruction scheme and stalling
scalar instruction commit if necessary. The WB stage also exposes interfaces for modifying vstart
and vconfig as vector instructions commit.

在M stage 通过使用暴露的TLB和MMU进行检测。

### 4.1.1 Rocket
> At the write-back stage, vector instructions which cannot retire due to ongoing activity in
the IFC can kill all younger instructions in the earlier pipeline stages, and request the core to
re-fetch and replay the instruction at the next PC.

Ongoing activity in the IFC指的是IFC正在工作，并不是一定出现问题。

**Saturn的Frontend占用了标量CPU的资源，并不是一个独立的可以和标量并行的路径??**

> Vector instructions which cannot pass the PFC due to a TLB miss or a lack of capacity in the backend instruction queues are replayed through Rocket’s existing replay mechanisms.

TLB miss 和 backend instruction queue full需要CPU等待, 直接使用Rocket的 replay机制

> Rocket does not maintain a speculative copy of the vtype and vl CSRs at the decode stage,
so a data hazard can interlock the decode stage whenever a vector instruction proceeds a vset
instruction. As shown in Figure 12, a vset will always induce a 2-cycle bubble on a proceeding
vector instruction. The effect of this is most noticeable in short-chime mixed-precision vector code,
in which vset instructions are frequent.

这里
- vsetvli 需要写入 vtype/vl 
- vadd.vv 需要读取 vtype/vl 

### 4.1.2 Shuttle
> Only one of the execution pipes in Shuttle can dispatch into the VFU, but any of the pipes can
execute a vset operation. However, during steady-state operation, Shuttle can dynamically con-
struct instruction packets at the decode stage to maximize instruction throughput given structural
hazards by stalling partial instruction packets.

Shuttle processor的单一核心中有多条流水线，可以同时issue多个指令

`steady-state operation`: 处理器流水线已经完全填满，并且以最大效率持续运行的状态。

`instruction packets`: 指令包是一组可以并行执行的指令集合，在每个周期，超标量处理器会尝试将多条指令打包成一个或多个指令包，然后将这些指令包同时发射到不同的执行管道。

> Unlike Rocket, Shuttle implements a bypass network for vset instructions modifying vtype
or vl. Vector instructions following a vset instruction do not need to stall, as the vtype and
vl operands can be accessed through the bypass network. However, a vector instruction cannot
proceed in the same instruction packet as a vset; it must proceed on the next cycle instead. Figure
14 shows how Shuttle can dynamically stall a partial instruction packet with the vadd to issue it
with a younger vset on the next cycle. This example also depicts how stalling the vadd maintains
2 IPC through Shuttle, and 1 IPC into the vector unit.

`IPC`: Instruction per Cycle

## 4.2 Memory Translation and Faults
> Since vector instructions may be speculative ahead of the commit point, any vector instruction
flushed by the scalar core is also flushed from the VFU.

### 4.2.1 Pipelined Fault Checker(PFC)


## 4.3 Scalar Vector Memory Disambiguation
> Vector memory instructions architecturally appear to execute sequentially with the scalar loads and
stores generated by the same hart. Scalar stores cannot execute while there is a pending older vector
load or store to that same address. Scalar loads cannot execute while there is a pending older vector
load to that same address, as doing so could violate the same-address load-load ordering axiom in
RVWMO since the vector and scalar loads access different paths into the memory system. Section
5.3 discusses the mechanisms for vector-vector memory disambiguation.

Scalar 和 Vector即使有不同的通路，也不能设计为并行的

## 4.4 Interface to VU and VLSU
> The micro-op presented to the VU and VLSU contains the instruction bits, scalar operands, and
current vtype/vstart/vl settings for this instruction. For memory operations, this bundle also
provides the physical page index of the accessed page for this instruction, since the PFC and IFC
crack vector memory instructions into single-page accesses. For segmented instructions where a
segment crosses a page, segstart and segend bits are additionally included in the bundle, to
indicate which slice of a segment resides in the current page.

# 6 Vector Backend
> The vector datapath (VU) handles all vector instruction scheduling, computation, and register file
access. The VU is organized around independent “sequencing paths”, which issue instructions
independently to a per-sequencer vector execution unit (VXU).

`Sequencing paths`: 从指令分发到功能单元执行的完整处理流水线。包含完整的Issue Queue, Instruction Sequencer, Vector Execution Unit.

> The datapath is organized around a “vector element group” as the base unit of register access
and compute. An element group is a fixed-width DLEN-bits-wide contiguous slice of the architectural
register file space. DLEN guides the parameterization of most critical components in the backend.
The register file, load-path, store-path, and most functional units are designed to process DLEN bits
per cycle (or 1 element group/cycle), regardless of element width (ELEN).

"vector element group": 在register file中的DLEN宽的一组连续切片。每次操作都是以DLEN为单位处理的

## 6.1 Issue Queue
> Vector instructions entering the VU are immediately assigned a unique age tag and enqueued into
an in-order dispatch queue (VDQ). The age tag is used to disambiguate age when determining data
hazards in the out-of-order sequencing stage. This means the age tag is wide enough to assign a
unique tag to every inflight operation in the backend. Age tags are recovered when instructions
complete execution.

> The VDQ provides a low-cost decoupling queue between the VU and the VLSU. The dequeue
side of the VDQ is connected to the vector issue queues (VIQs). **Each VIQ issues instructions in-
order into its corresponding instruction sequencer. Execution across the VIQs may be out-of-order,
as the VIQs may naturally “slip” against each other.**

## 6.2 Operation Sequencers
> The instruction sequencers convert a vector instruction into a sequence of operations that execute down the functional unit datapaths, one operation per cycle. 

> The sequencers advertise the requested register file read and write addresses for the next operation, as well as the age tag for the currently
sequenced instruction. If there are no structural hazards from non-pipelined functional units or
register file ports and there are no data hazards against older vector instructions, a sequencer will
issue an operation and update its internal state. 

每一个sequencer会广播自身的信息给其他sequencer，这些sequencer根据信息来判断是否issue micro op.


> An instruction will depart a sequencer along with
the last operation it sequences, eliminating dead time between successive vector instructions.

### 6.2.1 Load/Store Sequencers

## 6.3 Hazards
> Due to the out-of-order execution across the different sequences, RAW, WAW and WAR hazards are all possible. 

> Furthermore, supporting vector chaining implies that these hazards should be
resolved at sub-vector-register granularity. Since Saturn is designed around DLEN-wide element groups as the base throughput of the machine, Saturn resolves data hazards at DLEN granularity.

即使两条vector指令存在data dependence(在instruction层面), 但是他们操作不同的DLEN, still OK

> The scheduling mechanism precisely tracks which element groups an instruction or operation has yet to read-or-write to interlock the sequencers.

> In Saturn, the “out-of-order instruction window” includes all instructions in the issue queues
(but not the VDQ), the instructions currently in progress within the sequencers, and any operations
which have not yet completed execution in the functional unit datapaths. Each instruction in this
window must advertise a precise set of element groups that it will read from or write to in future
cycles, along with its age tag.

## 6.4 Vector Register File
