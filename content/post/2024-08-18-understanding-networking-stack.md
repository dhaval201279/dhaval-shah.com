---
title: Optimizing Linux's network stack
author: Dhaval Shah
type: post
date: 2024-09-01T02:00:50+00:00
url: /optimizing-linux-network-stack/
categories:
  - Performance
tags:
  - performance
  - tuning
  - linux
thumbnail: "images/wp-content/uploads/2024/09/Overview-of-Networking-Stack-Dark.png"
---
# Background
With Distributed Systems, network plays a huge role in System Performance. Typically speaking network would mainly comprise of -
1. Hardware - which would mainly include routers, NIC, switches etc
2. Software - which would mainly include [OS](https://en.wikipedia.org/wiki/Operating_system) [kernel](https://en.wikipedia.org/wiki/Kernel_(operating_system)) that may comprise of device drivers, protocols etc.

Note - At software level, protocols can be further categorized  into 2 :
1. Protocols at kernel level i.e. TCP, UDP etc
2. Protocols at application level i.e. HTTP - As far as application level protocol and its optimization is concerned, you may refer my older posts :
   - [Blocking HTTP client via Spring's RestTemplate](https://www.dhaval-shah.com/rest-client-with-desired-nfrs-using-springs-resttemplate/)
   - [Reactive HTTP client via Spring's WebClient](https://www.dhaval-shah.com/performant-and-optimal-spring-webclient/)
   - [RSocket Vs Webflux](https://www.dhaval-shah.com/performance-comparison-rsocket-webflux/)

In this article I will mainly be covering ways to optimize network at kernel level. But before that lets try to understand the flow data as a sender at kernel level

# Overview of Linux Networking Stack

[![Simplified Overview of Linux Networking Stack ](https://www.dhaval-shah.com/images/wp-content/uploads/2024/09/Overview-of-Networking-Stack-Dark.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2024/09/Overview-of-Networking-Stack-Dark.png)

The journey of sender application's data to the [NIC's](https://en.wikipedia.org/wiki/Network_interface_controller) interface begins in user space, where an application generates data to be transmitted over the network. This data is transferred to the kernel space through a system call via [Socket's](https://en.wikipedia.org/wiki/Unix_domain_socket) Send Buffers, as a **_struct sk_buff_** (socket buffer i.e. SKB) - a data type that holds the data and its associated metadata. The SKB then traverses the transport, network, and link layers, where relevant headers for protocols like TCP/UDP, IPv4, and MAC are added.

Link layer mainly comprises of **queueing discipline** (qdiscs). **qdiscs** operate as a parent-child hierarchy, allowing multiple child *qdiscs* to be configured under a parent qdisc. Depending on priority, the _qdiscs_ determines when the packet is to be forwarded to the **driver queue** (a.k.a Ring Buffer). Finally, the NIC reads the packets and eventually deque them on wire :sweat_smile:

## Optimizable Areas
1. **TCP Connection Queues** <br />
In order to manage inbound connection, Linux uses 2 types of backlog queues -
   - **_SYN Backlog_** - For incomplete transactions during TCP handshake
   -  **_Listen Backlog_** - For established sessions that are waiting to be accepted by application

Length of both these queues can be tuned independently 

``` bash
# Manages length of SYN Backlog Queue
net.ipv4.tcp_max_syn_backlog = 4096

# Manages length of Listen Backlog Queue
net.core.somaxconn = 1024
```

2. **TCP Buffering** <br />
By managing size of send and receive buffers of socket, data throughput can be tweaked. One needs to be mindful of the fact that large sized buffers guarantees throughput but that gain is at the cost of more memory spent per connection

Read and Write buffers of TCP can be managed via
``` bash
# Defines the minimum, default, and maximum size (in bytes) for the TCP receive buffer
net.ipv4.tcp_rmem = 4096 87380 16777216


# Defines the minimum, default, and maximum size (in bytes) for the TCP send buffer
net.ipv4.tcp_wmem = 4096 65536 16777216

# Enables automatic tuning of TCP receive buffer sizes
net.ipv4.tcp_moderate_rcvbuf = 1
```

3. **TCP Congestion Control** <br />
Congestion control algorithms play a vital role in managing TCP based network traffic, ensuring efficient data transmission, and maintaining network stability

It can be set using

``` bash
# Sets cubic as congestion control algorithm. CUBIC is a modern algorithm designed to perform better in high bandwidth and high latency networks
net.ipv4.tcp_available_congestion_control = cubic
```

4. **Other TCP Options** <br />
Other TCP parameters that can be tweaked

``` bash
# Enables TCP Selective Acknowledgement. Helps in maintaining high throughput along with reduced latency 
net.ipv4.tcp_sack = 1

# Enables TCP Forward Acknowledgement. Helps in faster recovery from multiple packet losses within a single window of data, improving overall TCP performance
net.ipv4.tcp_fack = 1

# Allows reuse of sockets in the TIME-WAIT state for new connections. This can help in reducing the latency associated with establishing new connections, leading to faster response times
net.ipv4.tcp_tw_reuse = 1

# Disables fast recycling of TIME-WAIT sockets. Helps maintain stability and reliability of TCP connections, ensuring that connections are properly closed and all packets are accounted for before the socket is reused
net.ipv4.tcp_tw_recycle = 0
```

5. **Queueing Disciplines (qdiscs)** <br />
_qdiscs_ are algorithms for scheduling and managing network packets

``` bash
# Sets qdisc to fq_codel
net.core.default_qdisc = fq_codel
```

# Conclusion
So we understood how data packet gets transmitted from an application to NIC. And while understanding the flow, we got a bird's eye view of key Linux Kernel components that play a major role from data transmission standpoint. 

Since network is one of the most often blamed component for poor performance of application, we also had a cursory overview of various properties that can be tweaked to optimize its performance.

In case you have experienced with above or any other properties, feel free to share within comments!

# References
1. [Brendan Gregg's System Performance Book](https://www.amazon.in/Systems-Performance-Brendan-Gregg-ebook/dp/B08J5QZPNC)