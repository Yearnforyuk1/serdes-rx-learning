# SerDes RX Learning Journal

面向多通道 SerDes RX 与 TSMC 28 nm 模拟/混合信号设计的系统化训练记录。

## 训练主题

- CDR 与数字环路滤波器
- 半速率 DFE
- Phase Interpolator（PI）
- Data / Phase / Error slicer
- RX clocking
- 模拟与数字校准
- Z 变换、极点零点、稳定性与定点实现

## 目录

每次练习保存在 `training/` 下，文件名格式为：

```text
YYYY-MM-DD-topic.md
```

每场记录依次包含：主问题、回答、点评、递进追问、追问回答和核心总结。

## 已完成练习

1. [PI 跟踪恒定频偏](training/2026-08-24-pi-frequency-offset-tracking.md)
2. [CDR 为什么需要比例路径和积分路径](training/2026-08-26-cdr-proportional-integral-paths.md)
3. [PI 环路滤波器的即时修正与状态记忆](training/2026-08-28-pi-filter-state-memory.md)
4. [数字累加器与 z=1 极点](training/2026-08-29-digital-accumulator-z1-pole.md)
