# 字节跳动 26届芯片验证 实习面经高频题

> 来源：小红书 | 发布时间：2月27日

【一、SystemVerilog】

- `virtual interface` 的作用，相关差异，没有 virtual interface 能否实现对应功能
- 旗语（semaphore）的含义与典型使用场景

【二、UVM】

| 问题 | 要点 |
|------|------|
| virtual sequence vs 普通 sequence | 差异 + 为什么命名为 virtual |
| UVM 启动接口 | 在 TB 中调用 `run_test` 后触发整个 UVM 平台执行的原理 |
| phase | `run_test` 开始执行的是哪个 phase，UVM 中 phase 的完整分类与分组 |

【三、AXI 协议】

- 多主机访问场景下保障一致性的方法
- 乱序接收的适用场景 vs 不适用场景
- 协议中 order 顺序的规定规则
- **outstanding transaction** 的定义，以及对系统性能的影响

【四、EDA 工具】

- VCS 工具的获取途径
- 是否使用过 VCS 跑仿真、查看波形

【面试建议】

1. 提前准备好自我介绍和项目经历，突出自身优势与特长
2. 熟练掌握基础：Verilog、UVM、AXI 协议
3. 多做编程练习，提升实践动手能力
4. 保持良好心态，自信回答
