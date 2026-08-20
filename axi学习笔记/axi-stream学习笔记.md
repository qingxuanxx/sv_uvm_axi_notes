# AMBA AXI4-Stream 学习笔记：从 Beat（拍）握手到 Packet（数据包）

> 适合读者：已经理解 VALID/READY，希望学习视频流、网络包、DSP 数据链路或 FPGA 中的流接口。
>
> 学习目标：理解 AXI4-Stream 与 memory-mapped AXI 的区别；正确处理反压（backpressure）、packet 边界和伴随信息（sideband）；掌握基本 monitor 与协议断言。

---

## 0. 文档定位与版本说明

本文主要依据 Arm《AMBA AXI-Stream Protocol Specification》ARM IHI 0051B。

AXI4-Stream 常简称：

- AXI-Stream
- AXIS
- AXI4S

规范新版术语使用 Transmitter 和 Receiver。工程资料中也常写 Master 和 Slave：

| 本文主要术语 | 常见旧称 | 方向 |
|---|---|---|
| Transmitter | Master / Source | 驱动 `TVALID` 和 payload |
| Receiver | Slave / Sink | 驱动 `TREADY` |

> **阅读主线：** 先用 `TVALID/TREADY` 判断一个 beat 是否真的传输，再用 `TKEEP/TSTRB` 判断哪些字节有效，最后用 `TLAST` 拼成 packet。
---

## 1. AXI4-Stream 是什么

AXI4-Stream 是单向、无地址的数据流协议。

```text
Transmitter  -- TDATA + sideband -->  Receiver
Transmitter  -- TVALID ----------->  Receiver
Transmitter  <- TREADY ------------  Receiver
```

典型应用：

- 视频像素流。
- 音频采样流。
- 网络 packet。
- DSP 流水线。
- ADC/DAC 数据。
- FPGA IP 之间的高速单向链路。
- DMA 的 stream side 与 memory-mapped side 之间转换。

### 1.1 它没有地址

AXI4-Stream 不告诉 Receiver “把数据写到哪个内存地址”。它只描述：

```text
现在有一个 beat，可否接收？
这个 beat 哪些字节有效？
它是不是 packet 的最后一拍？
它属于哪个 stream、要去哪个 destination？
```

### 1.2 单向不等于不能双向通信

一个 AXI4-Stream interface 只负责一个方向。若系统需要双向数据，使用两套独立接口：

```text
A -> B : 一套 AXI4-Stream
B -> A : 另一套 AXI4-Stream
```

| 对比项 | AXI4 memory-mapped | AXI4-Stream |
|---|---|---|
| 地址 | 有 AW/AR 地址 | 无地址 |
| 通道 | AW、W、B、AR、R 五通道 | 一个主数据流通道 |
| 方向 | 一套接口同时支持读写 | 一套接口单向 |
| 响应 | BRESP/RRESP | 没有通用 response |
| Burst | 地址定义的 burst | 连续 beat，可用 TLAST 划 packet |
| ID | 事务 ID | TID 表示 stream 标识 |
| 路由 | 地址译码 | 可用 TDEST |
| 典型用途 | 访问内存和寄存器 | 搬运连续数据/packet |

AXI4-Stream 的 `TID` 不是 AXI4 的 transaction ID。它描述 stream 身份，而不是 outstanding memory transaction。

---

## 2. 信号总览

### 2.1 必学信号

| 信号 | 方向 | 作用 |
|---|---|---|
| `ACLK` | Clock -> all | 上升沿采样 |
| `ARESETn` | Reset -> all | 低有效复位 |
| `TVALID` | Transmitter -> Receiver | 当前 payload 有效 |
| `TREADY` | Receiver -> Transmitter | 可以接收当前 payload |
| `TDATA` | Transmitter -> Receiver | 数据 |

### 2.2 常用 sideband

| 信号 | 作用 |
|---|---|
| `TKEEP` | 每字节是否必须保留/传输 |
| `TSTRB` | 已保留字节是 data byte 还是 position byte |
| `TLAST` | packet 边界，表示最后一个 transfer |
| `TID` | stream identifier |
| `TDEST` | 目的地/路由信息 |
| `TUSER` | 用户自定义信息 |
| `TWAKEUP` | 可选唤醒信号 |

payload 通常指：

```text
TDATA + TKEEP + TSTRB + TLAST + TID + TDEST + TUSER
```

---

## 3. VALID/READY 与阻塞稳定性

一次 transfer 发生在：

```systemverilog
// 必须在 ACLK 上升沿采样；只有双方同时为 1 才消费当前 beat。
t_fire = TVALID && TREADY;
```

### 3.1 三种合法关系

```text
TVALID 先，TREADY 后
TREADY 先，TVALID 后
TVALID 与 TREADY 同拍
```

### 3.2 Transmitter 的铁律

Transmitter 不允许等待 `TREADY=1` 才拉高 `TVALID`。

一旦 `TVALID=1`：

- 必须保持到真正握手。
- `TDATA` 和所有 sideband 必须保持稳定。
- 不能因为 Receiver 暂时不 ready 就丢弃或换成下一个 beat。

### 3.3 Receiver 的自由度

Receiver 可以：

- 提前拉高 `TREADY`，实现单拍接收。
- 看见 `TVALID` 后再拉高 `TREADY`。
- 在没有 `TVALID` 时改变 `TREADY`。

### 3.4 Backpressure

`TREADY=0` 就是 backpressure，表示下游暂时不能接收。

```text
TVALID=1, TREADY=0 -> 当前 beat 被阻塞，没有 transfer
TVALID=1, TREADY=1 -> 当前 beat 在上升沿被接收
```

### 3.5 连续满带宽传输

```text
Cycle       1      2      3      4
TVALID      1      1      1      1
TREADY      1      1      1      1
TDATA      D0     D1     D2     D3
```

每拍完成一个 transfer。`TVALID` 不需要在相邻 beat 之间拉低。

假设：

```text
Cycle       1      2      3      4
TVALID      1      1      1      1
TREADY      0      0      0      1
```

Cycle 1-3 都没有传输。Cycle 4 才真正接收。

因此 Cycle 1-4 之间必须保持：

- `TDATA`
- `TKEEP`
- `TSTRB`
- `TLAST`
- `TID`
- `TDEST`
- `TUSER`

如果 `TDATA` 每拍自增，下游最终只看到最后一个值，前面的数据已经丢失。

---

## 4. Byte、Beat 与 TKEEP/TSTRB

### 4.1 Byte lane

`TDATA` 按 8-bit byte lane 划分：

```text
byte 0 -> TDATA[7:0]
byte 1 -> TDATA[15:8]
byte 2 -> TDATA[23:16]
...
```

### 4.2 Transfer / Beat

一次 `TVALID && TREADY` 是一个 transfer，也常叫一个 beat。

一个 beat 可以包含多个 byte。

### 4.3 Packet

一组相关 transfer 可以组成一个 packet，最后一拍用 `TLAST=1` 标记。

```text
beat0 TLAST=0
beat1 TLAST=0
beat2 TLAST=1  -> packet end
```

### 4.4 Stream

相同 `TID` 和 `TDEST` 的 transfer 属于同一个 stream。不同 stream 可以共享一条物理 AXI4-Stream link。

规范区分：

### 4.5 Data byte

包含有效数据，必须保留内容和相对顺序。

### 4.6 Position byte

不包含有效数据值，但它的位置有意义，必须保留这个位置。常用于部分更新或保持字节位置关系。

### 4.7 Null byte

既不包含有效数据，位置也无须保留。interconnect 可以插入或移除 null byte，只要不破坏其他字节语义。

这就是为什么 AXI4-Stream 同时有 `TKEEP` 和 `TSTRB`。

最简单的记法是：`TKEEP` 先回答“这个字节位置要不要保留”，`TSTRB` 再回答“保留下来的字节是不是有效数据”。
对于 byte lane `x`：

```text
TKEEP[x] <-> TDATA[8x +: 8]
TSTRB[x] <-> TDATA[8x +: 8]
```

组合含义：

| `TKEEP` | `TSTRB` | byte 类型 |
|---:|---:|---|
| 0 | 0 | Null byte，可移除 |
| 0 | 1 | 保留组合，不应产生 |
| 1 | 0 | Position byte，位置保留，数据值无效 |
| 1 | 1 | Data byte，有效数据 |

### 4.8 只使用 `TKEEP`

很多常见 IP 不使用 position byte，等效认为保留的 byte 都是 data byte。此时可以：

```text
TSTRB = TKEEP
```

或者接口完全省略 `TSTRB`，按规范默认关系处理。

### 4.9 `TKEEP='0`

允许一个 transfer 的所有 `TKEEP` 位都是 0。

- 若 `TLAST=0`，这样的全-null transfer 可以被抑制。
- 若 `TLAST=1`，它可能仍承担 packet 结束语义，不能随意丢失 packet boundary。

### 4.10 常见最后一拍

32-bit TDATA，每拍 4 字节，一个 10-byte packet：

```text
beat 0: TKEEP=1111, TLAST=0  -> 4 bytes
beat 1: TKEEP=1111, TLAST=0  -> 4 bytes
beat 2: TKEEP=0011, TLAST=1  -> 2 bytes
```

这里假设低 byte lane 先使用，具体数据排列由系统约定和字节顺序决定。

---

## 5. TLAST 与 packet 边界

`TLAST=1` 表示当前已握手 transfer 是 packet 的最后一拍。

注意：

```text
TVALID=1, TLAST=1, TREADY=0
```

此时 packet 还没有结束，因为最后一拍尚未握手。

### 5.1 `TLAST` 的价值

- 通知 Receiver packet 完成。
- 给共享链路提供自然仲裁点。
- 防止两个 packet 被错误合并。

### 5.2 packet 内的 ID/DEST

同一个 packet 中的所有 byte 来自同一 source、去往同一 destination，`TID` 和 `TDEST` 应保持一致。

若 `TID/TDEST` 改变，应视为另一个 stream 的 transfer，而不是同一 packet 随意换身份。

### 5.3 零数据的 TLAST transfer

规范允许 `TLAST=1` 但不带 data byte 或 position byte，用于：

- 发送空 packet。
- 单独补充 packet end。
- 结束一个等待 `TLAST` 的操作。

Monitor 不能因为 `TKEEP='0` 就无条件忽略该 transfer。

---

## 6. TID 与 TDEST

### 6.1 `TID`

区分共享物理 link 上的多个数据 stream。

例如：

```text
TID=0 -> video channel 0
TID=1 -> video channel 1
```

### 6.2 `TDEST`

提供粗粒度路由信息，例如选择输出端口或目标处理单元。

### 6.3 stream 的唯一性

通常用二元组：

```text
stream_key = {TID, TDEST}
```

区分不同 stream。

### 6.4 Interleaving

不同 stream 的 transfer 可以在同一物理 link 上逐 transfer 交织：

```text
(ID=0, DEST=2, packet A beat0)
(ID=1, DEST=3, packet B beat0)
(ID=0, DEST=2, packet A beat1, TLAST)
(ID=1, DEST=3, packet B beat1, TLAST)
```

因此支持多 stream 的 monitor 不能只有一个全局 current_packet，必须按 `{TID,TDEST}` 保存上下文。

### 6.5 Ordering

同一 stream 的 transfer 必须保持顺序。不同 stream 之间没有统一的先后保证。

---

## 7. TUSER

`TUSER` 是用户自定义 sideband，常见用途：

- 视频帧起始标志。
- 错误标志。
- 时间戳。
- 元数据。
- 包分类信息。

### 7.1 两种常见语义

```text
per-byte TUSER：每个 byte 有相应用户位
per-transfer TUSER：整个 beat 共享一组信息
```

宽度转换、packing、null-byte removal 时必须保持 `TUSER` 与对应 byte 的关联。

### 7.2 最大风险

`TUSER` 的含义不由 AXI4-Stream 统一规定。两端如果对 bit 定义、有效时机或宽度转换规则理解不同，链路握手完全正常但功能仍然错误。

项目必须写清：

- 每一位含义。
- 是 per-byte 还是 per-transfer。
- 与 `TKEEP=0` 的关系。
- packet 内是否必须保持。
- 宽度转换时如何扩展/裁剪。

---

## 8. Reset

`ARESETn` 低有效。

Transmitter 在复位期间必须将 `TVALID` 拉低。复位释放后，最早在检测到 `ARESETn=1` 的时钟上升沿之后开始断言 `TVALID`。

其他 payload 在 `TVALID=0` 时可以是任意值，但工程上常清零以便波形调试。

Receiver 的 `TREADY` 复位值取决于实现容量。无论取何值，都不能让复位期间的无效 payload 被当成有效 transfer。

---

## 9. Width Conversion

### 9.1 Downsizing

把宽 beat 拆成多个窄 beat：

- 保持 byte 顺序。
- 正确拆分 `TKEEP/TSTRB`。
- `TLAST` 只放在最后一个输出 transfer。
- `TID/TDEST` 保持。
- `TUSER` 与对应 byte 保持关联。

例：64-bit -> 32-bit：

```text
input : 8 bytes, TLAST=1
output0: lower 4 bytes, TLAST=0
output1: upper 4 bytes, TLAST=1
```

若高 4 bytes 全是可删除 null byte，转换器可根据规则只输出必要 transfer，但不能丢失 `TLAST`。

### 9.2 Upsizing

把多个窄 beat 合并成一个宽 beat：

允许合并的前提包括：

- `TID` 相同。
- `TDEST` 相同。
- 不能跨越已断言的 `TLAST` packet boundary。
- byte 顺序和 sideband 关联保持。

### 9.3 Packing

移除 null byte，让有效 byte 更紧凑。Position byte 不能像 null byte 一样随意删除，因为它的位置有意义。

---

## 10. 一个 AXI4-Stream interface

```systemverilog
interface axi_stream_if #(
    // TKEEP/TSTRB 的宽度是 byte lane 数，因此 DATA_WIDTH 应为 8 的整数倍。
    parameter int DATA_WIDTH = 32,
    parameter int ID_WIDTH   = 4,
    parameter int DEST_WIDTH = 4,
    parameter int USER_WIDTH = 4
) (
    input logic ACLK,
    input logic ARESETn
);
    // Transmitter 驱动的数据与 sideband payload。
    logic [DATA_WIDTH-1:0]   TDATA;
    logic [DATA_WIDTH/8-1:0] TKEEP;
    logic [DATA_WIDTH/8-1:0] TSTRB;
    logic                    TLAST;
    logic [ID_WIDTH-1:0]     TID;
    logic [DEST_WIDTH-1:0]   TDEST;
    logic [USER_WIDTH-1:0]   TUSER;
    // TVALID 由 Transmitter 驱动，TREADY 由 Receiver 驱动。
    logic                    TVALID;
    logic                    TREADY;

    clocking tx_cb @(posedge ACLK);
        // Transmitter 驱动 payload/TVALID，只采样 Receiver 的 TREADY。
        default input #1step output #0;
        output TDATA, TKEEP, TSTRB, TLAST, TID, TDEST, TUSER, TVALID;
        input  TREADY;
    endclocking

    clocking rx_cb @(posedge ACLK);
        // Receiver 采样 payload/TVALID，只驱动自己的反压信号 TREADY。
        default input #1step output #0;
        input  TDATA, TKEEP, TSTRB, TLAST, TID, TDEST, TUSER, TVALID;
        output TREADY;
    endclocking

    clocking monitor_cb @(posedge ACLK);
        // 被动 Monitor 只观察双方信号，并在 TVALID && TREADY 时组装 beat。
        default input #1step;
        input TDATA, TKEEP, TSTRB, TLAST, TID, TDEST, TUSER;
        input TVALID, TREADY;
    endclocking
endinterface
```

若某些 sideband 不存在，最好通过参数和 generate 明确接口配置，而不是让无意义信号处于 X。

---

## 11. 验证视角

### 11.1 只在 fire 时采样

```systemverilog
// Monitor 不能只看 TVALID；阻塞期间同一个 beat 会保持多个周期。
if (vif.monitor_cb.TVALID && vif.monitor_cb.TREADY) begin
    // 只在真实 fire 事件中采一次 TDATA 和全部 sideband。
end
```

### 11.2 单 stream

```text
每次 fire：
    根据 TKEEP/TSTRB 追加 byte/position 信息
    保存 TUSER
    若 TLAST=1：发布 packet，清空当前 packet
```

### 11.3 多 stream

```systemverilog
typedef struct packed {
    // 把 TID 与 TDEST 组合成关联数组 key，分别累计多个交织 stream。
    bit [ID_WIDTH-1:0]   id;
    bit [DEST_WIDTH-1:0] dest;
} stream_key_t;
```

概念容器：

```text
packet_by_stream[{TID,TDEST}]
```

每个 stream 独立累计，遇到该 stream 的 `TLAST` 才结束对应 packet。

### 11.4 不能无条件丢弃 null-only beat

如果 `TKEEP='0` 且 `TLAST=1`，它仍可能表示空 packet 或 packet end，monitor 必须保留边界语义。

### 11.5 TVALID sticky

```systemverilog
property p_tvalid_sticky;
    @(posedge ACLK) disable iff (!ARESETn)
    // 当前 beat 未被 Receiver 接收时，下一拍 TVALID 不能撤销。
    TVALID && !TREADY |=> TVALID;
endproperty
a_tvalid_sticky: assert property (p_tvalid_sticky);
```

### 11.6 Payload 稳定

```systemverilog
property p_payload_stable;
    @(posedge ACLK) disable iff (!ARESETn)
    // backpressure 期间，数据、packet 边界和所有 stream 元数据都必须保持。
    TVALID && !TREADY
    |=> TVALID && $stable({TDATA, TKEEP, TSTRB, TLAST,
                           TID, TDEST, TUSER});
endproperty
a_payload_stable: assert property (p_payload_stable);
```

### 11.7 复位时 TVALID 为低

```systemverilog
property p_tvalid_low_in_reset;
    @(posedge ACLK)
    // 复位有效时同拍检查 TVALID=0。
    !ARESETn |-> !TVALID;
endproperty
```

### 11.8 非法 keep/strb 组合

```systemverilog
property p_strb_implies_keep;
    @(posedge ACLK) disable iff (!ARESETn)
    // 任一 TSTRB=1 的 lane 必须同时 TKEEP=1；禁止 keep=0/strb=1 组合。
    TVALID |-> ((TSTRB & ~TKEEP) == '0);
endproperty
```

即任何 `TSTRB=1` 的 byte lane 必须同时 `TKEEP=1`。

### 11.9 packet 内 ID/DEST 稳定

checker 需要记录每个 packet 是否仍在进行。

如果接口允许多个 stream 交织，就不能用一个全局状态要求 `TID` 始终不变；应按 `{TID,TDEST}` 分别保存每个 stream 的 packet 状态。

| 错误理解/实现 | 正确理解 |
|---|---|
| `TVALID=1` 就传了一拍 | 必须同时 `TREADY=1` |
| 下游不 ready 时可以换 TDATA | 阻塞期间全部 payload 必须稳定 |
| Transmitter 等 TREADY 才拉 TVALID | 可能与 Receiver 互等而死锁 |
| 相邻 beat 之间 TVALID 必须拉低 | 满带宽时可连续保持高 |
| TLAST 高就已经结束 packet | 必须 TLAST 所在 beat 真正握手 |
| TKEEP 是写使能 | 它表示 byte 是否保留在 stream 中 |
| TKEEP=0 的 beat 一定可丢 | TLAST=1 时仍有 packet 边界语义 |
| TSTRB=0 表示 null byte | 当 TKEEP=1 时表示 position byte |
| TID 等同 AXI transaction ID | 它表示 stream identifier |
| 多 stream monitor 只需一个 packet buffer | 应按 `{TID,TDEST}` 保存上下文 |
| Width converter 可以逐 beat 比较 | 应规范化为 packet/byte 级比较 |
| AXIS 自带错误响应 | 协议没有通用 response，错误通常放 TUSER 或旁路 |

---

## 12. 总结

### 12.1 学习重点排序

| 优先级 | 学习内容 |
|---|---|
| 🔴 高 | `TVALID/TREADY`、阻塞稳定性、`TKEEP/TSTRB`、`TLAST` |
| 🟡 中 | packet、`TID/TDEST`、多 stream 顺序、宽度转换 |
| 🟢 进阶 | `TUSER`、interface、monitor 和协议断言 |

### 12.2 易错点

| 易错理解 | 正确理解 |
|---|---|
| `TVALID=1` 就传了一拍 | 必须同时有 `TREADY=1` |
| 反压期间可以换 `TDATA` | 当前 beat 的全部 payload 必须保持 |
| `TLAST=1` 就结束 packet | TLAST 所在 beat 握手后才结束 |
| `TKEEP=0` 的 beat 都能丢 | 若同时有 `TLAST=1`，仍可能携带边界语义 |
| `TID` 等于 AXI transaction ID | `TID` 表示 stream 身份 |
| 多 stream 只需一个 packet buffer | 应按 `{TID,TDEST}` 分别保存上下文 |

### 12.3 最重要的 10 条规则

1. AXI4-Stream 无地址，一套接口只传一个方向。
2. Transfer 只在 `TVALID && TREADY` 上升沿发生。
3. Transmitter 不得等待 TREADY 才拉 TVALID。
4. TVALID 被阻塞时必须保持有效和全部 payload 稳定。
5. 满带宽时 TVALID 可以连续为 1，每拍一个 beat。
6. `TKEEP/TSTRB` 共同区分 data、position 和 null byte。
7. `TLAST` 只有在所在 beat 握手后才真正结束 packet。
8. `{TID,TDEST}` 常作为 stream key。
9. 不同 stream 可交织，同一 stream 必须保序。
10. Verification 应先采 beat，再按 stream 重建 packet 比较。

---

## 参考资料

- Arm, *AMBA AXI-Stream Protocol Specification*, ARM IHI 0051B, 2021。
- Arm, *Introduction to AMBA AXI4*, 102202 Issue 01, 2020（用于对比 memory-mapped AXI 的通道概念）。
