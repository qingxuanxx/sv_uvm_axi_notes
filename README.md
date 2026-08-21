# SV、UVM、AMBA 与开发工具学习笔记

> 数字IC验证（Digital IC Verification）学习笔记：SystemVerilog、UVM、AMBA 总线协议、Linux 开发工具 + 面试八股 + 校招面经。

## 内容概览

- 📘 **SystemVerilog 学习笔记**：数据类型、随机化、线程通信、面向对象、功能覆盖率（第 2~9 章）
- 🚀 **UVM 实战学习笔记**：平台搭建、UVM 基础、TLM、phase、sequence、寄存器模型、factory、可重用性（第 1~9 章）
- 🔌 **AMBA 总线协议**：AXI、AXI-Lite、AXI-Stream、AHB、APB
- 🛠️ **开发工具学习笔记**：Linux、Shell、Vim、Makefile、Git
- 🎯 **面试八股**：SV / UVM / AXI 高频考点（持续积累）
- 📄 **面试书笔记**：《Cracking Digital VLSI Verification Interview》整理
- 📝 **校招面经**：57 篇数字IC验证面试经验（持续更新）
- 🧩 **场景设计**：模块验证方案、feature 拆分、测试点分解、故障排查（面试隐藏重点）
- ✍️ **笔试真题**：各公司笔试记录与高频考点归纳
- 🖥️ **手撕代码**：面经手撕题目整理（RTL / 约束 / UVM / 覆盖率 / 线程OOP）

## 目录

```
sv_uvm_axi_notes/
├── sv学习笔记/         # SystemVerilog 学习笔记
│   ├── sv_ch2学习笔记.md    # 第2章：数据类型
│   ├── sv_ch3学习笔记.md    # 第3章：过程语句和子程序
│   ├── sv_ch4学习笔记.md    # 第4章：连接设计和测试平台
│   ├── sv_ch5学习笔记.md    # 第5章：面向对象编程基础
│   ├── sv_ch6学习笔记.md    # 第6章：随机化
│   ├── sv_ch7学习笔记.md    # 第7章：线程以及线程间的通信
│   ├── sv_ch8学习笔记.md    # 第8章：面向对象编程的高级技巧
│   ├── sv_ch9学习笔记.md    # 第9章：功能覆盖率
│   ├── sv_八股.md           # SV 八股（面试常考点积累）
│   └── sv_语法.md           # SV 语法速查
├── uvm学习笔记/        # UVM 实战学习笔记
│   ├── uvm_ch1学习笔记.md   # 第1章：与 UVM 的第一次接触
│   ├── uvm_ch2学习笔记.md   # 第2章：一个简单的 UVM 验证平台
│   ├── uvm_ch3学习笔记.md   # 第3章：UVM 基础
│   ├── uvm_ch4学习笔记.md   # 第4章：UVM 中的 TLM1.0 通信
│   ├── uvm_ch5学习笔记.md   # 第5章：UVM 验证平台的运行
│   ├── uvm_ch6学习笔记.md   # 第6章：UVM 中的 sequence
│   ├── uvm_ch7学习笔记.md   # 第7章：UVM 中的寄存器模型
│   ├── uvm_ch8学习笔记.md   # 第8章：UVM 中的 factory 机制
│   ├── uvm_ch9学习笔记.md   # 第9章：UVM 中代码的可重用性
│   ├── uvm_八股.md          # UVM 八股（面试常考点积累）
│   └── uvm_语法.md          # UVM 常用 API 与宏语法速查
├── axi学习笔记/        # AMBA 总线协议学习笔记
│   ├── apb学习笔记.md       # APB 协议
│   ├── ahb学习笔记.md       # AHB 协议
│   ├── axi学习笔记.md       # AXI 协议
│   ├── axi-lite学习笔记.md  # AXI-Lite 协议
│   ├── axi-stream学习笔记.md # AXI-Stream 协议
│   └── axi_八股.md          # AXI 八股（面试常考点积累）
├── 开发工具学习笔记/   # Linux 与工程开发工具学习笔记
│   ├── linux学习笔记.md     # Linux 环境与常用命令
│   ├── shell学习笔记.md     # Bash 命令与自动化脚本
│   ├── vim学习笔记.md       # Vim 编辑器
│   ├── makefile学习笔记.md  # GNU Make 与仿真流程
│   └── git学习笔记.md       # Git 版本管理与协作
├── 面试书_CrackingVLSI/ # 《Cracking Digital VLSI Verification Interview》笔记（10章）
│   ├── 00_前言与面试准备.md
│   ├── 01_数字逻辑设计.md
│   ├── 02_计算机体系结构.md
│   ├── 03_编程基础.md
│   ├── 04_硬件描述语言.md
│   ├── 05_验证基础.md
│   ├── 06_验证方法学.md
│   ├── 07_版本控制.md
│   ├── 08_逻辑推理与谜题.md
│   └── 09_非技术与行为面试.md
├── 面经/              # 数字IC验证面试经验（57篇，持续更新）
├── 手撕/              # 面经手撕代码题目整理（RTL/约束/UVM/覆盖率/线程OOP，含答案）
├── 场景/              # 场景设计题（模块验证方案/feature拆分/测试点分解/故障排查）
└── 笔试/              # 各公司笔试真题与考点积累
```

## 配套代码

- [verilog-learning](https://github.com/qingxuanxx/verilog-learning)：Verilog RTL 练习代码（组合/时序逻辑、状态机、FIFO、仲裁器等，按周组织）
- [systemverilog-learning](https://github.com/qingxuanxx/systemverilog-learning)：SystemVerilog 练习代码（数据类型、过程语句、接口与 clocking、OOP 等，按章节组织）

## 参考资料

### 书籍
- 《SystemVerilog 验证 测试平台编写指南》（张春 译）
- 《UVM 实战》（张强）
- 《Cracking Digital VLSI Verification Interview》（Ramdas Mozhikunnath & Robin Garg）

### 官方协议文档（AMBA）
- ARM AMBA AXI and ACE Protocol Specification（IHI 0022，含 AXI-Lite）：https://developer.arm.com/documentation/ihi0022
- ARM AMBA AXI-Stream Protocol Specification（IHI 0051）：https://developer.arm.com/documentation/ihi0051
- ARM AMBA 3 APB Protocol（IHI 0024）：https://developer.arm.com/documentation/ihi0024
- ARM AMBA AHB Protocol Specification（IHI 0033）：https://developer.arm.com/documentation/ihi0033

> 最后更新：2026-08-22
