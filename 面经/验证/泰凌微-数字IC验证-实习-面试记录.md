# 泰凌微-数字IC验证-实习-面试记录

> 来源：小红书 ｜ 第一次验证面试，面试官持续深挖基础概念，面试者自述概念掌握不清晰。

## 面试流程

- 自我介绍，介绍自学的开源 AXI 验证项目

## 项目提问

- scoreboard、reference model 的设计思路
- 追问 monitor 传输的数据内容、传输方式、数据类型、所用 port、config_db 的参数数量与含义
- 例化的主机、agent、monitor 数量与组件连接逻辑

## 八股

- sequence、sequencer、driver 的功能与互相通信机制
- 动态数组、关联数组、队列的区别
- 三种 fork 语句的区别；fork join 内 while(1) 循环的终止实现
- item 与 transaction 的区别
- logic 与 bit 的区别、logic 额外特性以及和 wire 的区别
- SPI 数据格式、AHB 的 burst 长度类型
- get_name 与 get_full_name 的区别
- 寄存器模型的最小单位、前门访问与后门访问的定义
- task 与 function 的区别、常用数据类型转换函数
- uvm 的 phase 相关问题：build 与 connect 的先后顺序、执行流程以及顺序原因

## 备注

- 项目脚本使用的语言与编写方式
- 面试反馈：暴露自身验证基础薄弱，很多知识点没有实操记忆
