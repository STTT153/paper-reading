# Report
## Configuration
- VLEN=1024
- DLEN=128
- LLC hit latency=8
- VLIQCapacity=4
- ROBCapacity=16

## Implement Segment Buffer
Current instruction latency result compare to RTL (unit-stride)

Unit-strided load `vle32.v` with differnet VL

| VL | Saturn‑baseline | MCA | abs‑difference | rel‑difference |
|----|-----------------|-----|----------------|----------------|
| 8  | 11              | 12  | 1              | 9.09%          |
| 16 | 13              | 14  | 1              | 7.69%          |
| 31 | 17              | 18  | 1              | 5.88%          |

Segment unit-strided load `vlsegNFe32` with differnet NF

| NF | Saturn‑baseline | MCA | abs‑difference | rel‑difference |
|----|-----------------|-----|----------------|----------------|
| 2  | 27              | 29  | 2              | 7.41%          |
| 3  | 28              | 30  | 2              | 7.14%          |
| 4  | 29              | 31  | 2              | 6.90%          |
| 5  | 46              | 48  | 2              | 4.35%          |
| 6  | 47              | 49  | 2              | 4.26%          |
| 7  | 48              | 50  | 2              | 4.17%          |
| 8  | 49              | 51  | 2              | 4.08%          |


## Modify Vector Frontend Stage
Vector instructions fall into one of the following categories
```
Single-page vector instructions include arithmetic and vector memory instructions for which the extent of the access can be bound to one physical page, at most. This includes unit-strided vector loads and stores that do not cross pages, as well as physically addressed accesses that access a large contiguous physical region. These are the most common vector instructions and need to proceed at high throughput through the VFU.

Multi-page vector instructions are memory instructions for which the extent of the instruction’s memory access can be easily determined, but the range crosses pages. These are somewhat common vector instructions, and must not incur a substantial penalty.

Iterative vector instructions include masked, indexed, or strided memory instructions that might access arbitrarily many pages. These instructions would fundamentally be performance-bound by the single-ported TLB, so the VFU can process these instructions iteratively.

```
Single-page vector instructions include **unit-strided** vector loads and stores that do not cross pages.

Iterative vector instructions include masked, **indexed, or strided** memory instructions that might access arbitrarily many pages.

For single-page vector instructions, they go through a pipelined-fault-checker (PFC). For Iterative instructions, they are further issued to a iterative-fault-checker (IFC)


```
Iterative instructions cannot be conservatively bound by the PFC. Instead, these instructions perform a no-op through the PFC and are issued to the IFC. Unlike the PFC, which operates page-by-page, the IFC executes element-by-element, requesting index and mask values from the VU for indexed and masked vector operations. The IFC generates a unique address for each element in the vector access, checks the TLB, and dispatches the element operation for that instruction to the VU and VLSU only if no fault is found. Upon a fault, the precise element index of the access that generates the fault is known, and all accesses preceding the faulting element would have been dispatched to the VU and VLSU.
```

The IFC generates a unique address for each element in the vector access, checks the TLB, and **dispatches the element operation for that instruction** to the VU and VLSU only if no fault is found.

Before implementation VL=16
| Instruction    | Saturn‑baseline | MCA  | abs‑difference | rel‑difference |
|----------------|-----------------|------|----------------|----------------|
| vlse32.v       | 43              | 37   | 6              | 13.95%         |
| vloxei.v       | 58              | 37   | 21             | 36.21%         |
| vluxei.v       | 58              | 37   | 21             | 36.21%         |
| vlsseg2e32.v   | 54              | BUG  | SKIP           | SKIP           |
| vloxseg2e32.v  | 71              | BUG  | SKIP           | SKIP           |
| vluxseg2e32.v  | 71              | BUG  | SKIP           | SKIP           |

After implementation VL=16
| Instruction    | Saturn‑baseline | MCA | abs‑difference | rel‑difference |
|----------------|-----------------|-----|----------------|----------------|
| vlse32.v       | 43              | 42  | 1              | 2.33%          |
| vloxei.v       | 58              | 58  | 0              | 0.00%          |
| vluxei.v       | 58              | 58  | 0              | 0.00%          |
| vlsseg2e32.v   | 54              | 47  | 7              | 12.96%         |
| vloxseg2e32.v  | 71              | 63  | 8              | 11.27%         |
| vluxseg2e32.v  | 71              | 63  | 8              | 11.27%         |

vlse.v with different VL
| VL | Saturn‑baseline | MCA | abs‑difference | rel‑difference |
|----|-----------------|-----|----------------|----------------|
| 8  | 23              | 24  | 1              | 4.35%          |
| 16 | 43              | 42  | 1              | 2.33%          |
| 31 | 88              | 77  | 11             | 12.50%         |

vloxei.v with different VL
| VL | Saturn‑baseline | MCA | abs‑difference | rel‑difference |
|----|-----------------|-----|----------------|----------------|
| 8  | 30              | 32  | 2              | 6.67%          |
| 16 | 58              | 58  | 0              | 0.00%          |
| 31 | 113             | 109 | 4              | 3.54%          |

vlsseg2e32.v with different VL
| VL | Saturn‑baseline | MCA | abs‑difference | rel‑difference |
|----|-----------------|-----|----------------|----------------|
| 8  | 32              | 29  | 3              | 9.38%          |
| 16 | 54              | 47  | 7              | 12.96%         |
| 31 | 102             | 81  | 21             | 20.59%         |

vlssegNFe32.v with different NF
| NF | Saturn‑baseline | MCA | abs‑difference | rel‑difference |
|----|-----------------|-----|----------------|----------------|
| 2  | 54              | 47  | 7              | 12.96%         |
| 3  | 59              | 60  | 1              | 1.69%          |
| 4  | 74              | 76  | 2              | 2.70%          |
| 5  | 91              | 94  | 3              | 3.30%          |
| 6  | 107             | 110 | 3              | 2.80%          |
| 7  | 123             | 125 | 2              | 1.63%          |
| 8  | 139             | 141 | 2              | 1.44%          |

vloxseg2.v with different VL
| VL | Saturn‑baseline | MCA | abs‑difference | rel‑difference |
|----|-----------------|-----|----------------|----------------|
| 8  | 38              | 37  | 1              | 2.63%          |
| 16 | 71              | 63  | 8              | 11.27%         |
| 31 | 129             | 113 | 16             | 12.40%         |