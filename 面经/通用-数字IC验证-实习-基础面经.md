# 数字IC验证 通用实习基础面经汇总

> 来源：小红书 | 发布时间：4月7日
>
> 覆盖协议验证、断言、SV/UVM 基础、电路基础、验证场景等多个方向。

【一、协议验证】

- 如何验证 AXI 协议
- 如何验证 outstanding 机制

【二、断言（SVA）】

- valid-ready 握手断言：`valid` 拉高的当周期 `ready` 为高
- 交叠 vs 非交叠的区别
- 多周期判断方法
- `$rose` 系统函数的用法

【三、SV 与 UVM 基础】

▸ SV

- `function` 与 `task` 的区别
- 线程同步方式：event、旗语（semaphore）等

▸ UVM

| 考点 | 内容 |
|------|------|
| phase 机制 | 各 phase 执行顺序 |
| factory 机制 | 工厂模式 + override |
| config_db 机制 | 四个参数 |
| TLM 通信 | port / export / imp / analysis_port |
| 覆盖率 | 功能覆盖率、代码覆盖率、断言覆盖率 |

【四、电路基础】

- 异步复位同步释放的原理
- 阻塞赋值 vs 非阻塞赋值
- 跨时钟域处理方法
- 异步 FIFO 原理与**深度计算**

【五、验证场景】

- 芯片设计全流程
- 异步 FIFO 的验证场景与要点
- 复位信号的验证注意事项
- Python 在 IC 验证工作中的应用场景
