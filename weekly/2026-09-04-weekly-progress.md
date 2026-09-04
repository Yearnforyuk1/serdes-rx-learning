# SerDes RX Weekly Progress — 2026-09-04

> 本周主线：从 CDR 数字环路滤波器继续推进到 half-rate DFE 的关键反馈时序，同时大量实际排查集中在 RX clocking 的自偏置时钟接收 buffer、PEX/寄生参数与传输线/S 参数问题。

## 1. 本周核心进展

### CDR / PI / 数字环路滤波器

- 进一步理解 PI-based CDR 中比例路径 `Kp` 与积分路径 `Ki/(1-z^-1)` 的分工：比例路径负责即时相位修正，积分路径保存历史状态并消除持续频偏。
- 能够从数字环路滤波器形式中识别积分器带来的 `z=1` 极点，并推导 PI filter 的零点位置。
- 对 `u[n]` 的增量表达有了更具体的物理理解：当 PD 输出方向突然反转时，比例项立即提供“刹车”，积分项仍保留之前累积的频偏记忆。
- 已完成 PI filter zero / accumulator / state-memory 等连续训练，开始从“知道 Kp/Ki 的作用”转向“能从 z-domain 解释其动态行为”。

### Half-rate DFE

- 本周重点进入 half-rate DFE 第一抽头反馈时序。
- 明确 half-rate 架构中 even/odd 两条路径之间存在 1 UI 的反馈时间预算；反馈逻辑、DAC/summer 等延迟之和必须落在该预算内。
- 在练习中对 65 ps 预算、62.5 ps 级反馈延迟进行了 timing 判断，能够识别非常小的剩余裕量以及潜在 timing violation。
- 已能从架构层面判断优化方案：若真正瓶颈来自 `tDAC + summer`，直接从关键反馈路径移除这部分延迟，比单纯压缩其他非关键延迟更有效。

### RX clocking / 自偏置时钟接收 Buffer

本周实际工程排查最集中在这一部分。

- 继续分析交流耦合 self-biased inverter clock buffer 的工作点、增益与级联行为。
- 对“单级 AC gain 正常，但两级级联总增益反而异常降低”的现象进行了系统排查，认识到不能只看单级小信号增益，还必须检查级间负载、反馈、DC operating point 和寄生参数。
- 两条版图看似一致的 clock path 在后仿中出现约 600 mV 与 425–450 mV 的不同 self-bias 点；进一步隔离发现：仅提取 C/CC 时结果正常，加入 R extraction 后问题出现。
- 因此目前最强证据指向寄生电阻网络/连接路径导致的工作点变化，而不是简单归因于 MOS 模型或电容寄生。
- 对后仿中 MOS operating point、Calibre extraction、AC differential stimulus 和测量参考节点等问题进行了进一步排查。

### 传输线 / S 参数 / 时钟互连

- 加深了“只要使用分布参数/S 参数模型，就必须显式考虑端口阻抗与反射”的认识。
- 理解开路端反射系数 `Γ=+1` 后，端点电压可能由入射波与反射波叠加而增大；因此理想源驱动的 S 参数网络中看到超过原输入幅度的瞬态电压，并不等价于有源增益。
- 理解 S 参数中的 dB 通常以波功率/幅度关系定义；对 1-to-2 分支中约 -3 dB 的功率分配意义进行了澄清。
- 观察到 E-shaped / branched clock interconnect 中不同位置的方波失真并不一定随距离单调增加；局部波形由入射波和多个反射波的相位叠加决定，因此更远端反而可能暂时看起来更接近方波。
- 当前已开始把传输线反射问题与后级 self-biased clock buffer 的实际输入波形联系起来，而不是把 interconnect 与 amplifier 分开看。

## 2. 已解决或明显推进的问题

1. **为什么 CDR 需要积分路径？** 不是单纯为了提高增益，而是为了保存控制状态、跟踪持续频偏；数字累加器对应 `z=1` 极点。
2. **Kp 的“即时作用”如何体现？** PD 决策翻转时，比例项立即改变控制量，而积分状态不会瞬间消失，因此形成快速制动 + 慢速记忆的组合。
3. **Half-rate DFE feedback timing 的核心约束是什么？** even/odd 反馈必须在约 1 UI 的可用时间内完成，关键路径总延迟必须小于该预算。
4. **两条相同 clock buffer 后仿工作点为何可能不同？** 当前隔离结果显示加入 R extraction 才出现异常，排查重点已经收敛到寄生电阻/连接网络，而非继续泛化怀疑所有 PEX 元件。
5. **为什么带 S 参数后必须关心匹配？** 因为互连被建模为传播波网络，阻抗不连续会产生反射；开路、分支、耦合电容都会改变反射和瞬态叠加。

## 3. 本周完成的实验 / 仿真 / 排查

- 对 self-biased inverter 两级 clock buffer 做 post-layout AC / operating-point 相关排查。
- 对比 PEX 配置：`C + CC` 与加入 `R` 后的结果，确认异常与 R extraction 强相关。
- 对两条版图相似 clock path 的 self-bias voltage 做比较，发现明显工作点差异。
- 分析 differential clock S-parameter 网络、开路端和分支节点的瞬态波形。
- 比较不同位置 clock waveform，观察到中间节点失真可能比远端节点更明显。
- 完成 half-rate DFE h1 feedback timing 练习，对 1 UI timing budget 和关键延迟路径进行判断。

## 4. 本周尚未重点推进的模块

### Slicers

本周没有新的系统性 slicer 训练。之前建立的 data / phase / error / margin slicer 架构认识仍在，但本周没有形成新的可验证结论，因此不强行补充。

### Analog / Digital Calibration

本周校准算法本身不是主线。与 calibration 相关的基础仍主要停留在 IQ calibration、DFE tap/error 信息以及 CDR tracking 的既有理解；尚未形成新的闭环 calibration 实验。

## 5. 当前最重要的知识缺口

### A. Half-rate DFE timing 需要从“算预算”进入“完整画关键路径”

下一步应能独立画出：

`previous decision → latch/retime → feedback logic → DAC/summer → next slicer decision`

并明确每个节点属于 even 还是 odd phase、可用时间到底是 0.5 UI、1 UI 还是其他架构相关窗口。

### B. CDR 需要继续从 PI filter 推进到闭环动态

目前已经掌握 pole/zero 和 Kp/Ki 的基本物理意义，下一步应进入：

- PD gain
- PI gain / phase-to-code gain
- loop bandwidth
- damping / stability
- frequency offset tracking
- jitter transfer / jitter tolerance

最终目标是能够从 RX block diagram 建立一个简化的离散时间 CDR 模型，而不仅是分析单独的数字 filter。

### C. RX clock buffer 的 PEX-R 根因仍需闭环

目前“R extraction 导致异常”只是定位方向，还不是最终根因。下一步应继续回答：

- 哪一段寄生 R 改变了 DC feedback path？
- 两条 supposedly symmetric path 的 extracted resistance 是否真的一致？
- 是否存在意外串联电阻、via/contact resistance、错误网络合并或反馈路径电阻差异？
- DC operating point 改变后，为什么会进一步导致 AC gain 差异？

这会是非常有价值的真实 post-layout debug 案例，建议最终整理成一份独立工程案例。

### D. S 参数知识需要从反射直觉进入 mixed-mode differential analysis

下一阶段重点应转向 `Sdd11 / Sdd21 / Sdc / Scd`、差分 100 Ω reference impedance、branch discontinuity，以及如何把 S 参数指标和 16 GHz clock edge / jitter 联系起来。

## 6. 下周建议主线

**主线 1：Half-rate DFE feedback timing** 继续做 1–2 个带具体 UI / delay 数值的关键路径题，直到能不依赖提示画出 timing diagram。

**主线 2：CDR loop model** 从目前的 `Kp + Ki/(1-z^-1)` 出发，把 PD、PI/VCO 等模块串起来，建立第一个完整离散时间闭环模型。

**主线 3：PEX debug case** 把 clock buffer 的 `C/CC OK → R extraction failure` 做成 controlled experiment：逐步打开 R 类别或对比关键网络 extracted R，找到造成 600 mV vs 425–450 mV 工作点差异的具体器件/网络。

Slicer 与 calibration 暂时不追求平均分配时间；等 CDR/DFE 两条主线推进一层后再重新接入。

## 7. 求职能力沉淀

本周最值得保留到未来面试中的不是某一个公式，而是两类能力：

- **系统推理能力**：能够从 half-rate DFE 架构推导反馈 timing budget，并根据关键路径判断真正应该优化的模块。
- **后仿 debug 能力**：面对“版图相同但两路工作点/增益不同”的问题，通过逐项改变 PEX 条件，将问题从泛泛的 post-layout mismatch 收敛到 R extraction。

如果后续把第二项完整定位到具体寄生网络，它可以成为一个很好的 SerDes RX / analog IC 面试项目案例：**现象 → 假设 → controlled experiment → root cause → design/layout fix → verification**。
