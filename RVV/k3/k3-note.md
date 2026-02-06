# K3
## 3. Hardware Design
### 3.1.4 Vector
> At the microarchitectural level,
the X100 VPU employs a decoupled approach relative to the scalar
core. **After initial decoding and scalar operand renaming within the main pipeline, vector instructions are dispatched to an independent vector unit for subsequent splitting and execution.** Compared to the
tightly-interconnected modules of the scalar core, the vector unit is
functionally more separated and occupies a significant silicon area.
A decoupled vector unit not only provides enhanced flexibility and
configurability but also simplifies physical layout and floorplanning
for the core.

Scalar unit sends vector instructions to vector unit's buffer for futher execution.

> Vector instructions are first pushed in a dedicated vector instruction buffer, which serves to decouple them from the main pipeline.
Instructions are then issued from this buffer for further processing.
The vector unit supports the concurrent issue of up to two memory
instructions and two computational instructions. **Each issue port
can handle an instruction micro-op in 128-bit granularity per cycle,
which means an instruction with LMUL=1 is issued as two 128-bit
micro-ops.** 

A vector instruction is decomposed to several micro-ops. 

> The design choice of employing dual 128-bit data paths,
as opposed to a single 256-bit path, is primarily driven by the need
to balance hardware resource costs effectively. Given the wide va-
riety and high computational complexity of vector operations, a
finer-grained datapath width enables more efficient and balanced
allocation of resources among different instruction types. For in-
stance, complex permutation operations may have only one 128-bit
execution unit and when two such instructions are issued concur-
rently, only the older one can acquire the resource. This granularity
also facilitates parallelism between distinct functional resources, al-
lowing, for example, an addition instruction and a shift instruction
to be issued simultaneously and executed on their respective units.

But it does not explicitly state that OoO (out-of-order) scheduling is applied. The execution order is depends on the sending. So I don't think they adopt OoO scheduling for vector instructions.

> Memory access efficiency is paramount for vector applications.
The X100 decouples address generation and data handling for vec-
tor load/store instructions: the address phase is processed by the
main pipeline’s Load-Store Unit (LSU) for address calculation and
memory access initiation, while the data phase is handled by the
vector unit for operand splitting and vector register dependency
checking. Communication and data exchange between the two are
coordinated via instruction IDs. The LSU leverages its two existing
memory pipelines to issue and process two vector memory instruc-
tions in parallel. Correspondingly, the vector unit is equipped with
two matching splitter circuits to handle the data paths of these two
memory instructions. By decoupling address and data processing,
the main pipeline can forward instructions to the LSU as soon as
addresses are ready, without waiting for vector register dependen-
cies to be resolved. This allows the LSU to initiate memory accesses
promptly, thereby accelerating data retrieval and improving over-
all memory efficiency. Furthermore, for handling complex vector
memory patterns such as segment accesses, the LSU incorporates a
dedicated transpose buffer to accelerate data reorganization, signif-
icantly boosting execution efficiency.

