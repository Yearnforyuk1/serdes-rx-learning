# SerDes RX 周学习日志｜2026-10-23

**统计周期：2026-10-17 ～ 2026-10-23**  
**记录原则：** 区分“已讨论/理解”“已实际操作”“已完成验证”；没有数据的地方不写成仿真结论。

## 本周概览

本周由前期的高速互连与模拟电路基础，转向 **CDR 的 BBPD 与抖动测试、CMOS 反相器型 PI、RX 时钟抖动及前端信号检测**。Virtuoso 的 PI 线性度计算和 AMS 波形观察也有实际操作，但尚无可复现的完整性能数据。

## 1. 核心学习进展

### CDR：BBPD、JTOL 与频率跟踪

- 结合已有 MMPD 学习材料，系统讨论 Alexander 型 bang-bang phase detector 的 data/edge 采样关系，以及半速率实现时的采样对齐和亚稳态问题。
- 将相邻 data/edge 判决关系写成 `A = D[n-1] XOR E[n]`、`B = E[n] XOR D[n]`，并用 `EARLY = B & ~A`、`LATE = A & ~B` 表达互斥相位指示；具体早晚极性仍须与电路采样定义核对。
- 明确区分 **RX jitter tolerance (JTOL)**、**CDR 的动态跟踪带宽**与**静态频偏跟踪范围（ppm）**。已讨论的测试方法是向 PRBS 注入正弦抖动、扫描频率/幅度并检查 BER，同时观察恢复时钟、PD 输出与 PI code。
- **状态：** 完成原理与测试方案讨论；未记录新的 JTOL 曲线、闭环带宽数值或频偏扫描结果。

### PI：CMOS 反相器型相位插值

- 进一步理解 PI 的相位并非简单由控制码线性决定，而是由 I/Q 路驱动权重、输入 slew、MOS 非线性、输出负载与后级判决阈值共同决定。
- 比较了合成节点后直接接 inverter 与交流耦合 self-biased inverter：后者可隔离输入共模并恢复偏置，但会引入 RC 建立、基线漂移和潜在 AM-to-PM 误差。
- 在 Virtuoso Calculator 中实际尝试 **DNL/INL 函数及 fit 设置**，观察了结果图；尚未形成经确认的全码相位表、INL/DNL 峰值或 PVT 结论。

### Slicers / Half-rate DFE

- 本周 slicer 相关讨论主要发生在 BBPD 的 data/edge 采样结构，而非独立的高速比较器电路优化。
- **Half-rate DFE** 本周没有新的 H1 feedback timing、error-slicer/LMS 或 tap 更新仿真；此前问题继续保留为待办，不将其写成已解决。

### RX Clocking：光时钟抖动与 T-coil

- 讨论光电探测器（PD）如何通过散粒噪声、带宽限制及幅度到时间转换增加电时钟抖动；建立近似关系 `σt ≈ σv / |dV/dt|`。讨论中的数值仅是示例，**不是实测 PD 抖动**。
- 开始从耦合电感、互感与桥接电容理解 T-coil 的带宽扩展，并尝试用零输出法分析零点；完整传递函数、零极点及 AC 验证尚未完成。
- 学习了上位机、AWG 与 BERT 在 RX 测试中的分工：配置控制、可编程波形/抖动注入、PRBS 与 BER 测量。

### RX 前端 / 模拟与数字校准

- 梳理 ATT 常见的电阻衰减、RC 补偿及反馈相关结构；对于“可编程 R+C+运放反馈”的实际电路，**仅凭器件组成还不能唯一确定其拓扑**，需根据反馈连接判断是偏置/共模控制、频率补偿还是有源衰减。
- 明确 RSSI 通常通过幅度/功率检测、整流或包络提取及低通滤波生成电压或数字码，可服务 AGC；RSSI **不等同于眼高**。
- 本周没有新的 DCC/IQ 校准收敛数据或模拟—数字校准闭环实验。

## 2. 已澄清的问题与实际操作

**已澄清的认识：** JTOL、CDR 跟踪带宽和频偏范围不能混为一谈；CMOS PI 的实际相位由非线性电流/电容动态共同决定；PD 噪声经边沿斜率转化为时间抖动；RSSI 不是眼图质量的直接替代指标。

**有记录的操作/现象：** 在 Cadence Calculator 中操作 DNL/INL 的 fit；AMS 仿真中仍存在 Direct Plot 无法直接画出某些数字结果、但 Result Browser 可查看 `tran-tran` 数据的现象。上述记录尚不足以确认性能指标或问题根因。

**未完成的验证：** 未记录 BBPD/JTOL 实测、PI 全码 INL/DNL、T-coil AC/PEX 对照、ATT/RSSI 电路扫描或 DFE/LMS 仿真。

## 3. 下周最重要的待办

1. **CDR：** 画出 half-rate BBPD 的采样时序与 EARLY/LATE 真值表；建立一个带 SJ 注入的 JTOL 测试平台，明确 BER 判据和频率扫描点。
2. **PI：** 扫描所有 PI code，导出相位—码值表，并用同一拟合定义计算 DNL/INL；对比直接 inverter 与 AC-coupled self-biased inverter 的 jitter、建立时间和 PVT。
3. **RX clocking：** 以 `σv / slew` 做 PD 噪声—抖动预算；选一个 T-coil 小信号电路完成零点推导并与 AC 仿真比较。
4. **恢复 DFE/校准主线：** 至少完成一次 half-rate DFE feedback timing 或 error-slicer/LMS 信息流的定量练习，避免连续搁置。

## 4. 求职复盘沉淀

本周形成的可复述技术主线是：**从 CDR 鉴相逻辑走向 JTOL 测试方法；从 PI 权重控制走向 INL/DNL 的实际测量；从光电噪声走向采样时钟的时间抖动预算。** 下一步需要补上波形、参数表、测试条件与量化结论，才能把“理解原理”升级为“独立验证设计”。

> **本周一句话：** CDR BBPD/JTOL、CMOS PI 线性度及光时钟噪声模型的理解进一步深化；已有少量工具操作，但没有可核实的新性能测量或根因闭环。
