# UVM 语法速查（ch2~ch5 重点）

> UVM 常用 API 与宏语法速查，覆盖《UVM 实战》第 2~5 章的重点基础语法。
>
> 阅读配套：`uvm_ch1~ch9学习笔记.md`（UVM 实战学习笔记）

## 目录

1. [类与注册](#一类与注册)
2. [对象创建](#二对象创建)
3. [phase 回调](#三phase-回调)
4. [agent 与 virtual interface](#四agent-与-virtual-interface)
5. [config_db 与字段自动化](#五config_db-与字段自动化)
6. [TLM 通信](#六tlm-通信)
7. [sequence 与激励](#七sequence-与激励)
8. [objection 与 drain time](#八objection-与-drain-time)
9. [phase 运行机制](#九phase-运行机制)
10. [日志、调试与顶层启动](#十日志调试与顶层启动)

---

## 一、类与注册

**类分两大类**：组件（进树、有 phase）和对象（不进树、无 phase）：

```systemverilog
// [1] 组件类：进 UVM 树、有 phase、构造带 parent
class my_driver extends uvm_driver;        // driver/sequencer/monitor/agent/env/test
class my_agent  extends uvm_agent;
class my_test   extends uvm_test;

// [2] 对象类：不进树、无 phase、按需创建
class my_transaction extends uvm_sequence_item;   // 事务
class my_sequence    extends uvm_sequence #(my_transaction);  // 序列

// [3] 参数化：driver/sequencer/sequence 用 #(事务类型)
class my_driver    extends uvm_driver    #(my_transaction);
class my_sequencer extends uvm_sequencer #(my_transaction);
```

**注册宏**（4 个维度：object/component × 普通/begin-end）：

```systemverilog
`uvm_object_utils(my_transaction)              // [1] 对象注册
`uvm_object_utils_begin(my_transaction)        // [2] 对象注册 + 登记字段
    `uvm_field_int(data, UVM_ALL_ON)
`uvm_object_utils_end
`uvm_component_utils(my_driver)                // [3] 组件注册
`uvm_component_utils_begin(my_driver)          // [4] 组件注册 + 登记字段
    `uvm_field_int(pre_num, UVM_ALL_ON)
`uvm_component_utils_end
// 参数化类用 param 版：`uvm_object_param_utils / `uvm_component_param_utils
```

> 记忆：**组件 = 平台固定设备（树/phase），对象 = 流动数据（transaction/sequence）；带 begin-end 才有字段自动化。**

---

## 二、对象创建

```systemverilog
// [1] new：组件带 parent（决定挂树位置），对象不带
function new(string name = "my_driver", uvm_component parent = null);
    super.new(name, parent);      // 必须先调 super
endfunction

// [2] create：必须用 create 不用 new（只有 create 走 factory，override 才生效）
tr  = my_transaction::type_id::create("tr");      // 对象（无 parent）
drv = my_driver::type_id::create("drv", this);    // 组件（this = 挂到当前组件下）
```

> 记忆：**`类型::type_id::create(名字, [parent])`；组件必须传 parent；name 是实例名（进 UVM 路径）。**

---

## 三、phase 回调

每个组件只需实现自己关心的 phase。**function phase 不耗时（搭结构/收尾），task phase 可耗时（跑激励）**：

```systemverilog
// [1] function phase：不耗时
function void build_phase(uvm_phase phase);   // 创建子组件、读配置（自顶向下）
    super.build_phase(phase);                 // 通常必须调 super（字段配置应用）
    drv = my_driver::type_id::create("drv", this);
endfunction
function void connect_phase(uvm_phase phase); // 连接端口（自底向上）
    super.connect_phase(phase);
    port.connect(...);
endfunction
function void end_of_elaboration_phase(uvm_phase phase);  // 检查结构/打印拓扑
function void report_phase(uvm_phase phase);              // 汇总 PASS/FAIL

// [2] task phase：可耗时
task main_phase(uvm_phase phase);
    // 跑激励（必须有 objection 才能持续，见第八节）
endtask
```

> 记忆：**build 自顶向下、connect 自底向上、task 并发；回调都带 `uvm_phase phase` 参数（用它 raise/drop/jump）。**

---

## 四、agent 与 virtual interface

**agent = 同一协议的一组组件**（driver + sequencer + monitor），用 `is_active` 区分输入/输出侧：

```systemverilog
class my_agent extends uvm_agent;
    `uvm_component_utils(my_agent)

    function void build_phase(uvm_phase phase);
        super.build_phase(phase);
        if (is_active == UVM_ACTIVE) begin   // [1] active：建 driver + sequencer
            drv = my_driver::type_id::create("drv", this);
            sqr = my_sequencer::type_id::create("sqr", this);
        end
        mon = my_monitor::type_id::create("mon", this);  // [2] 无论 active/passive 都有 monitor
    endfunction

    function void connect_phase(uvm_phase phase);
        super.connect_phase(phase);
        if (is_active == UVM_ACTIVE)
            drv.seq_item_port.connect(sqr.seq_item_export);  // [3] driver-sequencer 接线
    endfunction
endclass
```

**virtual interface（vif）**——class 与 HDL 信号的桥（class 不能直接持有 interface 实例）：

```systemverilog
virtual my_if vif;             // [1] 声明 virtual interface 句柄

// [2] top_tb 里 set，组件 build 里 get
uvm_config_db#(virtual my_if)::set(null, "uvm_test_top.env.i_agt.drv", "vif", input_if);
uvm_config_db#(virtual my_if)::get(this, "", "vif", vif);
```

> 记忆：**输入 agent 要发激励（active）、输出 agent 只观察（passive）；vif 必须靠 config_db 下发，拿不到就用 `uvm_fatal` 报错。**

---

## 五、config_db 与字段自动化

**config_db：set 寄快递、get 收快递**：

```systemverilog
// [1] set：四参数（上下文/路径/字段名/值）
uvm_config_db#(virtual my_if)::set(
    null,                              // top_tb 没 this 用 null
    "uvm_test_top.env.i_agt.drv",      // 目标路径（有 this 可写相对路径）
    "vif",                             // 字段名（get 必须一字不差）
    input_if
);

// [2] get：四参数，返回 1/0
if (!uvm_config_db#(virtual my_if)::get(this, "", "vif", vif))
    `uvm_fatal("NO_VIF", "vif not configured");

// [3] 挂默认 sequence（路径带 phase 名）
uvm_config_db#(uvm_object_wrapper)::set(
    this, "env.i_agt.sqr.main_phase", "default_sequence",
    my_sequence::type_id::get());
```

**字段自动化**——begin/end 注册宏 + 字段宏，自动获得 copy/compare/print/pack：

```systemverilog
`uvm_object_utils_begin(packet)
    `uvm_field_int(data, UVM_ALL_ON)                // 整数/bit/logic
    `uvm_field_enum(frame_kind_e, kind, UVM_ALL_ON) // 枚举（带类型名）
    `uvm_field_string(source, UVM_ALL_ON)           // 字符串
    `uvm_field_array_int(payload, UVM_ALL_ON)       // 动态数组
`uvm_object_utils_end

dst.copy(src);                    // 复制
same = actual.compare(expected);  // 比较（返回 1/0）
tr.print();                       // 打印
$cast(copy_tr, tr.clone());       // 克隆（返回 uvm_object，要 $cast 转回）
```

> 记忆：**类型、路径、字段名、时间四要素任一不匹配 get 失败；跨层高层赢、同层后写赢；组件内 set 用 this，top_tb 用 null。**

---

## 六、TLM 通信

组件间用端口交换 transaction：**PORT（发起）/ EXPORT（转发）/ IMP（实现）**，链路必须落到 IMP 或 FIFO。

**analysis：发布-订阅广播（验证平台主力）**：

```systemverilog
// [1] 发布者（monitor）：广播，发完不管
uvm_analysis_port #(my_transaction) ap;
ap = new("ap", this);
ap.write(tr);                        // 广播（write 是 function，不阻塞）

// [2] 订阅者（scoreboard 等）：实现 write
uvm_analysis_imp #(my_transaction, my_scoreboard) imp;   // 第二参数 = 实现类
function void write(my_transaction tr); ... endfunction
```

**一个组件收多路数据**（scoreboard 收期望 + 实际）：

```systemverilog
`uvm_analysis_imp_decl(_expected)     // [1] 宏：生成带后缀的端口类型
`uvm_analysis_imp_decl(_actual)
uvm_analysis_imp_expected #(packet, my_scoreboard) exp_imp;   // [2] 期望输入口
uvm_analysis_imp_actual   #(packet, my_scoreboard) act_imp;   // [3] 实际输入口
function void write_expected(...);    // [4] 方法名 = write_宏后缀
function void write_actual(...);
```

**FIFO：中间缓冲**（monitor 推完不管 → scoreboard 主动拉）：

```systemverilog
uvm_tlm_analysis_fifo #(packet) fifo;   // [1] 广播进、get 出
fifo = new("fifo", this);
mon.ap.connect(fifo.analysis_export);       // [2] 广播 → 存进仓库
scb.port.connect(fifo.blocking_get_export); // [3] 仓库 → scoreboard 主动取
// scoreboard 侧用 uvm_blocking_get_port，get() 没数据就阻塞等待
```

> 记忆：**简单订阅（coverage、轻量处理）用 IMP；要节奏解耦、主动拉取、多路缓冲用 FIFO。**

---

## 七、sequence 与激励

```systemverilog
// [1] 定义 sequence：继承 + body + 注册
class my_sequence extends uvm_sequence #(my_transaction);
    my_transaction m_trans;
    `uvm_object_utils(my_sequence)
    virtual task body();
        if (starting_phase != null)
            starting_phase.raise_objection(this);   // [2] sequence 里用 starting_phase
        repeat (10) `uvm_do(m_trans);               // [3] 创建+随机化+发送
        if (starting_phase != null)
            starting_phase.drop_objection(this);
    endtask
endclass

// [4] 启动：seq.start(sequencer)
seq = my_sequence::type_id::create("seq");
seq.start(env.i_agt.sqr);

// [5] driver 侧握手三行（与 get_next_item 配对）
seq_item_port.get_next_item(req);   // 取任务
drive_one_pkt(req);                 // 干任务
seq_item_port.item_done();          // 交任务（漏掉会卡住）
```

> 记忆：**`uvm_do` = 创建+随机化+发送一步完成；`uvm_do_with(req, {约束})` 加临时约束；driver 的 get_next_item/item_done 必须成对。**

---

## 八、objection 与 drain time

**objection = 耗时 phase 的"保活票"**——没有 objection 的耗时 phase 会 0 时间瞬间结束：

```systemverilog
// [1] 组件里
task main_phase(uvm_phase phase);
    phase.raise_objection(this);   // 举手：还有活
    // 耗时代码
    phase.drop_objection(this);    // 放手：干完了
endtask

// [2] sequence 里（用 starting_phase）
virtual task body();
    if (starting_phase != null)
        starting_phase.raise_objection(this);
    repeat (10) `uvm_do(m_trans)
    if (starting_phase != null)
        starting_phase.drop_objection(this);
endtask
```

**drain time（排空时间）**——激励发完后，DUT 流水可能还有数据，再等一段时间：

```systemverilog
task base_test::main_phase(uvm_phase phase);
    phase.phase_done.set_drain_time(this, 200ns);  // [1] objection 归零后再等 200ns
endtask
```

> 记忆：**raise 必须在第一个耗时语句前；raise/drop 必须成对；每个 phase 独立设 drain time，默认 0。**

---

## 九、phase 运行机制

**总体顺序**：

```text
build → connect → end_of_elaboration → start_of_simulation
    → run_phase（与下面 12 个 run-time phase 并行）
        pre_reset → reset → post_reset
        → pre_configure → configure → post_configure
        → pre_main → main → post_main
        → pre_shutdown → shutdown → post_shutdown
    → extract → check → report → final
```

**12 个 run-time phase 分 4 组**（每组 pre+核心+post）：reset 复位 / configure 配置 / main 主激励 / shutdown 收尾。

**phase jump（运行中跳回）**：

```systemverilog
task my_driver::main_phase(uvm_phase phase);
    fork
        forever begin
            seq_item_port.get_next_item(req);
            drive_one_pkt(req);
            seq_item_port.item_done();
        end
        begin
            @(negedge vif.rst_n);               // [1] 检测到复位
            phase.jump(uvm_reset_phase::get()); // [2] 跳回 reset phase
        end
    join
endtask
```

**timeout（超时防挂死）**：

```systemverilog
function void base_test::build_phase(uvm_phase phase);
    super.build_phase(phase);
    uvm_top.set_timeout(500ns, 0);   // [1] 最大仿真时间；0=允许后续覆盖
endfunction
// 命令行：+UVM_TIMEOUT="300ns,YES"
```

**domain（域，高级功能）**——让部分组件有独立的动态 phase 节奏：

```systemverilog
function void block_b::connect_phase(uvm_phase phase);
    super.connect_phase(phase);
    set_domain(local_domain, 1);   // [1] 本组件及所有后代放入新 domain（hier=1 递归）
endfunction
```

> 记忆：**build 自上而下、connect 自下而上、run 并发；jump 影响整个 domain、要防无限跳转；domain 只隔离 12 个 run-time phase，函数 phase 仍全局同步。**

---

## 十、日志、调试与顶层启动

**日志宏**：

```systemverilog
`uvm_info("ID", "消息内容", UVM_LOW)       // [1] 信息（3 参数：ID/消息/verbosity）
`uvm_warning("ID", "可疑但可继续")          // [2] 警告
`uvm_error("ID", "明确错误")                // [3] 错误（计入计数）
`uvm_fatal("ID", "无法继续的平台错误")      // [4] 致命（结束仿真）
```

**verbosity 六档**：NONE(0)/LOW(100)/MEDIUM(200)/HIGH(300)/FULL(400)/DEBUG(500)，默认阈值 MEDIUM。

**命令行参数**：

```text
+UVM_TEST_NAME=my_case0        # 选择测试用例（run_test() 空参时读取）
+UVM_VERBOSITY=UVM_HIGH        # 全局调日志阈值
+UVM_CONFIG_DB_TRACE           # 追踪 config_db set/get
+UVM_OBJECTION_TRACE           # 追踪 objection raise/drop
+UVM_PHASE_TRACE               # 追踪 phase 调度（STRT 开始/SKIP 跳过/DONE 完成）
+UVM_TIMEOUT="300ns,YES"       # 超时设置
```

**顶层启动**：

```systemverilog
`include "uvm_macros.svh"      // [1] 宏定义（uvm_do 等）
import uvm_pkg::*;             // [2] 引入 UVM 库

module top;
    initial begin
        run_test("my_test");   // [3] 启动指定测试（空参则读 +UVM_TEST_NAME）
    end
endmodule
```

> 记忆：**info 三参数、error 计数、fatal 退出；`+UVM_XXX` 全是命令行开关，不用重新编译。**

---

> 积累日期：2026-08-16
