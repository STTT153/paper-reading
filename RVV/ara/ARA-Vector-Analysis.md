# ARA Vector Physical Register File and OoO Execution Analysis

## 1. Vector Physical Register File (VRF) Architecture

### Data Layout

**Lane-based Partitioning:**
- ARA can be configured with 2-16 parallel vector lanes
- Each lane contains a slice of the Vector Register File (VRF)
- VRF is partitioned across lanes for bandwidth scalability
- Each lane has its own independent VRF, SIMD FPU, and SIMD ALU/Multiplier

**Bank Organization per Lane:**
- Each lane contains **8 single-ported (1RW) memory banks**
- Each bank is **64-bit wide**, matching the datapath width of each lane
- Banks are implemented using single-ported SRAM memory
- Example configuration: VLEN=4096 bits, 4 lanes → 4KiB per lane, 512B per bank

**Addressing Scheme:**
- Address bus to each lane is 12-bits wide (for 4KiB VRF per lane)
- **Least significant 3 bits**: select among 8 banks within a lane
- **Remaining 9 bits**: address bytes within the bank
  - 6 bits: address 1-of-64 64-bit locations within bank
  - 3 bits: address individual byte within 64-bit word

**Element Organization:**
- Elements are mapped to consecutive lanes in the VRF
- Within each lane, elements are arranged to maintain mapping consistency across different SEW (element width) values
- This creates **Lane Organization** vs **Natural Packing** distinction:
  - **Lane Organization**: How elements are stored in VRF (optimized for parallel processing)
  - **Natural Packing**: How elements are stored in memory (sequential byte order)

### Port Configuration

**Read/Write Ports per Lane:**
- **8 single-ported banks** per lane
- Each bank has **one address port for both reads and writes**
- **Worst-case access pattern**: 5 banks accessed simultaneously
  - 4 reads (3 source registers + mask register)
  - 1 write (destination register)
- **Maximum concurrent access**: Up to 8 banks per lane simultaneously when multiple instructions execute in different functional units

**Operand Delivery Interconnect:**
- **9 Operand Queues** connecting VRF to functional units:
  - 3 dedicated to VFPU/VMUL
  - 2 dedicated to VALU  
  - 2 dedicated to Mask Unit
  - 1 dedicated to VLSU
  - 1 dedicated to SLDU
- **6 Write Back Queues**: one from each functional unit

### Distribution Strategy

**Across Lanes:**
- Vector registers are distributed across all lanes
- Each lane processes its portion of vector elements in parallel
- Slide unit provides inter-lane communication for cross-lane operations

**Within Lanes (Bank Distribution):**
- Vector register number determines starting address in VRF
- Starting address = register_number × (VRF_bytes_per_register / number_of_lanes)
- All vector registers start in bank 0, which can cause bank conflicts
- **Bank conflict resolution**: Weighted round-robin priority arbiter per bank
  - Priority order: Mask register (v0) > operand A > operand B > operand C > destination register

## 2. Out-of-Order (OoO) Execution Architecture

### Instruction Flow

**1. Instruction Reception and Dispatch:**
- **Ara Dispatcher**: Receives vector instructions from CVA6 scalar core
- Performs legality checking, CSR management, and load/store reshuffling
- Issues decoded vector requests to backend via `ara_req_o` interface

**2. Global Sequencing:**
- **Ara Sequencer**: Central control module managing instruction dispatching
- Tracks running instructions and their mapping to Processing Elements (PEs/lane)
- Maintains global hazard table for dependency management
- Calculates start/end lanes for operand access based on vstart, vl, and vsew

**3. Lane-level Execution:**
- **Lane Sequencer**: Each lane has its own sequencer
- Receives requests from main Ara sequencer
- Interfaces with Operand Requester and functional units
- Manages lane-specific execution flow

### Hazard Management and Dependency Tracking

**Global Hazard Table:**
- Tracks RAW, WAR, WAW hazards across all instructions
- Computes hazards against read_list_q and write_list_q
- Broadcasts hazard information to operand requesters via `global_hazard_table_o`

**Operand Queues for Hazard Absorption:**
- Operand queues between VRF and functional units absorb delays due to banking conflicts
- Write back queues at functional unit outputs handle write-back conflicts
- All queues are 64-bits wide, matching datapath width

### Execution Units and Parallelism

**Per-Lane Functional Units:**
- **Vector Floating-point Unit (VFPU)**: Handles floating-point operations
- **Vector Multiplier (VMUL)**: Integer multiplication operations  
- **Vector Integer ALU (VALU)**: Integer arithmetic and logical operations

**Cross-Lane Units:**
- **Vector Load & Store Unit (VLSU)**: Handles memory operations
- **Slide Unit (SLDU)**: Handles vector slide and cross-lane operations

**Parallel Execution Capability:**
- Multiple instructions can execute simultaneously in different functional units
- Instructions can access up to 8 banks per lane concurrently
- Cross-lane operations handled by dedicated SLDU unit

### Special Handling Mechanisms

**Reshuffle Logic:**
- Required for instructions that change EEW (Effective Element Width)
- Prevents violation of tail-undisturbed policy
- Two-step process:
  1. De-shuffle destination register to natural ordering based on existing EEW
  2. Shuffle result to lane organization based on new EEW

**Shuffle/De-shuffle Logic:**
- Located between memory subsystem and VRF
- Converts between Natural Packing (memory) and Lane Organization (VRF)
- Uses SEW-tagged data to determine correct shuffle pattern

## Source Documentation

### Primary Sources:
- **`/Users/stevenyin/dev/ara/docs/source/modules/lane/vrf.md`**: Comprehensive VRF architecture documentation
- **`/Users/stevenyin/dev/ara/docs/source/modules/ara_dispatcher.md`**: Instruction dispatcher functionality
- **`/Users/stevenyin/dev/ara/docs/source/modules/ara_sequencer.md`**: Global sequencer and hazard management
- **`/Users/stevenyin/dev/ara/hardware/src/lane/vector_regfile.sv`**: VRF Verilog implementation
- **`/Users/stevenyin/dev/ara/FUNCTIONALITIES.md`**: Supported RVV 1.0 instruction set

### Key Technical Specifications:
- **RVV Version**: 1.0 compliant
- **ELEN**: 64 bits maximum element size
- **Lane Count**: Configurable 2-16 lanes
- **VRF Banks**: 8 single-ported banks per lane
- **Bank Width**: 64 bits per bank
- **Functional Units**: VFPU, VMUL, VALU per lane + VLSU/SLDU cross-lane

This analysis provides a comprehensive overview of ARA's vector physical register file architecture and out-of-order execution mechanisms based on the available documentation and source code.