# Paper Reading
This repository contains notes on research papers, technical documents, and learning materials, as well as a record of my personal research path. My general interests include Computer Systems, Machine Learning, Computer Architecture, and Networking.

# Research Path
- **26.1.9**: Surveyed approximate nearest neighbor (ANN) methods. Concluded that this direction may not be suitable; the main bottleneck appears to be memory bandwidth.

- **26.1.16**: Surveyed RISC-V extensions. Became particularly interested in compute-in-memory (CIM).

- **26.1.23**: Read the RISC-V Vector Extension (RVV) specification.

- **26.1.31**: Professor advised to look into Near-Memory Computation (NMC), which might help mitigate cache problems caused by long vectors. Observed that many optimizations happen at runtime rather than compile time.
Research task this week: Survey Xuantie C910 to determine: Whether vector instructions are executed in-order or out-of-order. At what granularity instructions are executed in-order vs. out-of-order (e.g., instruction, micro-op, vector element).

- **26.3.6**: TODO: Look into the micro-implementation of Ara.

- **26.3.13**: Reporst on the micro-implementation, TODOs: Look into the detail implmentation of Ara (Simulator), wrtie benchmarks of simulators.

- **26.3.20**: This week, write tests to determine whether vector chaining occurs in two specific scenarios: (1)Two  nonsequential access instructions chaining. (2) Mask-generation instructions chain with instructions that consume the generated masks.



# Reading List

## RISC-V Vector Extension
- [ ] Saturn Microarchitecture Manual
- [ ] RVV Manual Book
- [ ] Arrow: A RISC-V Vector Accelerator for Machine Learning Inference
- [ ] K3
- [ ] Xuantie-910

## HPC
- [ ] Communication-Avoiding General Matrix Multiplication within a single GPU

## ANN
- [x] HNSW paper

## ML and AI
- [ ] Attention is All You Need
