# AMBA AXI4-Stream 学习笔记：从 Beat 握手到 Packet 传输

> 适合读者：已经理解 VALID/READY，希望学习视频流、网络包、DSP 数据链路或 FPGA 中的流接口。
>
> 学习目标：理解 AXI4-Stream 与 memory-mapped AXI 的区别；能正确处理 backpressure、`TLAST`、`TKEEP/TSTRB`、`TID/TDEST/TUSER`；能设计 packet monitor、scoreboard 和协议断言。

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

---

## 2. 与 AXI4 memory-mapped 的区别

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

## 3. 信号总览

### 3.1 必学信号

| 信号 | 方向 | 作用 |
|---|---|---|
| `ACLK` | Clock -> all | 上升沿采样 |
| `ARESETn` | Reset -> all | 低有效复位 |
| `TVALID` | Transmitter -> Receiver | 当前 payload 有效 |
| `TREADY` | Receiver -> Transmitter | 可以接收当前 payload |
| `TDATA` | Transmitter -> Receiver | 数据 |

### 3.2 常用 sideband

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

## 4. VALID/READY 握手

一次 transfer 发生在：

```systemverilog
// 必须在 ACLK 上升沿采样；只有双方同时为 1 才消费当前 beat。
t_fire = TVALID && TREADY;
```

### 4.1 三种合法关系

```text
TVALID 先，TREADY 后
TREADY 先，TVALID 后
TVALID 与 TREADY 同拍
```

### 4.2 Transmitter 的铁律

Transmitter 不允许等待 `TREADY=1` 才拉高 `TVALID`。

一旦 `TVALID=1`：

- 必须保持到真正握手。
- `TDATA` 和所有 sideband 必须保持稳定。
- 不能因为 Receiver 暂时不 ready 就丢弃或换成下一个 beat。

### 4.3 Receiver 的自由度

Receiver 可以：

- 提前拉高 `TREADY`，实现单拍接收。
- 看见 `TVALID` 后再拉高 `TREADY`。
- 在没有 `TVALID` 时改变 `TREADY`。

### 4.4 Backpressure

`TREADY=0` 就是 backpressure，表示下游暂时不能接收。

```text
TVALID=1, TREADY=0 -> 当前 beat 被阻塞，没有 transfer
TVALID=1, TREADY=1 -> 当前 beat 在上升沿被接收
```

### 4.5 连续满带宽传输

```text
Cycle       1      2      3      4
TVALID      1      1      1      1
TREADY      1      1      1      1
TDATA      D0     D1     D2     D3
```

每拍完成一个 transfer。`TVALID` 不需要在相邻 beat 之间拉低。

---

## 5. 为什么阻塞稳定性如此重要

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

## 6. Byte、Beat、Packet、Stream

### 6.1 Byte lane

`TDATA` 按 8-bit byte lane 划分：

```text
byte 0 -> TDATA[7:0]
byte 1 -> TDATA[15:8]
byte 2 -> TDATA[23:16]
...
```

### 6.2 Transfer / Beat

一次 `TVALID && TREADY` 是一个 transfer，也常叫一个 beat。

一个 beat 可以包含多个 byte。

### 6.3 Packet

一组相关 transfer 可以组成一个 packet，最后一拍用 `TLAST=1` 标记。

```text
beat0 TLAST=0
beat1 TLAST=0
beat2 TLAST=1  -> packet end
```

### 6.4 Stream

相同 `TID` 和 `TDEST` 的 transfer 属于同一个 stream。不同 stream 可以共享一条物理 AXI4-Stream link。

---

## 7. 三种 byte 类型

规范区分：

### 7.1 Data byte

包含有效数据，必须保留内容和相对顺序。

### 7.2 Position byte

不包含有效数据值，但它的位置有意义，必须保留这个位置。常用于部分更新或保持字节位置关系。

### 7.3 Null byte

既不包含有效数据，位置也无须保留。interconnect 可以插入或移除 null byte，只要不破坏其他字节语义。

这就是为什么 AXI4-Stream 同时有 `TKEEP` 和 `TSTRB`。

---

## 8. `TKEEP` 与 `TSTRB`

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

### 8.1 只使用 `TKEEP`

很多常见 IP 不使用 position byte，等效认为保留的 byte 都是 data byte。此时可以：

```text
TSTRB = TKEEP
```

或者接口完全省略 `TSTRB`，按规范默认关系处理。

### 8.2 `TKEEP='0`

允许一个 transfer 的所有 `TKEEP` 位都是 0。

- 若 `TLAST=0`，这样的全-null transfer 可以被抑制。
- 若 `TLAST=1`，它可能仍承担 packet 结束语义，不能随意丢失 packet boundary。

### 8.3 常见最后一拍

32-bit TDATA，每拍 4 字节，一个 10-byte packet：

```text
beat 0: TKEEP=1111, TLAST=0  -> 4 bytes
beat 1: TKEEP=1111, TLAST=0  -> 4 bytes
beat 2: TKEEP=0011, TLAST=1  -> 2 bytes
```

这里假设低 byte lane 先使用，具体数据排列由系统约定和字节顺序决定。

---

## 9. `TLAST` 与 packet 边界

`TLAST=1` 表示当前已握手 transfer 是 packet 的最后一拍。

注意：

```text
TVALID=1, TLAST=1, TREADY=0
```

此时 packet 还没有结束，因为最后一拍尚未握手。

### 9.1 `TLAST` 的价值

- 通知 Receiver packet 完成。
- 给共享链路提供自然仲裁点。
- 防止两个 packet 被错误合并。

### 9.2 packet 内的 ID/DEST

同一个 packet 中的所有 byte 来自同一 source、去往同一 destination，`TID` 和 `TDEST` 应保持一致。

若 `TID/TDEST` 改变，应视为另一个 stream 的 transfer，而不是同一 packet 随意换身份。

### 9.3 零数据的 TLAST transfer

规范允许 `TLAST=1` 但不带 data byte 或 position byte，用于：

- 发送空 packet。
- 单独补充 packet end。
- 结束一个等待 `TLAST` 的操作。

Monitor 不能因为 `TKEEP='0` 就无条件忽略该 transfer。

---

## 10. `TID` 与 `TDEST`

### 10.1 `TID`

区分共享物理 link 上的多个数据 stream。

例如：

```text
TID=0 -> video channel 0
TID=1 -> video channel 1
```

### 10.2 `TDEST`

提供粗粒度路由信息，例如选择输出端口或目标处理单元。

### 10.3 stream 的唯一性

通常用二元组：

```text
stream_key = {TID, TDEST}
```

区分不同 stream。

### 10.4 Interleaving

不同 stream 的 transfer 可以在同一物理 link 上逐 transfer 交织：

```text
(ID=0, DEST=2, packet A beat0)
(ID=1, DEST=3, packet B beat0)
(ID=0, DEST=2, packet A beat1, TLAST)
(ID=1, DEST=3, packet B beat1, TLAST)
```

因此支持多 stream 的 monitor 不能只有一个全局 current_packet，必须按 `{TID,TDEST}` 保存上下文。

### 10.5 Ordering

同一 stream 的 transfer 必须保持顺序。不同 stream 之间没有统一的先后保证。

---

## 11. `TUSER`

`TUSER` 是用户自定义 sideband，常见用途：

- 视频帧起始标志。
- 错误标志。
- 时间戳。
- 元数据。
- 包分类信息。

### 11.1 两种常见语义

```text
per-byte TUSER：每个 byte 有相应用户位
per-transfer TUSER：整个 beat 共享一组信息
```

宽度转换、packing、null-byte removal 时必须保持 `TUSER` 与对应 byte 的关联。

### 11.2 最大风险

`TUSER` 的含义不由 AXI4-Stream 统一规定。两端如果对 bit 定义、有效时机或宽度转换规则理解不同，链路握手完全正常但功能仍然错误。

项目必须写清：

- 每一位含义。
- 是 per-byte 还是 per-transfer。
- 与 `TKEEP=0` 的关系。
- packet 内是否必须保持。
- 宽度转换时如何扩展/裁剪。

---

## 12. Reset

`ARESETn` 低有效。

Transmitter 在复位期间必须将 `TVALID` 拉低。复位释放后，最早在检测到 `ARESETn=1` 的时钟上升沿之后开始断言 `TVALID`。

其他 payload 在 `TVALID=0` 时可以是任意值，但工程上常清零以便波形调试。

Receiver 的 `TREADY` 复位值取决于实现容量。无论取何值，都不能让复位期间的无效 payload 被当成有效 transfer。

---

## 13. 发送端 RTL 模式

### 13.1 单级输出寄存器

```systemverilog
always_ff @(posedge ACLK or negedge ARESETn) begin
    if (!ARESETn) begin
        // 协议要求 Transmitter 在复位期间把 TVALID 拉低。
        TVALID <= 1'b0;
    end
    else if (!TVALID || TREADY) begin
        // 允许装载的两种情况：
        // 1. TVALID=0，输出寄存器本来就是空的；
        // 2. TREADY=1，旧 beat 本拍被接收，可无气泡地换入新 beat。
        TVALID <= in_valid;
        if (in_valid) begin
            // 只在确认可以装载时更新 payload；backpressure 时不会执行到这里。
            TDATA <= in_data;
            TKEEP <= in_keep;
            TLAST <= in_last;
        end
    end
end
```

关键条件：

```systemverilog
// 常用的单级 AXIS 输出寄存器装载使能。
// 当 TVALID=1 且 TREADY=0 时结果为 0，从而锁住当前 payload。
load_enable = !TVALID || TREADY;
```

当 `TVALID=1 && TREADY=0` 时，寄存器不能更新。

### 13.2 常见错误

```systemverilog
// 错误示例：没有检查当前输出是否被下游接收。
if (in_valid) begin
    TVALID <= 1;
    TDATA  <= in_data; // 下游阻塞时仍覆盖旧 beat
end
```

这样会在 backpressure 下丢数据。

---

## 14. 接收端 RTL 模式

Receiver 只有在有存储空间时拉高 `TREADY`：

```systemverilog
// FIFO 有空间时才能承诺接收；full 时通过 TREADY=0 反压上游。
assign TREADY = !fifo_full;

always_ff @(posedge ACLK) begin
    // 只有真实 handshake 才 push，一次阻塞 beat 不会被重复写入 FIFO。
    if (TVALID && TREADY) begin
        fifo_push(TDATA, TKEEP, TLAST, TID, TDEST, TUSER);
    end
end
```

不能只判断 `TVALID`，否则 FIFO full 时仍然写入。

### 14.1 组合路径与 skid buffer

长链路中若每一级 `TREADY` 都组合向上传播，会形成很长的 ready path。

常见解决：

- register slice。
- skid buffer。
- 小 FIFO。

它们既断开时序路径，又保证 backpressure 到达前的 beat 不丢失。

---

## 15. Width Conversion

### 15.1 Downsizing

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

### 15.2 Upsizing

把多个窄 beat 合并成一个宽 beat：

允许合并的前提包括：

- `TID` 相同。
- `TDEST` 相同。
- 不能跨越已断言的 `TLAST` packet boundary。
- byte 顺序和 sideband 关联保持。

### 15.3 Packing

移除 null byte，让有效 byte 更紧凑。Position byte 不能像 null byte 一样随意删除，因为它的位置有意义。

---

## 16. 一个 AXI4-Stream interface

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

## 17. Packet transaction 建模

### 17.1 Packet 级 item

```systemverilog
class axis_packet extends uvm_sequence_item;
    // packet 级 transaction 用动态 byte 数组表达与总线宽度无关的有效数据。
    rand byte unsigned payload[];
    // TID/TDEST 标识 stream；这里示例假设一个 packet 内保持不变。
    rand bit [3:0]     tid;
    rand bit [3:0]     tdest;
    rand bit [3:0]     tuser_first;

    // valid_gap_max 控制 Transmitter 主动插入的空拍，不是 Receiver backpressure。
    rand int unsigned valid_gap_max;
    rand int unsigned packet_length;

    constraint size_c {
        // 同时约束显式长度字段和动态数组长度，避免两者随机后不一致。
        packet_length inside {[0:2048]};
        payload.size() == packet_length;
        valid_gap_max inside {[0:10]};
    }

    // 注册 factory，便于派生错误 packet 或通过 override 替换类型。
    `uvm_object_utils(axis_packet)
endclass
```

Driver 负责把 `payload[]` 按 `DATA_WIDTH/8` 分割成 beats，并在最后一拍产生正确 `TKEEP/TLAST`。

### 17.2 Beat 级 item

协议 VIP 内部也常使用 beat item：

```systemverilog
class axis_beat extends uvm_sequence_item;
    // beat 级对象与 pin-level 一次 handshake 一一对应。
    bit [31:0] data;
    // keep/strb 按 byte lane 描述 data、position、null byte。
    bit [3:0]  keep;
    bit [3:0]  strb;
    // last 只有在这个 beat 真正 handshake 后才结束 packet。
    bit        last;
    bit [3:0]  id;
    bit [3:0]  dest;
    bit [3:0]  user;
endclass
```

推荐分层：

```text
sequence 使用 packet item 表达测试意图
driver 内部转换成 beat
monitor 先采 beat，再重建 packet
scoreboard 比 packet 内容
```

---

## 18. Driver 发送算法

伪代码：

```text
for each packet:
    将 payload 切成 beats
    for each beat:
        随机插入 TVALID gap
        驱动 payload + TVALID
        while !(TVALID && TREADY):
            保持全部信号
        握手后进入下一 beat
```

### 18.1 最后一拍 keep 计算

```systemverilog
// 每个 beat 能承载的 byte 数，例如 32-bit TDATA 对应 4 bytes。
int bytes_per_beat = DATA_WIDTH / 8;
// remain=0 表示 packet 长度恰好是整拍；否则是最后一拍的有效 byte 数。
int remain = packet.payload.size() % bytes_per_beat;

if (remain == 0)
    // 非空且整拍结束时，最后一拍所有 byte lane 都有效。
    last_keep = '1;
else
    // 假设从低 byte lane 开始连续放置有效数据：remain=2 得到 4'b0011。
    last_keep = (1 << remain) - 1;
```

空 packet 需要项目明确约定：可以发送 `TKEEP='0, TLAST=1` 的 transfer。

### 18.2 不要把 VALID gap 与 backpressure 混淆

- VALID gap：Transmitter 没有提供 beat。
- READY gap：Receiver 对已有 beat 施加 backpressure。

验证环境应独立随机化两者。

---

## 19. Monitor 重建 packet

### 19.1 只在 fire 时采样

```systemverilog
// Monitor 不能只看 TVALID；阻塞期间同一个 beat 会保持多个周期。
if (vif.monitor_cb.TVALID && vif.monitor_cb.TREADY) begin
    // 只在真实 fire 事件中采一次 TDATA 和全部 sideband。
end
```

### 19.2 单 stream

```text
每次 fire：
    根据 TKEEP/TSTRB 追加 byte/position 信息
    保存 TUSER
    若 TLAST=1：发布 packet，清空当前 packet
```

### 19.3 多 stream

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

### 19.4 不能无条件丢弃 null-only beat

如果 `TKEEP='0` 且 `TLAST=1`，它仍可能表示空 packet 或 packet end，monitor 必须保留边界语义。

---

## 20. Scoreboard 类型

### 20.1 恒等通路

输入 packet 应与输出 packet 完全一致：

- data byte 内容。
- byte 顺序。
- packet 边界。
- `TID/TDEST`。
- 约定的 `TUSER`。

### 20.2 Width converter

不能逐 beat 比较，因为输入输出 beat 宽度不同。应先规范化为 byte/packet 表示：

```text
input beats  -> canonical packet bytes + metadata
output beats -> canonical packet bytes + metadata
比较 canonical packet
```

### 20.3 Packet processor

Reference model 根据 packet 级算法预测输出，例如：

- 加/去 header。
- CRC。
- filter。
- resize。
- 路由到不同 `TDEST`。

### 20.4 多 stream 顺序

同一 stream 按序比较，不同 stream 可独立队列：

```text
expected[{id,dest}]
actual[{id,dest}]
```

---

## 21. SVA 协议断言

### 21.1 TVALID sticky

```systemverilog
property p_tvalid_sticky;
    @(posedge ACLK) disable iff (!ARESETn)
    // 当前 beat 未被 Receiver 接收时，下一拍 TVALID 不能撤销。
    TVALID && !TREADY |=> TVALID;
endproperty
a_tvalid_sticky: assert property (p_tvalid_sticky);
```

### 21.2 Payload 稳定

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

### 21.3 复位时 TVALID 为低

```systemverilog
property p_tvalid_low_in_reset;
    @(posedge ACLK)
    // 复位有效时同拍检查 TVALID=0。
    !ARESETn |-> !TVALID;
endproperty
```

### 21.4 非法 keep/strb 组合

```systemverilog
property p_strb_implies_keep;
    @(posedge ACLK) disable iff (!ARESETn)
    // 任一 TSTRB=1 的 lane 必须同时 TKEEP=1；禁止 keep=0/strb=1 组合。
    TVALID |-> ((TSTRB & ~TKEEP) == '0);
endproperty
```

即任何 `TSTRB=1` 的 byte lane 必须同时 `TKEEP=1`。

### 21.5 packet 内 ID/DEST 稳定

这需要 checker 保存 packet active 状态。对于支持多 stream 交织的接口，不能写成“全局从上一个 TLAST 前 TID 必须不变”，因为不同 stream 可以交织。应按 stream context 检查各 packet 的规则。

---

## 22. 功能覆盖计划

### 22.1 Packet 长度

- 0 byte 空 packet。
- 1 byte。
- 恰好 1 beat。
- 比 1 beat 多 1 byte。
- 恰好多个完整 beat。
- 大 packet。

### 22.2 最后一拍 `TKEEP`

- 每种连续有效 byte 数。
- 稀疏 `TKEEP`，若设计支持。
- 全 0 keep + TLAST。
- position byte 组合。

### 22.3 时序

- 无 backpressure，连续满带宽。
- 单拍 backpressure。
- 长 backpressure。
- 在 TLAST beat 上 backpressure。
- Transmitter 插入 VALID gap。
- back-to-back packets，无空拍。

### 22.4 多 stream

- 单一 TID/DEST。
- 多 TID。
- 多 TDEST。
- 不同 stream interleaving。
- 同 stream 多 packet 连续。

### 22.5 Cross

```text
packet_length x last_keep
packet_length x backpressure_length
TLAST_stall x TID
TID x TDEST
width_conversion x null_byte_pattern
```

---

## 23. 性能统计

### 23.1 Link utilization

```text
utilization = transfer_cycles / observed_cycles
transfer_cycles = count(TVALID && TREADY)
```

### 23.2 Source starvation

```text
TVALID=0, TREADY=1
```

表示下游有能力，但上游没有提供数据。

### 23.3 Backpressure

```text
TVALID=1, TREADY=0
```

表示上游有数据，但下游阻塞。

### 23.4 两者都低

```text
TVALID=0, TREADY=0
```

仅凭接口波形不能断定是哪一方造成性能问题，需要结合上下游内部状态。

---

## 24. 常见错误总表

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

## 25. 调试波形固定顺序

1. `ARESETn` 释放后 `TVALID` 是否按规则启动。
2. 只标出 `TVALID && TREADY` 的真实 transfer。
3. backpressure 期间 payload 是否稳定。
4. 根据 `TKEEP/TSTRB` 标出 data、position、null byte。
5. `TLAST` 是否位于真正最后一个已握手 beat。
6. packet 内 `{TID,TDEST}` 是否符合约定。
7. `TUSER` 是否与正确 byte/beat 对齐。
8. 若有 width conversion，packet byte 序列和边界是否保持。

---

## 26. 练习题

### 练习 1

`TVALID=1, TREADY=0` 连续 10 拍，产生几个 transfer？

答案：0 个。

### 练习 2

`TVALID=1, TREADY=0, TLAST=1`，packet 是否已经结束？

答案：没有。最后 beat 尚未握手。

### 练习 3

32-bit TDATA 发送 6-byte packet，低 lane 优先，最后一拍 `TKEEP` 应是什么？

答案：第一拍 `1111`，第二拍 `0011` 且 `TLAST=1`。

### 练习 4

`TKEEP=1, TSTRB=0` 对应什么 byte？

答案：Position byte。位置必须保留，但 `TDATA` 内容不是有效 data byte。

### 练习 5

两个 stream 的 `{TID,TDEST}` 不同，能否逐 beat 交织？

答案：可以，只要每个 stream 自身顺序和 packet 边界保持。

---

## 本章总结

### 知识链

```text
单向 stream
    -> TVALID/TREADY 握手
    -> backpressure 时 payload 稳定
    -> TDATA 按 byte lane 组织
    -> TKEEP/TSTRB 定义 byte 类型
    -> TLAST 定义 packet 边界
    -> TID/TDEST 区分 stream 和路由
    -> TUSER 传递项目自定义元数据
    -> monitor 按 stream 重建 packet
```

### 最重要的 10 条规则

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
