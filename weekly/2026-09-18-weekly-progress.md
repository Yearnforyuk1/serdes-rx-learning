# SerDes RX 周学习日志｜2026-09-18

## 本周概览

本周没有平均推进所有模块，重心明显转向 **RX clocking / AMS 混合信号仿真 / 版图工程操作**。相比前一周对 CDR、DFE 算法与数模接口时序的集中讨论，本周主要是在真实 Virtuoso 环境中处理长仿真、AMS netlisting、数字/模拟波形保存以及 clock/DCC 相关问题。

## CDR

本周没有新增系统性的 CDR 环路理论推导或参数扫描。当前 CDR 主线仍停留在此前已经建立的 Kp/Ki、量化状态、PI code 与 frequency-offset tracking 基础上。本周的 AMS 仿真问题与顶层数字 Verilog 集成，对后续完整 CDR mixed-signal verification 有直接工程意义。

## Half-rate DFE / Slicers

本周没有新的 half-rate DFE timing、tap adaptation 或 slicer 电路实验，因此不补写不存在的进展。此前 LMS/error slicer 信息流和 feedback timing 仍是后续需要继续推进的主线。

## PI

本周没有新的 PI INL、phase interpolation 或 jitter 仿真结果。下一阶段应继续把 PI 放回 CDR 闭环中分析，而不是重复孤立的 PI 基础。

## RX Clocking

### DCC 检测与滤波稳定时间

进一步明确了 DCC duty-cycle detector 若通过低通/积分方式把时钟占空比信息转换成近似直流控制量，其稳定时间与检测器增益、滤波时间常数和环路带宽有关。减小 RC 可以加快响应，但不能只追求速度，还需要考虑时钟纹波抑制、检测噪声和校准环稳定性之间的权衡。

### AMS 长仿真与波形完整性

本周重点排查了 Virtuoso AMS 长时间仿真中的工程问题：

- 长仿真可能出现进程仍存在但进度长时间不更新，需要区分计算量突然增大、I/O/结果写入、求解器收敛和真正卡死。
- 出现模拟波形可以继续到更晚时间、数字波形却中途停止的情况。这提示后续 Debug 不能只看 analog waveform，需要同时检查数字仿真 kernel、event activity、database/save 配置和 mixed-signal synchronization。
- AMS netlisting 遇到 `AMS-1245`，而 assembler.log 尾部没有直接显示根因。建立了更明确的排查意识：顶层报错通常只是汇总错误，需要继续向 netlister/config/view binding/具体 Verilog cell 的日志追踪，而不能把 `Running netlist assembly` 当作真正错误位置。

### 多 Verilog 模块的 AMS 顶层集成

进一步明确了多个 Verilog module 导入 Virtuoso 后的层次化使用思路：testbench 原则上只需要实例化设计顶层 symbol，顶层 Verilog 再通过 module hierarchy 实例化下级模块；前提是各 module/view、symbol、config binding 和 AMS elaboration 能正确解析整个 hierarchy。

## 模拟 / 数字校准

本周与校准直接相关的重点是 DCC detector 的低通/平均过程及稳定时间问题。当前需要继续建立完整的 calibration-loop 视角：detector 输出纹波、LPF 带宽、控制量 settling time、校准精度和环路稳定性不能分开设计。

## Virtuoso / 工程能力积累

本周补充了几个很实用的版图与仿真操作经验：

- Virtuoso layout 中开始关注多个 pin 的固定间距、左对齐和上下对齐等规则化排列，提高版图编辑效率和一致性。
- 对参数显示异常进行了排查：即使某个参数没有显示在实例旁边，只要属性存在，仍可能通过 `q` 打开属性窗口修改或赋变量；显示问题与参数本身是否存在需要分开判断。
- 长仿真、AMS netlisting 和 mixed analog/digital waveform 不一致的问题开始形成独立 Debug 分类，而不是统一归因于“AMS 有问题”。

## 本周已解决 / 已澄清的问题

1. DCC 检测器的稳定时间确实与滤波时间常数相关，但缩短 RC 不是无代价优化，需要同时考虑 ripple/noise/stability。
2. 多 Verilog AMS 工程不需要把所有子模块 symbol 都直接放进 testbench；正确的顶层 hierarchy 可以由顶层 module 继续实例化子模块。
3. AMS 数字与模拟波形时长不一致不是正常情况下必然存在的现象，应作为 digital kernel/database/event 或 mixed-signal execution 问题继续排查。
4. `AMS-1245` 属于上层 netlisting failure 提示，本身通常不是根因，需要继续定位更早、更具体的日志信息。

## 当前最重要的缺口

### 1. 建立 AMS Debug checklist

需要把近期频繁遇到的问题整理成固定顺序：

`hierarchy/config → view binding → netlisting/elaboration → connect rules/disciplines → analog solver + digital kernel → save/database → waveform`

目标是以后遇到“没波形、数字中断、netlist fail、长仿真不动”时能够快速定位所属层级。

### 2. 回到 SerDes 算法主线

本周工具与仿真 Debug 占比很高。下一周需要主动恢复 half-rate DFE 与 CDR 理论训练，避免连续多周被 EDA 问题完全牵着走。

### 3. DCC calibration loop 定量化

下一步建议建立一个简单的一阶模型，扫描 LPF RC / loop gain，并同时记录 settling time 与 residual ripple，把“减小 RC 会更快”升级为可量化的设计 trade-off。

## 下一步建议

1. 做一次 DCC detector/LPF 参数扫描：至少取 3 个 RC，记录 settling time、稳态纹波和最终 duty-cycle error。
2. 整理一页 AMS Debug flow，并在下一次真实报错时按固定顺序执行。
3. 恢复 half-rate DFE 训练，优先继续 error slicer / LMS information flow 与 feedback timing。
4. CDR 下一步进入带 PI-code quantization 的 closed-loop 小模型，把此前分散学习的 Kp/Ki、积分记忆和 PI resolution 串起来。

## 求职复盘价值

本周最值得沉淀的不是一个新的 SerDes 公式，而是 **mixed-signal integration 与 EDA Debug 能力**：能够区分 hierarchy/netlisting、digital event simulation、analog solver、waveform database 和电路本身的问题。对于真实 SerDes RX 项目，这类能力决定了能否把算法、电路和数字控制真正集成为可验证系统。