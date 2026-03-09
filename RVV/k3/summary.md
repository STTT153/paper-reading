# Survey on Microarchitecture (Saturn, XuanTie910 and K3)

## 1. Vector Backend

### 1.1 Saturn:

Vector Frontend (VFU) is integrated into the scalar core. It will send vector instructions together with vector CSRs (vtype, vl, lmul, vstart) information to Vector Dispatch Queue (VDQ) in Vector Datapath (VU).

VDQ will further dispatch instructions to different Vector Issue Queues (VIQs), based on the type of instructions. Each VIQ issues instructions in-order into its corresponding instruction sequencer. Execution across the VIQs may be out-of-order, as the VIQs may naturally "slip" against each other. (I don't think OoO on micro-ops granularity)

The sequencers decompose the instructions to a sequence of micro-op that execute down the functional unit datapaths, one operation per cycle. Sequencers advertise current instructions information(which DLEN bits of element group is executing and age tag) to other sequencers. Each sequencer keeps track of instructions in the issue queues, the instructions currently in progress within the sequencers, and any operations which have not yet completed execution in the functional unit datapaths. Based on these information, sequencer decide whether to send micro-op to execute or stall the datapath.

### 1.2 XuanTie-910

Vector registers are renamed in IR stage. Vector instructions are OoO scheduled in IS stage, together with scalar instructions (Age-Vector based scheduling algorithm is used).

Vector pipeline could consist of multiple slices (recommend 2 slices). A slice (lane) consists of a multi-port 64 bit vector physical register file and two OoO vector floating-point and integer execution pipelines. Each slice issues two vector instructions, they could be executed OoO.

### 1.3 K3 X100 Core

After decoding and renaming, vector instructions are sent to Vector Instruction Buffer for further execution.

VLEN = 256, ELEN = 64

The vector unit supports the concurrent issue of up to two memory instructions and two computational instructions. Each issue port can handle an instruction micro-op in 128-bit granularity per cycle, which means an instruction with LMUL=1 is issued as two 128-bit micro-ops.

No detail information about pipeline stage.

### 1.4 K3 A100 Core

Vector instructions are first pushed into a dedicated vector instruction buffer after initial decoding and scalar operand renaming in the main pipeline, then dispatched to an independent vector unit for subsequent splitting and execution.

VLEN = 1024, ELEN = 64

The vector unit classifies operators into five categories according to "instruction types + datapath characteristics": INT, FP, CRYPTO, FDV, and IME-1.0. Each category has dedicated execution resources with different issue widths and result stages.

The vector unit supports dual-issue execution with two parallel vector pipelines. Instructions are split into micro-ops based on 128-bit or 256-bit granularity depending on the operation type and data width. High-frequency operators (add/sub, compare, shift, logic, narrow-width multiply-accumulate) are mapped to wider execution resources capable of dual-issue and produce results as early as VE3 stage.

For complex operations like permutation and division, the design deliberately constrains issue bandwidth and datapath width to improve area and energy efficiency.

The execution follows a decoupled microarchitecture where address generation for vector memory instructions is handled by the main pipeline's Load-Store Unit (LSU), while data handling is processed by the vector unit. Communication between the two is coordinated via instruction IDs.