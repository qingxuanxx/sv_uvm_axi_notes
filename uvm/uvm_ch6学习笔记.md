# UVM实战 第 6 章学习笔记：UVM 中的 sequence
> **核心结论**：sequence 负责描述“产生什么激励以及按什么顺序产生”，driver 只负责“怎样把 transaction 驱动到 DUT”。sequencer 位于二者之间，完成请求仲裁、握手和应答路由。
>
> **记忆主线**：sequence 产生 item -> sequencer 仲裁 -> driver 取 item 并驱动 -> item_done 完成握手 -> 可选 response 返回 sequence。

---

## 6.0 本章定位
本章把前面零散使用的 sequence 机制系统化。

| 章节 | 核心问题 |
|------|----------|
| 6.1 | 为什么要把激励产生从 driver 中剥离，sequence 如何启动 |
| 6.2 | 多个 sequence 竞争同一 sequencer 时如何仲裁 |
| 6.3 | `uvm_do` 系列宏到底完成了哪些步骤 |
| 6.4 | sequence 如何嵌套、继承并访问具体 sequencer |
| 6.5 | virtual sequence 如何协调多个接口 |
| 6.6 | sequence 如何使用 `config_db` |
| 6.7 | driver 如何向 sequence 返回 response |
| 6.8 | sequence library 如何随机选择测试场景 |
学习时要分清三个对象：
1. sequence 是 `uvm_object`，不是 component。
2. sequencer 是 `uvm_component`，负责仲裁与转发。
3. driver 是 `uvm_component`，负责引脚级驱动。
### 零基础阅读提示
先牢牢记住一条流水线：
```text
sequence 造出 req -> sequencer 排队 -> driver 取走 req -> item_done 告知完成
```
- `start_item()` 表示申请发送资格。
- `finish_item()` 表示提交 item，并等待 driver 完成握手。
- `get_next_item()` 是 driver 取请求。
- `item_done()` 是 driver 归还完成通知。
- virtual sequence 不直接驱动 DUT，只负责协调多个真实 sequencer。
第一次阅读可以先使用 `uvm_do`，但一定要知道它内部仍然包含创建、随机化、仲裁和发送。
典型数据通路：
```text
test / virtual sequence
          |
          v
      sequence
          |
     sequence_item
          |
          v
      sequencer
          |
  seq_item_export/port
          |
          v
       driver
          |
          v
         DUT
```

---

## 6.1 sequence 基础
### 6.1.1 从 driver 中剥离激励产生功能
最初把随机化和驱动都写在 driver 中：
```systemverilog
task my_driver::main_phase(uvm_phase phase);
    my_transaction tr;
    phase.raise_objection(this);
    repeat (10) begin
        tr = new("tr");                 // driver 同时负责创建 transaction
        assert(tr.randomize());         // driver 同时决定激励内容
        drive_one_pkt(tr);              // driver 还负责引脚级驱动
    end
    phase.drop_objection(this);
endtask
```
这种写法在只有一种测试场景时可以运行，但扩展性很差。
若要测试 CRC 错误包、超长包、短包、乱序包，就必须反复修改 driver。
问题包括：
- driver 混合了激励策略与总线时序。
- 每增加一个 case 都可能修改稳定的驱动代码。
- 不同测试之间难以复用激励。
- driver 无法保持协议无关的测试意图。
- factory override 的粒度会变得很粗。
正确职责划分：

| 对象 | 主要职责 | 不应该承担 |
|------|----------|------------|
| sequence | 创建、随机化、组织 transaction | 引脚时序 |
| sequencer | 仲裁多个 sequence，路由 req/rsp | 产生业务激励 |
| driver | 将 transaction 转为 pin-level 操作 | 决定测试场景 |
拆分后的 driver：
```systemverilog
task my_driver::main_phase(uvm_phase phase);
    forever begin
        // 阻塞等待 sequencer 交付一个已经准备好的请求。
        seq_item_port.get_next_item(req);
        // driver 只关心协议时序，不关心 req 为什么具有这些字段值。
        drive_one_pkt(req);
        // 通知 sequencer：当前请求已完成，可以发下一个。
        seq_item_port.item_done();
    end
endtask
```
sequence 中产生激励：
```systemverilog
class normal_sequence extends uvm_sequence #(my_transaction);
    `uvm_object_utils(normal_sequence)
    function new(string name = "normal_sequence");
        super.new(name);
    endfunction
    virtual task body();
        repeat (10) begin
            // 宏会创建、随机化并发送 req。
            `uvm_do(req)
        end
    endtask
endclass
```
CRC 错误场景只需定义另一个 sequence：
```systemverilog
class crc_error_sequence extends uvm_sequence #(my_transaction);
    `uvm_object_utils(crc_error_sequence)
    function new(string name = "crc_error_sequence");
        super.new(name);
    endfunction
    virtual task body();
        repeat (10) begin
            `uvm_do_with(req, {
                crc_err == 1;            // 只改变测试意图
            })
        end
    endtask
endclass
```
driver 不需要因为测试场景变化而修改。
#### sequence 的本质
sequence 是一段可复用的事务级激励程序。
它可以：
- 产生一个 transaction。
- 产生一串 transaction。
- 嵌套调用其他 sequence。
- 协调多个 sequencer。
- 根据配置选择激励。
- 接收 driver 返回的 response。
- 通过继承扩展已有场景。
sequence 继承关系：
```text
uvm_object
   |
uvm_sequence_item
   |
uvm_sequence_base
   |
uvm_sequence #(REQ, RSP)
   |
用户 sequence
```
sequence 本身也是 `uvm_sequence_item` 的派生对象，因此宏既可以操作 item，也可以操作子 sequence。

---

### 6.1.2 sequence 的启动与执行
直接启动：
```systemverilog
task my_test::main_phase(uvm_phase phase);
    normal_sequence seq;
    phase.raise_objection(this);
    // sequence 是 object，使用 factory 的 object create 形式。
    seq = normal_sequence::type_id::create("seq");
    // 将 sequence 启动在指定 sequencer 上。
    seq.start(env.i_agt.sqr);
    phase.drop_objection(this);
endtask
```
`start()` 常用原型可理解为：
```systemverilog
seq.start(
    sequencer,       // sequence 使用的 sequencer
    parent_sequence, // 父 sequence；顶层启动时通常为 null
    priority,        // 仲裁优先级；-1 表示使用默认值
    call_pre_post    // 是否调用 pre_body/post_body
);
```
启动后的主要回调顺序：
```text
pre_start
   |
pre_body        仅 call_pre_post 为 1 时调用
   |
body            用户最常重载的任务
   |
post_body       仅 call_pre_post 为 1 时调用
   |
post_start
```
示例：
```systemverilog
class lifecycle_sequence extends uvm_sequence #(my_transaction);
    `uvm_object_utils(lifecycle_sequence) // 注册后才能 factory create
    virtual task pre_body();
        // start() 进入正式 body 之前执行，适合轻量准备工作。
        `uvm_info("SEQ", "pre_body", UVM_LOW)
    endtask
    virtual task body();
        // body 是 sequence 的主体；uvm_do 会产生并发送 req。
        `uvm_info("SEQ", "body", UVM_LOW)
        `uvm_do(req)
    endtask
    virtual task post_body();
        // body 完成后执行，适合统计或收尾。
        `uvm_info("SEQ", "post_body", UVM_LOW)
    endtask
endclass
```
#### default_sequence
可通过 `config_db` 为某个 sequencer 的某个 phase 设置默认 sequence。
按类型设置：
```systemverilog
uvm_config_db #(uvm_object_wrapper)::set(
    this,                                  // 从当前 test 的层次出发
    "env.i_agt.sqr.main_phase",            // 指定 sequencer 的具体 phase
    "default_sequence",                    // UVM 约定的字段名
    normal_sequence::type_id::get()        // 传类型包装器，不是 sequence 实例
);
```
按实例设置：
```systemverilog
normal_sequence seq;
seq = normal_sequence::type_id::create("seq");
uvm_config_db #(uvm_sequence_base)::set(
    this,
    "env.i_agt.sqr.main_phase",
    "default_sequence",
    seq
);
```
两种方式对比：

| 方式 | 特点 |
|------|------|
| 类型 wrapper | sequencer 启动时再通过 factory 创建，可继续 override |
| sequence 实例 | 可在设置前修改对象字段，但实例已经确定 |
工程上通常更推荐由 test 或 virtual sequence 显式 `start()`，因为控制关系更清晰。
#### starting_phase
sequence 不是 component，不能天然知道自己在哪个 phase 中运行。
手工设置方式：
```systemverilog
seq.starting_phase = phase;
seq.start(env.i_agt.sqr);
```
sequence 可在 `pre_body/post_body` 中控制 objection：
```systemverilog
virtual task pre_body();
    if (starting_phase != null)
        starting_phase.raise_objection(this);
endtask
virtual task post_body();
    if (starting_phase != null)
        starting_phase.drop_objection(this);
endtask
```
注意：
- 顶层 sequence 可以控制 objection。
- 被嵌套启动的子 sequence 通常不要重复控制 objection。
- 现代写法也可使用 `set_starting_phase()` 或 phase objection 自动机制。
- objection 必须成对，异常路径也不能漏掉 drop。

---

## 6.2 sequence 的仲裁机制
### 6.2.1 在同一 sequencer 上启动多个 sequence
多个 sequence 可以并发竞争同一个 sequencer：
```systemverilog
task my_test::main_phase(uvm_phase phase);
    short_sequence seq0;
    long_sequence  seq1;
    phase.raise_objection(this);
    seq0 = short_sequence::type_id::create("seq0");
    seq1 = long_sequence ::type_id::create("seq1");
    fork
        seq0.start(env.i_agt.sqr); // 两个分支同时竞争同一个 sequencer
        seq1.start(env.i_agt.sqr);
    join                         // 等待两个 sequence 都结束
    phase.drop_objection(this);
endtask
```
sequencer 的仲裁粒度通常是“一个 item 的发送请求”。
一个 sequence 获得一次授权并发送一个 item 后，其他 sequence 有机会在下一次仲裁中获胜。
因此并发 sequence 可能形成：
```text
seq0:item0
seq1:item0
seq0:item1
seq1:item1
...
```
仲裁结果还受以下因素影响：
- sequence 发出请求的先后时刻。
- 选择的仲裁算法。
- sequence 优先级。
- `lock/grab` 状态。
- `is_relevant()` 返回值。
- driver 消费 item 的速度。
#### 优先级
启动时指定优先级：
```systemverilog
fork
    seq0.start(env.i_agt.sqr, null, 100);
    seq1.start(env.i_agt.sqr, null, 200);
join
```
数值越大，优先级越高。
但优先级是否真正影响结果，取决于仲裁模式。
#### 常见仲裁模式
```systemverilog
env.i_agt.sqr.set_arbitration(UVM_SEQ_ARB_FIFO);
```
教材版本中常见名称可能省略 `UVM_` 前缀，理解其含义即可。

| 模式 | 含义 |
|------|------|
| `UVM_SEQ_ARB_FIFO` | 按请求到达顺序选择，忽略优先级 |
| `UVM_SEQ_ARB_WEIGHTED` | 以优先级作为权重随机选择 |
| `UVM_SEQ_ARB_RANDOM` | 在有效请求中随机选择 |
| `UVM_SEQ_ARB_STRICT_FIFO` | 先选最高优先级，再按 FIFO 选择 |
| `UVM_SEQ_ARB_STRICT_RANDOM` | 先选最高优先级，再随机选择 |
| `UVM_SEQ_ARB_USER` | 调用用户自定义仲裁函数 |
选择建议：
- 一般协议流量使用 FIFO。
- 需要模拟 QoS 时使用 weighted 或 strict 模式。
- 压力测试可使用 random。
- 复杂业务规则才考虑 user arbitration。
不要把高优先级误认为永久独占。
高优先级只影响每次仲裁选择，除非使用 lock 或 grab。

---

### 6.2.2 sequencer 的 lock 操作
`lock()` 用于让一个 sequence 连续拥有 sequencer。
```systemverilog
virtual task body();
    repeat (3)
        `uvm_do(req)
    // lock 请求也要进入 sequencer 的正常仲裁队列。
    lock();
    repeat (5) begin
        `uvm_do_with(req, {
            packet_type == CONTROL;
        })
    end
    // 必须显式释放，否则其他 sequence 可能永久等待。
    unlock();
    repeat (2)
        `uvm_do(req)
endtask
```
`lock()` 的语义：
1. sequence 提交锁请求。
2. 锁请求按当前仲裁规则等待。
3. 获得锁后，其他 sequence 不再获得 item 授权。
4. 当前 sequence 可连续发送多个 item。
5. 调用 `unlock()` 后恢复正常仲裁。
适合场景：
- 一组 transaction 必须原子执行。
- 协议配置序列不能被普通流量插入。
- 连续 burst 必须保持语义完整。
- 复位后的初始化步骤不能被打断。
风险：
- 忘记 `unlock()` 会造成饥饿或死锁。
- 持锁期间阻塞等待外部事件会长期占用 sequencer。
- 子 sequence 的锁范围不清晰会让调试困难。

---

### 6.2.3 sequencer 的 grab 操作
`grab()` 也会独占 sequencer，但它比 `lock()` 更强。
```systemverilog
virtual task body();
    // grab 请求插到普通 lock 请求之前，尽快抢占仲裁权。
    grab();
    repeat (4) begin
        `uvm_do_with(req, {
            packet_type == EMERGENCY;
        })
    end
    ungrab();
endtask
```
`lock` 与 `grab` 对比：

| 项目 | `lock()` | `grab()` |
|------|----------|----------|
| 是否独占 | 是 | 是 |
| 请求位置 | 正常仲裁队列尾部 | 抢占式插入锁请求之前 |
| 紧急程度 | 普通原子操作 | 紧急控制流 |
| 释放方法 | `unlock()` | `ungrab()` |
`grab()` 不能中断 driver 正在处理的 item。
它只能影响后续的 sequencer 仲裁。
使用原则：
- 普通原子序列优先用 `lock()`。
- 真正紧急的复位、错误恢复才考虑 `grab()`。
- 独占范围越短越好。
- 所有退出路径都要释放所有权。

---

### 6.2.4 sequence 的有效性
sequence 可以主动暂时退出仲裁。
sequencer 通过 `is_relevant()` 判断 sequence 当前是否有效。
默认实现返回 1。
```systemverilog
class throttled_sequence extends uvm_sequence #(my_transaction);
    `uvm_object_utils(throttled_sequence)
    int sent_count;
    bit ready_again;
    virtual function bit is_relevant();
        // 已发送 3 个 item 且尚未重新就绪时，暂不参与仲裁。
        return !((sent_count >= 3) && !ready_again);
    endfunction
    virtual task wait_for_relevant();
        // 当所有候选 sequence 都无效时，sequencer 可能调用此任务。
        #10us;
        // 必须改变导致无效的条件，否则会形成反复检查的死循环。
        ready_again = 1'b1;
    endtask
    virtual task body();
        repeat (10) begin
            `uvm_do(req)
            sent_count++;
        end
    endtask
endclass
```
关键规则：
- `is_relevant()` 是 function，不能耗时。
- `wait_for_relevant()` 是 task，可以等待。
- 两者一般应成对重载。
- `wait_for_relevant()` 返回前应使条件有机会变为有效。
- 不能在 `is_relevant()` 中依赖不稳定的副作用。
与独占机制的关系：
```text
lock/grab       主动扩大自己获得 sequencer 的机会
is_relevant=0   主动放弃自己参与 sequencer 仲裁的机会
```

---

## 6.3 sequence 相关宏及其实现
### 6.3.1 `uvm_do` 系列宏
常见宏：

| 宏 | 指定 sequencer | 指定优先级 | 附加约束 |
|----|----------------|------------|----------|
| `uvm_do(obj)` | 当前 | 否 | 否 |
| `uvm_do_pri(obj, pri)` | 当前 | 是 | 否 |
| `uvm_do_with(obj, c)` | 当前 | 否 | 是 |
| `uvm_do_pri_with(obj, pri, c)` | 当前 | 是 | 是 |
| `uvm_do_on(obj, sqr)` | 显式 | 否 | 否 |
| `uvm_do_on_pri(obj, sqr, pri)` | 显式 | 是 | 否 |
| `uvm_do_on_with(obj, sqr, c)` | 显式 | 否 | 是 |
| `uvm_do_on_pri_with(obj, sqr, pri, c)` | 显式 | 是 | 是 |
示例：
```systemverilog
`uvm_do(req)
`uvm_do_with(req, {
    addr inside {[16'h1000 : 16'h1fff]};
    length == 64;
})
`uvm_do_on_pri_with(req, p_sequencer.bus_sqr, 200, {
    kind == WRITE;
})
```
`uvm_do(req)` 可以近似理解为：
```systemverilog
// 1. 通过 factory 创建 req；若 req 已存在，行为要结合宏实现理解。
`uvm_create(req)
// 2. 请求 sequencer 授权，内部会执行 start_item。
start_item(req);
// 3. 对 transaction 随机化。
if (!req.randomize())
    `uvm_error("SEQ", "req randomize failed")
// 4. 完成发送，并等待 driver 对该 item 调用 item_done。
finish_item(req);
```
更完整的概念流程：
```text
create item
   |
wait_for_grant / start_item
   |
pre_do
   |
randomize
   |
mid_do
   |
send_request / finish_item
   |
wait_for_item_done
   |
post_do
```
#### 内联约束作用域
推荐直接写 item 字段：
```systemverilog
`uvm_do_with(req, {
    addr[1:0] == 0;
    data != 0;
})
```
若 sequence 与 item 中存在同名变量，可显式写对象名消除歧义：
```systemverilog
`uvm_do_with(req, {
    req.length == local::length;
})
```
随机化失败必须视为错误，而不是继续发送未知内容。
宏写法简洁，但调试复杂握手时应展开为 `start_item/finish_item`。

---

### 6.3.2 `uvm_create` 与 `uvm_send`
`uvm_create` 只创建对象，不随机化，也不发送。
`uvm_send` 发送已经准备好的对象，不自动重新随机化。
```systemverilog
virtual task body();
    int sequence_number;
    repeat (10) begin
        sequence_number++;
        `uvm_create(req)
        assert(req.randomize() with {
            payload.size() >= 64;
        }) else
            `uvm_fatal("RAND", "req randomize failed")
        // 随机化后进行定向修改。
        // 这里把序号写入 payload 尾部，便于 scoreboard 定位。
        req.payload[req.payload.size()-1] = sequence_number;
        `uvm_send(req)
    end
endtask
```
也可以使用 factory 手工创建：
```systemverilog
req = my_transaction::type_id::create("req");
assert(req.randomize());
`uvm_send(req)
```
需要指定优先级时：
```systemverilog
`uvm_send_pri(req, 200)
```
适合拆分 create/send 的场景：
- 随机化后计算 CRC。
- 根据随机结果修改关联字段。
- 插入 sequence 编号。
- 先构建一批 item，再按条件发送。
- 需要精确定位随机化失败发生在哪一步。

---

### 6.3.3 `uvm_rand_send` 系列宏
`uvm_rand_send` 用于对已经创建的对象重新随机化后发送。
```systemverilog
virtual task body();
    `uvm_create(req)
    repeat (10) begin
        // 每次重新随机化同一个对象，再发送。
        `uvm_rand_send(req)
    end
endtask
```
带约束：
```systemverilog
`uvm_rand_send_with(req, {
    length inside {[64:512]};
})
```
带优先级：
```systemverilog
`uvm_rand_send_pri_with(req, 150, {
    kind == READ;
})
```
注意对象复用风险：
- driver 或 subscriber 若保存了同一个句柄，下一次随机化会修改旧记录。
- analysis 广播后如需保留历史数据，应 clone/copy。
- 并行发送不能共享一个可变 item 对象。

---

### 6.3.4 `start_item` 与 `finish_item`
推荐掌握的显式写法：
```systemverilog
virtual task body();
    req = my_transaction::type_id::create("req");
    // 等待当前 sequence 获得 sequencer 的一次发送许可。
    start_item(req);
    // 授权后再随机化，可让随机化时读取接近发送时刻的状态。
    assert(req.randomize() with {
        addr[1:0] == 0;
    }) else
        `uvm_fatal("RAND", "request randomization failed")
    // 把 item 交给 driver，并等待 driver 调用 item_done。
    finish_item(req);
endtask
```
重要约束：
- `start_item()` 与 `finish_item()` 必须成对。
- 两者之间不要加入长时间延迟。
- 获得授权后应尽快完成随机化并发送。
- item 必须与当前 sequencer 接受的 REQ 类型兼容。
- `finish_item()` 返回表示 driver 已调用 `item_done()`，不一定表示 DUT 全部流水处理结束。
常见错误：
```systemverilog
start_item(req);
#100us;                 // 错误倾向：占着仲裁授权长时间不发送
finish_item(req);
```
正确思路是把耗时等待放在 `start_item()` 之前。

---

### 6.3.5 `pre_do`、`mid_do` 与 `post_do`
这些回调允许父 sequence 观察或修改即将执行的 item/子 sequence。
调用位置：
```text
获得 grant
   |
pre_do(is_item)
   |
随机化
   |
mid_do(item_or_sequence)
   |
发送并等待完成
   |
post_do(item_or_sequence)
```
示例：
```systemverilog
virtual task pre_do(bit is_item);
    if (is_item)
        `uvm_info("SEQ_CB", "prepare an item", UVM_HIGH)
    else
        `uvm_info("SEQ_CB", "prepare a child sequence", UVM_HIGH)
endtask
virtual function void mid_do(uvm_sequence_item this_item);
    my_transaction tr;
    if ($cast(tr, this_item)) begin
        // 随机化已经完成，此处可计算派生字段。
        tr.crc = calc_crc(tr.payload);
    end
endfunction
virtual function void post_do(uvm_sequence_item this_item);
    `uvm_info("SEQ_CB", "item completed", UVM_HIGH)
endfunction
```
回调适合横切逻辑：
- 自动填写校验字段。
- 统一打日志。
- 统计发送数量。
- 注入公共元数据。
不要把大量业务逻辑隐藏在回调中，否则阅读 body 时很难理解最终 item。

---

## 6.4 sequence 进阶应用
### 6.4.1 嵌套的 sequence
sequence 的参数既可以是 transaction，也可以在 body 中启动子 sequence。
先定义原子场景：
```systemverilog
class crc_sequence extends uvm_sequence #(my_transaction);
    `uvm_object_utils(crc_sequence)
    virtual task body();
        `uvm_do_with(req, {
            crc_err == 1;
            dmac == 48'h0000_0000_980f;
        })
    endtask
endclass
```
```systemverilog
class long_packet_sequence extends uvm_sequence #(my_transaction);
    `uvm_object_utils(long_packet_sequence)
    virtual task body();
        `uvm_do_with(req, {
            crc_err == 0;
            payload.size() == 1500;
        })
    endtask
endclass
```
组合场景：
```systemverilog
class mixed_sequence extends uvm_sequence #(my_transaction);
    `uvm_object_utils(mixed_sequence)
    virtual task body();
        crc_sequence         crc_seq;
        long_packet_sequence long_seq;
        repeat (10) begin
            // uvm_do 的参数也可以是 sequence。
            `uvm_do(crc_seq)
            `uvm_do(long_seq)
        end
    endtask
endclass
```
也可显式启动：
```systemverilog
crc_seq = crc_sequence::type_id::create("crc_seq");
crc_seq.start(m_sequencer, this);
```
嵌套的价值：
- 原子激励可组合成复杂业务流程。
- 约束只维护一份。
- 子 sequence 可独立测试。
- 父 sequence 只描述流程，不重复字段细节。
注意：
- 子 sequence 的 `parent_sequence` 应指向当前 sequence。
- 子 sequence 通常继承父 sequence 的 sequencer。
- 使用宏启动子 sequence 时，`pre_do` 的 `is_item` 为 0。
- 子 sequence 不宜独立 raise/drop 顶层 phase objection。

---

### 6.4.2 在 sequence 中使用 rand 类型变量
sequence 自身也可以包含 `rand` 字段。
```systemverilog
class burst_sequence extends uvm_sequence #(my_transaction);
    `uvm_object_utils(burst_sequence)
    rand int unsigned packet_count;
    rand int unsigned packet_length;
    constraint c_count {
        packet_count inside {[10:30]};
    }
    constraint c_length {
        packet_length inside {64, 128, 256, 512};
    }
    virtual task body();
        repeat (packet_count) begin
            `uvm_do_with(req, {
                payload.size() == local::packet_length;
            })
        end
    endtask
endclass
```
启动前随机化 sequence：
```systemverilog
seq = burst_sequence::type_id::create("seq");
assert(seq.randomize() with {
    packet_count == 20;
}) else
    `uvm_fatal("RAND", "sequence randomization failed")
seq.start(env.i_agt.sqr);
```
两层随机化：

| 层级 | 决定内容 |
|------|----------|
| sequence 随机化 | 场景级参数，如数量、比例、模式 |
| item 随机化 | 每个 transaction 的字段 |
这样可把“场景随机性”和“单包随机性”分开。

---

### 6.4.3 transaction 类型的匹配
sequence、sequencer、driver 的 REQ 类型必须兼容。
```systemverilog
class my_sequence extends uvm_sequence #(my_transaction);
class my_sequencer extends uvm_sequencer #(my_transaction);
class my_driver extends uvm_driver #(my_transaction);
```
推荐三者使用同一 transaction 类型。
若类型不匹配，可能在编译、连接或运行时 cast 阶段失败。
多态原则：
- driver 按基类 REQ 接收时，可以接收其派生 transaction。
- driver 若假设特定派生字段，必须 `$cast` 并检查结果。
- 不要仅依赖 factory 让不兼容的类型“碰巧能跑”。
```systemverilog
extended_transaction ext;
if (!$cast(ext, req))
    `uvm_fatal("TYPE", "driver received unexpected request type")
```

---

### 6.4.4 `p_sequencer` 的使用
`m_sequencer` 的静态类型是 `uvm_sequencer_base`。
它能支持通用 sequence 操作，但不能直接访问用户 sequencer 的自定义字段。
通过宏声明强类型 `p_sequencer`：
```systemverilog
class cfg_sequence extends uvm_sequence #(my_transaction);
    `uvm_object_utils(cfg_sequence)
    // 宏会声明并在启动时完成类型转换。
    `uvm_declare_p_sequencer(my_sequencer)
    virtual task body();
        if (p_sequencer == null)
            `uvm_fatal("PSEQ", "p_sequencer is null")
        `uvm_info("PSEQ",
                  $sformatf("port_id=%0d", p_sequencer.port_id),
                  UVM_LOW)
    endtask
endclass
```
区别：

| 句柄 | 静态类型 | 主要用途 |
|------|----------|----------|
| `m_sequencer` | `uvm_sequencer_base` | 通用 sequence 基础机制 |
| `p_sequencer` | 用户指定 sequencer 类型 | 访问自定义字段和子 sequencer |
风险：
- sequence 启动在错误类型的 sequencer 上会 cast 失败。
- 过度依赖 `p_sequencer` 会降低 sequence 可复用性。
- 通用 sequence 应优先通过配置对象获取参数。

---

### 6.4.5 sequence 的派生与继承
可以从已有 sequence 派生新场景。
```systemverilog
class base_data_sequence extends uvm_sequence #(my_transaction);
    `uvm_object_utils(base_data_sequence)
    rand int unsigned count = 10;
    virtual task body();
        repeat (count)
            send_one();
    endtask
    virtual task send_one();
        `uvm_do_with(req, {
            crc_err == 0;
        })
    endtask
endclass
```
```systemverilog
class error_data_sequence extends base_data_sequence;
    `uvm_object_utils(error_data_sequence)
    virtual task send_one();
        `uvm_do_with(req, {
            crc_err dist {0 := 9, 1 := 1};
        })
    endtask
endclass
```
设计建议：
- 把稳定流程放在基类 body。
- 把可变化步骤拆成 virtual task/function。
- 派生类只覆盖真正变化的策略。
- 使用 factory 注册，便于 type override。
- 避免形成过深的 sequence 继承树。

---

## 6.5 virtual sequence 的使用
### 6.5.1 带双路输入输出端口的 DUT
当 DUT 有多个接口时，每个 agent 通常有独立 sequencer。
单个普通 sequence 只能自然地绑定一个 sequencer。
virtual sequence 用于协调多个 sequencer 上的子 sequence。
结构：
```text
                 virtual_sequence
                         |
                virtual_sequencer
                 /               \
          p_bus_sqr           p_eth_sqr
              |                   |
       bus_sequence         eth_sequence
              |                   |
         bus_driver            eth_driver
```
virtual sequencer 通常不直接连接 driver。
它只保存其他 sequencer 的句柄。
```systemverilog
class my_virtual_sequencer extends uvm_sequencer;
    `uvm_component_utils(my_virtual_sequencer)
    bus_sequencer p_bus_sqr;
    eth_sequencer p_eth_sqr;
    function new(string name, uvm_component parent);
        super.new(name, parent);
    endfunction
endclass
```
在 env 中连接句柄：
```systemverilog
function void my_env::connect_phase(uvm_phase phase);
    super.connect_phase(phase);
    v_sqr.p_bus_sqr = bus_agt.sqr;
    v_sqr.p_eth_sqr = eth_agt.sqr;
endfunction
```
virtual sequence：
```systemverilog
class system_virtual_sequence extends uvm_sequence;
    `uvm_object_utils(system_virtual_sequence)
    `uvm_declare_p_sequencer(my_virtual_sequencer)
    virtual task body();
        bus_config_sequence cfg_seq;
        eth_data_sequence   data_seq;
        // 配置流量启动在 bus sequencer。
        `uvm_do_on(cfg_seq, p_sequencer.p_bus_sqr)
        // 数据流量启动在 ethernet sequencer。
        `uvm_do_on(data_seq, p_sequencer.p_eth_sqr)
    endtask
endclass
```
virtual sequence 的价值不是“发送一种特殊 transaction”，而是“组织系统级流程”。

---

### 6.5.2 sequence 之间的简单同步
顺序同步最简单：
```systemverilog
`uvm_do_on(reset_seq, p_sequencer.p_bus_sqr)
`uvm_do_on(config_seq, p_sequencer.p_bus_sqr)
`uvm_do_on(data_seq, p_sequencer.p_eth_sqr)
```
后一个 sequence 只有在前一个结束后才启动。
并行同步：
```systemverilog
fork
    `uvm_do_on(tx_seq, p_sequencer.p_tx_sqr)
    `uvm_do_on(rx_seq, p_sequencer.p_rx_sqr)
join
```
`join` 等待两路全部完成。
`join_any` 只等待任一路完成，剩余线程仍会继续运行，除非显式终止。
`join_none` 不等待，需要额外设计收尾同步。

---

### 6.5.3 sequence 之间的复杂同步
可使用 `uvm_event` 表达阶段事件：
```systemverilog
uvm_event cfg_done;
virtual task body();
    cfg_done = new("cfg_done");
    fork
        begin
            config_seq.start(p_sequencer.p_bus_sqr);
            cfg_done.trigger();            // 配置完成后发事件
        end
        begin
            cfg_done.wait_trigger();       // 数据流等待配置完成
            data_seq.start(p_sequencer.p_data_sqr);
        end
    join
endtask
```
也可使用共享状态加 `wait`：
```systemverilog
bit cfg_finished;
fork
    begin
        config_seq.start(p_sequencer.p_bus_sqr);
        cfg_finished = 1'b1;
    end
    begin
        wait (cfg_finished);
        data_seq.start(p_sequencer.p_data_sqr);
    end
join
```
复杂同步要考虑：
- 事件先 trigger 后 wait 是否会丢失。
- 复位后事件状态是否需要清理。
- 一个事件是单次通知还是计数信号。
- 并发分支失败时谁负责结束其他分支。
- sequence 退出时是否仍有后台线程存活。

---

### 6.5.4 仅在 virtual sequence 中控制 objection
推荐由顶层 virtual sequence 或 test 统一控制 objection。
```systemverilog
task my_test::main_phase(uvm_phase phase);
    system_virtual_sequence vseq;
    phase.raise_objection(this);
    vseq = system_virtual_sequence::type_id::create("vseq");
    vseq.start(env.v_sqr);
    phase.drop_objection(this);
endtask
```
子 sequence 只负责业务步骤：
```systemverilog
virtual task body();
    // 不在这里 raise/drop，避免组合后 objection 责任混乱。
    repeat (100)
        `uvm_do(req)
endtask
```
统一控制的优点：
- 测试结束条件清晰。
- 子 sequence 可自由组合。
- 避免多个子 sequence 重复 raise/drop。
- 易于设置 drain time。

---

### 6.5.5 在 sequence 中慎用 fork
危险写法：
```systemverilog
virtual task body();
    fork
        forever send_background_traffic();
    join_none
    // body 很快返回，但后台线程仍然存在。
endtask
```
可能后果：
- sequence 已结束，后台线程仍访问其成员。
- phase 结束时线程被强制杀死。
- 仍在等待 sequencer grant，造成难定位的挂起。
- 多次启动 sequence 后累积多个后台流量线程。
更可控的写法：
```systemverilog
virtual task body();
    bit stop_background;
    fork
        begin
            while (!stop_background)
                send_one_background_item();
        end
        begin
            run_foreground_flow();
            stop_background = 1'b1;
        end
    join
endtask
```
原则：
- 能 `join` 就明确 `join`。
- 使用 `join_any` 后明确处理剩余线程。
- 使用 `join_none` 时提供可验证的退出条件。
- 循环索引传入 fork 时声明 `automatic` 副本。

---

## 6.6 在 sequence 中使用 `config_db`
### 6.6.1 在 sequence 中获取参数
sequence 不是 component，因此没有稳定的 component 层次位置。
可借助 `m_sequencer` 作为上下文获取配置：
```systemverilog
virtual task body();
    int packet_count;
    if (!uvm_config_db #(int)::get(
            m_sequencer,                  // component 上下文
            "",                           // 从当前 sequencer 路径取值
            "packet_count",
            packet_count)) begin
        `uvm_fatal("CFG", "packet_count not found")
    end
    repeat (packet_count)
        `uvm_do(req)
endtask
```
test 中设置：
```systemverilog
uvm_config_db #(int)::set(
    this,
    "env.i_agt.sqr",
    "packet_count",
    100
);
```
推荐优先级：
1. sequence 自身字段直接赋值。
2. 配置对象传入 sequence。
3. 确需跨层共享时再使用 `config_db`。
直接字段更易追踪：
```systemverilog
seq.packet_count = cfg.packet_count;
seq.start(env.i_agt.sqr);
```

---

### 6.6.2 在 sequence 中设置参数
sequence 也可通过 sequencer 上下文设置配置：
```systemverilog
uvm_config_db #(bit)::set(
    m_sequencer,
    "uvm_test_top.env.scb",
    "traffic_started",
    1'b1
);
```
但 sequence 主动修改全局配置会产生隐式副作用。
更适合使用：
- `uvm_event` 通知。
- TLM transaction 通知。
- 共享配置对象中的受控状态。
- virtual sequence 直接调用明确接口。
路径必须根据上下文计算。
若 `cntxt=m_sequencer`，`inst_name` 并不天然从 `uvm_test_top` 开始。
调试时应打印：
```systemverilog
`uvm_info("CFG",
          $sformatf("context=%s", m_sequencer.get_full_name()),
          UVM_LOW)
```

---

### 6.6.3 `wait_modified` 的使用
`wait_modified` 等待某个配置项被重新设置。
```systemverilog
virtual task body();
    bit enable;
    forever begin
        // 首先读取当前值。
        void'(uvm_config_db #(bit)::get(
            m_sequencer, "", "enable_traffic", enable));
        if (enable)
            send_one_item();
        // 等待同一配置路径和字段发生 set 更新。
        uvm_config_db #(bit)::wait_modified(
            m_sequencer,
            "",
            "enable_traffic"
        );
    end
endtask
```
使用注意：
- 等待前先读取一次初值。
- 写入方必须对匹配路径执行 `set()`。
- 仿真结束前要有办法退出 forever。
- 高频运行时控制不宜依赖 config_db。
- `wait_modified` 表示“配置项被写”，不一定表示值真的变化。

---

## 6.7 response 的使用
### 6.7.1 `put_response` 与 `get_response`
若 sequence 需要 driver 返回执行结果，可使用 response 通道。
sequence：
```systemverilog
class read_sequence extends uvm_sequence #(bus_item);
    `uvm_object_utils(read_sequence)
    bus_item rsp;
    virtual task body();
        `uvm_do_with(req, {
            op == BUS_READ;
        })
        // 阻塞等待与本 sequence 对应的 response。
        get_response(rsp);
        `uvm_info("RSP",
                  $sformatf("read_data=0x%0h", rsp.rdata),
                  UVM_LOW)
    endtask
endclass
```
driver 产生独立 response：
```systemverilog
task bus_driver::main_phase(uvm_phase phase);
    bus_item rsp;
    forever begin
        seq_item_port.get_next_item(req);
        drive_request(req);
        rsp = bus_item::type_id::create("rsp");
        rsp.set_id_info(req);             // 复制 sequence/transaction 路由信息
        rsp.rdata  = sample_read_data();
        rsp.status = BUS_OK;
        // item_done(rsp) 在完成请求的同时返回应答。
        seq_item_port.item_done(rsp);
    end
endtask
```
也可先完成请求，再单独返回：
```systemverilog
seq_item_port.item_done();
seq_item_port.put_response(rsp);
```
`set_id_info(req)` 很重要。
它把 response 关联到原请求所属 sequence，sequencer 才能正确路由。

---

### 6.7.2 response 的数量问题
每个 sequence 内部有 response queue。
如果 driver 不断返回 response，而 sequence 不读取，队列会溢出。
默认深度通常较小，教材示例强调不能忽略此问题。
典型错误：
```systemverilog
repeat (100) begin
    `uvm_do(req)
    // driver 每次都返回 rsp，但这里从不 get_response。
end
```
修正：
```systemverilog
repeat (100) begin
    `uvm_do(req)
    get_response(rsp);                   // 一发一收
end
```
或将发送与接收并行：
```systemverilog
fork
    begin : send_thread
        repeat (100)
            `uvm_do(req)
    end
    begin : response_thread
        repeat (100) begin
            get_response(rsp);
            process_response(rsp);
        end
    end
join
```
也可调整队列深度，但这只是容量策略，不是替代消费逻辑。
```systemverilog
set_response_queue_depth(32);
```
把深度设为 `-1` 可能表示无限制，但会掩盖泄漏并增加内存占用。

---

### 6.7.3 response handler 与另类的 response
response handler 可让 response 到达时自动回调，而不必显式 `get_response()`。
```systemverilog
class async_response_sequence extends uvm_sequence #(bus_item);
    `uvm_object_utils(async_response_sequence)
    virtual task pre_body();
        // 默认关闭，必须显式开启。
        use_response_handler(1);
    endtask
    virtual function void response_handler(
        uvm_sequence_item response
    );
        bus_item typed_rsp;
        if (!$cast(typed_rsp, response)) begin
            `uvm_error("RSP", "unexpected response type")
            return;
        end
        process_response(typed_rsp);
    endfunction
    virtual task body();
        repeat (100)
            `uvm_do(req)
    endtask
endclass
```
特点：
- handler 是 function，不能阻塞。
- 重处理应转移到 FIFO 或后台 task。
- 开启 handler 后不要再按普通方式重复消费同一 response。
- 仍需处理 response 数量与 sequence 生命周期。

---

### 6.7.4 rsp 与 req 类型不同
sequence 模板支持独立的请求和应答类型：
```systemverilog
class read_sequence extends uvm_sequence #(
    bus_request,
    bus_response
);
    `uvm_object_utils(read_sequence)
    bus_response rsp;
    virtual task body();
        `uvm_do_with(req, {
            op == READ;
        })
        get_response(rsp);
    endtask
endclass
```
sequencer 与 driver 也要使用一致参数：
```systemverilog
class bus_sequencer extends uvm_sequencer #(
    bus_request,
    bus_response
);
class bus_driver extends uvm_driver #(
    bus_request,
    bus_response
);
```
独立 RSP 类型适合：
- 请求和返回字段差异很大。
- 读写共用 request，但 response 只包含状态和读数据。
- 希望从类型层面禁止误用请求字段。

---

## 6.8 sequence library
### 6.8.1 随机选择 sequence
sequence library 是多个 sequence 类型的集合，可随机挑选并执行。
定义 library：
```systemverilog
class traffic_sequence_library extends uvm_sequence_library #(
    my_transaction
);
    `uvm_object_utils(traffic_sequence_library)
    `uvm_sequence_library_utils(traffic_sequence_library)
    function new(string name = "traffic_sequence_library");
        super.new(name);
        init_sequence_library();          // 初始化已注册的 sequence 类型
    endfunction
endclass
```
把 sequence 加入 library：
```systemverilog
class short_sequence extends uvm_sequence #(my_transaction);
    `uvm_object_utils(short_sequence)
    `uvm_add_to_seq_lib(short_sequence, traffic_sequence_library)
    virtual task body();
        `uvm_do_with(req, {
            payload.size() inside {[46:128]};
        })
    endtask
endclass
```
启动 library：
```systemverilog
traffic_sequence_library lib;
lib = traffic_sequence_library::type_id::create("lib");
lib.start(env.i_agt.sqr);
```
适合场景：
- 从多个协议操作中随机选择。
- 快速构造 smoke/random 流量。
- 将已有原子 sequence 组成随机场景池。
不适合精确业务流程，因为随机选择难以表达严格依赖。

---

### 6.8.2 控制选择算法
常见选择模式：

| 模式 | 含义 |
|------|------|
| `UVM_SEQ_LIB_RAND` | 每次随机选择，可重复 |
| `UVM_SEQ_LIB_RANDC` | 随机循环，尽量每种执行后再重复 |
| `UVM_SEQ_LIB_ITEM` | 直接随机生成 item |
| `UVM_SEQ_LIB_USER` | 使用用户自定义选择算法 |
```systemverilog
lib.selection_mode = UVM_SEQ_LIB_RANDC;
```
RAND 与 RANDC：
- RAND 可能连续多次选中同一种 sequence。
- RANDC 更适合希望每轮覆盖所有原子场景的测试。
- RANDC 不代表每种 sequence 的功能覆盖率必然达到。
自定义算法可依据权重、覆盖率或系统状态选择下一个 sequence。
但覆盖率闭环通常还需要明确的 coverage feedback，不应只依赖随机 library。

---

### 6.8.3 控制执行次数
library 的执行次数可配置为一个随机范围：
```systemverilog
lib.min_random_count = 20;
lib.max_random_count = 50;
```
每次 library 启动时，在范围内决定要执行多少个 sequence。
建议：
- smoke test 使用较小范围。
- nightly regression 使用更大范围。
- 固定 seed 重现失败。
- 日志中记录 selection mode、执行次数和 seed。

---

### 6.8.4 使用 `sequence_library_cfg`
可将选择模式和次数集中放入配置对象。
概念示例：
```systemverilog
uvm_sequence_library_cfg cfg;
cfg = new("cfg");
cfg.selection_mode   = UVM_SEQ_LIB_RANDC;
cfg.min_random_count = 20;
cfg.max_random_count = 50;
lib.cfg = cfg;
lib.start(env.i_agt.sqr);
```
不同 UVM 版本对具体成员与赋值接口可能略有差异，应以项目使用版本为准。
配置对象的优点：
- test 可以统一控制 library 策略。
- sequence library 本体保持通用。
- smoke、full regression 可复用同一 library。
- 配置可打印并记录，便于重现。

---

## 6.9 sequence 握手全过程
### 6.9.1 请求路径
显式 sequence：
```systemverilog
start_item(req);
assert(req.randomize());
finish_item(req);
```
driver：
```systemverilog
seq_item_port.get_next_item(req);
drive_one_pkt(req);
seq_item_port.item_done();
```
时序关系：
```text
sequence                         sequencer                    driver
   |                                 |                           |
   | start_item(req)                 |                           |
   |------ wait for grant ---------->|                           |
   |<----------- grant --------------|                           |
   | randomize req                   |                           |
   | finish_item(req)                |                           |
   |------ send request ------------>|                           |
   |                                 |<-- get_next_item(req) -----|
   |                                 |------ req ---------------->|
   |                                 |                           | drive DUT
   |                                 |<----- item_done -----------|
   |<------ finish_item returns -----|                           |
```
`finish_item()` 返回只说明 item 的 driver 握手完成。
若 DUT 内部仍有流水线，测试不能据此立即结束。
### 6.9.2 `get_next_item` 与 `get`
常用 driver 取请求方式：
```systemverilog
seq_item_port.get_next_item(req);
drive(req);
seq_item_port.item_done();
```
或者：
```systemverilog
seq_item_port.get(req);
drive(req);
```
区别：

| 方式 | 完成通知 |
|------|----------|
| `get_next_item` | driver 必须调用 `item_done` |
| `get` | 取出时已完成 sequencer 侧请求握手，不再配对 `item_done` |
项目中应统一一种协议，不能把两套配对关系混用。
### 6.9.3 sequence 调试清单
当 driver 收不到 item：
1. 确认 sequence 的 `body()` 是否执行。
2. 确认 `start()` 传入的 sequencer 不是 null。
3. 确认 driver 的 `seq_item_port` 已连接 sequencer。
4. 确认 REQ 参数类型一致。
5. 确认 driver 进入了 `get_next_item()`。
6. 检查其他 sequence 是否持有 lock/grab。
7. 检查当前 sequence 的 `is_relevant()`。
8. 检查前一个 item 是否漏掉 `item_done()`。
9. 检查 phase 是否已经结束并杀死 sequence。
10. 打开 sequencer arbitration trace。
当 sequence 卡在 `finish_item()`：
- driver 可能没有运行。
- driver 可能没有调用 `item_done()`。
- driver 可能在等待永远不会到来的接口条件。
- sequencer-driver 连接可能错误。
- 请求类型转换可能失败。
当 response 错位：
- 检查 driver 是否调用 `rsp.set_id_info(req)`。
- 检查是否复用了正在被处理的 rsp 对象。
- 检查多个并发 sequence 是否都在消费自己的 response。
- 检查 response handler 和 `get_response()` 是否混用。
- 检查 RSP 模板类型是否一致。

---

## 本章总结（6.1-6.8）
### 学习重点排序
| 优先级 | 必须掌握 |
|--------|----------|
| 🔴 高 | sequence、sequencer、driver 的职责边界 |
| 🔴 高 | `start_item/finish_item` 与 `get_next_item/item_done` 握手 |
| 🔴 高 | `uvm_do`、`uvm_do_with`、`uvm_do_on` 的含义 |
| 🔴 高 | virtual sequence 协调多个 sequencer 的结构 |
| 🔴 高 | response 的路由、消费和队列问题 |
| 🟡 中 | 多 sequence 仲裁、优先级、lock 与 grab |
| 🟡 中 | 嵌套 sequence、sequence 随机化与继承 |
| 🟢 进阶 | `is_relevant/wait_for_relevant` 与 sequence library |
### 最重要的 10 条规则
| # | 规则 | 说明 |
|---|------|------|
| 1 | sequence 产生激励，driver 只负责驱动 | 不因测试场景修改稳定 driver |
| 2 | `start_item` 与 `finish_item` 必须成对 | 获得授权后尽快发送 |
| 3 | `get_next_item` 与 `item_done` 必须成对 | 漏掉会让后续 sequence 卡住 |
| 4 | 三处 REQ/RSP 模板类型必须匹配 | sequence、sequencer、driver 保持一致 |
| 5 | 复杂宏问题先展开成基础调用调试 | 明确 create、randomize、grant、send 顺序 |
| 6 | lock/grab 后必须释放 | 独占范围要短，不在持锁时长期等待 |
| 7 | virtual sequence 只协调，不直接驱动 DUT | 子 sequence 启动在真实 sequencer 上 |
| 8 | 顶层统一控制 objection | 子 sequence 保持可组合性 |
| 9 | driver 返回 rsp 时复制请求 ID | 使用 `set_id_info(req)` 保证路由 |
| 10 | 每个 response 都要被消费 | 防止 response queue 溢出 |
### 最容易错的点
| 易错点 | 正确理解 |
|--------|----------|
| sequence 是 component | 错，sequence 是 object，没有 component phase 层次 |
| `uvm_do` 只等于 randomize | 错，它还涉及创建、仲裁、发送和完成等待 |
| `finish_item` 返回表示 DUT 已处理完成 | 错，只表示 driver 已完成该 item 的握手 |
| 高优先级 sequence 会永久独占 | 错，是否优先取决于仲裁模式，独占要用 lock/grab |
| `grab` 可以中断正在驱动的 item | 错，只影响后续仲裁 |
| 只重载 `is_relevant` 就够了 | 通常错，应同时设计 `wait_for_relevant` |
| 子 sequence 都应该 raise objection | 错，通常由顶层 sequence 或 test 统一控制 |
| virtual sequencer 要连接 driver | 错，它主要保存真实 sequencer 句柄 |
| response 自动回到正确 sequence | 前提是正确保留 sequence ID 信息 |
| 把 response queue 设成无限大就解决问题 | 错，只是掩盖没有消费 response 的设计缺陷 |
### 一页复习表
| 需求 | 首选方法 |
|------|----------|
| 发送一个随机 item | `uvm_do(req)` |
| 给 item 增加内联约束 | `uvm_do_with(req, {...})` |
| 随机化后修改字段 | `uvm_create` + randomize + `uvm_send` |
| 精确观察握手 | `start_item` + randomize + `finish_item` |
| 组合已有场景 | 嵌套 sequence |
| 协调多个接口 | virtual sequence + virtual sequencer |
| 一组 item 不被插入 | `lock/unlock` |
| 紧急抢占后续仲裁 | `grab/ungrab` |
| 暂时退出仲裁 | `is_relevant/wait_for_relevant` |
| 获取执行结果 | response + `get_response` 或 handler |
