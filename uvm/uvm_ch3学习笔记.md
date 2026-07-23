# UVM实战 第 3 章学习笔记：UVM 基础
> **核心结论**：UVM 以 <code>uvm_object</code> 作为大多数类的共同基础，以 <code>uvm_component</code> 构造长期存在的验证平台树；field automation 负责对象字段的通用操作，report 机制负责日志策略，config_db 负责沿 UVM 路径传递配置。
>
> **记忆主线**：先分清 object/component -> 再看 component 怎样组成树 -> 再学字段自动化 -> 再学日志过滤与行为 -> 最后掌握 config_db 的路径、覆盖优先级和调试。

---

## 3.0 本章定位
第 2 章已经搭出完整平台，第 3 章开始解释平台能够运行的基础机制。 本章五条主线：

| 章节 | 核心问题 |
|------|----------|
| 3.1 | 什么类应该是 object，什么类应该是 component |
| 3.2 | parent 和实例名如何形成 UVM 树 |
| 3.3 | 如何自动 copy、compare、print、pack、unpack |
| 3.4 | 如何过滤、重载、计数、停止和分流 UVM 日志 |
| 3.5 | config_db 如何定位组件、决定优先级并完成配置 |

本章最容易混淆的三组概念：

1. 变量名、实例名和完整路径。
2. 类型继承关系与 UVM 树的父子关系。
3. config_db 的层次优先级与同层时间优先级。

> **版本背景**：教材示例主要基于 UVM 1.1d。涉及 <code>starting_phase</code>、<code>set_config_*</code> 等内容时，应注意不同 UVM 版本的差异。

### 零基础阅读提示

- `uvm_object` 可以理解成普通数据对象，不会长期挂在验证平台树上。
- `uvm_component` 可以理解成平台里的固定设备，会组成一棵树并自动经历 phase。
- `parent` 决定组件挂在哪里，`name` 决定它在这一层叫什么。
- field automation 是帮你自动完成 copy、compare、print 等重复劳动。
- `config_db` 类似按“类型 + 路径 + 字段名”投递配置，任意一项不匹配都会取不到。

读代码时先判断当前类是 object 还是 component，再看它的创建方式和生命周期，很多困惑会自然消失。

---

## 3.1 uvm_component 与 uvm_object
### 3.1.1 uvm_component 派生自 uvm_object
<code>uvm_component</code> 与 <code>uvm_object</code> 不是并列关系。 真正的关系是：
```text
uvm_void
└── uvm_object
    ├── uvm_transaction
    │   └── uvm_sequence_item
    │       └── uvm_sequence_base
    │           └── uvm_sequence #(REQ, RSP)
    └── uvm_report_object
        └── uvm_component
            ├── uvm_driver
            ├── uvm_monitor
            ├── uvm_agent
            ├── uvm_env
            ├── uvm_scoreboard
            ├── uvm_test
            └── uvm_root
```
因此：

- 所有 component 都是 object。
- object 具有的许多基础能力，component 也继承到了。
- 不是所有 object 都是 component。
- 只有 component 能成为 UVM component 树的结点。

#### component 比普通 object 多出的核心能力

| 能力 | uvm_object | uvm_component |
|------|------------|---------------|
| factory 注册与创建 | 支持 | 支持 |
| print/copy/compare | 支持 | 继承支持，但部分操作受限制 |
| name | 支持 | 支持 |
| parent | 没有 component parent | 有 |
| UVM 树结点 | 否 | 是 |
| phase 自动执行 | 否 | 是 |
| report 层次控制 | 基础能力 | 可沿 component 层次控制 |
| config 层次查找 | 不作为普通树结点 | 支持 |

component 两个最有代表性的特性：

1. 构造时通过 <code>parent</code> 建立树形关系。
2. UVM 自动调用其 phase 方法。

#### 类型继承与结构层次不要混淆
类型继承回答：
```text
my_driver 是哪种类？
my_driver extends uvm_driver
```
结构层次回答：
```text
drv 实例挂在哪个 component 下面？
uvm_test_top.env.i_agt.drv
```
同一个 <code>my_driver</code> 类型可以在不同 agent 中创建多个实例，它们类型相同，路径不同。
### 3.1.2 常用的派生自 uvm_object 的类
除 component 分支外，验证环境中的许多短生命周期对象都继承自 <code>uvm_object</code>。

| 类别 | 常见基类 | 主要作用 |
|------|----------|----------|
| transaction | <code>uvm_sequence_item</code> | 封装一次协议事务 |
| sequence | <code>uvm_sequence</code> | 组织并产生 sequence item |
| config object | <code>uvm_object</code> | 集中保存某个组件或环境的配置 |
| register item | <code>uvm_reg_item</code> | 寄存器访问事务 |
| register model 类 | <code>uvm_reg*</code> | 描述寄存器、字段、存储器和地址映射 |
| phase object | <code>uvm_phase</code> | 表示和控制 phase |

#### transaction 应继承 uvm_sequence_item
虽然 UVM 中存在 <code>uvm_transaction</code>，用户定义的 transaction 通常仍应直接继承 <code>uvm_sequence_item</code>。
```systemverilog
class packet extends uvm_sequence_item;
    rand bit [47:0] dmac;         // 目的 MAC，rand 表示可参与随机化
    rand bit [47:0] smac;         // 源 MAC
    rand byte       payload[];    // 动态数组，长度也可由约束控制

    `uvm_object_utils(packet)     // 注册 object，后续可用 factory 创建

    function new(string name = "packet");
        super.new(name);          // object 没有 parent 参数
    endfunction
endclass
```
原因：

- <code>uvm_sequence_item</code> 已经继承 <code>uvm_transaction</code>。
- 它增加了 sequence/sequencer 握手所需信息。
- 可以直接用于 <code>uvm_sequence #(packet)</code> 和 <code>uvm_driver #(packet)</code>。

#### sequence 也是 object
```systemverilog
class packet_sequence extends uvm_sequence #(packet);
    `uvm_object_utils(packet_sequence) // sequence 本身也是 object

    function new(string name = "packet_sequence");
        super.new(name);
    endfunction

    task body();
        packet req;

        repeat (10) begin
            `uvm_do(req)          // 创建、随机化并发送一个 packet
        end
    endtask
endclass
```
sequence：

- 不进入 component 树。
- 没有 parent 参数。
- 启动时与 sequencer 建立临时关系。
- body 执行完成后，其任务生命周期结束。

#### config object 与 config_db 的区别
二者不是同一个概念。

| 名称 | 含义 |
|------|------|
| config object | 用户定义的配置对象，把多个参数组织在一个类中 |
| config_db | UVM 提供的配置传递机制 |

典型做法：
```systemverilog
class agent_config extends uvm_object;
    uvm_active_passive_enum is_active; // 控制 agent 是否包含 driver/sequencer
    int pre_num;                       // 项目自定义配置参数
    virtual my_if vif;                 // 指向 top 中的真实 interface

    `uvm_object_utils(agent_config)    // 配置对象也建议通过 factory 创建

    function new(string name = "agent_config");
        super.new(name);
    endfunction
endclass
```
然后把整个 config object 通过 config_db 传给 agent。
### 3.1.3 常用的派生自 uvm_component 的类
#### uvm_driver
driver 把 transaction 转换成 DUT 接口信号。 参数化 driver 中常用的内建成员包括：

| 成员 | 作用 |
|------|------|
| <code>seq_item_port</code> | 从 sequencer 拉取请求并完成握手 |
| <code>req</code> | 当前请求 item |
| <code>rsp</code> | 可选响应 item |
| <code>rsp_port</code> | 发布 response |

```systemverilog
class my_driver extends uvm_driver #(packet);
    `uvm_component_utils(my_driver)
    task main_phase(uvm_phase phase);
        forever begin
            seq_item_port.get_next_item(req);
            drive_one_packet(req);       // 把 req 转为接口时序
            seq_item_port.item_done();
        end
    endtask
endclass
```
#### uvm_monitor
monitor 把 DUT 接口信号恢复成 transaction。 <code>uvm_monitor</code> 相比 <code>uvm_component</code> 没有增加很多字段，但继承专用基类能明确组件语义。
```systemverilog
class my_monitor extends uvm_monitor;
    uvm_analysis_port #(packet) ap;
    `uvm_component_utils(my_monitor)
endclass
```
技术上直接继承 <code>uvm_component</code> 也能实现 monitor，但不推荐破坏语义分类。
#### uvm_sequencer
sequencer：

- 管理一个或多个 sequence。
- 仲裁 sequence 的 item 请求。
- 与 driver 的 seq_item_port 建立连接。
- 把 sequence item 转交给 driver。

```systemverilog
class my_sequencer extends uvm_sequencer #(packet);
    `uvm_component_utils(my_sequencer)
endclass
```
#### uvm_scoreboard
scoreboard 比较 reference model 和 monitor 的输出。 <code>uvm_scoreboard</code> 也没有增加大量专用 API，但能清楚表达组件职责。
#### reference model
UVM 没有专门的 <code>uvm_reference_model</code> 类。 reference model 通常直接继承 <code>uvm_component</code>。
```systemverilog
class my_model extends uvm_component;
    `uvm_component_utils(my_model)
endclass
```
它可以使用 SystemVerilog 算法，也可以通过 DPI 调用 C/C++ 模型。
#### uvm_agent
agent 封装同一协议的 sequencer、driver 和 monitor。 <code>uvm_agent</code> 提供 <code>is_active</code>：

| 值 | 结构 |
|----|------|
| <code>UVM_ACTIVE</code> | sequencer + driver + monitor |
| <code>UVM_PASSIVE</code> | monitor |

agent 的主要价值是协议封装与复用。
#### uvm_env
env 容纳平台中固定存在的组件和连接。 不同 test 通常复用同一个 env，只改变配置和 sequence。
#### uvm_test
所有具体测试用例通常继承 <code>uvm_test</code> 或项目自己的 <code>base_test</code>。 test 负责：

- 创建 env。
- 配置平台。
- 选择或启动 sequence。
- 设置超时和报告策略。

### 3.1.4 与 uvm_object 相关的宏
#### object 注册宏对照

| 类是否参数化 | 是否使用 field automation | 注册宏 |
|--------------|---------------------------|--------|
| 否 | 否 | <code>uvm_object_utils</code> |
| 是 | 否 | <code>uvm_object_param_utils</code> |
| 否 | 是 | <code>uvm_object_utils_begin/end</code> |
| 是 | 是 | <code>uvm_object_param_utils_begin/end</code> |

普通 object：
```systemverilog
class packet extends uvm_sequence_item;
    `uvm_object_utils(packet)
endclass
```
参数化 object：
```systemverilog
class data_item #(int WIDTH = 32) extends uvm_sequence_item;
    rand bit [WIDTH-1:0] data;
    `uvm_object_param_utils(data_item #(WIDTH))
endclass
```
使用字段自动化：
```systemverilog
class packet extends uvm_sequence_item;
    rand bit [31:0] data;
    `uvm_object_utils_begin(packet)
        `uvm_field_int(data, UVM_ALL_ON)
    `uvm_object_utils_end
endclass
```
begin/end 之间没有 field 宏在语法上也可以成立，但没有字段自动化收益。
### 3.1.5 与 uvm_component 相关的宏
#### component 注册宏对照

| 类是否参数化 | 是否使用 field automation | 注册宏 |
|--------------|---------------------------|--------|
| 否 | 否 | <code>uvm_component_utils</code> |
| 是 | 否 | <code>uvm_component_param_utils</code> |
| 否 | 是 | <code>uvm_component_utils_begin/end</code> |
| 是 | 是 | <code>uvm_component_param_utils_begin/end</code> |

普通 component：
```systemverilog
class my_driver extends uvm_driver #(packet);
    `uvm_component_utils(my_driver)
endclass
```
带字段自动化的 component：
```systemverilog
class my_driver extends uvm_driver #(packet);
    int pre_num;
    `uvm_component_utils_begin(my_driver)
        `uvm_field_int(pre_num, UVM_ALL_ON)
    `uvm_component_utils_end
endclass
```
component 中使用 field automation 的特殊价值之一，是配合 <code>super.build_phase</code> 自动应用同名配置字段。
### 3.1.6 uvm_component 的限制
component 虽然继承 object，但由于它是树结点，并非所有 object 操作都适合它。
#### clone 不适合 component
object 的 <code>clone()</code> 可以理解为： <code>clone = new + copy</code> object 示例：
```systemverilog
packet p1;
packet p2;
p1 = packet::type_id::create("p1");
assert(p1.randomize());
$cast(p2, p1.clone());            // clone 自己分配目标对象
```
component clone 存在问题：

- 新 component 的 parent 无法正确指定。
- 可能破坏 UVM 树的唯一名字约束。
- 无法明确应在哪个 build 阶段挂到树上。

因此不要 clone component。
#### copy 与 clone 的区别

| 操作 | 目标对象是否必须先创建 | 是否分配新对象 |
|------|------------------------|----------------|
| <code>copy</code> | 是 | 否 |
| <code>clone</code> | 否 | 是 |

component 已经被正确创建并拥有 parent 后，理论上可以对字段执行 copy；但不要用 copy 复制结构关系。
#### 同一父结点下名字必须唯一
错误示例：
```systemverilog
a1 = comp_a::type_id::create("a1", this);
a2 = comp_a::type_id::create("a1", this); // 错：同一 parent 下重名
```
正确：
```systemverilog
a1 = comp_a::type_id::create("a1", this);
a2 = comp_a::type_id::create("a2", this);
```
原因：

- child 的名字参与 UVM 路径。
- 同级重名会使路径无法唯一标识组件。

### 3.1.7 uvm_component 与 uvm_object 的二元结构
UVM 把验证元素分成两类管理。
#### 长期存在的结构
component 通常在 build_phase 创建，贯穿整个仿真：

- test。
- env。
- agent。
- sequencer。
- driver。
- monitor。
- model。
- scoreboard。

#### 按需产生的数据或行为
object 可以频繁创建、传递和销毁：

- transaction。
- sequence。
- config object。
- register item。

| 判断问题 | 是 | 否 |
|----------|----|----|
| 需要成为 UVM 树结点吗 | component | 继续判断 |
| 需要自动执行 phase 吗 | component | object |
| 只是数据、配置或临时行为吗 | object | 重新审视职责 |

> **最实用的判断**：长期存在并参与平台结构的是 component；短生命周期的数据和场景通常是 object。

---

## 3.2 UVM 的树形结构
UVM 用一棵 component 树管理验证平台。 树形结构使框架能够：

- 统一执行 phase。
- 递归设置 report 策略。
- 根据路径查找 config。
- 构造稳定的组件层次。
- 遍历和调试平台结构。

### 3.2.1 uvm_component 中的 parent 参数
标准 component 构造函数：
```systemverilog
function new(string name, uvm_component parent);
    super.new(name, parent);
endfunction
```
在父 component 中创建子 component：
```systemverilog
function void my_env::build_phase(uvm_phase phase);
    super.build_phase(phase);
    i_agt = my_agent::type_id::create(
        "i_agt",                 // child 的实例名
        this                     // parent 是当前 env
    );
endfunction
```
<code>this</code> 使 UVM 得知：

- env 拥有 child <code>i_agt</code>。
- <code>i_agt</code> 的 parent 是 env。
- env 的 child 列表中应加入 <code>i_agt</code>。

#### 成员变量关系不等于 UVM parent 关系
SystemVerilog 知道 <code>i_agt</code> 是 env 的成员变量，但 UVM 框架不会自动从语言成员关系推导 parent。 parent 必须在 component 创建时显式传递。 错误： <code>i_agt = new(&quot;i_agt&quot;, null);      // 会挂到 uvm_top，而不是当前 env</code> 正确： <code>i_agt = my_agent::type_id::create(&quot;i_agt&quot;, this);</code>
### 3.2.2 UVM 树的根
测试类 <code>uvm_test_top</code> 不是整棵 UVM 树真正的根。 真正的根是全局唯一的 <code>uvm_top</code>。
```text
uvm_top : uvm_root
└── uvm_test_top : my_caseN
    └── env : my_env
        ├── i_agt : my_agent
        │   ├── sqr
        │   ├── drv
        │   └── mon
        ├── mdl
        ├── scb
        └── o_agt
            └── mon
```
重要关系：

| 结点 | parent |
|------|--------|
| <code>uvm_top</code> | null |
| <code>uvm_test_top</code> | <code>uvm_top</code> |
| test 下的 env | test |
| env 下的 agent/model/scoreboard | env |

#### parent 为 null 的 component
若创建 component 时 parent 为 null，UVM 会把它挂到 <code>uvm_top</code> 下。 <code>extra_env = my_env::type_id::create(&quot;extra_env&quot;, null);</code> 结果类似：
```text
uvm_top
├── uvm_test_top
└── extra_env
```
这样保证系统中仍只有一棵 UVM 树。
#### 获取 uvm_root
```systemverilog
uvm_root root;
root = uvm_root::get();           // 获取全局唯一 uvm_root
```
也可以使用全局变量 <code>uvm_top</code>。 <code>uvm_root::get()</code> 体现 singleton 模式：系统中只有一个根实例。
### 3.2.3 层次结构相关函数
#### get_parent
```systemverilog
uvm_component parent;
parent = get_parent();
```
当前 component 只有一个 parent，因此不需要参数。
#### get_child
```systemverilog
uvm_component child;
child = get_child("drv");
```
一个 component 可以有多个 child，因此按实例名获取。
#### get_children
```systemverilog
uvm_component children[$];
get_children(children);           // 把所有直接 child 放入队列
foreach (children[i]) begin
    `uvm_info(
        "CHILD",
        children[i].get_full_name(),
        UVM_LOW
    )
end
```
<code>get_children</code> 只返回直接孩子，不会自动递归整个子树。
#### get_first_child 与 get_next_child
```systemverilog
string child_name;
uvm_component child;
if (get_first_child(child_name)) begin
    do begin
        child = get_child(child_name);
        child.print();
    end
    while (get_next_child(child_name));
end
```
<code>child_name</code> 是遍历状态变量：

- 不必预先赋值。
- 遍历期间不要随意修改。
- 通过 ref 在函数之间传递状态。

#### get_num_children
```systemverilog
int count;
count = get_num_children();
```
返回当前 component 的直接 child 数量。
#### 常用层次函数速查

| 函数 | 作用 |
|------|------|
| <code>get_parent()</code> | 获取父 component |
| <code>get_child(name)</code> | 按实例名获取直接 child |
| <code>get_children(queue)</code> | 获取全部直接 child |
| <code>get_first_child(name)</code> | 开始 child 迭代 |
| <code>get_next_child(name)</code> | 继续 child 迭代 |
| <code>get_num_children()</code> | 获取直接 child 数量 |
| <code>get_name()</code> | 获取当前实例名 |
| <code>get_full_name()</code> | 获取完整 UVM 路径 |

---

## 3.3 field automation 机制
field automation 通过 <code>uvm_field_*</code> 宏注册字段，并为这些字段提供通用操作。 主要能力：

- copy。
- compare。
- print。
- pack/unpack。
- record。
- component 配置字段自动应用。

### 3.3.1 field automation 机制相关的宏
#### 标量字段

| 字段类型 | 宏 |
|----------|----|
| 整数、bit、logic 等 | <code>uvm_field_int</code> |
| real | <code>uvm_field_real</code> |
| enum | <code>uvm_field_enum</code> |
| uvm_object 句柄 | <code>uvm_field_object</code> |
| event | <code>uvm_field_event</code> |
| string | <code>uvm_field_string</code> |

示例：
```systemverilog
typedef enum {GOOD_FRAME, BAD_FRAME} frame_kind_e;
class packet extends uvm_sequence_item;
    rand bit [31:0] data;
    rand frame_kind_e kind;
    string source;
    `uvm_object_utils_begin(packet)
        `uvm_field_int   (data,   UVM_ALL_ON)
        `uvm_field_enum  (frame_kind_e, kind, UVM_ALL_ON)
        `uvm_field_string(source, UVM_ALL_ON)
    `uvm_object_utils_end
endclass
```
枚举标量宏需要显式给出枚举类型。
#### 动态数组字段

| 动态数组元素 | 宏 |
|--------------|----|
| int/bit/byte 等 | <code>uvm_field_array_int</code> |
| enum | <code>uvm_field_array_enum</code> |
| object | <code>uvm_field_array_object</code> |
| string | <code>uvm_field_array_string</code> |

```systemverilog
rand byte payload[];
`uvm_field_array_int(payload, UVM_ALL_ON)
```
#### 静态数组字段

| 静态数组元素 | 宏 |
|--------------|----|
| int/bit/byte 等 | <code>uvm_field_sarray_int</code> |
| enum | <code>uvm_field_sarray_enum</code> |
| object | <code>uvm_field_sarray_object</code> |
| string | <code>uvm_field_sarray_string</code> |

#### 队列字段

| 队列元素 | 宏 |
|----------|----|
| int/bit/byte 等 | <code>uvm_field_queue_int</code> |
| enum | <code>uvm_field_queue_enum</code> |
| object | <code>uvm_field_queue_object</code> |
| string | <code>uvm_field_queue_string</code> |

#### 关联数组字段
关联数组宏名称同时编码“元素类型”和“索引类型”。 格式可理解为： <code>uvm_field_aa_&lt;value_type&gt;_&lt;index_type&gt;</code> 例如：
```systemverilog
int counters[string];
`uvm_field_aa_int_string(counters, UVM_ALL_ON)
```
表示：

- 存储值类型是 int。
- 索引类型是 string。

常见宏：

- <code>uvm_field_aa_int_string</code>。
- <code>uvm_field_aa_string_string</code>。
- <code>uvm_field_aa_object_string</code>。
- <code>uvm_field_aa_int_int</code>。
- <code>uvm_field_aa_string_int</code>。
- <code>uvm_field_aa_object_int</code>。

#### 宏选择口诀
```text
先看容器：标量 / array / sarray / queue / aa
再看元素：int / enum / object / string
若是关联数组，再看 index 类型
```
### 3.3.2 field automation 机制的常用函数
#### copy
```systemverilog
packet src;
packet dst;
src = packet::type_id::create("src");
dst = packet::type_id::create("dst");
assert(src.randomize());
dst.copy(src);                    // 调用者 dst 是复制目标
```
记忆： <code>目标.copy(来源)</code> 目标对象必须先创建。
#### compare
```systemverilog
bit same;
same = actual.compare(expected);
```
返回值：

| 结果 | 含义 |
|------|------|
| 1 | 已注册并参与比较的字段一致 |
| 0 | 至少一个比较字段不一致 |

比较是否对称取决于自定义实现和 comparer 配置；常规字段比较可按 actual.compare(expected) 理解。
#### pack_bytes 与 unpack_bytes
```systemverilog
byte unsigned bytes[];
int bit_count;
bit_count = tr.pack_bytes(bytes); // 返回打包的 bit 数
```
恢复：
```systemverilog
packet tr2;
tr2 = packet::type_id::create("tr2");
// 若 tr2 含动态数组，必要时先确定数组大小
bit_count = tr2.unpack_bytes(bytes);
```
#### pack 与 unpack

| 函数 | 流元素类型 |
|------|------------|
| <code>pack</code>/<code>unpack</code> | bit |
| <code>pack_bytes</code>/<code>unpack_bytes</code> | byte |
| <code>pack_ints</code>/<code>unpack_ints</code> | int |

所有 pack/unpack 都要关注：

- field 注册顺序。
- 大小端配置。
- 动态数组长度。
- 不应在线上传输的控制字段。

#### print
<code>tr.print();</code> print 会显示参与打印的注册字段。 调试 transaction 时比逐字段 <code>$display</code> 更统一。
#### clone
```systemverilog
packet copy_tr;
$cast(copy_tr, tr.clone());
```
clone 返回 <code>uvm_object</code> 句柄，所以常需要 <code>$cast</code> 转回具体类型。
### 3.3.3 field automation 机制中标志位的使用
有些字段需要 print/compare/copy，但不能进入协议 pack。 例如 <code>crc_err</code> 只是测试控制标志，不是以太网帧字段。
```systemverilog
class packet extends uvm_sequence_item;
    rand bit [31:0] crc;
    rand bit        crc_err;      // 控制是否故意产生错误 CRC
    function void post_randomize();
        if (!crc_err)
            crc = calc_crc();     // 正常帧计算正确 CRC
        // crc_err==1 时保留随机 CRC，制造错误激励
    endfunction
    `uvm_object_utils_begin(packet)
        `uvm_field_int(crc,     UVM_ALL_ON)
        `uvm_field_int(crc_err, UVM_ALL_ON | UVM_NOPACK)
    `uvm_object_utils_end
endclass
```
这样 <code>crc_err</code>：

- 可以被 print。
- 可以被 copy。
- 可以参与 compare。
- 不会被 pack 到 DUT 接口数据流。

#### 常用 field 标志

| 功能 | 打开 | 关闭 |
|------|------|------|
| copy | <code>UVM_COPY</code> | <code>UVM_NOCOPY</code> |
| compare | <code>UVM_COMPARE</code> | <code>UVM_NOCOMPARE</code> |
| print | <code>UVM_PRINT</code> | <code>UVM_NOPRINT</code> |
| record | <code>UVM_RECORD</code> | <code>UVM_NORECORD</code> |
| pack | <code>UVM_PACK</code> | <code>UVM_NOPACK</code> |

常见组合：
```systemverilog
`uvm_field_int(ctrl, UVM_ALL_ON | UVM_NOPACK)
```
标志本质上是位掩码，因此使用按位或 <code>|</code> 组合。 当同时存在 PACK 与 NOPACK 时，禁止标志用于覆盖对应功能。
#### 哪些字段通常不应该 pack

- 错误注入开关。
- transaction 来源标签。
- 调试计数器。
- 时间戳。
- 期望值辅助字段。
- 不属于线协议的数据。

### 3.3.4 field automation 中宏与 if 的结合
某些协议字段只在特定类型事务中出现。 例如 VLAN 帧比普通以太网帧多一组 VLAN 字段。
```systemverilog
class packet extends uvm_sequence_item;
    rand bit [47:0] dmac;
    rand bit [47:0] smac;
    rand bit [15:0] vlan_info1;
    rand bit [2:0]  vlan_info2;
    rand bit        vlan_info3;
    rand bit [11:0] vlan_info4;
    rand bit [15:0] ether_type;
    rand byte       payload[];
    rand bit [31:0] crc;
    rand bit        is_vlan;      // 控制是否包含 VLAN 字段
    `uvm_object_utils_begin(packet)
        `uvm_field_int(dmac, UVM_ALL_ON)
        `uvm_field_int(smac, UVM_ALL_ON)
        if (is_vlan) begin
            `uvm_field_int(vlan_info1, UVM_ALL_ON)
            `uvm_field_int(vlan_info2, UVM_ALL_ON)
            `uvm_field_int(vlan_info3, UVM_ALL_ON)
            `uvm_field_int(vlan_info4, UVM_ALL_ON)
        end
        `uvm_field_int      (ether_type, UVM_ALL_ON)
        `uvm_field_array_int(payload,    UVM_ALL_ON)
        `uvm_field_int      (crc,        UVM_ALL_ON)
        // 控制字段本身不属于协议包
        `uvm_field_int(is_vlan, UVM_ALL_ON | UVM_NOPACK)
    `uvm_object_utils_end
endclass
```
普通帧： <code>assert(tr.randomize() with { is_vlan == 0; });</code> VLAN 帧： <code>assert(tr.randomize() with { is_vlan == 1; });</code> 优点：

- print 时字段语义明确。
- compare 失败能定位具体 VLAN 字段。
- pack 时普通帧不会携带 VLAN 字段。

注意：

- 条件字段结构增加了宏展开复杂度。
- pack 与 unpack 两侧必须对 <code>is_vlan</code> 有一致认知。
- unpack 前需要知道当前包是否包含 VLAN 字段。

---

## 3.4 UVM 中打印信息的控制
UVM report 机制把日志分成三个正交维度：

| 维度 | 回答的问题 |
|------|------------|
| severity | 这条消息有多严重 |
| verbosity | 这条 info 有多详细 |
| action | 消息出现后执行什么行为 |

另外还可以按 ID、component 路径和日志文件进行控制。
### 3.4.1 设置打印信息的冗余度阈值
#### verbosity 等级

| 级别 | 数值 |
|------|------|
| <code>UVM_NONE</code> | 0 |
| <code>UVM_LOW</code> | 100 |
| <code>UVM_MEDIUM</code> | 200 |
| <code>UVM_HIGH</code> | 300 |
| <code>UVM_FULL</code> | 400 |
| <code>UVM_DEBUG</code> | 500 |

显示规则：
```text
消息 verbosity <= 当前阈值 -> 显示
消息 verbosity > 当前阈值  -> 不显示
```
默认阈值通常为 <code>UVM_MEDIUM</code>。 因此 LOW 和 MEDIUM 显示，HIGH 通常不显示。
#### 查询当前阈值
```systemverilog
int level;
level = env.i_agt.drv.get_report_verbosity_level();
```
#### 只设置单个 component
<code>env.i_agt.drv.set_report_verbosity_level(UVM_HIGH);</code> 只影响 driver。
#### 递归设置 component 子树
<code>env.i_agt.set_report_verbosity_level_hier(UVM_HIGH);</code> 影响：

- i_agt。
- i_agt.sqr。
- i_agt.drv。
- i_agt.mon。
- 其他后代结点。

#### 按 ID 设置 verbosity
```systemverilog
env.i_agt.drv.set_report_id_verbosity(
    "DRV_DATA",
    UVM_HIGH
);
```
只调整该 component 中 ID 为 <code>DRV_DATA</code> 的消息阈值。 递归版本：
```systemverilog
env.i_agt.set_report_id_verbosity_hier(
    "DRV_DATA",
    UVM_HIGH
);
```
#### 命令行设置
<code>&lt;sim_command&gt; +UVM_VERBOSITY=UVM_HIGH</code> 也可使用： <code>&lt;sim_command&gt; +UVM_VERBOSITY=HIGH</code> 命令行适合临时增加调试日志，不必修改源码。
#### 为什么常在 connect_phase 设置
如果需要通过 <code>env.i_agt.drv</code> 访问后代 component，这些实例必须已经在 build_phase 创建完成。 因此常在 connect_phase 或之后设置层次化 report 策略。 如果只设置当前 component 自己，可以更早调用。
### 3.4.2 重载打印信息的严重性
UVM 常见 severity：

- <code>UVM_INFO</code>。
- <code>UVM_WARNING</code>。
- <code>UVM_ERROR</code>。
- <code>UVM_FATAL</code>。

可把某种 severity 重载为另一种。
```systemverilog
env.i_agt.drv.set_report_severity_override(
    UVM_WARNING,                 // 原严重性
    UVM_ERROR                    // 新严重性
);
```
作用：driver 内所有 warning 按 error 处理。
#### 按 severity + ID 重载
```systemverilog
env.i_agt.drv.set_report_severity_id_override(
    UVM_WARNING,
    "DRV_PROTOCOL",
    UVM_ERROR
);
```
只重载特定 ID 的 warning。
#### 命令行重载
<code>+uvm_set_severity=&lt;component&gt;,&lt;id&gt;,&lt;old_severity&gt;,&lt;new_severity&gt;</code> 示例： <code>+uvm_set_severity=&quot;uvm_test_top.env.i_agt.drv,DRV_PROTOCOL,UVM_WARNING,UVM_ERROR&quot;</code> 所有 ID 可使用 <code>_ALL_</code>。
#### 使用场景

- 第三方 VIP 把项目必须禁止的情况仅报告为 warning。
- 某项 warning 在当前项目中实际意味着测试失败。
- 临时降低已知、无害消息的严重性。

不要随意把真正错误降级，否则回归结果可能出现假通过。
### 3.4.3 UVM_ERROR 到达一定数量结束仿真
大量 error 出现后继续运行可能只会产生重复日志。 设置最大退出计数：
```systemverilog
function void base_test::build_phase(uvm_phase phase);
    super.build_phase(phase);
    env = my_env::type_id::create("env", this);
    set_report_max_quit_count(5); // 第 5 次计数事件后结束仿真
endfunction
```
查询：
```systemverilog
int limit;
limit = get_report_max_quit_count();
```
教材部分版本接口名称可能显示为 <code>get_max_quit_count</code>；使用时以项目 UVM 版本定义为准。 阈值为 0 通常表示不因计数达到上限而退出。
#### 命令行设置
<code>+UVM_MAX_QUIT_COUNT=6,NO</code> 含义：

- 退出阈值为 6。
- NO 表示后续设置不允许覆盖该值。

### 3.4.4 设置计数的目标
report 消息出现后执行什么操作，由 action 决定。 默认 <code>UVM_ERROR</code> 通常包含 <code>UVM_COUNT</code>。 把 warning 也加入计数：
```systemverilog
env.i_agt.drv.set_report_severity_action(
    UVM_WARNING,
    UVM_DISPLAY | UVM_COUNT
);
```
递归设置：
```systemverilog
env.i_agt.set_report_severity_action_hier(
    UVM_WARNING,
    UVM_DISPLAY | UVM_COUNT
);
```
#### 按 ID 设置 action
```systemverilog
env.i_agt.drv.set_report_id_action(
    "DRV_PROTOCOL",
    UVM_DISPLAY | UVM_COUNT
);
```
这个 ID 的 INFO/WARNING/ERROR 都会采用该 action。
#### 按 severity + ID 设置 action
```systemverilog
env.i_agt.drv.set_report_severity_id_action(
    UVM_WARNING,
    "DRV_PROTOCOL",
    UVM_DISPLAY | UVM_COUNT
);
```
控制粒度更精确。
#### 从计数中移除 error
```systemverilog
env.i_agt.drv.set_report_severity_action(
    UVM_ERROR,
    UVM_DISPLAY
);
```
这会让 error 显示但不计入 quit count。 一般不推荐在没有明确理由时这么做。
### 3.4.5 UVM 的断点功能
把 action 设置为 <code>UVM_STOP</code>，消息出现时进入仿真器交互调试状态。
```systemverilog
env.i_agt.drv.set_report_severity_action(
    UVM_WARNING,
    UVM_DISPLAY | UVM_STOP
);
```
精确到 ID：
```systemverilog
env.i_agt.drv.set_report_severity_id_action(
    UVM_WARNING,
    "DRV_PROTOCOL",
    UVM_DISPLAY | UVM_STOP
);
```
用途：

- 第一次协议错误出现时立即检查波形和调用栈。
- 避免日志滚动后难以定位最初异常。
- 统一不同仿真器上的 report 触发断点入口。

是否真正进入可交互状态仍取决于仿真器启动方式。
### 3.4.6 将输出信息导入文件中
#### 按 severity 分文件
```systemverilog
UVM_FILE info_log;
UVM_FILE error_log;
function void base_test::connect_phase(uvm_phase phase);
    super.connect_phase(phase);
    info_log  = $fopen("info.log",  "w");
    error_log = $fopen("error.log", "w");
    env.i_agt.drv.set_report_severity_file(
        UVM_INFO,
        info_log
    );
    env.i_agt.drv.set_report_severity_file(
        UVM_ERROR,
        error_log
    );
    // UVM_LOG 必须出现在 action 中，消息才写入指定文件
    env.i_agt.drv.set_report_severity_action(
        UVM_INFO,
        UVM_DISPLAY | UVM_LOG
    );
    env.i_agt.drv.set_report_severity_action(
        UVM_ERROR,
        UVM_DISPLAY | UVM_COUNT | UVM_LOG
    );
endfunction
```
只设置 file 而不加入 <code>UVM_LOG</code>，不会得到预期文件输出。
#### 按 ID 分文件
```systemverilog
env.i_agt.drv.set_report_id_file(
    "DRV_DATA",
    driver_log
);
env.i_agt.drv.set_report_id_action(
    "DRV_DATA",
    UVM_DISPLAY | UVM_LOG
);
```
#### 按 severity + ID 分文件
```systemverilog
env.i_agt.drv.set_report_severity_id_file(
    UVM_WARNING,
    "DRV_PROTOCOL",
    warning_log
);
```
#### 关闭日志文件
```systemverilog
function void base_test::final_phase(uvm_phase phase);
    super.final_phase(phase);
    if (info_log)
        $fclose(info_log);
    if (error_log)
        $fclose(error_log);
endfunction
```
### 3.4.7 控制打印信息的行为
#### action 类型

| action | 行为 |
|--------|------|
| <code>UVM_NO_ACTION</code> | 不执行任何操作 |
| <code>UVM_DISPLAY</code> | 输出到标准输出 |
| <code>UVM_LOG</code> | 输出到已配置日志文件 |
| <code>UVM_COUNT</code> | 增加 quit count |
| <code>UVM_EXIT</code> | 退出仿真 |
| <code>UVM_CALL_HOOK</code> | 调用 report hook |
| <code>UVM_STOP</code> | 停止并进入交互模式 |

action 是位掩码，可以组合： <code>UVM_DISPLAY | UVM_COUNT | UVM_LOG</code>
#### 默认行为

| severity | 典型默认 action |
|----------|-----------------|
| UVM_INFO | UVM_DISPLAY |
| UVM_WARNING | UVM_DISPLAY |
| UVM_ERROR | UVM_DISPLAY + UVM_COUNT |
| UVM_FATAL | UVM_DISPLAY + UVM_EXIT |

#### 完全关闭某类信息
```systemverilog
env.i_agt.drv.set_report_severity_action(
    UVM_INFO,
    UVM_NO_ACTION
);
```
与提高 verbosity 阈值不同，NO_ACTION 从行为层面关闭信息。
#### report 控制函数命名规律
<code>set_report_&lt;severity/id/severity_id&gt;_&lt;verbosity/action/file&gt;</code> 若函数名以 <code>_hier</code> 结尾，通常表示递归应用到 component 子树。 严重性 override 并非所有场景都提供 hier 版本，使用时查当前 UVM API。

---

## 3.5 config_db 机制
config_db 用于在 UVM component 层次中传递配置。 典型配置：

- virtual interface。
- active/passive 模式。
- 整数阈值。
- 字符串路径。
- 枚举模式。
- config object。
- default sequence wrapper。

### 3.5.1 UVM 中的路径
#### 获取完整路径
```systemverilog
function void my_driver::build_phase(uvm_phase phase);
    super.build_phase(phase);
    `uvm_info("PATH", get_full_name(), UVM_LOW)
endfunction
```
可能输出： <code>uvm_test_top.env.i_agt.drv</code>
#### 路径由实例名决定
<code>drv = my_driver::type_id::create(&quot;driver&quot;, this);</code> 变量名仍是 <code>drv</code>，但路径最后一段是 <code>driver</code>。
```text
变量访问：i_agt.drv
UVM 路径：uvm_test_top.env.i_agt.driver
```
推荐让变量名和实例名一致： <code>drv = my_driver::type_id::create(&quot;drv&quot;, this);</code> 这样代码和日志更容易对应。
#### uvm_top 的名字通常不显示
<code>uvm_top</code> 对应根实例，但常规完整路径从 <code>uvm_test_top</code> 开始显示。 因此 config_db 绝对路径通常写： <code>uvm_test_top.env.i_agt.drv</code> 而不是显式加入 <code>__top__</code>。
### 3.5.2 set 与 get 函数的参数
#### set
```systemverilog
uvm_config_db#(int)::set(
    this,                         // 起始上下文
    "env.i_agt.drv",              // 相对 this 的目标路径
    "pre_num",                    // 字段键名
    100                           // 配置值
);
```
四个参数：

| 参数 | 含义 |
|------|------|
| 1 | context component |
| 2 | 相对 context 的实例路径 |
| 3 | field name/key |
| 4 | value |

参数 1 与参数 2 联合确定目标范围。
#### get
```systemverilog
int pre_num;
if (!uvm_config_db#(int)::get(
        this,                     // 当前 driver
        "",                       // 当前实例自身
        "pre_num",                // 必须与 set 的 key 一致
        pre_num                   // 写入此变量
    )) begin
    `uvm_fatal("NO_CFG", "pre_num was not configured")
end
```
#### set/get 必须匹配的内容

| 项目 | 要求 |
|------|------|
| 参数化类型 | 必须兼容，例如两边都用 int |
| 路径 | set 的目标必须覆盖 get 所在实例 |
| field name | 字符串必须一致 |
| 时间 | get 之前必须已有有效 set |

get 第四个变量名不必与 field name 相同，但保持同名更易维护。
#### null 等价于 uvm_root::get()
在 top_tb 中没有 <code>this</code> component，可写：
```systemverilog
uvm_config_db#(virtual my_if)::set(
    null,
    "uvm_test_top.env.i_agt.drv",
    "vif",
    input_if
);
```
可以理解为：
```systemverilog
uvm_config_db#(virtual my_if)::set(
    uvm_root::get(),
    "uvm_test_top.env.i_agt.drv",
    "vif",
    input_if
);
```
#### 灵活拆分 context 和相对路径
以下目标可表示同一个 driver：
```systemverilog
// 从 test 出发
uvm_config_db#(int)::set(
    this,
    "env.i_agt.drv",
    "pre_num",
    100
);
// 从 env 出发
uvm_config_db#(int)::set(
    env,
    "i_agt.drv",
    "pre_num",
    100
);
```
推荐选择最清晰、最符合当前代码层次的写法。
### 3.5.3 省略 get 语句
component 字段使用 field automation 注册后，<code>super.build_phase</code> 可自动应用同名配置。
```systemverilog
class my_driver extends uvm_driver #(packet);
    int pre_num;
    `uvm_component_utils_begin(my_driver)
        `uvm_field_int(pre_num, UVM_ALL_ON)
    `uvm_component_utils_end
    function new(string name = "my_driver",
                 uvm_component parent = null);
        super.new(name, parent);
        pre_num = 3;              // 本地默认值
    endfunction
    function void build_phase(uvm_phase phase);
        `uvm_info(
            "CFG",
            $sformatf("before super: pre_num=%0d", pre_num),
            UVM_LOW
        )
        super.build_phase(phase); // 自动应用匹配配置字段
        `uvm_info(
            "CFG",
            $sformatf("after super: pre_num=%0d", pre_num),
            UVM_LOW
        )
    endfunction
endclass
```
test 中仍要 set：
```systemverilog
uvm_config_db#(int)::set(
    this,
    "env.i_agt.drv",
    "pre_num",                    // 必须与成员变量名一致
    100
);
```
#### 自动 get 的三个条件

1. component 使用 begin/end 版本的注册宏。
2. 字段使用对应的 <code>uvm_field_*</code> 宏注册。
3. <code>super.build_phase(phase)</code> 被调用。
4. set 的 field name 与成员变量名一致。

set 不能省略。
#### 工程建议
虽然自动配置写法短，但显式 get 往往更清楚：
```systemverilog
if (!uvm_config_db#(int)::get(this, "", "pre_num", pre_num))
    pre_num = 3;
```
显式 get 的优势：

- 能检查返回值。
- 能清楚表达必选配置还是可选配置。
- 不依赖 field automation 隐式行为。
- 调试更直接。

### 3.5.4 跨层次的多重设置
同一个目标字段可能被多个层次 set。 在 build 阶段，UVM 主要按“设置者的层次”决定优先级。
#### 高层设置优先
test 设置：
```systemverilog
uvm_config_db#(int)::set(
    this,
    "env.i_agt.drv",
    "pre_num",
    999
);
```
env 设置：
```systemverilog
uvm_config_db#(int)::set(
    this,
    "i_agt.drv",
    "pre_num",
    100
);
```
driver 通常得到 999，因为 test 的设置层次高于 env。
```text
uvm_test_top     高层，优先级高
└── env          低一层
    └── i_agt
        └── drv
```
#### 为什么高层设置优先
可复用 env 可能自带默认配置。 具体 test 必须能覆盖 env 默认值，而不修改 env 源码。 因此：

- env 提供默认策略。
- base_test 提供项目公共配置。
- 具体 test 提供场景级覆盖。

#### context 被写成 root 时会发生什么
如果 test 和 env 都用 <code>uvm_root::get()</code> 作为 context，它们在资源层次上看起来来自同一高层。 此时更可能由写入时间决定结果。 build_phase 自顶向下：
```text
test build 先执行
env build 后执行
```
env 后写入的值可能覆盖 test 的值。 所以教材建议：
> 在 component 内调用 set 时，第一参数尽量使用 this；只有无法获得 component this 的 top_tb 才使用 null/root。
### 3.5.5 同一层次的多重设置
当两次 set 来自同一层次时，通常后写入者生效。
```systemverilog
uvm_config_db#(int)::set(
    this, "env.i_agt.drv", "pre_num", 100
);
uvm_config_db#(int)::set(
    this, "env.i_agt.drv", "pre_num", 109
);
```
最终通常得到 109。
#### base_test 默认值 + 子 test 覆盖
```systemverilog
class base_test extends uvm_test;
    function void build_phase(uvm_phase phase);
        super.build_phase(phase);
        // 绝大多数测试使用的默认值
        uvm_config_db#(int)::set(
            this,
            "env.i_agt.drv",
            "pre_num",
            7
        );
    endfunction
endclass
```
特殊测试：
```systemverilog
class long_preamble_test extends base_test;
    function void build_phase(uvm_phase phase);
        super.build_phase(phase); // 先执行默认 set
        // 同层次、后执行，覆盖默认值
        uvm_config_db#(int)::set(
            this,
            "env.i_agt.drv",
            "pre_num",
            100
        );
    endfunction
endclass
```
这个模式可以避免在大量 test 中重复公共 set。
#### 多重设置优先级速记

| 情况 | 通常优先规则 |
|------|--------------|
| build 阶段跨层次 | 越靠近根的设置越优先 |
| 同一层次重复设置 | 后写入优先 |
| 都伪装成 root context | 层次差异消失，时间更关键 |

实际项目遇到复杂组合时，应使用 trace 验证，而不是只凭记忆。
### 3.5.6 非直线的设置与获取
#### 直线设置
祖先 component 给后代设置： <code>test -&gt; env -&gt; agent -&gt; driver</code> 例如 test 设置 driver。
#### 非直线设置
兄弟分支给另一分支设置：
```text
env
├── scoreboard  --set--> driver
└── agent
    └── driver
```
示例：
```systemverilog
uvm_config_db#(int)::set(
    get_parent(),                 // scoreboard 的 parent 是 env
    "i_agt.drv",
    "pre_num",
    200
);
```
风险：

- 同级 component 的 build_phase 顺序不应作为设计依赖。
- driver get 时，scoreboard 的 set 可能尚未执行。
- 平台结构耦合变强。

> **建议**：避免非直线 set，把配置集中放在 test、env 或 config object。
#### 非直线获取
model 想读取本来设置给 driver 的配置：
```systemverilog
int drv_pre_num;
void'(uvm_config_db#(int)::get(
    get_parent(),                 // 从 env 出发
    "i_agt.drv",
    "pre_num",
    drv_pre_num
));
```
非直线 get 可减少重复 set，但也让 model 依赖 driver 路径。 更可维护的方案往往是：

- 创建共享 config object。
- 同时把 config object 传给 driver 和 model。
- 或由 env 读取一次，再显式分配给相关组件。

### 3.5.7 config_db 机制对通配符的支持
完整路径设置：
```systemverilog
uvm_config_db#(virtual my_if)::set(
    null,
    "uvm_test_top.env.i_agt.drv",
    "vif",
    input_if
);
uvm_config_db#(virtual my_if)::set(
    null,
    "uvm_test_top.env.i_agt.mon",
    "vif",
    input_if
);
```
使用通配符：
```systemverilog
uvm_config_db#(virtual my_if)::set(
    null,
    "uvm_test_top.env.i_agt*",
    "vif",
    input_if
);
```
这可同时覆盖 input agent 下的 driver 和 monitor。
#### 通配符的优点

- 减少重复 set。
- 便于给整个 agent 子树传同一配置。
- 对规则且稳定的层次较方便。

#### 通配符的风险

- 可能意外匹配新加入的 component。
- 阅读代码时不清楚真实接收者。
- 重构命名后匹配范围可能变化。
- 多个通配符配置叠加后优先级难判断。
- 调试 config 泄漏更加困难。

推荐： <code>&quot;uvm_test_top.env.i_agt*&quot;</code> 不推荐过宽： <code>&quot;*i_agt*&quot;</code>
> **原则**：通配符要尽可能保留稳定的路径前缀。
### 3.5.8 check_config_usage
config_db 路径是字符串。 拼写错误仍是合法字符串，编译器不会报错。 例如：
```systemverilog
// 错把 i_agt 写成 i_atg
uvm_config_db#(int)::set(
    this,
    "env.i_atg.drv",
    "pre_num",
    7
);
```
driver 永远读不到该值，但 set 本身不会失败。
#### 检查写入但未读取的配置
```systemverilog
function void my_test::connect_phase(uvm_phase phase);
    super.connect_phase(phase);
    check_config_usage();
endfunction
```
为什么放在 connect_phase：

- 大多数普通配置已在 build_phase get。
- build 已结束，可以检查未消费资源。

#### 注意 default_sequence 的假阳性
default sequence 配置可能在 main_phase 才被 sequencer 获取。 若 connect_phase 就调用 <code>check_config_usage()</code>，它可能被报告为“写入但尚未读取”。 因此分析结果时要结合配置的预期读取阶段。
### 3.5.9 set_config 与 get_config
教材还介绍了旧式的 <code>set/get_config_int</code>、<code>set/get_config_string</code> 和 <code>set/get_config_object</code>。它们在旧版本中可与相应类型的 config_db 操作互通，但支持的类型较少。
```systemverilog
set_config_int("env.i_agt.drv", "pre_num", 999);
void'(get_config_int("pre_num", pre_num));
```
新代码优先使用参数化 config_db，它还能传递 enum、virtual interface 和自定义 config object：
```systemverilog
uvm_config_db#(int)::set(...);
uvm_config_db#(int)::get(...);
```
命令行仍可使用 <code>+uvm_set_config_int</code> 和 <code>+uvm_set_config_string</code>；整数可带 <code>'b</code>、<code>'o</code>、<code>'d</code>、<code>'h</code> 进制前缀。
> **版本提醒**：教材基于 UVM 1.1d；UVM 1.2 后旧式 set/get_config 已不再是推荐写法。

### 3.5.10 config_db 的调试
#### print_config
```systemverilog
function void my_test::connect_phase(uvm_phase phase);
    super.connect_phase(phase);
    print_config(1);              // 1：递归打印子树可见配置
endfunction
```
参数：

| 值 | 含义 |
|----|------|
| 0 | 只打印当前 component 可见配置 |
| 1 | 递归打印整个子树 |

输出可能很长，适合定位某个字段是否对目标 component 可见。
#### UVM_CONFIG_DB_TRACE
命令行： <code>&lt;sim_command&gt; +UVM_CONFIG_DB_TRACE</code> 它会打印 config_db set/get 的查找过程。 适合分析：

- 谁写入了某个字段。
- 哪个 component 执行了 get。
- 最终匹配了哪条资源。
- 通配符如何展开。
- 多重设置中哪个值生效。

#### 推荐调试顺序

1. 在目标 component 打印 <code>get_full_name()</code>。
2. 检查 set/get 参数化类型。
3. 检查 field name 拼写。
4. 检查 set 路径是否匹配真实实例名。
5. 检查 set 是否早于 get。
6. 调用 <code>check_config_usage()</code>。
7. 用 <code>print_config(1)</code> 查看可见资源。
8. 打开 <code>+UVM_CONFIG_DB_TRACE</code>。
9. 检查是否有更高层或更晚的 set 覆盖当前值。

---

## 本章总结（3.1-3.5）
### 学习重点排序
| 优先级 | 必须掌握 |
|--------|----------|
| 🔴 高 | uvm_component 继承自 uvm_object，但只有 component 组成 UVM 树 |
| 🔴 高 | parent、实例名、变量名和完整路径的关系 |
| 🔴 高 | config_db 的类型、context、路径、field name 和 value |
| 🟡 中 | field automation 的宏、函数和控制标志 |
| 🟡 中 | verbosity、severity、ID 与 action 的区别 |
### 最重要的 10 条规则
| # | 规则 | 说明 |
|---|------|------|
| 1 | 长期平台结构用 component | 临时数据和场景通常用 object |
| 2 | transaction 继承 uvm_sequence_item | 便于使用 sequence 机制 |
| 3 | create 的 name 决定 UVM 路径 | 变量名不决定实例路径 |
| 4 | 子 component 的 parent 通常传 this | 传 null 会挂到 uvm_top |
| 5 | 同一 parent 下实例名必须唯一 | 保证路径可唯一定位 |
| 6 | 非协议控制字段使用 UVM_NOPACK | 防止错误进入线数据 |
| 7 | report 三维度分别理解 | severity、verbosity、action 含义不同 |
| 8 | component 内 set 优先用 this | 保留清晰的层次优先级 |
| 9 | base_test 设默认值，具体 test 再覆盖 | 减少重复配置 |
| 10 | config 异常先查真实路径再开 trace | 字符串错误不会被编译器发现 |
### 最容易错的点
| 易错点 | 正确理解 |
|--------|----------|
| uvm_object 与 component 并列 | 错，component 间接继承 object |
| uvm_test_top 是真正树根 | 错，真正根是 uvm_top/uvm_root |
| 跨层次 set 永远后写优先 | 错，build 阶段通常先比较层次 |
| context 全写 root 更可靠 | 错，会丢失设置者层次信息 |
| check_config_usage 报告项一定有错 | 不一定，某些配置可能尚未读取 |

