# Chipyard PPA 获取流程调查

日期：2026-09-14

## 1. 调查目标与结论

本文调查如何从当前 Chipyard 设计获得功耗（Power）、性能
（Performance）和面积（Area）数据。

Chipyard 的标准 ASIC 实现入口是 `vlsi/` 目录中的 Hammer 流程。根据结果成熟度，PPA
数据可分为两档：

1. **综合后估算**：运行 `make syn`，快速获得标准单元面积、门数和初步时序；
2. **布局布线后结果**：运行 `make par`，利用实际布局、时钟树、布线和寄生参数获得更可信的面积与时序，再结合门级仿真的信号活动率进行功耗分析。

因此，用于设计比较的 PPA 应优先采用 post-P&R 结果。仅有 Chisel/RTL 或 RTL
仿真不能产生可比较、可复现的 PPA；还必须固定工艺库、PVT corner、时钟约束、
floorplan、SRAM 实现方式和测试 workload。

当前仓库自带 Sky130 + Yosys + OpenROAD 示例，可以用于开源的综合和布局布线调查。
但是集成的功耗 Makefile 使用 Cadence Joules/Voltus，并依赖 Synopsys VCS 生成门级
活动信息，因此开源教程不能直接通过现有 `power-par` target 取得完整功耗结果。

## 2. PPA 指标口径

### 2.1 Power

至少应记录：

- internal power；
- switching power；
- leakage power；
- total power；
- 对应电压、温度和工艺 corner；
- 产生信号活动率的 workload 与采样时间窗。

不带 VCD/SAIF 等真实活动率的静态功耗估算适合早期筛选，但不宜与
workload-driven power 混为同一组数据。

### 2.2 Performance

电路实现层面的 performance 通常指关键路径延迟或最高频率 `Fmax`。应至少记录：

- 目标时钟周期；
- setup WNS/TNS；
- hold violation；
- 关键路径；
- post-P&R `Fmax`。

若目标时钟周期为 `Tconstraint`、setup WNS 为 `WNS`，可以用下式作一次运行后的
粗略估计：

```text
Tmin(ns)  ~= Tconstraint(ns) - WNS(ns)
Fmax(MHz) ~= 1000 / Tmin(ns)
```

`WNS >= 0` 只表示当前约束通过，并不等于已经找到真正的 `Fmax`。更可靠的方法是逐步
收紧 clock period 并重新运行 P&R，找到能够完成 timing closure 的边界。

对于 CPU/SoC，电路频率不足以描述应用性能，还应在相同 workload 下记录 IPC、执行
周期数或运行时间。例如：

```text
execution_time = benchmark_cycles / clock_frequency
```

### 2.3 Area

至少应区分：

- standard-cell area；
- macro/SRAM area；
- core area；
- die area；
- placement utilization。

仅比较 standard-cell area 会遗漏 SRAM；仅比较 floorplan 的宽乘高，又可能掩盖不同
utilization。因此报告中需要明确采用哪一种 area 口径。

## 3. Chipyard/Hammer 流程

### 3.1 初始化

从 Chipyard 根目录执行：

```bash
cd /opt/chipyard
source env.sh
./scripts/init-vlsi.sh sky130
```

对于 Hammer 内置的开源工艺插件，`init-vlsi.sh` 支持 `sky130` 和 `asap7`。使用
NDA 工艺时，需要准备对应的 Hammer technology plugin、标准单元库、SRAM compiler
或 macro collateral，以及与工艺版本匹配的 EDA 工具。

### 3.2 配置工艺、工具和设计约束

Sky130 示例工艺配置为：

- [`vlsi/example-sky130.yml`](../vlsi/example-sky130.yml)

其中下列占位路径必须替换为本机的实际路径：

```yaml
technology.sky130:
  sky130A: "/path/to/sky130A"
  sram22_sky130_macros: "/path/to/sram22_sky130_macros"
```

开源工具选择位于：

- [`vlsi/example-openroad.yml`](../vlsi/example-openroad.yml)

该文件选择 Yosys 进行综合、OpenROAD 进行布局布线、KLayout/Magic 进行 DRC、
Netgen 进行 LVS。如果工具不在 `PATH`，需要在配置中补充相应 binary 的绝对路径。

设计级时钟和 floorplan 示例位于：

- [`vlsi/example-designs/sky130-openroad.yml`](../vlsi/example-designs/sky130-openroad.yml)

该教程为了容易完成 timing closure，将 `clock_uncore` 设置为 `50ns`，不能把这个
示例值直接当作待测系统的性能目标。设计评估前应设置真实的时钟周期、clock
uncertainty、顶层尺寸、macro placement、pin assignment 和电源网络。

### 3.3 运行自带 Sky130/OpenROAD 示例

```bash
cd /opt/chipyard/vlsi

make buildfile tutorial=sky130-openroad
make syn       tutorial=sky130-openroad
make par       tutorial=sky130-openroad
```

教程变量默认选择 `TinyRocketConfig`、Sky130、Yosys、OpenROAD，并把输出放在
`build-sky130-openroad` 下。`buildfile` 会生成适用于 VLSI 的 Verilog、处理 SRAM
mapping，并生成 Hammer build targets。

### 3.4 运行自定义 Chipyard 配置

假设配置名为 `MySoCConfig`，设计约束放在 `my-design.yml`：

```bash
cd /opt/chipyard/vlsi

make buildfile \
  CONFIG=MySoCConfig \
  tech_name=sky130 \
  INPUT_CONFS="example-openroad.yml example-sky130.yml my-design.yml"

make syn \
  CONFIG=MySoCConfig \
  tech_name=sky130 \
  INPUT_CONFS="example-openroad.yml example-sky130.yml my-design.yml"

make par \
  CONFIG=MySoCConfig \
  tech_name=sky130 \
  INPUT_CONFS="example-openroad.yml example-sky130.yml my-design.yml"
```

评估单个模块时，可以增加 `VLSI_TOP`。例如只综合 Rocket：

```bash
make syn \
  CONFIG=MySoCConfig \
  VLSI_TOP=Rocket \
  tech_name=sky130 \
  INPUT_CONFS="example-openroad.yml example-sky130.yml my-design.yml"
```

此时 `my-design.yml` 中 floorplan constraint 的顶层 `path` 也必须与
`VLSI_TOP` 一致。

## 4. 报告位置与数据提取

综合后的 log 和 collateral 位于对应设计的：

```text
.../syn-rundir/
.../syn-rundir/reports/
```

Chipyard 文档建议在进入 P&R 前检查综合产生的 `final-qor.rpt`，确认时序是否满足。

布局布线后的数据库、GDS 和报告位于：

```text
.../par-rundir/
.../par-rundir/timingReports/
```

不同 Hammer/tool plugin 产生的具体文件名可能不同，可以统一查找：

```bash
cd /opt/chipyard/vlsi

find build-sky130-openroad -type f \
  \( -iname '*.rpt' -o -iname '*report*' -o -iname '*.log' \) |
  sort
```

从报告和 log 中搜索面积与时序关键字：

```bash
rg -i "total.*area|design area|cell area|utilization" \
  build-sky130-openroad

rg -i "worst.*slack|wns|tns|critical path|setup slack" \
  build-sky130-openroad
```

OpenROAD 最终数据库可通过 Hammer 生成的 `open_chip` 脚本打开。典型操作为：

```bash
cd <design-output>/par-rundir
./generated-scripts/open_chip
```

## 5. 功耗分析路径

Chipyard 文档给出的 post-P&R、simulation-extracted power 命令形式为：

```bash
make power-par \
  BINARY=/path/to/workload.riscv \
  CONFIG=MySoCConfig \
  tech_name=<technology> \
  INPUT_CONFS="<tool.yml> <technology.yml> <design.yml>"
```

该流程先运行 post-P&R gate-level simulation，生成带时间窗的信号活动数据，再调用
功耗工具。当前 [`vlsi/power.mk`](../vlsi/power.mk) 中的工具选择为：

| 分析阶段 | 工具 |
| --- | --- |
| RTL power | Cadence Joules |
| post-synthesis power | Cadence Joules |
| post-P&R power | Cadence Voltus |
| gate-level activity simulation | Synopsys VCS |

因此有两条可行路径：

1. 配置 Cadence/Synopsys 商业工具和 license，沿用现有 `power-par` target；
2. 继续使用 OpenROAD，但补充 VCD/SAIF 导入与 `report_power` 脚本或对应 Hammer
   power plugin。

第二条路径需要额外实现，不能假设 `make power-par tutorial=sky130-openroad` 在当前
仓库中可以直接工作。

## 6. 可复现的 PPA 结果模板

建议每个设计点采用以下统一格式：

```text
Design/config:
RTL revision:
Technology/library:
PVT corner:
Supply voltage:
Temperature:
EDA tool and version:

Clock constraint (ns):
Post-P&R WNS/TNS (ns):
Hold status:
Estimated/closed Fmax (MHz):

Standard-cell area (um^2):
Macro/SRAM area (um^2):
Core area (mm^2):
Die area (mm^2):
Utilization (%):

Workload:
Activity source and window:
Internal power (mW):
Switching power (mW):
Leakage power (mW):
Total power (mW):

Benchmark cycles/IPC/runtime:
DRC/LVS status:
```

比较多个设计点时，必须保持 technology/library、PVT、clock uncertainty、
floorplan policy、utilization target、SRAM mapping、workload 和 power sampling window
一致，否则 PPA 差异可能主要来自实验条件而非微架构本身。

## 7. 当前工作区状态

调查时当前 `/opt/chipyard` 工作区尚不能直接运行上述开源 P&R：

- `vlsi/hammer` 尚未初始化；
- 当前 shell 的 `PATH` 中未找到 `yosys`；
- 当前 shell 的 `PATH` 中未找到 `openroad`；
- `vlsi/example-sky130.yml` 中的 Sky130 PDK 和 SRAM 路径仍为占位符。

因此下一步需要先安装或定位 Yosys/OpenROAD、准备 Sky130 PDK 与 SRAM collateral，
填写绝对路径并执行 `./scripts/init-vlsi.sh sky130`。这些条件满足后，先运行
`TinyRocketConfig` 教程验证工具链，再切换到目标系统配置，可以更容易区分环境问题与
设计问题。

## 8. 仓库内参考资料

- [`docs/VLSI/Basic-Flow.rst`](../docs/VLSI/Basic-Flow.rst)：自定义模块的 Hammer
  综合、P&R、功耗和 signoff 基本流程；
- [`docs/VLSI/Sky130-OpenROAD-Tutorial.rst`](../docs/VLSI/Sky130-OpenROAD-Tutorial.rst)：
  Sky130 + Yosys + OpenROAD 教程；
- [`vlsi/Makefile`](../vlsi/Makefile)：Chipyard 到 Hammer 的 build targets；
- [`vlsi/tutorial.mk`](../vlsi/tutorial.mk)：教程使用的 config、technology、tool 和
  output directory 设置；
- [`vlsi/power.mk`](../vlsi/power.mk)：当前集成的 RTL、post-synthesis 和 post-P&R
  功耗流程。
