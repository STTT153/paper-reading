# Report

Goal: compare the Vector performance with Scalar cores. Focus on gather instruction.

## Machine Infomation
- Shuttle: Dual issue, in-order pipeline, support Saturn integration
- BOOM:    Single issue, OoO pipeline, doesn't support Saturn integration (adopt a naive integration)
- Saturn:  VLEN=256, DLEN=128, Sequencer Topology is equivlent to GENV256D128ShuttleConfig

## BOOM-Saturn Integration

| Category   | Benchmark          | Elements | Repeats | Scalar cycles | Vector cycles |    Speedup | Scalar instructions | Vector instructions |
| ---------- | ------------------ | -------: | ------: | ------------: | ------------: | ---------: | ------------------: | ------------------: |
| memory     | `unit_stride_copy` |    8,192 |      16 |       973,277 |       714,129 | **1.363×** |             786,552 |              49,432 |
| memory     | `indexed_gather`   |    8,192 |      16 |     1,510,961 |     2,197,059 | **0.688×** |           1,179,783 |              90,407 | 
| memory     | `indexed_scatter`  |    8,192 |      16 |     1,619,879 |     2,156,104 | **0.751×** |           1,179,783 |              90,407 | 
| arithmetic | `i32_mul_add_xor`  |    8,192 |      16 |     2,262,964 |     2,123,610 | **1.066×** |           1,704,136 |              98,696 | 
| arithmetic | `fp32_saxpy`       |    8,192 |      16 |     1,831,452 |     1,312,635 | **1.395×** |           1,179,800 |              74,072 | 
| arithmetic | `fp32_dot_reduce`  |    8,192 |      16 |     1,301,096 |       978,954 | **1.329×** |             917,639 |              57,703 |
| control    | `masked_select`    |    8,192 |      16 |     2,307,882 |     2,066,381 | **1.117×** |           1,443,048 |              98,632 |
| mixed      | `stencil3`         |    8,190 |      16 |     1,423,428 |     2,897,405 | **0.491×** |           1,310,601 |             134,329 |

Vector doesn't perform well because of the naive integrating approach: The vector instruction waits until the ROB and scalar LSU are drained. Younger instructions remain blocked while the vector transaction is active. Saturn drains its backend and outstanding TileLink responses before it reports completion. BOOM then commits the instruction and refetches younger instructions so they observe the new vector CSR state.

## Standalone BOOM Scalar Run as a Reference
| Category   | Benchmark          | Elements | Repeats |    Cycles | Instructions | Cycles/element |   IPC |
| ---------- | ------------------ | -------: | ------: | --------: | -----------: | -------------: | ----: |
| memory     | `unit_stride_copy` |    8,192 |      16 |   969,880 |      786,552 |          7.399 | 0.811 |
| memory     | `indexed_gather`   |    8,192 |      16 | 1,517,962 |    1,179,783 |         11.581 | 0.777 |
| memory     | `indexed_scatter`  |    8,192 |      16 | 1,621,439 |    1,179,783 |         12.371 | 0.728 |
| arithmetic | `i32_mul_add_xor`  |    8,192 |      16 | 2,265,021 |    1,704,136 |         17.281 | 0.752 |
| arithmetic | `fp32_saxpy`       |    8,192 |      16 | 1,835,884 |    1,179,800 |         14.007 | 0.643 |
| arithmetic | `fp32_dot_reduce`  |    8,192 |      16 | 1,300,878 |      917,639 |          9.925 | 0.705 |
| control    | `masked_select`    |    8,192 |      16 | 2,314,904 |    1,443,048 |         17.661 | 0.623 |
| mixed      | `stencil3`         |    8,190 |      16 | 1,425,129 |    1,310,601 |         10.878 | 0.920 |

This shows that Saturn Integration doesn't affect the Scalar core performance.

## Shuttle-Saturn Integration as a Reference
| Category   | Benchmark          | Elements | Repeats | Scalar cycles | Vector cycles |     Speedup | Scalar instructions | Vector instructions | Scalar IPC | Vector IPC |
| ---------- | ------------------ | -------: | ------: | ------------: | ------------: | ----------: | ------------------: | ------------------: | ---------: | ---------: |
| memory     | `unit_stride_copy` |    8,192 |      16 |     1,287,107 |        66,740 | **19.285×** |             786,551 |              49,431 |      0.611 |      0.741 |
| memory     | `indexed_gather`   |    8,192 |      16 |     2,271,206 |       925,810 |  **2.453×** |           1,179,782 |              90,406 |      0.519 |      0.098 |
| memory     | `indexed_scatter`  |    8,192 |      16 |     1,467,716 |       700,432 |  **2.095×** |           1,179,782 |              90,406 |      0.804 |      0.129 |
| arithmetic | `i32_mul_add_xor`  |    8,192 |      16 |     2,292,505 |       147,817 | **15.509×** |           1,704,135 |              98,695 |      0.743 |      0.668 |
| arithmetic | `fp32_saxpy`       |    8,192 |      16 |           N/A |       104,900 |         N/A |                 N/A |              74,071 |        N/A |      0.706 |
| arithmetic | `fp32_dot_reduce`  |    8,192 |      16 |           N/A |        90,722 |         N/A |                 N/A |              57,701 |        N/A |      0.636 |
| control    | `masked_select`    |    8,192 |      16 |     2,093,765 |       164,251 | **12.747×** |           1,443,047 |              98,631 |      0.689 |      0.600 |
| mixed      | `stencil3`         |    8,190 |      16 |     1,385,364 |       281,899 |  **4.914×** |           1,310,600 |             134,328 |      0.946 |      0.477 |

Shuttle-Saturn performance is much more better. 

Shuttle doesn't support FP32 operation, so `fp32_saxpy`, `fp32_dot_reduce` are skipped.

## Standalone BOOM Scalar V.S. Shuttle-Saturn
| Category   | Benchmark          | Elements | Repeats | BOOM scalar cycles | Shuttle–Saturn vector cycles |     Speedup | BOOM scalar instructions | Shuttle–Saturn vector instructions | BOOM scalar IPC | Shuttle–Saturn vector IPC |
| ---------- | ------------------ | -------: | ------: | -----------------: | ---------------------------: | ----------: | -----------------------: | ---------------------------------: | --------------: | ------------------------: |
| memory     | `unit_stride_copy` |    8,192 |      16 |            969,880 |                       66,740 | **14.532×** |                  786,552 |                             49,431 |           0.811 |                     0.741 |
| memory     | `indexed_gather`   |    8,192 |      16 |          1,517,962 |                      925,810 |  **1.640×** |                1,179,783 |                             90,406 |           0.777 |                     0.098 |
| memory     | `indexed_scatter`  |    8,192 |      16 |          1,621,439 |                      700,432 |  **2.315×** |                1,179,783 |                             90,406 |           0.728 |                     0.129 |
| arithmetic | `i32_mul_add_xor`  |    8,192 |      16 |          2,265,021 |                      147,817 | **15.323×** |                1,704,136 |                             98,695 |           0.752 |                     0.668 |
| arithmetic | `fp32_saxpy`       |    8,192 |      16 |          1,835,884 |                      104,900 | **17.501×** |                1,179,800 |                             74,071 |           0.643 |                     0.706 |
| arithmetic | `fp32_dot_reduce`  |    8,192 |      16 |          1,300,878 |                       90,722 | **14.339×** |                  917,639 |                             57,701 |           0.705 |                     0.636 |
| control    | `masked_select`    |    8,192 |      16 |          2,314,904 |                      164,251 | **14.094×** |                1,443,048 |                             98,631 |           0.623 |                     0.600 |
| mixed      | `stencil3`         |    8,190 |      16 |          1,425,129 |                      281,899 |  **5.055×** |                1,310,601 |                            134,328 |           0.920 |                     0.477 |

This compares BOOM Scalar performance with Shuttle integrated Saturn.

### Theoretical Analysis
For simple benchmarks like `unit_stride_copy`, `fp32_saxpy`, `fp32_dot_reduce`, vector could achieve around 16x spead up. Take `fp32_saxpy` as an example.

```c
void kernel_saxpy(float *restrict dst, const float *restrict x,
                  const float *restrict y, float alpha, size_t n) {
  for (size_t i = 0; i < n; ++i)
    dst[i] = alpha * x[i] + y[i];
}
```

```asm
	neg	a4, a7
	and	a6, a4, a3
	slli	t0, t0, 1
	vsetvli	a4, zero, e32, m2, ta, ma
	mv	t1, a6
	mv	t2, a0
	mv	a5, a2
	mv	a4, a1
.LBB5_4:                                # Vector Loop Body, tail handling 
                                        # is done using scalar, omitted here for brevity.
	vl2re32.v	v8, (a4)				# x
	vl2re32.v	v10, (a5)				# y
	vfmacc.vf	v10, fa0, v8			# y[i] = y[i] + a * x[i]
	vs2r.v	v10, (t2)					# t2 = dst
	add	a4, a4, t0
	add	a5, a5, t0
	sub	t1, t1, a7
	add	t2, t2, t0
	bnez	t1, .LBB5_4
```

```asm
	beqz	a3, .LBB5_2
.LBB5_1:                                # Scalar loop body
	flw	fa5, 0(a1)
	flw	fa4, 0(a2)
	fmadd.s	fa5, fa5, fa0, fa4
	fsw	fa5, 0(a0)
	addi	a3, a3, -1
	addi	a0, a0, 4
	addi	a2, a2, 4
	addi	a1, a1, 4
	bnez	a3, .LBB5_1
.LBB5_2:
	ret
```

For vector, after first `vl2re32.v` finish, `vl2re32.v -> vfmacc.vf -> vs2r.v` can execute and chain to each other. In steady state, we can suppose hardware process DLEN(128 bit) every 2 cycles. 64 bit per cycle.

For scalar, because BOOM is OoO and single issue, assume memory operation's latency can be perfectly hidden, this yields 1 IPC, and need 9 instruction to process 1 FP32 element, (32 bit) data. Around 4 bit per cycle.

From another perspective, Scalar and Vector have similar IPC (0.643 and 0.706). Each iteration, vector process 512 bit of data (LMUL=2), while scalar process 32 bit of data. This also yields 16x speed up.


