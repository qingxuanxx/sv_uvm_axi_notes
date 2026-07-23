# UVM实战 第 9 章学习笔记：UVM 中代码的可重用性
> **核心结论**：可重用性不是把一个类做得无所不能，而是稳定公共结构、暴露小而明确的扩展点，并把模块级专属资源放在合适层次。callback 处理局部行为插入，factory 替换完整实现，参数化类抽象类型/位宽，芯片级重用则重新组合 env、寄存器模型和 sequence。
>
> **记忆主线**：公共功能留在基类 -> 变化点用 callback/factory/config 暴露 -> sequence 保持小而专一 -> 模块 env 不绑定芯片级资源 -> 芯片层重新组合接口、寄存器模型和 virtual sequence。

---

## 9.0 本章定位

| 章节 | 核心问题 |
|------|----------|
| 9.1 | callback 如何在不修改组件源码时插入行为 |
| 9.2 | 为什么“小而美”比万能类更容易复用 |
| 9.3 | 参数化类如何适配类型、地址和数据位宽 |
| 9.4 | 模块级 env 如何迁移到芯片级验证平台 |
可重用性有多个层次：

```text
同一测试中的函数复用
        |
同一项目中的 sequence/component 复用
        |
不同项目之间的 env/VIP 复用
        |
模块级环境进入芯片级验证
```

越往后，对接口稳定性、配置能力和层次边界的要求越高。

### 零基础阅读提示

- callback 是在固定流程中预留的小钩子，适合局部修改。
- factory override 是把整个实现类换掉，适合算法整体变化。
- sequence 负责测试流程，不能因为 callback 很灵活就把 sequence 全部取消。
- 参数化类解决静态类型或位宽差异，运行时开关更适合配置对象。
- block 到 chip 的复用不是原样搬运，而是保留模块内部能力、在芯片层重新接线。

遇到“应该用哪种机制”时，先问变化是局部动作、完整实现、测试流程，还是简单参数，再做选择。

---

## 9.1 callback 机制

### 9.1.1 广义的 callback 函数
callback 是框架在预定时机主动调用用户扩展逻辑。
SystemVerilog 的 `post_randomize()` 是最简单的 callback。
手工计算 CRC：

```systemverilog
my_transaction tr;
tr = new("tr");
assert(tr.randomize());
// 每次随机化后都必须记得调用，容易遗漏。
tr.crc = tr.calc_crc();
```

使用 `post_randomize()`：

```systemverilog
class my_transaction extends uvm_sequence_item;
    rand byte payload[];
         bit [31:0] crc;
    function void post_randomize();
        super.post_randomize();
        // randomize 成功后由语言自动回调。
        crc = calc_crc(payload);
    endfunction
endclass
```

优点：
- 调用时机固定。
- 用户不会忘记执行公共后处理。
- 随机化与派生字段维护封装在类内部。
UVM 中的广义 callback 还包括：
- sequence 的 `pre_body()`、`post_body()`。
- sequence 的 `pre_do()`、`mid_do()`、`post_do()`。
- phase 方法。
- report catcher 等框架扩展点。
广义 callback 与 UVM 专用 callback 机制的共同思想是“控制反转”。

```text
ordinary call : user code calls library
callback      : library/framework calls user extension
```

---

### 9.1.2 callback 机制的必要性
验证平台开发者无法预知所有使用者需求。
成熟 VIP 的 driver 在发送 transaction 前，用户可能希望：
- 写入包序号。
- 修改最后几个字节。
- 截短数据包。
- 注入 CRC 错误。
- 发送特殊前导码。
- 记录协议统计信息。
- 根据外部状态调整字段。
若没有扩展点，用户只能：
- 修改 VIP 源码。
- 派生并 factory override 整个 driver。
- 复制 driver 后自行维护。
callback 允许 VIP 预留局部插入点：

```text
driver stable flow
    get req
       |
    pre_tran callbacks   <- user extension
       |
    drive packet
       |
    post_tran callbacks  <- user extension
       |
    item_done
```

callback 适合“小范围行为差异”。
若整个算法和生命周期都改变，factory override 更清晰。

---

### 9.1.3 UVM 中 callback 机制的原理
仅定义一个 virtual 方法不够。
假设从 `my_driver` 派生 `new_driver` 并重写 `pre_tran()`，但验证平台仍实例化 `my_driver`，新方法不会运行。
UVM callback 引入两个角色：
1. callback 基类，声明扩展方法。
2. callback pool，保存某个组件实例对应的 callback 对象。
概念执行：

```systemverilog
foreach (callback_pool[i]) begin
    // driver 不需要知道具体派生 callback 类型，只调用统一虚接口。
    callback_pool[i].pre_tran(this, req);
end
```

结构：

```text
my_driver instance
       |
callback pool for this driver
       |-- callback object 0
       |-- callback object 1
       `-- callback object 2
```

一个组件实例可注册多个 callback。
同一 callback 类型也可分别添加到不同 driver 实例。

---

### 9.1.4 callback 机制的使用

#### 第一步：定义 callback 基类

```systemverilog
class driver_callback extends uvm_callback;
    `uvm_object_utils(driver_callback)
    function new(string name = "driver_callback");
        super.new(name);
    endfunction
    virtual task pre_tran(
        my_driver drv,
        ref my_transaction tr
    );
        // 默认空实现，派生 callback 按需重写。
    endtask
endclass
```

必须从 `uvm_callback` 派生。
扩展方法必须声明为 virtual。
传入 driver 句柄可访问上下文；`ref tr` 允许 callback 修改原 transaction。

#### 第二步：声明 callback pool

```systemverilog
typedef uvm_callbacks #(
    my_driver,          // 哪一种组件会执行这些 callback
    driver_callback     // pool 中允许保存的 callback 基类
) driver_callback_pool;
```

第一个参数是使用 callback 的组件类型。
第二个参数是 callback 基类类型。

#### 第三步：在 driver 中注册 callback 类型

```systemverilog
class my_driver extends uvm_driver #(my_transaction);
    `uvm_component_utils(my_driver)
    `uvm_register_cb(my_driver, driver_callback)
    // ...
endclass
```

如果 callback 类定义在 driver 之后，可先前向声明：

```systemverilog
typedef class driver_callback;
```

#### 第四步：在扩展点执行 callback

```systemverilog
task my_driver::main_phase(uvm_phase phase);
    forever begin
        seq_item_port.get_next_item(req);
        // 依次执行挂到当前 driver 实例上的全部 callback。
        `uvm_do_callbacks(
            my_driver,
            driver_callback,
            pre_tran(this, req)
        )
        drive_one_pkt(req);
        seq_item_port.item_done();
    end
endtask
```

宏参数：

| 参数 | 含义 |
|------|------|
| `my_driver` | 调用 callback 的组件类型 |
| `driver_callback` | callback 基类类型 |
| `pre_tran(this, req)` | 具体回调方法及实参 |
到这里属于 VIP/平台开发者的工作。

#### 第五步：用户实现具体 callback

```systemverilog
class append_id_callback extends driver_callback;
    `uvm_object_utils(append_id_callback)
    int unsigned packet_id;
    virtual task pre_tran(
        my_driver drv,
        ref my_transaction tr
    );
        packet_id++;
        // 在驱动前写入调试序号。
        tr.payload[tr.payload.size()-1] = packet_id;
        `uvm_info("CALLBACK",
                  $sformatf("append packet id %0d", packet_id),
                  UVM_MEDIUM)
    endtask
endclass
```

#### 第六步：实例化并加入 pool

```systemverilog
function void my_test::connect_phase(uvm_phase phase);
    append_id_callback cb;
    super.connect_phase(phase);
    cb = append_id_callback::type_id::create("cb");
    // 只挂到指定 driver 实例。
    driver_callback_pool::add(env.i_agt.drv, cb);
endfunction
```

必须在 driver 首次执行扩展点之前完成 add。
通常可在 build、connect 或 start_of_simulation 阶段安装。

#### callback 删除
需要取消时：

```systemverilog
// 从指定 driver 的 pool 中移除这个 callback 实例；之后不再执行它。
driver_callback_pool::delete(env.i_agt.drv, cb);
```

动态添加/删除要考虑并发执行时机，不要在遍历 pool 的同时随意修改。

#### 多 callback 顺序
若同一 driver 挂多个 callback，执行顺序会影响结果。
例如：

```text
callback A: calculate CRC
callback B: corrupt payload
```

与反向顺序得到不同 CRC。
有顺序依赖时应明确设计并记录，不要让多个 callback 隐式修改同一字段。

---

### 9.1.5 子类继承父类的 callback 机制
从已有 driver 派生新 driver 时，希望继续使用针对父类注册的 callback。

```systemverilog
class new_driver extends my_driver;
    `uvm_component_utils(new_driver)
    // 告诉 callback 系统：new_driver 的 callback 超类型是 my_driver。
    `uvm_set_super_type(new_driver, my_driver)
    // ...
endclass
```

新 driver 中仍调用父类 callback 类型：

```systemverilog
`uvm_do_callbacks(
    my_driver,
    driver_callback,
    pre_tran(this, req)
)
```

这样旧测试中添加到 callback pool 的对象仍能用于新一代 driver。
价值：
- 旧 case 无需修改。
- 新平台扩展 driver 功能。
- callback 扩展契约保持兼容。
注意：
- 子类必须满足父 callback 对组件接口的假设。
- 若 callback 强依赖父类内部实现，继承后仍可能失效。
- callback API 应使用稳定、受保护的公开方法。

---

### 9.1.6 使用 callback 实现所有测试用例
理论上可以把 transaction 生成、objection 和驱动流程都放入 callback。

```systemverilog
class all_in_one_callback extends uvm_callback;
    virtual task run(my_driver drv, uvm_phase phase);
        phase.raise_objection(drv);
        // 生成激励并直接调用 driver。
        // 技术上可行，但 callback 承担了 sequence 和 test 的职责。
        phase.drop_objection(drv);
    endtask
endclass
```

教材不推荐用 callback 取代 sequence。
原因与“只重载 driver 实现所有 case”类似：
- 激励生成与驱动再次耦合。
- 失去 sequencer 仲裁。
- 失去嵌套 sequence 复用。
- 多接口同步困难。
- callback 变成隐藏的主控制流。
正确定位：

```text
sequence : 描述主要测试流程
callback : 在稳定流程的特定点插入小变化
```

---

### 9.1.7 callback、sequence 和 factory 的选择

| 变化需求 | 推荐机制 |
|----------|----------|
| 改变 transaction 字段/发送组合 | sequence |
| 协调多个接口 | virtual sequence |
| 在稳定 driver 流程中插入少量动作 | callback |
| 整体替换 driver/model/monitor 算法 | factory override |
| 修改简单参数 | config object/config_db |
callback 的优势：
- 不替换整个组件。
- 可同时安装多个扩展。
- 可以精确绑定某个组件实例。
- 很适合第三方 VIP 的用户钩子。
factory 的优势：
- 完整实现替换更清晰。
- 派生类拥有明确类型和生命周期。
- 不需要在原类中预留每个 callback 点。
sequence 的优势：
- 激励逻辑可组合。
- sequencer 提供仲裁。
- virtual sequence 支持系统同步。

---

## 9.2 功能的模块化：小而美

### 9.2.1 Linux 的设计哲学：小而美
“小而美”的核心是每个模块只做好一件事，并通过明确接口组合。
验证代码中的表现：
- 一个 sequence 对应一个清晰场景。
- 一个 function 完成一个明确计算。
- 一个 callback 修改一个局部行为。
- 一个配置对象保存一组相关参数。
- component 只承担所属层次职责。
好处：
- 更容易阅读。
- 更容易单独测试。
- 更容易组合。
- 修改时影响范围小。
- 名字本身就能说明用途。
“小”不等于复制粘贴。
公共部分仍应提取到基类、工具函数或共享 sequence。

---

### 9.2.2 小而美与 factory 机制的重载
不要在一个类中用大量 if/else 处理所有项目、协议和异常。
不推荐：

```systemverilog
if (mode == NORMAL)
    run_normal();
else if (mode == CRC_ERROR)
    run_crc_error();
else if (mode == TIMEOUT)
    run_timeout();
else if (mode == SPECIAL_PROJECT_B)
    run_project_b();
```

可以设计小型派生类：

```text
base_driver
  |-- crc_error_driver
  |-- timeout_driver
  `-- project_b_driver
```

由 test 用 factory 选择需要的实现。
适用前提：
- 各行为具有明确语义。
- 公共流程能留在基类。
- 派生类只覆盖少量 virtual 方法。
- 类型数量不会无控制增长。
factory 与小类结合，能让测试只声明差异。

---

### 9.2.3 放弃建造强大 sequence 的想法
万能 sequence 常包含大量参数：
- 正常包还是错误包。
- 随机生成还是文件读取。
- 单文件还是多文件。
- 文件格式。
- 是否写出发送内容。
- 长中短包阈值。
- 各长度发送比例。
- 是否插入包序号。
- 是否执行特殊同步。
结果是：
- 分支过多。
- 参数组合难以验证。
- 用户必须理解全部选项。
- 文档与代码容易不同步。
- 修改一处可能影响无关模式。
推荐拆分：

```text
normal_sequence
crc_error_sequence
read_from_text_sequence
short_packet_sequence
mixed_length_sequence
```

复杂场景通过嵌套或 virtual sequence 组合：

```systemverilog
virtual task body();
    normal_sequence    normal_seq;
    crc_error_sequence error_seq;
    `uvm_do(normal_seq)
    `uvm_do(error_seq)
endtask
```

拆分原则：
- 名称能表达测试意图。
- 参数数量保持可理解。
- 每个 sequence 可独立运行。
- 共享发送逻辑放在基类 task。
- 不为每个微小数值差异新建类。

---

## 9.3 参数化的类

### 9.3.1 参数化类的必要性
参数化类把“算法结构”与“所处理的类型或位宽”分离。
UVM sequence：

```systemverilog
virtual class uvm_sequence #(
    type REQ = uvm_sequence_item,
    type RSP = REQ
) extends uvm_sequence_base;
```

用户指定 transaction 类型：

```systemverilog
class bus_sequence extends uvm_sequence #(
    bus_transaction
);
```

适合参数化的对象：
- 总线地址宽度。
- 总线数据宽度。
- transaction 类型。
- TLM 传输类型。
- driver/sequencer 的 REQ/RSP 类型。
- 通道数量等真正影响静态结构的常量。
不必参数化：
- 普通开关。
- 运行时可变化的测试参数。
- 与静态类型无关的数值。
- 只在单一项目中固定不变的无意义参数。
运行时选项更适合配置对象。

---

### 9.3.2 UVM 对参数化类的支持

#### 参数化 transaction

```systemverilog
class bus_transaction #(
    int ADDR_WIDTH = 16,
    int DATA_WIDTH = 32
) extends uvm_sequence_item;
    rand bit [ADDR_WIDTH-1:0] addr;
    rand bit [DATA_WIDTH-1:0] data;
    `uvm_object_param_utils(
        bus_transaction #(ADDR_WIDTH, DATA_WIDTH)
    )
    function new(string name = "bus_transaction");
        super.new(name);
    endfunction
endclass
```

参数化 object 使用 `uvm_object_param_utils`。
参数化 component 使用 `uvm_component_param_utils`。

#### 参数化 interface

```systemverilog
interface bus_if #(
    int ADDR_WIDTH = 16,
    int DATA_WIDTH = 32
)(
    input logic clk,
    input logic rst_n
);
    logic                  valid;
    logic                  write;
    logic [ADDR_WIDTH-1:0] addr;
    logic [DATA_WIDTH-1:0] wdata;
    logic [DATA_WIDTH-1:0] rdata;
endinterface
```

config_db 的类型参数必须完全一致：

```systemverilog
typedef virtual bus_if #(16, 32) bus_vif_t;
uvm_config_db #(bus_vif_t)::set(
    null,
    "uvm_test_top.env.bus_agt.*",
    "vif",
    vif
);
```

获取：

```systemverilog
bus_vif_t vif;
if (!uvm_config_db #(bus_vif_t)::get(this, "", "vif", vif))
    `uvm_fatal("VIF", "parameterized virtual interface not found")
```

`bus_if #(16, 32)` 与 `bus_if #(32, 32)` 是不同类型。

#### 参数化 sequencer

```systemverilog
class bus_sequencer #(
    int ADDR_WIDTH = 16,
    int DATA_WIDTH = 32
) extends uvm_sequencer #(
    bus_transaction #(ADDR_WIDTH, DATA_WIDTH)
);
    `uvm_component_param_utils(
        bus_sequencer #(ADDR_WIDTH, DATA_WIDTH)
    )
endclass
```

#### factory 与默认参数
教材强调，使用参数化类的 factory `type_id` 时应显式写出参数。

```systemverilog
bus_agent #(16, 32) bus_agt;
bus_agt = bus_agent #(16, 32)::type_id::create(
    "bus_agt",
    this
);
```

不要依赖省略参数后 factory 自动推断默认特化。
参数化类的每一种特化都是不同类型：

```text
bus_agent #(16, 32)
bus_agent #(32, 32)
bus_agent #(32, 64)
```

factory override 也必须针对相同参数化基类的兼容派生类型。

---

## 9.4 模块级到芯片级的代码重用

### 9.4.1 基于 env 的重用
假设芯片由 A、B、C 三个模块串联。
模块级验证时，每个模块有自己的输入 active agent 和输出 monitor。

```text
module env
  input active agent -> DUT block -> output passive agent
             |                         |
             +------ model/scb --------+
```

芯片级连接后，A 的输出就是 B 的输入，B 的输出就是 C 的输入。
若完整保留每个模块的 input monitor，会重复监测同一接口。
两种复用方案：
1. 内部模块 input agent 改为 passive。
2. 取消冗余 input agent，由上游 env 的 analysis port 向下游 model 供数。
教材更推荐第二种，因为能减少重复 monitor 并加强模块间接口检查。

#### 可复用模块 env

```systemverilog
class block_env extends uvm_env;
    `uvm_component_utils(block_env)
    bit in_chip;
    uvm_analysis_port   #(my_transaction) ap;
    uvm_analysis_export #(my_transaction) i_export;
    my_agent input_agt;
    my_agent output_agt;
    uvm_tlm_analysis_fifo #(my_transaction) model_fifo;
    function void build_phase(uvm_phase phase);
        super.build_phase(phase);
        output_agt = my_agent::type_id::create("output_agt", this);
        model_fifo = new("model_fifo", this);
        if (!in_chip) begin
            // 模块级：自己驱动和监测输入。
            input_agt = my_agent::type_id::create("input_agt", this);
            input_agt.is_active = UVM_ACTIVE;
        end
        else begin
            // 芯片级：输入 transaction 由上游 env 提供。
            i_export = new("i_export", this);
        end
    endfunction
    function void connect_phase(uvm_phase phase);
        super.connect_phase(phase);
        // 向下游模块输出本模块实际输出 transaction。
        ap = output_agt.ap;
        if (in_chip)
            i_export.connect(model_fifo.analysis_export);
        else
            input_agt.ap.connect(model_fifo.analysis_export);
    endfunction
endclass
```

芯片 env 组合：

```systemverilog
function void chip_env::build_phase(uvm_phase phase);
    super.build_phase(phase);
    env_a = block_env::type_id::create("env_a", this);
    env_b = block_env::type_id::create("env_b", this);
    env_b.in_chip = 1;
    env_c = block_env::type_id::create("env_c", this);
    env_c.in_chip = 1;
endfunction
function void chip_env::connect_phase(uvm_phase phase);
    super.connect_phase(phase);
    env_a.ap.connect(env_b.i_export);
    env_b.ap.connect(env_c.i_export);
endfunction
```

收益：
- 消除重复 monitor。
- 下游模型直接检查上游实际输出。
- 模块 env 仍能独立用于 block-level。
- 芯片层只负责组合，不复制模块内部逻辑。

#### `in_chip` 的设计建议
教材直接使用 bit 字段。
工程中可使用配置对象，让模式在创建前确定：

```systemverilog
block_env_cfg cfg;
cfg.in_chip = 1;
```

因为 `in_chip` 影响组件是否创建，必须在 block env 的 build 判断前设置。

---

### 9.4.2 寄存器模型的重用
模块级和芯片级的配置总线拓扑通常不同。
模块级：

```text
block bus agent -> block register interface
```

芯片级：

```text
chip bus agent -> interconnect -> A/B/C register interface
```

因此 bus agent 不宜固化在可复用 block env 内。
模块级专用 bus agent 可放在 base_test；芯片级由 chip env/test 创建总线 agent。
同理，root register model 不宜固定在 block env 中。
block env 只保存本模块寄存器模型指针：

```systemverilog
class block_env extends uvm_env;
    reg_model p_rm;
endclass
```

模块级 test 创建 block model 并赋值。
芯片级建立组合寄存器模型：

```systemverilog
class chip_reg_model extends uvm_reg_block;
    `uvm_object_utils(chip_reg_model)
    rand reg_model a_rm;
    rand reg_model b_rm;
    rand reg_model c_rm;
    function new(string name = "chip_reg_model");
        super.new(name, UVM_NO_COVERAGE);
    endfunction
    virtual function void build();
        default_map = create_map(
            "default_map", 0, 2, UVM_BIG_ENDIAN, 0
        );
        a_rm = reg_model::type_id::create("a_rm");
        a_rm.configure(this, "A");
        a_rm.build();
        a_rm.lock_model();
        default_map.add_submap(a_rm.default_map, 16'h0000);
        b_rm = reg_model::type_id::create("b_rm");
        b_rm.configure(this, "B");
        b_rm.build();
        b_rm.lock_model();
        default_map.add_submap(b_rm.default_map, 16'h4000);
        c_rm = reg_model::type_id::create("c_rm");
        c_rm.configure(this, "C");
        c_rm.build();
        c_rm.lock_model();
        default_map.add_submap(c_rm.default_map, 16'h8000);
    endfunction
endclass
```

芯片层创建并分发子模型指针：

```systemverilog
chip_rm = chip_reg_model::type_id::create("chip_rm");
chip_rm.configure(null, "top_tb.dut");
chip_rm.build();
chip_rm.lock_model();
chip_rm.reset();
env_a.p_rm = chip_rm.a_rm;
env_b.p_rm = chip_rm.b_rm;
env_c.p_rm = chip_rm.c_rm;
```

芯片级只增加：
- 顶层 map。
- 子 block 地址偏移。
- 芯片层 HDL path。
- 统一 bus sequencer/adapter 绑定。
模块寄存器定义本身无需复制。

#### 层次所有权原则

| 资源 | 推荐所有者 |
|------|------------|
| 模块数据路径 monitor/model | block env |
| 模块级独立配置 bus agent | block base_test |
| 芯片级共享配置 bus agent | chip env/test |
| 模块寄存器 block 定义 | 可复用 RAL package |
| 芯片 root reg block | chip 层 |
| 子 reg model 指针 | 传给对应 block env |

---

### 9.4.3 virtual sequence 与 virtual sequencer
模块级 virtual sequencer 往往包含模块测试专用资源。
例如内部模块 B 的模块级 virtual sequencer 可能同时保存：
- B 输入 sequencer。
- B 配置寄存器模型。
- 模块级专用同步对象。
到了芯片级，B 的输入不再由独立 driver 驱动，因此该 virtual sequencer 不再存在。
教材建议：
- 模块级 virtual sequencer 不必固化在 block env 中。
- 在模块 base_test 中创建模块级 virtual sequencer。
- 芯片级建立自己的 virtual sequencer。
芯片有多个边界输入时：

```systemverilog
class chip_virtual_sequencer extends uvm_sequencer;
    `uvm_component_utils(chip_virtual_sequencer)
    a_sequencer p_a_sqr;
    d_sequencer p_d_sqr;
    f_sequencer p_f_sqr;
    chip_reg_model p_rm;
endclass
```

芯片级 virtual sequence 复用模块普通 sequence：

```systemverilog
class chip_vseq extends uvm_sequence;
    `uvm_object_utils(chip_vseq)
    `uvm_declare_p_sequencer(chip_virtual_sequencer)
    virtual task body();
        a_data_sequence a_seq;
        d_data_sequence d_seq;
        f_data_sequence f_seq;
        fork
            `uvm_do_on(a_seq, p_sequencer.p_a_sqr)
            `uvm_do_on(d_seq, p_sequencer.p_d_sqr)
            `uvm_do_on(f_seq, p_sequencer.p_f_sqr)
        join
    endtask
endclass
```

注意应分别传入 `a_seq`、`d_seq`、`f_seq`，不能因复制代码把三个分支都写成同一句柄。

#### 哪些 sequence 容易跨层重用
可直接复用：
- 边界接口上的普通 transaction sequence。
- 不依赖模块 virtual sequencer 类型的原子 sequence。
- 通过显式寄存器模型句柄工作的配置 sequence。
难以直接复用：
- 用 `uvm_declare_p_sequencer(module_virtual_sequencer)` 强绑定模块层次的 virtual sequence。
- 直接引用模块 test 层句柄的 sequence。
- 假设模块级 driver 一定存在的 sequence。

#### 可复用寄存器配置 sequence
不推荐强绑定 p_sequencer：

```systemverilog
class a_cfg_seq extends uvm_sequence;
    `uvm_declare_p_sequencer(a_virtual_sequencer)
    virtual task body();
        p_sequencer.p_rm.ctrl.write(status, 1);
    endtask
endclass
```

推荐显式模型句柄：

```systemverilog
class a_cfg_seq extends uvm_sequence;
    `uvm_object_utils(a_cfg_seq)
    a_reg_model p_rm;
    virtual task body();
        if (p_rm == null)
            `uvm_fatal("RAL", "p_rm is null")
        p_rm.ctrl.write(status, 1, UVM_FRONTDOOR);
    endtask
endclass
```

模块级启动：

```systemverilog
cfg_seq = a_cfg_seq::type_id::create("cfg_seq");
cfg_seq.p_rm = module_rm;
cfg_seq.start(null);
```

芯片级启动：

```systemverilog
cfg_seq = a_cfg_seq::type_id::create("cfg_seq");
cfg_seq.p_rm = chip_rm.a_rm;
cfg_seq.start(null);
```

配置 sequence 依赖的是 RAL 对象，不依赖数据 sequencer，因此可 `start(null)`。

#### 通过名字查找 block
复杂芯片可使用 `find_block()`、`find_blocks()`、`get_blocks()` 或 `get_block_by_name()` 查找子模型。
但显式传递句柄通常更清晰、更容易静态检查。

---

## 9.5 可重用验证平台设计清单

### 9.5.1 VIP 开发者应提供
- 稳定 transaction 定义。
- 可配置 active/passive agent。
- 明确的 analysis port/export。
- 配置对象而非散落全局变量。
- 必要且命名清晰的 callback 点。
- factory 注册和可覆盖 virtual 方法。
- 不依赖具体 test 层次的 sequence。
- 错误处理与调试日志。

### 9.5.2 block env 不应硬编码
- `uvm_test_top` 绝对路径。
- 芯片级总线 agent。
- 芯片 root register model。
- 固定模块基地址。
- 只在某个 test 存在的 virtual sequencer。
- 外部模块内部句柄。

### 9.5.3 从 block 到 chip 的迁移步骤
1. 确认每个 agent 支持 active/passive。
2. 为 env 暴露输入 export 和输出 analysis port。
3. 删除芯片层重复 monitor。
4. 由上游 env transaction 驱动下游 model。
5. 把共享配置 bus agent 移到芯片层。
6. 组合各模块 reg block 并设置 offset。
7. 把子模型句柄分发给模块 env。
8. 在芯片层新建 virtual sequencer。
9. 复用边界普通 sequence。
10. 重写系统级 virtual sequence 完成多接口协调。

### 9.5.4 常见复用失败原因

| 现象 | 根因 |
|------|------|
| env 放到芯片层后出现重复 transaction | 两个 monitor 观察同一接口 |
| 内部模块仍在驱动 | agent 没有切换 passive 或移除 input driver |
| 模块寄存器地址错误 | block model 固化了模块级基地址 |
| 配置 sequence 无法启动 | 强绑定了不存在的模块 virtual sequencer |
| callback 没执行 | 未注册、未 add 或安装时机太晚 |
| 参数化 vif get 失败 | set/get 的参数特化不一致 |
| factory override 失败 | 参数化类型或 create 方式不匹配 |
| 万能 sequence 难维护 | 参数和分支数量失控 |

---

## 本章总结（9.1-9.4）

### 学习重点排序

| 优先级 | 必须掌握 |
|--------|----------|
| 🔴 高 | callback 基类、pool、register、do、add 五步 |
| 🔴 高 | callback、sequence、factory 的适用边界 |
| 🔴 高 | 小型 sequence 与组合式测试设计 |
| 🔴 高 | block env 在芯片级的输入/输出 transaction 连接 |
| 🔴 高 | 子寄存器模型通过顶层 map 重新组合 |
| 🟡 中 | 参数化类、interface 和 factory 注册 |
| 🟡 中 | 模块级与芯片级 virtual sequencer 的边界 |
| 🟢 进阶 | callback 继承及复杂 RAL block 查找 |

### 最重要的 10 条规则

| # | 规则 | 说明 |
|---|------|------|
| 1 | callback 基类必须派生自 `uvm_callback` | 扩展方法声明为 virtual |
| 2 | callback 必须注册、实例化并加入指定实例 pool | 只定义类不会自动执行 |
| 3 | callback 只处理局部插入行为 | 不把主测试流程隐藏进去 |
| 4 | 测试主流程继续使用 sequence | 保留仲裁、嵌套与 virtual sequence 能力 |
| 5 | 一个 sequence 尽量只表达一个场景 | 复杂场景通过组合获得 |
| 6 | 参数化只用于有真实静态意义的类型/位宽 | 运行时选项使用配置对象 |
| 7 | 参数化 config_db 类型必须完全一致 | 不同参数特化是不同类型 |
| 8 | 可复用 block env 不固定芯片级 bus/RAL 资源 | 由上层创建并注入句柄 |
| 9 | 芯片级重新建立 root reg model 和 virtual sequencer | 不强行搬用模块顶层对象 |
| 10 | 普通原子 sequence 比模块 virtual sequence 更易跨层复用 | 减少对具体 p_sequencer 类型依赖 |

### 最容易错的点

| 易错点 | 正确理解 |
|--------|----------|
| callback 等于普通虚方法重写 | 错，UVM callback 还需要 pool 和实例绑定 |
| callback 类加入 factory 后会自动执行 | 错，必须 `pool::add(component, cb)` |
| callback 越多越灵活 | 过多扩展点会让控制流隐蔽且顺序难管理 |
| callback 可以完全替代 sequence | 技术上可行但不推荐，会失去激励层复用能力 |
| 强大 sequence 比多个小 sequence 更可复用 | 通常相反，大量参数和分支会增加维护成本 |
| 所有类都应参数化 | 只有静态类型或结构参数有意义时才值得 |
| 参数化类默认参数可在所有 factory 语境省略 | 应显式写出具体特化，避免类型不匹配 |
| block env 中放 bus agent 更完整 | 芯片级总线拓扑变化时会阻碍复用 |
| 芯片层保留所有模块 monitor 最保险 | 可能重复采样同一接口并降低性能 |
| 模块 virtual sequence 一定能用于芯片级 | 若绑定模块专属 virtual sequencer，通常不能直接复用 |

### 机制选择速查

| 目标 | 首选方法 |
|------|----------|
| 插入局部前后处理 | callback |
| 替换完整实现类 | factory override |
| 描述 transaction 流程 | sequence |
| 协调多接口 | virtual sequence |
| 改变运行参数 | config object/config_db |
| 适配位宽或静态类型 | 参数化类 |
| block 到 chip 复用 | env 接口化 + 上层重新组合 |
| 寄存器跨层复用 | 子 reg block + chip root map |
