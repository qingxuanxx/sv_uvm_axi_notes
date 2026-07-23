# SystemVerilog 第 4 章学习笔记：连接设计和测试平台

> **核心思想**：测试平台应该独立于设计，并通过 `interface`、`clocking block`、`program block` 等机制与 DUT 稳定通信，避免传统 Verilog testbench 中常见的端口连接错误和竞争状态。

验证一个设计通常包括：生成输入激励 → 捕获输出响应 → 判断结果是否正确 → 衡量验证进度。

本章核心：用 `interface` 管连接，用 `modport` 管方向，用 `clocking block` 管时序，用 `program block` 管调度区域，用 `assertion` 管检查，用 `final block` 管结束总结。

---

## 4.1 将测试平台和设计分开

### 核心思想

| 部分 | 作用 |
|------|------|
| DUT | 实现设计功能 |
| Testbench | 产生激励、检查输出、模拟外部环境 |
| Top | 例化 DUT、testbench、interface、clock |

### 传统端口连接的问题

```systemverilog
// DUT
module arb_port(output logic [1:0] grant,
                input  logic [1:0] request,
                input  logic rst, clk);
    always_ff @(posedge clk or posedge rst) begin
        if (rst) grant <= 2'b00;
        else     grant <= request;
    end
endmodule

// Testbench
module test(input  logic [1:0] grant,
            output logic [1:0] request,
            output logic rst,
            input  logic clk);
    initial begin
        rst <= 1; request <= 0;
        @(posedge clk); rst <= 0; request <= 2'b01;
        repeat (2) @(posedge clk);
        if (grant != 2'b01) $display("ERROR");
        $finish;
    end
endmodule

// Top — 每个信号都要手动连接
module top;
    logic [1:0] grant, request;
    logic rst;
    bit clk;
    always #5 clk = ~clk;
    arb_port u_arb(grant, request, rst, clk);
    test     u_tb (grant, request, rst, clk);
endmodule
```

| 问题 | 说明 |
|------|------|
| 信号重复声明 | DUT、testbench、top 都要写一遍 |
| 连接容易出错 | 端口顺序错、信号名错都可能导致 bug |
| 扩展困难 | 新增一个信号需要改多个文件 |
| 不利于复用 | 每个模块都要重新整理端口 |

> 当两个模块之间有一组相关信号时，不应该一直手动列端口，应该使用 `interface` 把这些信号封装起来。

---

## 4.2 接口 Interface

### 4.2.1 interface 的作用

**接口（Interface）** = 一捆智能连线。可以包含：信号、时钟、modport、clocking block、断言、简单 task/function。

### 4.2.2 最简单的 interface

```systemverilog
interface arb_if(input bit clk);
    logic [1:0] grant, request;
    logic rst;
endinterface

// DUT 使用接口
module arb(arb_if arbif);
    always_ff @(posedge arbif.clk or posedge arbif.rst) begin
        if (arbif.rst) arbif.grant <= 2'b00;
        else           arbif.grant <= arbif.request;
    end
endmodule

// Testbench 使用接口
module test(arb_if arbif);
    initial begin
        arbif.rst <= 1; arbif.request <= 0;
        @(posedge arbif.clk); arbif.rst <= 0; arbif.request <= 2'b01;
    end
endmodule

// Top — 连线大幅简化
module top;
    bit clk;
    always #5 clk = ~clk;
    arb_if arbif(clk);   // 例化接口
    arb  u_arb(arbif);   // 传入接口
    test u_tb (arbif);
endmodule
```

| 优点 | 说明 |
|------|------|
| 减少端口数量 | 一组信号通过一个 interface 传递 |
| 降低连接错误 | top 层不用反复列出所有信号 |
| 便于扩展 | 新增信号只需在 interface 中添加 |
| 便于复用 | 总线协议可以封装成独立 interface |

⚠️ interface 定义不要写在 module 或 program 内部，否则可能变成局部定义，其他模块不可见。

### 4.2.3 连接 interface 和传统端口模块

旧 Verilog 模块仍是普通端口时，可在 top 中把 interface 信号拆开连接：

```systemverilog
module top;
    bit clk;
    always #5 clk = ~clk;
    arb_if arbif(clk);
    // .port(ifc.signal) 逐信号连接
    arb_port u_arb(.grant(arbif.grant), .request(arbif.request),
                   .rst(arbif.rst), .clk(arbif.clk));
    test u_tb(arbif);
endmodule
```

### 4.2.4 modport — 给信号分组并指定方向

```systemverilog
interface arb_if(input bit clk);
    logic [1:0] grant, request;
    logic rst;

    modport DUT(input request, rst, clk, output grant);
    modport TEST(output request, rst, input grant, clk);
    modport MONITOR(input request, grant, rst, clk);
endinterface

// 使用时指定 modport
module arb    (arb_if.DUT arbif);     // DUT 视角
module test   (arb_if.TEST arbif);    // TEST 视角
module monitor(arb_if.MONITOR arbif); // MONITOR 视角
```

| modport | 面向对象 | 信号方向 |
|---------|---------|---------|
| DUT | 设计模块 | request/rst 输入，grant 输出 |
| TEST | 测试平台 | request/rst 输出，grant 输入 |
| MONITOR | 监视器 | 所有信号只读 |

⚠️ modport 的方向是"相对于使用该 modport 的模块"来说的，不是固定信号方向。

### 4.2.5 接口监视模块

```systemverilog
module monitor(arb_if.MONITOR arbif);
    always @(posedge arbif.request[0]) begin
        $display("@%0t request[0] asserted", $time);
        @(posedge arbif.grant[0]);
        $display("@%0t grant[0] asserted", $time);
    end
endmodule
```

### 4.2.6 接口的优缺点

| 优点 ✅ | 缺点 ❌ |
|--------|--------|
| 适合协议封装（AXI、APB 等） | 点对点简单连接可能显得复杂 |
| 减少重复声明，降低连接错误 | 访问时要写 `bus.signal` |
| 支持 modport 区分视角 | 不同 interface 之间连接麻烦 |
| 支持 clocking block | 对专用一次性协议不一定划算 |

> 📌 接口中不能例化模块，但可以例化其他接口。

---

## 4.3 激励时序

### 4.3.1 为什么需要关注时序

testbench 和 DUT 之间存在两个关键动作：驱动 DUT 输入、采样 DUT 输出。如果 testbench 和 DUT 在同一时间片同时读写同一个信号，就会出现**竞争状态（Race Condition）**。

### 4.3.2 clocking block

```systemverilog
interface arb_if(input bit clk);
    logic [1:0] grant, request;
    logic rst;

    clocking cb @(posedge clk);
        output request;    // TB 驱动
        input  grant;      // TB 采样
    endclocking

    modport TEST(clocking cb, output rst);
    modport DUT(input request, rst, clk, output grant);
endinterface
```

testbench 中使用：

```systemverilog
program automatic test(arb_if.TEST bus);
    initial begin
        bus.cb.request <= 2'b01;  // 通过 clocking block 驱动
        @bus.cb;                  // 等待时钟块的有效沿
        if (bus.cb.grant != 2'b01)
            $display("ERROR");
    end
endprogram
```

| 写法 | 含义 |
|------|------|
| `clocking cb @(posedge clk);` | 定义同步时钟块 |
| `output request;` | testbench 通过 cb 驱动 request |
| `input grant;` | testbench 通过 cb 采样 grant |
| `@bus.cb` | 等待时钟块的有效时钟沿 |
| `bus.cb.request <= val;` | 同步驱动 |
| `bus.cb.grant` | 读取采样值 |

### 4.3.3 logic 和 wire 的选择

| 类型 | 特点 | 使用建议 |
|------|------|---------|
| `logic` | 可过程赋值，单驱动更安全 | interface 中常用 |
| `wire` | 需要连续赋值，适合多驱动网络 | 多驱动或三态总线使用 |

```systemverilog
interface async_if;
    logic l;  // 可直接过程赋值
    wire  w;  // 需 assign 驱动
endinterface

module test(async_if ifc);
    initial ifc.l <= 0;          // ✅ logic 可直接驱动
    assign ifc.w = local_wire;   // ✅ wire 用 assign
endmodule
```

### 4.3.4 Verilog 时序问题 — 为什么不能用 #0 / #1

| 方法 | 问题 |
|------|------|
| `#0` | 多个线程都用 #0 时，调度顺序不稳定 |
| `#1` | 受 timescale/timeprecision 影响，不可移植 |
| clocking block | ✅ 用语言机制规定采样和驱动时序 |

### 4.3.5 program block 和时序区域

SystemVerilog 一个时间片分为：

| 区域 | 执行内容 | 说明 |
|------|---------|------|
| Active | module 中的设计代码 | RTL、门级、时钟发生器 |
| Observed | 断言采样 | SVA 检查 |
| Reactive | program 中的 testbench | 与设计分离 |
| Postponed | 时间片末尾采样 | 只读（$monitor、$strobe） |

> 📌 program 块在 Reactive 区域执行，DUT 在 Active 区域执行，天然分离，消除竞争。

```systemverilog
program automatic test(arb_if.TEST bus);
    initial begin
        bus.cb.request <= 2'b01;
        repeat (2) @bus.cb;
        if (bus.cb.grant != 2'b01)
            $display("FAIL");
    end
endprogram
```

### 4.3.6 clocking block 的默认延时

| 方向 | 默认行为 | 说明 |
|------|---------|------|
| input | 前一个时间片末尾采样（`#1step`） | 读到 DUT 更新前的旧稳定值 |
| output | 当前时钟边沿驱动（`#0`） | 新值立即传给 DUT |

理解：时钟沿到来前，testbench 已采样了 DUT 的旧稳定输出；时钟沿到来时，testbench 驱动新的输入给 DUT。

### 4.3.7 仿真的结束

program 中所有 `initial` 块执行完毕 → 隐式 `$exit` → 仿真结束。多个 program 时，等最后一个 program 结束。

### 4.3.8 时钟发生器 — 放在 module 中

```systemverilog
// ✅ 正确：时钟发生器在 module 中（Active 区域）
module clock_generator(output bit clk);
    initial begin
        clk = 0;
        forever #5 clk = ~clk;
    end
endmodule

// ❌ 错误：放在 program 中会在 Reactive 区域，与 DUT 产生竞争
program bad_generator(output bit clk);
    initial forever #5 clk <= ~clk;
endprogram
```

> 📌 功能验证 testbench 检查 DUT 行为，不负责验证底层纳秒级时序；底层时序应由静态时序分析工具检查。

---

## 4.4 接口的驱动和采样

### 4.4.1 接口同步

```systemverilog
program automatic test(arb_if.TEST bus);
    initial begin
        @bus.cb;                      // 等待一个时钟块沿
        repeat (3) @bus.cb;           // 等待 3 个时钟沿
        @(posedge bus.cb.grant);      // 等待 grant 上升沿
        @(negedge bus.rst);           // 等待 rst 下降沿
        wait(bus.cb.grant == 1);      // 等待表达式成立
    end
endprogram
```

### 4.4.2 接口信号采样

从 clocking block 中读 input 信号时，读到的是时钟沿之前的采样值（Postponed 区域的旧值）。

```systemverilog
if (bus.cb.grant != 2'b01)   // ✅ 安全：读到的是上一个周期的稳定值
    $display("ERROR");
```

### 4.4.3 接口信号驱动

使用 clocking block 后，同步信号必须通过 `bus.cb.signal` 访问。

```systemverilog
bus.cb.request <= 2'b01;   // ✅ 正确：通过 cb 驱动
// bus.request <= 2'b01;   // ❌ 不推荐/可能非法
```

### 4.4.4 通过时钟块驱动信号

```systemverilog
bus.cb.request <= 1;        // 同步驱动，用 <=
bus.cb.cmd     <= cmd_buf;

// 等待两个时钟周期后驱动
##2 bus.cb.request <= 0;    // 等价于 repeat(2) @bus.cb;
```

⚠️ 异步地在时钟周期中间驱动 clocking block 信号，可能导致中间值丢失。

### 4.4.5 接口中的双向信号

```systemverilog
interface master_if(input bit clk);
    wire [7:0] data;             // 双向信号用 wire
    clocking cb @(posedge clk);
        inout data;
    endclocking
    modport TEST(clocking cb);
endinterface

program automatic test(master_if.TEST bus);
    initial begin
        bus.cb.data <= 'z;       // 三态释放
        @bus.cb;
        $displayh(bus.cb.data);  // 读总线
        @bus.cb;
        bus.cb.data <= 8'h5a;    // 驱动总线
        @bus.cb;
        bus.cb.data <= 'z;       // 释放
    end
endprogram
```

### 4.4.6 program 中为什么不允许 always 块

| 原因 | 说明 |
|------|------|
| 仿真结束语义 | program 中所有 initial 执行完 → 仿真结束，always 永不结束 |
| 测试是一次性的 | 测试平台有明确开始和结束，不是持续运行的硬件 |

如需循环行为：用 `initial forever` 替代。

### 4.4.7 时钟发生器总结

| 位置 | 是否适合 | 原因 |
|------|:---:|------|
| module | ✅ | clock 属于设计时序环境，Active 区域 |
| program | ❌ | 会进入 Reactive 区域，和 DUT 产生竞争 |

---

## 4.5 将这些模块都连接起来

### 4.5.1 top 模块的职责

```systemverilog
module top;
    bit clk;
    always #5 clk = ~clk;

    arb_if arbif(clk);
    arb     u_arb(arbif);
    test    u_tb (arbif);
    monitor u_mon(arbif);
endmodule
```

### 4.5.2 隐式端口连接 `.*`

如果端口名和当前作用域中的信号名一致，可用 `.*` 自动匹配：

```systemverilog
module top;
    bit clk;
    always #5 clk = ~clk;
    arb_if arbif(.*);
    arb    u_arb(.*);
    test   u_tb (.*);
endmodule
```

⚠️ `.*` 要求端口名和信号名**完全一致**、**类型兼容**。小工程可用，大工程慎用以免意外匹配。

### 4.5.3 interface 端口必须连接

含有 interface 端口的模块/程序**必须**连接到实际的 interface 实例，不能悬空编译。这和普通端口模块不同。

---

## 4.6 顶层作用域

### 4.6.1 $unit 和顶层作用域

SystemVerilog 可在 module/program/interface/package 之外定义 parameter、const、typedef、function/task：

```systemverilog
`timescale 1ns/1ns
parameter int TIMEOUT = 1_000_000;
const string timeout_msg = "ERROR: timeout";

module top;
    test u_test();
endmodule

program automatic test;
    initial begin
        #TIMEOUT;
        $display("%s", timeout_msg);
        $finish;
    end
endprogram
```

⚠️ `$unit` 的可见范围依赖工具的编译方式，可移植性不强。工程代码优先用 `package`。

### 4.6.2 $root — 绝对路径引用

```systemverilog
`define TOP $root.top   // 用宏封装路径

program automatic test;
    initial begin
        $display("clk=%b", `TOP.clk);        // 绝对引用（推荐）
        $display("clk=%b", $root.top.clk);    // 等价写法
    end
endprogram
```

---

## 4.7 程序——模块交互

### 4.7.1 基本规则

| 方向 | 是否允许 | 含义 |
|------|:---:|------|
| program 读写 module 信号 | ✅ 可以 | testbench 控制 DUT |
| program 调用 module task/function | ✅ 可以 | 用于后门访问 |
| module 访问 program | ❌ 不应该 | DUT 应独立于 testbench |

### 4.7.2 后门访问 Backdoor

testbench 不通过正常总线协议，而是直接调用 DUT 内部 task 修改状态：

```systemverilog
module mem_model;
    logic [7:0] mem[256];

    task backdoor_write(input int addr, input byte data);
        mem[addr] = data;
    endtask
endmodule

program automatic test;
    initial begin
        $root.top.u_mem.backdoor_write(8'h10, 8'h5a);  // 后门写入
    end
endprogram
```

| 场景 | 推荐方式 |
|------|---------|
| 初始化大存储器 | 后门写入 |
| 查询 DUT 内部状态 | module function 封装读取 |
| 避免误解内部编码 | 不直接读裸信号，用 function 返回抽象结果 |
| 正常协议验证 | 走前门接口，不要只用后门 |

---

## 4.8 SystemVerilog 断言 SVA

### 4.8.1 断言的作用

检查信号值、协议时序、X/Z 状态，辅助定位错误，收集覆盖率。

### 4.8.2 立即断言

```systemverilog
// 传统 if
if (bus.cb.grant != 2'b01)
    $display("ERROR");

// 立即断言（更紧凑）
a1: assert (bus.cb.grant == 2'b01);
```

> 📌 断言逻辑条件跟 if 相反：assert 期望括号内为**真**。

### 4.8.3 定制断言行为

```systemverilog
a1: assert (bus.cb.grant == 2'b01)
    grant_received++;                    // then：成功时执行
    else $error("Grant not asserted");   // else：失败时执行
```

四个消息级别：

| 函数 | 严重程度 |
|------|---------|
| `$info` | 信息 |
| `$warning` | 警告 |
| `$error` | 错误 |
| `$fatal` | 致命（终止仿真） |

### 4.8.4 并发断言

```systemverilog
interface arb_if(input bit clk);
    logic [1:0] request, grant;
    logic rst;

    property request_2state;
        @(posedge clk) disable iff (rst)
            !$isunknown(request);           // 非复位期间 request 不能 X/Z
    endproperty

    assert_request_2state:
        assert property(request_2state) else $error("request has X/Z");
endinterface
```

| 语法 | 含义 |
|------|------|
| `property ... endproperty` | 定义时序属性 |
| `@(posedge clk)` | 采样时钟 |
| `disable iff (rst)` | 复位期间关闭断言 |
| `$isunknown(x)` | 检查表达式是否含 X/Z |
| `assert property(...)` | 例化并启动并发检查 |

> 📌 将断言放在 interface 中，所有使用该接口的模块都会自动受到协议检查。

---

## 4.10 ref 端口的方向

`ref` 不是普通输入输出，也不是双向线网，而是**变量引用**（类似 C++ 引用）。

```systemverilog
module m(ref int x);
    initial x = x + 1;   // 直接修改外部变量
endmodule
```

| 项目 | ref | inout |
|------|-----|-------|
| 对象 | 变量 variable | 线网 net |
| 驱动解析 | 没有 | 根据驱动值和强度解析 |
| 多驱动 | 容易竞争 | 可正确解析 |
| 典型用途 | 共享变量、任务参数 | 双向总线 |

⚠️ ref 只能连接变量，不能连接 net；多个 ref 同时修改同一变量可能产生竞争。

---

## 4.11 仿真的结束

### 4.11.1 结束机制

```text
program 中所有 initial 块执行完毕
  → 隐式 $exit
  → 所有 program 退出
  → 隐式 $finish
  → 执行所有 final 块
  → 仿真真正结束
```

### 4.11.2 final 块

```systemverilog
program automatic test;
    int errors, warnings;

    initial begin
        // 主测试流程...
    end

    final begin   // 仿真结束前必定执行
        $display("Test done with %0d errors and %0d warnings",
                 errors, warnings);
    end
endprogram
```

| 限制 | 说明 |
|------|------|
| 不能有延时 | 不能写 `#`、`@`、`wait` |
| 不需要释放内存 | 仿真器自动处理 |
| 适合做的事 | 打印总结、关闭文件、输出覆盖率 |

---

## 本章总结（4.1–4.11，跳过 4.9）

### 知识链

```
DUT 与 Testbench 分离
  → interface 封装连接信号
    → modport 区分 DUT/TB/Monitor 方向
      → clocking block 解决同步采样和驱动时序
        → program block 避免调度竞争
          → assertion 检查协议合法性
            → top 统一连接
              → final block 做结束总结
```

### 关键概念速查

| 概念 | 英文 | 作用 |
|------|------|------|
| 待测设计 | DUT | 被验证的设计 |
| 测试平台 | Testbench | 产生激励、检查输出 |
| 接口 | Interface | 封装一组连接信号 |
| 端口方向分组 | modport | 区分 DUT/TB/Monitor 视角 |
| 时钟块 | Clocking Block | 规定同步驱动和采样时序 |
| 程序块 | Program Block | 在 Reactive 区域运行 testbench |
| 断言 | Assertion | 检查信号或协议是否满足条件 |
| 引用端口 | ref | 变量引用式端口 |
| final 块 | final block | 仿真结束前执行总结代码 |

### 你写的 ch4 代码对照表

| 文件 | 对应章节 | 知识点 |
|------|---------|--------|
| `1.sv` | 4.1, 4.3 | 传统 module 端口、竞争冒险演示 |
| `2.sv` | 4.2 | interface + modport（DUT/TB 视角） |
| `3.sv` | 4.3, 4.4 | 接口中封装 task、`#1step` 解决竞争 |
| `4.sv` | 4.3.5 | program 块 + reactive 区域 |
| `5.sv` | 4.3, 4.4 | clocking block、`input #1step`/`output #1ns` |
| `6.sv` | 4.6 | `$unit`/`$root`/`.*` 隐式连接 |
| `7.sv` | 4.8 | SVA 并发断言、`property`/`|->`/`disable iff` |
| `8.sv` | 4.11 | `final` 块、`$urandom_range`、program 自动结束 |

### 最重要的 10 条规则

| # | 规则 |
|---|------|
| 1 | 用 `interface` 替代分散的信号连线，减少出错 |
| 2 | 用 `modport` 给信号分组并指定方向 |
| 3 | 用 `clocking block` 消除时序竞争 |
| 4 | 测试代码放在 `program` 块中，声明为 `automatic` |
| 5 | 时钟发生器永远放在 `module` 中，不要放在 `program` 中 |
| 6 | 同步信号通过 `clocking block` 用 `<=` 驱动 |
| 7 | program 不能用 `always`，用 `initial forever` 代替 |
| 8 | 并发断言放在 `interface` 中，所有模块自动受保护 |
| 9 | 使用 `$root` 绝对路径避免跨模块引用歧义 |
| 10 | `final` 块用于打印总结报告，保证一定会执行 |

### 最容易错的点

| 易错点 | 正确理解 |
|--------|---------|
| interface 写在 module 内部 | 不推荐，可能变成局部定义 |
| modport 方向是固定信号方向 | 错，是相对于使用者的方向 |
| 有了 clocking block 还直接写 `bus.request` | 同步信号应写 `bus.cb.request` |
| `#0`/`#1` 可以稳定解决竞争 | 不可靠，应使用 clocking block |
| program 中可以写 `always` | 不可以，用 `initial forever` 替代 |
| 时钟发生器放在 program 中 | 错，应放在 module 中 |
| `.*` 总是安全的 | 只有端口名、信号名、类型匹配时才安全 |
| ref 等价于 inout | 错，ref 是变量引用，inout 是双向线网 |
| final 可以等待时钟 | 错，final 不能包含延时或事件等待 |
