# executor's perception

## 元信息

- 已学习文档列表：
  - [x] `TCP-Selective-Acknowledgment-Options.txt` (RFC 2018)
  - [x] `D-SACK-for-Extension.txt` (RFC 2883)
  - [x] `tcp.txt: 1,4448 line` (RFC 793, 2026-04-25) 全文已读
  - [x] `linux-TCB.md: 190 lines` (Linux 2.6.12 TCB 实现分析, 2026-04-27)
- 知识领域：TCP 协议 / 可靠传输 / 拥塞控制 / Linux 内核实现

---

## `TCP-Selective-Acknowledgment-Options.txt` (RFC 2018)

### 问题背景：累积确认的局限

> **速读总结**（1-3句概括本小节核心内容）
>
> 传统 TCP 使用累积确认（Cumulative Acknowledgment），接收方只能告知发送方"我已经收到了截止到序号 X 的所有数据"。当一个窗口内丢失多个数据包时，发送方每个 RTT 只能知道一个丢包，导致吞吐量急剧下降。
>
> **可回答问题**：为什么传统 TCP 在面对多丢包时性能差？累积确认有什么内在局限？

#### 精读笔记

- **关键概念**：累积确认（Cumulative ACK，`L67-L73`）：接收方通过 TCP 头部的 Acknowledgment Number 字段告知发送方"我已连续收到直到序号 N 的所有数据"。不在接收窗口左沿的乱序到达数据不会触发确认。
- **关键概念**：多丢包灾难（`L63-L73`）：一个窗口内多个丢包会导致 TCP 丢失 ACK 时钟（ACK-based clock），吞吐量严重下降。原因：发送方每个 RTT 只能学到一个丢包的信息。
- **关键概念**：激进发送的困境（`L29-L34`）：发送方可以选择提前重传未确认的数据，但这些数据可能已经被对方成功接收了——造成不必要的重传浪费。
- **证据/数据**：`L84-L89`：RDP 协议的实验表明，在丢包率高、延迟大的网络路径上，禁用选择性确认功能会"大幅增加重传段的数量"（greatly increases the number of retransmitted segments）。Fall & Floyd 1995 的仿真研究证明了 SACK 比非 SACK 的 Tahoe 和 Reno TCP 实现更有优势。

### SACK 机制概述

> **速读总结**：SACK 允许数据接收方在 ACK 中附带一块"选择性确认"信息，告诉发送方哪些非连续的数据块已经成功接收。发送方据此只重传真正丢失的段。
>
> **可回答问题**：SACK 是怎么工作的？它解决了什么问题？

#### 精读笔记

- **关键概念**：SACK 的两个选项（`L106-L110`）：
  1. **SACK-permitted**（Kind=4，2字节）：在 SYN 段中发送，表示本端支持 SACK。建立连接前的"握手"。
  2. **SACK option**（Kind=5，变长）：连接建立后在数据段中发送，承载具体的选择性确认信息。
- **关键概念**：SACK 块（Block）格式（`L193-L206`）：每个块由两个 32 位无符号整数定义——Left Edge（块首序号）和 Right Edge（块尾后一序号）。块表示"这些字节我已收到且连续"，但紧邻块上下界的字节尚未收到。
- **证据/数据**：`L208-L212`：每个 SACK 块占 8 字节 + 2 字节头部，TCP 选项区最多 40 字节，因此最多容纳 4 个块。若同时使用 Timestamp 选项（占用 10 字节+2 填充），则最多 3 个块。
- **关键概念**：SACK 是建议性的（Advisory，`L215-L219`）：接收方虽然报告了某块数据已收到，但后续可以丢弃它（称为 reneging）。因此发送方不能依赖 SACK 来释放缓冲区，必须等待累积 ACK 的确认。

### 接收方行为：生成 SACK 选项

> **速读总结**：接收方在收到乱序数据时应在 ACK 中携带 SACK 选项。第一个 SACK 块必须包含触发该 ACK 的段所在的连续块，后续块尽量填满可用的选项空间，优先重复最近报告过的块。
>
> **可回答问题**：接收方什么时候发 SACK？第一个块应该是什么？如何选择后续块？

#### 精读笔记

- **关键概念**：触发条件（`L241-L244`）：仅当接收队列中存在非连续数据时，才应发送 SACK 选项。即收到的段不能推进累积 ACK 的序号。
- **关键概念**：第一个块必须是最新的（`L254-L260`）：第一个 SACK 块必须包含触发这个 ACK 的段所在的连续数据块（除非该段推进了累积确认号）。这确保 SACK 反映接收缓冲区的最新变化。
- **关键概念**：块重复策略（`L267-L276`）：后续 SACK 块应该重复之前报告过的块（非当前第一个块的子集），确保任何非连续块至少在三个连续的 SACK 选项中被报告——抵抗 ACK 丢失。
- **质疑/批评**：`L287-L290`：规定"第一个块必须报告最新收到的段"很重要，但极端情况下如果大量 ACK 丢失，仍然可能导致发送方误判。

### 发送方行为：解释 SACK 并决定重传

> **速读总结**：发送方为每个待重传队列中的段维护一个 SACKed 标记位。收到 SACK 块后，将完全落在块内的段的 SACKed 位置位，后续跳过这些段不重传。超时后清空所有 SACKed 位。
>
> **可回答问题**：发送方收到 SACK 后怎么处理？重传策略是什么？超时后如何处理历史 SACK 信息？

#### 精读笔记

- **关键概念**：SACKed 标志位（`L306-L308`）：发送方为重传队列中每个段维护一个标志位，标记该段是否已被选择性确认。
- **关键概念**：重传跳过（`L319-L322`）：任何 SACKed 位为 1 的段在后续重传中被跳过。SACKed 位为 0 且序号小于已确认的最高 SACK 段的段可以被重传。
- **关键概念**：RTO 后的处理（`L324-L329`）：重传超时后必须清空所有 SACKed 位（因为接收方可能已经 renege）。发送方必须重传窗口左沿的段。段不能被释放，直到累积 ACK 推进窗口。
- **关键概念**：拥塞控制保留（`L345-L357`）：SACK 不改变拥塞控制的基本规则——快恢复期间每个 ACK 最多发一个段（Reno 风格）或两个段（慢启动）。超时仍然是兜底机制。

### 三句话复述

1. 问题：累积确认方案下，多丢包时发送方每个 RTT 只能发现一个丢包，吞吐量严重下降。
2. 方案：SACK 让接收方在 ACK 中携带多个"已收到数据块"的区间信息，发送方据此只重传真正丢失的段。
3. 结果：大幅减少不必要的重传，提高多丢包场景下的 TCP 吞吐量，同时保留拥塞控制的核心机制。

---

## `D-SACK-for-Extension.txt` (RFC 2883)

### 问题背景：SACK 未处理重复段

> **速读总结**：RFC 2018 的 SACK 只定义了如何报告"乱序到达的新数据"，但没有定义如何报告**重复接收到的数据段**。D-SACK（Duplicate-SACK）填补了这个空白。
>
> **可回答问题**：SACK 有什么没覆盖到的场景？D-SACK 解决了什么问题？

#### 精读笔记

- **关键概念**：D-SACK 的动机（`L43-L47`）：当接收方收到重复包时，通过 D-SACK 报告给发送方，发送方就能推断出**何时不必要地重传了一个包**，从而在网络重排序、ACK 丢失、包复制、过早超时等场景下做出更鲁棒的操作。
- **关键概念**：兼容性（`L81-L92`）：D-SACK 不需要额外的协商——只要两端协商了 SACK 能力，D-SACK 即可工作。不支持 D-SACK 的发送方会直接丢弃 D-SACK 块，正常处理其他 SACK 块。

### D-SACK 的具体规则和格式

> **速读总结**：D-SACK 复用 SACK 选项格式，将第一个 SACK 块用于报告重复段。若重复段属于一个更大的非连续数据块，第二个 SACK 块报告这个更大的块。后续块按 RFC 2018 处理。
>
> **可回答问题**：D-SACK 块怎么放？和普通 SACK 块怎么共存？

#### 精读笔记

- **关键概念**：D-SACK 的五条规则（`L187-L208`）：
  1. D-SACK 块仅用于报告最近收到的重复连续数据序列。
  2. 每个重复序列最多被一个 D-SACK 块报告。
  3. D-SACK 块的左/右边界定义与普通 SACK 块相同。
  4. 如果重复段属于累积 ACK 之上的一个更大块，第二个 SACK 块应报告这个更大的块。
  5. 后续 SACK 块按 RFC 2018 报告其他非连续块。
- **关键概念**：信息丢失风险（`L210-L212`）：每个重复段仅在一个 ACK 包中报告，如果该 ACK 丢失，关于该重复段的信息就永久丢失了。
- **关联**：D-SACK 的"第一条规则"在 SACK 选项空间中的优先级：第一块必须是 D-SACK（重复段），普通 SACK 块从第二块开始。

### 发送方如何识别 D-SACK

> **速读总结**：发送方通过比较第一个 SACK 块的序号空间与**同包中的累积 ACK 值**来识别 D-SACK。如果 SACK 块空间小于累积 ACK，说明是重复数据。不能与 snd.una 状态变量比较。
>
> **可回答问题**：发送方怎么知道第一个 SACK 块是 D-SACK？判断标准是什么？

#### 精读笔记

- **关键概念**：判断方法（`L464-L472`）：比较第一个 SACK 块的序号空间与**同一个 ACK 包中**的累积 ACK 字段。如果 SACK 空间 < 累积 ACK，说明该段被重复接收。不能与 snd.una 比较，因为 ACK 可能被重排序。
- **关键概念**：第二个判断（`L474-L479`）：如果第一个 SACK 块 > 累积 ACK，则与第二个 SACK 块比较——若第一个块是第二个块的子集，则说明是累积 ACK 之上的重复数据。

### D-SACK 的四种典型应用场景

> **速读总结**：D-SACK 可以检测四种场景：网络复制包、包重排序导致的虚假重传、ACK 丢失导致的超时、过早的 RTO 超时。
>
> **可回答问题**：D-SACK 在哪些实际场景中发挥作用？

#### 精读笔记

**场景1：网络复制包（`L511-L522`）**

- D-SACK 让发送方明确知道某个包被网络复制了。没有 D-SACK 时发送方只能猜测。

**场景2：包重排序导致的虚假重传（`L551-L581`）**

- 当包延迟到达（乱序超过 3 个包），Fast Retransmit 会误触发，发送方不必要地重传。D-SACK 让发送方知道这是一次"不必要的重传"。
- **关键概念**：重排序与丢包的区分（`L596-L597`）：D-SACK 使发送方更可靠地推断"第一次传输没有被丢弃"，从而可以调整 dupACK 阈值或撤销拥塞窗口缩减。

**场景3：ACK 丢失导致的超时（`L635-L648`）**

- 整个窗口的 ACK 都丢失了，发送方超时重传——但实际上数据已经全部到达。D-SACK 能让发送方判断"没有数据包被丢弃"。
- **潜在应用**：可以为 ACK 路径和 data 路径分别实施拥塞控制。

**场景4：过早的 RTO 超时（`L679-L700`）**

- RTO 设置过短，在数据包还在路上时就超时了。D-SACK 让发送方识别这是"过早超时"，从而调整 RTO 参数或撤销不必要的窗口缩减。

### 三句话复述

1. 问题：RFC 2018 的 SACK 没有定义如何报告重复段，导致发送方无法区分"真的丢包"和"虚假重传"。
2. 方案：D-SACK 复用 SACK 格式，用第一个 SACK 块报告重复段，并通过与累积 ACK 的比较来识别。
3. 结果：使 TCP 发送方能检测网络复制、包重排序、ACK 丢失和过早超时四种非丢包事件，从而做出更智能的重传和拥塞控制决策。

---

## `tcp.txt` (RFC 793, 2026-04-25)

### TCP 设计哲学——"宪法"级的定义

> **速读总结**：RFC 793 定义了 TCP 协议的完整规范——它是连接导向的、端到端的可靠传输协议。核心承诺：在不可靠的 IP 网络之上提供可靠的、有序的字节流传输。所有后来的扩展（包括 SACK、拥塞控制等）都在此框架内运作。
>
> **可回答问题**：TCP 最底层的设计原则是什么？TCP 最基本的确认和重传机制如何运作？

#### 精读笔记

- **关键概念**：可靠性三件套（`L435-L445`）：
  1. 每个字节一个序号（sequence number）
  2. 正面确认（positive ACK），接收方收到数据后发回确认
  3. 超时重传（timeout retransmission），ACK 未在超时间隔内到达则重传
- **关键概念**：TCP 四层功能（`L403-L510`）：
  - 基本数据传输（Basic Data Transfer）：分段传输连续字节流
  - 可靠性（Reliability）：序号+ACK+超时重传+校验和
  - 流量控制（Flow Control）：滑动窗口机制，接收方通告剩余缓冲区
  - 多路复用（Multiplexing）：端口/套接字（socket）机制
- **关键概念**：Robustness Principle（`L972-L974`）——"be conservative in what you do, be liberal in what you accept from others." 这是 TCP 设计的总则。
- **关键概念**：套接字（Socket）与连接（`L478-L510`）：socket = IP 地址 + 端口号。一个连接由一对 socket 唯一标识。全双工。
- **关键概念**：Transmission Control Block, TCB（`L1320-L1327`）：每个连接对应的控制块，存储所有状态变量。

### TCP 头部格式

> **速读总结**：TCP 头部固定 20 字节 + 可选项。关键字段：源/目端口（各 16 位）、序列号（32 位）、确认号（32 位）、标志位（URG/ACK/PSH/RST/SYN/FIN）、窗口（16 位）、校验和（16 位）、紧急指针、选项。
>
> **可回答问题**：TCP 头部长什么样？各字段有什么用？

#### 精读笔记

- **关键概念**：序列号（`L1125-L1129`）：32 位，表示段中第一个数据字节的序号。SYN 段中表示初始序列号（ISN），第一个数据字节是 ISN+1。
- **关键概念**：确认号（`L1131-L1135`）：32 位，ACK 标志置位时有效。表示发送方**期望收到的下一个序列号**。本质是累积的——确认号 N 表示"所有序号 < N 的字节都已收到"。
- **关键概念**：标志位（`L1147-L1154`）：URG（紧急指针有效）、ACK（确认有效）、PSH（推送）、RST（重置连接）、SYN（同步序号，建连用）、FIN（结束发送，断连用）。
- **关键概念**：窗口（`L1156-L1160`）：16 位，表示从确认号开始，发送方可发送的字节数。接收方用此进行流量控制。
- **关键概念**：选项（Options, `L1212-L1308`）：可变长。RFC 793 定义了两个基本选项：End of Option List（Kind=0，1字节）和 No-Operation（Kind=1，1字节），以及 Maximum Segment Size（MSS，Kind=2，4字节）。**所有这些选项格式被后来的 SACK（Kind=4,5）等扩展沿用。**

### 序列号空间与状态变量

> **速读总结**：TCP 用三个关键发送变量（SND.UNA / SND.NXT / SND.WND）和两个关键接收变量（RCV.NXT / RCV.WND）管理数据流。发送序列空间分为已确认、未确认、可发送、未来四段。
>
> **可回答问题**：TCP 如何跟踪哪些数据已发、哪些已确认？窗口机制如何工作？

#### 精读笔记

- **关键概念**：发送序列变量（`L1329-L1338`）：
  - `SND.UNA` (Send Unacknowledged)：最老的未确认序列号
  - `SND.NXT` (Send Next)：下一个要发送的序列号
  - `SND.WND` (Send Window)：发送窗口大小（来自接收方通告）
  - `ISS` (Initial Send Sequence Number)：初始发送序列号
- **关键概念**：接收序列变量（`L1340-L1345`）：
  - `RCV.NXT` (Receive Next)：下一个期望接收的序列号
  - `RCV.WND` (Receive Window)：接收窗口（可用缓冲区）
  - `IRS` (Initial Receive Sequence Number)：初始接收序列号
- **关键概念**：发送序列空间四段（`L1364-L1378`）：
  1. 已确认的旧序列号（< SND.UNA）
  2. 已发送但未确认的数据（SND.UNA ~ SND.NXT）
  3. 允许发送的新数据（SND.NXT ~ SND.UNA+SND.WND）
  4. 尚未允许的未来序列号
- **关键概念**："Acceptable ACK"判断（`L1673-L1676`）：`SND.UNA < SEG.ACK <= SND.NXT`。确认号必须在这个范围内才算合法。
- **关键概念**：ISN 选择（`L1791-L1796`）：基于 32 位时钟，低 4 位每约 4 微秒递增一次，循环周期约 4.55 小时（> MSL，确保唯一性）。
- **关键概念**：Modulo 2^32 比较（`L1610-L1618`）：序列号在有限空间中循环，所有比较必须模 2^32。这是所有序列号运算的基础。

### TCP 状态机与连接管理（三次握手、四次挥手）

> **速读总结**：TCP 连接状态机包括 11 个状态（CLOSED/ LISTEN/ SYN-SENT/ SYN-RECEIVED/ ESTABLISHED/ FIN-WAIT-1/ FIN-WAIT-2/ CLOSE-WAIT/ CLOSING/ LAST-ACK/ TIME-WAIT）。三次握手（SYN → SYN-ACK → ACK）建连，四次挥手（FIN → ACK → FIN → ACK）断连。
>
> **可回答问题**：TCP 怎么建连和断连？状态如何转移？

#### 精读笔记

- **关键概念**：三次握手（`L2033-L2041`，Figure 7）：
  1. A → B: `SYN, SEQ=100`   （A 告诉 B：我的 ISN 是 100）
  2. B → A: `SYN+ACK, SEQ=300, ACK=101` （B 告诉 A：收到你的 100，我的 ISN 是 300）
  3. A → B: `ACK, SEQ=101, ACK=301`     （A 确认收到 B 的 300）
- **关键概念**：三次握手的根本目的——防旧重复（`L2095-L2098`）："The principle reason for the three-way handshake is to prevent old duplicate connection initiations from causing confusion." 通过序列号的验证来区分"这是真的新连接"还是"网络里延迟的旧 SYN"。
- **关键概念**：四次挥手（`L2484-L2499`，Figure 13）：A 发 FIN → B 回 ACK → B 发 FIN → A 回 ACK → A 等待 2 MSL → CLOSED。
- **关键概念**：TIME-WAIT（2 MSL，`L1479-L1481`）：确保最后一个 ACK 到达对方，并让网络中残留的旧数据包过期消失。
- **关键概念**：RST 机制（`L2308-L2392`）：重置连接的三类情况——连接不存在时（CLOSED）、非同步状态、同步状态。RST 用于处理半开连接、拒绝非法连接等异常。

### 可靠通信与重传超时

> **速读总结**：TCP 可靠性的核心是"发送→复制到重传队列→等 ACK→收到则删除→超时则重传"。RTO 基于平滑 RTT（SRTT）动态计算：SRTT = α*SRTT + (1-α)*RTT，RTO = β*SRTT（有上下界）。
>
> **可回答问题**：TCP 如何保证可靠传输？重传超时怎么计算？

#### 精读笔记

- **关键概念**：重传机制（`L769-L781`）："When the TCP transmits a segment containing data, it puts a copy on a retransmission queue and starts a timer; when the acknowledgment for that data is received, the segment is deleted from the queue. If the acknowledgment is not received before the timer runs out, the segment is retransmitted."
- **关键概念**：RTT 与 RTO（`L2617-L2634`）：
  - RTT = 从发送到收到确认的往返时间
  - SRTT = α *SRTT + (1-α)* RTT  （α ≈ 0.8~0.9，平滑因子）
  - RTO = min[UBOUND, max[LBOUND, β * SRTT]]  （β ≈ 1.3~2.0，方差因子）
  - UBOUND ≈ 1 分钟，LBOUND ≈ 1 秒
- **质疑/批评**：RFC 793 的 RTO 计算只有一阶平滑，没有考虑 RTT 方差。后来 Jacobson 1988 的拥塞控制论文引入了 RTT 方差项（RTTVAR），大幅提升了 RTO 的准确性。
- **关键概念**：零窗口探测（`L2694-L2700`）：发送方必须定期重传（即使窗口为零），防止窗口更新丢失后死锁。

### 事件处理——TCP 的"运行时"

> **速读总结**：3.9 节定义了 TCP 对所有事件的响应——用户调用（OPEN/SEND/RECEIVE/CLOSE/ABORT）、到达段（SEGMENT ARRIVES）、超时（USER/RETRANSMISSION/TIME-WAIT）。每个事件的处理都因当前状态而异，形成了完整的状态机实现规范。
>
> **可回答问题**：TCP 收到一个段后具体做了什么？什么时候更新 SND.UNA？什么时候发送 ACK？RTO 超时后实际发生了什么？

#### 精读笔记

**用户调用——各状态的响应**

- **OPEN Call**（`L3331-L3387`）：
  - CLOSED 状态：创建 TCB，填充本地 socket/foreign socket/优先/安全参数，选择 ISS。被动 OPEN → LISTEN；主动 OPEN → 发送 SYN，置 SND.UNA=ISS, SND.NXT=ISS+1，进入 SYN-SENT。
  - LISTEN 状态：若指定了 foreign socket，转为主动，发送 SYN，进入 SYN-SENT。
  - 其他状态：返回"connection already exists"。
  
- **SEND Call**（`L3397-L3453`）：
  - LISTEN 状态：可触发自动连接（发送 SYN）。
  - ESTABLISHED/CLOSE-WAIT：分段化缓冲区，捎带确认（ACK=RCV.NXT）发送。URG 标志置 SND.UP。
  - FIN-WAIT 系列状态：返回"connection closing"。

- **RECEIVE Call**（`L3466-L3527`）：
  - ESTABLISHED/FIN-WAIT-1/FIN-WAIT-2：重组队列中的段到接收缓冲区，返回给用户。若数据不足则排队请求。
  - CLOSE-WAIT：仅返回已到手的剩余文本。

- **CLOSE Call**（`L3539-L3597`）：
  - LISTEN：删除 TCB，进入 CLOSED。
  - SYN-SENT：删除 TCB。
  - SYN-RECEIVED：若无待发数据，发送 FIN，进入 FIN-WAIT-1。
  - ESTABLISHED：排队等待所有 SEND 完成后，发送 FIN，进入 FIN-WAIT-1。
  - CLOSE-WAIT：发送 FIN，进入 CLOSING。

- **ABORT Call**（`L3610-L3650`）：
  - SYN-RECEIVED/ESTABLISHED/FIN-WAIT-1/-2/CLOSE-WAIT：发送 RST（`<SEQ=SND.NXT><CTL=RST>`），清除所有队列，删除 TCB，进入 CLOSED。

**SEGMENT ARRIVES——核心的八步处理**

这是 RFC 793 最核心的部分——段到达后按八步顺序处理（`L3726-L4380`）：

1. **第一步：检查序列号**（`L3962-L4027`）——四段可接受性测试
   - 四类情况：段长度 0/窗口 0、段长度 0/窗口 >0、段长度 >0/窗口 0、段长度 >0/窗口 >0。不合法则发送 ACK 并丢弃段。
   - 关键：**只有序列号在 RCV.NXT 到 RCV.NXT+RCV.WND-1 之间的段才是合法的**。

2. **第二步：检查 RST 位**（`L4029-L4061`）
   - SYN-RECEIVED：被动 OPEN → 回 LISTEN；主动 OPEN → 通知"connection refused"，删除 TCB。
   - ESTABLISHED/FIN-WAIT-1/-2/CLOSE-WAIT：通知用户"connection reset"，清空队列，删除 TCB。
   - CLOSING/LAST-ACK/TIME-WAIT：直接删除 TCB。

3. **第三步：检查安全和优先级**（`L4080-L4101`）
   - 若安全和优先级不匹配，发送 RST，删除连接。

4. **第四步：检查 SYN 位**（`L4103-L4122`）
   - 若 SYN 在窗口内 → 错误，发送 RST，删除连接。因已建立连接不应再收到 SYN。

5. **第五步：检查 ACK 字段——最关键的确认处理**（`L4139-L4230`）
   - **SYN-RECEIVED**：若 SND.UNA <= SEG.ACK <= SND.NXT → 进入 ESTABLISHED。
   - **ESTABLISHED**（`L4157-L4178`）：若 SND.UNA < SEG.ACK <= SND.NXT → **推进 SND.UNA**，从重传队列删除完全确认的段。窗口更新需要验证 (SND.WL1, SND.WL2) 防止旧段更新窗口。重复 ACK（SEG.ACK < SND.UNA）可忽略。
   - **FIN-WAIT-1**：若 FIN 被确认 → 进入 FIN-WAIT-2。
   - **LAST-ACK**：若 FIN 被确认 → 删除 TCB，进入 CLOSED。
   - **TIME-WAIT**：只接受对方重传的 FIN，回复 ACK，重启 2MSL 定时器。

6. **第六步：检查 URG 位**（`L4232-L4263`）：推进 RCV.UP，通知用户有紧急数据。

7. **第七步：处理段文本**（`L4265-L4322`）
   - ESTABLISHED/FIN-WAIT-1/-2：将数据移动到用户接收缓冲区。推进 RCV.NXT，调整 RCV.WND。发送 ACK（`<SEQ=SND.NXT><ACK=RCV.NXT><CTL=ACK>`），尽量捎带。

8. **第八步：检查 FIN 位**（`L4324-L4379`）
   - ESTABLISHED → CLOSE-WAIT；FIN-WAIT-1 → 若自身 FIN 已确认则 TIME-WAIT 否则 CLOSING；FIN-WAIT-2 → TIME-WAIT。

**超时处理（`L4434-L4451`）**

- **USER TIMEOUT**：清空所有队列，通知用户"connection aborted due to user timeout"，删除 TCB，进入 CLOSED。
- **RETRANSMISSION TIMEOUT**：**重传重传队列前端的段**，重新初始化重传定时器。（注意：这里很简略——只说了"重新发送队列前端的段"，后来的拥塞控制算法比这复杂得多）
- **TIME-WAIT TIMEOUT**：2MSL 到期 → 删除 TCB，进入 CLOSED。

### 三句话复述

1. 问题：如何在不可靠的 IP 分组交换网络之上实现可靠的端到端字节流通信？
2. 方案：面向连接的设计——每个字节有唯一序号，接收方返回累积确认，发送方超时重传未确认数据；配合滑动窗口流量控制和端口复用机制。
3. 结果：定义了一个运行了 40 多年的互联网核心协议框架——所有后续扩展（窗口缩放、SACK、拥塞控制等）都是在这个"宪法"之上增修的"修正案"。

---

## `linux-TCB.md` (Linux 2.6.12 TCB 实现分析, 2026-04-27)

### TCB 的五个缓冲区

> **速读总结**：Linux 2.6.12 的 TCB（`struct tcp_sock`）管理 5 个缓冲区队列，分别处理发送、接收、乱序、错误和预读。发送队列 `sk_write_queue` 同时承担重传队列和待确认队列的角色。
>
> **可回答问题**：Linux 内核如何实现 RFC 793 的 TCB 概念？数据包在内核中有哪些排队路径？

#### 精读笔记

- **关键概念**：五个缓冲区队列（`L1-L64`）：

| 队列 | 位置 | 存储内容 |
|:---|:---|:---|
| `sk_write_queue` | `include/net/sock.h:201` | 已从上层接收但未被 ACK 的数据（同时充当重传队列） |
| `sk_receive_queue` | `include/net/sock.h:202` | 已 ACK 到达但未 push 给上层的数据 |
| `out_of_order_queue` | `include/linux/tcp.h:336` | 在 RCV.NXT 之后但中间有缺口的乱序段 |
| `sk_error_queue` | `include/net/sock.h:220` | ICMP 错误信息，等待上层通过 getsockopt 读取 |
| `prequeue` (ucopy) | `include/linux/tcp.h:271-277` | VJ 风格的预读队列，上层 recv 时绕过 receive_queue 直接拷贝 |

- **关键概念**：发送队列的三重身份（`L65-L72`）：`sk_write_queue` 既是发送队列（存上层提交的数据）又是重传队列（超时后取出重发）又是待确认队列（收到 ACK 后移除）。不需要单独的重传队列——超时时扫描 `sk_write_queue` 找到超时未确认的段即可重传。

- **关键概念**：ICMP 错误如何找到 socket（`L73-L78`）：RFC 792 规定 ICMP 差错报文必须携带原始 IP 头的负载前 8 字节。TCP 头的前 8 字节正好包含源端口+目标端口，与 ICMP 报文中的源/目 IP 形成完整的四元组。内核据此在 socket 哈希表中查找对应 socket，将错误码存入 `sk_error_queue`。

### struct tcp_sock 结构全景

> **速读总结**：`struct tcp_sock`（`include/linux/tcp.h:233`）是 Linux TCB 的完整实现，按功能分为连接状态、窗口管理、MSS/分段、RTT 测量、拥塞控制、定时器、SACK、TCP 选项、接收队列管理、紧急数据、连接管理、保活、拥塞算法状态、时间戳、ECN、接收方估计 16 个模块。
>
> **可回答问题**：Linux TCB 内部包含哪些数据？RFC 793 的抽象状态变量在代码中长什么样？

#### 精读笔记

- **关键概念**：序列变量（`L48-L58`）——RFC 793 到 C 代码的直接映射：

| RFC 793 变量 | Linux 字段 (`tcp_sock`) | 含义 |
|:---|:---|:---|
| RCV.NXT | `rcv_nxt` | 期望接收的下一个序列号 |
| SND.NXT | `snd_nxt` | 下一个要发送的序列号 |
| SND.UNA | `snd_una` | 最老的等待确认字节 |
| SND.WL1/SND.WL2 | `snd_wl1` | 窗口更新保护序列号 |
| SND.WND | `snd_wnd` | 接收方通告的窗口 |
| RCV.WND | `rcv_wnd` | 本地接收窗口 |

- **关键概念**：拥塞控制字段（`L112-L117`）——RFC 793 没有定义拥塞控制，这是后来的增强：
  - `snd_cwnd`：发送拥塞窗口
  - `snd_ssthresh`：慢启动阈值
  - `packets_out`：飞行中的数据包数（已发未确认）
  - `retrans_out`：重传中的数据包数
  - `lost_out`：判断为丢失的数据包数
  - `sacked_out`：被 SACK 确认的数据包数

- **关键概念**：SACK 字段（`L125-L127`）：`duplicate_sack[1]` 存 D-SACK 块，`selective_acks[4]` 存最多 4 个 SACK 块——直接对应 RFC 2018 的块容量限制（40 字节选项区）。

- **关键概念**：数据结构层次（`L176-L180`）：
  ```
  struct sock           → 通用套接字（所有协议族共享）
    └── struct inet_sock → IPv4 特定（IP 地址、端口等）
        └── struct tcp_sock → TCP TCB（序列号、窗口、拥塞控制等）
  ```
  这是 Linux 内核中经典的分层继承模式：通用代码操作 `sock`，TCP 特定代码操作 `tcp_sock`。

- **关键概念**：TIME_WAIT 的内存优化（`L182-L190`）：TIME_WAIT 状态不需要完整的 `tcp_sock`（因为不再收发数据），Linux 使用轻量级 `struct tcp_tw_bucket`，只保存序列号、窗口、超时时间和时间戳。繁忙服务器上大量 TIME_WAIT 连接可因此大幅节省内存。

- **关键概念**：RTT 测量的实现细节（`L106-L110`）：`srtt` 以 <<3 格式存储（即乘以 8 的定点数），`mdev` 是中值偏差（mean deviation），`rttvar` 是平滑偏差。`rto` 由这些值计算得出。

- **关联**：拥塞控制算法模块化（`L157-L162`）——`tcp_sock` 中嵌入了 `vegas`、`westwood`、`bictcp` 等算法特定的结构体，Linux 在 2.6.12 就已支持可插拔的拥塞控制算法。

### 三句话复述

1. 问题：RFC 793 的 TCB 只是一个抽象概念——Linux 内核如何具体实现它？
2. 方案：`struct tcp_sock` 嵌入在 `sock→inet_sock` 继承链中，内含 5 个缓冲区队列 + 序列变量 + 窗口/拥塞/RTT/SACK/定时器等全部状态，`sk_write_queue` 同时充当发送队列和重传队列。
3. 结果：完整的生产级 TCB 实现，TIME_WAIT 状态通过轻量 `tcp_tw_bucket` 优化内存，拥塞控制算法可插拔。
