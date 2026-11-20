# 达摩院-数字IC验证-校招-一面

> 来源：小红书 ｜ 背景：海外 9 所院校硕士，数模混合芯片验证实习
>
> 次日进入人才库，面试未通过。面试官态度友好。

## 八股

### 寄存器验证

- 寄存器的验证方法
- 引入寄存器模型的优势
- 寄存器模型对验证完备性的提升作用

### Sequencer 相关

- p sequencer 的定义与用途
- m sequencer 与 p sequencer 的区别

### 线程控制

- 三类 `fork join` 语句的区别
- 终止 `fork join_none` 进程的方法

### Sequence

- 使用 `uvm_do` 宏时 sequence 是否会出现阻塞
- virtual sequencer 与 virtual sequence 的用法

### Testbench

- Testbench 的基本组成

## 手撕代码

> 编写寄存器随机验证对应的 **sequence item** 与 **sequence**，面试官随后针对代码细节提问。

## 备注

### 最后

- 面试官询问候选人是否有擅长但未考察到的内容。
- 次日查询：进入人才库，未通过。

### 关键收获

- 达摩院即使面试官态度友好，淘汰也很快
- 寄存器模型 + sequence item/sequence 是高频手撕方向
- 海外背景不会自动加分
