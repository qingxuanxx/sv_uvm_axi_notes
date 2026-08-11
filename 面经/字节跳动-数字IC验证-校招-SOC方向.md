# 字节跳动 数字IC验证（SOC方向）面经

> 来源：小红书 | 发布时间：03-18

【面试结构】

自我介绍 → 项目详细介绍 → 八股（UVM + SV + 覆盖率 + AXI）

【一、UVM】

- 环境整体架构
- driver 驱动 sequence 的方式
- factory 机制的原理与用法
- component 与 object 的区别
- 约束的编写方法

【二、SystemVerilog】

- 动态数组、关联数组、队列三者的区别
- 动态数组大小的约束方式
- 无重复元素的代码实现

【三、覆盖率】

| 问题 | 要点 |
|------|------|
| 功能覆盖率 fcov 的写法 | bins / coverpoint / covergroup |
| cross 交叉覆盖率的实现 | `cross cp1, cp2` |
| 无效 bins 的定义 | `ignore_bins` / `illegal_bins` |
| 如何确认验证完备性 | 覆盖率 + 功能点 review |
| 如何推进覆盖率收敛 | 分析 gap → 补 testcase → 迭代 |

【四、AXI 协议】

| 问题 | 要点 |
|------|------|
| 乱序（out-of-order） | ID 匹配 |
| outstanding | 多笔未完成事务 |
| 交织（interleaving） | 写交织 |
| size / length / 地址 / wstrb 关联 | 给定 size 计算 length |
| AXI4 是否存在写 ID 信号 | 有，WID |
| AXI4 是否支持写交织 | 不支持（AXI3 支持） |
