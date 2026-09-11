# SerDes RX 周学习日志｜2026-09-11

> 本周主线：从“模块功能理解”进一步转向 **RX 时序接口、DFE 自适应信息流、时钟链路非理想因素与 AMS/后仿调试**。本周没有为了覆盖目录而补写没有实际推进的 PI 内容。

## 1. 本周核心进展

### CDR

本周 CDR 没有新增完整环路推导，但延续了数字环路/量化状态的训练：进一步确认数字积分器或控制字更新必须使用**上一拍已经寄存/量化后的状态**，而不是把小于 1 LSB 的理想连续增量偷偷保留下来。因此当单次增量小于 1 LSB 且实现中没有额外残差累加器时，可以连续多拍量化为 0。这一点对以后判断 CDR dead zone、limit cycle、低 Ki 下的频偏跟踪能力很重要。

本周更大的推进实际上发生在与 CDR 紧密相关的数字时序概念：系统梳理了 input delay、arrival time、internal delay、setup 与 capture edge 的关系，并进一步追问为什么 setup 检查针对当前 launch/capture 关系中的“最晚数据到达”，而不是机械地拿 capture edge 前任意一个数据边沿比较。这为后续分析 CDR/校准数字模块接口时序打基础。

### Half-rate DFE / Slicers

本周把 DFE LMS 的理解推进到了**相关性与 error slicer 信息流**层面：LMS 更新的核心不是简单地“看 error 大小”，而是利用数据/历史判决与误差信号之间的相关性估计 tap 梯度。由此进一步追问了 error slicer 到底需要什么控制信息，以及它与 data slicer 接收 DFE tap 后完成主判决的职责差异。

这暴露出下一步需要补齐的关键点：要把 data slicer、error/margin slicer、summer/DFE tap、adaptation logic 画成一张完整的信息流图，明确每个 slicer 的比较阈值、输入信号和数字输出分别服务于“数据恢复”还是“系数更新”。

### PI

本周没有新的 PI 电路、INL 或相位插值仿真结论，因此不单独扩写。后续 PI 学习应继续与 CDR tracking 和 clock path jitter 结合，而不是重复基础相位插值原理。

### RX Clocking

本周 RX clocking 的工程理解明显加深，主要有三条线：

1. **频率相关传播延迟**：开始区分“幅度损耗随频率变化”和“传播相位/群延迟随频率变化”。真实互连的 R/L/G/C 及介质、导体损耗会让传播常数随频率变化，因此不同频率分量不仅衰减不同，相位延迟也可能不同；对高速 clock/data 来说，这最终体现为边沿形状和确定性时序误差。
2. **长距离两相时钟周期性交叉**：讨论了芯片内两相/差分长时钟走线为什么会周期性交叉，重点开始从几何对称、环境平均化和 mismatch 控制角度理解，而不只是把它看成布线习惯。
3. **占空比失真经过 inverter 后无法完整翻转**：观察到约 40% duty clock 经过反相器后低电平拉不下去、共模/平均电平发生变化。这提示后续分析不能只看逻辑 0/1，而要同时检查 PMOS/NMOS drive strength、输入高低电平持续时间、负载 RC、slew rate、静态工作点和级联恢复能力。

### 模拟/数字校准与接口时序

本周围绕 DCD 校准输出提出了一个很有价值的接口问题：**DCD 比较结果是 1/8-rate NRZ，而处理它的数字模块使用 full-rate clock，是否因此可以忽略 input delay？** 当前结论方向是：低速并不自动等于“不需要时序约束”。是否需要 input delay/同步处理取决于该信号与数字时钟之间是否有定义明确的同步关系、最坏到达窗口以及数字模块如何采样它；低 toggle rate 只能增加稳定窗口，不能替代时序定义。

同时系统梳理了数字 input delay 的定义：外部数据相对于参考时钟在模块边界处的最早/最晚到达关系；内部 STA 再把 input delay 与内部组合路径、setup/hold 等组合起来检查捕获是否成立。这一块对以后 analog calibration result → digital logic 的接口约束非常重要。

## 2. 本周仿真 / Debug 记录

### AMS / Virtuoso

本周继续排查 AMS 仿真中的可观测性问题：出现“明明选择保存信号，但仿真结果显示未保存”、Verilog-A 输出异常，以及 APS 有结果而 Spectre X high-performance option 下没有预期结果的现象。进一步区分了 Result Browser 中不同 transient result dataset，并关注如何观察数字模块内部 reg 变量。

同时确认门级 Verilog 可以进入 Virtuoso AMS 混合信号仿真流程，但必须特别注意 discipline/connect rule、供电与逻辑电平、timescale/时间精度、数字事件与模拟 solver 的接口，以及内部数字节点是否被保存。

### 前仿模型问题

本周遇到前仿真器件/模型引用异常：模型或 cell 定义实际上存在，但 schematic 中难以定位对应器件。这个问题当前属于环境/模型绑定排查，不把它误记成电路设计结论。

## 3. 已解决 / 明显澄清的问题

- 明确：**低速 calibration output 并不因为是 1/8-rate 就天然免除 input-delay / CDC / capture-window 分析。**
- 明确：STA 中 `input delay + internal delay + setup <= capture budget` 各项对应的是边界到达、模块内部传播和接收寄存器要求，不能把 input delay 理解成模块内部延迟。
- 加深：DFE LMS 应从“相关性/梯度估计”理解，error slicer 是 adaptation information path 的一部分，而不是另一路普通 data slicer。
- 加深：互连的频率依赖不仅体现在 loss，也体现在 phase constant / group delay，因此可能改变高速边沿时序。
- 工程经验：Virtuoso 版图误高亮大网络时，`Ctrl+C` 可以快速中止高亮操作，避免 CIW 卡顿继续恶化。

## 4. 当前最重要的知识缺口

### A. DFE adaptation 信息流还没有完全闭环

下一步要能够不看资料独立画出：

`data slicer decision → delayed decisions → tap multiplication/DAC → summer → slicer`

以及

`error/margin slicer → error information → correlation/update logic → tap code`

并解释 error slicer 的阈值为什么这样设置、它到底在估计什么梯度。

### B. Calibration output → digital logic 的 STA/CDC 边界

需要继续区分三种情况：完全同步、源同步/有固定相位关系、异步或慢速但无固定相位关系。重点不是“信号多慢”，而是**数字捕获端是否知道它什么时候可能变化**。

### C. RX clock buffer 的 duty-cycle / slew / operating-point 联动

40% duty 输入经过 inverter 后不能完整翻转值得继续做定量实验。建议下一次固定负载，分别扫描 duty cycle、输入 slew、PMOS/NMOS ratio 和 fanout，记录 `VOH/VOL`、crossing time、输出 duty cycle 与平均电平，从而区分是 drive-strength asymmetry、settling 不足还是 bias/负载问题。

### D. AMS 可观测性与仿真器差异

需要形成一套可复用 checklist：保存设置 → hierarchy/config → connect rule → analog/digital dataset → APS/Spectre X option → Verilog-A/Verilog internal signal visibility。这样以后遇到“电路可能没错，但波形看不到”时能快速把工具问题与设计问题分离。

## 5. 下周建议优先级

1. **DFE LMS + error slicer 闭环**：从公式相关性推进到完整硬件信息流和一次手算 tap update。
2. **Calibration interface timing**：用一个 1/8-rate calibration output + full-rate digital clock 的具体例子，分别做同步和异步两种 timing diagram，判断 setup/hold/CDC 要求。
3. **RX clock inverter duty-cycle 实验**：对本周 40% duty 不能完全翻转的问题做参数扫描，把现象变成可量化结论。
4. **CDR 数字量化模型**：在已有 Kp/Ki 理解上加入 PI-code LSB、饱和和 residual accumulation，比较是否存在 dead zone / limit cycle。

## 6. 求职复盘价值

本周最值得保留的不是某一个公式，而是两类能力开始变得更具体：

- **跨模拟/数字边界思考时序**：能够追问 calibration result 的速率、相位关系、input delay、capture edge 与 CDC，而不是因为信号“很慢”就忽略 STA。
- **从波形现象反推物理限制**：面对 duty-cycle distortion、互连频率相关延迟和 AMS 波形缺失，开始主动区分器件驱动/RC settling、传播相位与工具可观测性问题。

这两类能力都比单纯记住 SerDes 模块框图更接近实际 RX 设计与面试中的问题分析方式。
