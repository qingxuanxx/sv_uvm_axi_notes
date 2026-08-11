# 思朗 IC验证岗 一二面面经

> 来源：小红书

【一面：验证进阶与场景题】

▸ 进阶知识

| 问题 |
|------|
| xprop 与 initreg 相关知识 |
| 签核前验证还需要关注的要点 |
| 性能验证（performance verification）相关内容 |

▸ UVM 场景

- `main_phase` 运行中途需要触发复位时的跳转方式
- UVM 复位时，sequencer 中缓存了 sequence 已发送但未送出的 transaction，如何**清除残留 transaction**

▸ SoC 集成

- SoC 验证环境中是否保留 BT 级或 IP 级的验证环境
- 模块环境向上集成到 SoC 环境时，哪些组件可以**保留复用**

▸ DDR 场景

- 对 DDR 同一地址同时执行读写操作时，checker 的判断逻辑

【二面：协议 + UVM + 基础外设】

| 类别 | 问题 |
|------|------|
| DDR4 | 引入 BG（Bank Group）概念的原因 |
| UVM | phase 机制 |
| UVM | 寄存器模型相关知识 |
| DDR4 | 自动刷新 vs 自刷新的区别 |
| SPI | 信号线定义与时序规范 |
