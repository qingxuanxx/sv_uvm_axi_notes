# UVM实战 第 8 章学习笔记：UVM 中的 factory 机制
> **核心结论**：factory 把“代码中请求创建的类型”与“运行时真正创建的类型”分离。只要类已注册、使用 `type_id::create()` 创建且替代类型具有正确继承关系，测试就能在不修改原验证平台结构的情况下替换 transaction、sequence 或 component。
>
> **记忆主线**：注册类型 -> 设置 override -> 调用 factory create -> factory 查询替换表 -> 创建最终类型 -> 通过虚方法表现派生类行为。

---

## 8.0 本章定位

| 章节 | 核心问题 |
|------|----------|
| 8.1 | SystemVerilog 本身支持哪些重载 |
| 8.2 | factory 如何进行 type/instance override |
| 8.3 | transaction、sequence、component 应怎样选择重载点 |
| 8.4 | factory 如何根据类型或字符串创建对象 |
factory 主要提供两项能力：
1. 根据类型包装器或字符串创建对象。
2. 创建前查询 override 记录，决定最终实例类型。

```text
source code requests base_type
             |
             v
       factory lookup
        /           \
 no override       has override
      |                 |
 create base       create derived
```

factory 不是 C++ 意义上的函数重载，也不是约束重载。
它是一种“实例创建替换机制”。

### 零基础阅读提示

- 代码里写的是“请求创建什么类型”，factory 决定“最终实际创建什么类型”。
- `new()` 直接创建指定类型，不查询 factory。
- `type_id::create()` 会先查询 override 表。
- type override 影响所有匹配类型，instance override 只影响指定路径。
- 句柄仍可声明为父类，因为替代类必须继承原类。

读 override 代码时按三步走：先找原类型，再找替代类型，最后确认对象是不是通过 factory 创建。

---

## 8.1 SystemVerilog 对重载的支持

### 8.1.1 任务与函数的重载
教材所说的“重载”在这里更接近面向对象中的 override，即子类重写父类虚方法。

```systemverilog
class bird extends uvm_object;
    `uvm_object_utils(bird)       // 注册父类，factory 才能识别 bird
    function new(string name = "bird");
        super.new(name);
    endfunction
    virtual function void hungry();
        // virtual：通过父类句柄调用时，仍会执行实际对象的版本。
        $display("I am a bird, I am hungry");
    endfunction
    function void hungry2();
        // 没有 virtual：通过 bird 句柄调用时固定执行本版本。
        $display("I am a bird, I am hungry2");
    endfunction
endclass
```

```systemverilog
class parrot extends bird;
    `uvm_object_utils(parrot)     // 替代类型也必须注册
    function new(string name = "parrot");
        super.new(name);
    endfunction
    virtual function void hungry();
        // 重写父类虚方法，展示运行时多态。
        $display("I am a parrot, I am hungry");
    endfunction
    function void hungry2();
        $display("I am a parrot, I am hungry2");
    endfunction
endclass
```

通过父类句柄调用：

```systemverilog
function void print_hungry(bird b_ptr);
    b_ptr.hungry();       // virtual：按对象实际类型动态分派
    b_ptr.hungry2();      // non-virtual：按句柄静态类型调用 bird 版本
endfunction
```

若 `b_ptr` 指向 `parrot`：

```text
hungry()  -> parrot::hungry()
hungry2() -> bird::hungry2()
```

#### 虚方法的价值
- 调用者只依赖基类接口。
- 子类可以修改行为。
- UVM 可统一用 `uvm_component` 句柄遍历组件树。
- phase 调度不需要知道每个节点的具体类型。
例如 UVM 内部可概念化为：

```systemverilog
uvm_component c_ptr;
// c_ptr 可能实际指向 my_driver、my_monitor 或 my_env。
c_ptr.build_phase(phase);
```

因为 phase 方法是 virtual，最终调用对象实际类型的实现。

#### 重写规则
- 父类方法必须声明为 `virtual`。
- 子类方法签名应与父类一致。
- 返回类型、参数类型和参数方向应匹配。
- 需要保留父类行为时调用 `super.method()`。
- 不要因为同名就误认为发生了动态分派。

```systemverilog
virtual function void build_phase(uvm_phase phase);
    super.build_phase(phase);             // 保留父类建造逻辑
    // 添加子类行为
endfunction
```

---

### 8.1.2 约束的重载
SystemVerilog 允许子类用同名 constraint 替换父类 constraint。
基础 transaction：

```systemverilog
class my_transaction extends uvm_sequence_item;
    `uvm_object_utils(my_transaction)
    rand bit crc_err;
    rand bit sfd_err;
    rand bit pre_err;
    constraint default_cons {
        crc_err == 0;
        sfd_err == 0;
        pre_err == 0;
    }
endclass
```

CRC 异常 transaction：

```systemverilog
class crc_err_transaction extends my_transaction;
    `uvm_object_utils(crc_err_transaction)
    // 与父类同名，因此替换父类版本，而不是二者同时生效。
    constraint default_cons {
        crc_err == 1;
        sfd_err == 0;
        pre_err == 0;
    }
endclass
```

若子类使用不同约束名，则父子约束同时生效：

```systemverilog
class bad_transaction extends my_transaction;
    constraint crc_cons {
        crc_err == 1;
    }
endclass
```

如果父类 `default_cons` 约束 `crc_err == 0`，上述约束集合无解。

#### 为什么正常包不宜只用极低错误概率

```systemverilog
constraint almost_normal {
    // 权重极小不等于永远不会发生，长回归中仍可能随机出错误包。
    crc_err dist {0 := 999_999_999, 1 := 1};
}
```

极低概率不等于不可能。
正常功能测试若要求绝不注入错误，应使用硬约束 `crc_err == 0`。
概率约束适合随机压力测试，不适合定义绝对合法性。

#### `constraint_mode()` 方案

```systemverilog
tr.default_cons.constraint_mode(0);       // 关闭默认正常约束
assert(tr.randomize() with {
    crc_err == 1;
}) else
    `uvm_fatal("RAND", "CRC error transaction randomize failed")
```

对比：

| 方法 | 优点 | 风险 |
|------|------|------|
| 同名约束重载 | 场景类型清晰，适合 factory | 必须理解同名替换规则 |
| `constraint_mode(0)` | 灵活，可在运行时切换 | 容易忘记恢复或关闭错误对象 |
| `randomize() with` | 局部直观 | 多处重复场景约束 |
| dist 约束 | 适合概率注错 | 无法保证正常测试绝不出错 |

---

## 8.2 使用 factory 机制进行重载

### 8.2.1 factory 机制式的重载
设置 type override：

```systemverilog
function void my_test::build_phase(uvm_phase phase);
    bird bird_inst;
    super.build_phase(phase);
    // 所有后续通过 factory 请求 bird 的地方，改为创建 parrot。
    set_type_override_by_type(
        bird::get_type(),
        parrot::get_type()
    );
    bird_inst = bird::type_id::create("bird_inst");
endfunction
```

虽然句柄声明为 `bird`，实际对象是 `parrot`。

```text
static handle type : bird
requested type     : bird
factory result     : parrot
actual object type : parrot
```

调用 virtual 方法时表现为 `parrot` 行为。

#### factory override 生效的四个前提
1. 原类型和替代类型都注册到 factory。
2. 原对象使用 factory 的 `create()` 创建，而不是直接 `new()`。
3. 替代类型必须派生自原类型。
4. object 与 component 不能互相替代。
注册 object：

```systemverilog
class bird extends uvm_object;
    `uvm_object_utils(bird)
endclass
```

注册 component：

```systemverilog
class my_driver extends uvm_driver #(my_transaction);
    `uvm_component_utils(my_driver)
endclass
```

正确创建：

```systemverilog
bird_inst = bird::type_id::create("bird_inst");
```

绕过 factory：

```systemverilog
bird_inst = new("bird_inst");             // override 不会生效
```

继承关系：

```text
bird
  `-- parrot       合法：bird -> parrot
bird       bear
  无继承关系       非法：bird -> bear
```

替代结果必须能够赋给原句柄，否则 factory 创建后 cast 会失败。

#### 为什么 object 和 component 不能互换
object 构造形式：

```systemverilog
function new(string name = "obj");
```

component 构造形式：

```systemverilog
function new(string name, uvm_component parent);
```

两者创建接口、层次语义和生命周期不同。

---

### 8.2.2 重载的方式及种类
factory override 有两个维度：

```text
范围：type override / instance override
标识：by_type / by_name
```

#### type override by type

```systemverilog
set_type_override_by_type(
    my_monitor::get_type(),
    new_monitor::get_type()
);
```

作用于所有匹配的 factory create 请求。
适合：
- 整个平台统一替换 transaction。
- 所有同类 monitor 使用增强版本。
- 某 test 全局更换 sequence 策略。

#### instance override by type

```systemverilog
set_inst_override_by_type(
    "env.o_agt.mon",
    my_monitor::get_type(),
    new_monitor::get_type()
);
```

只替换指定实例路径。
适合：
- 多个相同 agent 中只修改一个通道。
- 对某个 scoreboard 或 monitor 注入特殊行为。
- 保持其余实例使用基础类型。
实例路径通常相对于调用 override 的 component。
若在 test 中调用，`"env.o_agt.mon"` 会与 test 完整路径组合。

#### type override by name

```systemverilog
set_type_override("my_monitor", "new_monitor");
```

#### instance override by name

```systemverilog
set_inst_override(
    "env.o_agt.mon",
    "my_monitor",
    "new_monitor"
);
```

按类型与按字符串对比：

| 方式 | 优点 | 缺点 |
|------|------|------|
| by type | 编译期检查较强，重构更可靠 | 写法稍长 |
| by name | 可由命令行或配置文本驱动 | 拼写错误常到运行时才暴露 |
工程代码优先使用 by type。
命令行和动态工具场景再使用 by name。

#### 命令行重载
类型重载概念格式：

```text
+uvm_set_type_override=my_monitor,new_monitor
```

实例重载概念格式：

```text
+uvm_set_inst_override=my_monitor,new_monitor,uvm_test_top.env.o_agt.mon
```

命令行 override 便于不改源码快速试验，但回归脚本必须记录参数。

#### type override 与 instance override 优先级
若同一次创建同时命中 instance 和 type override，instance override 更具体，通常优先。
调试时不能只看 type 表，还要核对实例路径。

---

### 8.2.3 复杂的重载

#### 连续重载

```systemverilog
set_type_override_by_type(
    bird::get_type(),
    parrot::get_type()
);
set_type_override_by_type(
    parrot::get_type(),
    big_parrot::get_type()
);
```

创建 `bird` 时解析链：

```text
bird -> parrot -> big_parrot
```

最终创建 `big_parrot`。
连续重载的每一步都应保持最终类型与最初请求类型兼容。

#### 替换已有 override

```systemverilog
set_type_override_by_type(
    bird::get_type(),
    parrot::get_type()
);
set_type_override_by_type(
    bird::get_type(),
    sparrow::get_type(),
    1                    // replace=1：替换之前 bird 的 override
);
```

最终：

```text
bird -> sparrow
```

若 `replace=0`，已有记录通常保留，后设置不会取代它。

#### 复杂 override 的风险
- override 链过长，难以判断最终类型。
- 基类 test 与派生 test 重复设置同一类型。
- instance override 路径写错后悄悄退回 type override。
- 字符串 override 拼写错误。
- 替代类型虽然继承合法，但行为假设不兼容。
- 循环 override 会触发 factory 报错。
设计建议：
- 每个 test 尽量只设置必要 override。
- override 靠近测试意图定义处。
- 注释“为什么替换”，不只写“替换什么”。
- 复杂场景启动时打印 factory 配置。

---

### 8.2.4 factory 机制的调试

#### 查看某次创建的解析结果

```systemverilog
factory.print_override_info(
    "my_monitor",
    "uvm_test_top.env.o_agt.mon"
);
```

按类型调试：

```systemverilog
factory.debug_create_by_type(
    my_monitor::get_type(),
    "uvm_test_top.env.o_agt.mon",
    "mon"
);
```

按名字调试：

```systemverilog
factory.debug_create_by_name(
    "my_monitor",
    "uvm_test_top.env.o_agt.mon",
    "mon"
);
```

这些接口用于显示解析过程，不一定真正保留创建对象。

#### 打印 factory 配置

```systemverilog
factory.print();
```

或：

```systemverilog
uvm_factory::get().print();
```

重点查看：
- Instance Overrides。
- Type Overrides。
- Requested Type。
- Override Type。
- Instance Path。
- 是否存在重复或未使用记录。

#### 打印 UVM 拓扑

```systemverilog
function void base_test::end_of_elaboration_phase(uvm_phase phase);
    super.end_of_elaboration_phase(phase);
    uvm_top.print_topology();
endfunction
```

factory 表说明“应该创建什么”，topology 说明“实际建出了什么 component”。
object 不在 component topology 中，需要通过类型名或对象打印确认。

#### override 不生效的排查顺序
1. 确认 override 在目标对象 create 之前设置。
2. 确认原类和替代类均已注册。
3. 确认目标使用 `type_id::create()` 而不是 `new()`。
4. 确认替代类派生自原类。
5. 确认 object/component 类别一致。
6. 确认 instance path 完整且层次正确。
7. 检查更具体的 instance override。
8. 检查后续 override 是否 replace 了当前记录。
9. 使用 `debug_create_by_type()`。
10. 打印 topology 确认 component 实际类型。

---

## 8.3 常用的重载

### 8.3.1 重载 transaction
基础 sequence 保持不变：

```systemverilog
class normal_sequence extends uvm_sequence #(my_transaction);
    `uvm_object_utils(normal_sequence)
    virtual task body();
        repeat (10) begin
            // 必须由 factory 创建，transaction override 才会生效。
            `uvm_do(req)
        end
    endtask
endclass
```

定义异常 transaction：

```systemverilog
class crc_err_tr extends my_transaction;
    `uvm_object_utils(crc_err_tr)
    constraint default_cons {
        crc_err == 1;
        sfd_err == 0;
        pre_err == 0;
    }
endclass
```

test 中替换：

```systemverilog
function void crc_test::build_phase(uvm_phase phase);
    super.build_phase(phase);
    set_type_override_by_type(
        my_transaction::get_type(),
        crc_err_tr::get_type()
    );
endfunction
```

优点：
- 原 sequence 不修改。
- 所有创建该 transaction 的位置统一切换。
- 异常约束集中在派生类中。
风险：
- 替换范围可能过大。
- monitor、model 等若也 factory create 同类对象，可能受到影响。
- sequence 若使用 `new()`，override 失效。
只想替换某个 sequence 中的 item 时，应考虑 instance override 或派生 sequence。

---

### 8.3.2 重载 sequence
父 sequence 嵌套原子 sequence：

```systemverilog
class case_sequence extends uvm_sequence #(my_transaction);
    `uvm_object_utils(case_sequence)
    virtual task body();
        normal_sequence nseq;
        repeat (10)
            `uvm_do(nseq)
    endtask
endclass
```

定义异常原子 sequence：

```systemverilog
class abnormal_sequence extends normal_sequence;
    `uvm_object_utils(abnormal_sequence)
    virtual task body();
        req = my_transaction::type_id::create("req");
        req.default_cons.constraint_mode(0);
        `uvm_rand_send_with(req, {
            crc_err == 1;
        })
    endtask
endclass
```

替换：

```systemverilog
set_type_override_by_type(
    normal_sequence::get_type(),
    abnormal_sequence::get_type()
);
```

`case_sequence` 不变，但其内部创建的 `normal_sequence` 变成异常版本。
适合：
- 流程本身发生变化。
- 不只是 transaction 约束变化。
- 需要插入等待、同步或多次发送。

---

### 8.3.3 重载 component
可以替换 driver、monitor、scoreboard 或 reference model。
CRC 注错 driver：

```systemverilog
class crc_driver extends my_driver;
    `uvm_component_utils(crc_driver)
    virtual function void inject_crc_err(my_transaction tr);
        // 故意破坏已经计算好的 CRC。
        tr.crc ^= 32'h1;
    endfunction
    virtual task main_phase(uvm_phase phase);
        forever begin
            seq_item_port.get_next_item(req);
            inject_crc_err(req);
            drive_one_pkt(req);
            seq_item_port.item_done();
        end
    endtask
endclass
```

test 中替换：

```systemverilog
set_type_override_by_type(
    my_driver::get_type(),
    crc_driver::get_type()
);
```

component override 适合修改：
- 引脚级异常时序。
- driver 编码或物理层错误。
- monitor 采样策略。
- scoreboard 比较策略。
- reference model 的异常算法。
如果只改变 item 内容，transaction 或 sequence override 通常更合适。

#### reference model 拆分
一个覆盖所有异常的 model 可能充满分支。
可以保留正常 model，再为少量特殊 case 派生异常 model。

```systemverilog
set_type_override_by_type(
    normal_model::get_type(),
    crc_error_model::get_type()
);
```

这样每个模型职责更清晰，但公共算法仍应放在基类中复用。

---

### 8.3.4 重载 driver 以实现所有测试用例
理论上可以把激励生成、数量控制、objection 和驱动都放回 driver，再通过 factory 替换 driver 实现不同 case。
教材明确不推荐这样做。
原因：
1. 破坏 sequence 与 driver 的职责分离。
2. driver 会同时包含场景策略和协议时序。
3. 无法充分利用嵌套 sequence 的复用能力。
4. 多 driver 同步比 virtual sequence 更困难。
5. 每个 case 都替换 driver 会产生大量重复代码。
推荐选择：

| 变化位置 | 首选机制 |
|----------|----------|
| item 字段和约束 | transaction override |
| transaction 发送流程 | sequence override |
| 多接口系统流程 | virtual sequence |
| 引脚级异常或协议驱动 | driver override |
| 比较或预测算法 | model/scoreboard override |
factory 是工具，不是让所有变化都集中到 component override 的理由。

---

## 8.4 factory 机制的实现

### 8.4.1 创建一个类实例的方法
普通创建：

```systemverilog
A a;
a = new();
```

参数化容器创建：

```systemverilog
class holder #(type T = uvm_object);
    T item;
    function new();
        item = new();
    endfunction
endclass
```

语言原生 `new()` 需要在编译时知道具体类型。
它不能直接根据运行时字符串 `"A"` 推导类型并实例化。
factory 通过注册表和代理对象补充这一能力。

---

### 8.4.2 根据字符串创建一个类
简化的 registry 思路：

```systemverilog
class registry #(type T = uvm_object, string Tname = "");
    string name = Tname;
    virtual function uvm_object create_object(string inst_name);
        T obj;
        obj = new(inst_name);
        return obj;
    endfunction
endclass
```

每个已注册类型拥有一个代理对象：

```text
"my_transaction" -> registry #(my_transaction)
"my_driver"      -> registry #(my_driver)
```

调用 `uvm_object_utils` 等宏时，主要完成：
- 定义 `type_id` registry 类型。
- 提供 `get_type()`。
- 提供 `get_type_name()`。
- 注册创建代理。
- 支持 factory create 和 override。
概念创建过程：

```text
requested string
      |
lookup registry table
      |
get proxy object
      |
proxy creates concrete object
      |
return as uvm_object/component
      |
caller casts to expected type
```

---

### 8.4.3 用 factory 创建实例的接口

#### 创建 object by name

```systemverilog
uvm_object raw_obj;
my_transaction tr;
raw_obj = factory.create_object_by_name(
    "my_transaction",
    "",
    "tr"
);
if (!$cast(tr, raw_obj))
    `uvm_fatal("FACTORY", "created object has wrong type")
```

#### 创建 object by type

```systemverilog
uvm_object raw_obj;
my_transaction tr;
raw_obj = factory.create_object_by_type(
    my_transaction::get_type(),
    "",
    "tr"
);
if (!$cast(tr, raw_obj))
    `uvm_fatal("FACTORY", "object cast failed")
```

日常代码更简洁：

```systemverilog
tr = my_transaction::type_id::create("tr");
```

#### 创建 component by name

```systemverilog
uvm_component raw_comp;
my_scoreboard scb;
raw_comp = factory.create_component_by_name(
    "my_scoreboard",
    get_full_name(),
    "scb",
    this
);
if (!$cast(scb, raw_comp))
    `uvm_fatal("FACTORY", "component cast failed")
```

#### 创建 component by type

```systemverilog
uvm_component raw_comp;
raw_comp = factory.create_component_by_type(
    my_scoreboard::get_type(),
    get_full_name(),
    "scb",
    this
);
```

日常代码：

```systemverilog
scb = my_scoreboard::type_id::create("scb", this);
```

component 应在父 component 的 `build_phase()` 或更早创建。
connect 完成后再动态插入 component 会破坏 UVM 生命周期和拓扑。

#### 四种底层接口对比

| 接口 | 目标 | 标识方式 | 关键附加参数 |
|------|------|----------|--------------|
| `create_object_by_name` | object | 字符串 | path、name |
| `create_object_by_type` | object | wrapper | path、name |
| `create_component_by_name` | component | 字符串 | path、name、parent |
| `create_component_by_type` | component | wrapper | path、name、parent |

---

### 8.4.4 factory 机制的本质
factory 可以理解为对 `new()` 的增强封装。
原始 `new()`：

```text
compile-time concrete type -> create exactly that type
```

factory create：

```text
requested type/name
      |
resolve instance override
      |
resolve type override chain
      |
find registered proxy
      |
proxy calls concrete constructor
      |
return compatible handle
```

factory 的代价：
- 注册和查表增加复杂度。
- 错误可能推迟到运行时。
- 多层 override 增加调试成本。
- 字符串接口缺少编译期保护。
factory 的收益：
- 验证平台结构与测试差异解耦。
- 基类环境可以保持稳定。
- 派生 test 只声明差异。
- VIP 用户可以扩展而不修改 VIP 源码。
- `run_test()` 可按字符串创建 test。

---

## 8.5 factory 设计与调试实践

### 8.5.1 设置 override 的时机
override 必须发生在目标对象创建之前。
推荐在 test 的 `build_phase()` 中先设置，再调用会创建目标的父类逻辑时要特别谨慎。

```systemverilog
function void derived_test::build_phase(uvm_phase phase);
    // 如果 base build 会创建目标对象，应先设置 override。
    set_type_override_by_type(base_tr::get_type(), error_tr::get_type());
    super.build_phase(phase);
endfunction
```

若 `super.build_phase()` 先完成对象创建，后设置 override 对已存在对象无效。
具体顺序要根据父类实现决定，不能机械地认为 `super` 永远第一句。

### 8.5.2 factory 与 config_db 的分工

| 需求 | 使用机制 |
|------|----------|
| 改变对象类型/算法实现 | factory override |
| 修改对象参数 | config_db 或配置对象 |
| 只改变 transaction 字段 | sequence 约束 |
| 改变层次连接关系 | env/test 的 build/connect 逻辑 |
不要为一个布尔配置创建大量派生类。
也不要用 config_db 模拟复杂多态算法。

### 8.5.3 可维护的 override 原则
- 基类应提供稳定接口和默认行为。
- 派生类只覆盖必要行为。
- 优先 type-safe 的 by-type API。
- instance override 路径使用明确常量或集中封装。
- 每个 override 都应有针对性的测试。
- 回归日志记录最终 factory 配置。
- 不依赖复杂的隐式 override 链。

---

## 本章总结（8.1-8.4）

### 学习重点排序

| 优先级 | 必须掌握 |
|--------|----------|
| 🔴 高 | factory override 生效的四个前提 |
| 🔴 高 | `new()` 与 `type_id::create()` 的区别 |
| 🔴 高 | type override 与 instance override |
| 🔴 高 | transaction、sequence、component 重载点选择 |
| 🟡 中 | 连续重载、replace 参数与调试接口 |
| 🟡 中 | by-type 与 by-name 的差异 |
| 🟢 进阶 | registry、proxy 和字符串创建原理 |

### 最重要的 10 条规则

| # | 规则 | 说明 |
|---|------|------|
| 1 | 要 override 的类必须注册 factory | 使用正确的 utils 宏 |
| 2 | 目标必须通过 factory create | 直接 `new()` 绕过 override |
| 3 | 替代类必须派生自原类 | 保证返回句柄类型兼容 |
| 4 | object 与 component 不互相替代 | 构造接口和生命周期不同 |
| 5 | override 必须在 create 之前设置 | 对已存在对象不会追溯生效 |
| 6 | 全局变化用 type override | 所有匹配请求统一替换 |
| 7 | 局部变化用 instance override | 精确路径比 type override 更具体 |
| 8 | 工程代码优先使用 by-type API | 编译期检查更可靠 |
| 9 | 复杂 override 必须打印和调试 | 核对解析链及最终类型 |
| 10 | 按变化所在层选择重载对象 | 字段、流程、引脚行为分别处理 |

### 最容易错的点

| 易错点 | 正确理解 |
|--------|----------|
| factory override 等于虚方法重写 | 错，一个替换实例类型，一个决定方法动态分派 |
| 注册了类就会自动 override | 错，还要设置记录并使用 factory create |
| `new()` 也会查询 override 表 | 错，`new()` 直接创建声明类型 |
| 父类可以替换子类 | 通常不合法，返回对象不能赋给更具体的子类句柄 |
| type override 只影响一个实例 | 错，它影响所有匹配的 create 请求 |
| instance path 从任意位置都写完整路径 | 要结合调用 override 的上下文判断相对路径 |
| 后设置的 override 一定覆盖前者 | 还取决于 `replace` 和具体匹配规则 |
| topology 能显示所有 object | 错，topology 只显示 component 树 |
| 所有异常测试都应重载 driver | 错，应根据变化层级选择 transaction/sequence/driver |
| 字符串创建与 by-type 一样安全 | 错，字符串拼写错误通常运行时才发现 |

### API 速查

| 需求 | API |
|------|-----|
| 全局按类型替换 | `set_type_override_by_type()` |
| 局部按类型替换 | `set_inst_override_by_type()` |
| 全局按名字替换 | `set_type_override()` |
| 局部按名字替换 | `set_inst_override()` |
| 查看 factory 表 | `factory.print()` |
| 调试类型创建 | `debug_create_by_type()` |
| 调试名字创建 | `debug_create_by_name()` |
| 创建 object | `object_type::type_id::create(name)` |
| 创建 component | `component_type::type_id::create(name, parent)` |
| 查看 component 实际拓扑 | `uvm_top.print_topology()` |
