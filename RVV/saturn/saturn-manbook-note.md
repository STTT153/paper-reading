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

## 2.3 The RISC-V Vector ISA
