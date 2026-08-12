# 泰凌微 IC验证一面

> 来源：小红书 | 9月1日

---

## 闲聊

- 工作地点要求
- 是否可以提前实习
- 接触验证行业的契机

## 项目部分

- DDR SIM 模型相关问题
- AXI-Lite agent 的具体实现，重点考察 **driver 部分的逻辑**
- 功能覆盖率的配置项与设计思路

## 八股部分

### UVM

- `pre_body()`、`body()`、`post_body()` 的执行顺序
- 子类重载并调用 `super.xx_body()` 时的完整执行流程
- UVM 环境的启动流程

### SystemVerilog 数据类型

- bit、logic、real、byte 等类型的区别
- bit 与 byte 的差异、有无符号的特性
- **logic 与 bit 的核心区别**
- interface 的作用与使用场景
- **inout 端口能否用 logic 类型**（不能，需 wire）

### 面向对象

- 重载（overload）的条件

### driver 与 sequencer

- 握手机制
- driver 向 sequence 传递 item 的方式

### 数组

- 动态数组与队列的区别
- 动态数组初始化扩容方法
- 队列清空方法

### 约束与随机化（重点）

- 常用约束关键字：inside、unique、solve before 的用法
- `:=` 与 `:/` 权重的区别
- 随机化完整流程：类内声明 rand 属性、类外调用 randomize 的执行逻辑
- 仅生成 32bit 无符号随机数：`$urandom()`
- 约束实现 bcd 值随机且不超过 a 值一半的写法

### 线程

- 三类 fork join 语句的区别
- 挂起后阻塞等待所有线程结束的实现方式
- 仅等待单个线程结束的实现思路

---

## 关键收获

- inout 端口不能用 logic（会多驱动）——易错细节
- 约束写法现场手撕概率高：bcd <= a/2 是典型题
- pre_body/body/post_body 执行顺序是 sequence 生命周期考点
