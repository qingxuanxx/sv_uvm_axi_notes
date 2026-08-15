# UVM 八股

> 面试常考的 UVM 八股积累（持续更新）。
>
> 阅读配套：`../uvm学习笔记/`（UVM 实战笔记）

## 目录

1. [uvm_object 与 uvm_component 的区别](#1-uvm_object-与-uvm_component-的区别)
2. [config_db 完成 set 后，若源变量发生变化，get 到的值是否同步更新？](#2-config_db-完成-set-后若源变量发生变化get-到的值是否同步更新)
3. [uvm_info 的消息等级分类](#3-uvm_info-的消息等级分类)

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
