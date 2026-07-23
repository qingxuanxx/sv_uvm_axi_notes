# UVM实战 第 7 章学习笔记：UVM 中的寄存器模型
> **核心结论**：UVM 寄存器模型用 `uvm_reg_block -> uvm_reg -> uvm_reg_field` 描述 DUT 寄存器结构，用 `uvm_reg_map + adapter + sequencer` 完成前门访问，用 HDL path 完成后门访问，并通过 desired/mirrored value 模拟和检查 DUT 状态。
>
> **记忆主线**：先建模字段与寄存器 -> 加入 block 和 map -> 绑定 bus sequencer 与 adapter -> 选择 frontdoor/backdoor -> 用 predict/mirror 保持模型与 DUT 一致。

---

## 7.0 本章定位
寄存器模型也称 RAL，即 Register Abstraction Layer。
它把“寄存器叫什么、位于哪里、字段怎么访问”从具体总线协议中抽象出来。

| 章节 | 核心问题 |
|------|----------|
| 7.1 | 为什么需要寄存器模型，基本对象是什么 |
| 7.2 | 如何建立并接入最小寄存器模型 |
| 7.3 | 前门访问和后门访问如何实现 |
| 7.4 | 层次、多字段、多地址和 memory 如何建模 |
| 7.5 | desired value 与 mirrored value 如何变化 |
| 7.6 | UVM 内建寄存器测试 sequence 如何使用 |
| 7.7 | predictor、predict、randomize 和宽度扩展 |
| 7.8 | 如何查找 root block 和按地址查寄存器 |
寄存器模型解决四类问题：
1. 用统一 API 访问不同总线上的寄存器。
2. 自动维护地址、字段、访问属性和复位值。
3. 支持前门与后门两条访问路径。
4. 保存镜像状态并提供自动检查 sequence。
### 零基础阅读提示
先把寄存器模型理解成 DUT 寄存器的“验证侧通讯录 + 影子副本”：
- `uvm_reg_field` 描述某几位。
- `uvm_reg` 描述一个完整寄存器。
- `uvm_reg_block` 把多个寄存器组织起来。
- `uvm_reg_map` 保存地址以及前门访问方式。
- adapter 把通用寄存器操作翻译成项目总线 transaction。
始终区分三个值：DUT 真实值、desired value 和 mirrored value。`get/set` 只操作模型，`read/write/mirror` 才可能访问 DUT。
完整结构：
```text
test / virtual sequence / reference model
                    |
                    v
             uvm_reg_block
                    |
             uvm_reg / field
                    |
                 reg_map
                    |
             uvm_reg_adapter
                    |
              bus sequencer
                    |
               bus driver
                    |
                   DUT
```
后门路径绕过 bus driver：
```text
uvm_reg
   |
HDL path
   |
DPI/VPI
   |
DUT internal signal
```

---

## 7.1 寄存器模型简介
### 7.1.1 带寄存器配置总线的 DUT
典型 DUT 同时包含：
- 数据输入/输出接口。
- APB、AXI-Lite、I2C 或自定义寄存器配置总线。
- 控制寄存器。
- 状态寄存器。
- 计数器。
- 内部 memory。
例如 `invert` 寄存器控制数据是否按位取反：
```text
invert = 0：输出原数据
invert = 1：输出取反数据
```
没有寄存器模型时，测试必须直接构造总线 transaction：
```systemverilog
bus_transaction tr;
tr = bus_transaction::type_id::create("tr");
tr.addr    = 16'h0009;
tr.bus_op  = BUS_WRITE;
tr.wr_data = 16'h0001;
tr.start(...);                         // 概念写法
```
问题是测试代码知道太多底层细节：
- 物理地址。
- 总线 transaction 类型。
- 读写操作编码。
- 字节使能。
- 大小端。
- 返回数据位置。
使用寄存器模型后：
```systemverilog
// 不再手写物理地址和 bus item；RAL 会根据 map 发起前门总线写。
rm.invert.write(status, 16'h1, UVM_FRONTDOOR);
```
测试只表达“向 invert 写 1”。

---

### 7.1.2 需要寄存器模型才能做的事情
寄存器模型带来的主要能力：

| 能力 | 说明 |
|------|------|
| 名字访问 | 使用 `rm.ctrl.enable`，不必硬编码地址和位位置 |
| 总线无关 | 同一个寄存器 sequence 可复用到不同 bus adapter |
| 字段访问 | 按 field 读写、随机化与检查 |
| 地址映射 | 支持偏移、子 block、多 map、多地址 |
| 前门访问 | 通过真实总线验证接口和译码逻辑 |
| 后门访问 | 直接访问 DUT 内部对象，速度快 |
| 状态镜像 | 保存 desired/mirrored value |
| 批量操作 | block 级 reset、mirror、update |
| 自动测试 | 复位值、位翻转、前后门一致性检查 |
不使用寄存器模型时，参考模型如果要读取控制寄存器，常见坏味道包括：
- 依赖全局变量。
- 保存 bus sequencer 句柄并自行启动读 sequence。
- 在多个组件中重复地址常量。
- 参考模型与具体总线 transaction 强耦合。
寄存器模型本质上重新定义了验证平台访问寄存器的接口。

---

### 7.1.3 寄存器模型中的基本概念
#### `uvm_reg_field`
最小建模单位，表示一个寄存器中的字段。
例如状态寄存器：
```text
bit 0  empty
bit 1  full
bit 2  overflow
bit 3  underflow
bit 15:4 reserved
```
前四项是 field，reserved 区域通常不必建成有业务含义的 field。
#### `uvm_reg`
表示一个完整寄存器。
一个 `uvm_reg` 至少包含一个 `uvm_reg_field`。
它定义：
- 寄存器总位宽。
- 覆盖率选项。
- 字段布局。
- HDL 路径片段。
- desired/mirrored value 操作。
#### `uvm_reg_block`
寄存器容器，可包含：
- 多个 `uvm_reg`。
- 多个 `uvm_mem`。
- 多个子 `uvm_reg_block`。
- 一个或多个 `uvm_reg_map`。
一个完整寄存器模型至少有一个 root `uvm_reg_block`。
#### `uvm_reg_map`
保存寄存器的地址映射和总线属性。
它负责：
- 将 block 内偏移转换为物理地址。
- 保存总线字节宽度。
- 保存大小端。
- 关联 bus sequencer 和 adapter。
- 发起前门读写 sequence。
#### 其他关键对象
| 对象 | 作用 |
|------|------|
| `uvm_reg_adapter` | `uvm_reg_bus_op` 与用户 bus item 互转 |
| `uvm_reg_predictor` | 根据 monitor 观测结果更新镜像值 |
| `uvm_mem` | 对 memory 建模，不维护每个存储单元的镜像 |
| `uvm_reg_file` | 对一组寄存器提供逻辑命名层次 |

---

## 7.2 简单的寄存器模型
### 7.2.1 只有一个寄存器的寄存器模型
#### 定义寄存器类
```systemverilog
class reg_invert extends uvm_reg;
    `uvm_object_utils(reg_invert)
    rand uvm_reg_field reg_data;
    function new(string name = "reg_invert");
        // 第二个参数是寄存器总宽度，不是有效字段宽度。
        // 第三个参数指定是否建立寄存器覆盖率模型。
        super.new(name, 16, UVM_NO_COVERAGE);
    endfunction
    virtual function void build();
        reg_data = uvm_reg_field::type_id::create("reg_data");
        reg_data.configure(
            this,       // parent：字段属于当前寄存器
            1,          // size：字段宽度为 1 bit
            0,          // lsb_pos：最低位位于 bit 0
            "RW",       // access：可读写
            1,          // volatile：值可能由硬件自行变化
            0,          // reset：复位值为 0
            1,          // has_reset：存在已知复位值
            1,          // is_rand：允许字段参与随机化
            0           // individually_accessible：不能单独总线访问
        );
    endfunction
endclass
```
注意：`uvm_reg::build()` 不会像 component 的 `build_phase()` 那样自动调用。
必须由上层 block 手工调用。
#### `uvm_reg_field::configure` 九个参数
| 参数 | 含义 | 常见错误 |
|------|------|----------|
| parent | 所属寄存器 | 写成 block 句柄 |
| size | field 位宽 | 写成整个寄存器宽度 |
| lsb_pos | 最低位位置 | 按 1 开始计数 |
| access | 存取属性 | 与 RTL 行为不一致 |
| volatile | 是否可能被硬件改变 | 误认为“是否掉电丢失” |
| reset | 复位值 | 超过 field 位宽 |
| has_reset | 复位值是否有效 | 未知复位值仍写 1 |
| is_rand | 是否可随机化 | 忽略 access 对它的限制 |
| individually_accessible | 是否可单独访问 | 与总线 byte enable 不匹配 |
#### 常见存取属性
| 属性 | 写行为 | 读行为 |
|------|--------|--------|
| `RO` | 写无影响 | 正常读 |
| `RW` | 正常写 | 正常读 |
| `RC` | 写无影响 | 读后清零 |
| `RS` | 写无影响 | 读后置位 |
| `WC` | 写操作清零 | 正常读 |
| `WS` | 写操作置位 | 正常读 |
| `W1C` | 写 1 清零，写 0 无影响 | 正常读 |
| `W1S` | 写 1 置位，写 0 无影响 | 正常读 |
| `W1T` | 写 1 翻转，写 0 无影响 | 正常读 |
| `W0C` | 写 0 清零，写 1 无影响 | 正常读 |
| `W0S` | 写 0 置位，写 1 无影响 | 正常读 |
| `W0T` | 写 0 翻转，写 1 无影响 | 正常读 |
| `WO` | 正常写 | 读报错或无效 |
| `W1` | 复位后仅第一次写有效 | 正常读 |
| `WO1` | 复位后仅第一次写有效 | 读报错或无效 |
UVM 还组合定义了 `WRC`、`WRS`、`WSRC`、`WCRS`、`W1SRC`、`W1CRS`、`W0SRC`、`W0CRS`、`WOC`、`WOS` 等属性。
建模属性必须与 RTL 的真实副作用一致，否则 mirror 和内建 sequence 会产生误报。
#### 定义 block
```systemverilog
class reg_model extends uvm_reg_block;
    `uvm_object_utils(reg_model)
    rand reg_invert invert;
    function new(string name = "reg_model");
        super.new(name, UVM_NO_COVERAGE);
    endfunction
    virtual function void build();
        // 参数：名称、基地址、每个总线拍的字节数、大小端、byte addressing。
        default_map = create_map(
            "default_map",
            0,
            2,
            UVM_BIG_ENDIAN,
            0
        );
        invert = reg_invert::type_id::create(
            "invert",
            ,
            get_full_name()
        );
        // 参数：parent block、reg_file、HDL path。
        invert.configure(this, null, "");
        // uvm_reg 的 build 必须手工调用。
        invert.build();
        // 把寄存器加入 map：偏移地址 0x9，map 权限 RW。
        default_map.add_reg(invert, 'h9, "RW");
    endfunction
endclass
```
#### map 的地址含义
```systemverilog
// base_addr 是 map 基地址，n_bytes 是每个总线 beat 的字节数。
// 最后两个参数决定大小端和地址按 byte/word 递增。
create_map("default_map", base_addr, n_bytes, endian, byte_addressing);
```

| 参数 | 含义 |
|------|------|
| `base_addr` | map 的基地址 |
| `n_bytes` | 每个总线 beat 的字节数 |
| `endian` | 大端或小端 |
| `byte_addressing` | 地址步进按 byte 还是按 bus word |
地址错误通常来自：
- 把字节地址当成字地址。
- `n_bytes` 配错。
- 大小端配错。
- 子 block offset 重复。
- map offset 与 RTL 地址译码不一致。

---

### 7.2.2 将寄存器模型集成到验证平台中
#### 创建并锁定模型
```systemverilog
class base_test extends uvm_test;
    `uvm_component_utils(base_test)
    reg_model  rm;
    my_adapter adapter;
    function void build_phase(uvm_phase phase);
        super.build_phase(phase);
        rm = reg_model::type_id::create("rm");
        // root block 的父 block 为 null。
        // 第二个参数设置根 HDL path；暂不后门访问时可为空。
        rm.configure(null, "");
        rm.build();
        // 完成结构后锁定，建立地址查找等内部信息。
        rm.lock_model();
        // 将寄存器模型状态恢复到声明的 reset value。
        rm.reset();
        adapter = my_adapter::type_id::create("adapter");
    endfunction
endclass
```
推荐顺序：
```text
create
  -> configure
  -> build
  -> lock_model
  -> reset
  -> set_sequencer / predictor connection
```
`lock_model()` 之后不应再添加寄存器、字段或 map。
#### adapter 的职责
寄存器层使用标准的 `uvm_reg_bus_op` 描述一次总线操作。
用户 driver 接受项目自定义的 `bus_transaction`。
adapter 在两者之间转换：
```text
uvm_reg_bus_op  --reg2bus--> bus_transaction
uvm_reg_bus_op  <--bus2reg-- bus_transaction
```
定义 adapter：
```systemverilog
class my_adapter extends uvm_reg_adapter;
    `uvm_object_utils(my_adapter)
    function new(string name = "my_adapter");
        super.new(name);
        // 若 driver 返回独立 response，应设置为 1。
        provides_responses = 0;
        // 若总线 transaction 使用 byte enable，可设为 1。
        supports_byte_enable = 0;
    endfunction
    virtual function uvm_sequence_item reg2bus(
        const ref uvm_reg_bus_op rw
    );
        bus_transaction tr;
        tr = bus_transaction::type_id::create("tr");
        tr.addr = rw.addr;
        if (rw.kind == UVM_READ) begin
            tr.bus_op = BUS_READ;
        end
        else begin
            tr.bus_op  = BUS_WRITE;
            tr.wr_data = rw.data;
        end
        return tr;
    endfunction
    virtual function void bus2reg(
        uvm_sequence_item bus_item,
        ref uvm_reg_bus_op rw
    );
        bus_transaction tr;
        if (!$cast(tr, bus_item)) begin
            `uvm_fatal("ADAPTER", "bus_item type mismatch")
            return;
        end
        rw.addr = tr.addr;
        rw.kind = (tr.bus_op == BUS_READ) ? UVM_READ : UVM_WRITE;
        rw.data = (rw.kind == UVM_READ) ? tr.rd_data : tr.wr_data;
        // 必须把协议执行状态转换成 UVM 状态。
        rw.status = tr.error ? UVM_NOT_OK : UVM_IS_OK;
    endfunction
endclass
```
adapter 常见错误：
- `bus2reg()` 忘记设置 `rw.status`。
- 读操作把 `wr_data` 填回 `rw.data`。
- `reg2bus()` 忽略 byte enable。
- 地址单位转换错误。
- `provides_responses` 与 driver 行为不一致。
#### 绑定 sequencer
```systemverilog
function void base_test::connect_phase(uvm_phase phase);
    super.connect_phase(phase);

    // 告诉 default_map：前门访问在哪个总线 sequencer 上启动，
    // 以及使用哪个 adapter 在 uvm_reg_bus_op 与 bus item 之间转换。
    rm.default_map.set_sequencer(
        env.bus_agt.sqr,
        adapter
    );
endfunction
```
从此以后，寄存器模型的前门访问知道：
1. 在哪个 sequencer 上发 bus item。
2. 用哪个 adapter 转换 transaction。

---

### 7.2.3 在验证平台中使用寄存器模型
#### 读取寄存器
```systemverilog
uvm_status_e   status;
uvm_reg_data_t value;
rm.invert.read(
    status,
    value,
    UVM_FRONTDOOR
);
if (status != UVM_IS_OK)
    `uvm_error("REG", "invert read failed")
```
#### 写入寄存器
```systemverilog
rm.invert.write(
    status,
    16'h0001,
    UVM_FRONTDOOR
);
if (status != UVM_IS_OK)
    `uvm_error("REG", "invert write failed")
```
常用参数：

| 参数 | 含义 |
|------|------|
| `status` | 输出操作状态 |
| `value` | 读出值或待写值 |
| `path` | `UVM_FRONTDOOR`、`UVM_BACKDOOR` 或默认路径 |
| `map` | 指定使用哪个地址 map |
| `parent` | 父 sequence，用于 sequence 路由和层次 |
在 virtual sequence 中使用：
```systemverilog
class cfg_vseq extends uvm_sequence;
    `uvm_object_utils(cfg_vseq)
    `uvm_declare_p_sequencer(my_virtual_sequencer)
    virtual task body();
        uvm_status_e status;
        p_sequencer.p_rm.invert.write(
            status,
            16'h1,
            UVM_FRONTDOOR,
            p_sequencer.p_rm.default_map,
            this                              // 当前 sequence 作为 parent
        );
    endtask
endclass
```
寄存器 API 与当前 sequence 的 REQ 类型无关。
普通数据 sequence 中也可以执行寄存器访问，但系统级配置通常放在 virtual sequence 中更清晰。

---

## 7.3 后门访问与前门访问
### 7.3.1 UVM 中前门访问的实现
前门访问通过 DUT 对外暴露的真实寄存器总线完成。
```text
reg.read/write
      |
uvm_reg_item / uvm_reg_bus_op
      |
adapter.reg2bus
      |
bus sequencer
      |
bus driver
      |
DUT bus pins
```
读操作完整流程：
1. 用户调用 `reg.read()`。
2. 寄存器模型根据 map 计算地址。
3. 生成标准寄存器操作描述。
4. `adapter.reg2bus()` 生成 bus request。
5. request 通过 bus sequencer 交给 driver。
6. driver 驱动总线并取得读数据。
7. driver 完成 item 或返回 response。
8. `adapter.bus2reg()` 把读数据写回寄存器操作。
9. `read()` 返回数据和 status。
10. 模型按预测策略更新 desired/mirrored value。
#### `provides_responses`
若 driver 直接修改 req 并 `item_done()`：
```systemverilog
adapter.provides_responses = 0;
```
若 driver 创建独立 rsp：
```systemverilog
adapter.provides_responses = 1;
```
driver 示例：
```systemverilog
seq_item_port.get_next_item(req);
drive_bus(req);
rsp = bus_transaction::type_id::create("rsp");
rsp.set_id_info(req);                    // 保留 response 路由信息
rsp.rd_data = sampled_data;
rsp.error   = sampled_error;
seq_item_port.item_done(rsp);
```
若该配置与 driver 不一致，寄存器模型可能从错误对象中取读数据。
#### 前门访问优缺点
| 优点 | 缺点 |
|------|------|
| 验证真实总线协议 | 仿真速度慢 |
| 验证地址译码 | 初始化大量寄存器耗时 |
| 验证权限和副作用 | 调试路径较长 |
| 最接近软件访问方式 | 依赖 bus agent 正常工作 |

---

### 7.3.2 后门访问操作的定义
后门访问绕过寄存器总线，直接读写 DUT 内部变量。
```text
reg.peek/poke 或 read/write(UVM_BACKDOOR)
                     |
                  HDL path
                     |
                 DUT signal
```
适合场景：
- 快速初始化大量寄存器。
- 读取只读计数器的内部值。
- 设置难以通过前门到达的状态。
- 验证前门访问结果。
- 故障注入。
风险：
- 绕过总线，不能证明 bus 访问正确。
- deposit 可能不等价于硬件真实写行为。
- HDL 层次变化会使路径失效。
- 优化、加密或门级网表可能隐藏对象。
- W1C、RC 等副作用需要模型正确处理。

---

### 7.3.3 使用 interface 进行后门访问操作
一种广义后门方案是在 interface 中直接层次引用 DUT：
```systemverilog
interface dut_backdoor_if;
    task poke_counter(bit [31:0] value);
        top_tb.dut.counter = value;
    endtask
    task peek_counter(output bit [31:0] value);
        value = top_tb.dut.counter;
    endtask
endinterface
```
通过 virtual interface 传给验证平台：
```systemverilog
virtual dut_backdoor_if bd_vif;
if (!uvm_config_db #(virtual dut_backdoor_if)::get(
        this, "", "bd_vif", bd_vif))
    `uvm_fatal("BD", "backdoor interface not found")
```
优点：
- 逻辑直观。
- 可封装特殊读写语义。
- 不依赖完整 RAL 模型。
缺点：
- 每个寄存器可能都要单独写任务。
- HDL 路径硬编码在 interface 中。
- 可移植性和可扩展性较差。
- 难以自动遍历整个寄存器模型。

---

### 7.3.4 UVM 中后门访问操作的实现：DPI + VPI
UVM 使用 DPI 把 SystemVerilog 调用连接到 C/C++，再使用 VPI 访问 HDL 对象。
概念链路：
```text
SystemVerilog RAL
       |
    DPI-C
       |
uvm_hdl_read / uvm_hdl_deposit
       |
      VPI
       |
DUT hierarchical object
```
常见 VPI 操作：
```c
vpi_get_value(obj, p_value);
vpi_put_value(obj, p_value, p_time, flags);
```
SystemVerilog 侧接口可理解为：
```systemverilog
import "DPI-C" context function int uvm_hdl_read(
    string path,
    output uvm_hdl_data_t value
);
import "DPI-C" context function int uvm_hdl_deposit(
    string path,
    uvm_hdl_data_t value
);
```
核心价值是把 HDL 层次路径变成字符串：
```systemverilog
uvm_hdl_read("top_tb.dut.counter", value);
```
字符串可以：
- 存储在寄存器模型中。
- 按 block/field 组合。
- 作为参数传递。
- 自动遍历和检查。

---

### 7.3.5 UVM 中后门访问操作接口
#### 为寄存器设置 HDL path
block 设置根路径：
```systemverilog
rm.configure(null, "top_tb.dut");
```
寄存器添加 path slice：
```systemverilog
invert.add_hdl_path_slice(
    "invert",     // 相对于 block 根路径的 RTL 对象名
    0,            // 放入寄存器模型值的起始位
    1              // slice 宽度
);
```
若一个逻辑寄存器由多个 RTL 变量拼成：
```systemverilog
counter.add_hdl_path_slice("counter_low",  0, 16);
counter.add_hdl_path_slice("counter_high", 16, 16);
```
最终模型值：
```text
counter[15:0]  <- counter_low
counter[31:16] <- counter_high
```
#### `read/write` 的后门模式
```systemverilog
rm.counter.read(status, value, UVM_BACKDOOR);
rm.counter.write(status, 32'h1234, UVM_BACKDOOR);
```
这类操作仍按寄存器访问策略处理模型状态和字段语义。
#### `peek/poke`
```systemverilog
rm.counter.peek(status, value);
rm.counter.poke(status, 32'h1fffd);
```
概念区别：

| API | 访问方式 | 是否强调寄存器语义 |
|-----|----------|--------------------|
| `read(..., UVM_BACKDOOR)` | 后门 | 是，按 read 语义处理 |
| `write(..., UVM_BACKDOOR)` | 后门 | 是，按 write 语义处理 |
| `peek()` | 后门 | 直接观察当前值 |
| `poke()` | 后门 | 直接 deposit 指定值 |
`poke()` 可以修改 RO 字段，因为它不是软件可见总线写操作。
使用后应根据测试目标确认这是否是期望行为。

---

## 7.4 复杂的寄存器模型
### 7.4.1 层次化的寄存器模型
大型 DUT 通常按子模块划分地址空间：
```text
global block : 0x0000 - 0x0fff
buffer block : 0x1000 - 0x1fff
mac block    : 0x2000 - 0x2fff
```
顶层模型只组合子 block：
```systemverilog
class chip_reg_model extends uvm_reg_block;
    `uvm_object_utils(chip_reg_model)
    rand global_block global_blk;
    rand buffer_block buffer_blk;
    rand mac_block    mac_blk;
    function new(string name = "chip_reg_model");
        super.new(name, UVM_NO_COVERAGE);
    endfunction
    virtual function void build();
        default_map = create_map(
            "default_map", 0, 2, UVM_BIG_ENDIAN, 0
        );
        global_blk = global_block::type_id::create("global_blk");
        global_blk.configure(this, "global");
        global_blk.build();
        global_blk.lock_model();
        default_map.add_submap(global_blk.default_map, 'h0000);
        buffer_blk = buffer_block::type_id::create("buffer_blk");
        buffer_blk.configure(this, "buffer");
        buffer_blk.build();
        buffer_blk.lock_model();
        default_map.add_submap(buffer_blk.default_map, 'h1000);
        mac_blk = mac_block::type_id::create("mac_blk");
        mac_blk.configure(this, "mac");
        mac_blk.build();
        mac_blk.lock_model();
        default_map.add_submap(mac_blk.default_map, 'h2000);
    endfunction
endclass
```
关键步骤：
1. 创建子 block。
2. `configure(parent, hdl_path)`。
3. 调用子 block 的 `build()`。
4. 锁定子模型。
5. 用 `add_submap()` 加入父 map。
子 block 内寄存器只写局部偏移。
父 map 负责叠加子 block 基地址。
例如 buffer block 内偏移 `0x3` 的物理地址为 `0x1003`。

---

### 7.4.2 `reg_file` 的作用
`uvm_reg_file` 提供逻辑层次分组。
它适合：
- 多通道具有相同寄存器集合。
- 硬件存在寄存器文件概念。
- 希望层次名表达业务归属。
概念层次：
```text
reg_block
   |
reg_file[channel0]
   |-- control
   |-- status
   `-- counter
```
寄存器配置时传入所属 reg_file：
```systemverilog
ctrl.configure(
    this,             // parent block
    channel_rf,       // parent reg_file
    "ctrl"             // HDL path
);
```
`reg_file` 主要影响逻辑组织与层次，不等同于独立 address map。

---

### 7.4.3 多个域的寄存器
一个寄存器通常包含多个 field：
```systemverilog
class status_reg extends uvm_reg;
    `uvm_object_utils(status_reg)
    rand uvm_reg_field enable;
    rand uvm_reg_field mode;
         uvm_reg_field error;
    function new(string name = "status_reg");
        super.new(name, 32, UVM_NO_COVERAGE);
    endfunction
    virtual function void build();
        enable = uvm_reg_field::type_id::create("enable");
        mode   = uvm_reg_field::type_id::create("mode");
        error  = uvm_reg_field::type_id::create("error");
        enable.configure(this, 1, 0, "RW", 0, 0, 1, 1, 0);
        mode.configure  (this, 2, 1, "RW", 0, 0, 1, 1, 0);
        error.configure (this, 1, 8, "W1C", 1, 0, 1, 0, 0);
    endfunction
endclass
```
必须检查：
- field 不能重叠。
- field 不能超出寄存器总宽度。
- reset value 不能超出 field 宽度。
- reserved 位读回值要符合设计约定。
- volatile 应与硬件自更新行为一致。
字段 API：
```systemverilog
rm.status.enable.set(1);
rm.status.mode.set(2);
rm.status.update(status, UVM_FRONTDOOR);
```
字段更新往往会产生整个寄存器的读改写，应关注副作用字段。

---

### 7.4.4 多个地址的寄存器
同一个逻辑寄存器可能通过多个地址访问。
例如：
- 普通读写地址。
- set alias 地址。
- clear alias 地址。
- 不同 CPU 视图中的地址。
- 多个 map 中不同偏移。
可把同一 `uvm_reg` 加入不同 map：
```systemverilog
cpu_map.add_reg(ctrl, 'h100, "RW");
dbg_map.add_reg(ctrl, 'h020, "RW");
```
访问时显式指定 map：
```systemverilog
ctrl.read(status, value, UVM_FRONTDOOR, cpu_map);
ctrl.read(status, value, UVM_FRONTDOOR, dbg_map);
```
若寄存器位宽超过总线宽度，一个逻辑寄存器也可能占多个物理地址。
map 会根据总线宽度、端序和地址步进拆分访问。

---

### 7.4.5 加入存储器
`uvm_mem` 用于描述 memory：
```systemverilog
class buffer_block extends uvm_reg_block;
    `uvm_object_utils(buffer_block)
    uvm_mem packet_mem;
    virtual function void build();
        default_map = create_map(
            "default_map", 0, 4, UVM_LITTLE_ENDIAN, 1
        );
        packet_mem = new(
            "packet_mem",
            1024,               // depth：1024 个存储单元
            32,                 // 每个单元 32 bit
            "RW",
            UVM_NO_COVERAGE
        );
        packet_mem.configure(this, "packet_mem");
        default_map.add_mem(packet_mem, 'h400, "RW");
    endfunction
endclass
```
memory 操作需要 offset：
```systemverilog
packet_mem.write(status, 10, 32'h1234_5678);
packet_mem.read(status, 10, value);
packet_mem.poke(status, 10, 32'hffff_0000);
packet_mem.peek(status, 10, value);
```
与寄存器不同：
- `uvm_mem` 不为每个存储单元维护 desired value。
- `uvm_mem` 不为每个存储单元维护 mirrored value。
- 获取 memory 实际内容必须执行访问操作。
- 大容量 memory 不宜逐单元建立寄存器对象。

---

## 7.5 寄存器模型对 DUT 的模拟
### 7.5.1 期望值与镜像值
每个寄存器模型对象要区分三种值：

| 值 | 所在位置 | 含义 |
|----|----------|------|
| actual value | DUT | RTL 中当前真实值 |
| desired value | RAL | 希望下一次 update 写入 DUT 的值 |
| mirrored value | RAL | 模型认为 DUT 当前应该具有的值 |
三者可能暂时不一致。
例如初始状态：
```text
DUT actual = 0
desired    = 0
mirrored   = 0
```
调用 `set(1)` 后：
```text
DUT actual = 0
desired    = 1
mirrored   = 0
```
调用 `update()` 后：
```text
DUT actual = 1
desired    = 1
mirrored   = 1
```
API：
```systemverilog
rm.invert.set(16'h1);                   // 只更新 desired
desired = rm.invert.get();              // 读取 desired
mirror  = rm.invert.get_mirrored_value();
rm.invert.update(status, UVM_FRONTDOOR);
rm.invert.peek(status, actual);         // 后门读取 DUT actual
```
`get()` 不访问 DUT。
`get_mirrored_value()` 也不访问 DUT。
两者都是 0 时间的模型查询。

---

### 7.5.2 常用操作对期望值和镜像值的影响
| 操作 | 是否访问 DUT | desired | mirrored |
|------|--------------|---------|----------|
| `get()` | 否 | 返回 | 不变 |
| `set(v)` | 否 | 设为 v | 不变 |
| `get_mirrored_value()` | 否 | 不变 | 返回 |
| `read()` | 是 | 按读结果更新 | 按读结果更新 |
| `write(v)` | 是 | 按写结果更新 | 按写结果更新 |
| `peek()` | 是，后门 | 按结果更新 | 按结果更新 |
| `poke(v)` | 是，后门 | 按结果更新 | 按结果更新 |
| `update()` | 必要时写 DUT | 作为目标值 | 成功后追上 desired |
| `mirror()` | 读 DUT | 按实际值更新 | 按实际值更新 |
| `predict(v)` | 否 | 设为预测值 | 设为预测值 |
| `randomize()` | 否 | 设为随机值 | 不变 |
| `reset()` | 否 | 设为模型 reset | 设为模型 reset |
#### `update()`
```systemverilog
rm.ctrl.set('h5);
if (rm.ctrl.needs_update())
    rm.ctrl.update(status, UVM_FRONTDOOR);
```
block 级 update：
```systemverilog
rm.update(status, UVM_FRONTDOOR);
```
它会递归检查 block 中需要更新的寄存器。
#### `mirror()`
```systemverilog
rm.status.mirror(
    status,
    UVM_CHECK,       // 与旧 mirrored value 不同则报告错误
    UVM_FRONTDOOR
);
```
`UVM_NO_CHECK` 用于同步模型。
`UVM_CHECK` 用于验证 DUT 与模型预测是否一致。
#### 易失字段
volatile 字段可能由硬件自行改变。
因此：
- mirrored value 很快就可能过期。
- 严格 mirror check 可能不适合任意时刻执行。
- 应在协议定义的稳定采样点读取。
- predictor 或参考模型需跟踪其变化规律。

---

## 7.6 寄存器模型中一些内建的 sequence
### 7.6.1 检查后门访问中 HDL 路径的 sequence
`uvm_reg_mem_hdl_paths_seq` 检查已配置 HDL path 是否可访问。
```systemverilog
uvm_reg_mem_hdl_paths_seq path_seq;
path_seq = uvm_reg_mem_hdl_paths_seq::type_id::create("path_seq");
path_seq.model = rm;
// 它依赖 model，不依赖总线 sequencer。
path_seq.start(null);
```
它会：
- 遍历模型中的寄存器和 memory。
- 尝试通过 HDL path 访问对象。
- 对无法解析的路径报告错误。
- 跳过没有配置 HDL path 的对象。
应尽早运行，避免到大量后门测试时才发现路径错误。

---

### 7.6.2 检查默认值的 sequence
`uvm_reg_hw_reset_seq` 检查 DUT 复位值与模型声明是否一致。
```systemverilog
uvm_reg_hw_reset_seq reset_seq;
reset_seq = uvm_reg_hw_reset_seq::type_id::create("reset_seq");
reset_seq.model = rm;
reset_seq.start(null);
```
测试前提：
1. DUT 已真正完成硬件复位。
2. map 已绑定 sequencer 和 adapter。
3. 寄存器模型的 reset value 正确。
4. 不稳定或副作用寄存器已排除。
sequence 会按模型复位值准备镜像，再通过前门读取 DUT 并比较。
排除所有内建寄存器测试：
```systemverilog
uvm_resource_db #(bit)::set(
    {"REG::", rm.invert.get_full_name(), ".*"},
    "NO_REG_TESTS",
    1,
    this
);
```
只排除 reset test：
```systemverilog
uvm_resource_db #(bit)::set(
    {"REG::", rm.invert.get_full_name(), ".*"},
    "NO_REG_HW_RESET_TEST",
    1,
    this
);
```
常需排除：
- free-running counter。
- 读清零字段。
- 上电值受 strap 或 fuse 影响的字段。
- 写一次字段。
- 复位后很快被硬件修改的状态位。

---

### 7.6.3 检查读写功能的 sequence
#### `uvm_reg_access_seq`
检查寄存器前门和后门访问的一致性。
概念流程：
```text
frontdoor write -> backdoor read -> compare
backdoor write  -> frontdoor read -> compare
```
```systemverilog
uvm_reg_access_seq access_seq;
access_seq = uvm_reg_access_seq::type_id::create("access_seq");
access_seq.model = rm;
access_seq.start(null);
```
它要求：
- 前门 map 可工作。
- 后门 HDL path 可工作。
- 字段 access 属性正确。
- DUT 当前没有其他 master 干扰。
排除某个寄存器：
```systemverilog
uvm_resource_db #(bit)::set(
    {"REG::", rm.counter.get_full_name(), ".*"},
    "NO_REG_ACCESS_TEST",
    1,
    this
);
```
#### `uvm_mem_access_seq`
检查 memory 前后门读写一致性。
```systemverilog
uvm_mem_access_seq mem_seq;
mem_seq = uvm_mem_access_seq::type_id::create("mem_seq");
mem_seq.model = rm;
mem_seq.start(null);
```
大 memory 全遍历可能非常耗时，应根据回归等级控制范围。
#### 其他常见内建 sequence
| sequence | 目的 |
|----------|------|
| `uvm_reg_bit_bash_seq` | 对可写 bit 逐位写 0/1 并读回 |
| `uvm_reg_mem_built_in_seq` | 组合执行多种寄存器/memory 检查 |
| `uvm_mem_walk_seq` | 对 memory 执行 walking pattern |
运行内建 sequence 前必须排除不适合通用算法的特殊寄存器。

---

## 7.7 寄存器模型的高级用法
### 7.7.1 使用 `reg_predictor`
#### auto predict
前门 API 自己发起访问并根据 driver 返回结果更新 mirror：
```systemverilog
rm.default_map.set_auto_predict(1);
```
这适合只有寄存器模型一个总线 master 的简单平台。
局限：
- 其他 master 发起的寄存器访问不会经过当前 RAL 调用。
- 总线 monitor 看到的真实操作可能与 API 期望不同。
- 多主机场景会漏掉外部写入。
#### explicit prediction
使用 monitor + `uvm_reg_predictor`：
```text
bus monitor
     |
analysis_port
     |
uvm_reg_predictor
     |
adapter.bus2reg
     |
reg map / mirrored value
```
声明：
```systemverilog
uvm_reg_predictor #(bus_transaction) predictor;
my_adapter monitor_adapter;
```
创建：
```systemverilog
predictor = uvm_reg_predictor #(bus_transaction)::type_id::create(
    "predictor",
    this
);
monitor_adapter = my_adapter::type_id::create("monitor_adapter");
```
连接：
```systemverilog
function void base_test::connect_phase(uvm_phase phase);
    super.connect_phase(phase);
    rm.default_map.set_sequencer(env.bus_agt.sqr, reg_adapter);
    predictor.map     = rm.default_map;
    predictor.adapter = monitor_adapter;
    env.bus_agt.mon.ap.connect(predictor.bus_in);
    // 使用 predictor 时关闭 auto predict，避免同一访问被预测两次。
    rm.default_map.set_auto_predict(0);
endfunction
```
predictor 的优点：
- 以 monitor 观察到的实际总线行为为准。
- 能看到多个 master 的访问。
- 与总线级 scoreboard/coverage 数据源一致。
- 更适合复杂 SoC 平台。
常见错误：
- predictor 没有设置 map。
- predictor 没有设置 adapter。
- monitor analysis port 没有连接。
- monitor transaction 地址或读写数据不完整。
- auto predict 未关闭，导致重复预测。

---

### 7.7.2 `UVM_PREDICT_DIRECT` 与 mirror 操作
`predict()` 只修改模型，不访问 DUT。
```systemverilog
bit ok;
ok = rm.counter.predict(
    expected_count,
    -1,                  // byte enable 全部有效
    UVM_PREDICT_DIRECT,
    UVM_FRONTDOOR
);
if (!ok)
    `uvm_error("PRED", "counter prediction failed")
```
预测类型：

| 类型 | 含义 |
|------|------|
| `UVM_PREDICT_DIRECT` | 参考模型直接指定模型值 |
| `UVM_PREDICT_READ` | 按一次读操作语义预测 |
| `UVM_PREDICT_WRITE` | 按一次写操作语义预测 |
`read/peek/write/poke` 内部也会按操作类型触发预测。
参考模型预测计数器：
```systemverilog
task my_model::process_packet(packet tr);
    uvm_reg_data_t count;
    count = p_rm.counter.get();
    count += tr.payload.size() + 18;
    // 只更新 desired/mirrored，不写 DUT counter。
    void'(p_rm.counter.predict(count));
endtask
```
测试结束时检查：
```systemverilog
p_rm.counter.mirror(
    status,
    UVM_CHECK,
    UVM_FRONTDOOR
);
```
这形成完整闭环：
```text
reference model predicts expected counter
                    |
             RAL mirrored value
                    |
mirror reads DUT actual counter and checks
```
`mirror(UVM_NO_CHECK)` 用来同步。
`mirror(UVM_CHECK)` 用来验证。
不要把两者目的混在一起。

---

### 7.7.3 寄存器模型的随机化与 `update`
字段、寄存器和 block 可形成随机化层次：
```systemverilog
class reg_invert extends uvm_reg;
    rand uvm_reg_field reg_data;
    constraint c_default {
        reg_data.value == 0;
    }
endclass
```
可以随机化不同层级：
```systemverilog
assert(rm.randomize());
assert(rm.invert.randomize());
assert(rm.invert.reg_data.randomize());
```
字段真正参与随机化需要同时满足：
- 字段句柄声明为 `rand`。
- `configure()` 的 `is_rand` 参数为 1。
- access 类型支持有意义的写操作。
常见可随机属性包括：
```text
RW, WRC, WRS, WO, W1, WO1
```
随机化更新 desired value，不更新 mirrored value，也不访问 DUT。
因此常与 update 配合：
```systemverilog
assert(rm.randomize()) else
    `uvm_fatal("REG_RAND", "register model randomization failed")
rm.update(status, UVM_FRONTDOOR);
```
适合随机初始化可配置寄存器。
不应随机化：
- RO 状态字段。
- 硬件计数器。
- reserved 字段。
- 写后会触发不可逆动作的 command 位。
- 会关闭时钟或复位当前总线的危险控制位。

---

### 7.7.4 扩展位宽
默认数据宽度由宏控制：
```systemverilog
`ifndef UVM_REG_DATA_WIDTH
  `define UVM_REG_DATA_WIDTH 64
`endif
```
地址宽度：
```systemverilog
`ifndef UVM_REG_ADDR_WIDTH
  `define UVM_REG_ADDR_WIDTH 64
`endif
```
byte enable 宽度通常由数据宽度推导：
```systemverilog
`ifndef UVM_REG_BYTENABLE_WIDTH
  `define UVM_REG_BYTENABLE_WIDTH \
      ((`UVM_REG_DATA_WIDTH-1)/8+1)
`endif
```
若项目需要超过默认宽度，应在编译 UVM package 前统一定义这些宏。
注意：
- 所有编译单元必须看到一致宏值。
- adapter 和 bus transaction 位宽也要同步修改。
- simulator 预编译 UVM 库可能需要重新编译。
- 数据、地址和 byte enable 三者必须一致。

---

## 7.8 寄存器模型的其他常用函数
### 7.8.1 `get_root_blocks`
可查找验证平台中所有 root register block：
```systemverilog
uvm_reg_block blocks[$];
reg_model     typed_rm;
uvm_reg_block::get_root_blocks(blocks);
if (blocks.size() == 0)
    `uvm_fatal("RAL", "no root register block found")
if (!$cast(typed_rm, blocks[0]))
    `uvm_fatal("RAL", "root block type mismatch")
```
root block 是没有父 `uvm_reg_block` 的最顶层 block。
子 block 不会作为独立 root 返回。
使用注意：
- 平台可能有多个 root block，不能盲目取 `[0]`。
- 应按 `get_full_name()` 或类型筛选。
- `$cast` 必须检查返回值。
- 显式传递模型句柄通常比全局查找更清晰。

---

### 7.8.2 `get_reg_by_offset`
可通过地址从 map 查找寄存器：
```systemverilog
uvm_reg target;
target = rm.default_map.get_reg_by_offset(
    addr,
    1'b1             // read：按读访问查找
);
if (target == null) begin
    `uvm_error("RAL", $sformatf("no register at 0x%0h", addr))
end
else begin
    target.read(status, value, UVM_FRONTDOOR, rm.default_map);
end
```
层次 map 中，顶层 map 可按完整物理 offset 查找子 block 寄存器。
例如：
```text
buffer block base = 0x1000
register offset   = 0x0003
top map lookup    = 0x1003
```
多地址寄存器可用 `get_addresses()` 查看所有映射地址：
```systemverilog
uvm_reg_addr_t addresses[];
target.get_addresses(rm.default_map, addresses);
foreach (addresses[i])
    `uvm_info("RAL",
              $sformatf("address[%0d]=0x%0h", i, addresses[i]),
              UVM_LOW)
```

---

## 7.9 实际工程中的建模流程
### 7.9.1 从寄存器规格到 RAL
建议流程：
1. 统一寄存器规格源，如 IP-XACT、SystemRDL、CSV 或专用表格。
2. 自动生成 `uvm_reg`、block、map 和 HDL path 骨架。
3. 人工审查特殊 access、副作用和 volatile 属性。
4. 编译并打印模型结构。
5. 运行 HDL path 检查。
6. 运行 reset value 检查。
7. 运行可写位 bit-bash/access test。
8. 接入 predictor。
9. 在功能 sequence 中按名字访问寄存器。
10. 把特殊寄存器排除规则纳入公共配置。
寄存器数量较多时不推荐手工维护全部 RAL 类。
手工代码更适合：
- 极小模型。
- 教学和原型。
- 自定义特殊寄存器行为。
- 对自动生成模型进行扩展。
### 7.9.2 模型结构自检
可打印模型：
```systemverilog
rm.print();
```
需要核对：
- 每个寄存器名称与层次。
- 每个 field 的宽度和 lsb。
- map 基地址和 offset。
- 总线 `n_bytes`。
- endian。
- access 属性。
- reset value 与 has_reset。
- HDL path。
### 7.9.3 前门访问故障定位
当 `reg.read/write` 不工作时：
1. 检查 `status` 是否为 `UVM_IS_OK`。
2. 检查 `lock_model()` 是否已经调用。
3. 检查 map 是否 `set_sequencer()`。
4. 检查 adapter 是否为 null。
5. 打印 `reg2bus()` 输入的 addr/data/kind。
6. 检查 bus sequence 是否进入 sequencer。
7. 检查 driver 是否收到 item。
8. 检查 monitor 看到的物理地址。
9. 检查 `bus2reg()` 是否设置 data/status。
10. 检查 byte addressing 与 endian。
### 7.9.4 后门访问故障定位
当 `peek/poke` 报错时：
1. 打印寄存器完整 HDL path。
2. 检查 root block `configure()` 的路径。
3. 检查子 block 相对路径。
4. 检查 `add_hdl_path_slice()` 名称、offset 和 size。
5. 检查 RTL generate 层次和数组索引。
6. 检查仿真器是否开启 VPI/ACC 可见性。
7. 检查优化是否删除内部对象。
8. 先运行 `uvm_reg_mem_hdl_paths_seq`。
9. 对比直接 `uvm_hdl_read()` 的结果。
10. 门级仿真重新核对网表层次。
### 7.9.5 镜像不一致故障定位
当 mirror check 失败时：
- 检查字段 access 是否与 RTL 一致。
- 检查 volatile 属性。
- 检查是否存在另一个 bus master。
- 检查 auto predict 与 predictor 是否同时开启。
- 检查 monitor 是否重复发布 transaction。
- 检查 read-clear/write-one-clear 副作用。
- 检查参考模型 `predict()` 的时机和值。
- 检查复位时是否调用 `rm.reset()`。
- 检查后门 `poke()` 后模型是否按预期更新。
- 检查实际采样时刻是否稳定。

---

## 本章总结（7.1-7.8）
### 学习重点排序
| 优先级 | 必须掌握 |
|--------|----------|
| 🔴 高 | `uvm_reg_block -> uvm_reg -> uvm_reg_field` 建模层次 |
| 🔴 高 | `uvm_reg_map + adapter + bus sequencer` 前门链路 |
| 🔴 高 | `read/write` 与 `peek/poke` 的区别 |
| 🔴 高 | desired、mirrored、actual 三种值 |
| 🔴 高 | `set/update` 与 `predict/mirror` 的配合 |
| 🟡 中 | HDL path、层次 block、多 field 与 memory |
| 🟡 中 | auto predict 与 `uvm_reg_predictor` |
| 🟢 进阶 | 内建 sequence、随机寄存器配置和宽度扩展 |
### 最重要的 10 条规则
| # | 规则 | 说明 |
|---|------|------|
| 1 | 寄存器总宽度与 field 有效宽度分开建模 | `uvm_reg::new` 填总宽度，field 填有效位 |
| 2 | reg/block 的 `build()` 必须手工调用 | 它不是 component phase |
| 3 | 模型完成后调用 `lock_model()` | 锁定后不再修改结构 |
| 4 | 前门访问必须绑定 map、sequencer 和 adapter | 三者缺一不可 |
| 5 | adapter 的 `bus2reg()` 必须回填 status 和读数据 | 否则访问结果和镜像不可信 |
| 6 | 后门访问必须正确配置 HDL path | 先用 path sequence 检查 |
| 7 | `set()` 只改 desired，`update()` 才可能写 DUT | `get()` 也不会访问 DUT |
| 8 | `predict()` 只改模型，`mirror()` 会读取 DUT | 一个预测，一个校验/同步 |
| 9 | 使用 predictor 时通常关闭 auto predict | 避免同一 transaction 被预测两次 |
| 10 | 特殊寄存器必须从通用内建测试中排除 | 计数器、RC、W1C、volatile 不能盲测 |
### 最容易错的点
| 易错点 | 正确理解 |
|--------|----------|
| RAL 是寄存器 RTL 的复制品 | 错，它是验证侧抽象模型，不会自动实时同步 DUT |
| `get()` 会读取 DUT | 错，只返回 desired value |
| `get_mirrored_value()` 一定等于 DUT 当前值 | 错，它只是模型最近预测的值 |
| `set()` 后 DUT 已改变 | 错，还需 `update()` 或显式 write |
| 后门 write 与 poke 完全相同 | 错，write 强调寄存器写语义，poke 直接 deposit |
| RO 字段绝对不能修改 | 前门不能写，但后门 poke 仍可能直接修改 RTL 对象 |
| map 中 offset 就是绝对物理地址 | 不一定，还要叠加 map 和子 block 基地址 |
| 开启 auto predict 后能看到所有 master | 错，只能自动跟踪经相应 RAL 路径发起的访问 |
| memory 与 reg 一样维护镜像 | 错，memory 不维护每个单元的 desired/mirrored value |
| 内建 sequence 可以无条件全跑 | 错，特殊寄存器必须设置排除项 |
### 常用 API 速查
| API | 是否访问 DUT | 主要作用 |
|-----|--------------|----------|
| `read()` | 是 | 前门或后门读取并更新模型 |
| `write()` | 是 | 前门或后门写入并更新模型 |
| `peek()` | 是 | 后门直接观察 |
| `poke()` | 是 | 后门直接设置 |
| `get()` | 否 | 读取 desired value |
| `set()` | 否 | 修改 desired value |
| `update()` | 可能 | 把 desired 写到 DUT |
| `mirror()` | 是 | 读取 DUT，同步或比较 mirror |
| `predict()` | 否 | 人工更新 desired/mirrored |
| `reset()` | 否 | 把模型恢复为声明的复位值 |
| `randomize()` | 否 | 随机化可写 field 的 desired value |
最终记忆：
```text
field 描述位
reg 描述寄存器
block 组织层次
map 负责地址
adapter 负责协议转换
frontdoor 验证真实总线
backdoor 提供快速直达
desired 表示想写什么
mirrored 表示模型认为 DUT 是什么
predict 更新模型
mirror 读取并检查 DUT
```
