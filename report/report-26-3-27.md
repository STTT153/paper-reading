# Report 26-3-27

## 1 Test on mask vector chaining
Wheter mask genenrating instructions will chain to otehr mask consuming insturcionts (int/float/mem)

### 1.1 Integer
Experiment Setup: VLEN=1024, DLEN=128, SEW=8, LMUL=4, **MultiALU(two integer sequencers)**

```
  vsetvli t5, t0, ELEM, LMUL_STR, ta, ma

  # Load source data
  vle8.v V_SRC1, (t2)
  vle8.v V_SRC2, (t3)

  # Generate mask: src1 == TEST_VAL
  vmseq.vi V_MASK, V_SRC1, TEST_VAL

  # Integer add with generated mask (masked operation)
  vadd.vv V_DEST, V_SRC1, V_SRC2, V_MASK.t

  # Store result
  vse8.v V_DEST, (t1)
```

### 1.2 Floating Point
Experiment Setup: VLEN=1024, DLEN=128, SEW=32, LMUL=8, MultiALU(two integer sequencers)

VLEN/DLEN * LMUL = 64

```
  # Pre-load compare value and src2 BEFORE mask generation
  LOAD_ELEM V_FP_SRC2, (t4)     
  LOAD_ELEM V_FP_TMP, (t3)

  # Load src1 for mask compare and FP operation
  LOAD_ELEM V_FP_SRC1, (t2)

  # Generate mask: V_FP_SRC1 == V_FP_SRC2 (compare value)
  vmseq.vx v0, V_FP_SRC1, t4

  # FP add with chained mask 
  vfadd.vv V_FP_DEST, V_FP_SRC1, V_FP_TMP, v0.t

  STORE_ELEM V_FP_DEST, (t1)
```

### 1.2 Memory Access
Experiment Setup: VLEN=1024, DLEN=128, SEW=8, LMUL=4, MultiALU(two integer sequencers)

```
  vsetvli t5, t0, ELEM, LMUL_STR, ta, ma

  # Load source data for comparison
  vle8.v V_SRC, (t2)

  # Generate mask: src == TEST_VAL
  vmseq.vi V_MASK, V_SRC, TEST_VAL

  # Masked load from src into dest (only where mask is set)
  vle8.v V_DEST, (t2), V_MASK.t
```


## 2 Test the performance impact of VLEN
Experiment setup: DLEN=128, VLEN varied, LMUL=4

Didn't apply pre-allocate, cache miss.

Benchmark chain:

```
  # FMA chain + additional vector op
  vmul.vv V_DEST, V_SRC1, V_SRC2    # v8 = v4 * v8
  vadd.vv V_DEST, V_DEST, V_SRC3    # v8 = v8 + v12
  vand.vv V_DEST, V_DEST, V_SRC4    # v8 = v8 & v0
```

---

| VLEN | 2048 | 1024 | 512 | 256 | 128 |
|------|------|------|-----|-----|-----|
| Cycle | 107 | 227 | 293 | 384 | 468 |

The results align with our expectation. However I'm not sure whether port conflicts affact chaining.

## 3 Benchmark on OpenClaw workflow

Inference Time (API Call) Dominates.

| Description | Total time (ms) | Tool(ms) | Inference(ms) | inference proportion |
|----------|------------|--------------|--------------|----------|
| simple_reply | 590.0 | 0.0 | 590.0 | 100.0% |
| file_operation | 990.0 | 0.6 | 989.4 | 99.9% |
| command_execution | 810.0 | 9.0 | 801.0 | 98.9% |
| json_processing | 980.0 | 0.3 | 979.7 | 100.0% |
| mixed_workflow | 970.0 | 4.3 | 965.7 | 99.6% |

### simple_reply
- **消息**: `echo test`
- **描述**: 简单回复，测量纯API推理时间
- **类别**: baseline

### file_operation
- **消息**: `创建文件 /tmp/openclaw_test/test.txt，内容为'Hello World'`
- **描述**: 文件创建操作，测量推理+文件I/O时间
- **类别**: tool_execution

### command_execution
- **消息**: `运行命令: ls -la /tmp/openclaw_test`
- **描述**: 命令执行，测量推理+进程启动时间
- **类别**: tool_execution

### json_processing
- **消息**: `创建包含100个用户数据的JSON文件 /tmp/openclaw_test/data.json`
- **描述**: JSON数据处理，测量推理+数据处理时间
- **类别**: tool_execution

### mixed_workflow
- **消息**: `创建10个测试文件在/tmp/openclaw_test/files/，统计文件数量，然后删除`
- **描述**: 混合工作流，测量完整任务执行时间
- **类别**: complex
