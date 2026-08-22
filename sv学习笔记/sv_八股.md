# SV 八股

> 面试常考的 SystemVerilog 八股积累（持续更新）。
>
> 阅读配套：`sv_ch2学习笔记.md`～`sv_ch9学习笔记.md`（SV 学习笔记）

## 目录

- [第2章 数据类型（Q1-Q6）](#第2章-数据类型q1-q6)
  - [1. wire、reg 和 logic 有什么区别？](#1-wirereg-和-logic-有什么区别)
  - [2. bit 与 logic 有什么区别？什么时候使用二态类型？](#2-bit-与-logic-有什么区别什么时候使用二态类型)
  - [3. packed 数组和 unpacked 数组有什么区别？](#3-packed-数组和-unpacked-数组有什么区别)
  - [4. 动态数组、队列和关联数组如何选择？](#4-动态数组队列和关联数组如何选择)
  - [5. 动态数组如何扩容并保留原数据？](#5-动态数组如何扩容并保留原数据)
  - [6. 关联数组适合什么场景？常用方法有哪些？](#6-关联数组适合什么场景常用方法有哪些)
- [第3章 过程语句和子程序（Q7-Q8）](#第3章-过程语句和子程序q7-q8)
  - [7. task 和 function 有什么区别？](#7-task-和-function-有什么区别)
  - [8. SystemVerilog 支持函数重载吗？如何实现相近需求？](#8-systemverilog-支持函数重载吗如何实现相近需求)
- [第4章 连接设计和测试平台（Q9-Q11）](#第4章-连接设计和测试平台q9-q11)
  - [9. interface 的作用是什么？](#9-interface-的作用是什么)
  - [10. modport 与 clocking block 分别解决什么问题？](#10-modport-与-clocking-block-分别解决什么问题)
  - [11. virtual interface 为什么需要 virtual？](#11-virtual-interface-为什么需要-virtual)
- [第6章 随机化（Q12-Q15）](#第6章-随机化q12-q15)
  - [12. 一次对象 randomize 的完整流程是什么？](#12-一次对象-randomize-的完整流程是什么)
  - [13. inside、dist 和 unique 分别做什么？](#13-insidedist-和-unique-分别做什么)
  - [14. solve before 会不会改变合法解集合？](#14-solve-before-会不会改变合法解集合)
  - [15. 约束动态数组时有哪些要点？](#15-约束动态数组时有哪些要点)
- [第7章 线程以及线程间通信（Q16-Q20）](#第7章-线程以及线程间通信q16-q20)
  - [16. fork...join、join_any 和 join_none 有什么区别？](#16-forkjoinjoin_any-和-join_none-有什么区别)
  - [17. wait fork 与 disable fork 分别做什么？](#17-wait-fork-与-disable-fork-分别做什么)
  - [18. 为什么 for 循环里的 fork 经常打印出相同的值？](#18-为什么-for-循环里的-fork-经常打印出相同的值)
  - [19. semaphore 的作用是什么？](#19-semaphore-的作用是什么)
  - [20. mailbox 的作用是什么？有界和无界 mailbox 有何区别？](#20-mailbox-的作用是什么有界和无界-mailbox-有何区别)
- [第8章 面向对象编程高级技巧（Q21）](#第8章-面向对象编程高级技巧q21)
  - [21. 基类句柄指向派生类对象时，virtual 有什么作用？](#21-基类句柄指向派生类对象时virtual-有什么作用)
- [第9章 功能覆盖率（Q22-Q27）](#第9章-功能覆盖率q22-q27)
  - [22. 代码覆盖率和功能覆盖率有什么区别？](#22-代码覆盖率和功能覆盖率有什么区别)
  - [23. covergroup、coverpoint、bin 和 cross 的关系是什么？](#23-covergroupcoverpointbin-和-cross-的关系是什么)
  - [24. ignore_bins、illegal_bins、wildcard bins 和 transition bins 分别是什么？](#24-ignore_binsillegal_binswildcard-bins-和-transition-bins-分别是什么)
  - [25. cross 覆盖率为什么容易组合爆炸？如何优化？](#25-cross-覆盖率为什么容易组合爆炸如何优化)
  - [26. 高代码覆盖率是否代表高功能覆盖率？两者不一致说明什么？](#26-高代码覆盖率是否代表高功能覆盖率两者不一致说明什么)
  - [27. 功能覆盖率应在什么时机采样？per-instance 有什么作用？](#27-功能覆盖率应在什么时机采样per-instance-有什么作用)

---

## 第2章 数据类型（Q1-Q6）

### 1. wire、reg 和 logic 有什么区别？

**题目来源**

- 平头哥 · 数字IC验证 · 校招 · 一面（wire、reg、logic 三者的区别）
- 集益威 · 数字IC验证 · 校招 · 一面（logic、reg、wire 三者的区别）
- 面试书《Cracking Digital VLSI Verification Interview》· 第4章 Q18（对应题目：reg、wire 与 logic 的区别）

**考点**

- net 与 variable 的本质区别
- `logic` 能否替代 `wire`
- 多驱动和 `inout` 的限制

**参考答案**

三类信号类型的核心区别是：**wire 是网络类型、管连接，reg 和 logic 是变量类型、管过程赋值**；`logic` 能替代 `reg`，但替代不了需要多驱动的 `wire`。

**第一，wire 是网络类型。** 它表示"连接"关系：连续赋值、模块端口、多个驱动源都可以驱动它，它自己不保存过程赋值的结果。

**第二，reg 是变量类型。** 它在 `always`、`initial` 这些过程块里赋值。注意名字叫 reg，不代表综合出来一定是寄存器，也可能综合成普通连线。

**第三，logic 是四态变量。** 平时用它替代 `reg`，也能由一根连续赋值驱动（单驱动）；但它本质是变量，多个驱动源同时驱动会冲突。

| 类型 | 类别 | 过程赋值 | 连续赋值 | 多驱动解析 |
|---|---|---|---|---|
| `wire` | net | 否 | 是 | 是 |
| `reg` | variable | 是 | 否 | 否 |
| `logic` | variable | 是 | 单驱动时可以 | 否 |

**inout 的限制**：双向端口通常必须声明成 net 类型（`wire`）；信号确实可能有多个驱动源时也一样，用 `wire`，不能图省事用 `logic`。

> 一句话：**wire 管连接、支持多驱动，reg/logic 管变量；logic 替代 reg 没问题，但多驱动和 inout 必须用 wire。**

---

### 2. bit 与 logic 有什么区别？什么时候使用二态类型？

**题目来源**

- 泰凌微 · 数字IC验证 · 校招 · 一面（logic 与 bit 的核心区别）
- 面试书《Cracking Digital VLSI Verification Interview》· 第4章 Q19（对应题目：bit 与 logic 数据类型）

**考点**

- 二态与四态
- X/Z 检测
- 仿真性能与语义正确性的取舍

**参考答案**

**bit 是二态，只有 0 和 1；logic 是四态，有 0、1、X、Z。** 核心区别是：四态值赋给二态变量时，X/Z 会被转成 0，可能把复位遗漏、多驱动这类问题悄悄掩盖掉。

**第一，表示能力。** bit 只存 0 和 1；logic 保留 X 和 Z，能反映硬件的未知态。

**第二，使用位置。** 连接 DUT、采样值、要检查未知态的信号，用 `logic`；确定不会出现 X/Z 的抽象数据，比如计数器、配置开关，用 `bit`，省内存、仿真也更快。

**第三，查未知态。** 想检查一个向量里有没有 X/Z，用 `$isunknown(bus_data)`，有就报错。

> 一句话：**bit 快但会吞掉 X/Z，logic 能保留硬件未知态；连接 DUT 的信号优先用 logic。**

---

### 3. packed 数组和 unpacked 数组有什么区别？

**题目来源**

- 合见工软 · 数字IC验证 · 校招 · 一面（packed 数组与 unpacked 数组的核心区别）
- 面试书《Cracking Digital VLSI Verification Interview》· 第4章 Q26（对应题目：packed 数组与 unpacked 数组）

**考点**

- 声明位置与内存布局
- 整体位运算和逐元素访问
- 混合数组维度顺序

**参考答案**

**packed 是连续位向量，unpacked 是元素集合。** 分界线在变量名：写在名字**左边**的是 packed 维度，写在**右边**的是 unpacked 维度。

**第一，packed 维度。** 所有 bit 紧密排成一段连续向量，可以整体做算术、位运算、切片和类型转换。

**第二，unpacked 维度。** 是一组独立元素，适合建模存储器、表格、对象集合。

**第三，混合数组。** 比如 `bit [31:0] mem [0:255]`，`mem[3][7:0]` 是先用右边的 unpacked 下标选第 3 个 word，再用左边的 packed 下标选低 8 位。

```systemverilog
bit [7:0] packed_data;          // 一个 8 bit 向量
bit       unpacked_data [8];    // 8 个独立 bit 元素
bit [31:0] mem [0:255];         // 256 个 32 bit word
```

packed 数组必须由可打包的位类型组成，类句柄不能放进去。

> 一句话：**左边 packed 是连续位向量，右边 unpacked 是元素集合；读混合数组先看右边选元素，再看左边选位。**

---

### 4. 动态数组、队列和关联数组如何选择？

**题目来源**

- 字节跳动 · 数字IC验证 · 校招 · SOC方向（动态数组、关联数组、队列三者的区别）
- 泰凌微 · 数字IC验证 · 校招 · 一面（动态数组与队列的区别）
- 知存科技 · 数字IC验证 · 校招（动态数组、队列、关联数组）
- 面试书《Cracking Digital VLSI Verification Interview》· 第4章 Q21（对应题目：大数组建模：动态数组 vs 关联数组）

**考点**

- 三类动态容器的分配方式
- 常用操作和使用场景

**参考答案**

**三种动态容器怎么选，看三件事：索引连不连续、大小变不变、要不要首尾操作。**

**第一，动态数组。** 用 `new[N]` 整体分配，长度定了之后适合大量随机访问。

**第二，队列。** 自动伸缩，`push_back`、`pop_front` 很方便，适合 FIFO、频繁首尾插删。

**第三，关联数组。** 用哪个键才分配空间，适合稀疏地址、ID 映射；常用方法有 `exists`、`first`、`next`、`delete`。

| 容器 | 下标 | 大小 | 典型用途 |
|---|---|---|---|
| 动态数组 `a[]` | 连续整数 | 用 `new[N]` 整体分配 | 长度偶尔变化、之后大量随机访问 |
| 队列 `q[$]` | 连续整数 | 自动伸缩 | FIFO、待比较事务、频繁首尾插删 |
| 关联数组 `aa[index_t]` | 整数、枚举、字符串等 | 用到哪个键才分配 | 稀疏地址空间、ID 到事务的映射 |

> 一句话：**连续变长用动态数组，频繁进出队用队列，稀疏键值映射用关联数组。**

---

### 5. 动态数组如何扩容并保留原数据？

**题目来源**

- 泰凌微 · 数字IC验证 · 校招 · 一面（动态数组初始化扩容方法）
- 面试书《Cracking Digital VLSI Verification Interview》· 第4章 Q29（对应题目：动态数组扩容并保留原值）

**考点**

- `new[]` 的动态数组语义
- 旧数组作为复制源

**参考答案**

**动态数组扩容想保留旧数据，必须把旧数组作为 `new[]` 的第二个参数。** 只写 `a = new[N]` 等于重新开一块数组，旧数据不保证保留。

```systemverilog
int a[] = '{10, 20, 30};
a = new[5](a);
// a[0:2] 保留 10、20、30，新元素使用默认值 0
```

**两个注意点**：新长度比旧长度小时，只复制放得下的前几个元素；`new[N]` 是动态数组专用构造，类对象用 `new()`，两者不是一回事，别搞混。

> 一句话：**动态数组保留数据扩容写 `new[new_size](old_array)`；省略复制源就相当于重新建一块数组。**

---

### 6. 关联数组适合什么场景？常用方法有哪些？

**题目来源**

- 字节跳动 · 数字IC验证 · 校招 · 一面（关联数组的使用场景、核心优势与内置函数）
- 面试书《Cracking Digital VLSI Verification Interview》· 第4章 Q21（对应题目：大数组建模：动态数组 vs 关联数组）

**考点**

- 稀疏存储
- 键是否存在的安全检查
- 遍历方法

**参考答案**

**关联数组只为实际用到的键分配空间**，特别适合超大但稀疏的地址空间、寄存器镜像、outstanding 表、字符串配置表。

**第一，核心优势。** 键可以是整数、枚举或字符串，内存按需分配，稀疏的大空间也不会爆内存。

**第二，使用习惯。** 读取前先 `exists()`——对不存在的键读取会得到默认值，容易把"没有这一项"和"这一项是零"搞混。

**第三，常用方法。** `num()`、`exists()`、`first()`、`last()`、`next()`、`prev()`、`delete()`。

```systemverilog
if (pending.exists(addr))
  pending.delete(addr);
```

> 一句话：**关联数组是稀疏键值表；最重要的习惯是读取前先 exists()，遍历用 first/next。**

---

## 第3章 过程语句和子程序（Q7-Q8）

### 7. task 和 function 有什么区别？

**题目来源**

- 小鹏汽车 · SOC验证 · 校招（function 和 task 的区别）
- 某TPU公司 · 数字IC验证 · 校招（function 与 task 的区别）
- 豪威科技 · 数字IC验证 · 校招 · 一面（task vs function：区别 + 参数输入类型）
- 通用 · 数字IC验证 · 实习 · 基础面经（function 与 task 的区别）
- 面试书《Cracking Digital VLSI Verification Interview》· 第4章 Q17（对应题目：task 与 function 的区别）

**考点**

- 是否允许消耗仿真时间
- 返回值与参数
- 调用方向限制

**参考答案**

**function 做零仿真时间的计算，task 做可能等待的行为**，分水岭就是能不能耗仿真时间。

**第一，function 不能耗时。** 不能有 `#`、`@`、`wait` 这些耗时语句，也不能调用可能耗时的 task；它可以返回一个值，或者声明成 `void`。

**第二，task 可以耗时。** 可以有多个 `output`、`inout`、`ref` 参数，但没有函数式返回值。

| 对比项 | function | task |
|---|---|---|
| 是否可耗时 | 不可以 | 可以 |
| 返回值 | 一个，或 `void` | 无函数返回值，可用输出参数 |
| 调用关系 | 可调用 function | 可调用 task 和 function |
| 适合场景 | 计算、转换、查询 | 驱动、等待、协议操作 |

**易错点**："function 零时间"指不推进仿真时间，不等于它不消耗 CPU 运行时间。

> 一句话：**function 做零仿真时间计算，task 做可能等待的行为；function 不能调用耗时 task。**

---

### 8. SystemVerilog 支持函数重载吗？如何实现相近需求？

**题目来源**

- 合见工软 · 数字IC验证 · 校招 · 一面（函数重载 function overloading 的实现方式）
- 面试书《Cracking Digital VLSI Verification Interview》· 第3章 Q44（对应题目：方法重载 Overloading 与方法重写 Overriding）

**考点**

- 用户自定义函数重载的限制
- 默认参数、按名传参和参数化子程序

**参考答案**

**SystemVerilog 不支持 C++ 那种"同名函数、不同参数列表"的重载**——面试先把这个限制说出来，别直接写两个同名函数。

**第一，限制。** 同一作用域不能定义多个同名、不同参数的用户函数。

**第二，相近需求怎么做。** 用不同函数名、默认参数、按名传参，或者参数化类/参数化子程序实现。

```systemverilog
function int calc(int a, int b = 1);   // 默认参数
  return a + b;
endfunction

int result = calc(.b(4), .a(3));       // 按名传参
```

**第三，细节。** 按名传参能避免参数顺序写错，默认值适合稳定的可选配置；C++ 的运算符重载在 SystemVerilog 里同样不支持，别混为一谈。

> 一句话：**SV 没有 C++ 式同名函数重载；用不同名字、默认参数、按名传参实现相近需求。**

---

## 第4章 连接设计和测试平台（Q9-Q11）

### 9. interface 的作用是什么？

**题目来源**

- 泰凌微 · 数字IC验证 · 校招 · 一面（interface 的作用与使用场景）
- 面试书《Cracking Digital VLSI Verification Interview》· 第4章 Q41（对应题目：接口 interface）

**考点**

- 信号封装与连接复用
- interface 中可包含的内容
- 与 class testbench 的桥接

**参考答案**

**interface 把一组相关信号和时序规则封装成一个接口实例**，减少模块端口数量和重复连接。

**第一，封装信号。** 一组信号集中定义，DUT 和 testbench 共用一个 interface 实例连接。

**第二，还能装更多。** 除了信号，interface 里还能放 clocking block、modport、task/function、断言和覆盖率。

```systemverilog
interface bus_if(input logic clk);
  logic valid, ready;
  logic [31:0] data;
endinterface
```

**第三，与 class 的桥接。** interface 是静态结构，仿真开始前（elaboration 阶段）就创建好；class 是运行时才创建的对象，不能直接持有 interface 实例，所以类环境用 `virtual interface` 句柄访问它。

> 一句话：**interface 把协议的信号、方向和时序集中封装，是 DUT 静态端口与类化验证平台之间的边界。**

---

### 10. modport 与 clocking block 分别解决什么问题？

**题目来源**

- 某TPU公司 · 数字IC验证 · 校招（clocking block 的作用与意义）
- 面试书《Cracking Digital VLSI Verification Interview》· 第4章 Q42、Q44（对应题目：modport 构造；clocking block 时钟块）

**考点**

- 访问方向与时序语义的区别
- driver 和 monitor 视角

**参考答案**

**modport 管"谁能以什么方向访问"，clocking block 管"什么时候采、什么时候驱"**——一个管权限方向，一个管时序语义。

**第一，modport。** 给 interface 的不同使用方规定哪些成员可见、信号方向是什么，DUT、driver、monitor 各看各的视角。

**第二，clocking block。** 规定在哪个时钟事件采样或驱动，以及输入输出 skew 是多少。

```systemverilog
interface bus_if(input logic clk);
  logic valid, ready;
  logic [31:0] data;

  clocking drv_cb @(posedge clk);
    output valid, data;
    input  ready;
  endclocking

  modport DUT(input clk, valid, data, output ready);
endinterface
```

**注意**：modport 不解决竞争冒险，clocking block 也不是权限系统；两者各管各的，可以组合使用。

> 一句话：**modport 管"谁能以什么方向访问"，clocking block 管"什么时候采、什么时候驱"。**

---

### 11. virtual interface 为什么需要 virtual？

**题目来源**

- 字节跳动 · 数字IC验证 · 实习 · 高频题（virtual interface 的作用，相关差异，没有 virtual interface 能否实现对应功能）
- 面试书《Cracking Digital VLSI Verification Interview》· 第4章 Q81（对应题目：虚拟接口 virtual interface）

**考点**

- 静态 interface 与动态 class 的连接
- 句柄绑定

**参考答案**

**virtual interface 是类对象访问静态 interface 实例的句柄。**

**第一，为什么需要。** interface 是静态层次，仿真开始前就建好；class 对象是运行时才创建的，没法直接持有 interface 实例。没有 virtual interface，driver、monitor 这些类就读不到 DUT 信号。

**第二，本质。** 它是指向真实 interface 实例的句柄，本身不创建硬件接口；使用前必须绑定实际实例，句柄是 null 时访问成员会报错。

**第三，还能带 modport。** virtual interface 可以带 modport 类型，限制 driver/monitor 的访问视角。

```systemverilog
class driver;
  virtual bus_if vif;

  function new(virtual bus_if vif);
    this.vif = vif;
  endfunction
endclass
```

> 一句话：**virtual interface 是类对象访问静态 interface 实例的句柄，提供运行时绑定和解耦，本身不是新的接口实例。**

---

## 第6章 随机化（Q12-Q15）

### 12. 一次对象 randomize 的完整流程是什么？

**题目来源**

- 泰凌微 · 数字IC验证 · 校招 · 一面（随机化完整流程：类内声明 rand 属性、类外调用 randomize 的执行逻辑）
- 面试书《Cracking Digital VLSI Verification Interview》· 第4章 Q52（对应题目：pre_randomize() 与 post_randomize()）

**考点**

- `pre_randomize`、求解、`post_randomize`
- 失败时的行为

**参考答案**

**一次 randomize 是"前处理 → 约束求解 → 后处理"三步，用返回值报告成功还是失败。**

**第一，pre_randomize。** 在求解前调用，适合开关变量或约束。

**第二，post_randomize。** 只有求解成功才调用，适合算派生字段（比如 CRC），别再偷偷改坏已经满足的约束。

**第三，失败行为。** 随机化失败时目标变量保持原值，必须检查返回值——不检查的话，旧值会伪装成新的随机结果。

```systemverilog
if (!pkt.randomize())
  $fatal(1, "packet randomization failed");
```

> 一句话：**randomize 是前处理—求解—后处理，并用返回值报告成功；不检查失败会让旧值伪装成新随机结果。**

---

### 13. inside、dist 和 unique 分别做什么？

**题目来源**

- 泰凌微 · 数字IC验证 · 校招 · 一面（常用约束关键字：inside、unique、solve before 的用法）
- 豪威科技 · 数字IC验证 · 校招 · 一面（constraint 中 dist 的几种权重约束）
- 面试书《Cracking Digital VLSI Verification Interview》· 第4章 Q49（对应题目：unique 约束）

**考点**

- 值集合、概率权重和唯一性
- `:=` 与 `:/`

**参考答案**

**inside 定合法集合，dist 调合法解的概率，unique 保证互不重复**——三个约束关键字各管一件事。

**第一，inside。** 限制变量属于某个集合或范围。

**第二，dist。** 在合法解之间设概率权重。`:=` 给范围内每个值相同权重，`:/` 把权重分摊给整个范围；权重只改变概率，不会让权重为正的值变成必然出现。

**第三，unique。** 要求数组元素或变量列表互不相同。

```systemverilog
constraint c {
  addr inside {[16'h1000:16'h1fff], 16'h3000};
  kind dist {READ := 3, WRITE := 1};
  unique {id_q};
}
```

> 一句话：**inside 定合法集合，dist 调合法解的概率，unique 保证互不重复；`:=` 按值给权，`:/` 按范围分权。**

---

### 14. solve before 会不会改变合法解集合？

**题目来源**

- 燧原科技 · 数字IC验证 · 校招 · AI方向（solve before 对约束随机的影响）
- 泰凌微 · 数字IC验证 · 校招 · 一面（常用约束关键字：inside、unique、solve before 的用法）
- 面试书《Cracking Digital VLSI Verification Interview》· 第4章 Q48（对应题目：solve...before 与概率分布）

**考点**

- 求解顺序与概率分布
- 不改变约束合法性

**参考答案**

**solve before 调整"先随机谁"，主要影响概率分布，通常不改变合法解集合。**

**第一，作用。** `solve a before b` 让求解器先给 a 选值，再在 a 的条件下给 b 选合法值。

**第二，为什么用。** 举个典型例子：`mode` 是控制变量，`mode==0` 时数据有 100 种合法解，`mode==1` 时只有 10 种。不写 solve before 时，求解器在"所有 (mode, data) 组合"里均匀选，`mode==0` 出现的概率约是 `mode==1` 的 10 倍；写了 `solve mode before data` 后，先让 `mode` 均匀二选一（约各 50%），再在选定的 mode 下随机 data——控制类别的分布就被"抹平"了。

**第三，什么时候没用。** 如果 a、b 之间没有约束关系，或者 a 已经被外部固定，`solve before` 通常没有实际效果。

> 一句话：**solve before 调整"先随机谁"，主要影响概率而不改变哪些组合合法。**

---

### 15. 约束动态数组时有哪些要点？

**题目来源**

- 字节跳动 · 数字IC验证 · 校招 · SOC方向（动态数组大小的约束方式）
- 面试书《Cracking Digital VLSI Verification Interview》· 第4章 Q53-Q56（对应题目：约束动态数组、降序数组、随机唯一数组、$countones 约束）

**考点**

- `size()`、`foreach`、`unique`、`$countones`
- 越界保护

**参考答案**

**约束动态数组要先定 size，再用 foreach 限元素。** 不定长度时，求解器可能选空数组，所有元素约束就"真空成立"了。

```systemverilog
class item;
  rand byte unsigned data[];
  rand bit [9:0] mask;

  constraint c {
    data.size() inside {[4:16]};        // 先定长度
    foreach (data[i]) {
      data[i] inside {[1:100]};
      if (i > 0) data[i] > data[i-1];   // 邻项递增：先保护下标
    }
    $countones(mask) == 3;              // 恰好 3 个 1，位置随机
  }
endclass
```

**三个要点**：先定 size；邻项关系先保护下标（访问 `i-1` 要加 `i > 0`）；固定 1 的个数用 `$countones`。严格递增本身已经保证不重复，不用再写 `unique`。

> 一句话：**数组约束先定 size，再用 foreach 限元素；邻项关系先保护下标，固定位数用 $countones。**

---

## 第7章 线程以及线程间通信（Q16-Q20）

### 16. fork...join、join_any 和 join_none 有什么区别？

**题目来源**

- 泰凌微 · 数字IC验证 · 校招 · 一面（三类 fork join 语句的区别）
- 知存科技 · 数字IC验证 · 校招（三种 fork join / fork join_any / fork join_none 的区别）
- 集益威 · 数字IC验证 · 校招 · 一面（fork join / join_any / join_none 的区别）
- 达摩院 · 数字IC验证 · 校招 · 一面（三类 fork join 语句的区别）
- 面试书《Cracking Digital VLSI Verification Interview》· 第4章 Q57（对应题目：fork-join / fork-join_any / fork-join_none）

**考点**

- 父线程恢复条件
- 子线程启动时机
- 未结束线程的处理

**参考答案**

**join 等全部，join_any 等一个但不杀其余，join_none 一个也不等**——区别在父线程什么时候继续。

**第一，fork...join。** 所有子线程结束，父线程才继续。

**第二，fork...join_any。** 任意一个子线程完成，父线程就继续；剩余线程还在后台跑，**不会自动被杀**。

**第三，fork...join_none。** 一个也不等，父线程继续；子线程要等父线程挂起或终止后才开始执行，所以紧跟其后的零时间代码可能先于子线程执行。

| 形式 | 父线程何时继续 | 未完成子线程 |
|---|---|---|
| `fork...join` | 所有子线程结束 | 无 |
| `fork...join_any` | 任意一个子线程结束 | 继续后台运行 |
| `fork...join_none` | 执行到后续阻塞语句时继续 | 全部后台运行 |

> 一句话：**join 等全部，join_any 等一个但不杀其余，join_none 一个也不等；不要把父线程继续和子线程终止混为一谈。**

---

### 17. wait fork 与 disable fork 分别做什么？

**题目来源**

- 泰凌微 · 数字IC验证 · 校招 · 一面（挂起后阻塞等待所有线程结束的实现方式）
- 达摩院 · 数字IC验证 · 校招 · 一面（终止 fork join_none 创建的进程的方法）
- 面试书《Cracking Digital VLSI Verification Interview》· 第4章 Q58（对应题目：wait fork 与 disable fork）

**考点**

- 等待所有后代线程
- 终止线程作用域
- guard fork 模式

**参考答案**

**wait fork 是等完所有后代线程，disable fork 是杀掉所有活动后代线程。**

**第一，wait fork。** 阻塞当前线程，直到它创建的所有子线程完成，适合多个 `join_none` 之后统一收口。

**第二，disable fork。** 立即终止当前线程的所有活动后代；但作用范围可能超预期——当前线程之前启动的其他后台任务也会一起被杀。

**第三，guard fork。** 工程上包一层隔离，让 `disable fork` 只清理竞争线程：

```systemverilog
fork begin : guard
  fork
    do_work();
    timeout();
  join_any
  disable fork;
end join
```

> 一句话：**wait fork 是等完所有后代，disable fork 是杀掉所有活动后代；用 guard fork 限制误杀范围。**

---

### 18. 为什么 for 循环里的 fork 经常打印出相同的值？

**题目来源**

- 豪威科技 · 数字IC验证 · 校招 · 一面（fork_join_none 代码题，分析输出结果）
- 面试书《Cracking Digital VLSI Verification Interview》· 第4章 Q60、Q61（对应题目：线程中的共享变量陷阱；fork 内嵌 for 循环产生几个并行进程）

**考点**

- 循环变量共享
- automatic 快照
- join_none 延迟启动

**参考答案**

**for 循环里 fork 的线程共享同一个循环变量，加上 join_none 延迟启动，最后多个线程打印出同一个最终值。**

**第一，原因。** 线程共享循环变量 `i`；`join_none` 子线程要等父线程挂起才运行，那时循环已经结束，所有线程看到的都是 `i` 的最终值。

**第二，解决。** 每轮循环创建 automatic 局部变量保存快照：

```systemverilog
for (int i = 0; i < 3; i++) begin
  automatic int j = i;     // 每轮进入块时先保存快照
  fork
    $display("%0d", j);
  join_none
end
wait fork;
```

> 一句话：**fork 分支若直接引用循环变量，读到的是共享变量的后续值；每轮用 automatic 局部变量捕获快照。**

---

### 19. semaphore 的作用是什么？

**题目来源**

- 字节跳动 · 数字IC验证 · 实习 · 高频题（旗语 semaphore 的含义与典型使用场景）
- 昆仑芯 · 数字IC验证 · 校招 · 一面（什么是 semaphore）
- 集益威 · 数字IC验证 · 校招 · 一面（是否使用过旗语 semaphore）
- 面试书《Cracking Digital VLSI Verification Interview》· 第4章 Q66（对应题目：信号量 semaphore）

**考点**

- 资源令牌
- 互斥与有限并发
- 阻塞和非阻塞获取

**参考答案**

**semaphore 用令牌（key）控制共享资源**——1 个 key 是互斥锁，多个 key 是并发配额。

**第一，获取。** `get(n)` 拿 n 个 key，不够就阻塞；`try_get(n)` 不够时立即返回 0。

**第二，归还。** 用完必须 `put(n)` 归还，数量要和获取一致。

**第三，典型用法。** `new(1)` 相当于互斥锁，`new(N)` 限制最大并发 N 个；常见错误是归还数量不一致、异常路径忘归还，导致其他线程永久阻塞。

```systemverilog
semaphore bus_lock = new(1);

bus_lock.get(1);
drive_bus();
bus_lock.put(1);
```

> 一句话：**semaphore 用令牌控制共享资源，1 个 key 是互斥锁，多个 key 是并发配额。**

---

### 20. mailbox 的作用是什么？有界和无界 mailbox 有何区别？

**题目来源**

- 小鹏汽车 · IP验证 · 校招（mailbox 的定义与典型使用场景）
- 面试书《Cracking Digital VLSI Verification Interview》· 第4章 Q67、Q68（对应题目：信箱 mailbox；有界与无界 mailbox）

**考点**

- 线程安全消息传递
- 阻塞/非阻塞接口
- 参数化类型

**参考答案**

**mailbox 是线程安全的消息队列**，常用于 generator 给 driver 传 transaction。

**第一，阻塞与非阻塞。** `put()` 在有界 mailbox 已满时阻塞，`get()` 在空时阻塞；`try_put`、`try_get`、`try_peek` 是非阻塞方法。

**第二，有界 vs 无界。** `new(8)` 是容量 8 的有界 mailbox，可以对生产者形成背压；不带容量或容量为 0 是无界 mailbox。

```systemverilog
mailbox #(packet) mbx = new(8); // 容量 8
mbx.put(pkt);
mbx.get(pkt);
```

**第三，传对象传句柄。** mailbox 传 class transaction 传的是句柄；生产者如果复用并修改同一个对象，消费者会看到被覆盖的数据，所以发送前通常要建新对象或深复制。最好用参数化 mailbox，避免类型不匹配。

> 一句话：**mailbox 可靠传消息，有界版本还能提供背压；传对象时传的是句柄，注意不要复用后篡改。**

---

## 第8章 面向对象编程高级技巧（Q21）

### 21. 基类句柄指向派生类对象时，virtual 有什么作用？

**题目来源**

- 小鹏汽车 · SOC验证 · 校招（virtual function/task 未声明 virtual 会出现什么问题）
- 芯动科技 · 数字IC验证 · 校招（讲解多态时结合 virtual 关键字说明子类重写）
- 面试书《Cracking Digital VLSI Verification Interview》· 第4章 Q74-Q76（对应题目：派生类覆盖是否必须加 virtual；基类句柄指向派生类对象；virtual 函数的调用顺序）

**考点**

- 静态类型与动态类型
- 方法重写和动态绑定

**参考答案**

**基类句柄决定"能看见哪些成员"，virtual 决定"运行时执行哪个类的方法"。**

**第一，静态类型 vs 动态类型。** `b = c` 赋值是合法的，因为 child "是一个" base；但句柄 `b` 的静态类型还是 base，只能访问 base 声明过的成员。

**第二，virtual 的作用。** 如果 base 的 `print()` 声明成 `virtual` 且 child 重写了它，`b.print()` 按对象动态类型执行 child 版本；没声明 virtual 就按句柄静态类型执行 base 版本——这就是"没写 virtual 会出现什么问题"的答案。

```systemverilog
base b;
child c = new();
b = c;
b.print();
```

**第三，重写要求。** 派生类重写 virtual 方法要保持兼容签名；通常基类声明 virtual 后派生类就保持虚方法语义，但显式写出来更利于阅读。

> 一句话：**基类句柄决定"能看见哪些成员"，virtual 决定"运行时执行哪个类的方法"。**

---

## 第9章 功能覆盖率（Q22-Q27）

### 22. 代码覆盖率和功能覆盖率有什么区别？

**题目来源**

- 小鹏汽车 · IP验证 · 校招（代码覆盖率与功能覆盖率的分类，是否做过断言验证）
- 芯原 · 数字IC验证 · 校招（功能覆盖率与代码覆盖率的区别）
- 面试书《Cracking Digital VLSI Verification Interview》· 第6章 Q100（对应题目：代码覆盖率与功能覆盖率的区别）

**考点**

- 覆盖对象和产生方式
- 两类覆盖率的互补关系
- 断言覆盖率

**参考答案**

**代码覆盖率回答"哪些代码被执行过"，功能覆盖率回答"计划里的功能场景是否出现过"**——一个自动统计，一个主动定义。

**第一，代码覆盖率。** 由仿真器从 RTL 结构自动统计，常见类型有语句、分支、条件、翻转、FSM。

**第二，功能覆盖率。** 由验证人员根据 spec 和验证计划主动定义，回答计划中的功能、场景、组合是否出现过。

**第三，断言覆盖率。** 统计 property 是否被触发、成功或失败，关注时序行为是否真正尝试过。

代码覆盖率不能证明功能正确，功能覆盖率也不会自动发现遗漏代码——两者要结合测试结果、断言和 bug 状态一起判断。

> 一句话：**代码覆盖率看 RTL 执行痕迹，功能覆盖率看验证计划目标，断言覆盖率看时序属性是否被尝试和满足。**

---

### 23. covergroup、coverpoint、bin 和 cross 的关系是什么？

**题目来源**

- 字节跳动 · 数字IC验证 · 校招 · SOC方向（功能覆盖率 fcov 的写法：bins / coverpoint / covergroup；cross 交叉覆盖率的实现）
- 某公司 · 数字IC验证 · 实习 · 一面（covergroup 所使用的 item 来源、内部成员变量；现场手写 coverpoint）
- 达摩院 · 数字IC验证 · 校招（covergroup 相关代码编写）
- 面试书《Cracking Digital VLSI Verification Interview》· 第6章 Q104、Q106（对应题目：功能覆盖率在 SV 中的两种构造；coverpoint 和 bins）

**考点**

- 功能覆盖率层级
- 采样事件
- 交叉覆盖

**参考答案**

**covergroup 是容器，coverpoint 是观测对象，bin 是目标分桶，cross 是分桶组合**——四级结构从大到小。

**第一，covergroup。** 一组覆盖模型的容器，定义后必须 `new()` 实例化。

**第二，coverpoint。** 指定要采样的变量或表达式。

**第三，bin。** 把取值或转换划分成可统计的目标。

**第四，cross。** 对多个 coverpoint 的 bins 做组合覆盖。

```systemverilog
covergroup bus_cg with function sample(bit write, bit [1:0] size);
  cp_write : coverpoint write;
  cp_size  : coverpoint size;
  wr_x_sz  : cross cp_write, cp_size;
endgroup

bus_cg cg = new();
cg.sample(tr.write, tr.size);
```

**采样时机**：在 transaction 完整、语义有效时采样，别机械地每拍采样未完成的总线信号；类内 covergroup 必须在类构造函数里实例化。

> 一句话：**covergroup 是容器，coverpoint 是观测对象，bin 是目标分桶，cross 是多个分桶的组合。**

---

### 24. ignore_bins、illegal_bins、wildcard bins 和 transition bins 分别是什么？

**题目来源**

- 字节跳动 · 数字IC验证 · 校招 · SOC方向（无效 bins 的定义：ignore_bins / illegal_bins）
- 面试书《Cracking Digital VLSI Verification Interview》· 第6章 Q108、Q109、Q112（对应题目：ignore_bins 与 illegal_bins；transition bins；wildcard bins）

**考点**

- 排除值、非法值、通配值和跨采样转换

**参考答案**

**ignore 排除统计，illegal 命中报错，wildcard 合并不关心位，transition 覆盖跨采样序列**——四种 bins 各管一类需求。

**第一，ignore_bins。** 合法但不关心的取值，不进入覆盖率分母。

**第二，illegal_bins。** 声明不允许出现的取值，命中时工具报告错误。

**第三，wildcard bins。** bin 描述里的 `x/z/?` 当作 0/1 通配符，合并不关心的位。

**第四，transition bins。** 用 `=>` 描述连续采样点上的值转换。

```systemverilog
coverpoint state {
  ignore_bins reserved = {3};
  illegal_bins bad     = {7};
  bins path            = (IDLE => BUSY => DONE);
}
```

**注意**：重要协议的非法行为优先用 SVA 检查——断言有时序、复位屏蔽和错误控制，别只靠 `illegal_bins`。

> 一句话：**ignore 排除统计，illegal 命中报错，wildcard 合并不关心位，transition 覆盖跨采样序列。**

---

### 25. cross 覆盖率为什么容易组合爆炸？如何优化？

**题目来源**

- 海光 · 数字IC验证 · 校招 · NoC方向（Checkpoint 中 cross 功能点出现指数级爆炸的优化方法）
- 面试书《Cracking Digital VLSI Verification Interview》· 第6章 Q113、Q114（对应题目：交叉覆盖率 cross；cross 创建了多少个 bins）

**考点**

- bins 的笛卡尔积
- 无效组合裁剪
- 覆盖模型与验证计划对应

**参考答案**

**cross 是笛卡尔积——默认 bin 数量近似等于各 coverpoint bin 数量的乘积**，维度一多就指数爆炸。

比如 16×16×8 会产生 2048 个组合，数据库、采样、收敛成本全都会涨。

**优化五点**：

1. 只交叉 spec 真正关心的变量；
2. 先合并过细的基础 bins，再做 cross；
3. 用 `ignore_bins`、`binsof`、`intersect` 排除无效组合；
4. 把一个大 cross 拆成多个有业务意义的小 cross；
5. 不为了数字好看交叉彼此无关的字段。

> 一句话：**cross 是笛卡尔积，优化核心是减少维度、合并分桶、排除无效组合，并让每个 cross 对应真实测试目标。**

---

### 26. 高代码覆盖率是否代表高功能覆盖率？两者不一致说明什么？

**题目来源**

- 某TPU公司 · 数字IC验证 · 校招（代码覆盖率高是否代表功能覆盖率一定高；代码覆盖率高但功能覆盖率低的场景分析方法）
- 面试书《Cracking Digital VLSI Verification Interview》· 第6章 Q102、Q103（对应题目：高功能+低代码说明什么；高代码+低功能说明什么）

**考点**

- 覆盖率不能互相推出
- 覆盖模型漏洞和死代码
- 收敛诊断

**参考答案**

**不能互相推出。** 代码覆盖率和功能覆盖率是两把不同的尺子，一高一低说明激励、设计代码或覆盖模型之间存在错位。

**第一，代码覆盖率高、功能覆盖率低。** 可能是激励跑了大量 RTL，但关键场景或组合没发生；也可能是功能覆盖模型的采样时机不对。

**第二，功能覆盖率高、代码覆盖率低。** 可能是功能模型不完整或太宽松，也可能存在不可达代码、冗余分支、未启用配置。

**第三，收敛判断。** 任何一项接近 100% 都不等于验证完成，还要看 testcase 是否通过、断言是否失败、waiver 是否合理、测试计划是否完整、已知 bug 是否关闭。

> 一句话：**代码覆盖和功能覆盖是两把不同尺子；一高一低首先说明激励、设计代码或覆盖模型之间存在错位。**

---

### 27. 功能覆盖率应在什么时机采样？per-instance 有什么作用？

**题目来源**

- 某公司 · 数字IC验证 · 实习 · 一面（covergroup 所使用的 item 来源、内部成员变量）
- 面试书《Cracking Digital VLSI Verification Interview》· 第6章 Q116、Q120（对应题目：covergroup 有哪几种采样方式；per-instance 与 per-type 的区别）

**考点**

- 事件采样与显式 sample
- transaction 级采样
- 类型汇总和实例独立报告

**参考答案**

**功能覆盖率在 transaction 完整、语义有效的时刻采样**，比每拍机械采样更准确。

**第一，两种采样方式。** covergroup 可以指定事件自动采样，也可以不带事件、由 monitor 在事务完整时显式调 `sample()`。协议验证通常优先后者——只有 monitor 知道地址、数据、响应这些字段什么时候组成一笔有效事务。

**第二，per-instance 的作用。** 同一 covergroup 类型可以创建多个实例，默认报告按类型合并；设置 `option.per_instance = 1` 后，可以分别看每个 agent、端口或通道实例的覆盖情况。

```systemverilog
covergroup pkt_cg with function sample(packet p);
  option.per_instance = 1;
  cp_kind : coverpoint p.kind;
endgroup
```

**注意**：per-instance 便于定位局部覆盖洞，但整体覆盖目标仍要根据验证计划判断，不能把拆分实例本身当成覆盖提升。

> 一句话：**在语义完整的 transaction 时刻采样；per_instance 用于分开观察各实例，而不是凭空增加覆盖。**
