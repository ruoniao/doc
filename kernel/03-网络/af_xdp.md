# AF_XDP

## Overview

AF_XDP是一种针对高性能数据包处理进行优化的地址族。

本文档假定读者已熟悉BPF和XDP。如果不熟悉，Cilium项目在http://cilium.readthedocs.io/en/latest/bpf/上有一份很好的参考指南。

使用XDP程序中的XDP_REDIRECT操作，程序可以使用bpf_redirect_map()函数将入站帧重定向到其他启用了XDP的netdevs。AF_XDP套接字使得XDP程序可以将帧重定向到用户空间应用程序中的内存缓冲区。

使用普通的socket()系统调用创建AF_XDP套接字（XSK）。每个XSK关联有两个环：RX环和TX环。套接字可以在RX环上接收数据包，也可以在TX环上发送数据包。这些环通过setsockopt XDP_RX_RING和XDP_TX_RING进行注册和大小设置。每个套接字至少需要其中一个环是必需的。RX或TX描述符环指向一个名为UMEM的内存区域中的数据缓冲区。RX和TX可以共享同一个UMEM，这样一来，数据包就不必在RX和TX之间复制。此外，**如果由于可能的重传需要将数据包保留一段时间，指向该数据包的描述符可以被更改以指向另一个并立即重用。这样可以避免数据复制。**

UMEM由若干大小相等的块组成。环中的一个描述符通过引用其addr来引用一个帧。addr只是整个UMEM区域内的偏移量。用户空间使用自己认为最合适的方式（如malloc、mmap、大页等）为这个UMEM分配内存，然后使用新的setsockopt XDP_UMEM_REG将这块内存区域注册到内核。UMEM还有两个环：FILL环和COMPLETION环。应用程序使用填充环向内核发送addr，内核会用RX数据填充它。一旦每个数据包被接收，这些帧的引用就会出现在RX环中。另一方面，完成环包含内核已经完全传输的帧addr，现在可以被用户空间再次使用，用于TX或RX。因此，出现在完成环中的帧addr是之前使用TX环传输的addr。总结一下，RX和FILL环用于RX路径，而TX和COMPLETION环用于TX路径。

然后，套接字最终通过bind()调用绑定到设备的特定队列ID，只有在完成绑定后流量才开始流动。

如果需要，UMEM可以在进程之间共享。如果进程想要这样做，它只需跳过UMEM及其相应的两个环的注册，在绑定调用中设置XDP_SHARED_UMEM标志，并提交它希望共享UMEM的进程的XSK以及自己新创建的XSK套接字。然后，新进程将在自己的RX环中接收到指向此共享UMEM的帧addr引用。请注意，由于环结构是单消费者/单生产者（出于性能原因），新进程必须创建自己的套接字，并关联RX和TX环，因为它不能与其他进程共享。这也是每个UMEM只有一组FILL和COMPLETION环的原因。处理UMEM是单个进程的责任。

那么如何将数据包从XDP程序分发到XSKs呢？这里有一个名为XSKMAP（或完整的BPF_MAP_TYPE_XSKMAP）的BPF映射。用户空间应用程序可以将XSK放置在此映射中的任意位置。然后，XDP程序可以将数据包重定向到此映射中的特定索引位置，此时XDP会验证该映射中的XSK是否确实绑定到该设备和环号。如果没有绑定，则丢弃数据包。如果该索引处的映射为空，则也会丢弃数据包。这也意味着目前必须加载一个XDP程序（并在XSKMAP中有一个XSK）才能通过XSK将任何流量传输到用户空间。

AF_XDP可以以两种不同的模式运行：XDP_SKB和XDP_DRV。如果驱动程序不支持XDP，或者在加载XDP程序时明确选择了XDP_SKB，则会使用XDP_SKB模式，它与通用的XDP支持一起使用SKB，并将数据复制到用户空间。这是一种适用于任何网络设备的回退模式。另一方面，如果驱动程序支持XDP，AF_XDP代码将使用它来提供更好的性能，但数据仍会被复制到用户空间。

## Concepts

为了使用AF_XDP套接字，需要设置一些相关的对象。

Jonathan Corbet还在LWN上写了一篇优秀的文章，“使用AF_XDP加速网络”。可以在https://lwn.net/Articles/750845/找到。

## UMEM

UMEM（User Mode DMA, 用户模式直接内存访问）是一个虚拟连续内存区域，被分成大小相等的帧。一个UMEM与一个netdev以及该netdev的特定队列ID相关联。它通过使用XDP_UMEM_REG setsockopt系统调用来创建和配置（块大小、前置空间、起始地址和大小）。UMEM通过bind()系统调用绑定到netdev和队列ID。

AF_XDP是与单个UMEM相关联的套接字，但一个UMEM可以有多个AF_XDP套接字。要通过套接字A共享通过UMEM创建的UMEM，下一个套接字B可以通过在struct sockaddr_xdp成员sxdp_flags中设置XDP_SHARED_UMEM标志，并将A的文件描述符传递给struct sockaddr_xdp成员sxdp_shared_umem_fd来实现。

UMEM有两个单生产者/单消费者环，用于在内核和用户空间应用程序之间传输UMEM帧的所有权。

## Rings

有四种不同类型的环：填充环（Fill）、完成环（Completion）、RX环和TX环。所有环都是单生产者/单消费者的，因此如果多个进程/线程对它们进行读写，则用户空间应用程序需要显式同步。

UMEM使用两个环：填充环和完成环。与UMEM关联的每个套接字必须具有RX队列、TX队列或两者都有。假设有一个设置有四个套接字（全部都执行TX和RX操作），那么将有一个填充环、一个完成环、四个TX环和四个RX环。

这些环是基于头（生产者）/尾（消费者）的环。生产者在struct xdp_ring生产者成员指出的索引处写入数据环，并增加生产者索引。消费者在struct xdp_ring消费者成员指出的索引处读取数据环，并增加消费者索引。

这些环是通过_RING setsockopt系统调用进行配置和创建的，并使用适当的偏移量通过mmap()映射到用户空间（XDP_PGOFF_RX_RING、XDP_PGOFF_TX_RING、XDP_UMEM_PGOFF_FILL_RING和XDP_UMEM_PGOFF_COMPLETION_RING）。

这些环的大小必须是2的幂大小。

## UMEM Fill Ring

填充环用于将UMEM帧的所有权从用户空间传输到内核空间。UMEM的地址在环中传递。举个例子，如果UMEM是64k，每个块是4k，那么UMEM有16个块，可以在0到64k之间传递地址。

传递到内核的帧用于入口路径（RX环）。

用户应用程序向此环生成UMEM地址。请注意，内核将屏蔽传入的地址。例如，对于2k的块大小，地址的log2(2048) LSB将被屏蔽，这意味着2048、2050和3000引用同一个块。

## UMEM Completetion Ring

完成环用于将UMEM帧的所有权从内核空间传输到用户空间。与填充环类似，UMEM索引被使用。

从内核传输到用户空间的帧是已发送的帧（TX环），可以由用户空间再次使用。

用户应用程序从这个环中消耗UMEM地址。

## RX Ring

RX环是套接字的接收端。环中的每个条目都是一个struct xdp_desc描述符。描述符包含UMEM偏移量（addr）和数据的长度（len）。

如果没有通过填充环将帧传递到内核，RX环上将不会（或不能）出现描述符。

用户应用程序从这个环中消耗struct xdp_desc描述符。

## TX Ring

TX环用于发送帧。填充struct xdp_desc描述符（索引、长度和偏移量）并将其传递到环中。

要开始传输，需要使用sendmsg()系统调用。这在将来可能会放宽。

用户应用程序向这个环生成struct xdp_desc描述符。

## XSKMAP / BPF_MAP_TYPE_XSKMAP


在XDP侧，有一种名为BPF_MAP_TYPE_XSKMAP（XSKMAP）的BPF映射类型，它与bpf_redirect_map()一起使用，用于将入站帧传递给套接字。

用户应用程序通过bpf()系统调用将套接字插入映射中。

请注意，如果一个XDP程序尝试将数据包重定向到与队列配置和netdev不匹配的套接字，该数据包将被丢弃。例如，一个AF_XDP套接字绑定到netdev eth0和队列17。只有针对eth0和队列17执行的XDP程序才能成功地将数据传递给该套接字。请参考示例应用程序（samples/bpf/）中的示例。

## Usage

