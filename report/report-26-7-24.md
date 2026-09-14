# Indexed load for different VL

## Goal

Measure the steady-state cost of an eight-way-unrolled `vluxei32.v` sequence
while varying both vector length (VL) and the index access pattern.

The program detects VLMAX for the selected LMUL at runtime and tests the
complete range `1..VLMAX`. Assuming VLEN=256 (X100) and SEW=32, the mapping is:

| LMUL | VLMAX | Tested VL range |
| ---: | ---: | ---: |
| 1 | 8 | 1–8 |
| 2 | 16 | 1–16 |
| 4 | 32 | 1–32 |

## Test matrix

Each row is a VL and each column is one of the following byte-offset patterns:

| Pattern | Index `i` byte offset | Locality exercised |
| --- | ---: | --- |
| `contiguous` | `(i * 4) % 4096` | Unit-stride `int32_t` data |
| `stride_16B` | `(i * 16) % 4096` | Four elements per 64-byte cache line |
| `cacheline_64B` | `(i * 64) % 4096` | One element per cache line; wraps after 64 lines |
| `random_in_page` | fixed shuffled aligned-word order | Irregular access inside the same page |

## Effect of memory system

In this test, I want to minimize the effects of cache misses, TLB misses, and page faults on the measurements.

The test data region is exactly 4 KiB in size, aligned to a 4 KiB boundary, and therefore fits entirely within the 64 KiB L1 data cache. All indices are 32-bit-aligned byte offsets in the inclusive range 0–4092, ensuring that every element accessed by vluxei32.v remains within the same 4 KiB page. This avoids cross-page accesses and minimizes TLB-related effects.

Each benchmark executes the same access pattern for thousands of iterations, allowing both the data and its page translation to remain hot after the initial warm-up. The measured L1D cache-miss rate is below 0.1%, as independently verified using `perf`.

## Method

For every matrix cell, the program:

1. builds the selected index vector in memory;
2. measures `baseline_kernel` and `indexed_load_kernel` as a pair, using either
   a per-thread `perf_event_open` CPU-cycle counter (`--cycel`, the default) or
   `clock_gettime(CLOCK_MONOTONIC)` (`--wallclock`);
3. repeats the pair for the configured repeat count, alternating which kernel
   runs first;
4. computes `(indexed - baseline) / (iterations * 8)` to obtain cycles or
   nanoseconds per `vluxei32.v`, then divides that value by VL to obtain the
   value per loaded element; and
5. reports the median paired result.

For every supported LMUL, the corresponding assembly kernel and baseline have
the same ABI, vector setup, index load, scalar loop, sink store, and fence. The
indexed loop contains eight `vluxei32.v` instructions that are absent from its
baseline. They are distributed over as many non-overlapping, LMUL-aligned
destination register groups as possible.

An example kernel for LMUL=1

```asm
    # void indexed_load_kernel_mX(const uint32_t *data,
    #                             const uint32_t *indices,
    #                             size_t vl, size_t iterations,
    #                             uint32_t *sink);

    .globl  indexed_load_kernel_m1
    .type   indexed_load_kernel_m1, @function
indexed_load_kernel_m1:
    vsetvli t0, a2, e32, m1, ta, ma
    vle32.v v0, (a1)
1:
    vluxei32.v v8, (a0), v0
    vluxei32.v v9, (a0), v0
    vluxei32.v v10, (a0), v0
    vluxei32.v v11, (a0), v0
    vluxei32.v v12, (a0), v0
    vluxei32.v v13, (a0), v0
    vluxei32.v v14, (a0), v0
    vluxei32.v v15, (a0), v0
    addi    a3, a3, -1
    bnez    a3, 1b
    vse32.v v15, (a4)
    fence   rw, rw
    ret
    .size   indexed_load_kernel_m1, .-indexed_load_kernel_m1
```

Kernel for scalar version.
```asm
scalar_load_kernel:
    li      t5, 0
1:
    mv      t0, a1
    mv      t1, a2
2:
    lwu     t2, 0(t0)
    add     t3, a0, t2
    lwu     t4, 0(t3)
    lwu     t5, 0(t3)
    lwu     t6, 0(t3)
    lwu     a5, 0(t3)
    lwu     a6, 0(t3)
    lwu     a7, 0(t3)
    lwu     t4, 0(t3)
    lwu     t5, 0(t3)
    addi    t0, t0, 4
    addi    t1, t1, -1
    bnez    t1, 2b
    addi    a3, a3, -1
    bnez    a3, 1b

    sw      t5, 0(a4)
    fence   rw, rw
    ret
    .size   scalar_load_kernel, .-scalar_load_kernel
```

## Experiment results

The results below use the current eight-way-unrolled, perf-event-based
implementation.

| Parameter | Value |
| --- | ---: |
| CPU | 3 |

All measurements are paired, baseline-subtracted CPU-cycle values. The
`cycles/vluxei` columns are normalized by `iterations * 8`; the
`cycles/element` columns are normalized once more by VL.

### LMUL = 1

| VL | contiguous cycles/vluxei | contiguous cycles/element | stride 16B cycles/vluxei | stride 16B cycles/element | cacheline 64B cycles/vluxei | cacheline 64B cycles/element | random in page cycles/vluxei | random in page cycles/element |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 1 | 107.6752 | 107.6752 | 107.3168 | 107.3168 | 107.2858 | 107.2858 | 107.2480 | 107.2480 |
| 2 | 94.2365 | 47.1183 | 94.1829 | 47.0914 | 94.2945 | 47.1472 | 94.2087 | 47.1043 |
| 3 | 81.1762 | 27.0587 | 81.1800 | 27.0600 | 81.1464 | 27.0488 | 81.1513 | 27.0504 |
| 4 | 68.1056 | 17.0264 | 68.1299 | 17.0325 | 67.9849 | 16.9962 | 67.9777 | 16.9944 |
| 5 | 55.0568 | 11.0114 | 54.9562 | 10.9912 | 54.9433 | 10.9887 | 54.9507 | 10.9901 |
| 6 | 41.9521 | 6.9920 | 41.9643 | 6.9940 | 41.9261 | 6.9877 | 41.9899 | 6.9983 |
| 7 | 28.9220 | 4.1317 | 28.9316 | 4.1331 | 28.9214 | 4.1316 | 28.9267 | 4.1324 |
| 8 | 15.8923 | 1.9865 | 15.9118 | 1.9890 | 15.9093 | 1.9887 | 15.9011 | 1.9876 |

### LMUL = 2

| VL | contiguous cycles/vluxei | contiguous cycles/element | stride 16B cycles/vluxei | stride 16B cycles/element | cacheline 64B cycles/vluxei | cacheline 64B cycles/element | random in page cycles/vluxei | random in page cycles/element |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 1 | 219.6995 | 219.6995 | 219.6622 | 219.6622 | 219.8101 | 219.8101 | 219.6577 | 219.6577 |
| 2 | 206.5465 | 103.2733 | 206.2645 | 103.1323 | 206.1956 | 103.0978 | 206.2210 | 103.1105 |
| 3 | 193.1747 | 64.3916 | 193.1711 | 64.3904 | 193.3204 | 64.4401 | 193.2355 | 64.4118 |
| 4 | 180.1732 | 45.0433 | 180.2164 | 45.0541 | 180.2027 | 45.0507 | 180.1310 | 45.0327 |
| 5 | 167.2193 | 33.4439 | 167.1642 | 33.4328 | 167.1389 | 33.4278 | 167.1631 | 33.4326 |
| 6 | 154.1887 | 25.6981 | 154.1056 | 25.6843 | 154.1522 | 25.6920 | 154.1049 | 25.6841 |
| 7 | 141.0796 | 20.1542 | 141.0872 | 20.1553 | 141.0777 | 20.1540 | 141.1579 | 20.1654 |
| 8 | 128.1050 | 16.0131 | 128.0617 | 16.0077 | 128.1181 | 16.0148 | 128.0397 | 16.0050 |
| 9 | 115.0571 | 12.7841 | 115.0594 | 12.7844 | 115.0402 | 12.7822 | 115.0371 | 12.7819 |
| 10 | 102.1296 | 10.2130 | 102.1206 | 10.2121 | 102.0924 | 10.2092 | 102.0424 | 10.2042 |
| 11 | 89.0142 | 8.0922 | 88.9929 | 8.0903 | 89.0096 | 8.0918 | 89.0270 | 8.0934 |
| 12 | 75.9842 | 6.3320 | 75.9777 | 6.3315 | 75.9934 | 6.3328 | 75.9904 | 6.3325 |
| 13 | 63.0047 | 4.8465 | 62.9843 | 4.8449 | 63.0052 | 4.8466 | 62.9614 | 4.8432 |
| 14 | 49.9615 | 3.5687 | 49.9532 | 3.5681 | 49.9415 | 3.5672 | 49.9468 | 3.5676 |
| 15 | 36.9546 | 2.4636 | 37.0533 | 2.4702 | 36.9597 | 2.4640 | 36.9356 | 2.4624 |
| 16 | 23.9106 | 1.4944 | 23.9039 | 1.4940 | 23.9516 | 1.4970 | 23.8946 | 1.4934 |

### LMUL = 4

| VL | contiguous cycles/vluxei | contiguous cycles/element | stride 16B cycles/vluxei | stride 16B cycles/element | cacheline 64B cycles/vluxei | cacheline 64B cycles/element | random in page cycles/vluxei | random in page cycles/element |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 1 | 444.3659 | 444.3659 | 444.5230 | 444.5230 | 444.4687 | 444.4687 | 443.5531 | 443.5531 |
| 2 | 430.6563 | 215.3282 | 430.5233 | 215.2617 | 430.6638 | 215.3319 | 430.6791 | 215.3395 |
| 3 | 417.6004 | 139.2001 | 417.6249 | 139.2083 | 417.5584 | 139.1861 | 417.5264 | 139.1755 |
| 4 | 404.5631 | 101.1408 | 404.4771 | 101.1193 | 404.4937 | 101.1234 | 404.5299 | 101.1325 |
| 5 | 391.4551 | 78.2910 | 391.5906 | 78.3181 | 391.5158 | 78.3032 | 391.5098 | 78.3020 |
| 6 | 378.5900 | 63.0983 | 378.4801 | 63.0800 | 378.5164 | 63.0861 | 378.5757 | 63.0959 |
| 7 | 365.4858 | 52.2123 | 365.4601 | 52.2086 | 365.5259 | 52.2180 | 365.5175 | 52.2168 |
| 8 | 352.4509 | 44.0564 | 352.5179 | 44.0647 | 352.4365 | 44.0546 | 352.4049 | 44.0506 |
| 9 | 339.5470 | 37.7274 | 339.4854 | 37.7206 | 339.3786 | 37.7087 | 339.3943 | 37.7105 |
| 10 | 326.5226 | 32.6523 | 326.3726 | 32.6373 | 326.9767 | 32.6977 | 327.0901 | 32.7090 |
| 11 | 313.8716 | 28.5338 | 314.1241 | 28.5567 | 313.6139 | 28.5104 | 313.3613 | 28.4874 |
| 12 | 300.3689 | 25.0307 | 300.4049 | 25.0337 | 300.5936 | 25.0495 | 300.3687 | 25.0307 |
| 13 | 287.3916 | 22.1070 | 287.2668 | 22.0974 | 287.3955 | 22.1073 | 287.3157 | 22.1012 |
| 14 | 274.3058 | 19.5933 | 274.3016 | 19.5930 | 274.4024 | 19.6002 | 274.3409 | 19.5958 |
| 15 | 261.3185 | 17.4212 | 261.2241 | 17.4149 | 261.3187 | 17.4212 | 261.3572 | 17.4238 |
| 16 | 248.3344 | 15.5209 | 248.2643 | 15.5165 | 248.2642 | 15.5165 | 248.3332 | 15.5208 |
| 17 | 235.3881 | 13.8464 | 235.2209 | 13.8365 | 235.2882 | 13.8405 | 235.2336 | 13.8373 |
| 18 | 222.3487 | 12.3527 | 222.2289 | 12.3461 | 222.1992 | 12.3444 | 222.1944 | 12.3441 |
| 19 | 209.2478 | 11.0130 | 209.2705 | 11.0142 | 209.1833 | 11.0096 | 209.1686 | 11.0089 |
| 20 | 196.2460 | 9.8123 | 196.1845 | 9.8092 | 196.2321 | 9.8116 | 196.2104 | 9.8105 |
| 21 | 183.1831 | 8.7230 | 183.1800 | 8.7229 | 183.1861 | 8.7231 | 183.1819 | 8.7229 |
| 22 | 170.2202 | 7.7373 | 170.1842 | 7.7356 | 170.1212 | 7.7328 | 170.0999 | 7.7318 |
| 23 | 157.1185 | 6.8312 | 157.1528 | 6.8327 | 157.1803 | 6.8339 | 157.1074 | 6.8308 |
| 24 | 144.0954 | 6.0040 | 144.0967 | 6.0040 | 144.0688 | 6.0029 | 144.1197 | 6.0050 |
| 25 | 131.1910 | 5.2476 | 131.0617 | 5.2425 | 131.0877 | 5.2435 | 131.1437 | 5.2457 |
| 26 | 118.1221 | 4.5432 | 118.0567 | 4.5406 | 118.0584 | 4.5407 | 118.0266 | 4.5395 |
| 27 | 105.0374 | 3.8903 | 105.0538 | 3.8909 | 105.0051 | 3.8891 | 105.0415 | 3.8904 |
| 28 | 92.0763 | 3.2884 | 92.0058 | 3.2859 | 92.0789 | 3.2885 | 91.9836 | 3.2851 |
| 29 | 79.0057 | 2.7243 | 78.9998 | 2.7241 | 79.0689 | 2.7265 | 78.9925 | 2.7239 |
| 30 | 65.9750 | 2.1992 | 65.9760 | 2.1992 | 65.9773 | 2.1992 | 65.9418 | 2.1981 |
| 31 | 52.9823 | 1.7091 | 52.9408 | 1.7078 | 52.9633 | 1.7085 | 53.0144 | 1.7101 |
| 32 | 39.9338 | 1.2479 | 39.9487 | 1.2484 | 39.9322 | 1.2479 | 39.9677 | 1.2490 |

## Scalar version

The scalar measurements used CPU 3 and the following command:

```sh
taskset -c 3 ./scalar_load --max-vl 32
```

| VL | contiguous cycles/lwu | stride 16B cycles/lwu | cacheline 64B cycles/lwu | random in page cycles/lwu |
| ---: | ---: | ---: | ---: | ---: |
| 1 | 0.6250 | 0.6256 | 0.6250 | 0.5000 |
| 2 | 0.2899 | 0.1776 | 0.1341 | 0.2500 |
| 3 | 0.3125 | 0.3980 | 0.3830 | 0.3751 |
| 4 | 0.3576 | 0.3470 | 0.3975 | 0.4062 |
| 5 | 0.4438 | 0.5249 | 0.4682 | 0.4297 |
| 6 | 0.4428 | 0.4588 | 0.4801 | 0.4604 |
| 7 | 0.4560 | 0.5000 | 0.4785 | 0.4861 |
| 8 | 0.4577 | 0.5018 | 0.4765 | 0.4531 |
| 9 | 0.4740 | 0.4877 | 0.4879 | 0.4735 |
| 10 | 0.4783 | 0.4895 | 0.4855 | 0.4987 |
| 11 | 0.4707 | 0.4989 | 0.4906 | 0.4782 |
| 12 | 0.4701 | 0.4805 | 0.5017 | 0.4897 |
| 13 | 0.4823 | 0.5107 | 0.4905 | 0.4719 |
| 14 | 0.4827 | 0.4829 | 0.5053 | 0.4840 |
| 15 | 0.4810 | 0.5010 | 0.4927 | 0.5043 |
| 16 | 0.4878 | 0.5008 | 0.5025 | 0.4966 |
| 17 | 0.4937 | 0.4940 | 0.4941 | 0.5096 |
| 18 | 0.4822 | 0.4963 | 0.5009 | 0.5087 |
| 19 | 0.4869 | 0.5012 | 0.4944 | 0.4946 |
| 20 | 0.4849 | 0.4872 | 0.5007 | 0.5026 |
| 21 | 0.4837 | 0.5072 | 0.4944 | 0.5120 |
| 22 | 0.4911 | 0.4891 | 0.5011 | 0.5064 |
| 23 | 0.4881 | 0.5011 | 0.4956 | 0.5062 |
| 24 | 0.4870 | 0.4999 | 0.5011 | 0.5112 |
| 25 | 0.4941 | 0.4959 | 0.4960 | 0.5158 |
| 26 | 0.4886 | 0.4963 | 0.5014 | 0.5053 |
| 27 | 0.4872 | 0.5005 | 0.4970 | 0.5106 |
| 28 | 0.4919 | 0.4924 | 0.5007 | 0.5143 |
| 29 | 0.4900 | 0.5054 | 0.4965 | 0.5049 |
| 30 | 0.4898 | 0.4920 | 0.5010 | 0.5091 |
| 31 | 0.4912 | 0.5008 | 0.4964 | 0.5125 |
| 32 | 0.4914 | 0.5002 | 0.5010 | 0.5046 |

## Selected perf results

The tables below extract the counters most relevant to the VL-dependent trend
from the raw `perf stat -d -d` output. The benchmark used `stride_16B`, CPU 3,
`--wallclock`, the default 100000 iterations and 5 repeats.

`ns/vluxei` and `ns/lwu` are the program's paired, baseline-subtracted timing
results. In contrast, `cycles`, `instructions`, and the L1 counters are raw
whole-process totals reported by the outer `perf stat`; they include both the
target and baseline runs and must not be interpreted as baseline-subtracted
counts.

### Vector, LMUL = 1

| VL | ns/vluxei | ns/element | cycles (M) | instructions (M) | L1D loads (M) | L1D load misses |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 1 | 48.9194 | 48.9194 | 433.694 | 8.944 | 60.100 | 5,159 |
| 2 | 42.8962 | 21.4481 | 381.773 | 8.797 | 56.097 | 4,869 |
| 3 | 36.9515 | 12.3172 | 329.468 | 8.738 | 52.099 | 5,105 |
| 4 | 31.0786 | 7.7696 | 277.593 | 8.629 | 48.098 | 5,076 |
| 5 | 25.0407 | 5.0081 | 225.086 | 8.375 | 44.100 | 5,082 |
| 6 | 19.1221 | 3.1870 | 173.201 | 8.472 | 40.098 | 5,023 |
| 7 | 13.1935 | 1.8848 | 121.107 | 8.371 | 36.099 | 4,921 |
| 8 | 7.2249 | 0.9031 | 68.545 | 8.124 | 32.098 | 4,936 |

### Vector, LMUL = 4 (measured VL = 1..8)

| VL | ns/vluxei | ns/element | cycles (M) | instructions (M) | L1D loads (M) | L1D load misses |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 1 | 154.6878 | 154.6878 | 1,362.148 | 11.269 | 192.098 | 5,144 |
| 2 | 149.3691 | 74.6845 | 1,311.722 | 12.448 | 188.098 | 5,190 |
| 3 | 142.6928 | 47.5643 | 1,257.449 | 10.818 | 184.100 | 4,952 |
| 4 | 136.8450 | 34.2113 | 1,206.306 | 11.451 | 180.098 | 5,058 |
| 5 | 130.9266 | 26.1853 | 1,153.506 | 10.714 | 176.099 | 5,062 |
| 6 | 124.8970 | 20.8162 | 1,100.223 | 10.497 | 172.098 | 5,018 |
| 7 | 119.0554 | 17.0079 | 1,048.653 | 10.472 | 168.100 | 5,052 |
| 8 | 112.9807 | 14.1226 | 996.593 | 10.402 | 164.099 | 5,141 |

### Scalar reference

| VL | ns/lwu | cycles (M) | instructions (M) | L1D loads (M) | L1D load misses |
| ---: | ---: | ---: | ---: | ---: | ---: |
| 1 | 0.2273 | 9.817 | 14.982 | 5.097 | 4,766 |
| 2 | 0.1737 | 15.717 | 24.029 | 10.096 | 4,873 |
| 3 | 0.1263 | 19.913 | 33.097 | 15.098 | 4,896 |
| 4 | 0.1282 | 24.388 | 42.028 | 20.097 | 4,839 |
| 5 | 0.2272 | 29.044 | 51.279 | 25.098 | 4,844 |
| 6 | 0.1989 | 30.379 | 60.029 | 30.097 | 4,815 |
| 7 | 0.2209 | 35.386 | 69.012 | 35.099 | 4,796 |
| 8 | 0.2227 | 39.398 | 78.030 | 40.099 | 4,874 |

### Endpoint trends

The following slopes are computed from the VL=1 and VL=8 endpoints. They are
descriptive summaries of these whole-process measurements, not
baseline-subtracted PMU counts.

| Test | VL range | timing change per +1 VL | cycles change per +1 VL | L1D-load change per +1 VL |
| --- | ---: | ---: | ---: | ---: |
| Vector LMUL=1 | 1–8 | -5.956 ns/vluxei | -52.164 M | -4.000 M |
| Vector LMUL=4 | 1–8 | -5.958 ns/vluxei | -52.222 M | -4.000 M |
| Scalar | 1–8 | -0.0007 ns/lwu; no monotonic trend | +4.226 M | +5.000 M |

The two vector configurations have almost identical per-VL slopes despite
different LMUL values. The scalar reference has the opposite L1D-load trend:
its count grows linearly with VL. L1D-load-miss counts remain around five
thousand in every case, so the large timing trend is not accompanied by a
corresponding cache-miss trend.

Another notable property of the raw data is that `dTLB-loads` and
`L1-dcache-loads` are almost identical in every run. Across all three tables,
the two counters differ by only 19–31 events out of millions. This unusually
close correspondence should be considered when interpreting the platform's
generic perf-event mappings.

## Raw perf output

The original terminal output is retained below for reproducibility.

<details>
<summary>Show the complete perf stat logs</summary>

```text
shiyin@ai-spacemitk3picoitx:~/workspace/rvv-microbench/memory/indexed-vl$ perf stat -d -d taskset -c 3 ./indexed_load  --pattern stride_16B --wallclo
ck --lmul 1 --vl 1
CPU = 3
LMUL = 1
Timing = wallclock
Kernel = paired
vl,stride_16B_difference_ns_per_vluxei,stride_16B_difference_ns_per_element
1,48.9194,48.9194

 Performance counter stats for 'taskset -c 3 ./indexed_load --pattern stride_16B --wallclock --lmul 1 --vl 1':

       197,342,749      task-clock:u                     #    0.994 CPUs utilized
                 0      context-switches:u               #    0.000 /sec
                 0      cpu-migrations:u                 #    0.000 /sec
               100      page-faults:u                    #  506.733 /sec
         8,944,440      instructions:u                   #    0.02  insn per cycle
                                                  #   47.68  stalled cycles per insn
       433,693,631      cycles:u                         #    2.198 GHz
       426,430,203      stalled-cycles-frontend:u        #   98.33% frontend cycles idle
       321,930,157      stalled-cycles-backend:u         #   74.23% backend cycles idle
         1,048,938      branches:u                       #    5.315 M/sec
             5,309      branch-misses:u                  #    0.51% of all branches
        60,099,820      L1-dcache-loads:u                #  304.545 M/sec
             5,159      L1-dcache-load-misses:u          #    0.01% of all L1-dcache accesses
       108,794,677      L1-icache-loads:u                #  551.298 M/sec
             3,141      L1-icache-load-misses:u          #    0.00% of all L1-icache accesses
        60,099,845      dTLB-loads:u                     #  304.545 M/sec
       429,095,962      iTLB-loads:u                     #    2.174 G/sec
               581      iTLB-load-misses:u               #    0.00% of all iTLB cache accesses

       0.198613794 seconds time elapsed

       0.198323000 seconds user
       0.000000000 seconds sys


shiyin@ai-spacemitk3picoitx:~/workspace/rvv-microbench/memory/indexed-vl$ perf stat -d -d taskset -c 3 ./indexed_load  --pattern stride_16B --wallclock --lmul 1 --vl 2
CPU = 3
LMUL = 1
Timing = wallclock
Kernel = paired
vl,stride_16B_difference_ns_per_vluxei,stride_16B_difference_ns_per_element
2,42.8962,21.4481

 Performance counter stats for 'taskset -c 3 ./indexed_load --pattern stride_16B --wallclock --lmul 1 --vl 2':

       173,715,999      task-clock:u                     #    0.993 CPUs utilized
                 0      context-switches:u               #    0.000 /sec
                 0      cpu-migrations:u                 #    0.000 /sec
               100      page-faults:u                    #  575.652 /sec
         8,797,219      instructions:u                   #    0.02  insn per cycle
                                                  #   42.56  stalled cycles per insn
       381,773,329      cycles:u                         #    2.198 GHz
       374,432,724      stalled-cycles-frontend:u        #   98.08% frontend cycles idle
       282,912,396      stalled-cycles-backend:u         #   74.10% backend cycles idle
         1,048,946      branches:u                       #    6.038 M/sec
             5,116      branch-misses:u                  #    0.49% of all branches
        56,097,261      L1-dcache-loads:u                #  322.925 M/sec
             4,869      L1-dcache-load-misses:u          #    0.01% of all L1-dcache accesses
        95,782,800      L1-icache-loads:u                #  551.376 M/sec
             3,031      L1-icache-load-misses:u          #    0.00% of all L1-icache accesses
        56,097,289      dTLB-loads:u                     #  322.925 M/sec
       377,094,536      iTLB-loads:u                     #    2.171 G/sec
               496      iTLB-load-misses:u               #    0.00% of all iTLB cache accesses

       0.174966721 seconds time elapsed

       0.174699000 seconds user
       0.000000000 seconds sys


shiyin@ai-spacemitk3picoitx:~/workspace/rvv-microbench/memory/indexed-vl$ perf stat -d -d taskset -c 3 ./indexed_load  --pattern stride_16B --wallclock --lmul 1 --vl 3
CPU = 3
LMUL = 1
Timing = wallclock
Kernel = paired
vl,stride_16B_difference_ns_per_vluxei,stride_16B_difference_ns_per_element
3,36.9515,12.3172

 Performance counter stats for 'taskset -c 3 ./indexed_load --pattern stride_16B --wallclock --lmul 1 --vl 3':

       149,935,042      task-clock:u                     #    0.991 CPUs utilized
                 0      context-switches:u               #    0.000 /sec
                 0      cpu-migrations:u                 #    0.000 /sec
               100      page-faults:u                    #  666.955 /sec
         8,738,327      instructions:u                   #    0.03  insn per cycle
                                                  #   36.90  stalled cycles per insn
       329,467,995      cycles:u                         #    2.197 GHz
       322,457,168      stalled-cycles-frontend:u        #   97.87% frontend cycles idle
       243,956,605      stalled-cycles-backend:u         #   74.05% backend cycles idle
         1,048,938      branches:u                       #    6.996 M/sec
             5,349      branch-misses:u                  #    0.51% of all branches
        52,098,526      L1-dcache-loads:u                #  347.474 M/sec
             5,105      L1-dcache-load-misses:u          #    0.01% of all L1-dcache accesses
        82,802,733      L1-icache-loads:u                #  552.257 M/sec
             3,129      L1-icache-load-misses:u          #    0.00% of all L1-icache accesses
        52,098,551      dTLB-loads:u                     #  347.474 M/sec
       325,113,533      iTLB-loads:u                     #    2.168 G/sec
               534      iTLB-load-misses:u               #    0.00% of all iTLB cache accesses

       0.151270568 seconds time elapsed

       0.150966000 seconds user
       0.000000000 seconds sys


shiyin@ai-spacemitk3picoitx:~/workspace/rvv-microbench/memory/indexed-vl$ perf stat -d -d taskset -c 3 ./indexed_load  --pattern stride_16B --wallclock --lmul 1 --vl 4
CPU = 3
LMUL = 1
Timing = wallclock
Kernel = paired
vl,stride_16B_difference_ns_per_vluxei,stride_16B_difference_ns_per_element
4,31.0786,7.7696

 Performance counter stats for 'taskset -c 3 ./indexed_load --pattern stride_16B --wallclock --lmul 1 --vl 4':

       126,364,666      task-clock:u                     #    0.990 CPUs utilized
                 0      context-switches:u               #    0.000 /sec
                 0      cpu-migrations:u                 #    0.000 /sec
               104      page-faults:u                    #  823.015 /sec
         8,628,515      instructions:u                   #    0.03  insn per cycle
                                                  #   31.35  stalled cycles per insn
       277,593,347      cycles:u                         #    2.197 GHz
       270,466,102      stalled-cycles-frontend:u        #   97.43% frontend cycles idle
       204,960,374      stalled-cycles-backend:u         #   73.83% backend cycles idle
         1,048,820      branches:u                       #    8.300 M/sec
             5,234      branch-misses:u                  #    0.50% of all branches
        48,098,004      L1-dcache-loads:u                #  380.629 M/sec
             5,076      L1-dcache-load-misses:u          #    0.01% of all L1-dcache accesses
        69,805,622      L1-icache-loads:u                #  552.414 M/sec
             3,195      L1-icache-load-misses:u          #    0.00% of all L1-icache accesses
        48,098,030      dTLB-loads:u                     #  380.629 M/sec
       273,122,849      iTLB-loads:u                     #    2.161 G/sec
               528      iTLB-load-misses:u               #    0.00% of all iTLB cache accesses

       0.127598161 seconds time elapsed

       0.123392000 seconds user
       0.003980000 seconds sys


shiyin@ai-spacemitk3picoitx:~/workspace/rvv-microbench/memory/indexed-vl$ perf stat -d -d taskset -c 3 ./indexed_load  --pattern stride_16B --wallclock --lmul 1 --vl 5
CPU = 3
LMUL = 1
Timing = wallclock
Kernel = paired
vl,stride_16B_difference_ns_per_vluxei,stride_16B_difference_ns_per_element
5,25.0407,5.0081

 Performance counter stats for 'taskset -c 3 ./indexed_load --pattern stride_16B --wallclock --lmul 1 --vl 5':

       102,385,999      task-clock:u                     #    0.991 CPUs utilized
                 0      context-switches:u               #    0.000 /sec
                 0      cpu-migrations:u                 #    0.000 /sec
               102      page-faults:u                    #  996.230 /sec
         8,374,849      instructions:u                   #    0.04  insn per cycle
                                                  #   26.09  stalled cycles per insn
       225,085,742      cycles:u                         #    2.198 GHz
       218,476,029      stalled-cycles-frontend:u        #   97.06% frontend cycles idle
       165,971,968      stalled-cycles-backend:u         #   73.74% backend cycles idle
         1,048,816      branches:u                       #   10.244 M/sec
             5,534      branch-misses:u                  #    0.53% of all branches
        44,100,099      L1-dcache-loads:u                #  430.724 M/sec
             5,082      L1-dcache-load-misses:u          #    0.01% of all L1-dcache accesses
        56,813,837      L1-icache-loads:u                #  554.898 M/sec
             3,140      L1-icache-load-misses:u          #    0.01% of all L1-icache accesses
        44,100,128      dTLB-loads:u                     #  430.724 M/sec
       221,130,321      iTLB-loads:u                     #    2.160 G/sec
               469      iTLB-load-misses:u               #    0.00% of all iTLB cache accesses

       0.103342369 seconds time elapsed

       0.103397000 seconds user
       0.000000000 seconds sys


shiyin@ai-spacemitk3picoitx:~/workspace/rvv-microbench/memory/indexed-vl$ perf stat -d -d taskset -c 3 ./indexed_load  --pattern stride_16B --wallclock --lmul 1 --vl 6
CPU = 3
LMUL = 1
Timing = wallclock
Kernel = paired
vl,stride_16B_difference_ns_per_vluxei,stride_16B_difference_ns_per_element
6,19.1221,3.1870

 Performance counter stats for 'taskset -c 3 ./indexed_load --pattern stride_16B --wallclock --lmul 1 --vl 6':

        78,910,289      task-clock:u                     #    0.984 CPUs utilized
                 0      context-switches:u               #    0.000 /sec
                 0      cpu-migrations:u                 #    0.000 /sec
               104      page-faults:u                    #    1.318 K/sec
         8,472,360      instructions:u                   #    0.05  insn per cycle
                                                  #   19.65  stalled cycles per insn
       173,201,373      cycles:u                         #    2.195 GHz
       166,454,151      stalled-cycles-frontend:u        #   96.10% frontend cycles idle
       126,955,012      stalled-cycles-backend:u         #   73.30% backend cycles idle
         1,048,821      branches:u                       #   13.291 M/sec
             5,251      branch-misses:u                  #    0.50% of all branches
        40,098,227      L1-dcache-loads:u                #  508.150 M/sec
             5,023      L1-dcache-load-misses:u          #    0.01% of all L1-dcache accesses
        43,800,702      L1-icache-loads:u                #  555.070 M/sec
             3,155      L1-icache-load-misses:u          #    0.01% of all L1-icache accesses
        40,098,252      dTLB-loads:u                     #  508.150 M/sec
       169,113,664      iTLB-loads:u                     #    2.143 G/sec
               508      iTLB-load-misses:u               #    0.00% of all iTLB cache accesses

       0.080167934 seconds time elapsed

       0.075919000 seconds user
       0.003995000 seconds sys


shiyin@ai-spacemitk3picoitx:~/workspace/rvv-microbench/memory/indexed-vl$ perf stat -d -d taskset -c 3 ./indexed_load  --pattern stride_16B --wallclock --lmul 1 --vl 7
CPU = 3
LMUL = 1
Timing = wallclock
Kernel = paired
vl,stride_16B_difference_ns_per_vluxei,stride_16B_difference_ns_per_element
7,13.1935,1.8848

 Performance counter stats for 'taskset -c 3 ./indexed_load --pattern stride_16B --wallclock --lmul 1 --vl 7':

        55,175,875      task-clock:u                     #    0.980 CPUs utilized
                 0      context-switches:u               #    0.000 /sec
                 0      cpu-migrations:u                 #    0.000 /sec
               100      page-faults:u                    #    1.812 K/sec
         8,371,145      instructions:u                   #    0.07  insn per cycle
                                                  #   13.68  stalled cycles per insn
       121,106,765      cycles:u                         #    2.195 GHz
       114,498,645      stalled-cycles-frontend:u        #   94.54% frontend cycles idle
        87,986,678      stalled-cycles-backend:u         #   72.65% backend cycles idle
         1,048,826      branches:u                       #   19.009 M/sec
             5,343      branch-misses:u                  #    0.51% of all branches
        36,099,495      L1-dcache-loads:u                #  654.262 M/sec
             4,921      L1-dcache-load-misses:u          #    0.01% of all L1-dcache accesses
        30,819,390      L1-icache-loads:u                #  558.566 M/sec
             3,218      L1-icache-load-misses:u          #    0.01% of all L1-icache accesses
        36,099,523      dTLB-loads:u                     #  654.263 M/sec
       117,151,390      iTLB-loads:u                     #    2.123 G/sec
               488      iTLB-load-misses:u               #    0.00% of all iTLB cache accesses

       0.056309202 seconds time elapsed

       0.056179000 seconds user
       0.000000000 seconds sys


shiyin@ai-spacemitk3picoitx:~/workspace/rvv-microbench/memory/indexed-vl$ perf stat -d -d taskset -c 3 ./indexed_load  --pattern stride_16B --wallclock --lmul 1 --vl 8
CPU = 3
LMUL = 1
Timing = wallclock
Kernel = paired
vl,stride_16B_difference_ns_per_vluxei,stride_16B_difference_ns_per_element
8,7.2249,0.9031

 Performance counter stats for 'taskset -c 3 ./indexed_load --pattern stride_16B --wallclock --lmul 1 --vl 8':

        31,211,958      task-clock:u                     #    0.971 CPUs utilized
                 0      context-switches:u               #    0.000 /sec
                 0      cpu-migrations:u                 #    0.000 /sec
               101      page-faults:u                    #    3.236 K/sec
         8,123,806      instructions:u                   #    0.12  insn per cycle
                                                  #    7.69  stalled cycles per insn
        68,544,961      cycles:u                         #    2.196 GHz
        62,455,699      stalled-cycles-frontend:u        #   91.12% frontend cycles idle
        48,948,101      stalled-cycles-backend:u         #   71.41% backend cycles idle
         1,048,734      branches:u                       #   33.600 M/sec
             5,147      branch-misses:u                  #    0.49% of all branches
        32,097,873      L1-dcache-loads:u                #    1.028 G/sec
             4,936      L1-dcache-load-misses:u          #    0.02% of all L1-dcache accesses
        17,800,568      L1-icache-loads:u                #  570.312 M/sec
             3,045      L1-icache-load-misses:u          #    0.02% of all L1-icache accesses
        32,097,892      dTLB-loads:u                     #    1.028 G/sec
        65,109,783      iTLB-loads:u                     #    2.086 G/sec
               489      iTLB-load-misses:u               #    0.00% of all iTLB cache accesses

       0.032131821 seconds time elapsed

       0.032248000 seconds user
       0.000000000 seconds sys


shiyin@ai-spacemitk3picoitx:~/workspace/rvv-microbench/memory/indexed-vl$ perf stat -d -d taskset -c 3 ./indexed_load  --pattern stride_16B --wallclock --lmul 4 --vl 1
CPU = 3
LMUL = 4
Timing = wallclock
Kernel = paired
vl,stride_16B_difference_ns_per_vluxei,stride_16B_difference_ns_per_element
1,154.6878,154.6878

 Performance counter stats for 'taskset -c 3 ./indexed_load --pattern stride_16B --wallclock --lmul 4 --vl 1':

       619,876,456      task-clock:u                     #    0.996 CPUs utilized
                 0      context-switches:u               #    0.000 /sec
                 0      cpu-migrations:u                 #    0.000 /sec
               101      page-faults:u                    #  162.936 /sec
        11,268,887      instructions:u                   #    0.01  insn per cycle
                                                  #  119.84  stalled cycles per insn
     1,362,148,397      cycles:u                         #    2.197 GHz
     1,350,513,891      stalled-cycles-frontend:u        #   99.15% frontend cycles idle
     1,014,948,402      stalled-cycles-backend:u         #   74.51% backend cycles idle
         1,049,006      branches:u                       #    1.692 M/sec
             5,158      branch-misses:u                  #    0.49% of all branches
       192,098,449      L1-dcache-loads:u                #  309.898 M/sec
             5,144      L1-dcache-load-misses:u          #    0.00% of all L1-dcache accesses
       654,139,165      L1-icache-loads:u                #    1.055 G/sec
             3,315      L1-icache-load-misses:u          #    0.00% of all L1-icache accesses
       192,098,475      dTLB-loads:u                     #  309.898 M/sec
     1,353,172,070      iTLB-loads:u                     #    2.183 G/sec
               525      iTLB-load-misses:u               #    0.00% of all iTLB cache accesses

       0.622240343 seconds time elapsed

       0.620806000 seconds user
       0.000000000 seconds sys


shiyin@ai-spacemitk3picoitx:~/workspace/rvv-microbench/memory/indexed-vl$ perf stat -d -d taskset -c 3 ./indexed_load  --pattern stride_16B --wallclock --lmul 4 --vl 2
CPU = 3
LMUL = 4
Timing = wallclock
Kernel = paired
vl,stride_16B_difference_ns_per_vluxei,stride_16B_difference_ns_per_element
2,149.3691,74.6845

 Performance counter stats for 'taskset -c 3 ./indexed_load --pattern stride_16B --wallclock --lmul 4 --vl 2':

       597,787,956      task-clock:u                     #    0.993 CPUs utilized
                 0      context-switches:u               #    0.000 /sec
                 0      cpu-migrations:u                 #    0.000 /sec
               101      page-faults:u                    #  168.956 /sec
        12,448,033      instructions:u                   #    0.01  insn per cycle
                                                  #  104.31  stalled cycles per insn
     1,311,722,072      cycles:u                         #    2.194 GHz
     1,298,510,592      stalled-cycles-frontend:u        #   98.99% frontend cycles idle
       975,954,898      stalled-cycles-backend:u         #   74.40% backend cycles idle
         1,048,986      branches:u                       #    1.755 M/sec
             5,109      branch-misses:u                  #    0.49% of all branches
       188,098,370      L1-dcache-loads:u                #  314.657 M/sec
             5,190      L1-dcache-load-misses:u          #    0.00% of all L1-dcache accesses
       648,708,654      L1-icache-loads:u                #    1.085 G/sec
             3,455      L1-icache-load-misses:u          #    0.00% of all L1-icache accesses
       188,098,398      dTLB-loads:u                     #  314.657 M/sec
     1,301,164,593      iTLB-loads:u                     #    2.177 G/sec
               578      iTLB-load-misses:u               #    0.00% of all iTLB cache accesses

       0.601946676 seconds time elapsed

       0.594674000 seconds user
       0.003991000 seconds sys


shiyin@ai-spacemitk3picoitx:~/workspace/rvv-microbench/memory/indexed-vl$ perf stat -d -d taskset -c 3 ./indexed_load  --pattern stride_16B --wallclock --lmul 4 --vl 3
CPU = 3
LMUL = 4
Timing = wallclock
Kernel = paired
vl,stride_16B_difference_ns_per_vluxei,stride_16B_difference_ns_per_element
3,142.6928,47.5643

 Performance counter stats for 'taskset -c 3 ./indexed_load --pattern stride_16B --wallclock --lmul 4 --vl 3':

       572,113,703      task-clock:u                     #    0.996 CPUs utilized
                 0      context-switches:u               #    0.000 /sec
                 0      cpu-migrations:u                 #    0.000 /sec
               100      page-faults:u                    #  174.790 /sec
        10,818,303      instructions:u                   #    0.01  insn per cycle
                                                  #  115.22  stalled cycles per insn
     1,257,449,334      cycles:u                         #    2.198 GHz
     1,246,503,933      stalled-cycles-frontend:u        #   99.13% frontend cycles idle
       936,967,372      stalled-cycles-backend:u         #   74.51% backend cycles idle
         1,048,980      branches:u                       #    1.834 M/sec
             5,401      branch-misses:u                  #    0.51% of all branches
       184,100,003      L1-dcache-loads:u                #  321.789 M/sec
             4,952      L1-dcache-load-misses:u          #    0.00% of all L1-dcache accesses
       609,340,690      L1-icache-loads:u                #    1.065 G/sec
             3,297      L1-icache-load-misses:u          #    0.00% of all L1-icache accesses
       184,100,026      dTLB-loads:u                     #  321.789 M/sec
     1,249,162,839      iTLB-loads:u                     #    2.183 G/sec
               554      iTLB-load-misses:u               #    0.00% of all iTLB cache accesses

       0.574162453 seconds time elapsed

       0.573090000 seconds user
       0.000000000 seconds sys


shiyin@ai-spacemitk3picoitx:~/workspace/rvv-microbench/memory/indexed-vl$ perf stat -d -d taskset -c 3 ./indexed_load  --pattern stride_16B --wallclock --lmul 4 --vl 4
CPU = 3
LMUL = 4
Timing = wallclock
Kernel = paired
vl,stride_16B_difference_ns_per_vluxei,stride_16B_difference_ns_per_element
4,136.8450,34.2113

 Performance counter stats for 'taskset -c 3 ./indexed_load --pattern stride_16B --wallclock --lmul 4 --vl 4':

       549,299,996      task-clock:u                     #    0.995 CPUs utilized
                 0      context-switches:u               #    0.000 /sec
                 0      cpu-migrations:u                 #    0.000 /sec
               102      page-faults:u                    #  185.691 /sec
        11,450,540      instructions:u                   #    0.01  insn per cycle
                                                  #  104.32  stalled cycles per insn
     1,206,306,051      cycles:u                         #    2.196 GHz
     1,194,529,234      stalled-cycles-frontend:u        #   99.02% frontend cycles idle
       897,951,666      stalled-cycles-backend:u         #   74.44% backend cycles idle
         1,048,982      branches:u                       #    1.910 M/sec
             5,069      branch-misses:u                  #    0.48% of all branches
       180,098,364      L1-dcache-loads:u                #  327.869 M/sec
             5,058      L1-dcache-load-misses:u          #    0.00% of all L1-dcache accesses
       587,490,494      L1-icache-loads:u                #    1.070 G/sec
             3,295      L1-icache-load-misses:u          #    0.00% of all L1-icache accesses
       180,098,391      dTLB-loads:u                     #  327.869 M/sec
     1,197,177,481      iTLB-loads:u                     #    2.179 G/sec
               560      iTLB-load-misses:u               #    0.00% of all iTLB cache accesses

       0.552323403 seconds time elapsed

       0.546247000 seconds user
       0.003987000 seconds sys


shiyin@ai-spacemitk3picoitx:~/workspace/rvv-microbench/memory/indexed-vl$ perf stat -d -d taskset -c 3 ./indexed_load  --pattern stride_16B --wallclock --lmul 4 --vl 5
CPU = 3
LMUL = 4
Timing = wallclock
Kernel = paired
vl,stride_16B_difference_ns_per_vluxei,stride_16B_difference_ns_per_element
5,130.9266,26.1853

 Performance counter stats for 'taskset -c 3 ./indexed_load --pattern stride_16B --wallclock --lmul 4 --vl 5':

       524,818,665      task-clock:u                     #    0.996 CPUs utilized
                 0      context-switches:u               #    0.000 /sec
                 0      cpu-migrations:u                 #    0.000 /sec
               101      page-faults:u                    #  192.447 /sec
        10,713,587      instructions:u                   #    0.01  insn per cycle
                                                  #  106.64  stalled cycles per insn
     1,153,505,662      cycles:u                         #    2.198 GHz
     1,142,545,061      stalled-cycles-frontend:u        #   99.05% frontend cycles idle
       858,851,800      stalled-cycles-backend:u         #   74.46% backend cycles idle
         1,048,980      branches:u                       #    1.999 M/sec
             5,190      branch-misses:u                  #    0.49% of all branches
       176,099,415      L1-dcache-loads:u                #  335.543 M/sec
             5,062      L1-dcache-load-misses:u          #    0.00% of all L1-dcache accesses
       569,992,371      L1-icache-loads:u                #    1.086 G/sec
             3,287      L1-icache-load-misses:u          #    0.00% of all L1-icache accesses
       176,099,441      dTLB-loads:u                     #  335.543 M/sec
     1,145,194,829      iTLB-loads:u                     #    2.182 G/sec
               548      iTLB-load-misses:u               #    0.00% of all iTLB cache accesses

       0.526941047 seconds time elapsed

       0.521911000 seconds user
       0.003984000 seconds sys


shiyin@ai-spacemitk3picoitx:~/workspace/rvv-microbench/memory/indexed-vl$ perf stat -d -d taskset -c 3 ./indexed_load  --pattern stride_16B --wallclock --lmul 4 --vl 6
CPU = 3
LMUL = 4
Timing = wallclock
Kernel = paired
vl,stride_16B_difference_ns_per_vluxei,stride_16B_difference_ns_per_element
6,124.8970,20.8162

 Performance counter stats for 'taskset -c 3 ./indexed_load --pattern stride_16B --wallclock --lmul 4 --vl 6':

       500,557,039      task-clock:u                     #    0.996 CPUs utilized
                 0      context-switches:u               #    0.000 /sec
                 0      cpu-migrations:u                 #    0.000 /sec
               101      page-faults:u                    #  201.775 /sec
        10,496,874      instructions:u                   #    0.01  insn per cycle
                                                  #  103.88  stalled cycles per insn
     1,100,222,823      cycles:u                         #    2.198 GHz
     1,090,438,604      stalled-cycles-frontend:u        #   99.11% frontend cycles idle
       819,840,755      stalled-cycles-backend:u         #   74.52% backend cycles idle
         1,048,977      branches:u                       #    2.096 M/sec
             5,225      branch-misses:u                  #    0.50% of all branches
       172,097,891      L1-dcache-loads:u                #  343.813 M/sec
             5,018      L1-dcache-load-misses:u          #    0.00% of all L1-dcache accesses
       545,347,843      L1-icache-loads:u                #    1.089 G/sec
             3,135      L1-icache-load-misses:u          #    0.00% of all L1-icache accesses
       172,097,918      dTLB-loads:u                     #  343.813 M/sec
     1,093,101,459      iTLB-loads:u                     #    2.184 G/sec
               561      iTLB-load-misses:u               #    0.00% of all iTLB cache accesses

       0.502349232 seconds time elapsed

       0.501517000 seconds user
       0.000000000 seconds sys


shiyin@ai-spacemitk3picoitx:~/workspace/rvv-microbench/memory/indexed-vl$ perf stat -d -d taskset -c 3 ./indexed_load  --pattern stride_16B --wallclock --lmul 4 --vl 7
CPU = 3
LMUL = 4
Timing = wallclock
Kernel = paired
vl,stride_16B_difference_ns_per_vluxei,stride_16B_difference_ns_per_element
7,119.0554,17.0079

 Performance counter stats for 'taskset -c 3 ./indexed_load --pattern stride_16B --wallclock --lmul 4 --vl 7':

       477,162,790      task-clock:u                     #    0.996 CPUs utilized
                 0      context-switches:u               #    0.000 /sec
                 0      cpu-migrations:u                 #    0.000 /sec
               100      page-faults:u                    #  209.572 /sec
        10,471,911      instructions:u                   #    0.01  insn per cycle
                                                  #   99.17  stalled cycles per insn
     1,048,652,863      cycles:u                         #    2.198 GHz
     1,038,480,272      stalled-cycles-frontend:u        #   99.03% frontend cycles idle
       780,914,253      stalled-cycles-backend:u         #   74.47% backend cycles idle
         1,048,978      branches:u                       #    2.198 M/sec
             5,371      branch-misses:u                  #    0.51% of all branches
       168,100,077      L1-dcache-loads:u                #  352.291 M/sec
             5,052      L1-dcache-load-misses:u          #    0.00% of all L1-dcache accesses
       519,963,093      L1-icache-loads:u                #    1.090 G/sec
             3,337      L1-icache-load-misses:u          #    0.00% of all L1-icache accesses
       168,100,103      dTLB-loads:u                     #  352.291 M/sec
     1,041,137,035      iTLB-loads:u                     #    2.182 G/sec
               567      iTLB-load-misses:u               #    0.00% of all iTLB cache accesses

       0.478967349 seconds time elapsed

       0.478071000 seconds user
       0.000000000 seconds sys


shiyin@ai-spacemitk3picoitx:~/workspace/rvv-microbench/memory/indexed-vl$ perf stat -d -d taskset -c 3 ./indexed_load  --pattern stride_16B --wallclock --lmul 4 --vl 8
CPU = 3
LMUL = 4
Timing = wallclock
Kernel = paired
vl,stride_16B_difference_ns_per_vluxei,stride_16B_difference_ns_per_element
8,112.9807,14.1226

 Performance counter stats for 'taskset -c 3 ./indexed_load --pattern stride_16B --wallclock --lmul 4 --vl 8':

       453,570,914      task-clock:u                     #    0.996 CPUs utilized
                 0      context-switches:u               #    0.000 /sec
                 0      cpu-migrations:u                 #    0.000 /sec
               102      page-faults:u                    #  224.882 /sec
        10,401,689      instructions:u                   #    0.01  insn per cycle
                                                  #   94.84  stalled cycles per insn
       996,592,756      cycles:u                         #    2.197 GHz
       986,494,328      stalled-cycles-frontend:u        #   98.99% frontend cycles idle
       741,975,990      stalled-cycles-backend:u         #   74.45% backend cycles idle
         1,048,995      branches:u                       #    2.313 M/sec
             5,326      branch-misses:u                  #    0.51% of all branches
       164,099,116      L1-dcache-loads:u                #  361.794 M/sec
             5,141      L1-dcache-load-misses:u          #    0.00% of all L1-dcache accesses
       494,860,360      L1-icache-loads:u                #    1.091 G/sec
             3,264      L1-icache-load-misses:u          #    0.00% of all L1-icache accesses
       164,099,143      dTLB-loads:u                     #  361.794 M/sec
       989,149,949      iTLB-loads:u                     #    2.181 G/sec
               549      iTLB-load-misses:u               #    0.00% of all iTLB cache accesses

       0.455599131 seconds time elapsed

       0.450568000 seconds user
       0.003987000 seconds sys

# perf scalar
shiyin@ai-spacemitk3picoitx:~/workspace/rvv-microbench/memory/indexed-scalar$ perf stat -d -d taskset -c 3 ./scalar_load  --pattern stride_16B --wallclock --vl 1
CPU = 3
Timing = wallclock
Kernel = paired
vl,stride_16B_difference_ns_per_lwu
1,0.2273

 Performance counter stats for 'taskset -c 3 ./scalar_load --pattern stride_16B --wallclock --vl 1':

         4,520,041      task-clock:u                     #    0.834 CPUs utilized
                 0      context-switches:u               #    0.000 /sec
                 0      cpu-migrations:u                 #    0.000 /sec
               101      page-faults:u                    #   22.345 K/sec
        14,982,161      instructions:u                   #    1.53  insn per cycle
                                                  #    0.35  stalled cycles per insn
         9,817,335      cycles:u                         #    2.172 GHz
         2,195,356      stalled-cycles-frontend:u        #   22.36% frontend cycles idle
         5,189,057      stalled-cycles-backend:u         #   52.86% backend cycles idle
         2,050,186      branches:u                       #  453.577 M/sec
             5,089      branch-misses:u                  #    0.25% of all branches
         5,096,898      L1-dcache-loads:u                #    1.128 G/sec
             4,766      L1-dcache-load-misses:u          #    0.09% of all L1-dcache accesses
         4,548,895      L1-icache-loads:u                #    1.006 G/sec
             2,975      L1-icache-load-misses:u          #    0.07% of all L1-icache accesses
         5,096,927      dTLB-loads:u                     #    1.128 G/sec
         6,598,914      iTLB-loads:u                     #    1.460 G/sec
               468      iTLB-load-misses:u               #    0.01% of all iTLB cache accesses

       0.005419999 seconds time elapsed

       0.005535000 seconds user
       0.000000000 seconds sys


shiyin@ai-spacemitk3picoitx:~/workspace/rvv-microbench/memory/indexed-scalar$ perf stat -d -d taskset -c 3 ./scalar_load  --pattern stride_16B --wallclock --vl 2
CPU = 3
Timing = wallclock
Kernel = paired
vl,stride_16B_difference_ns_per_lwu
2,0.1737

 Performance counter stats for 'taskset -c 3 ./scalar_load --pattern stride_16B --wallclock --vl 2':

         7,219,083      task-clock:u                     #    0.886 CPUs utilized
                 0      context-switches:u               #    0.000 /sec
                 0      cpu-migrations:u                 #    0.000 /sec
               104      page-faults:u                    #   14.406 K/sec
        24,028,822      instructions:u                   #    1.53  insn per cycle
                                                  #    0.46  stalled cycles per insn
        15,717,399      cycles:u                         #    2.177 GHz
         1,473,152      stalled-cycles-frontend:u        #    9.37% frontend cycles idle
        11,046,891      stalled-cycles-backend:u         #   70.28% backend cycles idle
         3,050,184      branches:u                       #  422.517 M/sec
             5,107      branch-misses:u                  #    0.17% of all branches
        10,096,419      L1-dcache-loads:u                #    1.399 G/sec
             4,873      L1-dcache-load-misses:u          #    0.05% of all L1-dcache accesses
        11,064,008      L1-icache-loads:u                #    1.533 G/sec
             3,128      L1-icache-load-misses:u          #    0.03% of all L1-icache accesses
        10,096,443      dTLB-loads:u                     #    1.399 G/sec
        12,381,993      iTLB-loads:u                     #    1.715 G/sec
               444      iTLB-load-misses:u               #    0.00% of all iTLB cache accesses

       0.008150143 seconds time elapsed

       0.008239000 seconds user
       0.000000000 seconds sys


shiyin@ai-spacemitk3picoitx:~/workspace/rvv-microbench/memory/indexed-scalar$ perf stat -d -d taskset -c 3 ./scalar_load  --pattern stride_16B --wallclock --vl 3
CPU = 3
Timing = wallclock
Kernel = paired
vl,stride_16B_difference_ns_per_lwu
3,0.1263

 Performance counter stats for 'taskset -c 3 ./scalar_load --pattern stride_16B --wallclock --vl 3':

         9,127,958      task-clock:u                     #    0.907 CPUs utilized
                 0      context-switches:u               #    0.000 /sec
                 0      cpu-migrations:u                 #    0.000 /sec
               104      page-faults:u                    #   11.394 K/sec
        33,096,766      instructions:u                   #    1.66  insn per cycle
                                                  #    0.40  stalled cycles per insn
        19,913,279      cycles:u                         #    2.182 GHz
         2,780,281      stalled-cycles-frontend:u        #   13.96% frontend cycles idle
        13,358,581      stalled-cycles-backend:u         #   67.08% backend cycles idle
         4,050,194      branches:u                       #  443.713 M/sec
             5,148      branch-misses:u                  #    0.13% of all branches
        15,097,839      L1-dcache-loads:u                #    1.654 G/sec
             4,896      L1-dcache-load-misses:u          #    0.03% of all L1-dcache accesses
        14,222,395      L1-icache-loads:u                #    1.558 G/sec
             2,990      L1-icache-load-misses:u          #    0.02% of all L1-icache accesses
        15,097,870      dTLB-loads:u                     #    1.654 G/sec
        16,505,777      iTLB-loads:u                     #    1.808 G/sec
               479      iTLB-load-misses:u               #    0.00% of all iTLB cache accesses

       0.010068498 seconds time elapsed

       0.010146000 seconds user
       0.000000000 seconds sys


shiyin@ai-spacemitk3picoitx:~/workspace/rvv-microbench/memory/indexed-scalar$ perf stat -d -d taskset -c 3 ./scalar_load  --pattern stride_16B --wallclock --vl 4
CPU = 3
Timing = wallclock
Kernel = paired
vl,stride_16B_difference_ns_per_lwu
4,0.1282

 Performance counter stats for 'taskset -c 3 ./scalar_load --pattern stride_16B --wallclock --vl 4':

        11,139,542      task-clock:u                     #    0.923 CPUs utilized
                 0      context-switches:u               #    0.000 /sec
                 0      cpu-migrations:u                 #    0.000 /sec
               104      page-faults:u                    #    9.336 K/sec
        42,028,138      instructions:u                   #    1.72  insn per cycle
                                                  #    0.40  stalled cycles per insn
        24,387,754      cycles:u                         #    2.189 GHz
         3,861,171      stalled-cycles-frontend:u        #   15.83% frontend cycles idle
        16,713,586      stalled-cycles-backend:u         #   68.53% backend cycles idle
         5,050,190      branches:u                       #  453.357 M/sec
             5,041      branch-misses:u                  #    0.10% of all branches
        20,097,101      L1-dcache-loads:u                #    1.804 G/sec
             4,839      L1-dcache-load-misses:u          #    0.02% of all L1-dcache accesses
        18,379,879      L1-icache-loads:u                #    1.650 G/sec
             3,014      L1-icache-load-misses:u          #    0.02% of all L1-icache accesses
        20,097,130      dTLB-loads:u                     #    1.804 G/sec
        21,083,653      iTLB-loads:u                     #    1.893 G/sec
               452      iTLB-load-misses:u               #    0.00% of all iTLB cache accesses

       0.012067140 seconds time elapsed

       0.012225000 seconds user
       0.000000000 seconds sys


shiyin@ai-spacemitk3picoitx:~/workspace/rvv-microbench/memory/indexed-scalar$ perf stat -d -d taskset -c 3 ./scalar_load  --pattern stride_16B --wallclock --vl 5
CPU = 3
Timing = wallclock
Kernel = paired
vl,stride_16B_difference_ns_per_lwu
5,0.2272

 Performance counter stats for 'taskset -c 3 ./scalar_load --pattern stride_16B --wallclock --vl 5':

        13,417,416      task-clock:u                     #    0.912 CPUs utilized
                 0      context-switches:u               #    0.000 /sec
                 0      cpu-migrations:u                 #    0.000 /sec
               100      page-faults:u                    #    7.453 K/sec
        51,278,928      instructions:u                   #    1.77  insn per cycle
                                                  #    0.39  stalled cycles per insn
        29,044,239      cycles:u                         #    2.165 GHz
         5,124,159      stalled-cycles-frontend:u        #   17.64% frontend cycles idle
        19,952,298      stalled-cycles-backend:u         #   68.70% backend cycles idle
         6,050,190      branches:u                       #  450.921 M/sec
             5,236      branch-misses:u                  #    0.09% of all branches
        25,097,754      L1-dcache-loads:u                #    1.871 G/sec
             4,844      L1-dcache-load-misses:u          #    0.02% of all L1-dcache accesses
        20,870,129      L1-icache-loads:u                #    1.555 G/sec
             3,105      L1-icache-load-misses:u          #    0.01% of all L1-icache accesses
        25,097,775      dTLB-loads:u                     #    1.871 G/sec
        25,358,286      iTLB-loads:u                     #    1.890 G/sec
               532      iTLB-load-misses:u               #    0.00% of all iTLB cache accesses

       0.014718956 seconds time elapsed

       0.014404000 seconds user
       0.000000000 seconds sys


shiyin@ai-spacemitk3picoitx:~/workspace/rvv-microbench/memory/indexed-scalar$ perf stat -d -d taskset -c 3 ./scalar_load  --pattern stride_16B --wallclock --vl 6
CPU = 3
Timing = wallclock
Kernel = paired
vl,stride_16B_difference_ns_per_lwu
6,0.1989

 Performance counter stats for 'taskset -c 3 ./scalar_load --pattern stride_16B --wallclock --vl 6':

        13,864,166      task-clock:u                     #    0.938 CPUs utilized
                 0      context-switches:u               #    0.000 /sec
                 0      cpu-migrations:u                 #    0.000 /sec
               103      page-faults:u                    #    7.429 K/sec
        60,029,310      instructions:u                   #    1.98  insn per cycle
                                                  #    0.34  stalled cycles per insn
        30,379,454      cycles:u                         #    2.191 GHz
         4,456,994      stalled-cycles-frontend:u        #   14.67% frontend cycles idle
        20,167,163      stalled-cycles-backend:u         #   66.38% backend cycles idle
         7,050,198      branches:u                       #  508.519 M/sec
             5,022      branch-misses:u                  #    0.07% of all branches
        30,097,114      L1-dcache-loads:u                #    2.171 G/sec
             4,815      L1-dcache-load-misses:u          #    0.02% of all L1-dcache accesses
        22,797,861      L1-icache-loads:u                #    1.644 G/sec
             2,954      L1-icache-load-misses:u          #    0.01% of all L1-icache accesses
        30,097,138      dTLB-loads:u                     #    2.171 G/sec
        27,102,099      iTLB-loads:u                     #    1.955 G/sec
               482      iTLB-load-misses:u               #    0.00% of all iTLB cache accesses

       0.014781618 seconds time elapsed

       0.011187000 seconds user
       0.003729000 seconds sys


shiyin@ai-spacemitk3picoitx:~/workspace/rvv-microbench/memory/indexed-scalar$ perf stat -d -d taskset -c 3 ./scalar_load  --pattern stride_16B --wallclock --vl 7
CPU = 3
Timing = wallclock
Kernel = paired
vl,stride_16B_difference_ns_per_lwu
7,0.2209

 Performance counter stats for 'taskset -c 3 ./scalar_load --pattern stride_16B --wallclock --vl 7':

        16,159,709      task-clock:u                     #    0.935 CPUs utilized
                 0      context-switches:u               #    0.000 /sec
                 0      cpu-migrations:u                 #    0.000 /sec
               101      page-faults:u                    #    6.250 K/sec
        69,012,127      instructions:u                   #    1.95  insn per cycle
                                                  #    0.36  stalled cycles per insn
        35,386,042      cycles:u                         #    2.190 GHz
         6,188,422      stalled-cycles-frontend:u        #   17.49% frontend cycles idle
        24,945,111      stalled-cycles-backend:u         #   70.49% backend cycles idle
         8,050,210      branches:u                       #  498.166 M/sec
             5,343      branch-misses:u                  #    0.07% of all branches
        35,098,556      L1-dcache-loads:u                #    2.172 G/sec
             4,796      L1-dcache-load-misses:u          #    0.01% of all L1-dcache accesses
        27,048,532      L1-icache-loads:u                #    1.674 G/sec
             2,994      L1-icache-load-misses:u          #    0.01% of all L1-icache accesses
        35,098,578      dTLB-loads:u                     #    2.172 G/sec
        32,096,972      iTLB-loads:u                     #    1.986 G/sec
               518      iTLB-load-misses:u               #    0.00% of all iTLB cache accesses

       0.017283150 seconds time elapsed

       0.017164000 seconds user
       0.000000000 seconds sys


shiyin@ai-spacemitk3picoitx:~/workspace/rvv-microbench/memory/indexed-scalar$ perf stat -d -d taskset -c 3 ./scalar_load  --pattern stride_16B --wallclock --vl 8
CPU = 3
Timing = wallclock
Kernel = paired
vl,stride_16B_difference_ns_per_lwu
8,0.2227

 Performance counter stats for 'taskset -c 3 ./scalar_load --pattern stride_16B --wallclock --vl 8':

        17,967,833      task-clock:u                     #    0.952 CPUs utilized
                 0      context-switches:u               #    0.000 /sec
                 0      cpu-migrations:u                 #    0.000 /sec
               102      page-faults:u                    #    5.677 K/sec
        78,030,190      instructions:u                   #    1.98  insn per cycle
                                                  #    0.36  stalled cycles per insn
        39,398,095      cycles:u                         #    2.193 GHz
         7,450,069      stalled-cycles-frontend:u        #   18.91% frontend cycles idle
        27,819,596      stalled-cycles-backend:u         #   70.61% backend cycles idle
         9,050,208      branches:u                       #  503.689 M/sec
             5,511      branch-misses:u                  #    0.06% of all branches
        40,098,561      L1-dcache-loads:u                #    2.232 G/sec
             4,874      L1-dcache-load-misses:u          #    0.01% of all L1-dcache accesses
        29,665,539      L1-icache-loads:u                #    1.651 G/sec
             3,034      L1-icache-load-misses:u          #    0.01% of all L1-icache accesses
        40,098,584      dTLB-loads:u                     #    2.232 G/sec
        36,107,432      iTLB-loads:u                     #    2.010 G/sec
               458      iTLB-load-misses:u               #    0.00% of all iTLB cache accesses

       0.018872148 seconds time elapsed

       0.015217000 seconds user
       0.003804000 seconds sys
```

</details>

## Observations

- Access pattern has little effect when all the data are placed in L1D cache

- Indexed-load latency is approximately linear in the tail length

The surprising result is that `cycles/vluxei` decreases as VL increases. It
does not scale with the number of active elements. Instead, the measurements could be described by
the empirical model

```text
Cycle(VL) = Constant + k * (VLMAX - VL)
```

where `k` is approximately 13.05 cycles for all measured LMUL values. Using
the `stride_16B` endpoints gives:

- The scalar version (around 0.5 elements/cycle) outperforms around 2 times than the vector version (around 1.2 elements/cycle with LMUL=4 VL=VLMAX). 

I suppose the scalar version's data (0.5 elemtns/cycle ) is because X100 has 2 LSU pipeline according to their paper.

## Build and run

The benchmark is intended to be compiled and run natively on a RISC-V Linux
machine with the vector extension:

```sh
make
./indexed_load > result.csv
```

The default iteration count is 100000, the default repeat count is 5, and the
default LMUL is 4. They and the maximum VL can be changed at runtime:

```sh
./indexed_load --iterations 200000 --repeats 7 --max-vl 32 > result.csv
```

Use `--cycel` for the internal perf-event CPU-cycle counter or `--wallclock`
for `clock_gettime` nanoseconds. They are mutually exclusive; `--cycel` is the
default. The correctly spelled `--cycle` is also accepted as an alias.

Use `--lmul` to select LMUL 1, 2, 4, or 8. Without `--vl` or `--max-vl`, the
program runs every VL from 1 through VLMAX for that LMUL:

```sh
./indexed_load --lmul 2 --pattern contiguous
```

Use `--vl` to run exactly one VL, which is useful for wrapping a single test
point with `perf stat`:

```sh
./indexed_load --vl 16 --pattern contiguous --iterations 200000 --repeats 7
```

When using an outer `perf stat`, select `--wallclock` so the program does not
open an internal hardware event. `--kernel baseline` and `--kernel target`
run the two sides separately; `--kernel paired` remains the default and emits
only their normalized difference. For example:

```sh
perf stat -e L1-dcache-loads:u,L1-dcache-load-misses:u \
  taskset -c 3 ./indexed_load --wallclock --kernel baseline \
  --lmul 2 --vl 4 --pattern contiguous --repeats 1

perf stat -e L1-dcache-loads:u,L1-dcache-load-misses:u \
  taskset -c 3 ./indexed_load --wallclock --kernel target \
  --lmul 2 --vl 4 --pattern contiguous --repeats 1
```

Subtract the first external perf count from the second to obtain the target
kernel's baseline-adjusted L1 event count. Use identical arguments for both
commands.

When `--vl` is used without `--lmul`, LMUL remains at its default value of 4.
When they are used together, the specified VL is checked against VLMAX for the
selected LMUL:

```sh
./indexed_load --lmul 1 --vl 8 --pattern contiguous
```

`--vl` and `--max-vl` are mutually exclusive. Both are checked against VLMAX
for the selected LMUL. `--max-vl` remains available for running a prefix of the
full `1..VLMAX` range.

Pass `--pattern` to run only one pattern and emit only that CSV column:

```sh
./indexed_load --pattern contiguous > contiguous.csv
./indexed_load --pattern random_in_page --iterations 200000 --repeats 7 > random.csv
```

Valid names are `contiguous`, `stride_16B`, `cacheline_64B`, and
`random_in_page`. Run `./indexed_load --help` for the option summary.

The same selection can be passed through the Makefile run target:

```sh
make run RUN_ARGS="--pattern contiguous --iterations 200000 --repeats 7"
```

The scalar comparison is under the sibling directory `../indexed-scalar/`.
Build it from this directory with `make scalar`, or run it with:

```sh
make scalar-run RUN_ARGS="--vl 16 --pattern contiguous --iterations 200000 --repeats 7"
```

With `--vl 16 --pattern contiguous`, the output contains both normalized
metrics for that test point:

```text
CPU = 3
LMUL = 4
Timing = perf_event cycles
Kernel = paired
vl,contiguous_difference_cycles_per_vluxei,contiguous_difference_cycles_per_element
16,...,...
```

The first metric divides by the `iterations * 8` dynamically executed vector
indexed-load instructions. The second divides once more by VL. The perf event
excludes kernel and hypervisor cycles and scales the count if the event was
multiplexed. Measurement noise can occasionally produce a small negative
baseline-subtracted result; it is not clamped.

## Reference environment

- Compiler: Ubuntu clang 21.1.8 (6ubuntu1)
- Target: `riscv64-unknown-linux-gnu`
- Flags: `-O2 -march=rv64gcv`
- Machine: K3
  - SpacemiT A100: VLEN 256, DLEN 256
  - SpacemiT X100: VLEN 1024, DLEN 1024 (to be confirmed)
- Cache:
  - L1d: 1 MiB total (16 × 64 KiB) 9 (assume 64B cahce line)
  - L1i: 1 MiB total (16 × 64 KiB)
  - L2: 10 MiB total (4 × 2560 KiB)
