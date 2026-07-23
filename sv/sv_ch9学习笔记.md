# SystemVerilog 第 9 章学习笔记：功能覆盖率

> **核心结论**：功能覆盖率用来回答“验证计划里的功能点到底测到了没有”。代码覆盖率告诉你代码执行过多少，功能覆盖率告诉你设计意图覆盖了多少。第 9 章重点是 `covergroup`、`coverpoint`、`bins`、`cross`、覆盖触发和覆盖率数据分析。
> **记忆方式**：计划定目标，covergroup 收数据，bin 分状态，cross 看组合，option 调权重，报告找盲区。

---

## 9.0 功能覆盖率的背景

受约束随机测试 CRT 能在巨大状态空间中随机游走，但问题是：随机测试跑了很多次以后，怎么知道验证已经足够？

覆盖率就是用来度量验证进度的指标。

| 概念 | 作用 |
|------|------|
| 验证计划 | 列出要测试哪些设计特征 |
| 覆盖率 | 度量这些特征是否被测试到 |
| 覆盖率数据库 | 每次仿真结束后保存覆盖结果 |
| 覆盖率报告 | 找出还没覆盖到的盲区 |
| 覆盖率收敛 | 根据报告继续加种子、改约束或写定向测试 |

覆盖率驱动验证的一般流程：

1. 根据设计规范写验证计划。
2. 在测试平台中加入覆盖点。
3. 用不同随机种子多次运行测试。
4. 合并覆盖率数据库。
5. 查看哪些 bin 没命中。
6. 优先增加随机种子。
7. 如果增长变慢，再修改约束。
8. 只有必要时才写定向测试。

> **注意**：100% 覆盖率不等于没有 bug。它只表示你定义的覆盖目标都被触发过。

---

## 9.1 覆盖率的类型

### 9.1.1 代码覆盖率

代码覆盖率由仿真工具自动插桩统计，不需要额外写 HDL 代码。

| 类型 | 衡量内容 |
|------|----------|
| 行覆盖率 | 哪些代码行执行过 |
| 路径覆盖率 | if/case 等路径是否走过 |
| 翻转覆盖率 | 单比特信号是否出现过 0/1 |
| FSM 覆盖率 | 状态和状态跳转是否出现过 |

代码覆盖率适合发现“实现代码没有被测试到”的问题，但不能证明设计功能完整。

```systemverilog
module dff(
    output logic q, q_l,
    input  logic clk, d, reset_l
);
    always @(posedge clk or negedge reset_l) begin
        q   <= d;                         // BUG：没有处理 reset_l
        q_l <= !d;                        // 每行都可能被执行到
    end
endmodule
```

这个 D 触发器漏掉了复位逻辑，但行覆盖率可能仍然是 100%。
`reset_l` 末尾的 `_l` 通常表示低有效复位，也就是 `reset_l = 0` 时复位有效。
敏感列表中写了 `negedge reset_l`，说明设计意图是：当 `reset_l` 从 1 变成 0 时，触发复位逻辑。

但上面的代码虽然会被 `negedge reset_l` 触发，always 块内部却没有判断 `reset_l`：
```systemverilog
q   <= d;
q_l <= !d;
```
所以即使复位信号到来，电路仍然只是采样 `d` 和 `!d`，没有把输出恢复到固定复位值。

正确写法应该显式写出复位分支。
```systemverilog
always @(posedge clk or negedge reset_l) begin
    if (!reset_l) begin
        q   <= 1'b0;
        q_l <= 1'b1;
    end
    else begin
        q   <= d;
        q_l <= !d;
    end
end
```
也就是说，代码覆盖率可能只看到 `q <= d` 和 `q_l <= !d` 这些语句被执行过，却看不出“复位后进入已知状态”这个设计意图根本没有实现。

---

### 9.1.2 功能覆盖率

功能覆盖率针对设计规范和验证计划，而不是直接针对 RTL 代码。

例如 D 触发器的验证计划不只要求“数据能被存储”，还应该要求“复位后进入已知状态”。

| 覆盖类型 | 面向对象 | 能发现的问题 |
|----------|----------|--------------|
| 代码覆盖率 | RTL 实现 | 哪些代码没执行 |
| 功能覆盖率 | 设计意图/验证计划 | 哪些功能没测试 |

功能覆盖率也常叫规范覆盖率。

---

### 9.1.3 漏洞率 Bug Rate

漏洞率是覆盖充分性的间接指标。

| 现象 | 可能含义 |
|------|----------|
| bug 还在持续出现 | 仍有很多状态空间未充分验证 |
| bug 数下降但覆盖率低 | 测试策略可能已经到头 |
| 覆盖率高但 bug 仍多 | 覆盖模型可能没描述真正危险区域 |
| 流片后还发现 bug | 验证计划或覆盖模型存在盲区 |

漏洞率要结合代码覆盖率和功能覆盖率一起看，不能单独作为完成标准。

---

### 9.1.4 断言覆盖率

断言通常用于检查设计行为，也可以用于统计某些序列是否发生过。

| 语句 | 作用 |
|------|------|
| `assert property` | 检查协议/时序关系，失败时报错 |
| `cover property` | 记录某个信号序列是否出现 |
| `covergroup` | 对数值、事务或表达式进行采样 |

`cover property` 适合观测信号序列；`covergroup` 适合统计事务字段和抽象状态。

---

## 9.2 功能覆盖策略

### 9.2.1 收集信息而非数据

功能覆盖率应该测量“有意义的设计状态”，而不是盲目测量所有数值。

以 FIFO 为例，不应该试图覆盖所有读写指针组合，而应该关注边界状态。

| 不推荐 | 推荐 |
|--------|------|
| 覆盖所有地址索引组合 | 覆盖 empty/full |
| 覆盖所有计数值 | 覆盖 0、1、半满、满、满到空 |
| 覆盖 32 位地址每个值 | 覆盖 memory 区、IO 区、边界地址 |

例如 FIFO 覆盖模型可以关注 `count` 的关键状态，而不是覆盖所有指针组合。
```systemverilog
covergroup FifoCov;
    coverpoint count {
        bins empty      = {0};
        bins one_entry  = {1};
        bins half       = {[DEPTH/2-1 : DEPTH/2+1]};
        bins full       = {DEPTH};
    }

    coverpoint empty;
    coverpoint full;
endgroup
```
这个覆盖模型能回答“FIFO 是否测到空、满、半满等关键状态”，比统计所有 `rd_ptr`/`wr_ptr` 组合更有意义。

更完整一点看，FIFO 的验证计划通常关心的是“状态”和“状态转换”。

| 覆盖目标 | 为什么重要 |
|----------|------------|
| `empty` | 检查空 FIFO 行为，例如空读是否被阻止 |
| `full` | 检查满 FIFO 行为，例如满写是否被阻止 |
| `almost_full` | 检查即将满时是否还能正确 backpressure |
| `write && read` | 检查同周期读写是否正确更新计数 |
| `empty => full` | 检查 FIFO 能从空填到满 |
| `full => empty` | 检查 FIFO 能从满读到空 |

不推荐的覆盖方式是把指针当成纯数据穷举。
```systemverilog
covergroup BadFifoCov;
    coverpoint rd_ptr;                   // 可能产生大量 bin
    coverpoint wr_ptr;                   // 可能产生大量 bin
    cross rd_ptr, wr_ptr;                // 组合更多，报告很难分析
endgroup
```
这个模型即使覆盖率低，也很难告诉你下一步该写什么测试；即使覆盖率高，也不代表 empty/full/overflow 等关键行为被测充分。

更推荐把覆盖点对齐到验证计划。
```systemverilog
covergroup GoodFifoCov;
    coverpoint count {
        bins empty        = {0};
        bins almost_empty = {1};
        bins middle       = {[2 : DEPTH-2]};
        bins almost_full  = {DEPTH-1};
        bins full         = {DEPTH};
    }

    coverpoint wr_en;
    coverpoint rd_en;

    // 关注危险组合：空读、满写、同周期读写
    cross count, wr_en, rd_en {
        bins empty_read = binsof(count.empty) && binsof(rd_en) intersect {1};
        bins full_write = binsof(count.full)  && binsof(wr_en) intersect {1};
        bins rw_same    = binsof(wr_en) intersect {1} && binsof(rd_en) intersect {1};
    }
endgroup
```
如果报告显示 `full_write` 没命中，下一步就很明确：需要写一个测试先把 FIFO 填满，再尝试写入。

> **经验**：覆盖点数量太多，报告会变得不可读；覆盖模型应反映验证计划中的信息。

---

### 9.2.2 只测量会使用的内容

功能覆盖率有仿真开销和数据库开销，所以不要测量自己不会分析的东西。

| 应该测量 | 不应该测量 |
|----------|------------|
| 验证计划明确列出的功能点 | 没人会看的大范围数值 |
| 边界条件、错误模式、协议组合 | 每个随机变量的所有可能取值 |
| 会指导下一步测试修改的点 | 只为了让报告看起来很多的点 |

例如 32 位地址不应该直接覆盖每个可能值，而应该按设计规格划分区域。
```systemverilog
covergroup AddrCov;
    coverpoint addr {
        bins boot_rom = {[32'h0000_0000 : 32'h0000_FFFF]};
        bins sram     = {[32'h1000_0000 : 32'h1000_FFFF]};
        bins io       = {[32'h2000_0000 : 32'h2000_FFFF]};
        bins boundary = {32'h1000_0000, 32'h1000_FFFF};
    }
endgroup
```
这样报告能直接告诉你 boot ROM、SRAM、IO 区和边界地址有没有被访问过，后续可以据此修改约束或补定向测试。

地址覆盖的重点不是“地址值越多越好”，而是“设计规定的区域有没有被测到”。
例如一个简单 SoC 地址空间如下：

| 地址范围 | 含义 | 应该覆盖什么 |
|----------|------|--------------|
| `0x0000_0000` - `0x0000_FFFF` | boot ROM | 普通访问、首地址、末地址 |
| `0x1000_0000` - `0x1000_FFFF` | SRAM | 读写访问、对齐/非对齐边界 |
| `0x2000_0000` - `0x2000_0FFF` | 外设寄存器 | 合法寄存器地址 |
| 其他地址 | illegal region | 错误响应或访问拒绝 |

可以把合法区域和非法区域都放进覆盖模型。
```systemverilog
covergroup BusAddrCov;
    coverpoint addr {
        bins boot_rom_start = {32'h0000_0000};
        bins boot_rom_body  = {[32'h0000_0004 : 32'h0000_FFFB]};
        bins boot_rom_end   = {32'h0000_FFFF};

        bins sram_start     = {32'h1000_0000};
        bins sram_body      = {[32'h1000_0004 : 32'h1000_FFFB]};
        bins sram_end       = {32'h1000_FFFF};

        bins io_regs        = {[32'h2000_0000 : 32'h2000_0FFF]};
        bins illegal        = default;
    }

    coverpoint write;
    cross addr, write;
endgroup
```
如果覆盖报告显示 `illegal` 没有命中，说明没有测试非法地址访问；如果 `sram_end` 没有命中，说明边界地址还没测到。
这类覆盖点可以直接指导下一步测试，而不是只让报告变厚。

如果覆盖率报告不会被用来改进测试，就不应该收集这些覆盖率。

---

### 9.2.3 测量的完备性

项目收尾时要同时看代码覆盖率、功能覆盖率和漏洞率。

| 情况 | 解释 | 下一步 |
|------|------|--------|
| 功能覆盖率高，代码覆盖率低 | 验证计划可能漏掉了功能 | 回到设计规范补计划 |
| 代码覆盖率高，功能覆盖率低 | 代码执行过，但没到达关键状态 | 修改约束或测试 |
| 两者都高，但 bug 仍多 | 覆盖模型可能不够深入 | 增加危险组合覆盖 |
| 覆盖率不再增长 | 当前随机策略到头 | 换种子、改约束、写定向测试 |

举例说明：

| 情况 | 例子 |
|------|------|
| 功能覆盖率高，代码覆盖率低 | 验证计划覆盖了 FIFO 普通读写和 empty/full，但 RTL 里还有 flush、overflow 分支从未执行 |
| 代码覆盖率高，功能覆盖率低 | 读写代码都执行过，但功能覆盖显示 full、empty 到 full、full 到 empty 都没命中 |
| 两者都高，但 bug 仍多 | 覆盖了读、写、empty、full，但没覆盖空读、满写、读写同时发生、reset 与传输同时发生 |
| 覆盖率不再增长 | 随机跑很多种子仍停在 85%，说明当前约束很难打到剩余场景，需要改约束或写定向测试 |

更具体的决策流程可以这样看：

1. 先看功能覆盖率。
   如果某些验证计划中的 bin 没命中，优先补这些场景。
2. 再看代码覆盖率。
   如果 RTL 中有分支从未执行，检查验证计划是否漏了对应功能。
3. 再看 bug 趋势。
   如果覆盖率很高但 bug 还很多，说明覆盖点可能太浅，只覆盖了表面状态。
4. 最后看覆盖率增长曲线。
   如果增加随机种子也不增长，就不要继续盲目跑随机，应改约束或写定向测试。

例如 FIFO 覆盖报告如下：

| 覆盖项 | 命中情况 | 判断 | 下一步 |
|--------|----------|------|--------|
| `empty` | hit | 空状态测到了 | 保持 |
| `full` | miss | 没测到满状态 | 增加连续写直到满的 sequence |
| `empty_read` | miss | 没测空读 | 写定向空读测试 |
| `rw_same` | hit | 同周期读写测到了 | 保持 |
| `overflow branch` 代码覆盖 | miss | RTL overflow 分支没执行 | 补满写测试，并检查验证计划是否包含 overflow |

这个例子说明，覆盖率报告的价值不在于数字本身，而在于把“下一步该测什么”说清楚。

---

## 9.3 功能覆盖率的简单例子

### 9.3.1 covergroup 和 coverpoint

`covergroup` 是一组在同一时刻采样的覆盖点；`coverpoint` 是具体要采样的变量或表达式。

```systemverilog
program automatic test;
    class Transaction;
        rand bit [31:0] data;             // 事务数据
        rand bit [2:0]  port;             // 3 bit，共 8 种端口值
    endclass

    Transaction tr;

    covergroup CovPort;
        coverpoint tr.port;               // 对 tr.port 的取值建 bin
    endgroup

    initial begin
        CovPort ck = new();               // 覆盖组必须实例化

        repeat (32) begin
            tr = new();
            assert(tr.randomize());        // 产生随机 port
            ck.sample();                   // 显式采样当前 tr.port
        end
    end
endprogram
```

| 语法 | 含义 |
|------|------|
| `covergroup CovPort` | 定义覆盖组 |
| `coverpoint tr.port` | 定义覆盖点 |
| `CovPort ck = new()` | 实例化覆盖组 |
| `ck.sample()` | 手动触发采样 |

如果 32 次随机只产生了 1 到 7，没有产生 0，覆盖率就是 7/8 = 87.5%。

---

### 9.3.2 覆盖率报告怎么读

覆盖率报告通常会列出每个 bin 的命中次数。

| 报告项 | 含义 |
|--------|------|
| Coverage | 当前覆盖百分比 |
| Goal | 目标覆盖率，默认 100 |
| Bin | 仓名 |
| # hits | 该仓被采样命中的次数 |
| at_least | 认为该仓被覆盖所需的最少命中次数 |

当某个 bin 命中为 0 时，说明相关场景没有被测到。

---

## 9.4 覆盖组详解

### 9.4.1 覆盖组放在哪里

覆盖组可以定义在 module、program、interface 或 class 中。

| 放置位置 | 适合场景 |
|----------|----------|
| program/module | 顶层或简单 testbench |
| interface | 采样接口信号 |
| class | 事务处理器、monitor、coverage collector |
| callback 类 | 在 Driver/Monitor 特定时刻采样 |

不要随便把覆盖组放在每个 transaction 对象里，因为每个事务对象都带一个覆盖组会增加开销。

---

### 9.4.2 类里的 covergroup

```systemverilog
class Transactor;
    Transaction tr;                       // 当前处理的事务
    mailbox mbx_in;                       // 上游传入事务的 mailbox

    covergroup CovPort;
        coverpoint tr.port;               // 采样类成员 tr 的字段
    endgroup

    function new(mailbox mbx_in);
        CovPort = new();                  // 覆盖组必须在构造函数中实例化
        this.mbx_in = mbx_in;             // 保存 mailbox 句柄
    endfunction

    task main();
        forever begin
            mbx_in.get(tr);               // 获取下一个事务
            ifc.cb.port <= tr.port;        // 发送到 DUT
            ifc.cb.data <= tr.data;
            CovPort.sample();             // 事务送出后采样
        end
    endtask
endclass
```

| 规则 | 说明 |
|------|------|
| covergroup 必须 `new()` | 忘记实例化时报告里不会出现该组 |
| 采样时刻要正确 | 应等数据真正到达 DUT 或监视点 |
| 命名要清楚 | 报告中靠名字定位覆盖项 |

---

## 9.5 覆盖组的触发

覆盖组定义了“采什么”，触发方式决定“什么时候采”。

| 触发方式 | 写法 | 适合场景 |
|----------|------|----------|
| 显式调用 | `cg.sample()` | 事务处理代码中手动控制采样点 |
| 回调采样 | 在 `post_tx()` 或 `post_rx()` 中调用 `sample()` | Driver/Monitor 已经有 callback 钩子 |
| 事件触发 | `covergroup cg @(event_name);` | 已经有事件能表示准确采样时刻 |
| SVA 触发 | SVA 检测协议事件后 `-> event` | 协议事件复杂，适合用断言序列识别 |

采样时刻很重要。过早采样可能记录到还没真正送到 DUT 的事务；过晚采样可能记录到已经被下一笔事务覆盖的值。

---

### 9.5.1 显式调用 sample()

`sample()` 适合在程序性代码中精确控制采样时刻。

```systemverilog
task send_one(Transaction tr);
    drive_to_dut(tr);                     // 先把事务送到 DUT
    CovPort.sample();                     // 再采样已发送事务
endtask
```

`sample()` 的含义是：**现在立刻采样 covergroup 中所有 coverpoint 的当前值**。

例如：
```systemverilog
covergroup CovPort;
    coverpoint tr.port;
endgroup

task main();
    forever begin
        mbx.get(tr);                      // 取到新事务
        drive_to_dut(tr);                 // 事务真正送出
        CovPort.sample();                 // 采样当前 tr.port
    end
endtask
```

如果 `tr.port` 依次为 3、5、3、0，那么每次 `sample()` 都会记录一次当前值：

| 调用次数 | 当前 `tr.port` | 覆盖结果 |
|----------|----------------|----------|
| 第 1 次 | 3 | port=3 命中 |
| 第 2 次 | 5 | port=5 命中 |
| 第 3 次 | 3 | port=3 命中次数增加 |
| 第 4 次 | 0 | port=0 命中 |

适用场景：

| 场景 | 原因 |
|------|------|
| 事务完成时采样 | 采样点不一定对应简单时钟边沿 |
| 一个覆盖组多个实例 | 每个实例可独立触发 |
| 回调中采样 | 由 Driver/Monitor 的事件决定 |

常见错误：

| 错误 | 后果 |
|------|------|
| 在 `mbx.get(tr)` 之前采样 | 采到旧事务或空句柄 |
| 在事务还没真正发送前采样 | 统计到 DUT 实际没收到的事务 |
| 忘记 `CovPort = new()` | 覆盖组没有实例，无法采样 |

---

### 9.5.2 使用回调采样

第 8 章的回调机制很适合连接功能覆盖率。

回调采样的思想是：Driver 或 Monitor 本来就知道事务什么时候真正发送或接收完成，于是在这个固定时机调用覆盖率回调。

流程：

```text
test 注册 coverage callback
        |
Driver 发送事务
        |
Driver 调用 post_tx(tr)
        |
coverage callback 采样 tr.port
```

如果 covergroup 直接采样 `post_tx(Transaction tr)` 的参数，容易混淆作用域。更稳妥的写法是让 covergroup 带采样参数。

```systemverilog
class Driver_cbs_coverage extends Driver_cbs;
    covergroup CovPort with function sample(Transaction tr);
        coverpoint tr.port;               // 采样 post_tx 传进来的事务字段
    endgroup

    function new();
        CovPort = new();                  // 创建覆盖组实例
    endfunction

    virtual task post_tx(Transaction tr);
        CovPort.sample(tr);               // 事务发送后采样当前事务
    endtask
endclass
```

在 test 中注册覆盖率回调：

```systemverilog
initial begin
    Driver_cbs_coverage dcc;

    env = new();
    env.gen_cfg();
    env.build();

    dcc = new();                          // 创建覆盖率回调对象
    env.drv.cbs.push_back(dcc);           // 放入 Driver 回调队列

    env.run();
    env.wrap_up();
end
```

注册后，Driver 里通常会有类似逻辑：

```systemverilog
task run();
    Transaction tr;

    forever begin
        gen2drv.get(tr);
        transmit(tr);                     // 真正发送到 DUT

        foreach (cbs[i])
            cbs[i].post_tx(tr);           // 发送后调用覆盖率回调
    end
endtask
```

为什么常放在 `post_tx()` 而不是 Generator 中采样？

| 采样位置 | 问题或含义 |
|----------|------------|
| Generator 中采样 | 只能说明事务被生成，不代表真的送到 DUT |
| Driver `pre_tx()` 中采样 | 如果后续被 drop，可能误认为已测试 |
| Driver `post_tx()` 中采样 | 表示事务已经真实发送 |
| Monitor 中采样 | 表示 DUT 接口上真实观察到了该事务 |

> **经验**：覆盖率、记分板、功能统计都属于被动观察逻辑，通常比 mailbox 事务处理器更适合用 callback 接入。

---

### 9.5.3 使用事件触发

covergroup 可以直接绑定事件，当事件触发时自动采样。

```systemverilog
event trans_ready;                        // 表示事务数据已经准备好

covergroup CovPort @(trans_ready);
    coverpoint ifc.cb.port;               // 事件触发时采样接口 port
endgroup
```

这表示：每次执行 `-> trans_ready` 时，`CovPort` 自动采样一次。

完整例子：

```systemverilog
event trans_ready;

covergroup CovPort @(trans_ready);
    coverpoint ifc.cb.port {
        bins p0 = {0};
        bins p1 = {1};
        bins others = {[2:7]};
    }
endgroup

initial begin
    CovPort cg = new();

    ifc.cb.port <= 3;
    -> trans_ready;                       // 采样 port=3

    ifc.cb.port <= 1;
    -> trans_ready;                       // 采样 port=1
end
```

执行流程：

```text
更新 ifc.cb.port
        |
触发 -> trans_ready
        |
covergroup 自动采样
        |
覆盖率数据库记录当前 port
```

如果已有事件准确表示采样时刻，事件触发比手动 `sample()` 更简洁。

常见事件名可以是：

| 事件 | 含义 |
|------|------|
| `trans_ready` | 事务字段已经准备好 |
| `packet_done` | 一个 packet 已经接收完成 |
| `write_done` | 一次写事务完成 |
| `read_rsp_seen` | 读响应已经被观察到 |

> **注意**：事件必须在数据稳定之后触发。否则 covergroup 可能采到旧值或中间值。

---

### 9.5.4 使用 SVA 触发

可以让断言序列触发事件，再由事件唤醒覆盖组。

```systemverilog
module mem(simple_bus sb);
    bit [7:0] data, addr;
    event write_event;                    // SVA 检测到写操作后触发

    cover property
        (@(posedge sb.clock) sb.write_ena == 1)
        -> write_event;                   // 写使能出现时触发事件
endmodule
```

上面的含义是：每个 `sb.clock` 上升沿检查 `sb.write_ena`。如果 `sb.write_ena == 1`，说明出现写操作，于是触发 `write_event`。

更接近真实协议的写操作通常不是只看 `write_ena`，而是看握手条件，例如：

```systemverilog
cover property
    (@(posedge sb.clock) sb.valid && sb.ready && sb.write)
    -> write_event;                       // 真正写握手发生时触发
```

这样可以避免在 `valid=1` 但 `ready=0` 时误采样。

```systemverilog
program automatic test(simple_bus sb);
    covergroup Write_cg @($root.top.m1.write_event);
        coverpoint $root.top.m1.data;     // 由 SVA 事件触发采样
        coverpoint $root.top.m1.addr;
    endgroup

    Write_cg wcg;

    initial begin
        wcg = new();                      // 覆盖组仍然需要实例化
        sb.write_ena <= 1;
        #10000 $finish;
    end
endprogram
```

这种方式适合已有 SVA 能准确识别“事务结束”或“协议事件”的场景。

执行流程：

```text
posedge sb.clock
        |
SVA 检查协议条件
        |
条件成立，触发 write_event
        |
Write_cg 自动采样 data/addr
        |
覆盖率记录一次真实写操作
```

适合用 SVA 触发的场景：

| 场景 | SVA 条件示例 | 采样内容 |
|------|--------------|----------|
| 写握手 | `valid && ready && write` | 写地址、写数据 |
| 读响应 | `rsp_valid && rsp_ready` | 读数据、响应码 |
| burst 结束 | `valid && ready && last` | burst 长度、结束状态 |
| 错误响应 | `rsp_valid && rsp_err` | 错误类型、地址 |

三种常见触发方式对比：

| 方式 | 优点 | 风险 |
|------|------|------|
| 手动 `sample()` | 简单直接，容易控制 | 采样点散落在代码中 |
| 事件触发 | 采样点集中，代码更干净 | 事件必须准确表示采样时刻 |
| SVA 触发 | 协议事件识别准确 | 写法更复杂，层次路径要小心 |

> **经验**：如果采样时刻能用一句程序代码明确表示，用 `sample()`；如果已有事件表示事务完成，用事件触发；如果采样时刻是复杂协议条件，用 SVA 触发。

---

## 9.6 数据采样和 bins

### 9.6.1 bin 的基本概念

SystemVerilog 会为 coverpoint 创建 bin，bin 是覆盖率统计的基本单位。

| 概念 | 含义 |
|------|------|
| 域 domain | 覆盖点可能取到的全部值 |
| bin | 一个或多个值的集合 |
| hit | 某个 bin 被采样命中一次 |
| 覆盖率 | 命中的 bin 数 / 总 bin 数 |

例如 3 bit 的 `port` 取值范围是 0 到 7，默认会创建 8 个自动 bin。
```systemverilog
bit [2:0] port;

covergroup CovPort;
    coverpoint port;                     // 默认按 port 的取值建 bin
endgroup
```

因为 `port` 是 3 bit，它的 domain 是 `{0,1,2,3,4,5,6,7}`。工具可能自动创建如下 bin：

| 自动 bin | 覆盖的值 |
|----------|----------|
| `port[0]` | `port == 0` |
| `port[1]` | `port == 1` |
| `port[2]` | `port == 2` |
| `...` | `...` |
| `port[7]` | `port == 7` |

如果采样过程如下：
```systemverilog
CovPort ck = new();

port = 3;
ck.sample();                             // 命中 port==3 的 bin

port = 5;
ck.sample();                             // 命中 port==5 的 bin

port = 3;
ck.sample();                             // port==3 的 bin 再命中一次
```

那么覆盖情况可以理解为：

| bin | 命中次数 |
|-----|----------|
| `port == 3` | 2 |
| `port == 5` | 1 |
| 其他 bin | 0 |

总共有 8 个 bin，命中了 2 个不同的 bin，所以覆盖率是 `2/8 = 25%`。命中次数会记录 hit 数，但覆盖率通常看“有多少 bin 至少命中过一次”。

---

### 9.6.2 限制自动 bin 数量

`option.auto_bin_max` 用来限制自动创建 bin 的最大个数，默认通常是 64。

```systemverilog
covergroup CovPort;
    coverpoint tr.port {
        option.auto_bin_max = 2;          // 3 bit 的 8 个值被分成 2 个 bin
    }
endgroup
```

也可以作用于整个 covergroup：

```systemverilog
covergroup CovPort;
    option.auto_bin_max = 2;              // 影响组内所有未手动分 bin 的覆盖点
    coverpoint tr.port;
    coverpoint tr.data;
endgroup
```

| 优点 | 风险 |
|------|------|
| 报告更短 | 可能掩盖具体没覆盖的值 |
| 大范围变量可控 | bin 太粗时覆盖率虚高 |

---

### 9.6.3 对表达式采样

coverpoint 可以采样表达式，但要特别注意表达式位宽。

```systemverilog
class Transaction;
    rand bit [2:0] hdr_len;               // 0 到 7
    rand bit [3:0] payload_len;           // 0 到 15
    rand bit [3:0] kind;
endclass

Transaction tr;

covergroup CovLen;
    len16: coverpoint (tr.hdr_len + tr.payload_len);
    len32: coverpoint (tr.hdr_len + tr.payload_len + 5'b0);
endgroup
```

`len16` 可能因为表达式位宽不够，只得到 16 个自动 bin；加 `5'b0` 是为了把表达式扩展到更大位宽。

> **注意**：表达式采样后一定要看报告，确认 bin 的范围和验证计划一致。

---

### 9.6.4 用户自定义 bin

自定义 bin 可以给有意义的范围命名，报告更容易读。

```systemverilog
covergroup CovLen;
    len: coverpoint (tr.hdr_len + tr.payload_len + 5'b0) {
        bins len[] = {[0:22]};            // 为 0 到 22 每个值建独立 bin
    }
endgroup
```

如果误写成 `[0:23]`，报告会显示 `len_17` 一直没有命中，从而暴露“最大长度其实只有 22”的建模问题。

---

### 9.6.5 命名覆盖点的 bin

```systemverilog
covergroup CovKind;
    coverpoint tr.kind {
        bins zero = {0};                  // 1 个 bin：kind == 0
        bins lo   = {[1:3], 5};           // 1 个 bin：1、2、3、5
        bins hi[] = {[8:$]};              // 多个独立 bin：8 到最大值
        bins misc = default;              // 捕捉前面没有列出的值
    }                                     // coverpoint 的大括号后不加分号
endgroup
```

| 写法 | 含义 |
|------|------|
| `bins zero = {0}` | 一个命名 bin |
| `bins lo = {[1:3], 5}` | 多个值合并成一个 bin |
| `bins hi[] = {[8:$]}` | 范围内每个值建独立 bin |
| `bins misc = default` | 捕获剩余值 |

> **经验**：自定义 bin 后，覆盖率只按你定义的 bin 计算，未列出的值会被忽略；通常建议加 `default` 防漏。

---

### 9.6.6 使用 `$` 表示范围边界

`$` 在 bin 范围中表示该类型的最小或最大边界。

```systemverilog
int i;

covergroup range_cover;
    coverpoint i {
        bins neg  = {[$:-1]};             // 所有负值
        bins zero = {0};                  // 零
        bins pos  = {[1:$]};              // 所有正值
    }
endgroup
```

---

### 9.6.7 条件覆盖 iff / start / stop

`iff` 用来在某些条件下关闭覆盖采样，最常见的是复位期间不采样。

```systemverilog
covergroup CoverPort;
    coverpoint port iff (!bus_if.reset);  // reset 为 1 时不收集覆盖率
endgroup
```

也可以使用 `start()` 和 `stop()` 控制某个覆盖组实例。

```systemverilog
initial begin
    CovPort ck = new();                   // 实例化覆盖组

    #1ns ck.stop();                       // 复位期间暂停覆盖率
    bus_if.reset = 1;

    #100ns bus_if.reset = 0;              // 复位结束
    ck.start();                           // 恢复覆盖率收集
end
```

---

### 9.6.8 枚举类型的 bin

枚举类型会自动为每个枚举值创建 bin。

```systemverilog
typedef enum {INIT, DECODE, IDLE} fsmstate_e;

fsmstate_e pstate, nstate;                // 当前状态和下一状态

covergroup cg_fsm;
    coverpoint pstate;                    // 自动生成 INIT/DECODE/IDLE 的 bin
endgroup
```

`auto_bin_max` 对枚举类型通常不起作用，因为枚举的值域已经由枚举值确定。

---

### 9.6.9 翻转覆盖率 Transition Coverage

覆盖点不仅可以统计值是否出现，也可以统计值的转换。

```systemverilog
covergroup CoverPort;
    coverpoint port {
        bins t1 = (0 => 1), (0 => 2), (0 => 3); // 统计 0 到 1/2/3 的跳转
    }
endgroup
```

更多写法：

| 写法 | 含义 |
|------|------|
| `(1,2 => 3,4)` | 1/2 到 3/4 的所有组合转换 |
| `(0 => 1 => 2)` | 连续三次采样必须依次为 0、1、2 |
| `(0 => 1[*3] => 2)` | 1 连续重复 3 次 |
| `(0 => 1[*3:5] => 2)` | 1 连续重复 3 到 5 次 |

> **注意**：transition 覆盖要求转换过程中的每个状态都被采样到。

---

### 9.6.10 wildcard bin

`wildcard` 把 X、Z、? 当作通配符。

```systemverilog
bit [2:0] port;

covergroup CoverPort;
    coverpoint port {
        wildcard bins even = {3'b??0};    // 所有最低位为 0 的值
        wildcard bins odd  = {3'b??1};    // 所有最低位为 1 的值
    }
endgroup
```

---

### 9.6.11 ignore_bins 和 illegal_bins

`ignore_bins` 表示这些值不参与覆盖率计算；`illegal_bins` 表示这些值一旦出现就是错误。

```systemverilog
bit [2:0] low_ports_0_5;                  // 只允许 0 到 5

covergroup CoverPort;
    coverpoint low_ports_0_5 {
        ignore_bins hi = {[6:7]};         // 6、7 不计入覆盖率
    }
endgroup
```

```systemverilog
covergroup CoverPort;
    coverpoint low_ports_0_5 {
        illegal_bins hi = {[6:7]};        // 6、7 出现时报错
    }
endgroup
```

| 写法 | 作用 |
|------|------|
| `ignore_bins` | 合法但不关心，不参与覆盖率 |
| `illegal_bins` | 不应该出现，出现要报错 |

---

### 9.6.12 状态机覆盖率

可以手工用 covergroup 统计 FSM 状态和状态转移，但更推荐使用代码覆盖率工具自动提取 FSM 覆盖率。

| 方法 | 优点 | 风险 |
|------|------|------|
| 手工 covergroup | 可以只关注关键状态 | 设计改动后容易漏改 |
| 工具 FSM 覆盖 | 自动提取状态和转移 | 只反映代码实现，不代表验证计划完整 |

---

## 9.7 交叉覆盖率

### 9.7.1 基本 cross

交叉覆盖率用于同时统计多个覆盖点的组合。

```systemverilog
class Transaction;
    rand bit [3:0] kind;                  // 16 种事务类型
    rand bit [2:0] port;                  // 8 种端口
endclass

Transaction tr;

covergroup CovPort;
    kind: coverpoint tr.kind;             // 给 coverpoint 起标签
    port: coverpoint tr.port;
    cross kind, port;                     // 创建 kind x port 的组合覆盖
endgroup
```

如果 `kind` 有 16 个 bin，`port` 有 8 个 bin，cross 会产生 16 x 8 = 128 个交叉 bin。

> **注意**：cross 很容易产生大量 bin，定义前要确认这些组合真的有验证价值。

---

### 9.7.2 对交叉 bin 标号

交叉覆盖会使用覆盖点中已有的 bin 名称，提高报告可读性。

```systemverilog
covergroup CovPortKind;
    port: coverpoint tr.port {
        bins port[] = {[0:$]};            // 每个端口单独一个 bin
    }

    kind: coverpoint tr.kind {
        bins zero = {0};
        bins lo   = {[1:3]};              // 多个 kind 合并到一个 bin
        bins hi[] = {[8:$]};
        bins misc = default;
    }

    cross kind, port;                     // cross 报告会带上 zero/lo/hi 等名字
endgroup
```

合并 bin 后，交叉 bin 数也会减少，但覆盖率含义会变粗。

---

### 9.7.3 排除部分交叉 bin

交叉覆盖中可以用 `bins of` 和 `intersect` 排除不关心的组合。

```systemverilog
covergroup CovPort;
    port: coverpoint tr.port {
        bins port[] = {[0:$]};
    }

    kind: coverpoint tr.kind {
        bins zero = {0};
        bins lo   = {[1:3]};
        bins hi[] = {[8:$]};
        bins misc = default;
    }

    cross kind, port {
        ignore_bins hi =
            binsof(port) intersect {7};   // 忽略所有 port==7 的组合

        ignore_bins md =
            binsof(port) intersect {0} &&
            binsof(kind) intersect {[9:11]}; // 忽略 port==0 且 kind 为 9..11

        ignore_bins lo =
            binsof(kind.lo);              // 使用已命名的 kind.lo 仓
    }
endgroup
```

| 语法 | 含义 |
|------|------|
| `binsof(cp)` | 选择某个覆盖点的 bin |
| `intersect {range}` | 只取与指定范围相交的值 |
| `&&` | 组合多个条件 |
| `binsof(kind.lo)` | 引用已命名 bin |

---

### 9.7.4 权重 option.weight

一个 covergroup 的总体覆盖率会综合 coverpoint 和 cross。可以用权重控制它们对总体的贡献。

```systemverilog
covergroup CovPort;
    kind: coverpoint tr.kind {
        bins zero = {0};
        bins lo   = {[1:3]};
        option.weight = 5;                // kind 在总覆盖率中权重为 5
    }

    port: coverpoint tr.port {
        bins port[] = {[0:$]};
        option.weight = 0;                // 只用于 cross，本身不计入总分
    }

    cross kind, port {
        option.weight = 10;               // 更重视组合覆盖
    }
endgroup
```

---

### 9.7.5 交叉覆盖的替代写法

如果 cross 语法太复杂，可以把多个变量拼接成一个 coverpoint。

```systemverilog
covergroup CrossManual;
    ab: coverpoint {tr.a, tr.b} {
        bins a0b0 = {2'b00};              // a=0, b=0
        bins a1b0 = {2'b10};              // a=1, b=0
        wildcard bins b1 = {2'b?1};       // b=1，不关心 a
    }
endgroup
```

| 方法 | 适合场景 |
|------|----------|
| 普通 `cross` | 组合比较自然，bin 数可控 |
| `binsof/intersect` | 需要精确忽略部分组合 |
| 拼接 coverpoint | 组合规则少，想写得更直观 |

---

## 9.8 通用的覆盖组

### 9.8.1 通过数值传递参数

covergroup 可以带参数，让同一份定义复用到不同范围。

```systemverilog
bit [2:0] port;                           // 0 到 7

covergroup CoverPort(input int mid);
    coverpoint port {
        bins lo = {[0:mid-1]};            // 低区间
        bins hi = {[mid:$]};              // 高区间
    }
endgroup

CoverPort cp;

initial begin
    cp = new(5);                          // lo=0..4, hi=5..7
end
```

这里 `mid` 只是在构造覆盖组时传入的值。

---

### 9.8.2 通过引用传递参数

如果希望 covergroup 在仿真过程中持续采样某个变量，要用 `ref`。

```systemverilog
bit [2:0] port_a, port_b;

covergroup CoverPort(ref bit [2:0] port, input int mid);
    coverpoint port {
        bins lo = {[0:mid-1]};
        bins hi = {[mid:$]};
    }
endgroup

CoverPort cpa, cpb;

initial begin
    cpa = new(port_a, 4);                  // 采样 port_a，分界点 4
    cpb = new(port_b, 2);                  // 采样 port_b，分界点 2
end
```

> **注意**：covergroup 参数方向遵循就近缺省。如果前一个参数是 `ref`，后一个没写方向也可能被当成 `ref`，常量就不能传入。

---

## 9.9 覆盖率数据分析

### 9.9.1 更多种子还是更多约束

覆盖率没满时，优先顺序通常是：

1. 继续使用更多随机种子。
2. 加长测试运行时间。
3. 增加或修改约束，把随机激励拉到盲区。
4. 最后才写定向测试。

| 报告现象 | 判断 |
|----------|------|
| 覆盖率持续增长 | 继续跑更多种子 |
| 某些 bin 一直 0 | 约束可能到不了目标区域 |
| 某些 bin 只命中一次 | 激励概率太低，可能要引导 |
| 增长完全停滞 | 换约束或定向测试 |

---

### 9.9.2 分布不均和 solve before

两个随机变量相加得到的长度，分布不一定均匀，类似两个骰子点数和中间值概率更大。

```systemverilog
class Transaction;
    rand bit [2:0] hdr_len;
    rand bit [3:0] payload_len;
    rand bit [4:0] len;

    constraint length {
        len == hdr_len + payload_len;     // len 被两个变量共同决定
    }
endclass
```

如果想让 `len` 更均匀，可以先求解 `len`。

```systemverilog
constraint length {
    len == hdr_len + payload_len;
    solve len before hdr_len, payload_len; // 先决定 len，再决定两个组成部分
}
```

`solve before` 不是赋值顺序，而是随机求解顺序提示。这里的意思是：先让求解器选择 `len`，再选择能满足 `len == hdr_len + payload_len` 的 `hdr_len` 和 `payload_len`。

| 写法 | 随机分布倾向 |
|------|--------------|
| 只有 `len == hdr_len + payload_len` | `hdr_len`/`payload_len` 的组合更自然，中间 `len` 更容易出现 |
| 加 `solve len before hdr_len, payload_len` | 先选择 `len`，让 `len` 的分布更接近均匀 |

例如 `len=0` 只有 `0+0` 一种组合，但 `len=7` 可以由 `0+7`、`1+6`、`2+5` 等多种组合得到。不加 `solve before` 时，中间值更容易命中；加上后，边界长度更容易被随机打到。

`dist` 不一定能解决这种问题，因为 `len` 仍然受等式约束影响。

---

## 9.10 仿真过程中查询覆盖率

SystemVerilog 支持在仿真过程中查询覆盖率。

| 函数 | 作用 |
|------|------|
| `$get_coverage()` | 查询所有覆盖组的总覆盖率 |
| `CoverGroup::get_coverage()` | 查询某个覆盖组类型的覆盖率 |
| `cg_inst.get_coverage()` | 查询某个覆盖组实例/类型相关覆盖率 |
| `cg_inst.get_inst_coverage()` | 查询单个实例覆盖率 |

```systemverilog
initial begin
    repeat (10000) begin
        run_one_transaction();

        if ($get_coverage() >= 95.0) begin // 全局覆盖率达到 95%
            $display("coverage target reached");
            break;
        end
    end
end
```

> **注意**：不要在同一次仿真里随意改变多个随机种子，否则 bug 难以复现。

---

## 9.11 本章总结

### 学习重点排序

| 优先级 | 必须掌握 |
|------|----------|
| 🔴 高 | 功能覆盖率和代码覆盖率的区别 |
| 🔴 高 | `covergroup`、`coverpoint`、`sample()` 的基本用法 |
| 🔴 高 | 自动 bin、自定义 `bins`、命名 bin |
| 🔴 高 | `ignore_bins`、`illegal_bins`、`iff` 的区别 |
| 🟡 中 | transition bin：`(0=>1)`、`[*]` 重复 |
| 🟡 中 | wildcard bin：`3'b??0` |
| 🟡 中 | cross coverage：`cross kind, port` |
| 🟡 中 | `binsof()`、`intersect`、`option.weight` |
| 🟢 进阶 | 参数化 covergroup：`input` / `ref` 参数 |

---

### 最重要的 10 条规则

| # | 规则 | 说明 |
|---|------|------|
| 1 | 功能覆盖率必须来自验证计划 | 不要为了覆盖而覆盖 |
| 2 | 关注信息，不要盲目覆盖所有数值 | FIFO 要看 empty/full，不是所有指针组合 |
| 3 | covergroup 必须实例化 | 忘记 `new()` 时报告里不会出现该覆盖组 |
| 4 | 采样时刻比采样变量更重要 | 要在事务真正有效时 `sample()` |
| 5 | 大范围变量不要依赖自动 bin | 用自定义 bin 描述关心的范围 |
| 6 | `default` bin 可以防止漏掉未考虑值 | 尤其适合手写命名 bin 时 |
| 7 | 复位期间通常要关闭覆盖 | 用 `iff` 或 `start()`/`stop()` |
| 8 | cross 会让 bin 数量相乘 | 交叉前先确认组合真的有意义 |
| 9 | `ignore_bins` 不计覆盖，`illegal_bins` 表示错误 | 两者语义完全不同 |
| 10 | 覆盖率不增长时先加种子，再改约束 | 定向测试是最后手段 |

---

### 最容易错的点

| 易错点 | 正确理解 |
|--------|----------|
| 以为代码覆盖率 100% 就完成验证 | 错，代码执行过不等于设计意图被测到 |
| 把覆盖点定义得太细 | 报告巨大且不可读，应该覆盖关键信息 |
| 忘记实例化 covergroup | 不会报空句柄错，但报告里没有该覆盖组 |
| 在事务还没被 DUT 接收前采样 | 覆盖率会统计到无效或被丢弃事务 |
| 对表达式采样不检查位宽 | 自动 bin 可能覆盖不到真实范围 |
| 自定义 bin 后忘记 `default` | 未列出的值会被静默忽略 |
| 把 `ignore_bins` 当错误检查 | 错，非法值应使用 `illegal_bins` 或断言 |
| cross 太多变量 | bin 数爆炸，覆盖率收敛很慢 |
| 在一次仿真中乱改随机种子 | bug 难以复现，调试成本很高 |

第 9 章的本质是：**把验证计划翻译成可执行的覆盖模型，用报告驱动随机种子、约束和测试的迭代，直到真正重要的设计状态都被测到。**
