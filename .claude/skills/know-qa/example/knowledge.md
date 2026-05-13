# executor knowledge

## 元信息

- 已学习文档列表：
  - `tcp.txt`
- 全局关键词索引：
  - 三次握手 → `tcp.txt` §3.4 建立连接
  - 序列号 → `tcp.txt` §3.3 序列号
  - 可靠传输 → `tcp.txt` §2.6 可靠通信
  - 流量控制 → `tcp.txt` §2.8 数据通信 & §3.7 窗口管理
  - 状态机 → `tcp.txt` §3.2 术语（状态图 Figure 6）
  - 紧急指针 → `tcp.txt` §3.1 头部格式 & §3.7 紧急信息

---

## `tcp.txt`

### 1. INTRODUCTION（引言）

> **速读总结**：TCP 是为分组交换网络及互联系统提供的高度可靠的端到端主机协议。本节介绍了 TCP 的动机、适用范围、接口概览和基本操作（基本数据传输、可靠性、流控、多路复用、连接、优先级/安全）。
>
> **可回答问题**：TCP 设计目标是什么？它在协议栈中的位置？基本功能模块有哪些？

#### 精读笔记

- **动机**：`[Page 1]`：TCP 是面向连接的、端到端的可靠协议，对下层（IP）假设为不可靠的数据报服务。它提供进程间通信。
- **协议分层**：`[Page 2] Figure 1`：应用层 → TCP → 互联网协议 → 通信网络。
- **可靠性机制**：`[Page 4]`：序列号 + 肯定确认 + 超时重传；校验和检测损坏；重排及去重。
- **流控**：`[Page 4]`：窗口（window）限定发送方未确认数据量。
- **多路复用**：`[Page 5]`：端口（port）+ 网络地址 = 套接字（socket）标识连接。
- **连接建立**：`[Page 5]`：基于时钟的序列号 + 三次握手防止错误初始化。
- **推函数（Push）**：`[Page 4]`：强制 TCP 立即发送数据，不等待填满段。

### 2. PHILOSOPHY（设计哲学）

> **速读总结**：描述互联网系统的组成（主机、网关、进程、端口），TCP 的操作模型（数据分段 → IP 数据报 → 网络包），以及鲁棒性原则（保守发送，宽容接收）。还讨论了可靠通信、连接建立与清除的哲学基础。
>
> **可回答问题**：TCP 如何实现可靠通信？为什么要用三次握手？鲁棒性原则是什么？

#### 精读笔记

- **鲁棒性原则**：`[Page 13]`：“be conservative in what you do, be liberal in what you accept from others”。
- **可靠通信**：`[Page 9-10]`：每个字节分配序列号；使用 SEG.SEQ 和 SEG.ACK；重传队列和定时器；窗口通告进行流控。
- **三次握手**：`[Page 12]`：使用 SYN 标志交换初始序列号（ISN），称为“three-way hand shake”，防止旧的重复连接请求。
- **被动/主动 OPEN**：`[Page 11]`：被动 OPEN 监听连接，主动 OPEN 发起连接；可使用通配符 foreign socket。
- **状态**：`[Page 10]`：TCB（传输控制块）保存连接状态；连接由一对 sockets 唯一标识。

### 3. FUNCTIONAL SPECIFICATION（功能规范）

#### 3.1 Header Format（头部格式）

> **速读总结**：TCP 段头格式，包括源/目的端口、序列号、确认号、数据偏移、控制位（URG/ACK/PSH/RST/SYN/FIN）、窗口、校验和、紧急指针、选项和填充。
>
> **可回答问题**：TCP 段头的各字段含义是什么？如何计算校验和？选项有哪些？

#### 精读笔记

- **控制位**：`[Page 16]`：URG（紧急指针有效）、ACK（确认字段有效）、PSH（推送）、RST（重置连接）、SYN（同步序列号）、FIN（发送方无更多数据）。
- **校验和**：`[Page 16-17]`：TCP 头部 + 数据（奇数字节填充 0）。伪头部包含源/目的 IP、协议、TCP 长度，提供抗误传保护。
- **选项**：`[Page 17-19]`：Kind 0（选项列表结束）、Kind 1（空操作）、Kind 2（MSS，最大段大小）；MSS 仅在 SYN 段中发送。
- **窗口**：`[Page 16]`：16 位，表示从确认号开始的接收方愿意接受的字节数。

#### 3.2 Terminology（术语）

> **速读总结**：定义了 TCB 中的关键变量（发送变量：SND.UNA, SND.NXT, SND.WND, SND.UP 等；接收变量：RCV.NXT, RCV.WND, RCV.UP 等），以及 TCP 连接的状态（LISTEN, SYN-SENT, SYN-RECEIVED, ESTABLISHED, FIN-WAIT-1/2, CLOSE-WAIT, CLOSING, LAST-ACK, TIME-WAIT, CLOSED）。给出了状态图（Figure 6）。
>
> **可回答问题**：TCP 有哪些状态？状态如何转换？发送窗口和接收窗口如何表示？

#### 精读笔记

- **发送序列空间**：`[Page 20] Figure 4`：SND.UNA 到 SND.NXT-1 为已发未确认，SND.NXT 到 SND.UNA+SND.WND-1 为新数据允许发送区。
- **接收序列空间**：`[Page 20] Figure 5`：RCV.NXT 到 RCV.NXT+RCV.WND-1 为允许接收区。
- **状态图**：`[Page 23] Figure 6`：详细展示各状态间的转换事件（主动/被动 OPEN、SYN、ACK、FIN、RST 等）及动作。例如：LISTEN + SYN → SYN-RCVD；SYN-SENT + SYN+ACK → ESTABLISHED。
- **重要常量**：MSL（Maximum Segment Lifetime）= 2 分钟。

#### 3.3 Sequence Numbers（序列号）

> **速读总结**：每个字节一个 32 位序列号，模 2^32 运算。序列号用于确认、重复检测、重排序。SYN 和 FIN 也占用序列号空间。初始序列号（ISN）由时钟生成（约 4μs 增长），周期约 4.55 小时。连接崩溃后需要“安静时间”（MSL）以避免旧段混淆。
>
> **可回答问题**：序列号如何工作？初始序列号为什么基于时钟？如何判断段是否在窗口内？

#### 精读笔记

- **窗口接纳测试**：`[Page 26]`：四个情况（段长度 0 或 >0，接收窗口 0 或 >0）。零窗口时只接受 ACK、RST、URG。
- **SYN/FIN 占位**：`[Page 26]`：SYN 视为在段数据之前，FIN 在数据之后，计入 SEG.LEN。
- **ISN 时钟**：`[Page 27]`：低位置按 4μs 递增，保证 MSL 内序列号唯一。
- **安静时间**：`[Page 28-29]`：崩溃后若丢失序列号记忆，需等待 MSL 才能发送新段，防止老副本干扰。

#### 3.4 Establishing a connection（建立连接）

> **速读总结**：三次握手过程（SYN, SYN+ACK, ACK）。同时打开的情况也能正确处理。通过重置（RST）处理旧的重复 SYN 及半开连接。
>
> **可回答问题**：三次握手的具体步骤？如何处理同时打开？如何检测和恢复半开连接？

#### 精读笔记

- **基本三次握手**：`[Page 31] Figure 7`：
  1. A → B：SYN(100)
  2. B → A：SYN(300), ACK(101)
  3. A → B：ACK(301)
- **同时打开**：`[Page 32] Figure 8`：A 和 B 都发送 SYN，然后都发送 SYN+ACK，最后都发送 ACK。
- **处理旧重复 SYN**：`[Page 33] Figure 9`：B 收到旧 SYN 响应后，A 发现 ACK 不正确则发 RST，使 B 回 LISTEN。
- **半开连接**：`[Page 33-35] Figures 10 & 11`：一端崩溃，另一端仍认为连接存在。当崩溃端重启并发送 SYN 或数据时，另一端返回 RST 或 ACK 触发 RST，从而断开。

#### 3.5 Closing a connection（关闭连接）

> **速读总结**：CLOSE 操作是单工的：主动关闭方发送 FIN，进入 FIN‑WAIT‑1，收到对端 FIN 后进入 TIME‑WAIT 并等待 2MSL 后才真正关闭。被动关闭方收到 FIN 后进入 CLOSE‑WAIT，再发送 FIN 进入 LAST‑ACK。同时关闭也能正确处理。
>
> **可回答问题**：TCP 如何优雅关闭？为什么要 TIME‑WAIT？2MSL 的作用是什么？

#### 精读笔记

- **正常关闭序列**：`[Page 39] Figure 13`：
  - A 主动 CLOSE → FIN(100)
  - B ACK(101) → FIN(300)
  - A ACK(301) → 等待 2MSL → CLOSED
- **TIME‑WAIT 用途**：`[Page 28]`：保证最后一个 ACK 能够被对方收到（若丢失，对方会重传 FIN），同时让网中残留的本连接段全部消失。
- **同时关闭**：`[Page 39] Figure 14`：双方同时发送 FIN，都进入 CLOSING，收到对方 FIN+ACK 后都进入 TIME‑WAIT。

#### 3.7 Data Communication（数据通信）

> **速读总结**：数据以段交换，使用超时重传保证可靠。窗口管理：接收方通告窗口，发送方遵守窗口。紧急数据通过 URG 指针和 URGP 标志传递。还给出重传超时（RTO）的动态计算示例（SRTT 算法）及零窗口探测。
>
> **可回答问题**：TCP 如何计算重传超时？窗口广告如何影响性能？紧急数据如何工作？

#### 精读笔记

- **RTO 计算**：`[Page 41]`：SRTT = α·SRTT + (1-α)·RTT，RTO = min(UB, max(LB, β·SRTT))。α ≈ 0.8~0.9，β ≈ 1.3~2.0，UB ≈ 1 分钟，LB ≈ 1 秒。
- **紧急指针**：`[Page 41-42]`：URG 段中的 urgent pointer 为正偏移，加上段 SEQ 得到“紧急数据最后一个字节的下一个序号”。接收方 RCV.UP 更新，通知用户进入紧急模式，直到 RCV.NXT 追上 RCV.UP。
- **窗口管理建议**：`[Page 42-44]`：避免缩小窗口（shrinking the window）；零窗口时需周期性探测（推荐 2 分钟间隔）；避免分割成过多小段（组合小窗口为大窗口）。

#### 3.8 Interfaces（接口）

> **速读总结**：定义了用户/TCP 接口的命令（OPEN, SEND, RECEIVE, CLOSE, ABORT, STATUS）以及 TCP 对用户的异步消息。还简要说明 TCP 与 IP 接口的参数（TOS, TTL 等）。
>
> **可回答问题**：用户程序如何调用 TCP？OPEN 的主动/被动模式有何区别？SEND/RECEIVE 的 PUSH 和 URGENT 标志的作用？

#### 精读笔记

- **OPEN**：`[Page 45]`：指定 local port, foreign socket, active/passive 标志，可带 precedence/security。返回 local connection name。
- **SEND**：`[Page 46-47]`：带 buffer, byte count, PUSH 和 URGENT 标志。PUSH 迫使立即发送，URGENT 设置紧急指针。
- **RECEIVE**：`[Page 48]`：分配接收缓冲区，返回实际计数、PUSH 和 URGENT 标志。数据根据窗口和 PUSH 决定是否立即返回。
- **CLOSE**：`[Page 49]`：不再发送，但可继续接收直到对端 FIN。
- **ABORT**：`[Page 50]`：立即发送 RST，删除 TCB。
- **IP 接口**：`[Page 51]`：TOS 默认 0（routine, normal delay/throughput/reliability），TTL 设为 1 分钟。

#### 3.9 Event Processing（事件处理）

> **速读总结**：详细描述了每个状态下处理用户命令（OPEN, SEND, RECEIVE, CLOSE, ABORT, STATUS）、到达段、超时（用户超时、重传超时、TIME‑WAIT 超时）的具体步骤。这是 TCP 实现的行为规范。
>
> **可回答问题**：各状态下收到 RST 或 SYN 应如何反应？如何更新发送窗口？如何生成 ACK？

#### 精读笔记

- **CLOSED 状态收到非 RST 段**：`[Page 65]`：发送 RST，SEQ 取决于有无 ACK。
- **LISTEN 状态收到 SYN**：`[Page 65-66]`：检查安全/优先级，发送 SYN+ACK，进入 SYN‑RCVD。
- **SYN‑SENT 状态处理 SYN+ACK**：`[Page 67-68]`：若 ACK 合适，进入 ESTABLISHED，发送 ACK。
- **ESTABLISHED 状态接受数据**：`[Page 72-75]`：处理 ACK（更新 SND.UNA 和窗口），处理 URG，将数据交付用户并推进 RCV.NXT，可能立即发送 ACK。
- **收到 FIN**：`[Page 75]`：根据状态转入 CLOSE‑WAIT, TIME‑WAIT, 或 CLOSING。
- **重传超时**：`[Page 77]`：重传队列中首段，重启定时器。

### 三句话复述

1. 问题：在不可靠的分组交换网络上实现可靠的进程间通信，需要解决数据丢失、乱序、重复、流量控制及连接管理问题。
2. 方案：TCP 通过为每个字节分配序列号实现可靠传输（确认+超时重传），用滑动窗口进行流控，通过三次握手建立连接（含时钟初始序列号），并用状态机管理连接生命周期。
3. 结果：协议定义了段头格式、状态转换表、事件处理规则及用户接口，确保端到端的有序、无损、受控的数据流传输。
