# TCB

## 五个缓冲区

根据对Linux 2.6.12内核源码的分析，TCB（传输控制块）进程对应需要5个缓冲区。以下是详细分析和源码来源：

  缓冲区列表

  1. 发送缓冲区 (sk_write_queue)
  2. 接收缓冲区 (sk_receive_queue)
  3. 乱序队列 (out_of_order_queue)
  4. 错误队列 (sk_error_queue)
  5. 预读队列 (prequeue，在ucopy结构中)

  源码来源

  1. 发送和接收缓冲区

  文件: include/net/sock.h:201-202
  struct sk_buff_head   sk_receive_queue;  // 接收队列
  > 已经按需到达(已经被 ACK)的 data, 但是未 Push 给上层的 data(未被上层 recv 给接收)
  struct sk_buff_head   sk_write_queue;     // 发送队列
  > 从上层 RCV 的 data (上层 send 的 buffer), 但是没有被 ACK 的 data

  1. 错误队列

  文件: include/net/sock.h:220
  struct sk_buff_head   sk_error_queue;     // 错误队列

  1. 乱序队列

  文件: include/linux/tcp.h:336
  struct sk_buff_head   out_of_order_queue; /*Out of order segments go here*/
  > 未被 ACK 的数据,  存放在 RCV.NXT 之后(未出现 RCV.NXT) 的报文段, 中间有缺口, 如果出现了本段 RVC.NXT 的报文段, 那么将把这一连续的报文段给放入接收队列

  1. 预读队列

  文件: include/linux/tcp.h:271-277
  struct {
      struct sk_buff_head       prequeue;        // 预读队列
      struct task_struct        *task;
      struct iovec*iov;
      int                       memory;
      int                       len;
  } ucopy;
  > 当上层 RCV 时, 绕过接收队列进行接收

  TCP序列变量对应关系

  从include/linux/tcp.h:249-250可以看到TCP序列变量的定义：
  __u32 rcv_nxt;        /*接下来我们想要接收的内容是：RCV.NXT */
  __u32 snd_nxt;        /* 接下来我们要发送的序列是：SND.NXT */
  __u32 snd_una;        /* 我们首先需要第一个字节的确认信息 - SND.UNA */
  __u32 snd_wl1;        /* 窗口更新序列 - SND.WL1 */
  __u32 snd_wnd;        /* 我们预期会收到的窗口信息 - SND.WND */
  __u32 rcv_wnd;        /* 当前接收窗口 - RCV.WND */
  __u32 urg_seq;        /* 收到的紧急指针序列 - RCV.UP/SND.UP */

  这些序列变量管理缓冲区的状态，但不直接对应缓冲区本身。缓冲区用于存储实际的数据包，而序列变量用于跟踪缓冲区的使用情况。

  总结

  TCB进程需要5个独立的缓冲区来处理TCP连接的发送、接收、乱序数据、错误和预读功能。这些缓冲区的设计遵循TCP协议的可靠传输要求，确保数据的有序传输和错误处理。

### 重传队列是否基于发送队列？发送队列是否充当“重传队列”和“确认队列”？

是的，发送缓冲区（sk_write_queue）同时承担了 重传队列 和 待确认队列 的角色。
当 TCP 发送一个数据段时，该段的数据会保留在 sk_write_queue 中（不会被立即删除），并启动重传定时器。此时这个段属于“已发送但未确认”。
如果收到 ACK，确认的序列号范围内的数据会被从 sk_write_queue 中移除（释放 skb）。
如果定时器超时，TCP 会从 sk_write_queue 中取出相应未确认的段重新发送。
因此，不需要单独的重传队列。sk_write_queue 是一个有序队列，按序列号排序（实际上按发送顺序），包含了所有已提交但未被完全确认的数据（包括已发送未确认和待发送的数据）。重传时只需扫描队列找到超时的段。

### ICMP错误是如何找到“主人”的？

根据 RFC 792 的规定，当一个路由器或主机无法正常处理 IP 数据报时，它会生成一个 ICMP 差错报文。这个报文的 “数据”部分必须包含导致该错误的原始数据报的 IP 首部及其负载的前 8 个字节。
对于 TCP 而言，IP 数据报的首部之后紧接着的就是 TCP 首部，而 TCP 首部的前 8 个字节恰好包含了源端口号和目标端口号。
因此，ICMP 差错报文中自然而然地携带了 TCP 连接的完整“四元组”（源IP、目标IP、源端口、目标端口）。内核的网络协议栈正是利用这些信息，才能在 socket 哈希表中精确地找到属于该连接的 socket，然后把 ICMP 错误类型（例如，ECONNREFUSED）转换为对应的错误码，并存入这个 socket 的 sk_error_queue 中。应用程序下一次从这个 socket 进行读取或写入操作时，内核就会返回这个错误码。

## Linux 2.6.12 TCB 结构分析

  TCB 的核心数据结构

  Linux 2.6.12 中 TCP TCB 主要通过 struct tcp_sock 实现，定义在 include/linux/tcp.h:233。

  TCB 中的主要元数据

1. 连接状态信息

- rcv_nxt - 期望接收的下一个序列号
- snd_nxt - 下一个要发送的序列号
- snd_una - 第一个等待确认的字节
- ca_state - 拥塞控制状态（Open/Disorder/CWR/Recovery/Loss）

  1. 窗口管理
- snd_wnd - 接收方通告的窗口大小
- rcv_wnd - 本地接收窗口
- max_window - 从对端看到的最大窗口
- window_clamp - 窗口上限

  1. MSS 和分段
- mss_cache - 缓存的有效 MSS
- advmss - 通告的 MSS
- tcp_header_len - TCP 头部长度

  1. RTT 测量
- srtt - 平滑的往返时间（<<3 存储）
- mdev - 往返时间的中等偏差
- rttvar - 平滑的偏差
- rto - 重传超时时间

  1. 拥塞控制
- snd_cwnd - 发送拥塞窗口
- snd_ssthresh - 慢启动阈值
- packets_out - 飞行中的数据包数
- retrans_out - 重传的数据包数
- lost_out - 丢失的数据包数
- sacked_out - SACK 确认的数据包数

  1. 定时器
- retransmit_timer - 重传定时器
- delack_timer - 延迟 ACK 定时器
- timeout - 超时时间

  1. SACK 支持
- duplicate_sack[1] - D-SACK 块
- selective_acks[4] - SACK 块（最多4个）

  1. TCP 选项

  struct tcp_options_received rx_opt 包含：
- 时间戳信息（ts_recent, rcv_tsval, rcv_tsecr）
- 窗口缩放因子（snd_wscale, rcv_wscale）
- SACK 许可标志
- MSS 夹逼值

  1. 接收队列管理
- out_of_order_queue - 乱序数据包队列
- ucopy.prequeue - 预队列（VJ 风格）

  1. 紧急数据
- urg_seq - 紧急指针序列号
- urg_data - 紧急数据和控制标志
- urg_mode - 紧急模式标志

  1. 连接管理
- bind_hash - 绑定哈希桶
- listen_opt - 监听选项（用于 LISTEN 状态）
- accept_queue - 已建立连接的待接受队列
- syn_wait_lock - SYN 等待队列锁

  1. 保活机制
- keepalive_time - 保活时间间隔
- keepalive_intvl - 保活探测间隔
- keepalive_probes - 允许的保活探测次数

  1. 拥塞算法状态

  支持多种拥塞控制算法，结构体中包含：
- Vegas 算法：vegas 结构（beg_snd_nxt, minRTT, cntRTT 等）
- Westwood 算法：westwood 结构（bw_est, rtt_min, cumul_ack 等）
- BIC 算法：bictcp 结构（cnt, last_max_cwnd, last_cwnd 等）

  1. 时间戳和序列号
- write_seq - 发送缓冲区的尾部位置
- pushed_seq - 最后推送的序列号
- copied_seq - 未读取数据的头部位置
- lsndtime - 最后发送数据包的时间戳

  1. ECN（显式拥塞通知）
- ecn_flags - ECN 状态位

  1. 接收方估计
- rcv_rtt_est - 接收方 RTT 估计
- rcvq_space - 接收队列空间估计

  数据结构层次

  struct sock (通用套接字)
  └── struct inet_sock (IPv4 特定)
  └── struct tcp_sock (TCP TCB)

  TIME_WAIT 状态的 TCB

  对于 TIME_WAIT 状态，使用轻量级的 struct tcp_tw_bucket（定义在 include/net/tcp.h:182），包含：
- tw_rcv_nxt, tw_snd_nxt - 序列号
- tw_rcv_wnd - 窗口信息
- tw_timeout - 超时时间
- 时间戳信息等

  这种设计优化了 TIME_WAIT 状态的内存使用，因为该状态在繁忙服务器上可能占用大量连接。
