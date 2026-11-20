# SerDes RX 周学习日志｜2026-11-20

- **周期**：2026-11-14 ～ 2026-11-20
- **关键词**：BB-CDR、JTOL、四相正交误差、half-rate DFE、PI、LVS、衬底连接
- **记录原则**：区分理论讨论、实际观察、已验证结果与待验证假设。

## 本周概览
本周重点从 PRBS/Demux 数据诊断转向 **BB-CDR 的四相采样误差与抖动容限**，同时集中处理 **Cadence Layout/LVS 的衬底连接和网络识别**。没有可核实的新 JTOL 曲线、PI 码扫描或修复后的 LVS 通过记录。

## 理解加深与已澄清

### CDR / PI：四相非正交与 BBPD 锁定位置
- 针对 half-rate DFE 中 data odd/even、edge odd/even 四相时钟，讨论 I/Q 非正交但 I+/I−、Q+/Q− 各自保持 180° 对称的情况。
- **关键区分**：BBPD 的边沿采样相位锁定目标、data sampling phase 与最佳眼中心不必相同。即使 edge 时钟能被调到跳变位置，正交误差仍可能使 data 时钟偏离眼中心；若偏差改变数据判决/边沿判决统计，也可能进一步改变 PD 特性和锁定点。
- 因此“正交误差完全不影响鉴相”只能在数据判决可靠、采样对应关系正确等条件下近似成立，不能作为普遍结论。
- PI 本周作为 CDR 相位调节执行器参与讨论；无新的 PI INL/DNL 数据。

### CDR：Jitter Tolerance（JTOL）
- 讨论了 SJ 注入、逐频点提高幅度直至 BER 门限、绘制 JTOL 曲线的仿真和芯片测试流程。
- **澄清**：JTOL 曲线的拐点不必等于 CDR 环路带宽；曲线还受 BBPD 非线性、PI 范围、数据路径和 BER 判据影响。应结合 jitter transfer、相位误差或闭环响应独立估计带宽。
- 本周为方法讨论；尚无实际 JTOL 扫频结果或带宽数值。

### Half-rate DFE / Slicers
- 本周围绕 data/edge slicer 的四相采样位置及其对 BBPD 的影响深化理解。
- 无新增 DFE tap、LMS 收敛或 slicer 晶体管级仿真。

### RX Clocking / 模拟数字校准
- 四相 I/Q 正交误差是本周的主要时钟非理想因素；应区分 duty-cycle 对称性、I/Q 正交性、相位插值误差。
- 没有新增 DCC/IQ 校准环路实现或收敛数据。

## 实际工程排查：Cadence Layout / LVS
- 讨论了 layout 的打散/组合、切割线段、未完成布线时的临时 netname 标记、PR boundary 对 LVS 的影响。
- 重点排查 **NMOS 附近标注 GND，却被报告衬底接到 VDD** 的现象：几何邻近或文字标签不能保证电连接；应检查 PDK 的 well/substrate/tap 层、实际连通性、guard ring 与提取网络，再结合 LVS 报告定位。
- **状态**：已明确排查思路；未确认衬底问题已修复或 LVS 已通过。

## 本周实验与结论状态
| 项目 | 状态 |
| --- | --- |
| 四相非正交对 BBPD / data sampling 的影响 | 完成条件化理论分析；待仿真 |
| CDR JTOL 曲线与环路带宽关系 | 完成测试方法讨论；未获得曲线 |
| Layout 衬底误接 VDD / LVS | 已观察异常并讨论排查；未闭环 |
| PI INL/DNL、DFE LMS、DCC 校准 | 本周无新增验证 |

## 下一步（优先级）
1. **四相误差受控仿真**：扫描 I/Q phase error（保持各自 P/N 180°），记录 EARLY/LATE 统计、PI 锁定码、data 采样相位和 BER，验证锁定偏移条件。
2. **JTOL 小规模扫频**：选定 PRBS、BER 判据与 SJ 频点，输出 JTOL 曲线，并用独立闭环相位响应估计带宽，避免直接将拐点当带宽。
3. **LVS 衬底闭环**：对照 tap/well/guard ring 与提取网络，修复后保存 LVS clean 报告及前后对比截图。
4. **恢复 DFE 定量主线**：分析 half-rate data/edge slicer 的采样裕量与 feedback timing；本周不重复无新增成果的 LMS 基础。

## 求职复盘线索
- **系统分析**：能区分 BBPD 锁定相位与 data 最佳采样相位，并提出正交误差的受控验证方案。
- **测试方法**：能解释 JTOL、环路带宽与 BER 判据的不同角色。
- **版图 Debug**：从 LVS 错误报告回溯物理层、tap 与提取连通性；后续须补充修复证据才能形成完整案例。
