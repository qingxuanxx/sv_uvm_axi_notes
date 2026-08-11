# SystemVerilog 第 5 章学习笔记：面向对象编程基础

> **核心结论**：用 `class` 把数据和操作封装成事务级对象，通过句柄动态创建和传递，让测试平台从"信号翻转"升级到"事务处理"。这是后续继承、约束随机、UVM 的基石。

---

## 5.1 概述

### 核心概念

**面向对象编程（OOP）** 的核心思想是：把数据和操作这些数据的代码封装在一起。

传统 Verilog 中描述一个总线事务，通常会写成多个数组：

```systemverilog
bit [31:0] addr_array[100];
bit [31:0] data_array[100];
bit [3:0]  cmd_array[100];
```

| 问题 | 说明 |
|------|------|
| 数据分散 | 一个 transaction 的地址、数据、命令分散在多个数组中 |
| 容量固定 | 数组大小需要提前定义，修改需要重新编译 |
| 难维护 | 数据结构和操作代码分离 |
| 不利于复用 | testbench 和底层信号耦合太紧 |

OOP 的做法是把这些信息封装成一个类：

```systemverilog
class Transaction;
    bit [31:0] addr;
    bit [31:0] data;
    bit [3:0]  cmd;

    function void display();
        $display("addr=%h, data=%h", addr, data);
    endfunction
endclass
```

这个类把事务的数据和显示事务的方法放在一起。

---

## 5.2 考虑名词，而非动词

### 核心思想

传统 testbench 容易围绕"动作"来写：创建事务 → 发送事务 → 接收结果 → 检查结果。OOP 更强调围绕"对象"来组织：Transaction、Generator、Driver、Monitor、Scoreboard。

| 传统思路 | OOP 思路 |
|---------|---------|
| 关注操作流程 | 关注对象和对象之间的关系 |
| 直接操作信号 | 操作 transaction |
| 代码耦合度高 | 组件边界清晰 |
| 难复用 | 易复用 |

### 验证平台中的角色

| 组件 | 作用 |
|------|------|
| Generator | 产生 transaction |
| Driver | 把 transaction 转换成 DUT 输入信号 |
| Monitor | 从 DUT 输出信号中采样并还原 transaction |
| Scoreboard | 比较实际结果和期望结果 |

```systemverilog
class Generator;
    Transaction tr;

    function Transaction create();
        tr = new();
        tr.addr = 32'h1000;
        tr.data = 32'h55aa;
        return tr;
    endfunction
endclass
```

Generator 不直接翻转信号，而是创建一个事务对象。

---

## 5.3 编写第一个类 Class

### 类的基本结构

类（Class）可以同时包含数据成员、函数 function、任务 task。

```systemverilog
class Transaction;
    bit [31:0] addr, crc, data[8];

    function void display();
        $display("Transaction addr=%h", addr);
    endfunction : display

    function void calc_crc();
        crc = addr;
        foreach (data[i])
            crc ^= data[i];
    endfunction : calc_crc
endclass : Transaction
```

| 代码 | 含义 |
|------|------|
| `class Transaction;` | 定义一个类 |
| `addr, crc, data` | 类中的数据成员 |
| `display()` | 类中的方法 |
| `calc_crc()` | 操作类中数据的方法 |
| `endclass : Transaction` | 类结束，加标签便于阅读 |

### 命名习惯

| 类型 | 推荐命名 |
|------|---------|
| 类名 | 首字母大写，如 `Transaction` |
| 常量 | 全大写，如 `CELL_SIZE` |
| 变量 | 小写，如 `count`, `addr` |
| 方法 | 小写或下划线，如 `calc_crc()` |

---

## 5.4 在哪里定义类

### 类可以定义的位置

| 位置 | 是否可行 | 说明 |
|------|---------|------|
| `program` 内 | 可以 | 小型 testbench 可用 |
| `module` 内 | 可以 | 不推荐作为主要风格 |
| `package` 内 | 推荐 | 适合工程化管理 |
| 文件顶层 | 可以 | 适合简单练习 |

### 推荐写法：放在 package 中

```systemverilog
package trans_pkg;
    class Transaction;
        bit [31:0] addr;
        bit [31:0] data;
    endclass
endpackage

program automatic tb;
    import trans_pkg::*;
    initial begin
        Transaction tr = new();
    end
endprogram
```

> 📌 一个类可以单独放一个 `.sv` 文件，一组相关类放进同一个 `package`，不同协议的类分开放。

---

## 5.5 OOP 术语

| 术语 | 英文 | 含义 | 类比 Verilog |
|------|------|------|-------------|
| 类 | Class | 定义对象模板 | module |
| 对象 | Object | 类的一个实例 | module instance |
| 句柄 | Handle | 指向对象的引用 | 实例名 / 指针类比 |
| 属性 | Property | 类中的变量 | reg/wire |
| 方法 | Method | 类中的 task/function | task/function |
| 原型 | Prototype | 方法声明头 | 函数声明 |

### 类、对象、句柄的关系

```systemverilog
Transaction tr;  // tr 是句柄（初始为 null）
tr = new();       // new 后才创建对象
tr.addr = 32'h10; // 通过句柄访问对象
```

> 📌 **类比**：类 = 房子的蓝图；对象 = 实际建好的房子；句柄 = 房子的地址。

---

## 5.6 创建新对象

### 5.6.1 声明句柄和创建对象

```systemverilog
Transaction tr;  // 声明句柄（初始 null）
tr = new();       // 创建对象：分配空间、初始化、返回句柄
```

| 类型 | 默认值 |
|------|--------|
| 二值变量 `bit` | `0` |
| 四值变量 `logic` | `X` |
| 句柄 | `null` |

### 5.6.2 定制构造函数 Constructor

构造函数名字固定为 `new`，不能有返回类型。

```systemverilog
class Transaction;
    logic [31:0] addr;
    logic [31:0] data[8];

    function new(logic [31:0] a = 32'h0, logic [31:0] d = 32'h5);
        addr = a;
        foreach (data[i])
            data[i] = d;
    endfunction
endclass

Transaction tr;
initial begin
    tr = new(32'h2000);  // addr=2000, data 使用默认值 5
end
```

### 5.6.3 将声明和创建分开

| 推荐 ✅ | 不推荐 ❌ |
|--------|---------|
| `Transaction tr;`<br>`tr = new();` | `Transaction tr = new();` |

原因：声明时创建会失去对初始化顺序的控制，不利于调试。

### 5.6.4 `new()` 和 `new[]` 的区别

| 写法 | 作用 | 示例 |
|------|------|------|
| `new()` | 创建一个类对象 | `tr = new();` |
| `new[10]` | 创建动态数组空间 | `arr = new[10];` |

⚠️ 二者不是一回事。

### 5.6.5 为对象创建句柄

```systemverilog
Transaction t1, t2;

initial begin
    t1 = new();   // t1 指向对象A
    t2 = t1;      // t1 和 t2 都指向对象A（句柄复制，不是对象复制）
    t1 = new();   // t1 指向对象B，t2 仍指向对象A
end
```

| 语句 | 含义 |
|------|------|
| `t2 = t1;` | 复制句柄，不复制对象 |
| `t1 = new();` | t1 改指向新对象 |

---

## 5.7 对象的解除分配 Deallocation

SystemVerilog 会自动回收不再被句柄引用的对象。

```systemverilog
Transaction t;
initial begin
    t = new();   // 创建第一个对象
    t = new();   // 第一个对象没有句柄指向 → 可被回收
    t = null;    // 第二个对象也被回收
end
```

| 语言 | 内存释放方式 |
|------|------------|
| C/C++ | 程序员手动释放 |
| SystemVerilog | 没有句柄引用时可自动回收（垃圾回收） |

⚠️ 链表等互相引用的结构，需要手动将所有句柄设为 `null` 才能回收。

---

## 5.8 使用对象

```systemverilog
Transaction tr;
initial begin
    tr = new();
    tr.addr = 32'h42;    // 直接访问变量
    tr.display();         // 调用方法
end
```

### 测试平台中建议保持变量公有

传统 OOP 倾向于通过 `get()`/`set()` 方法访问数据，但验证平台需要：

- 随机化字段
- 修改边界值
- 注入非法值
- 构造错误激励

```systemverilog
tr.crc = 32'hdead_beef;  // 故意写错 CRC，测试 DUT 错误处理
```

---

## 5.9 静态变量和静态方法

### 5.9.1 静态变量 `static`

静态变量属于**类**，被所有对象共享。

```systemverilog
class Transaction;
    static int count = 0;  // 所有实例共享，只存一份
    int id;                // 每个实例独有

    function new();
        id = count++;      // 为每个对象分配唯一 ID
    endfunction
endclass

Transaction t1 = new();  // id=0, count=1
Transaction t2 = new();  // id=1, count=2
```

### 5.9.2 通过类名访问静态变量 `::`

```systemverilog
$display("count=%0d", Transaction::count);  // 推荐
$display("count=%0d", t1.count);           // 可以但不推荐
```

### 5.9.3 静态变量的初始化

静态变量通常在声明时初始化，不要在构造函数中初始化（每次 `new()` 都会调用）。

### 5.9.4 静态方法

```systemverilog
class Transaction;
    static int count = 0;

    static function void display_count();
        $display("count=%0d", count);    // ✅ 可以访问静态变量
        // $display("%0d", id);           // ❌ 不能访问非静态变量
    endfunction
endclass

Transaction::display_count();  // 无需创建对象即可调用
```

静态方法只能访问静态成员，不能访问普通对象成员。

---

## 5.10 类的方法

类中的 `task` 或 `function` 称为**方法（Method）**，可以直接访问类成员。

```systemverilog
class Transaction;
    bit [31:0] addr, crc, data[8];

    function void display();
        $display("addr=%h crc=%h", addr, crc);
    endfunction
endclass
```

| 项目 | 类方法 | 普通函数 |
|------|--------|---------|
| 定义位置 | 类内部 | module/program/package |
| 访问类成员 | 可以直接访问 | 不可以直接访问 |
| 调用方式 | `object.method()` | `function_name()` |
| 存储类型 | 默认 automatic | 取决于上下文 |

---

## 5.11 在类之外定义方法 `extern`

当类变大时，可以在类内写方法原型，在类外写方法体。

```systemverilog
class Transaction;
    bit [31:0] addr, crc;

    extern function void display();   // 原型声明
endclass

function void Transaction::display();  // 类外实现，必须加 类名::
    $display("addr=%h crc=%h", addr, crc);
endfunction
```

| 常见错误 | 说明 |
|---------|------|
| 忘记写 `Transaction::` | 方法变成全局函数，无法访问类成员 |
| 原型和实现不匹配 | 返回类型、参数列表必须完全一致 |

---

## 5.12 作用域规则

### 名字查找规则

```text
当前作用域 → 上一级作用域 → ... → $root
```

```systemverilog
int limit = 10;            // $root.limit

program automatic p;
    int limit = 5;         // $root.p.limit

    initial begin
        int limit = 3;     // 局部变量
        $display("%0d", limit);  // 输出 3（最近的作用域）
    end
endprogram
```

### 常见陷阱：循环变量未声明

```systemverilog
program test;
    int i;  // 程序级变量

    class Bad;
        bit [31:0] data[];
        function void display();
            for (i = 0; i < data.size(); i++)  // ❌ 忘声明 i，用了程序级变量
                $display("%h", data[i]);
        endfunction
    endclass
endprogram
```

✅ 正确写法：

```systemverilog
for (int i = 0; i < data.size(); i++)  // 局部声明
```

> 📌 类建议放进 `package`，防止意外访问程序级变量。

### 5.12.1 `this` 是什么

当成员变量和参数同名时，用 `this` 区分：

```systemverilog
class Transaction;
    bit [31:0] addr;

    function new(bit [31:0] addr);
        this.addr = addr;  // this.addr = 成员变量, addr = 参数
    endfunction
endclass
```

---

## 5.13 在一个类内使用另一个类（组合）

```systemverilog
class Statistics;
    time startT, stopT;

    function void start();
        startT = $time;
    endfunction
endclass

class Transaction;
    bit [31:0] addr;
    Statistics stats;        // 句柄

    function new();
        stats = new();       // ⚠️ 必须在构造函数中创建内部对象！
    endfunction
endclass

Transaction tr = new();
tr.stats.start();            // 通过大类的句柄调用小类的方法
```

> ⚠️ `Statistics stats;` 只是声明句柄，必须写 `stats = new();`，否则 `stats == null`。

### 5.13.1 我的类该做成多大

| 情况 | 建议 |
|------|------|
| 一个类超过一屏，且功能混杂 | 考虑拆分 |
| 多个方法中有重复代码 | 抽出公共方法 |
| 小类只有一两个成员，且只用一次 | 可以合并到父类 |
| 某些成员形成独立功能 | 可以拆成子类 |

### 5.13.2 编译顺序问题：`typedef class`

```systemverilog
typedef class Statistics;    // 前置声明

class Transaction;
    Statistics stats;        // 现在可以用
endclass

class Statistics;            // 稍后完整定义
    time startT, stopT;
endclass
```

`typedef class B;` 相当于提前告诉编译器："后面会有一个叫 B 的类"。

---

## 5.14 理解动态对象

在 OOP 中，对象是在仿真运行过程中动态创建的。一个 testbench 可以创建成百上千个 transaction 对象，但实际代码里可能只有几个句柄在操作它们。

| 项目 | Verilog module | SystemVerilog class |
|------|---------------|-------------------|
| 实例创建时间 | 编译/展开时 | 仿真运行时 |
| 数量是否固定 | 固定 | 可动态变化 |
| 主要用途 | 描述硬件结构 | 描述测试平台对象 |

### 5.14.1 将对象传递给方法

传递的是句柄，不是对象本身。方法可以通过句柄修改对象内容。

```systemverilog
task transmit(Transaction t);
    t.stats.startT = $time;  // 可以修改对象成员
    // t = new();            // 只修改局部句柄，不影响外部
endtask

Transaction tr = new();
transmit(tr);  // tr 指向的对象被修改
```

### 5.14.2 在方法中修改句柄 → 必须用 `ref`

```systemverilog
// ❌ 错误
function void create(Transaction tr);
    tr = new();               // 只修改了局部副本，外部仍是 null
endfunction

// ✅ 正确
function void create(ref Transaction tr);
    tr = new();               // 修改了外部的句柄
endfunction
```

| 目的 | 是否需要 ref |
|------|------------|
| 修改对象内部变量 | 不需要 |
| 让外部句柄指向新对象 | 需要 |
| 修改句柄本身 | 需要 |

### 5.14.3 每次循环创建新对象

```systemverilog
// ❌ 错误：反复修改同一个对象
task generator_bad(int n);
    Transaction t = new();
    repeat (n) begin
        t.addr = $random();
        transmit(t);           // 传入的都是同一个对象
    end
endtask

// ✅ 正确：每次循环创建新对象
task generator_good(int n);
    Transaction t;
    repeat (n) begin
        t = new();             // 每次创建新对象
        t.addr = $random();
        transmit(t);
    end
endtask
```

> 📌 每一个独立 transaction 都应该 `new` 一个新对象。

### 5.14.4 句柄数组

```systemverilog
Transaction tarray[10];         // 10个句柄，初始都是 null

foreach (tarray[i]) begin
    tarray[i] = new();          // 必须逐个创建对象
    tarray[i].addr = i;
end

// 动态数组
Transaction tarray[];
tarray = new[10];               // 创建 10 个句柄位置（不是 10 个对象）
foreach (tarray[i])
    tarray[i] = new();          // 还要单独 new 对象
```

⚠️ SystemVerilog 中没有真正的"对象数组"，常说的对象数组本质上是"句柄数组"。

---

## 5.15 对象的复制

### 5.15.1 使用 `new` 浅复制 — 危险

```systemverilog
Transaction src = new();
Transaction dst = new src;   // 浅复制：只复制顶层变量
// src.stats 和 dst.stats 指向同一个 Statistics 对象！
dst.stats.startT = 96;
$display("%t", src.stats.startT);  // 输出 96，两边都被改了
```

### 5.15.2–5.15.3 自定义深复制 `copy()` — 推荐

```systemverilog
class Statistics;
    time startT, stopT;

    function Statistics copy();
        copy = new();
        copy.startT = startT;
        copy.stopT  = stopT;
    endfunction
endclass

class Transaction;
    bit [31:0] addr, crc, data[8];
    Statistics stats;
    static int count = 0;
    int id;

    function new();
        stats = new();
        id = count++;
    endfunction

    function Transaction copy();
        copy = new();                 // 创建新对象（id 自动递增）
        copy.addr = addr;
        copy.crc  = crc;
        copy.data = data;
        copy.stats = stats.copy();    // 递归深复制内部对象
    endfunction
endclass
```

### 三种操作对比

| 操作 | 写法 | 创建新顶层对象？ | 创建新内部对象？ |
|------|------|:---:|:---:|
| 句柄赋值 | `dst = src;` | ❌ | ❌ |
| 浅复制 | `dst = new src;` | ✅ | ❌ |
| 深复制 | `dst = src.copy();` | ✅ | ✅ |

### 5.15.4 流操作符 pack/unpack

```systemverilog
class Transaction;
    bit [31:0] addr, crc, data[8];  // 需要打包的数据
    static int count = 0;
    int id;                           // 不需要打包的字段

    function void pack(ref byte bytes[40]);
        bytes = {>> {addr, crc, data}};   // 大端序打包
    endfunction

    function void unpack(ref byte bytes[40]);
        {>> {addr, crc, data}} = bytes;   // 大端序解包
    endfunction
endclass
```

> 📌 流操作符不能直接作用于整个对象，只打包需要传输的成员（排除时间戳、ID 等）。

---

## 5.16 公有和私有

```systemverilog
class Packet;
    bit [31:0] addr;           // public — 默认
    protected bit [15:0] crc;  // protected — 仅本类和子类可访问
    local string str;          // local — 仅本类内部可访问
endclass
```

| 修饰符 | 可见性 | 测试平台建议 |
|--------|--------|------------|
| 默认（public） | 任何地方 | ✅ 推荐 |
| `protected` | 本类 + 子类 | 需要访问器方法 |
| `local` | 仅本类 | 需要访问器方法 |

> 📌 测试平台的目标是**最大化对 DUT 的控制**（包括注入错误），所以建议变量保持 public。

---

## 本章总结（5.1–5.16）

### 核心知识链

```
class → handle → new() → object → method → static/this/scope → copy → pack/unpack → public/protected/local
```

### 你写的 ch5 代码对照表

| 文件 | 对应章节 | 知识点 |
|------|---------|--------|
| `1.sv` | 5.6, 5.8 | 基本类定义、`new()`、句柄声明、对象创建 |
| `2.sv` | 5.6.2 | 带默认参数的构造函数、`this` 指针 |
| `3.sv` | 5.9 | 静态变量、静态方法、类名访问 `::` |
| `4.sv` | 5.11 | `extern` 外部实现方法 |
| `5.sv` | 5.16 | `public` / `protected` / `local` 访问控制 |
| `6.sv` | 5.13 | 类组合（Transaction 包含 Statistics） |
| `7.sv` | 5.15.1–5.15.3 | 浅复制 vs 深复制、自定义 `copy()` |
| `8.sv` | 5.14.2, 5.7 | `ref` 传递句柄、垃圾回收 |
| `9.sv` | 5.15.4 | `packed struct` 中介、`{>>{}}` 打包解包 |

### 最重要的 10 个结论

| # | 结论 |
|---|------|
| 1 | `class` 是对象模板，`object` 是类的实例 |
| 2 | `Transaction tr;` 只是声明句柄，不会创建对象 |
| 3 | `tr = new();` 才真正创建对象 |
| 4 | `t2 = t1;` 是句柄赋值，不是对象复制 |
| 5 | `static` 变量被所有对象共享 |
| 6 | `this` 用来明确访问当前对象成员 |
| 7 | 类外定义方法必须写 `ClassName::method()` |
| 8 | 方法参数传对象时，默认传的是句柄 |
| 9 | 修改外部句柄本身需要 `ref` |
| 10 | 有内部对象句柄时，需要自己写深拷贝 `copy()` |

### 最容易错的点

| 易错点 | 正确理解 |
|--------|---------|
| 声明句柄等于创建对象 | 错，必须 `new()` |
| 句柄赋值等于复制对象 | 错，只是两个句柄指向同一对象 |
| `new[10]` 创建 10 个对象 | 错，只创建 10 个句柄位置 |
| 对象传参不会修改原对象 | 错，方法可通过句柄修改对象内容 |
| 所有字段都应该私有 | 验证平台中通常保持公有更方便 |
| `new src` 是深拷贝 | 错，它是浅拷贝 |
| 类外函数不用写类名 | 错，必须写 `ClassName::method` |
