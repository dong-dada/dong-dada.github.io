---
layout: post
title:  "RDMA 论文阅读"
date:   2024-09-04 12:00:00 +0800
---

* TOC
{:toc}

## FaSST: Fast, Scalable and Simple Distributed Transactions with Two-Sided (RDMA) Datagram RPCs

[原文链接](https://www.usenix.org/conference/osdi16/technical-sessions/presentation/kalia)

FaSST 是个分布式事务系统。

现有的基于 RDMA 的分布式事务系统使用的是单边 (one-side) 的 RDMA 原语，有若干缺点，FaSST 通过双边(two-side)的不可靠数据报规避了这些缺点。

现有系统中使用 RDMA 单边原语来节省远端 CPU。在事务处理系统中的数据结构会更复杂，比如需要先查索引再查数据，所以需要多次 RDMA 单边操作才能完成。

这篇论文提出通过双边的不可靠数据报消息，构建 RPC 系统。

使用不可靠数据报的时候需要处理报文丢失等问题，但丢包发生是非常罕见的，论文提到在 IB 网络中传输 50PB 数据没有发现丢包。对于罕见的丢包情况，论文的做法是在 RPC 请求方设置超时。

RDMA 的 RC 模式提供了可靠连接服务，这种一对一的通信模式代价是需要创建许多 QP 对，如果需要与 N 个远端机器通信，本端就需要创建 N 个 QP。这种方式存在扩展性问题，因为 NIC 上的内存是有限的，缓存不了太多 QP 状态。

RDMA 的 UD 模式提供了不可靠数据包服务，它只能支持 SEND/RECV 这种双边通信，但它可以使用一个 QP 和 N 个远端主机通信，扩展性更好。

商用 NIC 目前没有提供 RD(可靠数据报) 传输服务。

使用单边 RDMA 操作设计分布式事务系统时的一些优化策略：
- Value-in-index: FaRM 通过把数据和索引放在一起的做法，可以用 1 次 READ 操作读取到索引和数据，但是这种做法会让传输的数据量放大到 6-8 倍。
- Caching the index: 缓存索引到本机，这样只需一次 READ 操作读取数据就可以了，代价是索引带来的内存占用。

数据报相对连接的优势:
- 创建 QP 的数量少，每个线程可以独占 QP，不需要处理共享 QP 带来的竞争问题。
- 通过 "Doorbell batching" 门铃批处理可以降低 CPU 占用。门铃批处理的意思是向 NIC 投递多个 WR 之后，再写入 NIC 上的门铃寄存器，要求 NIC 除了这批请求。对于数据报而言可以投递一批任务再敲响门铃，但是对于连接 QP，进程必须敲响多个门铃才可以，取决于这批请求的目标有多少个。

