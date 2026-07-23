# SystemVerilog 第 8 章学习笔记：面向对象编程的高级技巧

> **核心结论**：第 8 章的主线是把第 5 章的 class、第 6 章的随机约束、第 7 章的 mailbox/线程组织起来，形成一个更容易扩展的验证平台。继承负责扩展事务，虚方法负责动态绑定，蓝图模式负责替换事务类型，copy 保证事务独立，回调负责给 Driver/Monitor 预留钩子。
> **记忆方式**：继承改事务，virtual 管分派，blueprint 换对象，copy 保独立，callback 插行为。

---

## 8.1 继承简介

### 8.1.1 为什么需要继承

在验证平台中，经常先有一个“正常事务”，后来又需要在它的基础上增加错误注入、特殊约束、延迟、统计信息等功能。
如果把所有可能行为都写进一个大类，短期看起来简单，但后续每加一种测试都要改同一个文件，维护压力很大。
继承的作用就是：**不修改原始类，在已有类基础上增加新行为**。
| 写法 | 思路 | 问题 |
|------|------|------|
| 一个超大类 | 把正常事务、错误事务、特殊约束全部塞进去 | 类越来越难读，测试之间互相影响 |
| 合成 | 一个类中包含另一个类 | 适合“有一个”的关系，但层次可能变深 |
| 继承 | 新类扩展旧类 | 适合“是一个”的关系，能复用基类接口 |
BadTr 是一种 Transaction，只是多了错误注入能力，所以适合用继承。

---

### 8.1.2 Transaction 基类

事务基类保存所有事务共有的数据和方法，例如源地址、目的地址、数据、CRC、显示函数和 CRC 计算函数。
```systemverilog
class Transaction;
    rand bit [31:0] src, dst, data[8];   // 随机变量
    bit  [31:0] crc;                     // 计算得到的 CRC

    // virtual：允许派生类覆盖这个方法，实现多态
    virtual function void calc_crc();
        crc = src ^ dst ^ data.xor;       // data.xor 对数组所有元素做异或归约
    endfunction

    // prefix 用来给日志增加前缀，例如 "DEBUG " 或层次名
    virtual function void display(input string prefix = "");
        $display("%sTr: src=%h, dst=%h, crc=%h",
                 prefix, src, dst, crc);
    endfunction
endclass
```
| 成员 | 作用 |
|------|------|
| `src`/`dst` | 事务的源地址和目的地址 |
| `data[8]` | 事务携带的数据 |
| `crc` | 校验字段 |
| `calc_crc()` | 根据事务字段计算 CRC |
| `display()` | 打印调试信息 |
> **注意**：`calc_crc()` 和 `display()` 被声明为 `virtual`，是为了后续派生类可以覆盖它们。如果不加 `virtual`，基类句柄调用时可能无法动态调用派生类版本。

---

### 8.1.3 扩展 Transaction：BadTr

BadTr 从 Transaction 派生，用来表示“带错误 CRC 的事务”。
```systemverilog
class BadTr extends Transaction;
    rand bit bad_crc;                     // 为 1 时故意制造 CRC 错误

    virtual function void calc_crc();
        super.calc_crc();                // 先计算正确 CRC
        if (bad_crc)
            crc = ~crc;                  // 再制造错误 CRC
    endfunction

    virtual function void display(input string prefix = "");
        $write("%sBadTr: bad_crc=%b, ", prefix, bad_crc); // 先打印子类字段
        super.display();                 // 再复用父类的显示函数
    endfunction
endclass : BadTr
```
| 语法 | 含义 |
|------|------|
| `extends Transaction` | BadTr 继承 Transaction |
| `super.calc_crc()` | 调用父类版本的 `calc_crc()` |
| `bad_crc` | 派生类新增的错误控制字段 |
| 覆盖 `display()` | 在原有显示基础上加入派生类信息 |
`super` 只能调用直接父类的成员，不能写 `super.super.new()` 这种跨层调用。

---

### 8.1.4 OOP 术语对照

| 术语 | 英文 | 含义 |
|------|------|------|
| 类 | class | 对象模板 |
| 属性 | property | 类中的变量 |
| 方法 | method | 类中的 task/function |
| 基类/父类/超类 | base/parent/super class | 被继承的类 |
| 派生类/子类/扩展类 | derived/child/extended class | 继承得到的新类 |
| 原型 | prototype | 方法声明头，包括参数和返回类型 |
| 覆盖 | override | 子类重新定义父类的虚方法 |

---

### 8.1.5 派生类构造函数

如果基类构造函数带参数，派生类必须显式定义构造函数，并且第一行调用 `super.new(...)`。
```systemverilog
class Base1;
    int var;                             // 父类中需要初始化的数据

    function new(input int var);
        this.var = var;                  // this.var 是成员变量，var 是形参
    endfunction
endclass

class Extended extends Base1;
    function new(input int var);
        super.new(var);                  // 必须是第一行
        // 这里再写派生类自己的初始化
    endfunction
endclass
```
| 情况 | 规则 |
|------|------|
| 基类 `new()` 无参数 | 派生类可以不显式调用 |
| 基类 `new()` 有参数 | 派生类 `new()` 第一行必须调用 `super.new(...)` |
| 想跳过中间父类 | 不允许，不能 `super.super.new()` |

---

### 8.1.6 Driver 中使用基类句柄

Driver 不应该关心事务到底是 Transaction 还是 BadTr。它只需要使用 Transaction 这个公共接口。
```systemverilog
class Driver;
    mailbox gen2drv;                     // Generator 到 Driver 的事务通道

    function new(input mailbox gen2drv);
        this.gen2drv = gen2drv;          // 保存外部传进来的 mailbox 句柄
    endfunction

    task main;
        Transaction tr;                  // 基类句柄，可接收 Transaction/BadTr

        forever begin
            gen2drv.get(tr);             // 阻塞等待下一个事务对象
            tr.calc_crc();               // virtual 决定实际调用哪个版本
            // drive DUT with tr.src, tr.dst, tr.data ...
        end
    endtask
endclass
```
基类句柄 `Transaction tr` 可以指向 Transaction 对象，也可以指向 BadTr 对象。
| 实际对象类型 | `tr.calc_crc()` 调用结果 |
|-------------|--------------------------|
| Transaction | `Transaction::calc_crc()` |
| BadTr | `BadTr::calc_crc()` |
> **关键点**：Driver 只依赖基类接口。这样新事务类型可以进入同一个 Driver，而不需要重写 Driver。

---

## 8.2 蓝图模式 Blueprint

### 8.2.1 简单 Generator 的限制

最直接的发生器会在循环里创建一个 Transaction，随机化后放入 mailbox。
```systemverilog
class Generator;
    mailbox gen2drv;                     // 输出通道：把事务送给 Driver
    Transaction tr;                      // 当前正在生成的事务句柄

    function new(input mailbox gen2drv);
        this.gen2drv = gen2drv;          // 连接到 Environment 创建的 mailbox
    endfunction

    task run;
        forever begin
            tr = new();                  // 写死为 Transaction
            assert(tr.randomize());       // 使用 Transaction 默认约束随机化
            gen2drv.put(tr);             // 将对象句柄传给 Driver
        end
    endtask
endclass
```
这个写法有两个问题。
| 问题 | 说明 |
|------|------|
| 类型写死 | 每次都创建 Transaction，无法自然创建 BadTr |
| 约束固定 | 只能使用 Transaction 原本的约束 |
解决思路：把“创建什么对象”从 Generator 中拿出来，交给一个可替换的模板对象。

---

### 8.2.2 蓝图模式的核心思路

蓝图对象相当于“事务模板”。Generator 不直接 `new Transaction`，而是随机化蓝图，再复制蓝图。
```systemverilog
class Generator;
    mailbox gen2drv;                     // 发生器到驱动器的通道
    Transaction blueprint;               // 蓝图对象：决定每轮生成哪种事务

    function new(input mailbox gen2drv);
        this.gen2drv = gen2drv;
        blueprint = new();               // 默认蓝图
    endfunction

    task run;
        Transaction tr;                  // 本轮真正要发送的事务副本

        forever begin
            assert(blueprint.randomize()); // 先随机化模板对象
            tr = blueprint.copy();        // 复制蓝图，而不是直接发送蓝图
            gen2drv.put(tr);             // 每次 put 的都是独立对象
        end
    endtask
endclass
```
| 步骤 | 含义 |
|------|------|
| 创建 `blueprint` | 生成默认事务模板 |
| 测试中替换 `blueprint` | 改变 Generator 产生的事务类型 |
| 随机化 `blueprint` | 得到本轮事务的随机值 |
| `copy()` | 生成独立事务对象 |
| `put()` | 发送给下游 Driver |
> **注意**：不能把 `blueprint` 本身直接放进 mailbox。否则下一轮随机化会修改同一个对象，导致前面送出的事务也被改变。

---

### 8.2.3 Environment 三阶段

教材使用 Build、Run、Wrap-up 三阶段组织测试平台。
| 阶段 | 作用 |
|------|------|
| `build()` | 创建组件、连接 mailbox/interface |
| `run()` | 启动 Generator、Driver、Monitor 等线程 |
| `wrap_up()` | 检查结果、打印报告、清理资源 |
这种结构把固定平台搭建逻辑放在 Environment 中，把测试差异放在顶层 test 中。

---

### 8.2.4 使用扩展事务替换蓝图

如果希望 Generator 产生 BadTr，不需要改 Generator，只需要在 test 中替换蓝图。
```systemverilog
program automatic test;
    Environment env;                     // 顶层测试环境句柄

    initial begin
        env = new();                     // 创建环境对象
        env.build();                     // 创建 gen/drv/mailbox

        begin
            BadTr bad = new();           // 创建派生事务作为新蓝图
            env.gen.blueprint = bad;      // 用 BadTr 替换默认蓝图
        end

        env.run();                       // 此后 Generator 会产生 BadTr
        env.wrap_up();                   // 收尾检查
    end
endprogram
```
| 组件 | 是否需要知道 BadTr |
|------|-------------------|
| test | 需要 |
| Environment | 不需要 |
| Generator | 不需要 |
| Driver | 不需要 |
这就是蓝图模式的价值：变化集中在 test 中，底层平台保持通用。

---

### 8.2.5 用扩展类改变随机约束

继承不仅可以增加字段和方法，也可以增加或覆盖约束。
```systemverilog
class Nearby extends Transaction;
    constraint c_nearby {
        dst inside {[src-100 : src+100]}; // 限制 dst 靠近 src
    }
endclass

program automatic test;
    Environment env;

    initial begin
        env = new();
        env.build();

        begin
            Nearby nb = new();           // 新蓝图只改变随机约束
            env.gen.blueprint = nb;      // Generator 代码不用改
        end

        env.run();
        env.wrap_up();
    end
endprogram
```
如果派生类中定义了与基类同名的约束，派生类约束会替代基类约束。
> **经验**：新增约束适合“缩小随机空间”；同名覆盖约束适合“改变原有规则”，但要谨慎命名，避免别人读代码时误判行为。

---

## 8.3 类型向下转换和虚方法

### 8.3.1 向上赋值

派生类句柄可以直接赋给基类句柄，这叫向上转换。
```systemverilog
Transaction tr;                          // 基类句柄
BadTr bad;                               // 派生类句柄
bad = new();                             // 实际创建 BadTr 对象
tr = bad;                                // 合法：基类句柄指向派生类对象
$display(tr.src);                        // 只能访问基类中声明的成员
tr.display();                            // virtual 时调用 BadTr::display()
```
为什么合法？因为 BadTr 对象中包含 Transaction 的所有字段和方法。

---

### 8.3.2 为什么需要向下转换

基类句柄只能访问基类接口，即使它实际指向 BadTr 对象，也不能直接访问 `bad_crc`。
```systemverilog
Transaction tr;
BadTr bad;
tr = new();                              // 实际对象是 Transaction
bad = tr;                                // 错误：不能直接把基类句柄赋给派生类句柄
```
原因是：Transaction 对象不一定真的有 BadTr 的字段。
如果确实知道 `tr` 指向的是 BadTr 对象，需要使用 `$cast`。

---

### 8.3.3 `$cast` 的用法

```systemverilog
Transaction tr;                          // 基类句柄
BadTr bad, bad2;                         // 派生类句柄
bad = new();                             // 创建 BadTr 对象
tr = bad;                                // 向上赋值

// 任务形式：失败时仿真器报错
$cast(bad2, tr);                         // tr 实际指向 BadTr，所以成功

// 函数形式：失败时返回 0，可自己处理
if (!$cast(bad2, tr))
    $display("cannot assign tr to bad2");
```
| `$cast` 形式 | 失败时行为 | 使用场景 |
|-------------|------------|---------|
| `$cast(dst, src);` | 运行时报错 | 类型必须正确 |
| `if (!$cast(dst, src))` | 返回 0，不自动报错 | 允许失败并自行处理 |
> **注意**：`$cast` 复制的是句柄，不是对象内容。转换成功后，两个句柄仍然指向同一个对象。

---

### 8.3.4 虚方法的动态绑定

虚方法根据**对象实际类型**调用，而不是根据句柄声明类型调用。
```systemverilog
class Transaction;
    rand bit [31:0] src, dst, data[8];   // 事务随机字段
    bit  [31:0] crc;                     // 派生计算字段

    virtual function void calc_crc();
        crc = src ^ dst ^ data.xor;      // 基类默认 CRC 算法
    endfunction
endclass

class BadTr extends Transaction;
    rand bit bad_crc;                    // 子类新增字段

    virtual function void calc_crc();
        super.calc_crc();                // 复用父类算法，避免重复代码
        if (bad_crc)
            crc = ~crc;                  // 在父类结果上注入错误
    endfunction
endclass
```
```systemverilog
Transaction tr;
BadTr bad;

initial begin
    tr = new();                          // 创建普通事务
    tr.calc_crc();                        // Transaction::calc_crc()

    bad = new();                         // 创建错误事务
    bad.calc_crc();                       // BadTr::calc_crc()

    tr = bad;                            // 基类句柄指向派生类对象
    tr.calc_crc();                        // BadTr::calc_crc()
end
```
| 是否 virtual | `tr = bad; tr.calc_crc();` 调用 |
|-------------|--------------------------------|
| 是 | `BadTr::calc_crc()` |
| 否 | `Transaction::calc_crc()` |

---

### 8.3.5 虚方法签名

派生类覆盖虚方法时，必须保持相同签名。
| 签名组成 | 说明 |
|----------|------|
| 方法名 | 必须一致 |
| 参数个数 | 必须一致 |
| 参数类型 | 必须一致 |
| 返回类型 | 必须一致 |
| task/function 类型 | 不能互换 |
```systemverilog
class Base;
    // 父类先确定统一接口
    virtual function void display(string prefix = "");
    endfunction
endclass

class Child extends Base;
    virtual function void display(string prefix = ""); // 正确
    endfunction
    // virtual function void display();                 // 错误：签名不同
endclass
```
> **经验**：基类虚方法的参数要提前设计。后面再改签名，会牵连整个继承体系。

---

## 8.4 合成、继承和其他替代方法

### 8.4.1 “有一个”和“是一个”

选择合成还是继承，最简单的判断是看关系。
| 关系 | 适合方法 | 例子 |
|------|----------|------|
| “有一个” | 合成 | 数据包有 header 和 payload |
| “是一个” | 继承 | BadTr 是一种 Transaction |
合成把多个小对象组合成一个大对象；继承把一个已有对象特化成另一个对象。

---

### 8.4.2 合成和继承对比

| 问题 | 更适合继承 | 更适合合成 |
|------|------------|------------|
| 是否把多个独立小类组合成大类 | 否 | 是 |
| 新类是否与旧类处于相近抽象级别 | 是 | 否 |
| 低级字段是否总是存在 | 是 | 否 |
| 旧代码能否直接处理新增信息 | 是 | 否 |
例如 Driver 只关心 Transaction 接口，那么 BadTr 很适合继承 Transaction。

---

### 8.4.3 合成的问题

以以太网帧为例，如果一个 MAC 帧对象中再包含一个 VLAN 对象，字段访问会变深。
```systemverilog
// 不推荐：示意合成带来的层次
class EthMacFrame;
    typedef enum {II, IEEE} kind_e;      // 帧格式类型
    rand kind_e kind;                    // 决定本帧是哪种格式
    rand bit [47:0] da, sa;              // 目的/源 MAC 地址
    rand bit [15:0] len;                 // 长度字段
    rand Vlan vlan_h;                    // 子对象：额外一层层次
endclass

class Vlan;
    rand bit [15:0] vlan;                // VLAN 信息
endclass
```
访问 VLAN 字段可能变成：
```systemverilog
eth_h.vlan_h.vlan
```
合成还可能带来随机化问题：是否创建 `vlan_h` 取决于随机字段 `kind`，但随机化前对象又必须已经创建。

---

### 8.4.4 继承的问题

继承可以减少字段访问层次，但也有代价。
| 代价 | 说明 |
|------|------|
| 需要虚方法 | 可扩展行为要提前声明为 virtual |
| 需要 `$cast` | 基类句柄转派生类句柄时要做运行时检查 |
| 签名固定 | 虚方法参数后续不好改 |
| 初始化复杂 | 基类无法约束派生类才有的字段 |
| 不支持多继承 | SystemVerilog 一个类只能继承一个父类 |
所以继承适合“稳定基类 + 局部变化”的场景，不适合所有建模问题。

---

### 8.4.5 现实中的折中：扁平类

对于协议帧这种字段很多、变体很多的对象，有时一个不分层的大类反而更实用。
| 方案 | 优点 | 缺点 |
|------|------|------|
| 合成 | 层次清楚 | 随机化和字段访问可能麻烦 |
| 继承 | 替换方便 | 需要虚方法和类型转换 |
| 扁平类 | 约束集中、调试直接 | 类可能比较大 |
扁平类通常使用 `kind` 这类判别字段，再配合条件约束控制不同格式下哪些字段有效。
> **经验**：验证建模的目标不是追求最“纯”的 OOP，而是可控、可调试、可扩展。

---

## 8.5 对象的复制

### 8.5.1 为什么蓝图必须 copy

蓝图模式中，Generator 每轮都会随机化 `blueprint`。如果直接发送 `blueprint` 句柄，所有事务都会指向同一个对象。
正确做法是：随机化蓝图，然后复制出一个独立事务。
```systemverilog
assert(blueprint.randomize());           // 随机化模板
tr = blueprint.copy();                   // 复制成独立事务
gen2drv.put(tr);                         // 发送副本，而不是发送模板
```
| 写法 | 后果 |
|------|------|
| `gen2drv.put(blueprint)` | 多个 mailbox 条目指向同一对象 |
| `gen2drv.put(blueprint.copy())` | 每个事务都是独立对象 |

---

### 8.5.2 基类中的 copy

```systemverilog
class Transaction;
    rand bit [31:0] src, dst, data[8];   // 需要复制的随机字段
    bit  [31:0] crc;                     // 需要复制的计算字段

    virtual function Transaction copy();
        copy = new();                    // 创建新的 Transaction 对象
        copy.src  = src;                 // 逐字段复制
        copy.dst  = dst;
        copy.data = data;
        copy.crc  = crc;
    endfunction
endclass
```
copy 的返回类型是 Transaction，因为虚方法覆盖时必须保持签名一致。

---

### 8.5.3 派生类中的 copy

```systemverilog
class BadTr extends Transaction;
    rand bit bad_crc;                    // 子类自己的字段也要复制

    virtual function Transaction copy();
        BadTr bad;                       // 用子类句柄创建子类对象
        bad = new();
        bad.src     = src;               // 复制父类字段
        bad.dst     = dst;
        bad.data    = data;
        bad.crc     = crc;
        bad.bad_crc = bad_crc;           // 复制子类字段
        return bad;                      // 返回类型仍可写成 Transaction
    endfunction
endclass : BadTr
```
虽然返回类型是 Transaction，但实际返回对象可以是 BadTr。
> **注意**：派生类 copy 不能只复制基类字段，必须复制派生类新增字段，否则错误注入状态会丢失。

---

### 8.5.4 copy_data 方法

直接在每个派生类 copy 中复制所有字段，会导致重复代码。更好的方式是拆成 `copy()` 和 `copy_data()`。
```systemverilog
class Transaction;
    rand bit [31:0] src, dst, data[8];
    bit  [31:0] crc;

    // 只负责复制“当前类”拥有的数据
    virtual function void copy_data(input Transaction to);
        to.src  = src;
        to.dst  = dst;
        to.data = data;
        to.crc  = crc;
    endfunction

    // 负责创建对象，并调用 copy_data 填充数据
    virtual function Transaction copy();
        copy = new();
        copy_data(copy);
    endfunction
endclass
```
派生类只负责复制自己新增的数据。
| 方法 | 职责 |
|------|------|
| `copy()` | 创建目标对象，返回句柄 |
| `copy_data()` | 把字段复制到目标对象 |
BadTr 的 `copy_data()` 先调用 `super.copy_data(to)`，再 `$cast` 到 BadTr 并复制 `bad_crc`。

---

### 8.5.5 指定复制目标

copy 还可以支持“复制到已有对象”。
```systemverilog
class Transaction;
    virtual function Transaction copy(Transaction to = null);
        if (to == null)
            copy = new();                // 未指定目标：创建新对象
        else
            copy = to;                   // 指定目标：复用已有对象

        copy_data(copy);                 // 统一复制字段
    endfunction
endclass
```
BadTr 版本也必须保持相同签名；如果传入已有对象，需要 `$cast` 成 BadTr 后再调用 `copy_data()`。
| 参数 | 行为 |
|------|------|
| `to == null` | 创建一个新对象 |
| `to != null` | 复用已有对象 |
| 派生类复用对象 | 先 `$cast` 检查类型 |

---

## 8.6 抽象类和纯虚方法

### 8.6.1 抽象类的作用

抽象类可以被继承，但不能直接实例化。
```systemverilog
virtual class BaseTr;
    static int count;                    // 所有事务共享的计数器
    int id;                              // 每个事务自己的唯一编号

    function new();
        id = count++;                    // 创建对象时自动分配 id
    endfunction

    // 纯虚方法：只规定接口，具体行为留给派生类实现
    pure virtual function bit compare(input BaseTr to);
    pure virtual function BaseTr copy(input BaseTr to = null);
    pure virtual function void display(input string prefix = "");
endclass : BaseTr
```
| 语法 | 含义 |
|------|------|
| `virtual class` | 抽象类，不能直接 `new()` |
| `pure virtual function` | 纯虚方法，只有原型，没有方法体 |
| `static int count` | 所有对象共享的计数器 |
| `id = count++` | 给每个事务分配唯一编号 |
抽象类适合放在验证包中，作为所有事务的统一父类。

---

### 8.6.2 纯虚方法

纯虚方法强制派生类实现统一接口。
| 方法 | 目的 |
|------|------|
| `compare()` | 记分板比较事务 |
| `copy()` | 蓝图模式复制事务 |
| `display()` | 调试输出事务 |
如果一个派生类没有实现所有纯虚方法，它仍然不能被实例化。
> **注意**：纯虚方法只能出现在抽象类中。抽象类也可以包含普通方法，例如 BaseTr 的 `new()`。

---

### 8.6.3 Transaction 实现 BaseTr

Transaction 继承 BaseTr 后，必须实现 `compare()`、`copy()` 和 `display()`。
```systemverilog
class Transaction extends BaseTr;
    rand bit [31:0] src, dst, crc, data[8]; // Transaction 的具体字段

    extern virtual function bit compare(input BaseTr to);          // 类外实现
    extern virtual function BaseTr copy(input BaseTr to = null);   // 类外实现
    extern virtual function void display(input string prefix = ""); // 类外实现
    extern function new();                                         // 构造函数
endclass
```
| 方法 | 实现要点 |
|------|----------|
| `compare()` | 先 `$cast` 成 Transaction，再逐字段比较 |
| `copy()` | 创建或复用对象，再复制字段 |
| `display()` | 打印 id 和关键字段 |
| `new()` | 调用 `super.new()`，保留 BaseTr 的 id 分配逻辑 |

#### pure virtual 和 extern virtual 的区别

`pure virtual` 和 `extern virtual` 都可以在类里只写方法头，但含义不同。

`pure virtual` 表示父类只规定接口，不提供实现，子类必须实现。
```systemverilog
virtual class A;
    pure virtual function void f();
endclass
```
`A` 不能直接 `new()`。如果某个子类没有实现 `f()`，这个子类也仍然是抽象的，不能实例化。

`extern virtual` 表示方法有实现，只是函数体写在类外面。
```systemverilog
class B;
    extern virtual function void f();
endclass

function void B::f();
    $display("B::f");
endfunction
```
`B` 可以直接 `new()`，因为 `f()` 有具体实现。

| 写法 | 有没有实现 | 是否强制子类实现 | 类能不能直接 `new()` |
|------|------------|------------------|----------------------|
| `pure virtual function void f();` | 没有 | 是 | 不能 |
| `extern virtual function void f();` | 有，写在类外 | 否 | 可以 |

> **记忆**：`pure virtual` 是“我不实现，你们必须实现”；`extern virtual` 是“我实现了，只是写在外面”。

---

### 8.6.4 抽象类的价值

| 没有抽象类 | 使用抽象类 |
|------------|------------|
| 每个事务类接口可能不同 | 所有事务都有统一接口 |
| Generator 难以参数化 | Generator 可以依赖 BaseTr |
| Scoreboard 难以通用 | Scoreboard 可以调用 `compare()` |
| 调试输出风格不统一 | 统一 `display()` |
抽象类不是为了增加代码量，而是为了让后面的组件能只依赖公共接口。

---

## 8.7 回调 Callback

### 8.7.1 为什么需要回调

Driver 可能需要很多可选行为。
| 可选行为 | 例子 |
|----------|------|
| 注入错误 | 改写 CRC、破坏数据 |
| 丢弃事务 | 随机 drop packet |
| 延迟事务 | 插入随机等待 |
| 同步事务 | 等待某个事件后再发送 |
| 记分板连接 | 把期望事务送给 scoreboard |
| 覆盖率采样 | 收集功能覆盖率 |
如果每种行为都改 Driver，Driver 会越来越复杂。
回调的做法是：Driver 在固定位置调用一个可扩展的对象方法，具体行为由 test 决定。

---

### 8.7.2 回调基类

```systemverilog
virtual class Driver_cbs;
    // 发送前回调：可修改事务，也可把 drop 置 1 来丢弃事务
    virtual task pre_tx(ref Transaction tr, ref bit drop);
        // 默认不做任何动作
    endtask

    // 发送后回调：适合做覆盖率采样、日志记录等
    virtual task post_tx(ref Transaction tr);
        // 默认不做任何动作
    endtask
endclass
```
| 方法 | 调用时机 | 常见用途 |
|------|----------|----------|
| `pre_tx()` | 发送事务前 | drop、delay、错误注入、记分板期望值 |
| `post_tx()` | 发送事务后 | 覆盖率采样、日志记录 |
这里的 `pre_tx()` 和 `post_tx()` 不是纯虚方法，因为很多回调只需要实现其中一个。

---

### 8.7.3 Driver 调用回调

```systemverilog
class Driver;
    Driver_cbs cbs[$];                   // 回调队列，可注册多个回调

    task run();
        bit drop;                        // 是否丢弃当前事务
        Transaction tr;                  // 当前要处理的事务

        forever begin
            drop = 0;                    // 每个事务开始时先清零
            agt2drv.get(tr);             // 从上游取一个事务

            foreach (cbs[i])
                cbs[i].pre_tx(tr, drop); // 发送前依次调用回调

            if (drop)
                continue;                // 回调要求丢弃时，不发送

            transmit(tr);                // 真正驱动 DUT

            foreach (cbs[i])
                cbs[i].post_tx(tr);      // 发送后依次调用回调
        end
    endtask
endclass
```
| 代码 | 含义 |
|------|------|
| `Driver_cbs cbs[$]` | 保存多个回调对象 |
| `foreach (cbs[i])` | 按队列顺序调用回调 |
| `drop` | 回调可控制是否丢弃事务 |
| `transmit(tr)` | 真正发送事务 |
> **注意**：教材原意是 drop 为真时丢弃事务。实际代码中应写 `if (drop) continue;`，避免逻辑写反。

---

### 8.7.4 使用回调注入干扰

```systemverilog
class Driver_cbs_drop extends Driver_cbs;
    virtual task pre_tx(ref Transaction tr, ref bit drop);
        // 每 100 个事务随机丢弃 1 个
        drop = ($urandom_range(0, 99) == 0); // 命中 0 时 drop=1
    endtask
endclass
```
在 test 中注册回调。
注册方式：`env.drv.cbs.push_back(new_driver_callback);`
| 队列操作 | 效果 |
|----------|------|
| `push_back()` | 放到队尾，较晚执行 |
| `push_front()` | 放到队首，较早执行 |
回调顺序会影响语义，例如先 drop 再采样覆盖率，和先采样再 drop，统计结果不同。

---

### 8.7.5 记分板简介

记分板用于保存期望结果，并与实际结果比较。
```systemverilog
class Scoreboard;
    Transaction scb[$];                  // 保存期望事务

    function void save_expect(input Transaction tr);
        scb.push_back(tr);               // 将期望事务放到队尾
    endfunction

    function void compare_actual(input Transaction tr);
        int q[$];                        // 保存匹配到的队列下标

        q = scb.find_index(x) with (x.src == tr.src); // 简化示例：按 src 匹配

        case (q.size())
            0:       $display("No match found");
            1:       scb.delete(q[0]);   // 匹配成功，删除对应期望项
            default: $display("Error, multiple matches found!");
        endcase
    endfunction : compare_actual
endclass : Scoreboard
```
| 匹配数量 | 含义 |
|----------|------|
| 0 | 没有找到期望事务 |
| 1 | 理想情况，删除该期望项 |
| 多个 | 匹配条件太弱，需要更复杂比较 |

---

### 8.7.6 用回调连接记分板

```systemverilog
class Driver_cbs_scoreboard extends Driver_cbs;
    Scoreboard scb;                      // 保存记分板句柄

    function new(input Scoreboard scb);
        this.scb = scb;                  // 构造时连接到环境中的记分板
    endfunction

    virtual task pre_tx(ref Transaction tr, ref bit drop);
        scb.save_expect(tr);             // 发送前记录期望事务
    endtask
endclass
```
test 中注册记分板回调。
```systemverilog
begin
    Driver_cbs_scoreboard dcs = new(env.scb); // 创建带 scb 句柄的回调
    env.drv.cbs.push_back(dcs);               // 注册到 Driver 回调队列
end
```
Monitor 也可以有自己的回调，用来把 DUT 实际输出送入 `compare_actual()`。
> **经验**：记分板和覆盖率通常是被动组件，适合通过回调接入测试平台，而不是让它们主动监听多个 mailbox。

---

### 8.7.7 用回调调试

如果事务处理器行为不符合预期，可以临时添加一个 debug 回调。
```systemverilog
class Driver_cbs_debug extends Driver_cbs;
    virtual task pre_tx(ref Transaction tr, ref bit drop);
        $display("%m before transmit");  // %m 打印当前层次路径
        tr.display("DEBUG ");            // 打印事务内容
    endtask
endclass
```
| 调试方法 | 用途 |
|----------|------|
| 打印 `%m` | 查看当前层次路径 |
| 在队首插入 debug 回调 | 查看其他回调修改前的事务 |
| 在队尾插入 debug 回调 | 查看其他回调修改后的事务 |
调试回调应该尽量不改变事务内容，避免调试代码本身影响测试结果。

---

## 8.8 本章总结

### 学习重点排序

| 优先级 | 必须掌握 |
|------|----------|
| 🔴 高 | `extends`、`super`、基类句柄指向派生类对象 |
| 🔴 高 | `virtual` 方法、override、多态动态绑定 |
| 🔴 高 | 蓝图模式：`blueprint.randomize()` + `blueprint.copy()` |
| 🔴 高 | `$cast()` 向下转换、任务形式和函数形式区别 |
| 🟡 中 | `copy()`、`copy_data()`、复制目标参数 `to = null` |
| 🟡 中 | 抽象类 `virtual class`、纯虚方法 `pure virtual` |
| 🟡 中 | 回调 `pre_tx()`/`post_tx()`、回调队列 `cbs[$]` |
| 🟢 进阶 | Scoreboard 通过回调接入 Driver/Monitor |
| 🟢 进阶 | 合成、继承、扁平类三种建模方式的取舍 |

---

### 最重要的 9 条规则

| # | 规则 | 说明 |
|---|------|------|
| 1 | 子类用 `extends` 扩展父类 | 适合“BadTr 是一种 Transaction”这种“是一个”的关系 |
| 2 | 可能被子类改写的方法要声明为 `virtual` | 否则基类句柄可能只调用基类版本 |
| 3 | 子类复用父类行为时用 `super.method()` | 例如 BadTr 先算正确 CRC，再翻转 CRC |
| 4 | 基类句柄可以指向派生类对象 | `Transaction tr; tr = bad;` 合法 |
| 5 | 访问派生类专有字段必须先 `$cast` | `bad_crc` 不在 Transaction 接口中 |
| 6 | 虚方法覆盖必须保持签名一致 | 参数个数、参数类型、返回类型都不能乱改 |
| 7 | 蓝图模式必须发送 `copy()` 后的副本 | 不能把 `blueprint` 本体直接放进 mailbox |
| 8 | 派生类的 `copy()` 必须复制派生字段 | 只复制基类字段会丢失 `bad_crc` 等信息 |
| 9 | 回调用于插入测试特有行为 | drop、delay、scoreboard、coverage 都适合用 callback |

---

### 最容易错的点

| 易错点 | 正确理解 |
|--------|----------|
| 以为继承可以替代所有合成 | 错，继承适合“是一个”，合成适合“有一个” |
| 忘记给基类方法加 `virtual` | 基类句柄调用时不会动态分派到子类版本 |
| 以为 `super` 能跨多层调用 | 错，只能调用直接父类，不能 `super.super` |
| 直接 `bad = tr` | 错，基类句柄转派生类句柄要用 `$cast` |
| 以为 `$cast` 会复制对象 | 错，只转换/复制句柄，不复制对象内容 |
| 直接发送 `blueprint` | 错，下一次随机化会改掉同一个对象 |
| 子类 `copy()` 忘记复制新增字段 | 错，BadTr 的 `bad_crc` 也必须复制 |
| 虚方法覆盖时改参数列表 | 错，签名不一致就不是同一个多态接口 |
| 把回调方法写成纯虚方法 | 不一定合适，因为很多回调只实现 `pre_tx()` 或 `post_tx()` |
| 回调队列顺序随便放 | 错，drop、scoreboard、coverage 的先后会改变测试语义 |

第 8 章的本质是：**把测试平台中会变化的行为变成可替换对象或可插拔回调，让稳定平台少改，让测试代码只表达本测试的特殊意图。**
