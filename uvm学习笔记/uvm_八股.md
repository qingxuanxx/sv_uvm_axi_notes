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
10. [UVM 的 phase 分为哪两大类？](#10-uvm-的-phase-分为哪两大类)
11. [UVM phase 的总体执行顺序与 run_test 的起点](#11-uvm-phase-的总体执行顺序与-run_test-的起点)
12. [run_phase 与 main_phase 的区别](#12-run_phase-与-main_phase-的区别)
13. [为什么 build_phase 是自顶向下执行？](#13-为什么-build_phase-是自顶向下执行)
14. [运行中检测到复位，如何跳回 reset_phase？（phase jump）](#14-运行中检测到复位如何跳回-reset_phase-phase-jump)
15. [UVM 的 objection 机制是什么？有什么用？](#15-uvm-的-objection-机制是什么有什么用)

---

## 1. uvm_object 与 uvm_component 的区别

### 题目来源

- 小米 · ASIC验证 · 实习（uvm_object 与 uvm_component 的区别）
- 平头哥 · 数字IC验证 · 校招 · 一面（uvm_object 与 uvm_component 的区别）
- 新凯来 · 数字IC验证 · 校招（uvm_object 与 uvm_component 的区别）
- 集益威 · 数字IC验证 · 校招 · 一面（component 与 object 的区别）

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

- 合见工软 · 数字IC验证 · 校招 · 一面（config_db 完成 set 后，若源变量发生变化，get 到的值是否同步更新）

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

- 合见工软 · 数字IC验证 · 校招 · 一面（uvm_info 的消息等级分类）

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

## 10. UVM 的 phase 分为哪两大类？

### 题目来源

- 中兴通讯 · 数字IC验证 · 校招 · 领军计划（耗时与不耗时 phase 的区分）
- 海光 · 数字IC验证 · 校招 · NoC方向（UVM 核心 phase 的分类与简要介绍）

### 考点

- function phase 与 task phase 的区别（是否耗仿真时间）
- 12 个 run-time phase 的归属与 4 段式结构

### 参考答案

UVM 的 phase 分两大类，分水岭是**能不能耗仿真时间**：

**function phase（函数，不耗时）**：瞬间完成，用来搭结构和收尾。包括构建段的 build、connect、end_of_elaboration、start_of_simulation，和收尾段的 extract、check、report、final。

**task phase（任务，可耗时）**：能等时钟、能耗时间，用来真正跑测试。包括 run_phase 和 12 个 run-time phase。

12 个 run-time phase 分 4 组，每组是"pre_* → 核心 → post_*"的包裹结构：reset（复位）、configure（配置）、main（主激励）、shutdown（收尾），顺序执行，全程与 run_phase 并行。

| 类别 | 是否耗时 | 包含 |
|------|----------|------|
| function phase | 否 | build/connect/end_of_elaboration/start_of_simulation/extract/check/report/final |
| task phase | 是 | run_phase + 12 个 run-time phase |

> 一句话：function 不耗时管搭建收尾，task 可耗时管真正运行；12 个 run-time phase 是 4 组三明治（pre+核心+post）。

---

## 11. UVM phase 的总体执行顺序与 run_test 的起点

### 题目来源

- 通用 · 数字IC验证 · 实习 · 基础面经（phase 机制：各 phase 执行顺序）
- 字节跳动 · 数字IC验证 · 实习 · 高频题（run_test 开始执行的是哪个 phase、UVM 中 phase 的完整分类与分组）
- 中兴通讯 · 数字IC验证 · 校招 · 领军计划（UVM Phase 机制的总体执行顺序）

### 考点

- 三段式顺序：构建 → 运行 → 收尾
- build 自顶向下、connect 自底向上
- run_test 的启动流程

### 参考答案

总体分三段，固定顺序执行：**构建 → 运行 → 收尾**。

构建段 4 个 function phase：build（自顶向下建树）→ connect（自底向上连线）→ end_of_elaboration（检查结构）→ start_of_simulation（仿真前最后设置）。运行段：run（run_phase 与 12 个 run-time phase 并行）。收尾段 4 个：extract（提取统计）→ check（查遗漏）→ report（汇总 PASS/FAIL）→ final（清理）。

`run_test("test")` 启动后：UVM 创建 test 对象并挂到 uvm_test_top，平台**从 build_phase 开始自顶向下**搭建组件树，之后依次走完 connect、run、check、report。

> 一句话：构建（build→connect→检查）→ 运行（run 并行 12 个动态 phase）→ 收尾（extract→check→report→final）；run_test 从 build 开始。

---

## 12. run_phase 与 main_phase 的区别

### 题目来源

- 小米 · 处理器验证 · 校招（run_phase 与 main_phase 的区别）
- 芯动科技 · 数字IC验证 · 校招（run_phase 与 main_phase 的区别）
- 小鹏汽车 · SOC验证 · 校招（区别；main_phase 写了 raise_objection 而 run_phase 没写，run_phase 能否正常运行）
- 某TPU公司 · 数字IC验证 · 校招（两者关系；main_phase 里 raise/drop objection 时 run_phase 是否继续执行）

### 考点

- run_phase 与 12 个 run-time phase 的并行关系
- 两条线各有独立 objection，互不干扰
- 进入 extract 前两条线都必须收工

### 参考答案

**run_phase 和 main_phase 是并行关系，不是先后关系**。run_phase 是一条总流水线，从 start_of_simulation 一直跑到 extract 之前；main_phase 只是 12 个 run-time phase 之一（主激励阶段），是精细流水线中间的一段。

两者各有各的 objection 计数，互不干扰：

**第一，只在 main_phase raise、run_phase 没写 objection**——run_phase 照常能运行。因为 run_phase 自己的计数一直是 0，它早就"完成"了；main 的 objection 只影响 main 这条线，两条线各自独立推进。

**第二，只在 run_phase raise**——12 个动态 phase 因为没人 raise objection 会被瞬间跳过，但 run_phase 还在跑，测试照样进行；等 run_phase 的 objection 归零才进 extract。

进入 extract 前，两条线必须都收工：12 个 run-time phase 全部走完，且 run_phase 完成或被终止。

> 一句话：run_phase 与 main_phase 并行不互等、各有各的 objection；进 extract 前两条线都得结束。

---

## 13. 为什么 build_phase 是自顶向下执行？

### 题目来源

- 面试书《Cracking Digital VLSI Verification Interview》· 验证方法学章（build_phase 在组件层次中自顶向下执行的原因）
- 豪威科技 · 数字IC验证 · 校招 · 一面（build_phase / main_phase / configure_phase 中 AXI VIP 和 APB VIP 的配置——应用型）

### 考点

- top-down 的因果必然：父组件在 build 里 create 子组件
- 与 connect bottom-up、task phase 并发的对比
- 漏创建组件的后果

### 参考答案

build_phase 自顶向下是**因果必然**，不是约定：phase 调度器调用某个组件 build 的前提，是它已经存在于树上；而组件上树靠**父组件在自己的 build_phase 里 create**。env.build 不执行，i_agt 就不存在，调度器永远不会调 i_agt.build。

漏创建或拼错名字的组件不在树上，它的 build 及之后所有 phase 静默不执行，后续引用得到 null，错误往往在很后面才暴露。

对比另外两类：**connect_phase 自底向上**（子组件端口先就绪，父组件才能做跨组件连接）；**task phase 并发运行**（按树顺序启动各组件 main，互不等待，全部启动后统一等 objection 归零）。

| phase 类型 | 执行方向 | 原因 |
|------------|----------|------|
| build | 自顶向下 | 父创建子，因果必然 |
| connect | 自底向上 | 父要用子的成品端口 |
| main 等 task | 并发 | fork 启动 + 统一等待 |

> 一句话：build 父先子后（父创建子）、connect 子先父后（父用子的成品）、task 并发（fork + 会合）。

---

## 14. 运行中检测到复位，如何跳回 reset_phase？（phase jump）

### 题目来源

- 思朗 · 数字IC验证 · 校招 · 面经（main_phase 运行中途需要触发复位时的跳转方式）

### 考点

- phase jump 的用法与影响范围
- 跳转的副作用处理与防死循环

### 参考答案

运行中 DUT 突然复位（比如 rst_n 拉低），可以用 `phase.jump` 让整个 schedule 跳回 reset_phase 重新执行，最常见的就是 `phase.jump(uvm_reset_phase::get())`。

典型做法是 main_phase 里用 fork 开两个并行进程：一个正常发激励，另一个专门盯 rst_n 的下降沿，一发现复位就调用 jump。

跳转有三个注意点：

**第一，影响整个 domain，不只是调用者。** jump 把当前 domain 里所有组件的动态 phase 一起拉回目标 phase，不是只跳自己。

**第二，要处理副作用。** 跳转时当前 phase 的并发进程被终止、没 drop 的 objection 会被清理并报 warning、driver 手里的 req 可能没 item_done、scoreboard 的期望队列和 FIFO 都是旧状态——所以跳回 reset 后，除了复位 DUT，还要清空平台自己的状态（清队列、清 FIFO）。

**第三，防止无限跳转。** jump 回 reset 后 main 会重新执行，如果每次进 main 都无条件 jump 就死循环了，要用一个标志位保证只跳一次。

> 一句话：**运行中复位用 phase.jump 跳回 reset 重新走——记得处理副作用、防死循环，且它影响整个 domain。**

---

## 15. UVM 的 objection 机制是什么？有什么用？

### 题目来源

- 面试书《Cracking Digital VLSI Verification Interview》· 验证方法学章（objection 是什么、有什么用——五星考点）
- 小鹏汽车 · SOC验证 · 校招（main_phase 写了 raise_objection 而 run_phase 没写，run_phase 能否正常运行）
- 某TPU公司 · 数字IC验证 · 校招（main_phase 里 raise/drop objection 时 run_phase 是否继续执行）

### 考点

- objection 的本质：task phase 的存活计数
- raise/drop 配对、计数归零才结束 phase
- 谁应该控制 objection（sequence 最佳）
- 与 run_phase 的独立计数关系

### 参考答案

objection 是 UVM 用来**控制 task phase 何时结束**的机制，本质是挂在 phase 上的一个**存活计数**：raise_objection 让计数加一，表示"我还有活没干完，phase 别结束"；drop_objection 让计数减一，表示"我干完了"。计数归零、同时所有 phase 进程都结束，phase 才放行。

用法标准三件套：干活前 raise、跑耗时的激励、干完 drop。

三个关键规则：

**第一，没人 raise 的 phase 会被瞬间跳过。** task phase 里写了耗时代码但没人 raise，UVM 可以在零时间立即结束它——所以耗时代码必须有人"保活"，这就是 trace 里 SKIP 的含义。

**第二，应该由最清楚测试边界的一方控制。** driver、monitor 通常是 forever 循环，不知道测试何时结束，不适合控制 objection；sequence 最清楚激励从哪开始、到哪结束，是 UVM 推荐的控制者。

**第三，run_phase 与动态 phase 各有独立计数。** run_phase 的 raise 只保护 run_phase 自己，不影响 main_phase，反过来也一样。所以"只在 main_phase raise、run_phase 没 raise"时，run_phase 照常能运行；两条线互不干扰，进 extract 前两条线都得归零。

> 一句话：**objection 是 task phase 的存活计数——举手干活、放手交班、计数归零才放行；由最清楚测试边界的 sequence 控制；run_phase 与动态 phase 各持各的证。**

---

