# UVM实战 第 4 章学习笔记：UVM 中的 TLM1.0 通信
> **核心结论**：TLM1.0 用 transaction 和标准端口替代 component 之间的直接变量访问。PORT 表示操作发起端，EXPORT 用于转发接口，IMP 是最终实现端；判断连接方向时看控制流，判断 transaction 方向时看数据流。
>
> **记忆主线**：先确定 put/get/transport -> 再确定 blocking/nonblocking -> 发起者使用 PORT -> 层次透传使用 EXPORT -> 最终处理者使用 IMP 或 FIFO。

---

## 4.0 本章定位
第 2 章已经使用 analysis_port、blocking_get_port 和 analysis FIFO，本章系统解释这些通信结构。

| 章节 | 核心问题 |
|------|----------|
| 4.1 | TLM、put/get/transport、PORT/EXPORT 是什么 |
| 4.2 | PORT、EXPORT、IMP 可以怎样连接 |
| 4.3 | analysis、多个 IMP 和 FIFO 应该怎样选择 |

学习本章时始终区分三件事：

1. 谁主动调用接口。
2. transaction 实际流向哪里。
3. 哪个 component 最终执行 put/get/write 等方法。

### 零基础阅读提示

把 TLM 想成组件之间传递“包裹”的标准接口：

- PORT 是主动发起调用的一端。
- EXPORT 是中间转接头。
- IMP 是真正处理调用的终点。
- put 表示“我把数据给你”，get 表示“我主动向你要数据”。
- analysis 表示广播，发送者不关心有几个订阅者。

判断 PORT 时看谁调用函数，不要看 transaction 最终向哪边流；这是本章最重要也最容易绕晕的一点。

---

## 4.1 TLM1.0
### 4.1.1 验证平台内部的通信
component 之间需要交换 transaction，例如 monitor 把采集结果传给 scoreboard。

#### 不推荐的通信方式

| 方式 | 问题 |
|------|------|
| 全局变量 | 任意代码都可修改，来源和时序不可控 |
| 直接访问 public 成员 | 发送者必须持有接收者句柄，耦合过强 |
| 共享 config object | 需要第三方分发，仍可能被其他组件误改 |
| 自建 mailbox/semaphore | 能实现，但每个项目都要重复编写协议 |

直接访问 scoreboard：
```systemverilog
class my_monitor extends uvm_monitor;
    my_scoreboard scb;            // monitor 必须了解 scoreboard 的具体类型

    task send(packet tr);
        scb.received_tr = tr;      // 可直接访问其他 public 字段，边界不清晰
    endtask
endclass
```
这种代码把 monitor 与 scoreboard 强绑定，scoreboard 类型或层次变化时 monitor 也要修改。

#### TLM 通道的价值
标准 TLM 通道只暴露约定操作：

- 发送者只能执行端口支持的操作。
- 接收者只需实现规定接口。
- transaction 类型在编译时检查。
- 支持 blocking/nonblocking。
- 支持层次转发、广播和 FIFO 缓冲。

```text
monitor --transaction/TLM--> scoreboard

monitor 不需要知道 scoreboard 内部队列、比较器和统计变量。
```

---

### 4.1.2 TLM 的定义
TLM 是 Transaction Level Modeling，即事务级建模。
它关注一笔完整 transaction，而不是 pin 级每个时钟和比特。

| 抽象层级 | 传递内容 | 典型位置 |
|----------|----------|----------|
| 信号级 | data、valid、ready 等信号 | driver/monitor 与 DUT |
| transaction 级 | packet、bus_item、reg_item | UVM component 之间 |

UVM 支持 TLM1.0 和 TLM2.0，本章讨论 TLM1.0。

#### put 操作
发起者 A 把 transaction 发送给目标 B。

```text
控制流：A(PORT) ------调用------> B(EXPORT/IMP)
数据流：A ----------------------> B
```

#### get 操作
发起者 A 主动向目标 B 索取 transaction。

```text
控制流：A(PORT) ------调用------> B(EXPORT/IMP)
数据流：A <---------------------- B
```

虽然数据从 B 流向 A，但发起调用的仍是 A，所以 A 使用 PORT。

#### transport 操作
发起者先发送 request，再取得 response。

```text
A(PORT) --request--> B(IMP)
A(PORT) <--response-- B(IMP)
```

transport 可理解为一次 request-response 操作。

#### PORT/EXPORT 描述控制流
PORT 和 EXPORT 的判断依据不是 transaction 方向，而是谁发起操作：

| 操作 | 发起者 | 发起端 | transaction 方向 |
|------|--------|--------|------------------|
| put | 调用 put 的一方 | PORT | PORT -> 目标 |
| get | 调用 get 的一方 | PORT | 目标 -> PORT |
| transport | 调用 transport 的一方 | PORT | 先去后回 |

---

### 4.1.3 UVM 中的 PORT 与 EXPORT
端口名称由“阻塞性质 + 操作 + 角色”组成。

```text
uvm_<blocking/nonblocking>_<put/get/...>_<port/export/imp>
```

#### 常用 PORT
```systemverilog
// put：调用者把 transaction 推给下游。
uvm_blocking_put_port       #(T);
uvm_nonblocking_put_port    #(T);
uvm_put_port                #(T);

// get/peek：调用者主动从对端取数据；peek 不消费队头。
uvm_blocking_get_port       #(T);
uvm_nonblocking_get_port    #(T);
uvm_get_port                #(T);
uvm_blocking_peek_port      #(T);
uvm_nonblocking_peek_port   #(T);
uvm_peek_port               #(T);
uvm_blocking_get_peek_port  #(T);
uvm_nonblocking_get_peek_port #(T);
uvm_get_peek_port           #(T);

// transport：一次调用同时包含 request 和 response。
uvm_blocking_transport_port #(REQ, RSP);
uvm_nonblocking_transport_port #(REQ, RSP);
uvm_transport_port          #(REQ, RSP);
```

#### 常用 EXPORT
EXPORT 与上述 PORT 一一对应，只需把后缀替换为 <code>_export</code>。
```systemverilog
uvm_blocking_put_export       #(T);
uvm_blocking_get_export       #(T);
uvm_blocking_transport_export #(REQ, RSP);
```

#### 端口类型的能力

| 名称 | 支持接口 |
|------|----------|
| blocking_put | put |
| nonblocking_put | try_put、can_put |
| put | put、try_put、can_put |
| blocking_get | get |
| nonblocking_get | try_get、can_get |
| get | get、try_get、can_get |
| blocking_peek | peek |
| nonblocking_peek | try_peek、can_peek |
| blocking_transport | transport |
| nonblocking_transport | nb_transport |

端口一旦选择，就只能执行它定义的操作。需要另一类操作时，应增加匹配端口。

---

## 4.2 UVM 中各种端口的互连
### 4.2.1 PORT 与 EXPORT 的连接
连接由高控制优先级一方调用：
```systemverilog
A_port.connect(B_export);        // 正确：PORT 主动连接 EXPORT
// B_export.connect(A_port);     // 错误方向
```

#### 声明和创建 PORT
```systemverilog
class producer extends uvm_component;
    uvm_blocking_put_port #(packet) out_port;

    `uvm_component_utils(producer)

    function void build_phase(uvm_phase phase);
        super.build_phase(phase);
        out_port = new("out_port", this); // TLM port 也要创建
    endfunction
endclass
```

#### 声明和创建 EXPORT
```systemverilog
class bridge extends uvm_component;
    uvm_blocking_put_export #(packet) in_export;

    `uvm_component_utils(bridge)

    function void build_phase(uvm_phase phase);
        super.build_phase(phase);
        in_export = new("in_export", this);
    endfunction
endclass
```

#### min_size 与 max_size
PORT 构造函数还可限定连接数量：
```systemverilog
out_port = new(
    "out_port",
    this,
    1,                            // 至少连接 1 个下游
    2                             // 最多连接 2 个下游
);
```

默认通常为 <code>min_size=1</code>、<code>max_size=1</code>。

#### PORT 与 EXPORT 不能成为完整链路终点
PORT/EXPORT 只负责调用或转发，不实现实际 put。

```text
PORT -> EXPORT -> ???
```

若没有最终实现端，UVM 会在连接检查阶段报告连接数量或接口错误。

---

### 4.2.2 UVM 中的 IMP
IMP 是 implementation port，是 TLM 调用链的最终实现端。

控制优先级：
```text
PORT > EXPORT > IMP
```

连接链必须最终到达 IMP：
```text
PORT -> IMP
PORT -> EXPORT -> IMP
PORT -> PORT -> EXPORT -> IMP
```

#### 常用 IMP
```systemverilog
// 第二个参数 IMP 是最终实现 put/get 等方法的 component 类型。
uvm_blocking_put_imp       #(T, IMP);
uvm_nonblocking_put_imp    #(T, IMP);
uvm_blocking_get_imp       #(T, IMP);
uvm_nonblocking_get_imp    #(T, IMP);
uvm_blocking_transport_imp #(REQ, RSP, IMP);
uvm_analysis_imp           #(T, IMP);
```

IMP 参数中的 <code>IMP</code> 是实现接口方法的 component 类型。

#### blocking put 的调用链
```text
A.port.put(tr)
 -> B.export.put(tr)
 -> B.imp.put(tr)
 -> B.put(tr)
```

真正处理 transaction 的是 component B 的 <code>put</code> 方法。

#### 接收 component
```systemverilog
class consumer extends uvm_component;
    uvm_blocking_put_imp #(packet, consumer) in_imp;

    `uvm_component_utils(consumer)

    function void build_phase(uvm_phase phase);
        super.build_phase(phase);
        in_imp = new("in_imp", this);
    endfunction

    task put(packet tr);
        `uvm_info("CONS", "received packet", UVM_LOW)
        tr.print();               // 实际处理逻辑由 component 实现
    endtask
endclass
```

没有定义匹配的 <code>put</code>，编译或展开 IMP 时会报找不到接口方法。

---

### 4.2.3 PORT 与 IMP 的连接
这是最直接的 TLM 连接。

```systemverilog
function void my_env::connect_phase(uvm_phase phase);
    super.connect_phase(phase);

    // 控制流从 producer 的 PORT 指向 consumer 的 IMP。
    // 连接完成后，prod.out_port.put(tr) 最终会调用 cons.put(tr)。
    prod.out_port.connect(cons.in_imp);
endfunction
```

producer：
```systemverilog
task producer::main_phase(uvm_phase phase);
    packet tr;

    repeat (10) begin
        tr = packet::type_id::create("tr");
        assert(tr.randomize());
        out_port.put(tr);         // 阻塞到 consumer.put 返回
    end
endtask
```

#### IMP 必须实现的方法

| 端口族 | component 需要实现 |
|--------|--------------------|
| blocking_put | put |
| nonblocking_put | try_put、can_put |
| put | put、try_put、can_put |
| blocking_get | get |
| nonblocking_get | try_get、can_get |
| get | get、try_get、can_get |
| blocking_peek | peek |
| nonblocking_peek | try_peek、can_peek |
| get_peek | get/try_get/can_get + peek/try_peek/can_peek |
| blocking_transport | transport |
| nonblocking_transport | nb_transport |
| transport | transport、nb_transport |

blocking 调用可以落到 task；nonblocking 操作必须由 function 实现，不能消耗仿真时间。

#### 完整的层次化 put 链路
下面把 PORT-PORT、PORT-EXPORT 和 EXPORT-IMP 串在一起。

最底层 producer：
```systemverilog
class packet_source extends uvm_component;
    uvm_blocking_put_port #(packet) tx_port;

    `uvm_component_utils(packet_source)

    function void build_phase(uvm_phase phase);
        super.build_phase(phase);
        tx_port = new("tx_port", this);
    endfunction

    task main_phase(uvm_phase phase);
        packet tr;

        repeat (8) begin
            tr = packet::type_id::create("tr");
            assert(tr.randomize());
            tx_port.put(tr);      // 调用沿整条链向下传播
        end
    endtask
endclass
```

父层 source_agent 向外暴露 PORT：
```systemverilog
class source_agent extends uvm_agent;
    packet_source src;
    uvm_blocking_put_port #(packet) tx_port;

    `uvm_component_utils(source_agent)

    function void build_phase(uvm_phase phase);
        super.build_phase(phase);
        src     = packet_source::type_id::create("src", this);
        tx_port = new("tx_port", this);
    endfunction

    function void connect_phase(uvm_phase phase);
        super.connect_phase(phase);
        src.tx_port.connect(this.tx_port); // child PORT -> parent PORT
    endfunction
endclass
```

接收层使用 EXPORT 向内部 IMP 转发：
```systemverilog
class packet_sink extends uvm_component;
    uvm_blocking_put_imp #(packet, packet_sink) rx_imp;
    packet received[$];

    `uvm_component_utils(packet_sink)

    function void build_phase(uvm_phase phase);
        super.build_phase(phase);
        rx_imp = new("rx_imp", this);
    endfunction

    task put(packet tr);
        packet copy_tr;

        $cast(copy_tr, tr.clone()); // 保存独立副本
        received.push_back(copy_tr);
    endtask
endclass

class sink_agent extends uvm_agent;
    packet_sink sink;
    uvm_blocking_put_export #(packet) rx_export;

    `uvm_component_utils(sink_agent)

    function void build_phase(uvm_phase phase);
        super.build_phase(phase);
        sink      = packet_sink::type_id::create("sink", this);
        rx_export = new("rx_export", this);
    endfunction

    function void connect_phase(uvm_phase phase);
        super.connect_phase(phase);
        rx_export.connect(sink.rx_imp); // parent EXPORT -> child IMP
    endfunction
endclass
```

env 完成中间连接：
```systemverilog
function void my_env::connect_phase(uvm_phase phase);
    super.connect_phase(phase);
    src_agt.tx_port.connect(sink_agt.rx_export);
endfunction
```

最终调用链：
```text
src.tx_port
 -> src_agt.tx_port
 -> sink_agt.rx_export
 -> sink.rx_imp
 -> sink.put(tr)
```

调试时从最终 <code>put</code> 反向检查，每一级的操作族和 transaction 类型必须一致。

---

### 4.2.4 EXPORT 与 IMP 的连接
EXPORT 也可直接连接到低优先级 IMP：
```systemverilog
function void bridge::connect_phase(uvm_phase phase);
    super.connect_phase(phase);
    in_export.connect(local_imp);
endfunction
```

EXPORT 可作为局部链路的起点，也常作为父层暴露子组件接口的中间端口。

```text
外部 PORT -> bridge.EXPORT -> bridge/local IMP
```

匹配规则与 PORT-IMP 相同：blocking_put_export 最终仍要求实现 <code>put</code>。

---

### 4.2.5 PORT 与 PORT 的连接
PORT-PORT 用于把子组件的发起端口向父层透传。

```text
child.C_port -> parent.A_port -> target.IMP
```

父 component 内：
```systemverilog
function void parent_comp::connect_phase(uvm_phase phase);
    super.connect_phase(phase);

    child.C_port.connect(this.A_port); // 子 PORT 向父 PORT 连接
endfunction
```

外层 env：
```systemverilog
A_inst.A_port.connect(B_inst.B_imp);
```

PORT-PORT 可以跨多层，但最终仍需到达匹配 IMP。

---

### 4.2.6 EXPORT 与 EXPORT 的连接
EXPORT-EXPORT 用于把父层的被调用接口向内部子组件透传。

```text
source.PORT -> parent.C_export -> child.B_export -> child.IMP
```

```systemverilog
function void parent_comp::connect_phase(uvm_phase phase);
    super.connect_phase(phase);
    this.C_export.connect(child.B_export);
endfunction
```

外层：
```systemverilog
source.out_port.connect(parent.C_export);
```

#### 层次连接方向速记

| 同类型连接 | 典型方向 |
|------------|----------|
| PORT -> PORT | child PORT 连接 parent PORT |
| EXPORT -> EXPORT | parent EXPORT 连接 child EXPORT |

不要只看图形上下位置，最终以“谁调用 connect、谁是被调用接口”判断。

---

### 4.2.7 blocking_get 端口的使用
get 中，调用者主动索取数据，因此调用者使用 PORT。

```text
控制流：consumer.PORT ------> producer.IMP
数据流：consumer <----------- producer
```

#### 数据提供者
```systemverilog
class producer extends uvm_component;
    uvm_blocking_get_imp #(packet, producer) get_imp;
    packet queue[$];

    `uvm_component_utils(producer)

    function void build_phase(uvm_phase phase);
        super.build_phase(phase);
        get_imp = new("get_imp", this);
    endfunction

    task get(output packet tr);
        wait (queue.size() != 0); // blocking get 可以等待
        tr = queue.pop_front();
    endtask
endclass
```

#### 数据使用者
```systemverilog
class consumer extends uvm_component;
    uvm_blocking_get_port #(packet) get_port;

    task main_phase(uvm_phase phase);
        packet tr;
        forever begin
            get_port.get(tr);     // 主动请求下一笔 transaction
            tr.print();
        end
    endtask
endclass
```

#### 连接
```systemverilog
cons.get_port.connect(prod.get_imp);
```

get 的 PORT 在数据接收方，进一步证明 PORT/EXPORT 描述控制流，而不是数据流。

---

### 4.2.8 blocking_transport 端口的使用
transport 一次调用同时传入 request 和返回 response。

#### 发起者
```systemverilog
uvm_blocking_transport_port #(packet, packet_rsp) trans_port;

task requester::main_phase(uvm_phase phase);
    packet req;
    packet_rsp rsp;

    req = packet::type_id::create("req");
    assert(req.randomize());

    trans_port.transport(req, rsp); // 阻塞等待响应生成
    rsp.print();
endtask
```

#### 目标 component
```systemverilog
uvm_blocking_transport_imp #(packet, packet_rsp, responder) trans_imp;

task responder::transport(
    packet req,
    output packet_rsp rsp
);
    `uvm_info("RSP", "received request", UVM_LOW)

    rsp = packet_rsp::type_id::create("rsp");
    rsp.status = check_request(req); // 根据 request 形成 response
endtask
```

#### 连接
```systemverilog
req_inst.trans_port.connect(rsp_inst.trans_imp);
```

transport 适合严格的一问一答；持续流水、乱序响应常需要更解耦的结构。

---

### 4.2.9 nonblocking 端口的使用
nonblocking 操作必须立即返回，因此接口是 function。

#### try_put 与 can_put
```systemverilog
class producer extends uvm_component;
    uvm_nonblocking_put_port #(packet) put_port;

    task main_phase(uvm_phase phase);
        packet tr;

        repeat (10) begin
            tr = packet::type_id::create("tr");
            assert(tr.randomize());

            while (!put_port.can_put())
                #10;             // 调用者自行决定重试策略

            if (!put_port.try_put(tr))
                `uvm_warning("PUT", "try_put failed after can_put")
        end
    endtask
endclass
```

#### IMP 实现
```systemverilog
class consumer extends uvm_component;
    uvm_nonblocking_put_imp #(packet, consumer) put_imp;
    packet queue[$];

    function bit can_put();
        return queue.size() < 4;  // 当前是否还有缓存空间
    endfunction

    function bit try_put(packet tr);
        if (!can_put())
            return 0;

        queue.push_back(tr);
        return 1;
    endfunction
endclass
```

也可直接循环尝试：
```systemverilog
while (!put_port.try_put(tr))
    #10;
```

#### blocking 与 nonblocking

| 特性 | blocking | nonblocking |
|------|----------|-------------|
| 接口实现 | task 或兼容函数 | function |
| 可消耗时间 | 可以 | 不可以 |
| 无资源时 | 调用阻塞等待 | 立即返回失败 |
| 重试策略 | 接收方/通道内部 | 调用方负责 |

#### nonblocking get 模板
get 发起者主动请求数据，但调用必须立即返回：
```systemverilog
class requester extends uvm_component;
    uvm_nonblocking_get_port #(packet) get_port;

    task main_phase(uvm_phase phase);
        packet tr;

        forever begin
            if (get_port.can_get()) begin
                if (get_port.try_get(tr))
                    consume(tr);          // 成功取得并消费 transaction
            end
            else begin
                @(posedge vif.clk);       // 当前无数据，调用者自行等待
            end
        end
    endtask
endclass
```
提供者必须实现：
```systemverilog
function bit can_get();
    return queue.size() != 0;
endfunction

function bit try_get(output packet tr);
    if (!can_get())
        return 0;

    tr = queue.pop_front();
    return 1;
endfunction
```

#### get 与 peek 的实现对照
```systemverilog
task get(output packet tr);
    wait (queue.size() != 0);
    tr = queue.pop_front();       // 取出并删除队头
endtask

task peek(output packet tr);
    wait (queue.size() != 0);
    tr = queue[0];                // 只查看，队头仍留在队列中
endtask
```
peek 返回对象句柄；若调用者会修改对象，应先 clone，避免间接改变 FIFO 中的数据。

#### 连接合法性速查

| 起点 | 可连接到 | 能否成为最终终点 |
|------|----------|------------------|
| PORT | PORT、EXPORT、IMP | 否 |
| EXPORT | EXPORT、IMP | 否 |
| IMP | 不向下发起连接 | 是 |

连接失败时依次检查：

1. 操作族是否一致：put/get/peek/transport。
2. blocking 属性是否一致。
3. transaction 参数类型是否一致。
4. connect 方向是否由高控制优先级端发起。
5. 整条链是否最终落到 IMP。
6. PORT 的 min_size/max_size 是否满足。

---

## 4.3 UVM 中的通信方式
### 4.3.1 UVM 中的 analysis 端口
analysis_port 用于发布/订阅式广播。

特点：

- 一对多连接。
- 只有 <code>write</code> 操作。
- write 是 function。
- 发布者不等待订阅者响应。
- 一个 transaction 可同时送到 scoreboard、coverage collector 等。

#### 发布者
```systemverilog
class my_monitor extends uvm_monitor;
    uvm_analysis_port #(packet) ap;

    `uvm_component_utils(my_monitor)

    function void build_phase(uvm_phase phase);
        super.build_phase(phase);
        ap = new("ap", this);
    endfunction

    task main_phase(uvm_phase phase);
        packet tr;
        forever begin
            tr = collect_packet();
            ap.write(tr);         // 广播给所有已连接订阅者
        end
    endtask
endclass
```

#### 订阅者
```systemverilog
class coverage_collector extends uvm_component;
    uvm_analysis_imp #(packet, coverage_collector) imp;

    function void build_phase(uvm_phase phase);
        super.build_phase(phase);
        imp = new("imp", this);
    endfunction

    function void write(packet tr);
        sample_packet(tr);        // write 不能阻塞
    endfunction
endclass
```

#### 一对多连接
```systemverilog
mon.ap.connect(scb.input_imp);
mon.ap.connect(cov.imp);
```

每个 analysis 链仍必须最终到达 analysis_imp、subscriber 或 FIFO 中的实现端。

#### 句柄共享风险
analysis_port 向多个订阅者传递的是对象句柄。

若某订阅者修改 transaction，其他订阅者可能看到修改后的对象。

建议：

- monitor 发布后不要继续修改对象。
- 需要修改时先 clone/copy。
- 订阅者默认把输入 transaction 当作只读数据。

---

### 4.3.2 一个 component 内有多个 IMP
scoreboard 常同时接收 model 的 expected 和 monitor 的 actual。

两个普通 analysis_imp 都会要求名为 <code>write</code> 的函数，无法区分来源。

#### 声明不同 IMP 后缀
```systemverilog
`uvm_analysis_imp_decl(_expected)
`uvm_analysis_imp_decl(_actual)

class my_scoreboard extends uvm_scoreboard;
    uvm_analysis_imp_expected #(packet, my_scoreboard) exp_imp;
    uvm_analysis_imp_actual   #(packet, my_scoreboard) act_imp;

    packet expected_q[$];

    `uvm_component_utils(my_scoreboard)

    function void build_phase(uvm_phase phase);
        super.build_phase(phase);
        exp_imp = new("exp_imp", this);
        act_imp = new("act_imp", this);
    endfunction

    function void write_expected(packet tr);
        expected_q.push_back(tr); // model 输出进入期望队列
    endfunction

    function void write_actual(packet tr);
        packet expected;

        if (expected_q.size() == 0) begin
            `uvm_error("SCB", "actual arrived before expected")
            return;
        end

        expected = expected_q.pop_front();
        if (!tr.compare(expected))
            `uvm_error("SCB", "packet mismatch")
    endfunction
endclass
```

宏后缀与方法后缀必须一致：

```text
_expected -> write_expected
_actual   -> write_actual
```

#### 双 analysis IMP 的完整连接
model 和 monitor 分别发布期望值、实际值：
```systemverilog
class my_model extends uvm_component;
    uvm_analysis_port #(packet) exp_ap;
    // 计算完成后调用 exp_ap.write(expected)
endclass

class my_monitor extends uvm_monitor;
    uvm_analysis_port #(packet) act_ap;
    // 采集完成后调用 act_ap.write(actual)
endclass
```

scoreboard 接收时复制句柄并按来源处理：
```systemverilog
`uvm_analysis_imp_decl(_expected)
`uvm_analysis_imp_decl(_actual)

class my_scoreboard extends uvm_scoreboard;
    uvm_analysis_imp_expected #(packet, my_scoreboard) exp_imp;
    uvm_analysis_imp_actual   #(packet, my_scoreboard) act_imp;
    packet expected_q[$];
    int compare_count;

    `uvm_component_utils(my_scoreboard)

    function void build_phase(uvm_phase phase);
        super.build_phase(phase);
        exp_imp = new("exp_imp", this);
        act_imp = new("act_imp", this);
    endfunction

    function void write_expected(packet tr);
        packet copy_tr;

        $cast(copy_tr, tr.clone()); // analysis 广播传句柄，入队前复制
        expected_q.push_back(copy_tr);
    endfunction

    function void write_actual(packet tr);
        packet expected;

        if (expected_q.size() == 0) begin
            `uvm_error("SCB", "unexpected actual packet")
            return;
        end

        expected = expected_q.pop_front();
        compare_count++;

        if (!tr.compare(expected)) begin
            `uvm_error("SCB", "actual packet differs from expected")
            expected.print();
            tr.print();
        end
    endfunction
endclass
```

env 连接两路：
```systemverilog
function void my_env::connect_phase(uvm_phase phase);
    super.connect_phase(phase);
    mdl.exp_ap.connect(scb.exp_imp);
    o_agt.ap.connect(scb.act_imp);
endfunction
```

这种结构要求两个 <code>write_xxx</code> 足够轻量，因为 analysis write 是 function，不能阻塞等待另一条输入。

#### agent 暴露 monitor 的 analysis_port
推荐让 agent 对外提供协议观察端口：
```systemverilog
function void my_agent::connect_phase(uvm_phase phase);
    super.connect_phase(phase);
    ap = mon.ap;                  // 复用 monitor 的端口句柄
endfunction
```

env 只需连接 <code>o_agt.ap</code>，不必深入访问 <code>o_agt.mon.ap</code>。

---

### 4.3.3 使用 FIFO 通信
analysis FIFO 把非阻塞广播转换为可阻塞拉取。

```text
monitor.analysis_port
        |
        v
FIFO.analysis_export -> FIFO buffer -> FIFO.blocking_get_export
                                             |
                                             v
                                scoreboard.blocking_get_port
```

#### env 声明 FIFO
```systemverilog
uvm_tlm_analysis_fifo #(packet) exp_fifo;
uvm_tlm_analysis_fifo #(packet) act_fifo;

function void my_env::build_phase(uvm_phase phase);
    super.build_phase(phase);

    exp_fifo = new("exp_fifo", this);
    act_fifo = new("act_fifo", this);
endfunction
```

#### scoreboard 声明 get port
```systemverilog
class my_scoreboard extends uvm_scoreboard;
    uvm_blocking_get_port #(packet) exp_port;
    uvm_blocking_get_port #(packet) act_port;

    function void build_phase(uvm_phase phase);
        super.build_phase(phase);
        exp_port = new("exp_port", this);
        act_port = new("act_port", this);
    endfunction
endclass
```

#### env 连接
```systemverilog
function void my_env::connect_phase(uvm_phase phase);
    super.connect_phase(phase);

    mdl.ap.connect(exp_fifo.analysis_export);
    scb.exp_port.connect(exp_fifo.blocking_get_export);

    o_agt.ap.connect(act_fifo.analysis_export);
    scb.act_port.connect(act_fifo.blocking_get_export);
endfunction
```

#### scoreboard 主动获取
```systemverilog
task my_scoreboard::main_phase(uvm_phase phase);
    packet expected;
    packet actual;

    forever begin
        exp_port.get(expected);   // 可按 scoreboard 节奏阻塞等待
        act_port.get(actual);

        if (!actual.compare(expected))
            `uvm_error("SCB", "compare failed")
    end
endtask
```

FIFO 的优势：

- 隐藏 IMP 细节。
- 允许发送者和接收者以不同节奏运行。
- scoreboard 可主动 get。
- 多路输入更易组成端口/FIFO 数组。

#### 更稳健的双 FIFO scoreboard
若 expected 和 actual 到达时间差异较大，可分别接收后再比较：
```systemverilog
class my_scoreboard extends uvm_scoreboard;
    uvm_blocking_get_port #(packet) exp_port;
    uvm_blocking_get_port #(packet) act_port;

    packet expected_q[$];
    packet actual_q[$];
    int matched_count;

    `uvm_component_utils(my_scoreboard)

    task main_phase(uvm_phase phase);
        fork
            collect_expected();
            collect_actual();
            compare_queues();
        join
    endtask

    task collect_expected();
        packet tr;

        forever begin
            exp_port.get(tr);      // 只负责接收 expected
            expected_q.push_back(tr);
        end
    endtask

    task collect_actual();
        packet tr;

        forever begin
            act_port.get(tr);      // 只负责接收 actual
            actual_q.push_back(tr);
        end
    endtask

    task compare_queues();
        packet expected;
        packet actual;

        forever begin
            wait (expected_q.size() != 0);
            wait (actual_q.size()   != 0);

            expected = expected_q.pop_front();
            actual   = actual_q.pop_front();
            matched_count++;

            if (!actual.compare(expected))
                `uvm_error(
                    "SCB",
                    $sformatf("mismatch at packet %0d", matched_count)
                )
        end
    endtask
endclass
```

仿真结束时检查残留：
```systemverilog
function void my_scoreboard::check_phase(uvm_phase phase);
    super.check_phase(phase);

    if (expected_q.size() != 0)
        `uvm_error(
            "SCB",
            $sformatf("%0d expected packets remain", expected_q.size())
        )

    if (actual_q.size() != 0)
        `uvm_error(
            "SCB",
            $sformatf("%0d actual packets remain", actual_q.size())
        )
endfunction
```

若 DUT 支持乱序输出，不能简单按队头比较，应按 transaction ID、tag 或地址建立关联数组匹配。

---

### 4.3.4 FIFO 上的端口及调试
<code>uvm_tlm_fifo</code> 提供 put/get/peek 等多组接口。

<code>uvm_tlm_analysis_fifo</code> 在此基础上增加 analysis_export 和 write 接口。

#### get 与 peek

| 操作 | 返回队头 transaction | 从 FIFO 删除 |
|------|----------------------|--------------|
| get | 是 | 是 |
| peek | 是 | 否 |

peek 适合预览下一笔数据，真正消费仍需 get。

#### put_ap 与 get_ap
FIFO 收到 put 时，还会通过 <code>put_ap</code> 发布该 transaction。

FIFO 完成 get 时，会通过 <code>get_ap</code> 发布被取走的 transaction。

```text
put -> FIFO buffer + put_ap.write(tr)
get -> pop FIFO buffer + get_ap.write(tr)
```

可用它们做流量监控或覆盖率收集。

#### FIFO 调试函数

| 函数 | 作用 |
|------|------|
| <code>used()</code> | 当前缓存 transaction 数 |
| <code>is_empty()</code> | FIFO 是否为空 |
| <code>is_full()</code> | FIFO 是否已满 |
| <code>size()</code> | FIFO 容量上限 |
| <code>flush()</code> | 清空缓存 |

#### FIFO 容量
```systemverilog
fifo = new(
    "fifo",
    this,
    16                            // 最多缓存 16 笔 transaction
);
```

容量为 0 表示不限制大小。

#### 复位时清空
```systemverilog
function void my_env::flush_fifos();
    exp_fifo.flush();
    act_fifo.flush();
endfunction
```

清空前要确认阻塞 get、scoreboard 队列和 DUT 流水状态，避免只清一半造成错位。

---

### 4.3.5 用 FIFO 还是用 IMP

| 对比 | 直接 IMP | FIFO |
|------|----------|------|
| 代码路径 | 短，write 立即进入接收者 | 中间增加缓存 |
| 接收方式 | 被动回调 | 主动 get |
| 节奏解耦 | 弱 | 强 |
| 多输入 | 需要多个 IMP 后缀 | 可使用 FIFO/port 数组 |
| 延迟 | 基本无缓冲延迟 | 由队列解耦 |
| 调试 | 连接直接 | 可查看 used/empty/full |
| 适合场景 | coverage、简单订阅 | scoreboard、速率不匹配 |

#### 直接 IMP 更适合

- 简单 analysis subscriber。
- 功能覆盖率采集。
- write 中只做轻量、无阻塞处理。
- 输入数量少且固定。

#### FIFO 更适合

- scoreboard 需要主动同步多路数据。
- 接收处理可能晚于发送。
- 多通道可用数组和循环构建。
- 需要缓存深度和 flush 调试能力。

#### 多端口数组
```systemverilog
uvm_blocking_get_port #(packet) exp_port[16];
uvm_tlm_analysis_fifo #(packet) exp_fifo[16];

function void build_phase(uvm_phase phase);
    super.build_phase(phase);

    foreach (exp_port[i]) begin
        exp_port[i] = new($sformatf("exp_port_%0d", i), this);
        exp_fifo[i] = new($sformatf("exp_fifo_%0d", i), this);
    end
endfunction
```

连接：
```systemverilog
foreach (exp_port[i]) begin
    mdl.ap[i].connect(exp_fifo[i].analysis_export);
    exp_port[i].connect(exp_fifo[i].blocking_get_export);
end
```

并发读取时循环变量要 automatic：
```systemverilog
foreach (exp_port[i]) begin
    fork
        automatic int channel = i;
        forever begin
            packet tr;
            exp_port[channel].get(tr);
            process_expected(channel, tr);
        end
    join_none
end
```

#### TLM 连接故障定位表

| 现象 | 常见原因 | 检查方法 |
|------|----------|----------|
| connection count 为 0 | 端口没有 connect 或链路没到 IMP | 打印 topology，逐级检查 |
| No field named put/get/write | IMP 所在 component 未实现规定方法 | 对照操作族补接口 |
| 类型不兼容 | transaction 参数或 blocking 属性不一致 | 对照完整类型声明 |
| write 后数据错乱 | 多订阅者共享同一对象句柄 | 发布后只读，入队前 clone |
| blocking get 永久等待 | 上游未写 FIFO 或连接方向错误 | 查看 FIFO.used 与 write 日志 |
| FIFO 持续增长 | consumer 速度不足或没有 get | 监控 used/is_full |
| 最后一批数据残留 | phase 提前结束或 drain 不足 | check_phase 检查队列 |
| 多通道串线 | 数组索引或 automatic 变量错误 | 打印 channel 与 transaction ID |

推荐调试顺序：

1. 用 <code>uvm_top.print_topology()</code> 确认 component 层次。
2. 确认每个 TLM 端口已在 build_phase 创建。
3. 从发起 PORT 沿 connect 链追到最终 IMP。
4. 核对 put/get/analysis 操作族。
5. 核对 transaction 参数类型。
6. 在最终 put/get/write 中打印调用日志。
7. FIFO 结构检查 used、empty、full。
8. 在 check_phase 检查所有缓存和期望队列。

设计评审时再确认：

- 通信究竟需要推送、拉取还是请求-响应？
- 接收者是否需要缓存和节奏解耦？
- analysis 订阅者会不会修改共享 transaction？
- FIFO 容量、复位 flush 和结束残留是否有明确策略？
- 多通道是否携带可定位的 channel/transaction ID？
- 所有阻塞调用是否都有机会解除阻塞？

---

## 本章总结（4.1-4.3）
### 学习重点排序

| 优先级 | 必须掌握 |
|--------|----------|
| 🔴 高 | PORT 表示发起者，EXPORT 转发，IMP 最终实现 |
| 🔴 高 | put/get 的控制流与数据流方向不同 |
| 🔴 高 | 所有连接最终必须落到 IMP 或含 IMP 的 FIFO |
| 🔴 高 | analysis_port 的一对多 write 广播 |
| 🟡 中 | blocking/nonblocking 接口方法的区别 |
| 🟡 中 | FIFO 对 analysis 与 blocking_get 的桥接 |
| 🟢 进阶 | 多 IMP 后缀、层次透传和端口数组 |

### 最重要的 10 条规则

| # | 规则 | 说明 |
|---|------|------|
| 1 | 先看谁发起调用，再选 PORT | 不按 transaction 方向判断 PORT |
| 2 | PORT/EXPORT 只转发接口 | 最终处理必须由 IMP/component 实现 |
| 3 | 高优先级端调用 connect | PORT -> EXPORT -> IMP |
| 4 | 端口类型必须完整匹配 | 操作、阻塞属性和 transaction 类型都要一致 |
| 5 | TLM 端口在 build_phase 创建 | connect_phase 只负责连接 |
| 6 | nonblocking 方法必须是 function | 不能消耗仿真时间 |
| 7 | analysis write 不能阻塞 | 重处理应放入 FIFO 或后续进程 |
| 8 | 多个 analysis_imp 使用 imp_decl 后缀 | 后缀决定 write_xxx 方法名 |
| 9 | scoreboard 多路同步优先考虑 FIFO | 接收者可主动控制节奏 |
| 10 | analysis 订阅者默认只读 transaction | 修改前先 clone/copy |

### 最容易错的点

| 易错点 | 正确理解 |
|--------|----------|
| get 的数据流向 PORT，所以提供者应使用 PORT | 错，发起 get 的接收者使用 PORT |
| PORT 可以作为连接终点 | 错，最终必须到 IMP |
| EXPORT 自己实现 put/get | 错，EXPORT 主要负责透传 |
| blocking_put 与 nonblocking_put 可直接互连 | 错，接口类型必须匹配 |
| can_put 成功后 try_put 必然永远成功 | 不一定，状态可能变化，应检查返回值 |
| analysis_port.write 会等待订阅者 | 错，它是 function 式广播 |
| FIFO 的 analysis_export 一定是普通 EXPORT | 名字如此，但内部实现端本质包含 IMP |
| peek 会消费队头数据 | 错，peek 保留 FIFO 内容 |
| 多路 IMP 可以都实现同一个 write | 需要 imp_decl 后缀区分来源 |
| FIFO 一定优于 IMP | 两者适用于不同复杂度和同步需求 |
