# 汇顶科技 · 数字IC验证 · 校招一面-3

> 来源：小红书 ｜ 原定 45 分钟，实际超时约 10 分钟。

## 项目提问

- 验证环境如何生成
- AI agent 的使用
- 寄存器模型
- SVA 检查了哪些内容、如何判定断言被执行了
- 如何保证 reference model 的正确性
- AHB matrix 的 slave 长时间占用总线导致卡死，有没有验过
- 如何判定验证做得完备了

## 八股

### SystemVerilog

- `logic`、`wire`、`reg` 的区别
- 队列、动态数组
- `event` 的两种等待方式
- 随机产生整数、seed 种子的作用

### UVM

- factory 机制
- phase 机制
- `run_test`
- `uvm_object` 与 `uvm_component` 及其生命周期
- sequence 想拿到 virtual interface 该怎么办
