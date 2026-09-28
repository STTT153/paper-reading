# Report

## 1. Shuttle and Boom gather comparison

### Gather unrolling
```riscv
.Linner32_\@:
    lw a3, 0(a5)
    add a3, a0, a3
    lwu a3, 0(a3)
    add t0, t0, a3
    .if \unroll >= 2
    lw a3, 4(a5)
    add a3, a0, a3
    lwu a3, 0(a3)
    add t1, t1, a3
    .endif
    .if \unroll >= 4
    lw a3, 8(a5)
    add a3, a0, a3
    lwu a3, 0(a3)
    add t2, t2, a3
    lw a3, 12(a5)
    add a3, a0, a3
    lwu a3, 0(a3)
    add t3, t3, a3
    .endif
    .if \unroll >= 8
    lw a3, 16(a5)
    add a3, a0, a3
    lwu a3, 0(a3)
    add t4, t4, a3
    lw a3, 20(a5)
    add a3, a0, a3
    lwu a3, 0(a3)
    add t5, t5, a3
    lw a3, 24(a5)
    add a3, a0, a3
    lwu a3, 0(a3)
    add t6, t6, a3
    lw a3, 28(a5)
    add a3, a0, a3
    lwu a3, 0(a3)
    add a4, a4, a3
    .endif
    addi a5, a5, (4 * \unroll)
    addi a6, a6, -\unroll
    bnez a6, .Linner32_\@
    addi a7, a7, -1
    bnez a7, .Louter32_\@
    add a0, t0, t1
    add a0, a0, t2
    add a0, a0, t3
    add a0, a0, t4
    add a0, a0, t5
    add a0, a0, t6
    add a0, a0, a4
    ret
    .endm
```
- Total Elements: 128
- Working Set: 8 KiB
- Speed up: BOOM cycles/ Shuttle cycles

| Path | EEW | Pattern | Unroll | Shuttle cycles | BOOM cycles | BOOM Speed up |
|---|---:|---|---:|---:|---:|---:|
| l1-hot | 32-bit | line-local | 2 | 931 | 786 | 1.184x |
| l1-hot | 32-bit | line-local | 4 | 938 | 690 | 1.359x |
| l1-hot | 32-bit | line-local | 8 | 934 | 642 | 1.455x |
| l1-hot | 64-bit | line-local | 2 | 933 | 784 | 1.190x |
| l1-hot | 64-bit | line-local | 4 | 933 | 688 | 1.356x |
| l1-hot | 64-bit | line-local | 8 | 945 | 640 | 1.477x |
| llc-hot | 32-bit | line-local | 2 | 931 | 786 | 1.184x |
| llc-hot | 32-bit | line-local | 4 | 930 | 684 | 1.360x |
| llc-hot | 32-bit | line-local | 8 | 930 | 636 | 1.462x |
| llc-hot | 64-bit | line-local | 2 | 934 | 778 | 1.201x |
| llc-hot | 64-bit | line-local | 4 | 933 | 688 | 1.356x |
| llc-hot | 64-bit | line-local | 8 | 932 | 634 | 1.470x |
| llc-hot | 32-bit | set-spread | 2 | 931 | 780 | 1.194x |
| llc-hot | 32-bit | set-spread | 4 | 930 | 684 | 1.360x |
| llc-hot | 32-bit | set-spread | 8 | 930 | 636 | 1.462x |
| llc-hot | 64-bit | set-spread | 2 | 934 | 778 | 1.201x |
| llc-hot | 64-bit | set-spread | 4 | 937 | 682 | 1.374x |
| llc-hot | 64-bit | set-spread | 8 | 932 | 634 | 1.470x |
| llc-hot | 32-bit | random-permutation | 2 | 4,440 | 2,855 | 1.555x |
| llc-hot | 32-bit | random-permutation | 4 | 4,297 | 2,801 | 1.534x |
| llc-hot | 32-bit | random-permutation | 8 | 4,255 | 2,774 | 1.534x |
| llc-hot | 64-bit | random-permutation | 2 | 4,391 | 2,783 | 1.578x |
| llc-hot | 64-bit | random-permutation | 4 | 4,324 | 2,811 | 1.538x |
| llc-hot | 64-bit | random-permutation | 8 | 4,309 | 2,809 | 1.534x |

BOOM has around 1.5x speed up when unrolling factor is 8(same as vector scalar comparison).

Shuttle doesn't work well because there is a lot of dependency in the code.

BOOM can mitigate through register renaming.

Unrolling factor increases => more speed up. This might because of less total instruction not unrolling itself.

### Scalar Spmv results

| Core | Cycles | FLOPs / 1000 cycles |
|---|---:|---:|
| Shuttle | 174,422 | 112 |
| BOOM | 144,216 | 136 |

## 2. Saturn PPA Study
| vsgPorts | SGTCM banks | status | syn cells | syn area (um²) | P&R area (um²) | util (%) | setup power | hold power | unit |
|---:|---:|---|---:|---:|---:|---:|---:|---:|---|
| 8 | 0 | syn_done | 1927330 | 248491.605600 |  |  |  |  |  |
| 8 | 16 | par_running | 1962736 | 252627.178860 |  |  |  |  |  |
| 8 | 32 | syn_done | 1984684 | 255071.268000 |  |  |  |  |  |
| 8 | 64 | syn_done | 2026998 | 260458.446780 |  |  |  |  |  |
| 8 | 128 | syn_done | 2113580 | 270585.860580 |  |  |  |  |  |
| 16 | 16 | syn_done | 1990233 | 255871.812060 |  |  |  |  |  |
| 16 | 32 | syn_done | 2031554 | 260565.361920 |  |  |  |  |  |
| 16 | 64 | syn_done | 2109842 | 270107.155440 |  |  |  |  |  |
| 16 | 128 | syn_done | 2266329 | 289816.516080 |  |  |  |  |  |
| 32 | 16 | syn_done | 2045519 | 262507.082580 |  |  |  |  |  |
| 32 | 32 | syn_done | 2122402 | 271406.320920 |  |  |  |  |  |
| 32 | 64 | syn_done | 2268014 | 290807.489520 |  |  |  |  |  |
| 32 | 128 | syn_done | 2575041 | 327160.590839 |  |  |  |  |  |

Power data need to wait until P&R (takes significantly long time and I encountered a failure)

Current result shows that Area is O(BP) where B is numbers of SGTCM banks, P is numbers of vsgPorts. Next step I want to run on benchmarks to get some data like Performance/ Area or Performance/ Power.

Also need to look at how other hardware paper evaluate PPA data.