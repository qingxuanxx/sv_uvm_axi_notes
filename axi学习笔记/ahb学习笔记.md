# AMBA AHB 学习笔记：从高性能总线到协议验证

> 适合读者：已经掌握 APB 协议，准备学习 AHB 高性能总线的验证工程师。
>
> 学习目标：看懂 AHB 时序；理解流水线、burst、等待状态和错误响应；能写 AHB interface、monitor、scoreboard 和协议断言；能区分 AHB 与 APB 的验证差异。

---

## 0. 文档定位与版本说明

本文主要依据 Arm《AMBA AHB Protocol Specification》ARM IHI 0033C（AMBA 5 AHB）。

版本演进：

| 版本 | 说明 |
|---|---|
| Issue A | 定义 AHB-Lite 接口 |
| Issue B | 引入 AHB5（在 AHB-Lite 基础上增加能力） |
| Issue C | 增加信号宽度属性、写 strobe、用户信号、奇偶校验接口保护 |

官方新版术语：

| 本文术语 | 旧资料术语 | 含义 |
|---|---|---|
| Manager | Master | 发起传输的一方（CPU、DMA） |
| Subordinate | Slave | 响应传输的一方（内存控制器、外设） |

初学优先级：

1. 先掌握 `HADDR`、`HTRANS`、`HWRITE`、`HREADY` 组成的传输流程。
2. 再掌握 burst、等待状态和 ERROR 响应。
3. 最后了解写 strobe、锁定传输、端序等扩展特性。

---

## 1. AHB 是什么

AHB 全称 Advanced High-performance Bus，定位是**高性能、高带宽**的片上主干总线。

常见系统结构：

```text
CPU / DMA（Manager）
    |
  AHB Interconnect（含 Decoder + Multiplexor + 仲裁）
    |
    +---------+---------+-----------+
    |         |         |           |
  Memory    SRAM     AHB-to-APB    其他
  Controller              桥
                           |
                         APB 外设（UART/GPIO/Timer）
```

AHB 的核心特点：

- **流水线（pipeline）传输**：地址阶段和数据阶段重叠，下一笔传输的地址可以在上一笔传输的数据阶段发出，实现高吞吐。
- **突发传输（burst）**：支持 4/8/16 拍和不定长 burst。
- **多 Manager**：支持多个主机，需要仲裁器（interconnect 负责）。
- **等待状态**：Subordinate 用 `HREADYOUT` 插入等待。
- **分离读写数据总线**：`HWDATA` 和 `HRDATA` 独立。
- 所有信号以 `H` 开头（HCLK、HADDR...），与其他总线区分。

> 记忆：**AHB 是"主干道"追求吞吐，APB 是"小区路"追求简单低功耗**。AHB 支持流水线/burst/多主机，APB 只有两拍握手。

### 1.1 AHB 与 APB 的区别

| 对比项 | AHB | APB |
|---|---|---|
| 定位 | 高性能主干总线 | 低速外设总线 |
| 传输方式 | 流水线（地址/数据重叠） | 两拍（SETUP + ACCESS） |
| burst | ✅ 支持 | ❌ 不支持 |
| 多主机 | ✅ 支持（需仲裁） | ❌ 单主机 |
| 等待状态 | ✅ HREADY | ✅ PREADY |
| 错误响应 | ✅ HRESP（ERROR） | ✅ PSLVERR |
| 读写数据总线 | 分离（HWDATA/HRDATA） | 共享 |

---

## 2. 信号总览

### 2.1 全局信号

| 信号 | 方向 | 作用 |
|---|---|---|
| `HCLK` | Clock -> all | 总线时钟，所有信号时序相对 HCLK 上升沿 |
| `HRESETn` | Reset -> all | 低有效复位（唯一低有效信号） |

### 2.2 Manager 信号（Manager 产生）

| 信号 | 方向 | 作用 |
|---|---|---|
| `HADDR` | -> Sub/Decoder | 字节地址（建议 10~64 位） |
| `HTRANS[1:0]` | -> Sub | 传输类型：IDLE/BUSY/NONSEQ/SEQ |
| `HWRITE` | -> Sub | 1=写，0=读；整个 burst 内必须保持不变 |
| `HSIZE[2:0]` | -> Sub | 传输大小：8/16/32/64...位 |
| `HBURST[2:0]` | -> Sub | burst 类型：SINGLE/INCR/WRAP4/INCR4... |
| `HWDATA` | -> Sub | 写数据总线 |
| `HWSTRB` | -> Sub | 写 strobe：每个字节一位（可选特性） |
| `HPROT` | -> Sub | 保护控制：访问类型信息 |
| `HMASTLOCK` | -> Sub | 锁定传输指示（如 SWP 指令） |
| `HNONSEC` | -> Sub | 安全/非安全传输（AHB5） |
| `HEXCL`/`HMASTER` | -> EAM/Sub | 独占访问相关（AHB5） |

### 2.3 Subordinate 信号（Subordinate 产生）

| 信号 | 方向 | 作用 |
|---|---|---|
| `HRDATA` | -> Multiplexor | 读数据总线 |
| `HREADYOUT` | -> Multiplexor | 1=传输完成；0=延长传输（插等待） |
| `HRESP` | -> Multiplexor | 0=OKAY，1=ERROR |
| `HEXOKAY` | -> Multiplexor | 独占访问结果（AHB5） |

### 2.4 Decoder 与 Multiplexor 信号

| 信号 | 产生者 | 作用 |
|---|---|---|
| `HSELx` | Decoder | 每个 Subordinate 一根选择线，地址译码结果 |
| `HREADY` | Multiplexor | 综合后的全局就绪：HIGH=上一传输完成 |
| `HRDATA`/`HRESP`/`HEXOKAY` | Multiplexor | 从被选中的 Subordinate 选通给 Manager |

> 关键：**每个 Subordinate 有自己的 HREADYOUT，互联把它们综合成全局 HREADY**。所有 Subordinate 都准备好，HREADY 才为 HIGH。

---

## 3. 传输流程

### 3.1 一次传输的两个阶段

```text
地址阶段（Address phase）  : 一个 HCLK 周期，Manager 驱动地址和控制信号
数据阶段（Data phase）    : 一个或多个周期，用 HREADY 控制长度
```

最简单的零等待传输：1 个地址周期 + 1 个数据周期。

```text
       HCLK      _/‾‾\_/‾‾\_/‾‾\_
       HADDR     [ A ]  [ B ]        <- 地址 A 后紧跟地址 B（流水线！）
       HWRITE    [写A ] [读B ]
       HRDATA           [Data(A)]
       HREADY    [ 1  ] [ 1  ]
                 |--A 地址--|--A 数据--
                 |        |--B 地址--|--B 数据--
```

**关键：地址阶段和数据阶段重叠**——传输 A 的数据阶段同时也是传输 B 的地址阶段。这是 AHB 流水线的本质，也是高性能的来源。

### 3.2 等待状态（wait states）

Subordinate 用 `HREADYOUT` 拉低来延长数据阶段：

```text
      HREADY   [1]  [0]  [0]  [1]     <- 两个等待状态
```

- 等待期间写数据必须保持稳定；读数据只需在传输完成时有效。
- **延长数据阶段会顺带延长下一笔传输的地址阶段**（流水线被阻塞）。

> 记忆：**HREADY=0 插等待、HREADY=1 完成；Subordinate 有最大等待数限制，否则会拖垮整个总线性能**。

### 3.3 传输类型（HTRANS）

| HTRANS[1:0] | 类型 | 含义 |
|---|---|---|
| 00 | IDLE | 无数据传输，Subordinate 必须零等待 OKAY 响应并忽略 |
| 01 | BUSY | burst 中间插入空闲周期（Manager 还没准备好下一拍），Subordinate 必须零等待 OKAY 并忽略 |
| 10 | NONSEQ | 单笔传输或 burst 的第一拍，地址与上一笔无关 |
| 11 | SEQ | burst 的后续拍，地址 = 上一地址 + 传输大小（wrap 则绕回） |

> 记忆：**NONSEQ 开头、SEQ 续拍、BUSY 中间歇、IDLE 没活干**。单个传输 = NONSEQ（长度 1 的 burst）。

### 3.4 传输大小（HSIZE）

| HSIZE[2:0] | 大小 |
|---|---|
| 000 | 8 位（Byte） |
| 001 | 16 位（Halfword） |
| 010 | 32 位（Word） |
| 011 | 64 位（Doubleword） |
| 100~111 | 128/256/512/1024 位 |

> 注意：HSIZE 必须 ≤ 数据总线宽度（32 位总线只能用 000/001/010）；HSIZE 在整个 burst 内保持不变。

### 3.5 突发传输（HBURST）

| HBURST[2:0] | 类型 | 含义 |
|---|---|---|
| 000 | SINGLE | 单笔传输 |
| 001 | INCR | 不定长递增 burst |
| 010 | WRAP4 | 4 拍回卷 burst |
| 011 | INCR4 | 4 拍递增 burst |
| 100 | WRAP8 | 8 拍回卷 burst |
| 101 | INCR8 | 8 拍递增 burst |
| 110 | WRAP16 | 16 拍回卷 burst |
| 111 | INCR16 | 16 拍递增 burst |

**INCR（递增）**：地址逐拍递增，不绕界。

**WRAP（回卷）**：地址到达边界时绕回。边界 = 拍数 × 传输大小。

举例：4 拍 word（4 字节）burst，wrap 边界 = 4 × 4 = 16 字节。起始地址 0x34 时：

```text
0x34 -> 0x38 -> 0x3C -> 0x30（回卷！）
```

> 记忆：**wrap 边界 = 拍数 × 每拍字节数**；INCR 不绕界、WRAP 绕界。Manager 不能发起跨越 1KB 边界的递增 burst。

---

## 4. 总线互联

单 Manager 系统只需要 Decoder + Multiplexor：

- **Decoder（地址译码）**：译码 HADDR，产生 HSELx 选中对应 Subordinate，同时给 Multiplexor 控制信号。
- **Multiplexor（多路选择）**：把被选中 Subordinate 的 HRDATA/HRESP 选通给 Manager，并综合全局 HREADY。

多 Manager 系统需要 interconnect（含仲裁），仲裁器决定哪个 Manager 获得总线。

> 验证提示：**地址译码和 HREADY 综合是 AHB 环境最常见的 bug 来源**——译码漏地址空间、HREADY 综合逻辑错会导致整条总线卡死。

---

## 5. Subordinate 响应（HRESP + HREADYOUT）

完整的传输响应 = HRESP + HREADYOUT 的组合：

| HRESP | HREADYOUT | 含义 |
|---|---|---|
| 0（OKAY） | 0 | 传输挂起（还在等待） |
| 0（OKAY） | 1 | 传输成功完成 |
| 1（ERROR） | 0 | ERROR 响应第一拍 |
| 1（ERROR） | 1 | ERROR 响应第二拍 |

### 5.1 三种完成方式

1. **立即完成**：HREADY HIGH + OKAY。
2. **插入等待后完成**：HREADYOUT 拉低若干拍，最后 OKAY 完成。
3. **ERROR 响应**：报错（如写只读地址）。

### 5.2 ERROR 响应的两拍机制

ERROR 响应**必须占两个周期**（OKAY 可以单周期完成）：

```text
第一拍：HRESP=ERROR, HREADYOUT=0（延长一拍）
第二拍：HRESP=ERROR, HREADYOUT=1（结束）
```

**为什么必须两拍**：因为总线是流水线的，Subordinate 发出 ERROR 时，下一笔传输的地址已经广播到总线上了。两拍给 Manager 时间取消下一笔访问、把 HTRANS 改成 IDLE。

> 记忆：**OKAY 一拍、ERROR 两拍**。ERROR 后 Manager 可以取消 burst 剩余拍（也可以继续）；读 ERROR 时建议 HRDATA 驱 0。

---

## 6. 数据总线与端序

- **HWDATA**：Manager 写数据。传输被延长时数据要保持稳定直到 HREADY HIGH。
- **HRDATA**：Subordinate 读数据，经过 Multiplexor 到 Manager。
- 窄传输（如 16 位传在 32 位总线上）：只驱动对应字节通道，Subordinate 按端序选正确的字节通道。

---

## 7. 验证视角：AHB 环境怎么搭

### 7.1 interface 要点

```systemverilog
interface ahb_if(input logic hclk, input logic hrstn);
    logic [31:0] haddr;
    logic [1:0]  htrans;
    logic        hwrite;
    logic [2:0]  hsize;
    logic [2:0]  hburst;
    logic [31:0] hwdata;
    logic [31:0] hrdata;
    logic        hready;      // 全局综合后
    logic        hresp;
    logic        hsel;
endinterface
```

### 7.2 验证要点（monitor/scoreboard/断言）

| 检查点 | 验证内容 |
|---|---|
| 传输类型合法性 | NONSEQ 后只能跟 SEQ/BUSY/IDLE/NONSEQ；BUSY 不能在固定长度 burst 的最后一拍 |
| 地址递增 | SEQ 地址 = 上一地址 + HSIZE；WRAP 在边界回卷 |
| 流水线正确性 | 数据阶段与下一地址阶段重叠采样 |
| HREADY 综合 | 所有 Subordinate 就绪才 HREADY HIGH |
| ERROR 两拍 | HRESP=ERROR 必须持续两拍且第二拍 HREADYOUT HIGH |
| burst 完整性 | 固定长度 burst 必须以 SEQ 结束，不能 BUSY 结尾 |
| 1KB 边界 | INCR burst 不能跨 1KB 边界 |

### 7.3 常见错误场景（测试要覆盖）

- 等待状态下 Manager 非法改变 HTRANS/HADDR。
- WRAP burst 地址计算错误（绕回点算错）。
- ERROR 响应只给了一拍。
- BUSY 用在固定长度 burst 最后一拍。
- 窄传输时字节通道（byte lane）驱动错误。
- 多 Subordinate 时 HREADY 综合漏了某个子模块。

---

## 本章总结

### 学习重点排序

| 优先级 | 内容 |
|---|---|
| 🔴 高 | 两阶段传输与流水线（地址/数据重叠） |
| 🔴 高 | 传输类型（IDLE/BUSY/NONSEQ/SEQ） |
| 🔴 高 | burst 类型与 WRAP 边界计算 |
| 🔴 高 | HREADY 等待状态与 HREADYOUT 综合 |
| 🟡 中 | ERROR 两拍响应机制 |
| 🟡 中 | HSIZE/HBURST 编码表 |
| 🟢 进阶 | 锁定传输、写 strobe、端序、AHB5 独占访问 |

### 最容易错的点

| 易错点 | 正确理解 |
|---|---|
| AHB 与 APB 一样两拍完成 | 错，AHB 是流水线，地址/数据重叠 |
| OKAY 和 ERROR 都是单拍 | 错，ERROR 必须两拍 |
| BUSY 可以在 burst 末尾 | 固定长度 burst 必须以 SEQ 结束 |
| WRAP 和 INCR 一样 | 错，WRAP 在边界绕回 |
| 每个 Subordinate 的 HREADY 直接给 Manager | 错，要经 Multiplexor 综合成全局 HREADY |
| HSIZE 可以大于总线宽度 | 错，HSIZE ≤ 数据总线宽度 |

---

> 参考资料：Arm《AMBA AHB Protocol Specification》ARM IHI 0033C
