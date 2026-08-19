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

## 4.1 TLM1.0（🔴 高）
### 4.1.1 验证平台内部的通信（🔴 高）
component 之间需要交换 transaction，例如 monitor 把采集结果传给 scoreboard。

**不推荐的通信方式**

| 方式 | 问题 |
|------|------|
| 全局变量 | 任意代码都可修改，来源和时序不可控 |
| 直接访问 public 成员 | 发送者必须持有接收者句柄，耦合过强 |
| 共享 config object | 需要第三方分发，仍可能被其他组件误改 |
| 自建 mailbox/semaphore | 能实现，但每个项目都要重复编写协议 |

直接访问 scoreboard：
```systemverilog
class my_monitor extends uvm_monitor;
    my_scoreboard scb;            // [1] monitor 握着 scoreboard 的句柄
    task send(packet tr);
        scb.received_tr = tr;      // [2] 直接塞进别人的 public 字段
    endtask
endclass
```
两个问题：

1. **类型耦合**：`my_scoreboard scb`——monitor 必须知道 scoreboard 的**具体类名**。明天 scoreboard 改名、换实现（比如换成继承类），monitor 要跟着改；
2. **边界不清**：`scb.received_tr = tr`——monitor 直接写别人的字段，**scoreboard 内部怎么存数据、怎么处理，被 monitor 越权干涉了**。边界一破，出 bug 时责任说不清。

**TLM 通道的价值**
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

### 4.1.2 TLM 的定义（🔴 高）
TLM 是 Transaction Level Modeling，即事务级建模。
它关注一笔完整 transaction，而不是 pin 级每个时钟和比特。

| 抽象层级 | 传递内容 | 典型位置 |
|----------|----------|----------|
| 信号级 | data、valid、ready 等信号 | driver/monitor 与 DUT |
| transaction 级 | packet、bus_item、reg_item | UVM component 之间 |

UVM 支持 TLM1.0 和 TLM2.0，本章讨论 TLM1.0。

**put 操作**
发起者 A 把 transaction 发送给目标 B。

```text
控制流：A(PORT) ------调用------> B(EXPORT/IMP)
数据流：A ----------------------> B
```

**get 操作**
发起者 A 主动向目标 B 索取 transaction。

```text
控制流：A(PORT) ------调用------> B(EXPORT/IMP)
数据流：A <---------------------- B
```

虽然数据从 B 流向 A，但发起调用的仍是 A，所以 A 使用 PORT。

**transport 操作**
发起者先发送 request，再取得 response。

```text
A(PORT) --request--> B(IMP)
A(PORT) <--response-- B(IMP)
```

transport 可理解为一次 request-response 操作。

**PORT/EXPORT 描述控制流**
PORT 和 EXPORT 的判断依据不是 transaction 方向，而是谁发起操作：

| 操作 | 发起者 | 发起端 | transaction 方向 |
|------|--------|--------|------------------|
| put | 调用 put 的一方 | PORT | PORT -> 目标 |
| get | 调用 get 的一方 | PORT | 目标 -> PORT |
| transport | 调用 transport 的一方 | PORT | 先去后回 |

---

### 4.1.3 UVM 中的 PORT 与 EXPORT（🔴 高）
端口名称由“阻塞性质 + 操作 + 角色”组成。

```text
uvm_<blocking/nonblocking>_<put/get/...>_<port/export/imp>
```

**常用 PORT**
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

**常用 EXPORT**
EXPORT 与上述 PORT 一一对应，只需把后缀替换为 <code>_export</code>。
```systemverilog
uvm_blocking_put_export       #(T);
uvm_blocking_get_export       #(T);
uvm_blocking_transport_export #(REQ, RSP);
```

**端口类型的能力**

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

## 4.2 UVM 中各种端口的互连（🟡 中）
### 4.2.1 PORT 与 EXPORT 的连接（🟡 中）
连接由高控制优先级一方调用：
```systemverilog
A_port.connect(B_export);        // [1] 正确：PORT 主动连 EXPORT
// B_export.connect(A_port);     // [2] 错误方向
```

**连接由"高控制优先级"一方发起**——PORT（发起端）主动去 connect 后面的 EXPORT/IMP。方向反了会报错。

**端口也要 new 和创建（和组件一样）**

**TLM 端口不是自动存在的，要在 build_phase 里 new**（带名字 + parent）：

```systemverilog
class producer extends uvm_component;
    uvm_blocking_put_port #(packet) out_port;  // [1] 声明

    `uvm_component_utils(producer)

    function void build_phase(uvm_phase phase);
        super.build_phase(phase);
        out_port = new("out_port", this);  // [2] 创建！端口也是组件树的一部分
    endfunction
endclass
```

EXPORT 同理（`uvm_blocking_put_export #(packet) in_export;` + `new("in_export", this)`）。**忘 new = null 引用，连接时崩**。

**min_size 与 max_size**

PORT 构造函数还可限定连接数量：
```systemverilog
out_port = new(
    "out_port",
    this,
    1,                            // min：至少连接 1 个下游
    2                             // max：最多连接 2 个下游
);
```

默认通常为 <code>min_size=1</code>、<code>max_size=1</code>。

**PORT 与 EXPORT 不能成为完整链路终点**
PORT/EXPORT 只负责调用或转发，不实现实际 put。

```text
PORT -> EXPORT -> ???   ← 断在这里不行
```

PORT 只负责**发起调用**、EXPORT 只负责**转发**，它们都**不实现实际方法**（put 的代码不在这里）。链路的**终点必须是 IMP（或 FIFO）**：

```
PORT -> EXPORT -> IMP   ✅ 完整
PORT -> EXPORT          ❌ 断了，没人实现 put
```

---

### 4.2.2 UVM 中的 IMP（🟡 中）
IMP 是 implementation port，是 TLM 调用链的最终实现端。

控制优先级：
```text
PORT（发起） > EXPORT（转发） > IMP（实现）
```

连接链必须最终到达 IMP：
```text
PORT -> IMP                    ✅ 直连
PORT -> EXPORT -> IMP          ✅ 中间透传
PORT -> PORT -> EXPORT -> IMP  ✅ 多层转发
```

**常用 IMP**
```systemverilog
// 第二个参数 IMP 是最终实现 put/get 等方法的 component 类型。
uvm_blocking_put_imp       #(T, IMP);
//                           ↑   ↑
//                      数据类型  实现类
uvm_nonblocking_put_imp    #(T, IMP);
uvm_blocking_get_imp       #(T, IMP);
uvm_nonblocking_get_imp    #(T, IMP);
uvm_blocking_transport_imp #(REQ, RSP, IMP);
uvm_analysis_imp           #(T, IMP);
```

IMP 参数中的 <code>IMP</code> 是实现接口方法的 component 类型。

**blocking put 的调用链**
```text
A.port.put(tr)          // A 发起
 -> B.export.put(tr)    // 转发
 -> B.imp.put(tr)       // 到 IMP
 -> B.put(tr)           // 真正执行的是 B 的 put 方法！
```

真正处理 transaction 的是 component B 的 <code>put</code> 方法——IMP 只是"入口"，具体逻辑在 B 里。

**接收 component**
```systemverilog
class consumer extends uvm_component;
    uvm_blocking_put_imp #(packet, consumer) in_imp;  // [1] 声明 IMP，指明实现者是 consumer

    function void build_phase(uvm_phase phase);
        super.build_phase(phase);
        in_imp = new("in_imp", this);                  // [2] new 创建
    endfunction

    task put(packet tr);                               // [3] 实现 put！方法名要和端口操作一致
        `uvm_info("CONS", "received packet", UVM_LOW)
        tr.print();
    endtask
endclass
```

没有定义匹配的 <code>put</code>，编译或展开 IMP 时会报找不到接口方法。

---

### 4.2.3 PORT 与 IMP 的连接（🟡 中）
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
        out_port.put(tr);         // 阻塞：没收到 consumer.put 返回就不继续
    end
endtask
```

**IMP 必须实现的方法**

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

**完整的层次化 put 链路**

一段 TLM 连接 = 一串"传话筒" + 一个"干活的人"。先认识四个角色：

| 角色 | 组件 | 干什么 |
|------|------|--------|
| **发起者** | `packet_source` | 真正调 `put` 发数据（最底层） |
| **外层传话筒** | `source_agent` | 把内部发起者的 PORT 透传给外面 |
| **接收传话筒** | `sink_agent` | 把 EXPORT 接进内部 |
| **干活的人** | `packet_sink` | 真正实现 `put`，收到数据存队列 |

**连接规则一句话**：每层只做一件事——**发起者用 PORT 往外发，agent 负责透传，接收方用 EXPORT 接进内部，最后落到 IMP 调用的 put 方法**。

**最底层发起者**（只管发，8 个包）：

```systemverilog
class packet_source extends uvm_component;
    uvm_blocking_put_port #(packet) tx_port;

    `uvm_component_utils(packet_source)

    function void build_phase(uvm_phase phase);
        super.build_phase(phase);
        tx_port = new("tx_port", this);        // [1] 端口也要 new
    endfunction

    task main_phase(uvm_phase phase);
        packet tr;
        repeat (8) begin
            tr = packet::type_id::create("tr");
            assert(tr.randomize());
            tx_port.put(tr);                   // [2] 发起调用，沿链传播
        end
    endtask
endclass
```

**父层透传 PORT**（agent 不干活，只把内部端口"递出去"）：

```systemverilog
class source_agent extends uvm_agent;
    packet_source src;
    uvm_blocking_put_port #(packet) tx_port;   // [1] agent 自己也开一个 PORT

    function void connect_phase(uvm_phase phase);
        super.connect_phase(phase);
        src.tx_port.connect(this.tx_port);     // [2] 内部 PORT → 自己 PORT，透传
    endfunction
endclass
```

**接收方**（EXPORT 接进内部，IMP 落地）：

```systemverilog
class packet_sink extends uvm_component;
    uvm_blocking_put_imp #(packet, packet_sink) rx_imp;   // [1] IMP：实现者是 packet_sink

    task put(packet tr);                                  // [2] 真正干活的 put
        packet copy_tr;
        $cast(copy_tr, tr.clone());                       // [3] 存副本，防止外部再改
        received.push_back(copy_tr);
    endtask
endclass

class sink_agent extends uvm_agent;
    packet_sink sink;
    uvm_blocking_put_export #(packet) rx_export;          // [1] EXPORT：转发用

    function void connect_phase(uvm_phase phase);
        super.connect_phase(phase);
        rx_export.connect(sink.rx_imp);                   // [2] EXPORT → 内部 IMP
    endfunction
endclass
```

**env 最后一接，全链贯通**：

```systemverilog
src_agt.tx_port.connect(sink_agt.rx_export);   // 外部 PORT → 外部 EXPORT，中间一接完成
```

**最终调用链**（记住这一条就够了）：

```text
src.tx_port → src_agt.tx_port → sink_agt.rx_export → sink.rx_imp → sink.put(tr)
    发起             透传               转发             到 IMP        真正干活
```

中间全是传话筒，**只有最后的 `sink.put(tr)` 在干活**。调试时从终点 put 反向逐环检查，每级端口操作族（put/get）和 transaction 类型（packet）必须一致。

---

### 4.2.4 EXPORT 与 IMP 的连接（非重点：4.2.4~4.2.6）

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

**层次连接方向速记**

| 同类型连接 | 典型方向 |
|------------|----------|
| PORT -> PORT | child PORT 连接 parent PORT |
| EXPORT -> EXPORT | parent EXPORT 连接 child EXPORT |

不要只看图形上下位置，最终以“谁调用 connect、谁是被调用接口”判断。

---

### 4.2.7 blocking_get 端口的使用（非重点：4.2.7~4.2.9）
get 中，调用者主动索取数据，因此调用者使用 PORT。

```text
控制流：consumer.PORT ------> producer.IMP
数据流：consumer <----------- producer
```

**数据提供者**
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

**数据使用者**
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

**连接**
```systemverilog
cons.get_port.connect(prod.get_imp);
```

get 的 PORT 在数据接收方，进一步证明 PORT/EXPORT 描述控制流，而不是数据流。

---

### 4.2.8 blocking_transport 端口的使用
transport 一次调用同时传入 request 和返回 response。

**发起者**
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

**目标 component**
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

**连接**
```systemverilog
req_inst.trans_port.connect(rsp_inst.trans_imp);
```

transport 适合严格的一问一答；持续流水、乱序响应常需要更解耦的结构。

---

### 4.2.9 nonblocking 端口的使用
nonblocking 操作必须立即返回，因此接口是 function。

**try_put 与 can_put**
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

**IMP 实现**
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

**blocking 与 nonblocking**

| 特性 | blocking | nonblocking |
|------|----------|-------------|
| 接口实现 | task 或兼容函数 | function |
| 可消耗时间 | 可以 | 不可以 |
| 无资源时 | 调用阻塞等待 | 立即返回失败 |
| 重试策略 | 接收方/通道内部 | 调用方负责 |

**nonblocking get 模板**
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

**get 与 peek 的实现对照**
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

**连接合法性速查**

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

## 4.3 UVM 中的通信方式（🔴 高）
### 4.3.1 UVM 中的 analysis 端口（🔴 高）
analysis_port 用于**发布/订阅式广播**：发布者（monitor）把数据"扔出去"，**所有连着的订阅者都能收到**，发布者不管谁收、收几个。

特点：

| 特点 | 含义 |
|------|------|
| 一对多 | 一个 analysis_port 可以连多个订阅者 |
| 只有 write 操作 | 没有 put/get/try，就一个 write() |
| write 是 function | 不能阻塞、不能耗时（立即返回） |
| 不等待响应 | 发布者 write 完就走，不等订阅者处理完 |
| 一次广播多方收 | monitor 采到的数据同时给 scoreboard、coverage collector |

为什么 write 必须是 function：因为广播是"发完不管"——如果 write 能阻塞，发布者就得等所有订阅者处理完，一对多就变成"一对多排队"，失去广播意义。

**发布者（monitor）——持有 analysis_port，调 write：**
```systemverilog
class my_monitor extends uvm_monitor;
    uvm_analysis_port #(packet) ap;      // [1] 发布者：analysis_port

    task main_phase(uvm_phase phase);
        forever begin
            tr = collect_packet();
            ap.write(tr);                // [2] 广播，不等待
        end
    endtask
endclass
```

**订阅者（coverage_collector）——持有 analysis_imp，实现 write：**
```systemverilog
class coverage_collector extends uvm_component;
    uvm_analysis_imp #(packet, coverage_collector) imp;  // [1] 订阅者：analysis_imp

    function void write(packet tr);      // [2] 实现 write（注意是 function！）
        sample_packet(tr);
    endfunction
endclass
```

**注意**：订阅者的 write 也必须是 **function**（因为要匹配发布者的非阻塞语义）。

**一对多连接**

```systemverilog
mon.ap.connect(scb.input_imp);   // [1] 连 scoreboard
mon.ap.connect(cov.imp);         // [2] 同一个 ap 再连 coverage —— 一对多！
```

一个 monitor 的数据同时进 scoreboard（比对）和 coverage（采样）——**这就是验证平台里 analysis 存在的最大理由**。

**注意**：analysis 链的终点也必须是 **analysis_imp / subscriber / FIFO**（不能断在 PORT/EXPORT 上，和 4.2 的规则一致）。

**句柄共享风险**
analysis 传的是对象句柄（不是副本）——多个订阅者收到的是同一个对象：

```systemverilog
scb 收到 tr 后改了 tr.data = 0;   // [1] 改了共享对象
cov 再收时看到的 data 已经变了！  // [2] 被坑了
```

建议：

- **monitor 发布后不要再修改对象**。
- 订阅者**需要修改就先 clone/copy**。
- 订阅者**默认把收到的 transaction 当只读**。

---

### 4.3.2 一个 component 内有多个 IMP（🔴 高）
scoreboard 要收两路：

- model 发来的 **expected**（期望值）
- monitor 发来的 **actual**（实际值）

如果直接用两个 `uvm_analysis_imp`，**两个 IMP 都要求实现叫 `write` 的方法**——同一个类里不能有两个同名 `write`，无法区分数据来自哪一路（问题所在）。

**声明不同 IMP 后缀**
```systemverilog
`uvm_analysis_imp_decl(_expected)   // [1] 宏：生成 uvm_analysis_imp_expected 类型
`uvm_analysis_imp_decl(_actual)     // [2] 宏：生成 uvm_analysis_imp_actual 类型
```

这两个宏**自动生成两个新端口类型**，分别要求实现**带后缀的方法**：

```systemverilog
class my_scoreboard extends uvm_scoreboard;
    uvm_analysis_imp_expected #(packet, my_scoreboard) exp_imp;   // [3] 期望输入口
    uvm_analysis_imp_actual   #(packet, my_scoreboard) act_imp;   // [4] 实际输入口

    function void write_expected(packet tr);   // [5] expected 来了 → 这个函数
        expected_q.push_back(tr);
    endfunction

    function void write_actual(packet tr);     // [6] actual 来了 → 这个函数
        // 取出期望、比对
    endfunction
endclass
```

宏后缀与方法后缀必须一致：

```text
_expected -> write_expected
_actual   -> write_actual
```

**双 analysis IMP 的完整连接**

scoreboard 收两路数据：model 发**期望值**（expected），monitor 发**实际值**（actual）。两个数据源各发各的：

```systemverilog
class my_model extends uvm_component;
    uvm_analysis_port #(packet) exp_ap;   // [1] model 的发布口：发期望
    // 计算完成后 exp_ap.write(expected)
endclass

class my_monitor extends uvm_monitor;
    uvm_analysis_port #(packet) act_ap;   // [2] monitor 的发布口：发实际
    // 采集完成后 act_ap.write(actual)
endclass
```

scoreboard 用两个带后缀的 IMP 接收，核心逻辑一句话：**期望入队、实际取队头比对**：

```systemverilog
`uvm_analysis_imp_decl(_expected)     // [1] 宏：生成"期望输入口"类型
`uvm_analysis_imp_decl(_actual)       // [2] 宏：生成"实际输入口"类型

class my_scoreboard extends uvm_scoreboard;
    uvm_analysis_imp_expected #(packet, my_scoreboard) exp_imp;   // [3] 期望输入口
    uvm_analysis_imp_actual   #(packet, my_scoreboard) act_imp;   // [4] 实际输入口
    packet expected_q[$];

    function void write_expected(packet tr);   // [5] 期望来了：入队（先复制防被改）
        packet copy_tr;
        $cast(copy_tr, tr.clone());
        expected_q.push_back(copy_tr);
    endfunction

    function void write_actual(packet tr);     // [6] 实际来了：取队头期望来比
        packet expected;
        if (expected_q.size() == 0) begin
            `uvm_error("SCB", "unexpected actual packet")   // [7] 实际先到 = 有异常
            return;
        end
        expected = expected_q.pop_front();
        if (!tr.compare(expected))
            `uvm_error("SCB", "actual packet differs from expected")
    endfunction
endclass
```

env 把两路接进来：

```systemverilog
mdl.exp_ap.connect(scb.exp_imp);   // [1] 期望路
o_agt.ap.connect(scb.act_imp);     // [2] 实际路
```

**两个注意点**：

1. **write_xxx 必须轻量**：write 是 function 不能阻塞——不能在 write_actual 里"等期望入队"，只能"没有就报错返回"（就是上面 [7] 的处理）。
2. **入队前 clone**：analysis 传句柄，期望入队前复制，防止后面被外部修改污染队列。

---

### 4.3.3 使用 FIFO 通信（🔴 高）

**FIFO = 中间仓库**：把 monitor 的"广播"（analysis，发完不管）转换成 scoreboard 的"拉取"（get，主动要）——发送者只管往里存，接收者按自己节奏取。

```text
monitor.analysis_port
        |
        v
FIFO.analysis_export -> FIFO buffer -> FIFO.blocking_get_export
                                             |
                                             v
                                scoreboard.blocking_get_port
```

**搭建：env 建 FIFO，scoreboard 建 get port**

env 里建两个"仓库"（期望一个、实际一个）：

```systemverilog
uvm_tlm_analysis_fifo #(packet) exp_fifo;
uvm_tlm_analysis_fifo #(packet) act_fifo;
// build_phase 里 new("exp_fifo", this) / new("act_fifo", this)
```

scoreboard 声明两个"取货口"：

```systemverilog
uvm_blocking_get_port #(packet) exp_port;
uvm_blocking_get_port #(packet) act_port;
```

**连接：一边存、一边取**

```systemverilog
mdl.ap.connect(exp_fifo.analysis_export);           // [1] model 的广播 → 存进仓库
scb.exp_port.connect(exp_fifo.blocking_get_export); // [2] scoreboard 从仓库取

o_agt.ap.connect(act_fifo.analysis_export);
scb.act_port.connect(act_fifo.blocking_get_export);
```

**用法：scoreboard 按自己节奏取（简单版）**

```systemverilog
forever begin
    exp_port.get(expected);   // [1] 等仓库里有期望
    act_port.get(actual);     // [2] 等仓库里有实际
    if (!actual.compare(expected))
        `uvm_error("SCB", "compare failed")   // [3] 比较
end
```

**FIFO 的好处**：发送者（model/monitor）和接收者（scoreboard）**节奏解耦**——发方推完就走，收方有空才取；scoreboard 还能用 blocking get 主动等。

**更稳的双 FIFO scoreboard（先收齐再比）**

上面简单版**边收边比**，如果 expected 和 actual 到达节奏差很多会卡。稳健版改成 **三个并行进程：各收各的、专门比较**：

```systemverilog
task main_phase(uvm_phase phase);
    fork
        collect_expected();   // [1] 专收期望，存队列
        collect_actual();     // [2] 专收实际，存队列
        compare_queues();     // [3] 两边都有货才取出来比
    join
endtask

// collect_expected：exp_port.get(tr) → expected_q.push_back(tr)
// collect_actual：  act_port.get(tr) → actual_q.push_back(tr)

task compare_queues();
    forever begin
        wait (expected_q.size() != 0);   // [4] 等期望有货
        wait (actual_q.size()   != 0);   // [5] 等实际有货
        expected = expected_q.pop_front();
        actual   = actual_q.pop_front();
        if (!actual.compare(expected))
            `uvm_error("SCB", "mismatch")
    end
endtask
```

**分工**：两个"收货员"各收各的存队列，一个"比对员"两边都有货才取一对来比——**谁快谁先存着，互不阻塞**。

**仿真结束检查残留（check_phase）**

```systemverilog
function void my_scoreboard::check_phase(uvm_phase phase);
    if (expected_q.size() != 0)
        `uvm_error("SCB", "%0d expected packets remain", expected_q.size())
    if (actual_q.size() != 0)
        `uvm_error("SCB", "%0d actual packets remain", actual_q.size())
endfunction
```

**为什么查残留**：仿真结束队列里还有没比完的数据 = **DUT 少输出或多输出**（漏了尾巴或多了货）——这是"最后一包没比较"那类 bug 的检查点。

**乱序输出的提醒**

**如果 DUT 支持乱序输出，不能按队头一比一**——expected 和 actual 到达顺序可能不同，简单 `pop_front` 会误报 mismatch。**应按 transaction 的 ID/tag/地址建关联数组匹配**。

### 4.3.4 FIFO 上的端口及调试（🟡 中）

**FIFO 是个"多面手"**：既能当仓库（analysis 写入、get 取出），还自带广播口和调试函数。

**两种 FIFO**

| FIFO 类型 | 能力 |
|-----------|------|
| `uvm_tlm_fifo` | put/get/peek 等基础接口 |
| `uvm_tlm_analysis_fifo` | 在基础上**增加 analysis_export + write**（能被 monitor 广播写入）——验证平台用这个 |

**get 与 peek 的区别**

| 操作 | 返回队头 transaction | 从 FIFO 删除 |
|------|:---:|:---:|
| get | 是 | **是** |
| peek | 是 | **否**（只看不取） |

peek = **预览**队头数据（比如先看看是不是要的那笔），真正消费还得 get。

**FIFO 自带的两个广播口（put_ap / get_ap）**

```text
put → 存入 FIFO 缓冲区 + put_ap.write(tr)   // [1] 存进去时广播一份
get → 从 FIFO 弹出     + get_ap.write(tr)   // [2] 取出来时广播一份
```

**用途**：流量监控、覆盖率收集——不用改 FIFO 内部，监听这两个口就能统计"存了多少、取了多少"。

**FIFO 调试函数（查表）**

| 函数 | 作用 |
|------|------|
| `used()` | 当前缓存多少笔 |
| `is_empty()` | 是否为空 |
| `is_full()` | 是否已满 |
| `size()` | 容量上限 |
| `flush()` | 清空缓存 |

**FIFO 容量**

```systemverilog
fifo = new("fifo", this, 16);   // [1] 最多缓存 16 笔
```

**容量为 0 = 不限制大小**。

**复位时清空**

```systemverilog
function void my_env::flush_fifos();
    exp_fifo.flush();
    act_fifo.flush();
endfunction
```

**清空不是"点一下就完"**：清空前要确认**阻塞 get、scoreboard 队列和 DUT 流水状态**——如果 scoreboard 还卡在 `get()` 等数据，或 DUT 流水里还有货，只清 FIFO 会造成**新旧数据错位**（只清一半比不清更乱）。
### 4.3.5 用 FIFO 还是用 IMP（🔴 高）

**一句话决策**：**简单订阅用 IMP（轻、直接），需要节奏解耦/主动拉取用 FIFO（稳、灵活）**。

| 对比 | 直接 IMP | FIFO |
|------|----------|------|
| 代码路径 | 短，write 立即进接收者 | 中间加缓存 |
| 接收方式 | 被动回调（数据来就触发） | 主动 get（自己拉） |
| 节奏解耦 | 弱（发一个处理一个） | 强（发存取拉互不阻塞） |
| 多输入 | 要多个 IMP 后缀 | 可用 FIFO/port 数组 |
| 延迟 | 基本无缓冲 | 由队列解耦 |
| 调试 | 连接直接 | 可看 used/empty/full |
| 适合 | coverage、简单订阅 | scoreboard、速率不匹配 |

**什么时候选 IMP**

- 简单 analysis subscriber（比如覆盖率采集）；
- write 里只做**轻量、不阻塞**的处理；
- 输入数量少且固定。

**什么时候选 FIFO**

- scoreboard 要**主动同步**多路数据；
- 接收处理可能**晚于**发送（速率不匹配）；
- 多通道（用数组 + 循环构建）；
- 需要缓存深度和 flush 调试能力。

**多端口数组（多通道写法）**

```systemverilog
uvm_blocking_get_port #(packet) exp_port[16];   // [1] 16 个取货口
uvm_tlm_analysis_fifo #(packet) exp_fifo[16];   // [2] 16 个仓库

// build：foreach 循环创建，名字带索引
foreach (exp_port[i]) begin
    exp_port[i] = new($sformatf("exp_port_%0d", i), this);
    exp_fifo[i] = new($sformatf("exp_fifo_%0d", i), this);
end

// 连接：每个通道 仓库←model、取货口←仓库
foreach (exp_port[i]) begin
    mdl.ap[i].connect(exp_fifo[i].analysis_export);
    exp_port[i].connect(exp_fifo[i].blocking_get_export);
end
```

**并发读取的坑——循环变量要 automatic**：

```systemverilog
foreach (exp_port[i]) begin
    fork
        automatic int channel = i;   // [3] 关键！否则所有进程共享同一个 i
        forever begin
            packet tr;
            exp_port[channel].get(tr);
            process_expected(channel, tr);
        end
    join_none
end
```

`automatic int channel = i` 把 i 的**当前值复制一份**给每个进程——不加 automatic，fork 的进程运行时 i 已经循环到末尾，**全部通道串线**（这是经典 bug）。

**TLM 故障定位表（排障查这个）**

| 现象 | 常见原因 | 检查方法 |
|------|----------|----------|
| connection count 为 0 | 没 connect 或链路没到 IMP | 打印 topology 逐级查 |
| No field named put/get/write | IMP 组件没实现对应方法 | 对照操作族补接口 |
| 类型不兼容 | transaction 参数或 blocking 属性不一致 | 对照完整类型声明 |
| write 后数据错乱 | 多订阅者共享句柄 | 发布后只读、入队前 clone |
| blocking get 永久等待 | 上游没写 FIFO 或连错方向 | 看 FIFO.used 与 write 日志 |
| FIFO 持续增长 | 消费太慢或没人 get | 监控 used/is_full |
| 最后数据残留 | phase 提前结束或 drain 不足 | check_phase 查队列 |
| 多通道串线 | 数组索引或 automatic 变量错 | 打印 channel 与 transaction ID |

**推荐调试顺序**（按步骤走）：print_topology → 确认端口都 new → 从 PORT 沿链追到 IMP → 核对操作族 → 核对类型 → 终点打印日志 → FIFO 看 used/empty/full → check_phase 查残留。

**设计评审自查**（6 问）：推送还是拉取？要不要缓存解耦？订阅者会改共享对象吗？FIFO 容量/复位 flush/残留有策略吗？多通道有 channel/ID 标识吗？所有阻塞调用都能解除阻塞吗？
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