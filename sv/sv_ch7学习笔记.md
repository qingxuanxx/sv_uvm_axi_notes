# SystemVerilog 第 7 章学习笔记：线程以及线程间的通信

> **核心结论**：第 7 章 7.1–7.6 的主线是：如何在 SystemVerilog 测试平台中创建线程、停止线程，并用事件、旗语、信箱完成线程间同步与数据交换。
> **记忆方式**：**fork 管线程，disable 停线程，event 做通知，semaphore 管资源，mailbox 传事务。**

---

## 7.1 线程的使用

### 7.1.1 三种 fork 对比

```systemverilog
initial begin
    $display("@%0t start", $time);
    #10 $display("@%0t before fork", $time);
    fork
        #50 $display("@%0t branch A", $time);
        #10 $display("@%0t branch B", $time);
        begin
            #30 $display("@%0t branch C1", $time);
            #10 $display("@%0t branch C2", $time);
        end
    join
    $display("@%0t after join", $time);    // t=60 时执行
end
```

| 语句 | 父线程行为 | 适用场景 |
|------|----------|---------|
| `fork...join` | 等待**所有**子线程完成 | 多个任务全部完成后再继续 |
| `fork...join_none` | **不等待**，立即继续 | 后台启动 monitor/driver/generator |
| `fork...join_any` | 等待**第一个**子线程完成 | 超时检测、先到先服务 |

```systemverilog
// join_none：最常用的测试平台形式
fork
    #10 $display("@%0t child", $time);
join_none
$display("@%0t parent continues immediately", $time);
// 输出：@0 parent continues immediately  →  @10 child

// join_any：用于超时检测
fork
    #100 $display("timeout");
    wait(done == 1);
join_any
$display("one branch finished");
```

### 7.1.2 在类中创建线程

线程不在 `new()` 中启动，而在 `run()` 任务中用 `fork...join_none` 启动：

```systemverilog
class Generator;
    task run(int n);
        fork
            repeat (n) begin
                Packet p = new();            // ⚠️ 每次循环必须 new()
                assert(p.randomize());
                transmit(p);
            end
        join_none                            // 非阻塞启动
    endtask

    task transmit(Packet p);
        // ... 驱动 DUT 接口 ...
    endtask
endclass

Generator gen = new();
initial begin
    gen.run(10);                             // 启动 10 个事务，不阻塞
    // ... 同时启动其他组件 ...
end
```

| 位置 | 应该做什么 | 不应该做什么 |
|------|----------|------------|
| `new()` | 初始化变量、保存句柄 | 启动线程 |
| `run()` | 启动事务处理线程 | 做静态初始化 |

> 📌 `new()` 只负责"搭对象"，`run()` 才负责"跑线程"。

### 7.1.3 动态线程

每次调用 `check_trans` 动态创建一个独立的监视线程，实现并行等待多个响应：

```systemverilog
task check_trans(Transaction tr);
    fork
        begin
            wait(bus.cb.addr == tr.addr);
            $display("@%0t: Addr match %d", $time, tr.addr);
        end
    join_none
endtask

initial begin
    repeat (10) begin
        Transaction tr = new();
        assert(tr.randomize());
        transmit(tr);
        check_trans(tr);                     // 创建独立监视线程，不等它完成
    end
end
```

### 7.1.4 线程中的自动变量 — 关键陷阱！

```systemverilog
// ❌ 错误：fork 捕获的是 j 的引用，最终 j=3，所有线程打印 "3"
program no_auto;
    initial begin
        for (int j = 0; j < 3; j++)
            fork
                $write(j);                   // 漏洞！引用而非值
            join_none
        #0 $display("\n");                   // 输出：333
    end
endprogram

// ✅ 正确：每轮循环创建局部副本
program automatic bug_free;
    initial begin
        for (int j = 0; j < 3; j++) begin
            int k = j;                       // 保存 j 的副本
            fork
                $write(k);
            join_none
        end
        #0 $display;                         // 输出：012
    end
endprogram
```

> ⚠️ 在 `fork...join_none` + `for` 循环中，子线程拿到的是变量**引用**而非创建那一刻的**值**。必须每轮循环保存副本。

### 7.1.5 wait fork — 等待所有子线程

```systemverilog
task run_threads();
    fork
        check_trans(tr1);                    // 线程 1
        check_trans(tr2);                    // 线程 2
        check_trans(tr3);                    // 线程 3
    join_none
    // ... 做其他工作 ...
    wait fork;                               // 等待上述所有线程结束
endtask
```

### 7.1.6 线程间共享变量 — 声明范围陷阱

```systemverilog
// ❌ 类中忘记声明 i，误用了程序级全局 i，两个线程互相干扰
program bug;
    int i;                                   // 全局变量
    class Buggy { int data[10];
        task transmit;
            fork
                for (i=0; i<10; i++) send(data[i]);  // i 未声明！
            join_none
        endtask
    endclass
endprogram

// ✅ 方案1：循环变量本地化  for (int i=0; ...)
// ✅ 方案2：使用 foreach  foreach(data[i]) ...
```

---

## 7.2 停止线程

### 7.2.1 停止单个线程 — disable 标号

```systemverilog
parameter TIME_OUT = 1000;

task check_trans(Transaction tr);
    fork
        begin
            fork : timeout_block                 // 命名块
                begin
                    wait(bus.cb.addr == tr.addr);
                    $display("Addr match");
                end
                #TIME_OUT $display("Error: timeout");
            join_any                             // 谁先完成就...
            disable timeout_block;               // 杀掉另一个
        end
    join_none
endtask
```

### 7.2.2 停止多个线程 — disable fork

```systemverilog
initial begin
    check_trans(tr0);                            // 线程 0（在 fork 外，不受影响）
    fork                                         // 线程 1
        check_trans(tr1);                        // 线程 2
        check_trans(tr2);                        // 线程 3
        #500 disable fork;                       // 停止线程 1~3
    join
end
```

> ⚠️ 用 `fork...join` 把目标代码包围起来可以限制 `disable fork` 的作用范围。

### 7.2.3 禁止被多次调用的任务 — 危险！

```systemverilog
// ❌ disable task_name 会停止该任务的所有活动实例！
task wait_for_timeout(int id);
    if (id == 0) disable wait_for_timeout;        // 把 id=1、id=2 也杀了
endtask
```

| 需求 | 推荐写法 |
|------|---------|
| 停止超时分枝 | `disable labeled_block` |
| 停止当前 fork 子线程 | `disable fork` |
| 可控退出 | 用标志位/事件/mailbox 通知自然结束 |

---

## 7.3 线程间的通信 IPC

| 机制 | 主要用途 | 传数据？ | 排队？ |
|------|---------|:---:|:---:|
| `event` | 通知某件事发生 | ❌ | ❌ |
| `semaphore` | 控制共享资源访问（互斥） | ❌ | 部分 |
| `mailbox` | 在线程间传递事务 | ✅ | ✅ |

---

## 7.4 事件 Event

类比：`->e` = 按门铃，`@e` = 等门铃响，`e.triggered` = 查看这一拍门铃是否响过。

### 7.4.1 边沿阻塞 `@` — 可能错过事件

```systemverilog
event e1, e2;
initial begin
    -> e1;                         // 触发 e1
    @e2;                           // 阻塞等 e2（被唤醒）
end
initial begin
    -> e2;                         // 触发 e2（唤醒上线程）
    @e1;                           // 等 e1 — 但 e1 已触发过了！永远阻塞！
end
```

### 7.4.2 电平等待 `wait(e.triggered())` — 不会错过

```systemverilog
event e1, e2;
initial begin
    -> e1;
    wait(e2.triggered());          // 即使 e2 已触发也不阻塞
    $display("after trigger");     // ✅ 两个线程都能执行到这里
end
initial begin
    -> e2;
    wait(e1.triggered());          // e1 已被触发，直接通过
    $display("after trigger");
end
```

| 方式 | 特点 | 适用场景 |
|------|------|---------|
| `@e` | 边沿敏感，可能错过 | 等未来事件 |
| `wait(e.triggered())` | 电平敏感，不会错过当前 time slot | 检查事件是否发生过 |

### 7.4.3 循环中使用事件的陷阱

```systemverilog
// ❌ 零延时死循环！
forever begin
    wait(handshake.triggered());  // 同一 time slot 内始终为真 → 无限循环
    process_in_zero_time();
end

// ✅ 用边沿敏感
forever begin
    @handshake;                   // 每次触发只执行一次
    process_in_zero_time();
end
```

### 7.4.4 传递事件

事件可以作为句柄传递给子程序/类，解耦层次路径：

```systemverilog
class Generator;
    event done;
    function new(event done);
        this.done = done;
    endfunction
    task run();
        fork
            begin
                // ... 产生事务 ...
                -> done;           // 通知外部：完成
            end
        join_none
    endtask
endclass

program automatic test;
    event gen_done;
    Generator gen = new(gen_done);  // 传入事件句柄
    initial begin
        gen.run();
        wait(gen_done.triggered()); // 等待发生器完成
    end
endprogram
```

### 7.4.5 等待多个事件

线程完成顺序不确定，不能按顺序等待。正确做法：

```systemverilog
event done[N_GENERATORS];

// 方法1：fork + wait fork
foreach (gen[i])
    fork
        automatic int k = i;
        wait(done[k].triggered());
    join_none
wait fork;

// 方法2：静态计数器（最简洁）
class Generator;
    static int thread_count = 0;
    task run();
        thread_count++;            // 启动 +1
        fork
            begin
                // ... 工作代码 ...
                thread_count--;    // 完成 -1
            end
        join_none
    endtask
endclass

wait(Generator::thread_count == 0);
```

---

## 7.5 旗语 Semaphore

控制对共享资源（如总线）的互斥访问：

```systemverilog
program automatic test(bus_ifc.TB bus);
    semaphore sem;

    initial begin
        sem = new(1);             // 只有一把钥匙
        fork
            sequencer();
            sequencer();          // 两个线程竞争同一资源
        join
    end

    task sendTrans();
        sem.get(1);               // 获取钥匙（阻塞直到可用）
        @bus.cb;
        bus.cb.addr <= t.addr;    // 独占访问总线
        sem.put(1);               // 归还钥匙
    endtask
endprogram
```

| 操作 | 说明 |
|------|------|
| `sem = new(N)` | 创建 N 把钥匙 |
| `sem.get(N)` | 获取 N 把钥匙，不够则阻塞 |
| `sem.put(N)` | 归还 N 把钥匙 |
| `sem.try_get(N)` | 尝试获取，返回 1(成功)/0(失败)，不阻塞 |

> ⚠️ `put()` 可以归还比 `get()` 更多的钥匙（逻辑错误仿真器不会阻止）。多钥匙请求时 `get(1)` 可能插队 `get(2)`。

---

## 7.6 信箱 Mailbox

**Mailbox** = 线程安全的 FIFO，解耦生产者和消费者：

```systemverilog
mailbox mbx = new();      // 无界信箱
mailbox mbx = new(1);     // 定容信箱（容量 1）

mbx.put(item);            // 放入（满则阻塞）
mbx.get(item);            // 取出（空则阻塞）
mbx.peek(item);           // 探视：拷贝但不移除
mbx.try_get(item);        // 非阻塞尝试，返回 1(成功)/0(失败)
```

### 7.6.1 基本用法：Generator ↔ Driver

```systemverilog
class Generator;
    mailbox mbx;
    function new(mailbox mbx); this.mbx = mbx; endfunction

    task run(int count);
        Transaction tr;
        repeat (count) begin
            tr = new();               // ⚠️ 每次必须 new 新对象！
            assert(tr.randomize());
            mbx.put(tr);              // 放入信箱
        end
    endtask
endclass

class Driver;
    mailbox mbx;
    function new(mailbox mbx); this.mbx = mbx; endfunction

    task run(int count);
        Transaction tr;
        repeat (count) begin
            mbx.get(tr);              // 从信箱取出
            drive_to_dut(tr);         // 驱动 DUT
        end
    endtask
endclass

// 顶层：共享一个邮箱连接两者
mailbox mbx = new();
Generator gen = new(mbx);
Driver drv = new(mbx);
fork gen.run(n); drv.run(n); join
```

> ⚠️ **最经典的 bug**：在循环外 `new()` 一个对象，循环内反复 `randomize()` 并 `put()`。信箱存的是句柄，所有条目指向**同一对象**，最终只看到最后一组随机值。

### 7.6.2 三种同步方法

| 方法 | 同步程度 | 复杂度 | 核心机制 |
|------|:---:|:---:|------|
| 定容信箱 | 提前 1 个事务 | 低 | `mbx = new(1);` |
| 定容信箱 + `peek` + `get` | 较严格 | 中 | 探视后处理完再取出 |
| 信箱 + 事件 `@handshake` | 严格同步 | 中 | 生产者 put 后等事件才继续 |

```systemverilog
// 严格同步：信箱 + 事件
class Producer {
    mailbox mbx; event handshake;
    task run();
        for (int i=1; i<4; i++) begin
            mbx.put(i);
            @handshake;               // 等消费者说"处理完了"
        end
    endtask
}
class Consumer {
    mailbox mbx; event handshake;
    task run();
        int i;
        repeat (3) begin
            mbx.get(i);
            $display("Consumer: %0d", i);
            -> handshake;             // 通知生产者
        end
    endtask
}
```

---

## 综合示例：event + semaphore + mailbox

```systemverilog
program automatic tb;
    class Transaction;
        rand bit [7:0] addr;
        rand bit [7:0] data;
    endclass

    mailbox mbx;
    semaphore bus_sem;
    event gen_done;

    task generator(int n);
        Transaction tr;
        repeat (n) begin
            tr = new();
            assert(tr.randomize());
            $display("@%0t GEN put addr=%0h data=%0h", $time, tr.addr, tr.data);
            mbx.put(tr);
        end
        -> gen_done;
    endtask

    task driver(int n);
        Transaction tr;
        repeat (n) begin
            mbx.get(tr);
            bus_sem.get(1);
            $display("@%0t DRV drive addr=%0h data=%0h", $time, tr.addr, tr.data);
            #10;
            bus_sem.put(1);
        end
    endtask

    initial begin
        mbx = new();
        bus_sem = new(1);
        fork
            generator(5);
            driver(5);
        join_none
        wait(gen_done.triggered());
        wait fork;
        $display("@%0t all threads finished", $time);
    end
endprogram
```

| 对应关系 | 知识点 |
|---------|--------|
| `fork...join_none` | generator 和 driver 并发运行 |
| `mailbox mbx` | Generator → Driver 传递事务 |
| `semaphore bus_sem` | Driver 使用总线前获取钥匙 |
| `event gen_done` | Generator 完成通知主线程 |
| `wait fork` | 等待所有派生线程结束 |

---

## 本章总结（7.1–7.6）

### 记忆口诀

```
fork 开线程：
    join      等全部
    join_any  等一个
    join_none 不等待

disable 停线程：
    disable label      停指定块
    disable fork       停当前派生线程
    disable task_name  慎用！误杀所有同名 task

event 做通知：
    ->e                 触发
    @e                  等下一次
    wait(e.triggered)   当前时间步触发过也算

semaphore 管资源：
    get 拿钥匙，put 还钥匙，try_get 尝试拿

mailbox 传事务：
    put 放，get 取，空则 get 等，满则 put 等
```

### 学习重点排序

| 优先级 | 必须掌握内容 |
|:---:|------|
| 🔴 | `fork...join / join_any / join_none` 的区别 |
| 🔴 | `join_none` + `for` 循环变量必须拷贝 |
| 🔴 | `disable label` 与 `disable fork` 的作用范围 |
| 🔴 | `event` 的 `@e` 与 `wait(e.triggered())` 区别 |
| 🟡 | `semaphore` 用于互斥访问共享资源 |
| 🟡 | `mailbox` 用于线程间传递 transaction |

### 最容易错的 5 个点

| # | 易错点 | 正确做法 |
|---|--------|---------|
| 1 | `join_none` 后父线程不等待 | 需要同步时用 `wait fork` |
| 2 | 循环变量在 fork 内用引用 | 每轮创建 `automatic int k = i;` 副本 |
| 3 | `@event` 错过已触发事件 | 用 `wait(e.triggered())` |
| 4 | mailbox 循环外 new 对象 | 循环内 `new()` 新对象再 `put` |
| 5 | semaphore get/put 不成对 | 异常路径也要 `put`，否则永久阻塞 |
