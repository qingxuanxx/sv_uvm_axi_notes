# SystemVerilog 第 3 章学习笔记：过程语句和子程序

> **本章核心**：SystemVerilog 在过程语句和子程序上向 C/C++ 看齐 — `for` 内声明变量、`break`/`continue`、`return`、`ref` 引用传递、`automatic` 动态存储、缺省参数值、命名参数传递。这些改进让 testbench 代码更简洁、更安全、更接近软件语言。

---

## 3.1 过程语句

SystemVerilog 从 C/C++ 引入了很多过程语句改进：

| 特性 | 说明 | 示例 |
|------|------|------|
| `for` 内声明变量 | 作用域仅限于循环 | `for (int i=0; i<10; i++)` |
| `++` / `--` | 自增/自减，前缀或后缀 | `i++`, `--j` |
| `do-while` | 先执行再判断，至少一次 | `do {...} while (cond);` |
| 块标签 | `begin : label` ↔ `end : label` | 提高可读性、方便调试 |
| `continue` | 跳过本轮，进入下一轮 | 配合 `if` 使用 |
| `break` | 终止整个循环 | 配合 `if` 使用 |

### 基本代码示例

```systemverilog
initial begin : example
    int array[10], sum, j;

    // for 循环内声明 i
    for (int i = 0; i < 10; i++) array[i] = i;

    // do-while
    sum = array[9]; j = 8;
    do sum += array[j]; while (j--);

    // continue 和 break
    for (int i = 0; i <= 10; i++) begin
        if (i == 5) continue;   // 跳过 i=5
        if (i == 8) break;      // i=8 退出循环
        $display("i=%0d", i);
    end
    $display("Sum=%4d", sum);
end : example
```

### 文件读取中的应用

```systemverilog
initial begin
    bit [127:0] cmd; int file;
    file = $fopen("commands.txt", "r");
    while (!$feof(file)) begin
        $fscanf(file, "%s", cmd);
        case (cmd)
            ""    : continue;        // 空行跳过
            "done": break;           // 结束标记
            default: $display("cmd = %s", cmd);
        endcase
    end
    $fclose(file);
end
```

> 📌 `continue` 跳过本轮剩余语句，`break` 立即终止循环。文件读取模式是构建可配置 testbench 的基础。

---

## 3.2 任务、函数以及 void 函数

### 3.2.1 task 和 function 的区别

| 项目 | task | function |
|------|:---:|:---:|
| 可以消耗时间（`#`, `@`, `wait`） | ✅ | ❌ |
| 可以包含 `#delay` | ✅ | ❌ |
| 可以包含 `@event` | ✅ | ❌ |
| 可以调用 task | ✅ | ❌（fork...join_none 子线程除外） |
| 必须有返回值 | ❌ | Verilog 中需要 |
| 典型用途 | 驱动总线、等待事件 | 计算值、检查状态 |

### 3.2.2 void 函数

**void 函数（Void Function）** 没有返回值、不消耗时间，可被 task 和 function 调用。

```systemverilog
function void print_state();
    $display("@%0t: state = %s", $time, cur_state.name());
endfunction
```

| 写成 task | 写成 void function |
|----------|-------------------|
| 不能被 function 调用 | 可以被 task 和 function 调用 |
| 语义上可能消耗时间 | 明确表示不消耗时间 |
| 灵活性较差 | 灵活性更好 |

> 📌 建议：所有不消耗时间的调试子程序都定义成 `function void`，不要用 `task`。

### 3.2.3 忽略函数返回值

```systemverilog
void'($fscanf(file, "%d", value));  // 显式告诉编译器：我不关心返回值
```

---

## 3.3 任务和函数概述

### 3.3.1 去掉 begin...end

`task...endtask` 和 `function...endfunction` 已能界定子程序边界，内部 `begin...end` 可选。

```systemverilog
// ❌ Verilog-1995 风格
task multiple_lines;
    begin
        $display("First line");
        $display("Second line");
    end
endtask

// ✅ SystemVerilog 风格
task multiple_lines;
    $display("First line");
    $display("Second line");
endtask : multiple_lines
```

---

## 3.4 子程序参数

### 3.4.1 C 语言风格的参数声明

```systemverilog
// ❌ Verilog-1995：重复声明（方向 + 类型 + 名称）
task mytask2;
    output [31:0] x;
    reg    [31:0] x;
    input  y;
endtask

// ✅ SystemVerilog：C 风格（方向和类型合并）
task mytask1(output logic [31:0] x, input logic y);
endtask
```

| 优点 | 说明 |
|------|------|
| 更简洁 | 不用重复声明方向和类型 |
| 更清晰 | 参数在子程序头部一目了然 |
| 更接近 C | 便于有软件背景的人理解 |

### 3.4.2 参数的方向

| 方向 | 含义 |
|------|------|
| `input` | 输入，默认方向 |
| `output` | 输出，子程序结束时返回 |
| `inout` | 输入输出，进入时复制进来，退出时复制出去 |
| `ref` | 引用传递，直接操作调用者变量 |

```systemverilog
task T3(input logic a, b, output bit [15:0] u, v);
    // a, b = input logic（1比特）
    // u, v = output bit [15:0]
endtask
```

⚠️ **强烈建议**：始终明确写出所有参数的类型和方向，不要依赖默认规则（见 3.4.6 节的陷阱）。

### 3.4.3 ref 参数 — 引用传递

传统 `input`/`output`/`inout` 本质是复制：调用时复制进去，结束时复制回来。对大数组，复制开销很大。

| 传递方式 | 行为 | 开销 |
|---------|------|------|
| 默认（值传递） | 复制整个数组到堆栈 | 大数组性能差 |
| `ref`（引用传递） | 传递地址，直接操作原数组 | 零拷贝 |
| `const ref`（只读引用） | 传递地址但不能修改 | 零拷贝 + 安全 |

```systemverilog
// const ref：只读引用，传递大数组零开销
function void print_checksum(const ref bit [31:0] a[]);
    bit [31:0] checksum = 0;
    for (int i = 0; i < a.size(); i++)
        checksum ^= a[i];
    $display("The array checksum is %0d", checksum);
endfunction

// ref：需要修改数组时使用
function void clear_array(ref int a[]);
    foreach (a[i])
        a[i] = 0;
endfunction
```

| 场景 | 推荐参数类型 |
|------|------------|
| 小整数输入 | `input` |
| 大数组只读 | `const ref` |
| 大数组需要修改 | `ref` |
| 需要返回多个值 | `output` 或 `ref` |
| 多线程中希望立即看到变量变化 | `ref` |

### ref 在多线程中的应用

`ref` 参数变化可以被其他线程**立刻**看到，无需等任务结束：

```systemverilog
task bus_read(input logic [31:0] addr, ref logic [31:0] data);
    bus.request = 1'b1;
    @(posedge bus.grant) bus.addr = addr;
    @(posedge bus.enable) data = bus.data;  // data 一变化，外部 @data 立即触发
    bus.request = 1'b0;
    @(negedge bus.grant);
endtask

logic [31:0] addr, data;
initial fork
    bus_read(addr, data);     // ref 让 data 变化立即可见
    thread2: begin
        @data;                // 不等任务结束就能触发
        $display("Read %h from bus", data);
    end
join
```

| ref vs output | 外部何时看到变化 |
|--------------|:---:|
| `output` 参数 | 任务结束后 |
| `ref` 参数 | 任务内一修改，立即可见 |

### 3.4.4 参数的缺省值

```systemverilog
function void print_checksum(
    ref   bit [31:0] a[],
    input bit [31:0] low  = 0,         // 缺省值 0
    input int        high = -1          // -1 表示"到最后"
);
    if (high == -1 || high >= a.size())
        high = a.size() - 1;

    bit [31:0] checksum = 0;
    for (int i = low; i <= high; i++)
        checksum += a[i];
    $display("checksum = %0d", checksum);
endfunction
```

调用：

```systemverilog
print_checksum(a);          // 全部元素（使用缺省值）
print_checksum(a, 2, 4);    // a[2:4]
print_checksum(a, 1);       // 从 1 开始到最后
print_checksum(a, , 2);     // a[0:2]（low 用缺省值）
```

> 📌 用 `-1` 作为缺省值判断"调用时是否指定了该参数"是个好技巧。

### 3.4.5 采用名字进行参数传递

```systemverilog
task many(input int a = 1, b = 2, c = 3, d = 4);
    $display("%0d %0d %0d %0d", a, b, c, d);
endtask

initial begin
    many(6, 7, 8, 9);       // 6  7  8  9 — 指定所有
    many();                  // 1  2  3  4 — 全用缺省
    many(.c(5));             // 1  2  5  4 — 只指定 c
    many(, 6, .d(8));        // 1  6  3  8 — 混合方式
end
```

| 建议 | 说明 |
|------|------|
| 参数很少 | 位置传参 |
| 参数很多 | 名字传参更清晰 |
| 参数有默认值 | 名字传参更明确 |

### 3.4.6 常见的代码错误 — 参数继承陷阱

```systemverilog
// ❌ 错误：a 和 b 继承了前一个参数的 ref 方向
task sticky(ref int array[50], int a, b);
    // a 和 b 变成了 ref int！
    // 编译器不会警告，但对简单 int 用 ref 无必要且低效
endtask

// ✅ 正确：明确指定所有参数的方向
task sticky(ref int array[50], input int a, b);
endtask
```

> ⚠️ 只要参数列表中出现了非默认方向或非默认类型，后面的参数都建议显式写方向和类型。

---

## 3.5 子程序的返回

### 3.5.1 return 语句

task 和 function 都可以使用 `return`。

```systemverilog
// task 中提前返回
task load_array(int len, ref int array[]);
    if (len <= 0) begin
        $display("Bad len");
        return;                    // 提前退出
    end
    for (int i = 0; i < len; i++)
        array[i] = i;
endtask

// function 中返回值
function bit transmit();
    // ... 发送处理 ...
    return ~ifc.cb.error;          // 返回状态：0=error
endfunction
```

| 场景 | 作用 |
|------|------|
| 检测到错误 | 提前退出 |
| 减少嵌套 | 不用把后续代码全包进 else |
| 函数返回值 | 直接返回表达式 |

### 3.5.2 从函数中返回一个数组

三种方式：

**方式一：typedef 数组类型（有拷贝开销，适合小数组）**

```systemverilog
typedef int fixed_array5[5];

function fixed_array5 init(input int start);
    foreach (init[i])
        init[i] = i + start;
endfunction

initial begin
    fixed_array5 f5;
    f5 = init(5);                // 整个数组被复制
    foreach (f5[i])
        $display("f5[%0d] = %0d", i, f5[i]);
end
```

**方式二：ref 参数（无拷贝开销，推荐）**

```systemverilog
function void init(ref int f[5], input int start);
    foreach (f[i])
        f[i] = i + start;
endfunction

initial begin
    int fa[5];
    init(fa, 5);                 // 直接修改 fa，零拷贝
end
```

**方式三：包装到类中返回句柄**（见第 5 章）

| 方式 | 拷贝开销 | 适用场景 |
|------|:---:|------|
| typedef + 返回值 | 有 | 小数组 |
| ref 参数 | 无 | 大数组（推荐） |
| 类 + 句柄 | 无 | 面向对象场景 |

---

## 3.6 局部数据存储

### 3.6.1 静态存储 vs 自动存储

| 存储类型 | 行为 | 多线程安全 |
|---------|------|:---:|
| 静态（默认） | 所有调用共享同一份变量 | ❌ |
| automatic | 每次调用有独立的变量副本 | ✅ |

### 3.6.2 automatic 的作用

```systemverilog
program automatic test;

    task wait_for_mem(input [31:0] addr, expected_data, output success);
        while (bus.addr !== addr)
            @(bus.addr);
        success = (bus.data == expected_data);
    endtask
endprogram
```

如果不加 `automatic`，多线程同时调用 `wait_for_mem` 会共享参数，第二次调用会覆盖第一次调用的参数，导致阻塞条件永远无法触发。

> 📌 测试平台的 `program` 建议一律写成 `program automatic`。

### 3.6.3 变量初始化的静态陷阱

```systemverilog
// ❌ 漏洞：静态存储下局部变量在仿真开始时只初始化一次
program initialization;
    task check_bus;
        repeat (5) @(posedge clock);
        if (bus_cmd == 'READ) begin
            logic [7:0] local_addr = addr << 2;  // 只在仿真开始时初始化一次！
            $display("Local Addr = %h", local_addr);
        end
    endtask
endprogram
```

**两种修复方式：**

```systemverilog
// ✅ 方式1：program 声明为 automatic
program automatic initialization;
    // 局部变量每次进入时都会被重新初始化
endprogram

// ✅ 方式2：分离声明和初始化
logic [7:0] local_addr;
local_addr = addr << 2;    // 每次进入都会重新执行赋值语句
```

| 修复方式 | 说明 |
|---------|------|
| `program automatic` | 所有变量变成动态存储 |
| 分离声明和赋值 | 不在声明语句中给初值，每次进入时单独赋值 |

---

## 3.7 时间值

### 3.7.1 timeunit 和 timeprecision

替代传统的 `` `timescale`` 编译指令：

```systemverilog
module timing;
    timeunit 1ns;            // 时间单位：1 纳秒
    timeprecision 1ps;        // 时间精度：1 皮秒

    initial begin
        $timeformat(-9, 3, "ns", 8);
        // 参数：时间标度(-9=ns)、小数位数(3)、后缀("ns")、最小宽度(8)
        #1        $display("%t", $realtime);  // 1.000ns
        #2ns      $display("%t", $realtime);  // 3.000ns
        #0.1ns    $display("%t", $realtime);  // 3.100ns
        #41ps     $display("%t", $realtime);  // 3.141ns
    end
endmodule
```

### 3.7.2 带单位的时间值

```systemverilog
#1ns;         // 延时 1 纳秒
#20ps;        // 延时 20 皮秒
#0.1ns;       // 延时 0.1 纳秒
```

### 3.7.3 $timeformat 参数

```systemverilog
$timeformat(-9, 3, "ns", 8);
```

| 参数 | 含义 |
|------|------|
| `-9` | 显示单位为 ns |
| `3` | 小数点后 3 位 |
| `"ns"` | 时间后缀字符串 |
| `8` | 最小显示宽度 |

### 3.7.4 time 和 real 时间变量

```systemverilog
`timescale 1ps / 1ps
module ps;
    initial begin
        real rdelay = 800fs;   // real：0.800 精确存储
        time tdelay = 800fs;   // time：舍入后得 1（64位整数）

        $timeformat(-15, 0, "fs", 5);
        #r delay;               // 时延舍入为 1ps
        $display("%t", rdelay); // "800fs" → real 保持精度
        #t delay;               // 时延再次为 1ps
        $display("%t", tdelay); // "1000fs" → time 已舍入为 1
    end
endmodule
```

| 变量类型 | 小数精度 | 说明 |
|---------|:---:|------|
| `time`（64位整数） | ❌ 舍入 | 按时间精度舍入，丢失小数值 |
| `real`（实数） | ✅ 保留 | 保持小数精度 |

### 3.7.5 $time vs $realtime

| 系统函数 | 返回类型 | 特点 |
|---------|---------|------|
| `$time` | 64 位整数 | 按时间精度舍入，无小数部分 |
| `$realtime` | 实数 | 带小数部分的完整时间值 |

> 📌 测试平台需要精确时间（例如统计延时）时，优先使用 `$realtime`。

---

## 本章总结（3.1–3.8）

### 知识链

```
for / break / continue / do-while
  → task / function / void function
    → C 风格参数 + ref / const ref
      → 缺省参数值 + 命名参数传递
        → return 提前返回
          → automatic 动态存储
            → timeunit / $timeformat / $realtime
```

### 关键概念速查表

| 概念 | 作用 |
|------|------|
| `for (int i=0; ...)` | 循环变量作用域局部化 |
| `continue` / `break` | 跳过本轮 / 终止循环 |
| `function void` | 无返回值且不消耗时间，可被所有子程序调用 |
| `ref` | 引用传递，零拷贝，立即可见 |
| `const ref` | 只读引用，零拷贝 + 安全 |
| 缺省参数值 | 新调用不破坏已有代码 |
| 命名参数 (`.name`) | 只指定需要的参数 |
| `return` | 提前退出 task/function |
| `automatic` | 每次调用独立存储，多线程安全 |
| `$realtime` | 精确时间，保留小数 |

### 你写的 ch3 代码对照表

| 文件 | 对应章节 | 知识点 |
|------|---------|--------|
| `1.sv` | 3.1 | `for` 循环声明变量、`break`/`continue`、`do-while`、块标签 |
| `2.sv` | 3.2 | task / function / void function |
| `3.sv` | 3.4 | 值传递、`const ref`、`ref`、缺省参数、命名参数 |
| `4.sv` | 3.5 | task/function 中 `return` 提前返回 |
| `5.sv` | 3.6 | `automatic` 动态存储 vs 静态存储 |
| `6.sv` | 3.7 | `timeunit`/`timeprecision`、`$timeformat`、`$realtime` vs `$time` |

### 最重要的 10 条规则

| # | 规则 |
|---|------|
| 1 | `for` 循环变量直接在循环头声明：`for (int i=0; ...)` |
| 2 | 不消耗时间的调试子程序用 `function void`，不要用 `task` |
| 3 | 传递大数组用 `const ref`（只读）或 `ref`（读写），避免值拷贝 |
| 4 | 始终明确写出所有参数的类型和方向，不要依赖默认继承 |
| 5 | 用 `-1` 作为缺省值来区分"调用时是否指定了该参数" |
| 6 | 命名参数传递让多参数调用清晰可读：`.c(5)` |
| 7 | `return` 让 task/function 提前退出，减少 else 嵌套 |
| 8 | `program` 声明为 `automatic`，避免多线程变量覆盖 |
| 9 | 不在声明语句中初始化变量，或分开声明和赋值 |
| 10 | 精确时间用 `$realtime`，别用 `$time` |

### 最容易错的点

| 易错点 | 正确理解 |
|--------|---------|
| function 可以等待时钟 | 错，function 不能消耗时间 |
| 不耗时 task 随便写 | 更推荐写成 `function void` |
| 参数默认规则无害 | 错，可能继承前一个参数的方向和类型 |
| 大数组直接 input 传参 | 会复制数组，性能差 |
| `ref` 和 `output` 一样 | 错，`ref` 是实时引用，`output` 结束时才更新 |
| 局部变量声明时初始化总是安全 | 错，静态存储下可能只初始化一次 |
| 多线程调用 task 不需要 automatic | 错，可能共享参数导致覆盖 |
| `time` 能保存小数时间 | 错，`time` 是整数 |
| `$time` 和 `$realtime` 一样 | 错，`$realtime` 保留小数 |
