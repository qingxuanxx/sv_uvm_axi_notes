# 集益威-数字IC验证-秋招-面试记录

> 来源：小红书 ｜ 27届秋招 ｜ 以八股为主

## 面试流程

- 自我介绍后针对简历展开提问

## 项目提问

- **项目框架**、各组件功能、组件之间**通信方式**

## 八股

### UVM

- **寄存器模型**
- **component 和 object 的区别**
- **write 和 peek 的区别**（TLM 的 put/write/peek 相关）

### SystemVerilog

- **fork 系列语句区别**（fork-join/join_any/join_none）

### 总线协议

- **AXI 协议**

### PHY

- **PHY 相关知识**

## 反问

- 公司主营业务：**高速接口 IP、SoC、车规芯片**，有实际流片经历
- Serdes 数字部分：112G 高频场景下 ADC 输出之后的**均衡、CDR 属于数字部分**
- 公司规模约 200 人，节奏对标 985，基本双休

## 备注

- 八股覆盖面：UVM（组件/通信/寄存器模型/object-vs-component）+ AXI + **PHY/Serdes**（高速接口方向）+ SV 线程（fork 系列）+ TLM（write vs peek）。
- 高速接口 IP 公司会问 PHY/Serdes/CDR——**面试前查公司业务方向**，对口准备。
- write 与 peek 的区别（TLM）：write 是广播推入，peek 是查看队头但不消费（get 才消费）。
