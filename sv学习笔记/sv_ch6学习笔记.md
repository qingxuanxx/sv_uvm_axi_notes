# SystemVerilog 第 6 章学习笔记：随机化

> **核心结论**：6.1–6.14 的主线就是用 `rand/randc + constraint + randomize()` 生成"合法但多样"的事务级激励，再通过概率控制、约束开关、前后处理函数、数组约束和事务序列，把单个随机包扩展成真实验证场景。这部分是后续 UVM sequence、coverage-driven verification 的基础。

---

## 知识网络

```
                    ┌── rand / randc 随机变量
                    │
CRT ──→ constraint ──→ randomize() ──→ 事务级激励
          │                │
          ├── inside       ├── constraint_mode()
          ├── dist         ├── randomize() with
          ├── if / ->      ├── rand_mode()
          └── solve before ├── pre/post_randomize()
                           │
                           └── 数组约束 ──→ 事务序列 ──→ 真实验证场景
```

---

## 6.1–6.3：随机化的目的与基本机制

### 为什么需要 CRT（受约束随机测试）

| 定向测试 | CRT |
|---------|-----|
| 只能测你预料到的 Bug | 能发现你预料不到的 Bug |
| 功能项倍增时无法扩展 | 自动产生大量有效激励 |
| 人工检查结果 | 参考模型自动比对 |
| 编写大量独立测试用例 | 一套代码 + 改变 seed = 无数测试 |

> **核心思路**：改变随机种子（seed）就能改变整个测试行为，用 CPU 时间换人工检查时间。

### 6.2 需要随机化的 8 个方面

| # | 方面 | 说明 | 典型例子 |
|---|------|------|---------|
| 1 | 器件配置 | DUT 内部寄存器和模式 | 通道数量、时钟分频、工作模式 |
| 2 | 环境配置 | 连接DUT的外部环境 | 连接器件个数、拓扑结构 |
| 3 | 原始输入数据 | 总线写数据、信元填充 | 数据包 payload |
| 4 | 封装后的输入 | 多层协议控制域 | TCP→IP→Ethernet 各层头部 |
| 5 | 协议异常 | 通信中断、超时 | 握手信号不按预期到达 |
| 6 | 延时 | 协议范围内的随机延时 | 总线 grant 在请求后 1~3 周期到达 |
| 7 | 事务状态 | 多事务交错的随机顺序 | burst 传输中穿插中断 |
| 8 | 错误和违规 | 设计规范之外的输入 | 注入错误 CRC、非法操作码 |

> 📌 不要只随机 data，控制路径、配置路径、异常路径更容易出 Bug。

### 6.3 最小语法模板

```systemverilog
class Packet;
    rand  bit [7:0] addr;      // rand  — 随机，可重复（掷骰子）
    rand  bit [7:0] data;
    randc bit [1:0] kind;      // randc — 周期随机，值取完一圈才重复（发牌）

    constraint c_addr {
        addr inside {[0:100]};
    }
endclass

Packet p = new();
initial begin
    assert(p.randomize())       // 返回 1=成功, 0=失败
        else $fatal("randomize failed");
end
```

| 语法 | 含义 | 类比 |
|------|------|------|
| `rand` | 每次随机化都重新取值，可能重复 | 掷骰子 |
| `randc` | 周期随机，所有可能值轮完一圈才重复 | 发牌（洗牌后再发） |
| `constraint` | 限制随机值范围，保证激励合法 | 合法空间的边界 |
| `randomize()` | 调用约束求解器产生满足约束的值 | 在合法空间内随机选点 |
| `assert(obj.randomize())` | **必须**检查随机化是否成功 | 失败时变量可能为非法值 |

### 什么可以随机化？

| 可以随机化 ✅ | 不可以随机化 ❌ |
|-------------|--------------|
| `bit`, `logic`, `reg`（整型变量） | `string`（字符串—求解器无法对文本数学求解） |
| `int`, `byte`, `shortint`, `longint` | 句柄/指针（不能 `rand MyClass obj;`） |
| 枚举类型 | 实数（早期标准限制） |

### 随机函数一览

| 函数 | 说明 | 示例 |
|------|------|------|
| `$random()` | 32 位有符号随机数 | `a = $random();` |
| `$urandom()` | 32 位无符号随机数 | `b = $urandom();` |
| `$urandom_range(max, min)` | 指定范围的无符号随机数 | `c = $urandom_range(0, 3);` |
| `$urandom_range(n)` | 等价于 `$urandom_range(n, 0)` | `d = $urandom_range(255);` |
| `$dist_exponential()` | 指数分布，构造非线性延时 | `#($dist_exponential(100));` |

### ⚠️ 三条铁律

1. **不能在构造函数 `new()` 里调用 `randomize()`** — 随机化前需要调整约束、权重或模式
2. **必须检查 `randomize()` 返回值** — 失败返回 0，不检查会导致使用非法值
3. **同一个 seed + 同一个测试平台 → 相同结果** — CRT 是可重现的

---

## 6.4：约束 `constraint` 是本章核心

约束是**声明式代码**，不是顺序执行的。所有约束同时有效，求解器**并行**求解。

### 6.4.1 简单表达式

```systemverilog
class Order;
    rand bit [15:0] lo, med, hi;

    constraint c_good {
        lo < med;     // 每个表达式只能有一个关系操作符
        med < hi;
    }
    // ❌ 不能写 lo < med < hi — 这会被解析为 (lo < med) < hi（语义错误）
endclass
```

> ⚠️ 约束里不能写赋值语句。要固定变量值，写 `len == 42;`，不要写 `len = 42;`。

### 6.4.2 权重分布 `dist`

```systemverilog
rand int src, dst;
constraint c_dist {
    // := 每个值的权重相同
    src dist {0 := 40, [1:3] := 60};
    // src=0 概率 40/220, src=1/2/3 各 60/220

    // :/ 权重均分到范围内每个值
    dst dist {0 :/ 40, [1:3] :/ 60};
    // dst=0 概率 40/100, dst=1/2/3 各 20/100
}
```

| 写法 | 含义 | 示例 |
|------|------|------|
| `val := weight` | 该值权重为 weight | `0 := 40` |
| `[lo:hi] := weight` | 范围内每个值权重相同 | `[1:3] := 60` → 1/2/3 各 60 |
| `[lo:hi] :/ weight` | 总权重为 weight，均分到每个值 | `[1:3] :/ 60` → 1/2/3 各 20 |

### 6.4.3 动态改变权重

```systemverilog
class BusOp;
    typedef enum {BYTE, WORD, LWRD} length_e;
    rand length_e len;
    bit [31:0] w_byte = 1, w_word = 3, w_lwrd = 5;  // 运行时可变

    constraint c_len {
        len dist {BYTE := w_byte, WORD := w_word, LWRD := w_lwrd};
    }
endclass
```

### 6.4.4 集合 `inside`

```systemverilog
constraint c_range {
    c inside {[lo:hi]};                // lo ≤ c ≤ hi
    b inside {[$:4], [20:$]};          // $ = 位宽范围内的极值
    s inside {0, [2:10], [100:107]};   // 混合离散+范围
    !(c inside {[lo:hi]});             // 取反 = 集合之外
}
```

> ⚠️ `inside` 集合中**重复值不会增加选中概率**（集合具有互异性），要用 `dist` 做加权。`low > high` 会产生空集合导致约束失败。

### 6.4.5 条件约束

```systemverilog
// -> 蕴含操作符（适合枚举/布尔条件）
// 语法：(条件) -> 约束;
(io_space_mode) -> addr[31] == 1'b1;
// 含义：如果条件为真，则约束必须成立；如果条件为假，忽略约束

// if-else（适合真假分支）
if (op == READ)
    len inside {[BYTE:LWRD]};
else
    len == LWRD;
```

| 方式 | 适用场景 |
|------|---------|
| `->` | 单条约束的条件化，简洁 |
| `if-else` | 多分支的条件约束 |

### 6.4.6 双向约束

约束是**声明式、并行、双向**的，不是顺序程序：

```systemverilog
rand logic [15:0] r, s, t;
constraint c_bidir {
    r < t;  t < 30;  s == r;  s > 25;
}
// 求解器同时计算四个表达式：
// s > 25 → r > 25（因为 s==r）→ r < t → t ≥ 27
// t < 30 → t ∈ [27, 29]
// 没有直接约束 t 下限，但通过双向反推自动得出
```

> 📌 即使 `->` 和 `if-else` 看起来像程序语句，它们也是双向的。`{(a==1) -> (b==0)}` 等价于 `{!(a==1) || b==0;}`，如果增加 `b==1`，求解器会把 a 置为 0。

### 6.4.7 `solve...before` 引导概率

```systemverilog
class Prob;
    rand bit x;         // 0 或 1
    rand bit [1:0] y;   // 0, 1, 2, 3
    constraint c { (x==0) -> y==0; }
    // solve x before y;   // 先求解 x，再根据 x 求 y
endclass
```

| 情况 | x=0 概率 | y=0 概率 |
|------|---------|---------|
| 无 `solve before` | 1/2 | 5/8 |
| `solve x before y` | 1/2 | 5/8（可能不同） |
| `solve y before x` | 1/5 | 1/2（完全不同！） |

> ⚠️ 不要随意使用 `solve before`，只有概率分布明显不符合测试目标时才用，否则降低可读性和求解效率。它**不改变合法解集合**，只改变概率分布。

### 6.4.8 约束块性能优化

| 慢 ❌ | 快 ✅ | 原因 |
|------|------|------|
| `addr % 4096` | `addr[11:0]` | 位提取代替取模运算 |
| `addr / 4096` | `addr >> 12` | 移位代替除法 |
| 32位乘法/除法 | 位宽限制后的运算 | 求解器处理大位宽运算很慢 |

---

## 6.5：解的概率

### 关系操作对概率的影响

```systemverilog
// 无约束的类：x(0/1) × y(0/1/2/3) = 8 种解，各 1/8
class Unconstrained;
    rand bit x;
    rand bit [1:0] y;
endclass

// 带关系约束的类：(x==0) -> y==0
class Imp1;
    rand bit x;
    rand bit [1:0] y;
    constraint c_xy { (x==0) -> y==0; }
endclass
// 8 种原始组合 → 5 种有效解
// x=0,y=0：概率 1/2（4 种情况合并）
// x=1,y=0/1/2/3：各 1/8
```

### 再增加 `y > 0` 约束，双向反推

```systemverilog
constraint c { y > 0; (x==0) -> y==0; }
// y>0 排除了 y=0 的解
// 为了满足 (x==0)->y==0，当 y 非零时 x 必须为 1
// 结果：x=1 概率 1，y=1/2/3 各 1/3
```

| 概念 | 说明 |
|------|------|
| 解空间 | 所有满足约束的组合 |
| 默认概率 | 求解器不保证直觉上的均匀分布 |
| `solve before` | 只改变求解顺序和概率分布，不改变合法解集合 |
| 关系操作 | 会合并解集（条件成立时多种情况并入一条） |

---

## 6.6–6.8：控制约束与临时约束

| 小节 | 核心机制 | 用途 |
|------|---------|------|
| 6.6 控制多个约束块 | `constraint_mode()` | 打开/关闭某个约束块 |
| 6.7 有效性约束 | `valid_xxx` 约束 | 保证默认生成合法事务 |
| 6.8 内嵌约束 | `randomize() with {}` | 本次随机化临时追加约束 |

### 6.6 控制约束块开关

```systemverilog
class Packet;
    rand int len;
    constraint c_short { len inside {[1:10]}; }   // 短包约束
    constraint c_long  { len inside {[100:200]}; } // 长包约束
endclass

Packet p = new();
p.constraint.c_long.constraint_mode(0);   // 关闭长包约束
assert(p.randomize());                    // 现在按短包随机

p.constraint_mode(0);                     // 关闭对象所有约束
assert(p.randomize());                    // len 没有任何限制
```

### 6.7 有效性约束的设计模式

区分两类约束：
- **协议有效性约束**：保证激励本身合法（如 CRC 正确、帧格式正确），通常不关闭
- **测试专用约束**：针对特定测试场景（如只测短包），可根据需要开关

### 6.8 内嵌约束 `randomize() with`

```systemverilog
// 在调用处临时添加约束
assert(p.randomize() with {
    len inside {[4:8]};
    addr >= 100;
});
```

| 场景 | 推荐做法 |
|------|---------|
| 默认生成合法事务 | 类中写有效性约束 |
| 测试错误处理 | 临时关闭 `valid_xxx` 约束 |
| 某个 testcase 要特殊值 | 用 `randomize() with {}` |
| 类内约束冲突 | 用 `constraint_mode()` 关掉冲突约束 |

> ⚠️ `randomize() with` 只能**增加限制**（缩小合法空间），不能覆盖类里已有约束。如果类里已约束 `addr inside {[0:100]}`，再写 `with {addr == 200;}` 会失败。

---

## 6.9–6.10：前后处理与随机控制

### 执行生命周期

```
pre_randomize()  →  约束求解  →  post_randomize()
   (设置条件)         (rand/randc赋值)   (计算衍生字段)
```

| 机制 | 执行时机 | 典型用途 |
|------|---------|---------|
| `pre_randomize()` | `randomize()` 求解前 | 动态调整参数上限、权重 |
| `post_randomize()` | `randomize()` 求解后 | 计算 checksum、CRC、ECC、保存历史值 |
| `rand_mode(0)` | 随机化前关闭 | 固定某个变量，只随机其他 |
| `randomize(null)` | 随时 | 只检查不改变：当前值是否合法？ |
| `randomize(a, b)` | 随机化时 | 只随机指定变量，其余当状态变量 |

```systemverilog
class Packet;
    rand bit [31:0] addr;
    rand bit [31:0] data;
    bit [31:0] checksum;

    function void pre_randomize();
        max_len = $urandom_range(8, 64);  // 随机决定本次约束上限
    endfunction

    function void post_randomize();
        checksum = data ^ addr;           // 计算衍生字段（CRC等）
    endfunction
endclass
```

### 固定变量/只检查/只随机部分

```systemverilog
// 固定单个变量 → 其他变量仍然随机
p.len.rand_mode(0);
p.len = 42;
assert(p.randomize());           // addr/data 随机，len 固定为 42

// 只检查当前值是否满足约束（不改值）
assert(p.randomize(null));       // 类似"验证"而非"生成"

// 只随机部分变量
assert(p.randomize(addr, data)); // kind 保持原来的值不变
```

> ⚠️ `rand_mode(0)` 不会清除变量旧值，但如果固定的值违反约束，`randomize()` 仍会失败。`pre_randomize()` 和 `post_randomize()` 是函数，不能写 `#`、`@`、`wait`。

---

## 6.11：常见错误与调试

| # | 问题 | 现象 | 正确做法 |
|---|------|------|---------|
| 1 | 约束矛盾 | `randomize()` 返回 0 | 逐块关约束定位冲突 |
| 2 | 使用有符号类型 | `byte`/`int` 可能生成负数 | 用 `bit [N:0]` 无符号类型 |
| 3 | 链式比较 `lo<mid<hi` | 语义错误，结果不可预知 | 拆成 `lo<mid; mid<hi;` |
| 4 | 空集合 `[low:high]` | `low > high` 导致无解 | `randomize()` 前检查上下限 |
| 5 | 不检查返回值 | 随机化失败后还继续仿真 | 始终 `assert(randomize())` |
| 6 | `with` 想覆盖内部约束 | 随机化失败 | `constraint_mode(0)` 后再 `with` |
| 7 | 变量位宽过大 | 溢出仍满足约束 | 根据真实范围限制位宽 |
| 8 | 句柄未 `new()` | 运行时崩溃 | randomize 前检查 `!= null` |

```systemverilog
// ❌ 不推荐：byte 是有符号类型，可能产生负的包长
rand byte pkt_len;
constraint c { pkt_len inside {[1:64]}; } // -128~-1 也在范围内！

// ✅ 推荐：明确无符号范围
rand bit [7:0] pkt_len;
constraint c { pkt_len inside {[1:64]}; } // 只有 1~64
```

> ⚠️ 有符号变量会让求解器产生你意想不到的合法数学解（如负数包长），在随机约束中避免使用有符号类型。

---

## 6.12–6.13：数组约束

### 6.12 foreach 约束

| 知识点 | 语法 | 作用 |
|------|------|------|
| 约束数组大小 | `arr.size() inside {[1:10]};` | 控制动态数组/队列长度 |
| 约束每个元素 | `foreach(arr[i]) arr[i] inside {[0:255]};` | 对每个元素加范围约束 |
| 数组求和 | `arr.sum() with (int'(item)) < 1024;` | 控制总长度、脉冲总数 |
| 相邻元素关系 | `arr[i] > arr[i-1];`（`i>0`） | 生成递增序列 |
| 唯一元素 | 辅助 `randc` 类 | 生成元素不重复的数组 |
| 句柄数组 | 每个元素必须先 `new()` | 求解器不会自动创建对象实例 |

```systemverilog
constraint c_array {
    data.size() inside {[1:8]};          // 数组大小在 1~8 之间

    foreach (data[i])
        data[i] inside {[0:255]};        // 每个元素 0~255

    data.sum() with (int'(item)) < 1024; // 所有元素之和 < 1024
}
```

### 6.13.1 递增序列约束

```systemverilog
constraint c_ascend {
    foreach (d[i])
        if (i > 0)
            d[i] > d[i-1];     // 比前一个元素大
}
```

### 6.13.2 唯一元素值（高效方法）

```systemverilog
// ✅ 推荐：randc 辅助类 — 比嵌套 foreach 高效得多（O(N) vs O(N²)）
class Randc8;
    randc bit [7:0] val;
endclass

class UniqueArray;
    bit [7:0] ua[64];
    function void pre_randomize();
        Randc8 rc8 = new();
        foreach (ua[i]) begin
            assert(rc8.randomize());
            ua[i] = rc8.val;
        end
    endfunction
endclass

// ❌ 不推荐：嵌套 foreach — 产生 O(N²) 级约束
constraint c { foreach(ua[i]) foreach(ua[j]) if(i!=j) ua[i]!=ua[j]; }
```

### 6.13.3 随机化句柄数组

```systemverilog
class RandArray;
    rand RandStuff array[];  // 句柄数组
    constraint c { array.size() inside {[1:MAX_SIZE]}; }

    function new();
        array = new[MAX_SIZE];  // 1. 提前分配最大容量的句柄
        foreach (array[i])
            array[i] = new();   // 2. 创建每个对象实例
    endfunction
endclass
// 随机化时：数组大小可以不变/减小，但不能增大
```

| 注意点 | 说明 |
|------|------|
| `foreach` 约束数量 | 数组越大，约束越多，求解越慢 |
| 嵌套 `foreach` | 产生 O(N²) 级约束，只适合极小的数组 |
| `sum()` | 注意元素位宽和符号扩展问题 |
| 句柄数组 | 必须在 `new()` 中提前分配并创建所有对象 |
| 动态数组上限 | 一定要设置 size 上限，防止求解器生成巨大数组 |

---

## 6.14：产生原子激励和场景

### 激励的四个层次

| 层次 | 含义 | 例子 |
|------|------|------|
| 原子激励 | 单个随机事务 | 一个 bus transaction、一个 packet |
| 历史相关发生器 | 当前事务依赖上一个事务 | 80% 概率继续上次的读写命令，地址递增 |
| 完整事务序列 | 一次随机化整个事务流 | DMA 传输、burst、网络包序列 |
| 组合序列 | 多个场景拼接 | 下载邮件 + 浏览网页 + 表单输入 |

### 6.14.1 历史相关的原子发生器

```systemverilog
class AtomicGen;
    rand bit [31:0] addr;
    bit [31:0] last_addr;

    function void post_randomize();
        last_addr = addr;  // 保存历史，下一次 randomize() 可参考
    endfunction
endclass
```

### 6.14.2 `randsequence` — BNF 风格序列

```systemverilog
initial begin
    for (int i = 0; i < 15; i++) begin
        randsequence (stream)
            stream: cfg_read := 1 |    // 权重 1 (10%)
                    io_read  := 2 |    // 权重 2 (20%)
                    mem_read := 5;     // 权重 5 (50%)

            cfg_read: { cfg_read_task; }        // 可单次调用...
                    | { cfg_read_task; } cfg_read;  // 也可递归调用
            io_read : { io_read_task; }
                    | { io_read_task; } io_read;
            mem_read: { mem_read_task; }
                    | { mem_read_task; } mem_read;
        endsequence
    end
end
```

| 优点 | 缺点 |
|------|------|
| 程序性代码，可逐步调试 | 产生的代码和约束类风格不同 |
| 可内嵌 $display 调试 | 难以通过扩展来修改序列 |
| 权重控制清晰 | 第 8 章会学到更好的方式（类继承） |

### 6.14.3–6.14.4 随机对象数组 + 组合序列

一次性随机化整个事务序列 → 可以预先约束总长度、总校验和：

```systemverilog
constraint c_seq {
    foreach (seq[i])
        if (i > 0)
            seq[i].addr >= seq[i-1].addr;  // 地址递增
}
```

| 方法 | 优点 | 缺点 |
|------|------|------|
| 逐个事务生成 | 简单，容易调试 | 无法提前知道整个序列信息 |
| 整个序列一起随机 | 可约束总长度、总校验和 | 约束更复杂，求解压力更大 |
| 组合序列 | 更接近真实 workload | 需要更好的场景建模能力 |

> 📌 单个随机事务只是起点，真实验证要构造"事务流"。DMA、缓存填充、网络访问不是孤立事务，而是有上下文、有历史关系的序列。

---

## 本章总结（6.1–6.14）

### 学习重点排序

| 优先级 | 必须掌握 |
|:---:|------|
| 🔴 高 | `rand`、`randc`、`constraint`、`randomize()`、`inside`、`dist` |
| 🔴 高 | `constraint_mode()`、`randomize() with {}`、`rand_mode()` |
| 🟡 中 | `solve before`、`pre_randomize()`、`post_randomize()` |
| 🟡 中 | `foreach` 约束、`sum()`、动态数组、句柄数组 |
| 🟢 进阶 | 历史相关事务、完整随机序列（`randsequence`）、组合场景 |

### 最重要的 10 条规则

| # | 规则 | 说明 |
|---|------|------|
| 1 | 随机化目标是**事务级**，不是孤立 bit | 一次 randomize() 生成一组有意义的激励 |
| 2 | 所有 `randomize()` 都要**检查返回值** | 失败返回 0，不检查后续会使用非法值 |
| 3 | 默认生成**合法**事务，错误注入时再关闭约束 | 用 `constraint_mode(0)` 关闭有效性约束 |
| 4 | `constraint` 是**声明式**代码，不是顺序执行 | 所有约束同时有效，并行求解 |
| 5 | `inside` 管**合法集合**，`dist` 管**概率权重** | 集合内各值等概率，dist 做不等权重 |
| 6 | `with {}` 只能**增加**约束，不能覆盖已有约束 | 需要覆盖时先 `constraint_mode(0)` |
| 7 | `rand_mode(0)` 固定变量，但**不关闭相关约束** | 固定值违反约束时，`randomize()` 仍会失败 |
| 8 | `pre_randomize()` 设参数，`post_randomize()` 生成派生字段 | 前后处理函数不能写耗时语句 |
| 9 | 数组约束要**限制 size 上限**，避免求解器压力过大 | 尤其是动态数组的 size 约束 |
| 10 | 从单个事务升级到**事务序列**，才接近真实验证 | 历史相关、完整序列、组合场景 |

### 最容易错的点

| 易错点 | 正确理解 |
|--------|---------|
| `lo < med < hi` 是合法约束 | 错，被解析为 `(lo < med) < hi`，应拆成两个 |
| `inside` 中重复值会增加概率 | 错，集合有互异性，重复不增加概率 |
| `randomize() with` 可以覆盖已有约束 | 错，只能增加限制，不能覆盖 |
| 约束是顺序执行行 | 错，是声明式并行求解、双向的 |
| `rand_mode(0)` 会清除旧值 | 错，只让后续 randomize() 不再改变它 |
| `solve before` 改变解集合 | 错，只改变概率分布，不改变合法解集合 |
| `byte` / `int` 在随机化里和 `bit` 一样 | 错，有符号类型可能产生负值 |
| 始终不检查 `randomize()` 返回值 | 错，失败时变量可能为非法值 |
