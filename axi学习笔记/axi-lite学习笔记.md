# AMBA AXI4-Lite 学习笔记：五通道握手与寄存器接口

> 适合读者：已经学过 APB，准备接触 AXI；希望先用较小的协议子集理解五通道、VALID/READY 和独立握手。
>
> 学习目标：读懂 AXI4-Lite 波形；理解 AW/W 独立握手和响应依赖；掌握基本 monitor 与协议断言。

---

## 0. 文档定位与版本边界

AXI4-Lite 是 AXI4 的简化子集，主要用于控制寄存器接口。

本文以 Arm IHI 0022H 的 AXI4-Lite 章节为主要依据，并用《Introduction to AMBA AXI4》解释五通道和握手。

你提供的 IHI 0022L 属于更新的 AMBA 5 规范。阅读时可以参考通用思想，但不要把其中 AXI5-Lite 的新增信号当成 AXI4-Lite 必选信号。

> **阅读主线：** 五个通道各自握手；写地址 AW 和写数据 W 可以任意先后；收到两者后才产生写响应 B。

术语说明：

| 本文术语 | 新版 Arm 常用术语 |
|---|---|
| Master | Manager / Requester |
| Slave | Subordinate / Completer |

工程和 IP 文档中仍大量使用 AXI4 的 Master/Slave 命名，本文保留它，便于对照信号方向。

---

## 1. AXI4-Lite 是什么

AXI4-Lite 可以理解为：

```text
保留 AXI 的五个独立通道和 VALID/READY 握手
去掉 burst、ID、乱序完成和 exclusive access
每次只传输一个 data beat
```

典型用途：

- CPU 配置 DMA 寄存器。
- 读写 GPIO、UART、Timer 等控制寄存器。
- FPGA 中处理器系统访问自定义 IP 寄存器。
- 低吞吐量状态和控制通路。

不适合：

- 大块内存搬运。
- 高吞吐量连续数据。
- 依赖 burst、多个 ID 或乱序返回的场景。

### 1.1 与 APB 的关键区别

| 对比项 | APB | AXI4-Lite |
|---|---|---|
| 通道 | 地址、数据共用一次两阶段传输 | 5 个独立通道 |
| 握手 | `PSEL/PENABLE/PREADY` | 每通道 `VALID/READY` |
| 最少周期 | 至少 2 拍 | 某个通道可 1 拍握手 |
| 读写并发 | 不支持 | 读、写通路可并发 |
| 写地址和写数据 | 同一传输中同时稳定 | 两个独立通道，可任意先后 |
| 响应 | `PSLVERR` | `BRESP/RRESP` |

AXI4-Lite 语法看似只是信号更多，真正难点是“通道彼此独立”。

---

## 2. 五个通道与信号

```mermaid
flowchart LR
    M["AXI4-Lite Master"] -->|"AW: 写地址"| S["AXI4-Lite Slave"]
    M -->|"W: 写数据"| S
    S -->|"B: 写响应"| M
    M -->|"AR: 读地址"| S
    S -->|"R: 读数据 + 读响应"| M
```

| 通道 | 方向 | 内容 |
|---|---|---|
| AW | Master -> Slave | 写地址和写保护属性 |
| W | Master -> Slave | 写数据和字节使能 |
| B | Slave -> Master | 写响应 |
| AR | Master -> Slave | 读地址和读保护属性 |
| R | Slave -> Master | 读数据和读响应 |

### 2.1 为什么读没有单独响应通道

读数据必须从 Slave 返回 Master，因此 `RRESP` 可以与 `RDATA` 一起放在 R 通道。

写数据方向是 Master 到 Slave，写结果要反方向返回，所以需要单独的 B 通道。

### 2.2 全局信号

| 信号 | 作用 |
|---|---|
| `ACLK` | 所有输入在上升沿采样 |
| `ARESETn` | 低有效复位 |

### 2.3 AW 写地址通道

| 信号 | 方向 | 作用 |
|---|---|---|
| `AWADDR` | M -> S | 写地址 |
| `AWPROT[2:0]` | M -> S | 特权、安全、指令/数据属性 |
| `AWVALID` | M -> S | 写地址有效 |
| `AWREADY` | S -> M | Slave 可以接收写地址 |

### 2.4 W 写数据通道

| 信号 | 方向 | 作用 |
|---|---|---|
| `WDATA` | M -> S | 写数据 |
| `WSTRB` | M -> S | 每字节写使能 |
| `WVALID` | M -> S | 写数据有效 |
| `WREADY` | S -> M | Slave 可以接收写数据 |

### 2.5 B 写响应通道

| 信号 | 方向 | 作用 |
|---|---|---|
| `BRESP[1:0]` | S -> M | 写响应状态 |
| `BVALID` | S -> M | 写响应有效 |
| `BREADY` | M -> S | Master 可以接收写响应 |

### 2.6 AR 读地址通道

| 信号 | 方向 | 作用 |
|---|---|---|
| `ARADDR` | M -> S | 读地址 |
| `ARPROT[2:0]` | M -> S | 访问属性 |
| `ARVALID` | M -> S | 读地址有效 |
| `ARREADY` | S -> M | Slave 可以接收读地址 |

### 2.7 R 读数据通道

| 信号 | 方向 | 作用 |
|---|---|---|
| `RDATA` | S -> M | 读数据 |
| `RRESP[1:0]` | S -> M | 读响应状态 |
| `RVALID` | S -> M | 读数据和响应有效 |
| `RREADY` | M -> S | Master 可以接收读返回 |

---

## 3. AXI4-Lite 去掉了什么

AXI4-Lite 的核心限制：

- 每笔事务只有 1 个 data beat。
- 数据总线宽度固定为 32 位或 64 位。
- 不支持 AXI ID。
- 不支持 burst。
- 不支持 exclusive access。
- 不支持数据交织。
- 所有事务按顺序完成。

与完整 AXI4 信号的等效关系：

| 完整 AXI4 字段 | AXI4-Lite 等效含义 |
|---|---|
| `AxLEN` | 固定为 0，即 1 beat |
| `AxSIZE` | 固定为数据总线宽度 |
| `AxBURST` | 无意义，因为只有 1 beat |
| `AxLOCK` | 固定 Normal access |
| `AxCACHE` | 固定 Non-modifiable、Non-bufferable |
| `WLAST/RLAST` | 每笔都是最后一拍，等效为 1 |
| ID signals | 不存在 |

“Outstanding”指请求已经接收，但最终响应还没有返回。AXI4-Lite 可以同时存在多笔这样的事务，但没有 ID 可供区分，因此响应必须保持顺序。

最简单的 Slave 可以在上一笔完成前拉低 `READY`，把自己限制成一次只处理一笔。

---

## 4. VALID/READY 握手

每个通道都使用相同规则：

```systemverilog
// 该表达式必须在 ACLK 上升沿采样；组合值为 1 才完成一次通道 transfer。
transfer = VALID && READY;
```

### 4.1 Source 与 Destination

在每个通道里，Source 是“提供内容的一方”，负责 `VALID`；Destination 是“接收内容的一方”，负责 `READY`。它们不一定分别等于 Master 和 Slave。

| 通道 | Source：驱动 VALID | Destination：驱动 READY |
|---|---|---|
| AW | Master | Slave |
| W | Master | Slave |
| B | Slave | Master |
| AR | Master | Slave |
| R | Slave | Master |

“VALID 一定由 Master 驱动”是错误的。B、R 通道的 Source 是 Slave。

可以把握手想成：Source 说“东西已经放好了”，Destination 说“我现在能接”。只有两句话在同一个上升沿同时成立，传输才发生。
### 4.2 三种合法时序

```text
情况 A：VALID 先到，READY 后到
情况 B：READY 先到，VALID 后到
情况 C：VALID 与 READY 同拍到
```

只要在某个上升沿两者同时为 `1`，该通道完成一次传输。

### 4.3 Source 的铁律

Source 不允许等待 READY 后才拉高 VALID。

```text
错误：Master 等 AWREADY，Slave 又等 AWVALID -> 永久死锁
```

正确规则：

1. Source 有有效 payload 时自主拉高 VALID。
2. 一旦 VALID 拉高，在握手完成前必须保持 VALID。
3. 握手完成前 payload 必须稳定。

### 4.4 Destination 可以等待 VALID

Destination 可以：

- 提前拉高 READY。
- 看到 VALID 后再拉高 READY。
- 在 VALID 出现前自由改变 READY。

为了单拍接收，常建议 READY 在确实有容量时提前为高。

### 4.5 接口组合路径限制

AXI 接口输入到输出之间不能存在组合路径。实际设计通常使用寄存器、FIFO 或 skid buffer 断开组合依赖。

例如，不应简单写成：

```systemverilog
// 错误示例：AWREADY 直接由输入 WVALID 组合产生，既形成输入到输出的组合路径，
// 又错误地把本应独立的 AW、W 两个通道绑在一起。
assign AWREADY = WVALID; // 输入直接组合影响输出，且耦合两个独立通道
```

---

## 5. 写事务与写响应

一次 AXI4-Lite 写事务包含：

```text
一次 AW 握手 + 一次 W 握手 + 一次 B 握手
```

关键依赖：

```text
AW 与 W：没有固定先后关系
BVALID：必须等 AW 和 W 都已被接收后才能产生
```

### 5.1 三种合法到达顺序

```text
顺序 1：AW 先，W 后
顺序 2：W 先，AW 后
顺序 3：AW 与 W 同拍
```

Slave 必须正确处理自己声明可以接收的所有合法顺序。

### 5.2 最常见的错误实现

```systemverilog
// 错误示例：这个条件要求 AW 与 W 恰好同一拍握手。
// 如果地址先被接收、数据几拍后才到，本次写事务就会丢失。
if (AWVALID && AWREADY && WVALID && WREADY)
    do_write();
```

这个实现只接受“AW 与 W 恰好同拍握手”。若 AW 先握手后撤销，W 隔几拍才来，写事务会永久丢失。

### 5.3 正确思路：分别保存

```text
AW handshake -> 保存地址，置 aw_hold_valid
W  handshake -> 保存数据/STRB，置 w_hold_valid
两者都有效   -> 执行写操作，产生 B response
```

`BRESP` 编码：

| 值 | 名称 | AXI4-Lite 含义 |
|---|---|---|
| `2'b00` | OKAY | 正常完成 |
| `2'b01` | EXOKAY | AXI4-Lite 不支持 exclusive，通常不使用 |
| `2'b10` | SLVERR | Slave 能译码，但访问失败 |
| `2'b11` | DECERR | 通常由 interconnect 表示地址无法译码 |

Slave 拉高 `BVALID` 后，必须保持 `BRESP` 稳定，直到 `BVALID && BREADY` 握手。

写操作完成不等于 Master 已收到响应：

```text
内部寄存器更新 -> BVALID 拉高 -> 等待 BREADY -> B 通道完成
```

在 `BREADY=0` 时，Slave 不能覆盖尚未接收的响应。

---

## 6. 读事务

一次 AXI4-Lite 读事务包含：

```text
一次 AR 握手 + 一次 R 握手
```

顺序关系：

```text
必须先接收 AR
然后 Slave 才能对该请求产生 RVALID/RDATA/RRESP
```

### 6.1 R 通道阻塞

若 `RVALID=1, RREADY=0`：

- `RVALID` 必须保持为 1。
- `RDATA` 必须稳定。
- `RRESP` 必须稳定。
- 没有额外缓冲时，不应再接收会覆盖返回槽的新读请求。

---

## 7. 写字节使能 WSTRB

32 位接口中：

```text
WSTRB[0] -> WDATA[7:0]
WSTRB[1] -> WDATA[15:8]
WSTRB[2] -> WDATA[23:16]
WSTRB[3] -> WDATA[31:24]
```

按字节更新：

```systemverilog
// WSTRB 每一位对应 WDATA 的一个 8-bit byte lane。
for (int i = 0; i < DATA_WIDTH/8; i++) begin
    // 只覆盖使能字节；其余字节依靠非阻塞赋值保持原值。
    if (wstrb_q[i])
        reg_data[i*8 +: 8] <= wdata_q[i*8 +: 8];
end
```

规范允许 Slave：

- 完整支持 `WSTRB`。
- 对非存储器类寄存器忽略它，并当成全宽写。
- 检测不支持的组合并报错。

但如果 Slave 提供 memory access，就必须正确支持 `WSTRB`。

### 7.1 `WSTRB='0`

全 0 写 strobe 的事务仍可在 AXI4-Lite 接口上传递。设计不能假设 interconnect 一定会抑制它。

常见处理是正常返回 OKAY，但不更新任何字节；具体项目应明确规定。

---

## 8. AxPROT 访问属性

`AWPROT` 和 `ARPROT` 的含义相同：

| 位 | `0` | `1` |
|---|---|---|
| `[0]` | Unprivileged | Privileged |
| `[1]` | Secure | Non-secure |
| `[2]` | Data | Instruction |

简单外设经常忽略 `AxPROT`，安全系统可能据此拒绝访问。

---

## 9. 验证视角

### 9.1 VALID 被阻塞时保持

可对五个通道分别检查：

```systemverilog
property p_awvalid_sticky;
    @(posedge ACLK) disable iff (!ARESETn)
    // AW 被阻塞时，下一拍 AWVALID 仍必须为 1。
    AWVALID && !AWREADY |=> AWVALID;
endproperty

property p_bvalid_sticky;
    @(posedge ACLK) disable iff (!ARESETn)
    // 同一规则也适用于反向的 B 通道，此时 Source 是 Slave。
    BVALID && !BREADY |=> BVALID;
endproperty
```

### 9.2 Payload 在阻塞期间稳定

```systemverilog
property p_aw_stable;
    @(posedge ACLK) disable iff (!ARESETn)
    // 地址通道阻塞时，VALID 与整个 AW payload 都必须保持。
    AWVALID && !AWREADY |=> AWVALID && $stable({AWADDR, AWPROT});
endproperty

property p_w_stable;
    @(posedge ACLK) disable iff (!ARESETn)
    // 写数据阻塞时不能提前切换到下一笔数据。
    WVALID && !WREADY |=> WVALID && $stable({WDATA, WSTRB});
endproperty

property p_r_stable;
    @(posedge ACLK) disable iff (!ARESETn)
    // Master 未接收时，Slave 必须保存返回数据和响应。
    RVALID && !RREADY |=> RVALID && $stable({RDATA, RRESP});
endproperty
```

### 9.3 复位时输出 VALID 为低

规范要求 Master 的 `ARVALID/AWVALID/WVALID` 和 Slave 的 `RVALID/BVALID` 在复位期间为低。

```systemverilog
property p_valid_low_in_reset;
    @(posedge ACLK)
    // 示例同时观察双方接口；实际 checker 通常按 Master/Slave 责任拆开绑定。
    !ARESETn |-> !(AWVALID || WVALID || ARVALID || BVALID || RVALID);
endproperty
```

实际绑定断言时要根据接口角色拆分，避免一个 checker 同时驱动双方责任。

### 9.4 B 响应不能凭空出现

可在 checker 中维护已接受 AW/W 计数：

```text
aw_count += AWVALID && AWREADY
w_count  += WVALID  && WREADY
b_count  += BVALID  && BREADY

任意时刻：b_count <= min(aw_count, w_count)
```

对于 single-outstanding Slave，还可以断言 `BVALID` 产生前已各接受一次 AW 和 W。

### 9.5 写 Monitor 不能只看同拍

AW 和 W 独立，monitor 应维护两个队列：

```text
AW handshake -> aw_queue.push_back(address/prot)
W  handshake -> w_queue.push_back(data/strb)

当两个队列都非空：
    取各自最早元素，形成一个待响应 write transaction

B handshake：
    给最早待响应 write transaction 填入 BRESP 并发布
```

AXI4-Lite 没有 ID，事务必须按顺序匹配。

### 9.6 读 Monitor

```text
AR handshake -> read_request_queue.push_back(address/prot)
R  handshake -> 取最早请求，填入 RDATA/RRESP，发布事务
```

### 9.7 Monitor 关注的是握手

只有这些事件才进入队列：

```systemverilog
// fire 事件是 monitor 入队和协议计数的唯一依据；只有 VALID 不能算传输。
aw_fire = AWVALID && AWREADY;
w_fire  = WVALID  && WREADY;
b_fire  = BVALID  && BREADY;
ar_fire = ARVALID && ARREADY;
r_fire  = RVALID  && RREADY;
```

VALID 单独为高只表示 Source 正在等待，不能重复采样成多个 transfer。

| 错误 | 后果 | 正确做法 |
|---|---|---|
| 只在 AW/W 同拍时写 | 通道错开就丢事务 | 独立保存 AW 和 W |
| Source 等 READY 才拉 VALID | 双方互等导致死锁 | Source 有数据就拉 VALID |
| VALID 阻塞时改变 payload | 接收方得到不确定内容 | 保持 VALID 和 payload |
| `BVALID` 未等 AW/W 都接收 | 写响应凭空出现 | 同时追踪地址和数据 |
| `RVALID && !RREADY` 时改 RDATA | Master 采样到错误数据 | 保持 RDATA/RRESP |
| 用 VALID 单独计数 transfer | 一个阻塞传输被重复计数 | 只在 VALID && READY 计数 |
| 忽略 `WSTRB` | 部分写破坏其他字节 | 按字节合并或明确报错 |
| 把 AXI4-Lite 当 APB | 强制地址数据同拍 | 接受五通道独立性 |
| 认为 Lite 不允许 outstanding | 错过协议能力 | 无 ID 但可多笔，必须有序 |
| 未限制返回槽却继续接收请求 | 响应被覆盖 | 使用 FIFO 或拉低 READY |

---

## 10. 总结

### 10.1 学习重点排序

| 优先级 | 学习内容 |
|---|---|
| 🔴 高 | 五通道、`VALID/READY`、AW/W 独立、B/R 响应依赖 |
| 🟡 中 | backpressure、`WSTRB`、顺序和 outstanding |
| 🟢 进阶 | `AxPROT`、monitor、SVA 和实现优化 |

### 10.2 易错点

| 易错理解 | 正确理解 |
|---|---|
| `VALID=1` 就发生传输 | 必须在上升沿同时有 `VALID && READY` |
| Source 可以等 READY 再拉 VALID | 可能与对端互等而死锁 |
| AW 和 W 必须同时到达 | 两个通道独立，可以任意先后 |
| 收到 AW 就可以产生 B | 必须先接收完整的 AW 和 W |
| 阻塞期间可以改变 payload | VALID 保持时 payload 也必须稳定 |

### 10.3 最重要的 10 条规则

1. AXI4-Lite 有 AW、W、B、AR、R 五个独立通道。
2. 任一通道只在 `VALID && READY` 的上升沿传输。
3. Source 不能等待 READY 才拉 VALID。
4. VALID 一旦拉高，握手前必须保持。
5. 阻塞期间 payload 必须稳定。
6. AW 和 W 可以任意先后或同拍到达。
7. B 必须在 AW 和 W 都被接收后产生。
8. R 必须对应已经接收的 AR。
9. AXI4-Lite 每笔只有 1 beat、无 ID、按序完成。
10. Monitor 必须按各通道握手重建完整事务。

---

## 参考资料

- Arm, *AMBA AXI and ACE Protocol Specification*, ARM IHI 0022H, Part A 与 Part B, 2020。
- Arm, *Introduction to AMBA AXI4*, 102202 Issue 01, 2020。
- Arm, *AMBA AXI Protocol Specification*, ARM IHI 0022 Issue L, 2025（用于确认新版版本边界和通用术语）。
