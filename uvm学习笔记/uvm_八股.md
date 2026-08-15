# UVM 八股

> 面试常考的 UVM 八股积累（持续更新）。
>
> 阅读配套：`../uvm学习笔记/`（UVM 实战笔记）

## 目录

1. [uvm_object 与 uvm_component 的区别](#1-uvm_object-与-uvm_component-的区别)
2. [config_db 完成 set 后，若源变量发生变化，get 到的值是否同步更新？](#2-config_db-完成-set-后若源变量发生变化get-到的值是否同步更新)
3. [uvm_info 的消息等级分类](#3-uvm_info-的消息等级分类)
4. [config_db 的四个参数是什么？](#4-config_db-的四个参数是什么)
5. [描述 UVM 的树形结构](#5-描述-uvm-的树形结构)
6. [多个组件对同一个配置 set，最终谁生效？](#6-多个组件对同一个配置-set最终谁生效)
7. [config_db 字段名前缀匹配（agt_* 的坑）](#7-config_db-字段名前缀匹配agt_-的坑)
8. [config_db 的 set/get 层级匹配规则](#8-config_db-的-setget-层级匹配规则)
9. [uvm_component 的生命周期](#9-uvm_component-的生命周期)

---

## 1. uvm_object 与 uvm_component 的区别

### 题目来源

- 小米 · ASIC验证 · 实习（收尾八股）
- 平头哥 · 数字IC验证 · 校招 · 一面（凉经）
- 新凯来 · 数字IC验证 · 校招（UVM 基础）
- 集益威 · 数字IC验证 · 校招 · 一面（UVM 基类）

### 考点

- UVM 基类继承体系（component 派生自 object）
- component 树 / parent / phase 生命周期

### 参考答案

uvm_component 和 uvm_object 不是并列关系，**uvm_component 继承自 uvm_object**（中间隔了 uvm_report_object）。所以所有 component 都是 object，但不是所有 object 都是 component。

区别主要四点：

**第一，进不进树**。component 通过构造时的 parent 参数建立父子关系，进入 UVM 树，路径形如 `uvm_test_top.env.i_agt.drv`；object 不进树，没有 parent。

**第二，生命周期**。component 有 UVM 自动执行的 phase（build、connect、main、report 等），贯穿整个仿真；object 按需创建、用完即弃，没有 phase。

**第三，构造与注册**。component 构造函数是 `new(name, parent)`，注册用 `uvm_component_utils`；object 构造函数是 `new(name)`，注册用 `uvm_object_utils`。

**第四，典型成员**。driver、monitor、agent、env、test、scoreboard 都是 component；transaction、sequence、config object 是 object。

一句话总结：**component 是长期存在的平台结构，object 是短生命周期的数据和行为。**

| 对比项 | uvm_object | uvm_component |
|--------|-----------|---------------|
| 继承关系 | 基类 | 继承自 object |
| UVM 树结点 | 否 | 是 |
| parent | 无 | 有 |
| phase 自动执行 | 否 | 是 |
| 构造 | new(name) | new(name, parent) |
| 注册宏 | uvm_object_utils | uvm_component_utils |
| 典型成员 | transaction、sequence | driver、env、test |

---

## 2. config_db 完成 set 后，若源变量发生变化，get 到的值是否同步更新？

### 题目来源

合见工软 · 数字IC验证 · 校招 · 一面（原题）

### 考点

- config_db 机制：set 与 get 的传递语义
- 值拷贝 vs 引用传递（基础类型 vs object 类型）

### 参考答案

不会同步更新。config_db 的 set 对基础类型是**值拷贝**——调用 set 的那一刻，把源变量的当前值复制一份存进 config_db 的资源表里。之后源变量怎么改，都只影响它自己，资源表里存的还是当时拷贝的值，所以 get 拿到的永远是 set 时刻的值。

**但要特别注意一个例外**：如果 set 的是 object 类型（比如自定义的 config object），传的是**句柄引用**，资源表里存的是对象指针，get 拿到的是同一个对象。这时 set 之后再去改那个对象的字段，接收方是能看到变化的。

所以这道题的核心就是一句话：**基础类型值拷贝、改源变量不影响；object 类型引用、改对象会生效。**

| 传的类型 | set 语义 | set 后改源变量，get 会不会变 |
|----------|----------|------------------------------|
| 基础类型（int/bit/string/enum） | 值拷贝 | ❌ 不变 |
| object 类型（config object） | 句柄引用 | ✅ 会变（同一个对象） |

---

## 3. uvm_info 的消息等级分类

### 题目来源

合见工软 · 数字IC验证 · 校招 · 一面（原题）

### 考点

- UVM report 机制：verbosity（冗余度）等级
- verbosity 与 severity 的区分

### 参考答案

`uvm_info` 的消息等级就是 UVM 的 verbosity 等级，从低到高一共有六档：**NONE 是 0，LOW 是 100，MEDIUM 是 200，HIGH 是 300，FULL 是 400，DEBUG 是 500**。

| 级别 | 数值 |
|------|------|
| UVM_NONE | 0 |
| UVM_LOW | 100 |
| UVM_MEDIUM | 200 |
| UVM_HIGH | 300 |
| UVM_FULL | 400 |
| UVM_DEBUG | 500 |

显示规则是：**消息的等级小于等于当前阈值才显示**。默认阈值是 MEDIUM，所以 LOW 和 MEDIUM 能看到，HIGH 及以上默认看不到。

想让更详细的日志显示出来，本质就是把阈值调高，三种办法：

1. **代码里调**：写 `set_report_verbosity_level(UVM_HIGH)` 只影响当前组件；加个 `_hier` 写成 `..._level_hier(UVM_HIGH)` 就变成"我和我下面所有子组件都调"。注意要等组件都建好（connect_phase 之后）才能用路径去设置。

2. **按 ID 调**：`set_report_id_verbosity("DRV_DATA", UVM_HIGH)` 只让标签叫 "DRV_DATA" 的日志变详细，其他日志不变。日志太多的时候用这个精准找某条消息。

3. **命令行调**：启动命令加 `+UVM_VERBOSITY=UVM_HIGH` 就全局生效，不用改代码，临时调试最方便。

**别混淆**：verbosity 管"消息写得有多详细"，severity 管"消息有多严重"（INFO/WARNING/ERROR/FATAL 四档），是两个不同的维度。

---

## 4. config_db 的四个参数是什么？

### 题目来源

- 豪威科技 · 数字IC验证 · 校招 · 一面（config_db 四个参数含义）
- 通用 · 数字IC验证 · 实习 · 基础面经（config_db 机制）
- 某TPU · 数字IC验证 · 校招 · 面经（config_db 核心作用与各参数含义）

### 考点

- config_db 的 set 四参数
- context / 路径 / 字段名 / 值的含义与配合

### 参考答案

config_db 是 UVM 的**配置传递机制**，set 有四个参数：**context（起始上下文）、实例路径、field name（字段名）、value（配置值）**。

**context** 是"谁在调 set"——组件内部有 this 就用 this，top_tb 是 module 没有 this 就用 null；**实例路径**是相对 context 的目标组件路径，比如从 test 出发写 "env.i_agt.drv"，它和第一参数联合确定"配置发给谁"；**field name** 是配置的标签，get 时必须一字不差；**value** 是配置值本身，类型由尖括号里的参数化类型决定，比如 `uvm_config_db#(int)` 就传 int。

get 时也是四个参数：当前组件、空串（表示自己）、字段名、接收变量。

| 参数 | 含义 | 类比 |
|------|------|------|
| 1 context | 起始上下文（谁在寄） | 寄件人 |
| 2 实例路径 | 目标组件路径（寄给谁） | 收件地址 |
| 3 field name | 字段名（标签） | 包裹标签 |
| 4 value | 配置值（内容） | 包裹内容 |

> 一句话：**set 是寄快递（谁寄、寄给谁、标签、内容），get 是收快递；类型、路径、字段名三个钥匙任一不对就收不到**。

---

## 5. 描述 UVM 的树形结构

### 题目来源

- 燧原科技 · 数字IC验证 · 校招 · AI方向（描述 UVM 的树形结构）

### 考点

- UVM 树形结构
- uvm_top 与 uvm_test_top 的区别
- 树的作用（phase 调度 / 配置寻址 / 调试）

### 参考答案

UVM 用一棵**组件树**管理整个验证平台，**真正的根是 uvm_top，不是 uvm_test_top**。

从根往下看：**uvm_top** 是全局唯一的根（uvm_root），由 UVM 框架自动创建，不随测试变化；**uvm_test_top** 是当前跑的测试用例，由 run_test 创建，跑不同测试就换成不同的类，但名字固定不变；再往下是 **env**（平台容器），下面挂 agent（输入 active、输出 passive）、reference model、scoreboard，agent 下面再挂 driver、monitor、sequencer。组件通过构造时的 parent 参数挂树，**路径由实例名决定**，比如 `uvm_test_top.env.i_agt.drv`。

这棵树有三个用途：**统一执行 phase**（build 自顶向下、connect 自底向上、run 并行）；**config_db 按路径查配置**（路径就是树上寻址）；**遍历调试**（`uvm_top.print_topology()` 打印整棵树）。

> 一句话：**uvm_top 是真根管框架、uvm_test_top 是测试根可替换，整棵树承载 phase 调度、配置寻址和结构调试**。

---

## 6. 多个组件对同一个配置 set，最终谁生效？

### 题目来源

- 某TPU · 数字IC验证 · 校招 · 面经（多层级组件对同一变量执行 set 操作时的生效规则）

### 考点

- config_db 多重设置优先级
- 跨层次设置（高层优先）与同层次设置（后写优先）
- context 用 this vs root 的影响

### 参考答案

分两种情况——**跨层次看地位，同层次看时间**。

**跨层次设置**（比如 test 和 env 都 set 同一个字段）：build 阶段按设置者的层次决定，**越靠近根、层次越高的优先**，所以 test 会覆盖 env。这样设计是有意的：可复用的 env 提供默认配置，具体 test 能覆盖它又不用改 env 源码——对应工程上的三层分工：env 给默认值、base_test 给公共配置、具体 test 给场景覆盖。

**同一层次重复设置**：**后写入者生效**。这就是 base_test 和子 test 的配合原理——子 test 在 super 之后再 set 一次就覆盖默认值。

**有个坑**：如果 test 和 env 都用 `uvm_root::get()` 当 context，层次信息丢失，只能按时间分胜负；而 build 自顶向下，test 先跑、env 后跑，env 反而赢了，与预期相反。所以**组件内 set 第一参数尽量用 this**。

> 一句话：**跨层高层赢、同层后写赢、都写成 root 只剩时间说了算**。

---

## 7. config_db 字段名前缀匹配（agt_* 的坑）

### 题目来源

- 汇顶科技 · 数字IC验证 · 校招 · 一面（config_db 字段名的前缀匹配规则）

### 考点

- config_db 字段名通配符
- 前缀匹配规则与配置错乱
- 字段名写全的工程建议

### 参考答案

config_db 的字段名支持**通配符前缀匹配**。如果 set 时字段名写成 "agt_*"，它会匹配所有以 "agt_" 开头的字段名——比如你想配 "agt_axis"，但 get 端用了 "agt_*"，就会把一堆 agt_ 开头的配置项都匹配上。

**这个坑的本质**：字段名不写全、用通配符，会意外匹配多个配置项导致配置错乱；而且很难排查——set 本身不报错，只是 get 拿到的值不对。

工程上两个建议：**字段名尽量写完整精确**，避免宽泛通配符；**调试时先查匹配范围**，用 `print_config` 或 `+UVM_CONFIG_DB_TRACE` 看实际匹配了哪些项。

> 一句话：**字段名用通配符省事但会误伤，前缀匹配可能把不该配的也配了，写全最稳**。

---

## 8. config_db 的 set/get 层级匹配规则

### 题目来源

- 合见工软 · 数字IC验证 · 校招 · 一面（config_db 中 set/get 操作的层级匹配规则）

### 考点

- set/get 匹配四要素（类型 / 路径 / 字段名 / 时间）
- 配置不生效的排查方法

### 参考答案

set 和 get 要能对上，需要**四个条件同时满足**，缺一个 get 就失败：

1. **参数化类型一致**——set 用 `uvm_config_db#(int)`，get 也必须 `#(int)`，类型不同相当于存在不同柜子里。
2. **路径要覆盖**——set 的目标路径必须覆盖 get 所在位置（要么是上级路径，要么指向自己）；路径拼写错误最常见，比如 i_agt 写成 i_atg，编译器不报错但永远取不到。
3. **字段名一致**——set 和 get 的 field name 必须一字不差，包括大小写。
4. **时间顺序**——get 之前必须已有 set 执行过，先寄后收。

排查"配置不生效"就按这四要素查：**先打印 `get_full_name()` 对照真实路径，再核对类型和字段名，然后确认 set 在 get 之前**，最后用 `check_config_usage()` 查"写了但没人读"的配置，或开 `+UVM_CONFIG_DB_TRACE` 追踪。

> 一句话：**类型、路径、字段名、时间，四个条件任一不满足 get 就失败——排查配置问题按这四要素逐个查**。

---

## 9. uvm_component 的生命周期

### 题目来源

- 乐鑫科技 · 数字IC验证 · 校招 · 笔试（uvm_component 生命周期）

### 考点

- component 生命周期与 phase 机制
- 构建阶段（build/connect）与运行阶段（run/check/report）
- component 与 object 生命周期对比

### 参考答案

component 的生命周期由 **UVM 的 phase 机制**管理，分**构建、运行、收尾**三个阶段：

**构建阶段（函数 phase，不耗时）**：build_phase 自顶向下创建组件树（test 建 env、env 建 agent/model/scoreboard、agent 建 driver/monitor），同时读取配置；connect_phase 自底向上连接端口（TLM 和 driver-sequencer 接线）；之后还有 end_of_elaboration（检查结构、打印拓扑）和 start_of_simulation（最后设置）。

**运行阶段（任务 phase，可耗时）**：包括 12 个 run-time phase，核心是 reset（复位）、configure（配置）、main（主要激励）、shutdown（收尾），main_phase 靠 objection 控制何时结束。

**收尾阶段（函数 phase）**：extract_phase（提取统计）→ check_phase（查残留、查遗漏）→ report_phase（汇总输出 PASS/FAIL）→ final_phase（清理，如关日志文件）。

**关键对比**：component 创建一次、贯穿仿真、受 phase 调度；object 用完即弃、没有 phase。

> 一句话：**component 生命周期 = build 建树 → connect 接线 → run 运行 → extract/check/report 收尾 → final 清理**。

---
