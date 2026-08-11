# SystemVerilog 第 2 章学习笔记：数据类型

> **核心结论**：SystemVerilog 在 Verilog 基础之上新增了双状态类型（`bit`/`byte`/`int` 等）、动态数组、队列、关联数组、数组方法（`sum`/`find`/`sort` 等）、结构体、枚举类型和字符串类型。这些改进让验证代码能以更高抽象层次编写，不用纠结比特层次的表示问题。

---

## 2.1 内建数据类型

### 2.1.1 logic 类型

**`logic`** 是 SystemVerilog 对 Verilog `reg` 的改进。它既可以被过程赋值，也可以被连续赋值、门级单元或模块驱动。

```systemverilog
module logic_demo(input logic rst_h);
    parameter int CYCLE = 20;
    logic q, q_l, d, clk, rst_l;

    initial begin
        clk = 0;                         // 过程赋值
        forever #(CYCLE/2) clk = ~clk;
    end
    assign rst_l = ~rst_h;               // 连续赋值
    not u_not(q_l, q);                   // 门级驱动
    my_dff u_dff(.q(q), .d(d), .clk(clk), .rst_l(rst_l));
endmodule
```

| 类型 | 特点 | 使用场景 |
|------|------|---------|
| `logic` | 只能有**一个**结构性驱动源 | 大多数 RTL/testbench 信号 |
| `wire` | 可以有多驱动源，进行解析 | 双向总线、多驱动信号 |

> ⚠️ `logic` 不能有多个驱动源。三态总线等信号必须用 `wire`。

### 2.1.2 双状态数据类型

```systemverilog
bit        b;        // 1 bit，双状态，无符号
bit [31:0] b32;      // 32 bit，无符号
int        i;        // 32 bit，有符号
int unsigned ui;     // 32 bit，无符号
byte       b8;       // 8 bit，有符号（范围 -128~127）
shortint   s;        // 16 bit，有符号
longint    l;        // 64 bit，有符号
real       r;        // 双精度浮点数
```

| 类型 | 可取值 | 示例 |
|------|------|------|
| 四状态 | 0/1/X/Z | `logic`、`reg`、`integer` |
| 双状态 | 0/1 | `bit`、`byte`、`shortint`、`int`、`longint` |

> ⚠️ `byte` 是有符号类型，最大值 127（不是 255）。随机化时可能产生非预期负值。想要 0~255 用 `bit [7:0]`。

### 2.1.3 检查 X/Z：`$isunknown`

双状态变量会把 X/Z 转换成 0/1，可能掩盖 DUT 的未知值。

```systemverilog
if ($isunknown(iport) == 1)
    $display("@%0t: X/Z detected on iport = %b", $time, iport);
```

---

## 2.2 定宽数组

### 2.2.1 声明和初始化

```systemverilog
int lo_hi[0:15];       // 16 个元素，下标 0~15
int c_style[16];        // 16 个元素，下标 0~15，C 风格

// 多维数组
int array2[0:7][0:3];   // 8 行 4 列
int array3[8][4];        // 紧凑写法
array2[7][3] = 1;        // 设置最后一个元素
```

> 📌 越界读取返回元素类型的缺省值（`logic`→X，`bit`/`int`→0，`wire` 无驱动→Z）。

### 2.2.2 常量数组初始化

```systemverilog
int ascend[4] = '{0, 1, 2, 3};            // 初始化 4 个元素
int descend[5];
descend = '{4, 3, 2, 1, 0};                // 为 5 个元素赋值
descend[0:2] = '{5, 6, 7};                 // 改前 3 个元素
ascend = '{4{8}};                          // {8,8,8,8}
descend = '{9, 8, default: 1};             // {9,8,1,1,1}
```

| 格式 | 含义 |
|------|------|
| `'{val1, val2, ...}` | 列表初始化 |
| `'{N{val}}` | N 个元素重复同一个值 |
| `'{..., default: val}` | 未显式赋值的元素取默认值 |

### 2.2.3 `for` 和 `foreach`

```systemverilog
bit [31:0] src[5], dst[5];

for (int i = 0; i < $size(src); i++)  // $size 返回数组宽度
    src[i] = i;

foreach (dst[j])                       // j 自动声明，作用域仅限于循环
    dst[j] = src[j] * 2;
```

**多维数组的 `foreach`**：

```systemverilog
int md[2][3] = '{'{0,1,2}, '{3,4,5}};

foreach (md[i, j])                     // 逗号隔开，同一个方括号里
    $display("md[%0d][%0d]=%0d", i, j, md[i][j]);

// 只遍历第一个维度
foreach (md[i]) begin
    foreach (md[, j])                  // 遍历第二个维度
        $write("%3d", md[i][j]);
    $display;
end
```

> ⚠️ 多维数组的语法是 `foreach(md[i,j])`，不是 `foreach(md[i][j])`。`foreach` 遍历原始声明的数组范围。

### 2.2.4 复制和比较

```systemverilog
bit [31:0] src[5] = '{0,1,2,3,4}, dst[5] = '{5,4,3,2,1};

if (src == dst)                        // 聚合比较
    $display("src == dst");
dst = src;                             // 聚合复制

if (src[1:4] == dst[1:4])              // 数组片段比较
    $display("src[1:4] == dst[1:4]");
```

> 📌 算术运算（如加法）不能用聚合操作，必须用循环。

### 2.2.5 同时使用位下标和数组下标

```systemverilog
bit [31:0] src[5] = '{5{5}};
$displayb(src[0],         // 'b101 或 'd5
          src[0][0],      // 'b1
          src[0][2:1]);   // 'b10
```

### 2.2.6 合并数组 `packed` vs 非合并 `unpacked`

| 类型 | 声明 | 存储方式 | 特点 |
|------|------|---------|------|
| 非合并 | `bit [7:0] b[4];` | 4 个独立 byte，有空隙 | 维度在变量名后 |
| 合并 | `bit [3:0][7:0] a;` | 32-bit 连续比特流 | 维度在类型和变量名之间 |

```systemverilog
// 合并数组：4 个字节合并成 32 比特
bit [3:0][7:0] bytes;
bytes = 32'hCafe_Dada;
$displayh(bytes, bytes[3], bytes[3][7]); // 整体、最高字节、最高位

// 混合：合并 + 非合并
bit [3:0][7:0] barray[3];    // 3 个合并元素，每个 32 比特
barray[0] = 32'h0123_4567;
barray[0][3]    = 8'h01;     // 最高字节
barray[0][1][6] = 1'b1;      // 读一个比特
```

| 场景 | 推荐 |
|------|------|
| 需要整体当成向量操作 | 合并数组 |
| 需要 `@` 等待数组变化 | 合并数组（`@` 只能用于标量或合并数组） |
| 普通元素列表 | 非合并 / 动态数组 / 队列 |

---

## 2.3 动态数组

```systemverilog
int dyn[], d2[];

initial begin
    dyn = new[5];              // 分配 5 个元素
    foreach (dyn[i])
        dyn[i] = i;            // 初始化

    d2 = dyn;                  // 复制（两个数组独立）
    d2[0] = 99;
    $display("dyn[0]=%0d d2[0]=%0d", dyn[0], d2[0]); // 0 vs 99

    dyn = new[20](dyn);        // 扩容到 20，保留旧数据
    dyn = new[100];            // 分配 100 个新元素，旧数据丢弃
    dyn.delete();              // 删除所有元素
end
```

| 操作 | 说明 |
|------|------|
| `new[N]` | 分配 N 个元素 |
| `new[N](old)` | 分配 N 个元素，复制 old 的内容到前部 |
| `size()` | 返回当前宽度 |
| `delete()` | 删除所有元素 |

---

## 2.4 队列

```systemverilog
int q[$]  = {0, 2, 5};         // 队列常量不需要单引号
int q2[$] = {3, 4};
int j = 1;

initial begin
    q.insert(1, j);            // {0,1,2,5}  在位置 1 前插入
    q.insert(3, q2);           // {0,1,2,3,4,5}
    q.delete(1);               // {0,2,3,4,5} 删除位置 1

    q.push_front(6);           // {6,0,2,3,4,5}
    j = q.pop_back();          // {6,0,2,3,4}, j=5
    q.push_back(8);            // {6,0,2,3,4,8}
    j = q.pop_front();         // {0,2,3,4,8}, j=6

    q.delete();                // {} 清空
end
```

### 使用下标串联代替方法

```systemverilog
q = {q[0], j, q[1:$]};     // 在 1 前插入 j
q = {q[0:2], q2, q[3:$]};  // 插入整个队列
q = {q[0], q[2:$]};        // 删除 q[1]
q = {6, q};                // 队首插入
q = {q, 8};                // 队尾插入
j = q[$]; q = q[0:$-1];    // 队尾弹出
j = q[0]; q = q[1:$];      // 队首弹出
q = {};                    // 清空
```

| 性能 | 说明 |
|------|------|
| 首尾插入/删除 | O(1)，恒定时间 |
| 中间插入/删除 | O(n)，需要搬移元素 |

> 📌 队列适合做 scoreboard、事务缓冲区。不需要使用链表（2.6节），队列在所有方面更好。

---

## 2.5 关联数组

用于稀疏存储。只在被写入时才分配空间，不对整个地址范围分配。

### 以 bit 向量为索引

```systemverilog
bit [63:0] assoc[bit [63:0]], idx = 1;

initial begin
    repeat (8) begin
        assoc[idx] = idx;         // 写入稀疏元素：1,2,4,8...
        idx = idx << 1;
    end

    foreach (assoc[i])
        $display("assoc[%h] = %h", i, assoc[i]);

    // first/next 遍历
    if (assoc.first(idx))
        do
            $display("assoc[%h]=%h", idx, assoc[idx]);
        while (assoc.next(idx));

    assoc.first(idx); assoc.delete(idx);
end
```

### 以字符串为索引

```systemverilog
int switch[string];

initial begin
    switch["min_address"] = 42;
    switch["max_address"] = 1492;

    if (switch.exists("max_address"))
        $display("max = %0d", switch["max_address"]);
    else
        $display("max = 1000");

    foreach (switch[s])
        $display("switch['%s'] = %0d", s, switch[s]);
end
```

| 方法 | 说明 |
|------|------|
| `first(idx)` / `next(idx)` | 遍历 |
| `exists(key)` | 检查索引是否存在 |
| `delete(key)` / `delete()` | 删除指定/全部元素 |
| `num()` | 返回元素个数 |

> 📌 关联数组适合：超大稀疏内存、命令名→操作码映射、配置项存储。

---

## 2.6 链表

> ⚠️ **不用学**。SystemVerilog 的队列在所有方面都优于链表。能用队列就不要用链表。

---

## 2.7 数组的方法

适用于所有非合并数组：定宽数组、动态数组、队列、关联数组。

### 2.7.1 缩减方法

```systemverilog
bit on[10];
int total;

foreach (on[i]) on[i] = i;

$display("on.sum = %0d", on.sum());            // 单比特和 → 可能溢出！
$display("on.sum = %0d", on.sum() + 32'd0);    // 加 32位常数 → 正确
total = on.sum();                               // 赋给 32位变量 → 正确
$display("int sum = %0d", on.sum() with (int'(item)));  // with 强制位宽
```

| 方法 | 说明 |
|------|------|
| `sum()` | 所有元素求和 |
| `product()` | 所有元素求积 |
| `and()` / `or()` / `xor()` | 按位缩减 |

> ⚠️ 单比特数组的 `sum()` 结果也是单比特！用 `with (int'(item))` 或赋给足够宽的变量。

### 2.7.2 定位方法

```systemverilog
int d[] = '{9, 1, 8, 3, 4, 4}, tq[$];

tq = d.min();                          // {1} — 返回队列
tq = d.max();                          // {8}
tq = d.unique();                       // {9, 1, 8, 3, 4} 去重

tq = d.find() with (item > 3);             // {9, 8, 4, 4}
tq = d.find_index() with (item > 3);       // {0, 2, 4, 5} 返回索引
tq = d.find_first() with (item == 4);      // {4}
tq = d.find_first_index() with (item==8);  // {2}
tq = d.find_last() with (item == 4);       // {4}
tq = d.find_last_index() with (item == 4); // {5}
```

| with 中的 `item` | 含义 |
|------|------|
| 默认名 `item` | 代表数组中当前遍历的元素 |
| 自定义名 | `d.find(x) with (x > 3)` |

### 带条件的 sum

```systemverilog
count = d.sum() with (item > 7);             // 2：计数 >7
total = d.sum() with ((item > 7) * item);     // 17 = 9+8
total = d.sum() with (item < 8 ? item : 0);   // 12 = 1+3+4+4
```

### 2.7.3 排序方法

```systemverilog
int d[] = '{9, 1, 8, 3, 4, 4};

d.reverse();    // '{4,4,3,8,1,9} — 改变原数组
d.sort();       // '{1,3,4,4,8,9}
d.rsort();      // '{9,8,4,4,3,1}
d.shuffle();    // 随机打乱
```

> 📌 定位方法创建新队列返回；排序方法修改原数组。`reverse` 和 `shuffle` 不能带 `with`。

### 2.7.4 数组方法建立记分板

```systemverilog
typedef struct packed {
    bit [7:0]  addr;
    bit [7:0]  pr;
    bit [15:0] data;
} Packet;

Packet scb[$];

function void check_addr(bit [7:0] addr);
    int intq[$];
    intq = scb.find_index() with (item.addr == addr);
    case (intq.size())
        0: $display("Addr %h not found", addr);
        1: scb.delete(intq[0]);                        // 匹配成功
        default: $display("ERROR: Multiple hits");
    endcase
endfunction
```

---

## 2.8 选择存储类型

| 需求 | 推荐类型 | 理由 |
|------|---------|------|
| 索引连续、编译时已知 | 定宽数组 | 最快 |
| 索引连续、运行时才知道 | 动态数组 | 灵活 |
| 元素数量频繁变化 | 队列 | 首尾 O(1) |
| 稀疏索引（随机地址） | 关联数组 | 按需分配 |
| 超大稀疏存储器建模 | 关联数组 | 只存写入的值 |
| 字符串→值映射 | 字符串索引关联数组 | 天然映射 |
| 记分板（频繁增删） | 队列 | 增删方便 |

| 类型 | 灵活性 | 内存效率 | 存取速度 |
|------|:---:|:---:|:---:|
| 定宽 | 低 | 最高 | 最快 |
| 动态 | 中 | 高 | 快 |
| 队列 | 高 | 中 | 首尾快，中间慢 |
| 关联 | 最高 | 低 | 最慢 |

---

## 2.9 使用 `typedef` 创建新类型

```systemverilog
parameter int OPSIZE = 8;
typedef reg [OPSIZE-1:0] opreg_t;     // 创建新类型
opreg_t op_a, op_b;                   // 等价于 reg [7:0] op_a, op_b

// 常用：32 位双状态无符号
typedef bit [31:0] uint;
typedef int unsigned uint;             // 等效定义

// 数组类型 — 维度写在类型名后面
typedef int fixed_array5[5];
fixed_array5 f5;
```

> 📌 约定：自定义类型后缀 `_t`，结构后缀 `_s`，枚举后缀 `_e`，联合后缀 `_u`。

---

## 2.10 结构体 `struct`

### 2.10.1 基本定义和初始化

```systemverilog
typedef struct {
    bit [7:0] r, g, b;
} pixel_s;

pixel_s my_pixel;
my_pixel.r = 8'hff;

// 初始化
typedef struct { int a; byte b; shortint c; int d; } my_struct_s;
my_struct_s st = '{32'haaaa_aaaa, 8'hbb, 16'hcccc, 32'hdddd_dddd};
```

### 2.10.2 合并结构 `packed struct`

```systemverilog
typedef struct packed {
    bit [7:0] r, g, b;
} pixel_p_s;                          // 3 字节连续存储，无空隙，整体 24 bit

pixel_p_s p;
p = 24'hff_80_00;
$display("r=%h g=%h b=%h", p.r, p.g, p.b);
```

| 类型 | 存储特点 | 适用场景 |
|------|---------|------|
| 普通 struct | 成员独立存储，可能有空隙 | 软件风格数据组织 |
| packed struct | 连续比特存储，可整体当向量 | 寄存器、指令、硬件字段 |

### 2.10.3 联合 `union`

```systemverilog
typedef union { int i; real f; } num_u;
num_u un;
un.f = 0.0;  // 设置浮点值
```

> ⚠️ 不要为了省几个字节就用 union。更推荐用带判别变量的类。

---

## 2.11 类型转换

### 2.11.1 静态转换 `type'(expr)`

```systemverilog
int i;
real r;
i = int'(10.0 - 0.1);    // 不做检查，可能丢数据
r = real'(42);
```

### 2.11.2 动态转换 `$cast`

```systemverilog
typedef enum {RED, BLUE, GREEN} color_e;
color_e color;
int c = 2;

if (!$cast(color, c))     // 有边界检查，失败返回 0，不修改目标
    $display("cast failed");
```

> 📌 整数→枚举必须显式转换，推荐用 `$cast`（有检查），别用静态转换（无检查）。

### 2.11.3 流操作符 `>>` 和 `<<`

```systemverilog
bit [7:0] j[4] = '{8'ha, 8'hb, 8'hc, 8'hd};
int h;

h = {>> {j}};          // 0a0b0c0d — 大端打包（最常用）
h = {<< {j}};          // b030d050 — 位倒序
h = {<<byte{j}};       // 0d0c0b0a — 字节倒序

// 拆分到变量
bit [7:0] q, r, s, t;
{>> {q, r, s, t}} = j;  // 分散到四个变量

// 队列间转换（字 → 字节）
bit [15:0] wq[$] = {16'h1234, 16'h5678};
bit [7:0]  bq[$];
bq = {>> {wq}};        // {12, 34, 56, 78}
```

| 操作符 | 方向 | 说明 |
|------|------|------|
| `>>` | 从左到右 | 大端序（MSB 在先，最常用） |
| `<<` | 从右到左 | 小端序/位倒序 |
| `<< N` | 按 N 位分段 | 指定片段宽度 |

> ⚠️ 数组声明 `[256]` 等价于 `[0:255]`，但很多数组用 `[high:low]` 声明。下标方向不一致会导致流操作结果颠倒。

---

## 2.12 枚举类型

### 2.12.1 基本用法

```systemverilog
typedef enum {INIT, DECODE, IDLE} fsmstate_e;   // 默认 0,1,2
fsmstate_e pstate, nstate;

case (pstate)
    IDLE   : nstate = INIT;
    INIT   : nstate = DECODE;
    default: nstate = IDLE;
endcase

$display("Next state is %s", nstate.name());  // 打印状态名
```

| 优点 | 说明 |
|------|------|
| 可读性强 | `IDLE` 比 `2'b10` 清楚 |
| `.name()` | 调试时可打印符号名 |
| 类型检查 | 比宏/parameter 更安全 |

### 2.12.2 指定枚举值和避坑

```systemverilog
typedef enum {INIT, DECODE=2, IDLE} fsm_e; // INIT=0, DECODE=2, IDLE=3
```

> ⚠️ 枚举变量默认初始化为 0。确保 0 对应一个合法枚举名：
> ```systemverilog
> // ❌ FIRST=1 开始，默认值 0 非法
> typedef enum {FIRST=1, SECOND, THIRD} ordinal_e;
> // ✅ 预留 0
> typedef enum {BAD_O=0, FIRST=1, SECOND, THIRD} ordinal_e;
> ```

### 2.12.3 枚举方法

```systemverilog
typedef enum {RED, BLUE, GREEN} color_e;
color_e color;

color = color.first();     // RED
color = color.last();      // GREEN
color = color.next();      // 下一个（环形绕回）
color = color.next(2);     // 往后第 2 个
color = color.prev();      // 前一个

// 遍历所有成员
color = color.first();
do begin
    $display("%0d/%s", color, color.name());
    color = color.next();
end while (color != color.first());  // 环形绕回时完成
```

### 2.12.4 枚举类型转换

```systemverilog
color = BLUE;               // 合法常量直接赋值
c = color;                  // 枚举→整数：自动
if (!$cast(color, c))       // 整数→枚举：必须显式！推荐 $cast
    $display("cast failed");
// color = color_e'(c);     // 静态转换：不推荐，越界也不报错
```

---

## 2.13 常量 `const`

```systemverilog
initial begin
    const byte colon = ":";
    // colon = ";";    // ❌ 编译错误：const 变量不能修改
end
```

| 方式 | 特点 | 建议 |
|------|------|------|
| 宏 `` `define`` | 文本替换，全局 | 少用，易冲突 |
| `parameter` | 有作用域，可带类型 | 配置常量 |
| `typedef` | 创建类型别名 | 替代类型宏 |
| `const` | 初始化后不可改 | 局部不可变值 |

---

## 2.14 字符串 `string`

```systemverilog
string s;

initial begin
    s = "IEEE ";
    $display("%0d", s.getc(0));      // 73 ('I')
    $display("%s", s.tolower());     // "ieee "

    s.putc(s.len()-1, "-");          // 改最后一个字符
    s = {s, "P1800"};                // 字符串串接
    $display("%s", s.substr(2, 5));  // "EE-P"

    my_log($psprintf("%s %5d", s, 42));
end

task my_log(string message);
    $display("@%0t: %s", $time, message);
endtask
```

| 特点 | 说明 |
|------|------|
| 动态长度 | 不需要预先声明宽度 |
| 单个字符是 `byte` | 用 `getc`/`putc` 访问 |
| 不以 `\0` 结尾 | 不同于 C 字符串 |
| 可直接拼接 | `{s1, s2}` |

| 方法 | 说明 |
|------|------|
| `s.len()` | 返回长度 |
| `s.getc(N)` | 取位置 N 的字符（byte） |
| `s.putc(N, C)` | 将字符 C 写到位置 N |
| `s.toupper()` / `s.tolower()` | 大小写转换 |
| `s.substr(start, end)` | 提取子串 |
| `$psprintf(...)` | 格式化（替代 `$sformat`） |

---

## 2.15 表达式的位宽

```systemverilog
bit one = 1'b1;
bit [7:0] b8;

// ❌ 单比特 + 单比特 = 单比特溢出：1+1=0
$displayb(one + one);

// ✅ 方法 1：赋值给宽位变量
b8 = one + one;                 // 1+1=2

// ✅ 方法 2：加宽位常数
$displayb(one + one + 2'b0);    // 1+1=2

// ✅ 方法 3：强制类型转换
$displayb(2'(one) + one);       // 1+1=2
```

> ⚠️ 小位宽表达式做运算时务必主动扩展位宽，否则容易溢出。

---

## 本章总结

### 你写的 ch2 代码对照表

| 文件 | 对应章节 | 知识点 |
|------|---------|--------|
| `1.sv` | 2.1 | `logic` vs `bit`、`$isunknown()` 检测 X/Z |
| `2.sv` | 2.5 | 关联数组（地址索引）、`first()`/`next()` 遍历、`delete()` |
| `3.sv` | 2.2.6 | 非合并 vs 合并数组 |
| `4.sv` | 2.3 | 动态数组 `new[]`、`sum()`、`delete()`、复制 |
| `5.sv` | 2.4, 2.7 | 队列操作、`find with`、`unique()`、`min/max`、`sum with` |
| `6.sv` | 2.5 | 关联数组（字符串+地址索引）、`exists()` |
| `7.sv` | 2.10, 2.11 | 结构体、流操作符 `{>>{}}` 大端打包、`{<<{}}` 小端打包 |
| `8.sv` | 2.12 | 枚举类型、`.next()`/`.prev()`/`.first()`、环形遍历 |

### 数据类型速查

| 需求 | 推荐 | 需求 | 推荐 |
|------|------|------|------|
| 普通信号 | `logic` | 多驱动信号 | `wire` |
| 高性能 | `bit`/`int` | X/Z 检查 | `logic`+`$isunknown` |
| 定长数组 | 定宽数组 | 运行时定长 | 动态数组 |
| 频繁增删 | 队列 | 稀疏存储 | 关联数组 |
| 字段组合 | `struct` | 状态码 | `enum` |

### 最重要的 10 条规则

1. 默认用 `logic`，多驱动用 `wire`
2. `byte` 有符号（-128~127），0~255 用 `bit [7:0]`
3. 双状态变量连 DUT 输出时用 `$isunknown` 检查 X/Z
4. 单比特数组 `sum()` 也用单比特！用 `with (int'(item))` 或赋给宽变量
5. 队列首尾 O(1)，中间 O(n)，别用链表；`foreach(md[i,j])` 不是 `[i][j]`
6. 枚举默认从 0 开始，确保 0 对应合法枚举名
7. 整数→枚举用 `$cast()`（有检查），别用静态转换
8. 小位宽运算主动扩展位宽（赋给宽变量/加常数/强制转换）
9. `inside` 集合重复值不增加概率，用 `dist` 加权
10. `typedef int[5] arr_t` 错，数组维度应写在类型名后
| `one + one` 一定 = 2 | 错，单 bit 表达式可能溢出成 0 |
