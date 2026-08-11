# 芯动科技 IC验证 笔试

> 来源：小红书

【一、填空题】

1. 九进制数 `16` 与九进制数 `27` 相加，结果用九进制表示
2. 补全 Verilog 语句符号：给定 `@(posedge clk) a # #1 b | -> # #1 c`，要求在 `a {} b {} c` 中填入符号使功能相等

【二、简答题】

▸ 1. task 和 function

描述 SystemVerilog 中 `task` 和 `function` 的关系。

▸ 2. SVA 断言

编写 SVA 断言：实现 `req` 拉高后，3 个周期后 `ack` 拉高。

▸ 3. 时序分析

两个 D 触发器之间：组合逻辑延迟 Tcomb、clk 到 Q 端延迟 Tcq、时钟偏移 Tskew。写出**建立时间**与**保持时间**需满足的公式。

▸ 4. 时序图 → interface + transaction

给定一个时序图，编写对应的 `interface` 和 `transaction item`。

▸ 5. 跨时钟域脉冲转换

DUT 功能：将 **a 时钟域**的单周期脉冲转换为 **b 时钟域**的单周期脉冲。

- （1）简述该 DUT 使用的跨时钟域处理方法
- （2）说明需要构造哪些场景来验证 DUT 的正确性

▸ 6. 双向 PAD 信号与 IP 信号

- （1）在 DUV 例化中，如何将 IP 和 PAD 相连以实现信号双向传输
- （2）在 Testbench 中添加代码验证双向连接是否正确（可使用 `force` 语句）
