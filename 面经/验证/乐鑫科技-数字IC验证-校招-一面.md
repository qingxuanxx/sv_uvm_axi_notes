# 乐鑫科技-数字IC验证-校招-一面

> 来源：小红书
>
> 约束与后仿

## 面试流程

### 闲聊

- 本科与硕士绩点
- 硕士阶段印象最深的课程、课程作业亮点

## 项目提问

- 介绍验证项目
- 因项目涉及后仿，追加提问后仿与时序约束相关知识

## 八股

### SV / UVM

- 事务类地址约束覆盖方法
- 约束交集场景，uvm_do_with / randomize with 使用
- 三种约束重载方式：
  - 派生事务类重写同名约束块
  - constraint_mode 搭配 randomize with 重载
  - post_randomize 函数修改变量值

## 手撕代码

- 使用 SystemVerilog 或 Verilog 手撕斐波那契数列
