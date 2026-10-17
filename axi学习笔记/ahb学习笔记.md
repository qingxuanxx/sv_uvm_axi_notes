# AMBA AHB 学习笔记：单总线流水线，高性能主干

> 适合读者：已经学过 APB（两阶段）和 AXI（五通道），准备补全 AMBA 家族中间一环的验证工程师。
>
> 学习目标：看懂 AHB 时序；理解流水线、HREADY 全局流控、HTRANS 四状态、burst 与 ERROR 两拍响应。

---

## 0. 文档定位与版本说明

本文主要依据 Arm《AMBA AHB Protocol Specification》ARM IHI 0033C（AMBA 5 AHB）。

版本演进：

| 版本 | 说明 |
|---|---|
| Issue A | 定义 AHB-Lite 接口 |
| Issue B | 引入 AHB5（在 AHB-Lite 基础上增加能力） |
| Issue C | 增加信号宽度属性、写 strobe、用户信号、奇偶校验接口保护 |

> **术语说明：本文统一使用传统命名 Master / Slave**（新版规范称 Master / Slave，含义相同：Master 是发起传输的一方如 CPU、DMA，Slave 是响应传输的一方如内存控制器、外设；本文统一用 Master/Slave）。

> **阅读主线：** 先把同一周期中的"上一笔数据"和"下一笔地址"分开（流水线），再学习等待、burst、响应和互联。

---

## 1. AHB 是什么

**一句话定位**：AHB 是 AXI 的"前辈"——高性能系统总线，**一条总线（地址数据共用）+ 流水线 + 全局流控**，介于 APB（最简单）和 AXI（最强）之间。

AMBA 家族定位：

| 协议 | 定位 | 一句话 |
|---|---|---|
| APB | 低速外设 | 单车道小马路，最便宜 |
| **AHB** | 高性能系统 | **单车道但会流水**（地址/数据重叠） |
| AXI | 超高性能 | 五车道高速路 |

常见系统结构：

```text
CPU / DMA（Master）
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

- **流水线（pipeline）传输**：地址阶段和数据阶段重叠，下一笔的地址可以在上一笔的数据阶段发出，实现高吞吐。
- **突发传输（burst）**：支持 4/8/16 拍和不定长 burst。
- **多 Master**：支持多个主机，需要仲裁器。
- **等待状态**：Slave 用 `HREADYOUT` 插入等待。
- **分离读写数据总线**：`HWDATA` 和 `HRDATA` 独立。
- 所有信号以 `H` 开头（HCLK、HADDR...）。

> 记忆：**AHB 是"主干道"追求吞吐，APB 是"小区路"追求简单低功耗。**

### 1.1 AHB 与 APB 的区别

| 对比项 | AHB | APB |
|---|---|---|
| 定位 | 高性能主干总线 | 低速外设总线 |
| 传输方式 | **流水线**（地址/数据重叠） | 两拍（SETUP + ACCESS） |
| burst | ✅ 支持 | ❌ 不支持 |
| 多主机 | ✅ 支持（需仲裁） | ❌ 单主机 |
| 等待状态 | ✅ HREADY | ✅ PREADY |
| 错误响应 | ✅ HRESP（ERROR 两拍） | ✅ PSLVERR（单拍） |
| 读写数据总线 | 分离（HWDATA/HRDATA） | 分离（PWDATA/PRDATA） |

> 记忆：**APB 是"点单→等上菜→再点下一单"（串行）；AHB 是"前一个菜还在做，后一个已经下单"（流水线）。**

---

## 2. 信号总览

### 2.1 全局信号

| 信号 | 方向 | 作用 |
|---|---|---|
| `HCLK` | Clock -> all | 总线时钟，所有信号时序相对 HCLK 上升沿 |
| `HRESETn` | Reset -> all | **唯一低有效信号**；可异步断言、同步撤销 |

### 2.2 Master 信号（Master 产生）

| 信号 | 方向 | 作用 |
|---|---|---|
| `HADDR` | -> Sub/Decoder | 字节地址（建议 10~64 位） |
| `HTRANS[1:0]` | -> Sub | 传输类型：IDLE/BUSY/NONSEQ/SEQ |
| `HWRITE` | -> Sub | 1=写，0=读；**整个 burst 内必须保持不变** |
| `HSIZE[2:0]` | -> Sub | 传输大小：8/16/32/64...位 |
| `HBURST[2:0]` | -> Sub | burst 类型：SINGLE/INCR/WRAP4/INCR4... |
| `HWDATA` | -> Sub | 写数据总线 |
| `HWSTRB` | -> Sub | 写 strobe：每个字节一位（可选特性） |
| `HPROT` | -> Sub | 保护控制：访问类型信息 |
| `HMASTLOCK` | -> Sub | 锁定传输指示（如 SWP 指令） |
| `HNONSEC` | -> Sub | 安全/非安全传输（AHB5） |

### 2.3 Slave 信号（Slave 产生）

| 信号 | 方向 | 作用 |
|---|---|---|
| `HRDATA` | -> Multiplexor | 读数据总线 |
| `HREADYOUT` | -> Multiplexor | **1=传输完成；0=延长传输（插等待）** |
| `HRESP` | -> Multiplexor | 0=OKAY，1=ERROR |
| `HEXOKAY` | -> Multiplexor | 独占访问结果（AHB5） |

### 2.4 Decoder 与 Multiplexor 信号

| 信号 | 产生者 | 作用 |
|---|---|---|
| `HSELx` | Decoder | 每个 Slave 一根选择线，地址译码结果 |
| `HREADY` | Multiplexor | **路由后的全局就绪**：HIGH=上一传输完成 |
| `HRDATA`/`HRESP`/`HEXOKAY` | Multiplexor | 从被选中的 Slave 选通给 Master |

> **先这样理解：** 当前是谁的数据阶段，互联就选择谁的 `HREADYOUT` 作为全局 `HREADY`。其他未被访问的 Slave 不参与本次完成判断。

---

## 3. 传输流程

### 3.1 一次传输的两个阶段

```text
地址阶段（Address phase）: 一个 HCLK 周期，Master 驱动地址和控制信号
数据阶段（Data phase）  : 一个或多个周期，用 HREADY 控制长度
```

最简单的零等待传输：1 个地址周期 + 1 个数据周期。

```text
       周期        [   0   ] [   1   ] [   2   ]
       HADDR      [   A   ] [   B   ] [ IDLE  ]
       HWRITE     [  写A  ] [  读B  ] [   -   ]
       HWDATA               [Data(A)]
       HRDATA                         [Data(B)]
       HREADY     [   1   ] [   1   ] [   1   ]
                  |--A 地址--|--A 数据--|
                            |--B 地址--|--B 数据--|
```

**关键：地址阶段和数据阶段重叠**——传输 A 的数据阶段同时也是传输 B 的地址阶段。这是 AHB 流水线的本质，也是高性能的来源。

**类比**：餐厅点餐流水线——前一个客人的菜还在做（数据阶段 A），后一个客人已经在下单了（地址阶段 B）。

### 3.2 等待状态（wait states）

Slave 用 `HREADYOUT` 拉低来延长数据阶段：

```text
      HREADY   [1]  [0]  [0]  [1]     <- 两个等待状态
```

- 等待期间**写数据必须保持稳定**；读数据只需在传输完成时有效。
- **延长数据阶段会顺带延长下一笔传输的地址阶段**（流水线被阻塞）。
- 对有效的 `NONSEQ/SEQ`，等待期间地址和控制通常必须保持；`IDLE` 和 `BUSY` 存在规范允许的 `HTRANS` 变化例外。

等待期间允许的变化：

| 当前 `HTRANS` | 可以改成 | 结果 |
|---|---|---|
| `IDLE` | `NONSEQ` | 开始一笔新的有效传输 |
| 固定长度 burst 的 `BUSY` | `SEQ` | 继续原 burst |
| 不定长 `INCR` 的 `BUSY` | `SEQ` | 继续原 burst |
| 不定长 `INCR` 的 `BUSY` | `IDLE`/`NONSEQ` | 终止原 burst |
| 已经有效的 `NONSEQ/SEQ` | 通常不能改变 | 保持到 `HREADY=1` |

**理解**：`IDLE/BUSY` 是"还没有真正提交访问"，`NONSEQ/SEQ` 是"访问已经摆到总线上"——后者一旦出现，就不能在完成前随意撤换。

**注意**：`BUSY` 只表示 Master 暂时不能给出 burst 的下一拍，**不表示 Slave 正忙**。Slave 必须忽略 `BUSY`，并返回零等待 OKAY。

> 记忆：**HREADY=0 插等待、HREADY=1 完成；Slave 有最大等待数限制，否则会拖垮整个总线性能。**

### 3.3 传输类型（HTRANS）

**为什么 AHB 需要 HTRANS？** 因为它是单总线流水线，主设备必须告诉总线"我这拍发的是新传输还是续传"——AXI 不需要这个（分通道天然区分）。

| HTRANS[1:0] | 类型 | 含义 | 类比 |
|---|---|---|---|
| 00 | IDLE | 无数据传输，Slave 必须零等待 OKAY 并忽略 | 空车 |
| 01 | BUSY | burst 中间插入空闲周期（Master 还没准备好下一拍） | 堵车但没下高速 |
| 10 | NONSEQ | **单笔传输或 burst 的第一拍**，地址与上一笔无关 | 换路线重新出发 |
| 11 | SEQ | burst 的后续拍，地址 = 上一地址 + 传输大小（wrap 则绕回） | 沿原路线继续 |

**关键规则（面试爱考）**：

- burst 的**第一拍必须是 NONSEQ**；
- 后续拍是 **SEQ**（连续地址）或 BUSY（主设备暂时没数据）；
- **固定长度 burst 最后一拍不能是 BUSY**（必须以 SEQ 结束）。

> 记忆：**NONSEQ 开头、SEQ 续拍、BUSY 中间歇、IDLE 没活干。单个传输 = NONSEQ（长度 1 的 burst）。**

### 3.4 传输大小（HSIZE）

| HSIZE[2:0] | 大小 |
|---|---|
| 000 | 8 位（Byte） |
| 001 | 16 位（Halfword） |
| 010 | 32 位（Word） |
| 011 | 64 位（Doubleword） |
| 100~111 | 128/256/512/1024 位 |

> 注意：HSIZE 必须 ≤ 数据总线宽度（32 位总线只能用 000/001/010）；**HSIZE 在整个 burst 内保持不变**。

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

**INCR（递增）**：地址逐拍递增，不绕界。**WRAP（回卷）**：地址到达边界时绕回。

**WRAP 边界计算**（三步走，别背错）：

```text
boundary_size = beat_count × bytes_per_beat        // 窗口大小
boundary_base = floor(start / boundary_size) × boundary_size  // 窗口起点（向下对齐）
address(n) = boundary_base + ((start - boundary_base + n × bytes_per_beat) mod boundary_size)
```

例：WRAP4 word（4 字节）从 `0x38` 开始，窗口 `0x30~0x3F`：

```text
0x38 -> 0x3C -> 0x30（回卷！）-> 0x34
```

**为什么引入 WRAP？** 用于 cache line 填充（critical-word-first）：先取 CPU 要的字，再回绕补齐整行，减少 CPU 等待。

**burst 约束（必背）**：

- 所有有效地址必须按 `HSIZE` 对齐。
- `HSIZE`、`HWRITE`、`HBURST` 在同一 burst 内保持一致。
- 固定长度 `INCR4/8/16`、`WRAP4/8/16` 必须以 `SEQ` 结束，**不能以 `BUSY` 结束**。
- `SINGLE` 后不能紧跟 `BUSY`。
- 不定长 `INCR` 可以在 `BUSY` 后用 `IDLE` 或 `NONSEQ` 结束。
- **递增 burst 的所有地址必须位于同一个 1KB 区域**（AHB 是 1KB，AXI 是 4KB！）。

---

## 4. 总线互联

单 Master 系统只需要 Decoder + Multiplexor：

- **Decoder（地址译码）**：译码 HADDR，产生 HSELx 选中对应 Slave。
- **Multiplexor（多路选择）**：把当前数据阶段对应 Slave 的 `HRDATA/HRESP/HREADYOUT` 路由给 Master，形成全局 `HREADY`。

多 Master 系统需要 interconnect（含仲裁），仲裁器决定哪个 Master 获得总线。

### 4.1 地址译码与 1KB 边界

地址译码在地址阶段产生 `HSELx`，返回 mux 则在数据阶段选择目标的响应。**因为两个阶段会重叠，返回选择必须把地址阶段的译码结果延迟一拍，不能直接使用当前 `HADDR`。**

规范要求分配给一个 Slave 的最小地址空间为 **1KB**，并且起止地址位于 1KB 边界。Master 不跨 1KB 发递增 burst，正是为了避免 burst 中途切换到另一个译码区域。

### 4.2 默认 Slave

如果系统地址图存在空洞，互联必须提供默认 Slave：

| 地址是否命中 | `HTRANS` | 默认响应 |
|---|---|---|
| 否 | `NONSEQ/SEQ` | **两周期 ERROR** |
| 否 | `IDLE/BUSY` | 零等待 OKAY，并忽略 |

这样，访问不存在地址时不会让总线悬空；同时，IDLE 和 BUSY 也不会被误报成错误。

> 验证提示：**地址译码和返回通路选择是 AHB 环境最常见的 bug 来源**——返回选择必须使用数据阶段对应的目标，不能直接跟随当前 `HADDR`。

---

## 5. Slave 响应（HRESP + HREADYOUT）

完整的传输响应 = HRESP + HREADYOUT 的组合：

| HRESP | HREADYOUT | 含义 |
|---|---|---|
| 0（OKAY） | 0 | 传输挂起（还在等待） |
| 0（OKAY） | 1 | **传输成功完成** |
| 1（ERROR） | 0 | **ERROR 响应第一拍** |
| 1（ERROR） | 1 | **ERROR 响应第二拍** |

### 5.1 三种完成方式

1. **立即完成**：HREADY HIGH + OKAY（一拍）。
2. **插入等待后完成**：HREADYOUT 拉低若干拍，最后 OKAY 完成。
3. **ERROR 响应**：报错（如写只读地址），必须两拍。

### 5.2 ERROR 响应的两拍机制（重点）

**ERROR 响应必须占两个周期**（OKAY 可以单周期完成）：

```text
第一拍：HRESP=ERROR, HREADYOUT=0（延长一拍，总线停住锁存错误）
第二拍：HRESP=ERROR, HREADYOUT=1（结束，释放总线）
```

**为什么必须两拍？** 因为总线是流水线的，Slave 发出 ERROR 时，**下一笔传输的地址已经广播到总线上了**。两拍给 Master 时间取消下一笔访问、把 HTRANS 改成 IDLE。

**典型 error 场景**：地址译码无目标、访问保护区域、写只读寄存器、从机内部错误、不支持的传输类型。

> 记忆：**OKAY 一拍、ERROR 两拍**。ERROR 后 Master 可以取消 burst 剩余拍（也可以继续）；读 ERROR 时建议 HRDATA 驱 0。

### 5.3 HREADYOUT 与 HREADY 的分工（核心理解）

**每个从机有自己的 HREADYOUT（自己能不能完成）；interconnect 按"当前数据阶段的目标从机"选一个，合成全局 HREADY 广播回去。**

为什么需要两个信号？因为流水线错位——拍 N 的**数据阶段**对应拍 N-1 的**地址阶段**，同一拍可能有"从机 A 的数据"和"从机 B 的地址"同时在总线上。总线该听谁的 HREADY？**听数据阶段目标（A）的**。

```text
拍1: 地址阶段 → 目标 A（慢）
拍2: 地址阶段 → 目标 B（快）
     数据阶段 → 传 A 的数据
     → interconnect 选 HREADYOUT_A → A 没准备好 → 全局 HREADY=0 → 所有人都等
```

**从机完成判断**：用 `HSEL && HREADY`（被选中 && 全局就绪），**不能只看自己的 HREADYOUT**——全局可能被别的从机拉低，从机必须服从全局。

> 一句话：**HREADYOUT 是从机向总线"报状态"的声明口，HREADY 是 interconnect 合成的全局流控；从机完成看 HSEL && HREADY。**

---

## 6. 数据总线与端序

- **HWDATA**：Master 写数据。传输被延长时数据要保持稳定直到 HREADY HIGH。
- **HRDATA**：Slave 读数据，经过 Multiplexor 到 Master。
- 窄传输（如 16 位传在 32 位总线上）：只驱动对应字节通道，Slave 按端序选正确的字节通道。

### 6.1 窄传输与 byte lane

`HSIZE` 表示本拍真正传输多少字节，数据总线宽度表示接口最多能同时搬运多少字节。32 位总线上的 byte 或 halfword 访问只使用部分 byte lane。

判断有效通道时要同时考虑：

1. `HADDR` 的低位。
2. `HSIZE` 指定的传输大小。
3. 接口采用的端序。
4. 写传输是否实现 `HWSTRB`。

### 6.2 端序

AHB 不只有简单的 little-endian 与 big-endian 两种。规范区分：

| 端序 | 基本含义 |
|---|---|
| Little-endian | 最低地址对应最低有效字节 |
| Byte-invariant big-endian | 最低地址对应最高有效字节 |
| Word-invariant big-endian | 大于 word 时按 word 块组织，块内采用 big-endian |

做窄传输或宽度转换时，应明确写出地址、传输大小、总线宽度和有效 lane，再进行映射。

### 6.3 写 strobe `HWSTRB`

`HWSTRB` 是可选的数据阶段信号，每一位固定对应 `HWDATA` 的一个字节：`HWSTRB[n]` 对应 `HWDATA[(8n)+7:(8n)]`。

- `HWSTRB` 与 `HWDATA` 同相位，等待期间必须保持。
- 每个 burst beat 的 `HWSTRB` 可以不同。
- 全部 strobe 为 0 是允许的，表示这一拍不更新任何字节。
- 非活动 byte lane 对应的 strobe 被 Slave 忽略。
- strobe 与物理数据位的对应不随端序改变。

---

## 7. 时钟、复位与信号有效性

`HRESETn` 是协议中唯一低有效信号。它可以异步断言，但必须在 `HCLK` 上升沿之后**同步撤销**。组件还应定义复位至少保持多少个周期。

**复位期间**：

- Master 必须让 `HTRANS=IDLE`，并保证地址控制信号处于合法电平。
- Slave 必须让 `HREADYOUT=HIGH`，**避免复位状态把总线永久阻塞**（中茵微考过"hready_out 的复位值"——答案是 HIGH）。

信号并不是每个周期都具有完整业务含义：

| 条件 | 必须有效的主要信号 |
|---|---|
| `HTRANS != IDLE` | 地址和控制属性 |
| 写数据阶段 | `HWDATA/HWSTRB` |
| 读数据阶段完成 | `HRDATA` |
| 数据阶段 | `HRESP/HREADYOUT` |

不要求有效的 payload 可以取任意值，但推荐驱动为 0 或 X；无效 byte lane 推荐驱动 0，避免不同传输之间泄漏旧数据。

---

## 8. AHB5 扩展：属性、原子性与独占访问

### 8.1 `HPROT` 和内存类型

基础 `HPROT[3:0]` 描述指令/数据、特权级以及访问能否缓冲或修改。初学时先掌握 `[1:0]` 的访问类型和权限：

| 位 | 名称 | 作用 |
|---|---|---|
| `HPROT[0]` | Data/Instruction | 1 为数据访问，0 为指令取值 |
| `HPROT[1]` | Privileged | 1 为特权访问 |
| `HPROT[2]` | Bufferable | 说明写响应能否来自中间点 |
| `HPROT[3]` | Modifiable | 传输特征是否允许修改 |
| `HPROT[4]` | Lookup | 是否必须查询 cache |
| `HPROT[5]` | Allocate | 是否建议进行 cache 分配 |
| `HPROT[6]` | Shareable | 是否与其他 Master 共享 |

这些属性会影响互联是否允许缓冲、预取、合并或改变响应点，不能把它们当成没有作用的 sideband。

### 8.2 安全访问 `HNONSEC`

接口声明支持 Secure_Transfers 时，`HNONSEC=0` 表示 Secure，`HNONSEC=1` 表示 Non-secure。权限检查通常由互联或目标完成，属性必须与地址阶段一起采样。

### 8.3 锁定、单副本原子性与 Exclusive

这三个概念不同：

| 概念 | 解决的问题 |
|---|---|
| `HMASTLOCK` | 一个传输序列是否具有锁定语义 |
| Single-copy atomicity | 其他观察者是否可能看到部分更新 |
| Exclusive Transfer | 独占读之后的条件式写是否成功 |

Exclusive access 的典型流程：

1. Master 对地址执行 Exclusive Read。
2. Exclusive Access Monitor 记录地址和 `HMASTER`。
3. Master 根据读值计算新值。
4. Master 对同一地址执行 Exclusive Write。
5. 若期间没有冲突，写成功并返回 `HEXOKAY`；否则写失败且不能更新内存。

Exclusive Transfer 的三个重要限制：**只能是单 beat、地址必须按 `HSIZE` 对齐、不能包含 `BUSY`**。

### 8.4 用户信号与接口奇偶校验

AHB5 允许 `HAUSER/HWUSER/HRUSER/HBUSER` 随请求、写数据、读数据和响应传递项目自定义信息。协议规定相位和有效条件，但不规定业务编码。

Issue C 定义的 check 信号可保护地址、控制、数据和响应接口。它属于接口传输保护，**不等同于存储器 ECC**。

---

## 9. 总结

### 9.1 学习重点排序

| 优先级 | 内容 |
|---|---|
| 🔴 高 | 两阶段传输与流水线（地址/数据重叠） |
| 🔴 高 | 传输类型（IDLE/BUSY/NONSEQ/SEQ） |
| 🔴 高 | burst 类型与 WRAP 边界计算 |
| 🔴 高 | HREADY 等待状态与 HREADYOUT 综合 |
| 🟡 中 | ERROR 两拍响应机制 |
| 🟡 中 | HSIZE/HBURST 编码表 |
| 🟢 进阶 | 锁定传输、写 strobe、端序、AHB5 独占访问 |
| 🟢 进阶 | reset、信号有效性、用户信号和奇偶校验 |

### 9.2 易错点

| 易错点 | 正确理解 |
|---|---|
| AHB 与 APB 一样两拍完成 | 错，AHB 是流水线，地址/数据重叠 |
| OKAY 和 ERROR 都是单拍 | 错，ERROR 必须两拍 |
| BUSY 可以在 burst 末尾 | 固定长度 burst 必须以 SEQ 结束 |
| WRAP 和 INCR 一样 | 错，WRAP 在边界绕回 |
| 所有 Slave 都必须 ready 才能推进 | 错，互联选择当前数据阶段目标的 HREADYOUT 形成 HREADY |
| HSIZE 可以大于总线宽度 | 错，HSIZE ≤ 数据总线宽度 |
| 从机完成只看自己的 HREADYOUT | 错，完成看 HSEL && HREADY（全局） |
| 边界是 4KB | AHB 是 1KB（AXI 才是 4KB） |

---

## 参考资料

- Arm《AMBA AHB Protocol Specification》，ARM IHI 0033C。
