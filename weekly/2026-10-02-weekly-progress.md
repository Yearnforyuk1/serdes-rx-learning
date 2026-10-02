# SerDes RX 周学习日志｜2026-10-02

## 本周概览

本周的重点明显转向 **SerDes RX mixed-signal 集成与 Virtuoso/AMS 验证流程**。相比继续推进 CDR/DFE 理论，本周主要解决了“数字/模拟顶层如何正确连接、数字信号如何在 Virtuoso 中按预期观察、AMS 中大量 Verilog 模块如何组织，以及 CDF 参数如何正确传递到仿真”的工程问题。CDR、half-rate DFE、PI、slicer 和 calibration 本周没有形成新的系统性理论推导，因此不强行填充。

## 1. 本周学会或理解加深的内容

### RX / AMS 顶层集成

- 进一步明确了数模混合顶层验证不能只看 schematic 是否“连上了”，需要同时验证 **symbol/interface、config/view binding、Verilog hierarchy、analog/digital discipline 以及实际 netlist**。
- 对大量 Verilog 模块的 AMS 组织方式进一步理顺：顶层 testbench 通常只需要例化实际的顶层 Verilog module 对应的 symbol，子模块由 Verilog hierarchy 继续展开；关键是确保所有模块文件被正确编译、hierarchy 可解析、view/config 绑定正确。
- 开始把“顶层接线验证”从肉眼检查提升到 **schematic → netlist → waveform/logic behavior** 的闭环验证思路。

### Virtuoso 数字波形观察

- 深入排查了数字信号无法通过 Direct Plot 正常画出的情况，并进一步区分了 Result Browser 中不同 result/dataset 的来源。
- 理解了为什么有些结果前面显示 `V`，有些显示 `L`：它们对应不同类型的模拟/逻辑结果数据，因此不能简单按同一种 waveform 数据处理。
- 明确了 **Result Browser 能看到数字结果 ≠ Direct Plot 一定能直接画出同样的信号**；需要确认 save/result 类型以及 Direct Plot 支持的对象。
- `tran` 与 `tran-tran` 的区别也开始从“波形名字不同”转向结果数据结构和仿真结果层级去理解。

### CDF / 参数传递

- 本周重点排查了 `FREQBAND` 类型错误，并确认某些 CDF 参数虽然在界面上表现为表达式字符串，但最终是否能够作为数值表达式参与仿真，取决于 CDF parameter 的类型以及表达式解析相关设置。
- 实际检查后确认，当前需要关注的是 **string 参数及其 parseAsCEL / parseAsNumber 设置**，而不是简单把参数改成 numeric。
- 这建立了一个重要的 AMS/Virtuoso 调试意识：**CDF 表单里的“显示值”“字符串”“CEL expression”“最终 Spectre 数值”是不同层级，不能混为一谈。**

## 2. 已解决或明显收敛的问题

1. **AMS 多 Verilog 模块如何批量组织**：明确了不需要在 testbench 中逐个例化所有子模块；应保持顶层 hierarchy，并确保对应 Verilog source 文件整体加入编译/仿真配置。
2. **数字信号为什么 Result Browser 能看、Direct Plot 却不一定能画**：问题进一步收敛到 result 类型、保存方式和 Direct Plot 对不同数据对象的支持，而不是简单认为“数字波形没有生成”。
3. **FREQBAND 类型错误**：已经定位到 CDF parameter 的类型/表达式解析链路，确认了需要从 string + expression parsing 设置入手。
4. **数字/模拟顶层接线验证**：形成了从 hierarchy/config/netlist 到 waveform 的验证思路，而不是只检查 schematic 连线。

## 3. 本周实验 / 仿真 / 工程排查及结论

- 对 Virtuoso Result Browser、Direct Plot、`tran` / `tran-tran` 结果进行了对照排查，进一步理解模拟结果与数字逻辑结果在数据库中的区别。
- 对 CDF parameter form 做了实际属性检查，确认 `FREQBAND` 相关参数是 string 类型，并围绕 `parseAsCEL` / `parseAsNumber` 的开关组合进行定位。
- 对 AMS 顶层 Verilog hierarchy 与模块文件组织进行了梳理，为后续大规模 gate-level / RTL mixed-signal 仿真减少手工添加模块的工作量。
- 对顶层数模接口的验证方法进行了整理，重点是从“结构正确”推进到“仿真行为正确”。

## 4. 本周没有新增系统性结论的模块

- **CDR：** 没有新的 loop-filter、jitter transfer 或 frequency-offset tracking 推导。
- **Half-rate DFE：** 没有新的 feedback timing、tap adaptation 或 error-slicer 实验。
- **PI：** 没有新的 phase interpolation、code linearity 或 jitter 实验。
- **Slicers：** 没有新的 slicer 电路级分析。
- **模拟/数字校准：** 没有新的 calibration algorithm；本周更多是 calibration/mixed-signal 系统所依赖的 AMS 集成与观测基础设施。

## 5. 最重要的待补知识

1. **把 AMS integration 从“会配置”推进到“会验证”**：建立固定的 top-level connection checklist，包括 port direction、discipline、view binding、Verilog hierarchy、netlist 和关键 waveform。
2. **数字结果数据库**：进一步弄清 `L` / `V`、`tran` / `tran-tran`、digital event result 与 analog waveform result 之间的对应关系，最终形成 Direct Plot 的稳定操作路径。
3. **CDF 参数机制**：继续理解 parameter type、default value、expression parsing 和 netlisting substitution 的完整链路，避免以后遇到类似 `FREQBAND` 错误时靠试开关解决。
4. **回到 SerDes 理论主线**：AMS 工程问题已经连续占据较多学习时间，下周应恢复至少一个 CDR/DFE 主线练习，避免理论训练中断。

## 6. 下周建议

- 完成一次 **数字/模拟顶层连接的系统验证**：从 schematic/config 到最终 netlist，再用一个可控 test pattern 检查关键数字控制信号和模拟响应。
- 整理一页 **Virtuoso AMS Debug Checklist**，覆盖 hierarchy、config、Verilog source、discipline、netlisting、save/result 和 Direct Plot。
- 恢复 **CDR 或 half-rate DFE** 的定量练习；优先选择已有基础上的新问题，避免重复已经掌握的 Kp/Ki、基础量化题。

## 求职复盘价值

本周最值得沉淀的不是某一个 SerDes 电路公式，而是 **mixed-signal integration 和 EDA Debug 能力**：能够区分 schematic 结构、Verilog hierarchy、CDF 参数、netlist、simulation result 和 waveform viewer 各自所在的层级，并通过逐层验证定位问题。

这类经验以后可以整理成项目中的工程能力描述：

**现象 → 定位层级 → controlled check → 修改配置/参数 → 重新 netlist/simulate → waveform 验证。**

---

### 本周一句话总结

> 本周主要把 SerDes RX 的“电路学习”向“可验证的 mixed-signal 系统”推进了一步：重点掌握了 AMS hierarchy、CDF 参数解析、数字结果观察和顶层数模连接验证的方法，同时需要下周主动把学习重心拉回 CDR/DFE 主线。
