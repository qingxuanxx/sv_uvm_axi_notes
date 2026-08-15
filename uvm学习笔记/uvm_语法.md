# UVM 语法

> UVM 常用 API 与宏语法速查（持续更新），当前覆盖第 2、3、5 章内容。
>
> 阅读配套：`uvm_ch1~ch9学习笔记.md`（UVM 实战学习笔记）

## 目录

1. [类与继承](#一类与继承)
2. [构造函数 new](#二构造函数-new)
3. [注册宏](#三注册宏)
4. [创建：type_id::create](#四创建type_idcreate)
5. [phase 回调](#五phase-回调)
6. [objection](#六objection)
7. [日志宏](#七日志宏)
8. [report 控制函数](#八report-控制函数)
9. [config_db](#九config_db)
10. [TLM 端口](#十tlm-端口)
11. [sequence 基础](#十一sequence-基础)
12. [field automation 字段宏](#十二field-automation-字段宏)
13. [树操作函数](#十三树操作函数)
14. [命令行参数](#十四命令行参数)
15. [phase 运行机制](#十五phase-运行机制)
16. [objection 与 drain time](#十六objection-与-drain-time)
17. [超时与 domain](#十七超时与-domain)

---

## 一、类与继承

UVM 的类分两大类，**先分清是哪种，后面所有语法才不会混**：

```systemverilog
// [1] 组件类：进 UVM 树、有 phase 生命周期、构造带 parent
class my_driver extends uvm_driver;      // 最常用的组件基类
class my_env    extends uvm_env;
class my_test   extends uvm_test;
class my_agent  extends uvm_agent;
class my_monitor extends uvm_monitor;

// [2] 对象类：不进树、无 phase、按需创建销毁
class my_transaction extends uvm_sequence_item;   // 事务
class my_sequence    extends uvm_sequence #(my_transaction);  // 序列

// [3] 参数化：driver/sequencer/sequence 用 #(事务类型) 指定传递哪种事务
class my_driver    extends uvm_driver    #(my_transaction);
class my_sequencer extends uvm_sequencer #(my_transaction);
```

> 记忆：**组件 = 平台的固定设备（树/phase），对象 = 流动的数据（transaction/sequence）**。

---

## 二、构造函数 new

```systemverilog
// [1] 组件：new(name, parent) —— parent 决定挂树位置
function new(string name = "my_driver", uvm_component parent = null);
    super.new(name, parent);   // 必须先调 super
endfunction

// [2] 对象：new(name) —— 没有 parent
function new(string name = "my_transaction");
    super.new(name);
endfunction
```

> 注意：name 是**实例名**（进 UVM 路径），不是类名；parent 传 this 就是挂到当前组件下面。

---

## 三、注册宏

注册 = 给类办"UVM 身份证"，factory 才能识别它。**object 和 component 用不同前缀**：

```systemverilog
// [1] object 注册（无字段自动化）
`uvm_object_utils(my_transaction)

// [2] object 注册 + 字段登记（begin/end 版）
`uvm_object_utils_begin(my_transaction)
    `uvm_field_int(data, UVM_ALL_ON)
`uvm_object_utils_end

// [3] component 注册
`uvm_component_utils(my_driver)

// [4] component 注册 + 字段登记（配合 super.build_phase 可自动应用配置）
`uvm_component_utils_begin(my_driver)
    `uvm_field_int(pre_num, UVM_ALL_ON)
`uvm_component_utils_end

// [5] 参数化类用 param 版（参数要写全）
`uvm_object_param_utils(data_item #(WIDTH))
`uvm_component_param_utils(bus_agent #(16, 32))
```

> 记忆：**4 个维度（object/component × 普通/begin-end）**，带 begin-end 才能登记字段、才有字段自动化。

---

## 四、创建：type_id::create

**必须用 create 不用 new**——只有 create 经过 factory，override 替换才能生效：

```systemverilog
// [1] 对象创建（无 parent）
tr = my_transaction::type_id::create("tr");

// [2] 组件创建（带 parent，this = 挂到当前组件下）
drv = my_driver::type_id::create("drv", this);
```

> 记忆：`类型::type_id::create(名字, [parent])`，组件必须传 parent。

---

## 五、phase 回调

phase 是 UVM 自动调用的生命周期方法，**每个组件只需实现自己关心的**：

```systemverilog
// [1] 函数 phase：不耗时，搭结构
function void build_phase(uvm_phase phase);    // 创建子组件、读配置（自顶向下）
    super.build_phase(phase);
    drv = my_driver::type_id::create("drv", this);
endfunction

function void connect_phase(uvm_phase phase); // 连接端口（自底向上）
    super.connect_phase(phase);
    port.connect(...);
endfunction

function void end_of_elaboration_phase(uvm_phase phase);  // 检查结构/打印拓扑
function void report_phase(uvm_phase phase);              // 汇总结果 PASS/FAIL

// [2] 任务 phase：可耗时，跑激励
task main_phase(uvm_phase phase);
    // 干活（要有 objection 才能持续）
endtask
```

> 注意：**build 是 top-down，connect 是 bottom-up**；别忘了 `super.xxx_phase(phase)`（父类可能有必要的默认行为）。

---

## 六、objection

objection = 耗时 phase 的"保活票"。**没有 objection 的耗时 phase 会瞬间结束**：

```systemverilog
task main_phase(uvm_phase phase);
    phase.raise_objection(this);   // [1] 举手：我还有活没干完
    // ... 耗时代码 ...
    phase.drop_objection(this);    // [2] 放手：干完了
endtask
```

> 注意：raise 必须在**第一个耗时语句之前**；raise/drop 必须成对；sequence 里用 `starting_phase.raise_objection(this)`。

---

## 七、日志宏

UVM 统一日志宏，比 $display 多 ID、verbosity、自动带时间/路径：

```systemverilog
`uvm_info("ID", "消息内容", UVM_LOW)       // [1] 普通信息：3 个参数
`uvm_warning("ID", "可疑但可继续")          // [2] 警告
`uvm_error("ID", "明确错误")                // [3] 错误（计入计数）
`uvm_fatal("ID", "无法继续的平台错误")      // [4] 致命（结束仿真）
```

> 记忆：**info 三参数（ID/消息/verbosity），error 计数、fatal 退出**。verbosity 六档：NONE(0)/LOW(100)/MEDIUM(200)/HIGH(300)/FULL(400)/DEBUG(500)，默认阈值 MEDIUM，HIGH 及以上默认看不见。

---

## 八、report 控制函数

控制日志的阈值、严重性、行为、文件。**命名规律：`set_report_<谁>_<做什么>`**：

```systemverilog
// [1] 调阈值：verbosity
env.i_agt.drv.set_report_verbosity_level(UVM_HIGH);        // 单个组件
env.i_agt.set_report_verbosity_level_hier(UVM_HIGH);       // 递归整个子树（_hier）
set_report_id_verbosity("DRV_DATA", UVM_HIGH);             // 按 ID

// [2] 改判严重性：severity override
set_report_severity_override(UVM_WARNING, UVM_ERROR);      // 所有 warning 按 error 处理
set_report_severity_id_override(UVM_WARNING, "DRV_PROTOCOL", UVM_ERROR);  // 只改某 ID

// [3] 限次退出
set_report_max_quit_count(5);    // error 计数到 5 结束仿真

// [4] 行为 action（位掩码可组合）
set_report_severity_action(UVM_WARNING, UVM_DISPLAY | UVM_COUNT);  // 打印+计数
set_report_severity_action(UVM_WARNING, UVM_DISPLAY | UVM_STOP);   // 打印+停下调试
set_report_severity_file(UVM_ERROR, error_log);           // 指定文件（配 UVM_LOG 才生效）
set_report_severity_action(UVM_ERROR, UVM_DISPLAY | UVM_COUNT | UVM_LOG);
```

> 记忆：`_hier` = 递归子树；action 位掩码（DISPLAY 打印/COUNT 计数/LOG 写文件/STOP 停下/EXIT 退出）可组合；**只设 file 不加 UVM_LOG 不写文件**。

---

## 九、config_db

配置传递机制，set 是寄快递、get 是收快递：

```systemverilog
// [1] set：四个参数（上下文/路径/字段名/值）
uvm_config_db#(virtual my_if)::set(
    null,                              // top_tb 没 this 用 null
    "uvm_test_top.env.i_agt.drv",      // 完整路径（有 this 可写相对路径）
    "vif",                             // 字段名（get 必须一字不差）
    input_if                           // 配置值
);

// [2] get：四个参数，返回 1/0
if (!uvm_config_db#(virtual my_if)::get(
        this, "", "vif", vif))
    `uvm_fatal("NO_VIF", "vif not configured");

// [3] 传其他类型
uvm_config_db#(int)::set(this, "env.i_agt.drv", "pre_num", 100);
uvm_config_db#(agent_config)::set(this, "env.i_agt", "cfg", cfg);
uvm_config_db#(uvm_object_wrapper)::set(
    this, "env.i_agt.sqr.main_phase", "default_sequence",
    my_sequence::type_id::get());      // 挂默认 sequence

// [4] 调试工具
check_config_usage();          // 查"写了没人读"的配置（放 connect_phase）
print_config(1);               // 打印可见配置（1=递归子树）
// 命令行：+UVM_CONFIG_DB_TRACE 追踪 set/get 全过程
```

> 记忆：**类型、路径、字段名、时间四要素任一不匹配 get 失败**；优先级——跨层高层赢、同层后写赢；组件内 set 用 this，top_tb 用 null。

---

## 十、TLM 端口

ch2 用到的三种端口（通信在 connect_phase 接线）：

```systemverilog
// [1] 广播端口：monitor 发布事务（推，不等接收方）
uvm_analysis_port #(my_transaction) ap;
ap = new("ap", this);
ap.write(tr);                          // 非阻塞广播

// [2] 拉取端口：model/scoreboard 收事务（拉，会阻塞等待）
uvm_blocking_get_port #(my_transaction) port;
port.get(tr);                          // 没数据就等

// [3] FIFO 桥接：缓冲，解耦"推"和"拉"的节奏
uvm_tlm_analysis_fifo #(my_transaction) fifo;
fifo = new("fifo", this);

// [4] connect_phase 里接线
mon.ap.connect(fifo.analysis_export);          // 广播 → FIFO 收口
mdl.port.connect(fifo.blocking_get_export);    // FIFO 出口 → 拉取口
```

> 记忆：**analysis_port 是广播喇叭（write 不等）、get_port 是收货口（get 会等）、FIFO 当缓冲仓库**。

---

## 十一、sequence 基础

sequence 产生事务、sequencer 派发、driver 驱动：

```systemverilog
// [1] 定义 sequence：继承 + body + 注册
class my_sequence extends uvm_sequence #(my_transaction);
    my_transaction m_trans;
    `uvm_object_utils(my_sequence)
    virtual task body();
        if (starting_phase != null)
            starting_phase.raise_objection(this);   // 控制 objection
        repeat (10) `uvm_do(m_trans);               // 创建+随机化+发送
        if (starting_phase != null)
            starting_phase.drop_objection(this);
    endtask
endclass

// [2] 启动：seq.start(sequencer)
seq = my_sequence::type_id::create("seq");
seq.start(env.i_agt.sqr);

// [3] driver 侧握手三行（与 get_next_item 配对）
seq_item_port.get_next_item(req);   // 取任务
drive_one_pkt(req);                 // 干任务
seq_item_port.item_done();          // 交任务（漏掉会卡住）
```

> 记忆：**`uvm_do` = 创建+随机化+发送一步完成；`uvm_do_with(req, {约束})` 加临时约束；driver 的 get_next_item/item_done 必须成对**。

---

## 十二、field automation 字段宏

在 begin/end 注册宏里登记字段，自动获得 copy/compare/print/pack：

```systemverilog
`uvm_object_utils_begin(packet)
    `uvm_field_int(data, UVM_ALL_ON)                    // 整数/bit/logic
    `uvm_field_enum(frame_kind_e, kind, UVM_ALL_ON)     // 枚举（要带类型名）
    `uvm_field_string(source, UVM_ALL_ON)               // 字符串
    `uvm_field_array_int(payload, UVM_ALL_ON)           // 动态数组
    `uvm_field_int(crc_err, UVM_ALL_ON | UVM_NOPACK)    // 控制字段：全自动但不打包
`uvm_object_utils_end
```

自动获得的操作：

```systemverilog
dst.copy(src);                    // 复制
same = actual.compare(expected);  // 比较（返回 1/0）
tr.print();                       // 打印
pack_bytes(arr);                  // 打包（按注册顺序）
$cast(copy_tr, tr.clone());       // 克隆（返回 uvm_object 句柄，要 $cast 转回）
```

> 记忆：**枚举宏多带类型名、控制字段加 UVM_NOPACK（不进数据流）、pack 顺序=注册顺序**。

---

## 十三、树操作函数

在任何组件里查询树结构：

```systemverilog
get_full_name();          // 完整路径（如 uvm_test_top.env.i_agt.drv）
get_parent();             // 父组件（只有一个，无参数）
get_child("drv");         // 按名取子组件（多个，要参数）
get_children(queue);      // 全部直接孩子（不递归）
get_num_children();       // 孩子数量
uvm_top / uvm_root::get() // 真正的根（全局唯一）
uvm_test_top              // 测试实例名（config_db 绝对路径从它开始写）
```

> 记忆：**parent 只有一个不用参数，child 一堆要指名；路径由实例名（create 的字符串）决定，不是变量名**。

---

## 十四、命令行参数

不改源码调 UVM 行为：

```text
+UVM_TEST_NAME=my_case0        # 选择测试用例（run_test() 空参时读取）
+UVM_VERBOSITY=UVM_HIGH        # 全局调日志阈值
+UVM_MAX_QUIT_COUNT=6,NO       # 限次退出（NO=锁死不许覆盖）
+uvm_set_severity=组件,ID,旧,新 # 改判严重性
+UVM_CONFIG_DB_TRACE           # 追踪 config_db set/get
+UVM_OBJECTION_TRACE           # 追踪 objection raise/drop
```

> 记忆：**`+UVM_XXX` 全是命令行开关，回归脚本里常用，不用重新编译**。

---

## 十五、phase 运行机制

phase 分**函数 phase**（不耗时）和**任务 phase**（可耗时），整体顺序：

```text
build → connect → end_of_elaboration → start_of_simulation
    → run_phase（与下面 12 个 run-time phase 并行）
        pre_reset → reset → post_reset
        → pre_configure → configure → post_configure
        → pre_main → main → post_main
        → pre_shutdown → shutdown → post_shutdown
    → extract → check → report → final
```

```systemverilog
// [1] 函数 phase 回调签名（不耗时：建结构/收尾）
function void build_phase(uvm_phase phase);
function void connect_phase(uvm_phase phase);
function void end_of_elaboration_phase(uvm_phase phase);
function void start_of_simulation_phase(uvm_phase phase);
function void extract_phase(uvm_phase phase);
function void check_phase(uvm_phase phase);
function void report_phase(uvm_phase phase);
function void final_phase(uvm_phase phase);

// [2] 任务 phase 回调签名（可耗时：跑激励）
task reset_phase(uvm_phase phase);
task configure_phase(uvm_phase phase);
task main_phase(uvm_phase phase);
task shutdown_phase(uvm_phase phase);
// 每个还有 pre_xxx / post_xxx 变体
```

**执行顺序要点**：

- build_phase：**自顶向下**（父先建，子才能建）。
- 其他函数 phase：**自底向上**（子先准备端口，父再连接）。
- 任务 phase：按树顺序**启动后并发运行**，每个组件完成自己的 main 后要等同 domain 其他组件都完成，才一起进入下一个 phase。
- 兄弟组件之间**不能依赖 phase 执行顺序**（标准不保证先后）。

> 记忆：**build 自上而下、connect 自下而上、run 并发；同 phase 的 objection 全部撤销才进入下一 phase**。

**phase 跳转（jump）**——运行中跳回某个 phase：

```systemverilog
task my_driver::main_phase(uvm_phase phase);
    fork
        forever begin
            seq_item_port.get_next_item(req);
            drive_one_pkt(req);
            seq_item_port.item_done();
        end
        begin
            @(negedge vif.rst_n);              // 检测到复位
            phase.jump(uvm_reset_phase::get()); // 跳回 reset phase
        end
    join
endtask
```

> 注意：jump 影响**整个 domain**，不只是调用者；跳转前要处理未 drop 的 objection、清空 FIFO 和队列；防止无限跳转（用标志位限制只跳一次）。

---

## 十六、objection 与 drain time

**objection 控制任务 phase 何时结束**——所有 objection 归零后该 phase 才结束：

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
    #1000;                          // 给 DUT 和 scoreboard 留处理时间
    if (starting_phase != null)
        starting_phase.drop_objection(this);
endtask
```

**drain time（排空时间）**——最后一个事务发完后，DUT 流水可能还有数据在跑，需要再等一段时间：

```systemverilog
task base_test::main_phase(uvm_phase phase);
    phase.phase_done.set_drain_time(this, 200ns);  // objection 归零后再等 200ns
endtask
```

流程：最后一个 drop → objection 归零 → 等待 drain time → 进入 post_main_phase。**每个 phase 独立设置，默认 drain time 为 0**。

**objection 调试**：

```text
+UVM_OBJECTION_TRACE    # 命令行追踪谁 raise/drop、计数变化
```

> 记忆：**没有 objection 的耗时 phase 会 0 时间结束；raise 要在第一个耗时语句前；drain time 解决"激励发完但 DUT 输出还没出来"的问题**。

---

## 十七、超时与 domain

**timeout（超时）**——仿真挂起时自动退出，防止无限跑：

```systemverilog
// [1] 代码里设置（推荐在 base_test 的 build_phase）
function void base_test::build_phase(uvm_phase phase);
    super.build_phase(phase);
    uvm_top.set_timeout(500ns, 0);   // 最大仿真时间 500ns；0=允许后续覆盖
endfunction

// [2] 命令行
// +UVM_TIMEOUT="300ns,YES"   YES=允许代码覆盖 / NO=锁死
```

> 注意：timeout 按**最长合法测试**设置，过短误杀慢场景、过长浪费回归资源。

**domain（域）**——让部分组件拥有独立的 phase 节奏（高级功能）：

```systemverilog
class block_b extends uvm_component;
    uvm_domain local_domain;
    `uvm_component_utils(block_b)
    function new(string name, uvm_component parent);
        super.new(name, parent);
        local_domain = new("local_domain");   // 创建新 domain
    endfunction
    function void connect_phase(uvm_phase phase);
        super.connect_phase(phase);
        set_domain(local_domain, 1);          // 把本组件及所有后代放入新 domain
    endfunction
endclass
```

作用：默认所有组件在 common domain，12 个 run-time phase 必须同步；放进新 domain 后，该子树可以**独立推进**自己的 reset/main 等阶段。phase jump 也只影响调用者所在 domain。

> 记忆：**domain 只隔离 12 个 run-time phase，函数 phase 仍全局同步**；单时钟、统一复位的平台通常不需要用 domain。

---

> 积累日期：2026-08-15
