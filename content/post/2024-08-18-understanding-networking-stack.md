---
title: Understanding OS's network stack
author: Dhaval Shah
type: post
date: 2024-08-18T07:00:50+00:00
url: /understanding-fundamentals-of-networking-stack/
categories:
  - Performance
tags:
  - performance
  - tuning
  - linux
thumbnail: "images/wp-content/uploads/2022/09/linux-performance-image.png"
---


[![](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/linux-performance-image.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/linux-performance-image.png)
-----------------------------------------------------------------------------------------------------------------------------------------
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

------ // Image // --------
The journey of sender application's data to the [NIC's](https://en.wikipedia.org/wiki/Network_interface_controller) interface begins in user space, where an application generates data to be transmitted over the network. This data is transferred to the kernel space through a system call via [Socket's](https://en.wikipedia.org/wiki/Unix_domain_socket) Send Buffers, as a **_struct sk_buff_** (socket buffer i.e. SKB) - a data type that holds the data and its associated metadata. The SKB then traverses the transport, network, and link layers, where relevant headers for protocols like TCP/UDP, IPv4, and MAC are added.

Link layer mainly comprises of **queueing discipline** (qdiscs). **qdiscs** operate as a parent-child hierarchy, allowing multiple child *qdiscs* to be configured under a parent qdisc. Depending on priority, the _qdiscs_ determines when the packet is to be forwarded to the **driver queue** (a.k.a Ring Buffer). Finally, the NIC reads the packets and eventually deque them on wire :phew:

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
Congestion control algorithms play a vital role in managing network traffic, ensuring efficient data transmission, and maintaining network stability

4. Other TCP Options

5. Queueing Disciplines

6. Socket Options


^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Key tools that can help in gaining kernel level metrics
### _sar_
It is a tool to collect system metrics

[![Network - sar](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/network-sar.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/network-sar.png)

#### Activate _sar_
Edit the file _/etc/default/sysstat_ by changing _ENABLED_ from "false" to "true" and then restart its service by below command

{{< highlight bash >}}
  sudo service sysstat restart
{{< /highlight >}}

#### _sar_ options and its metrics understanding
Various options provided by Linux version -
1. -n DEV: Network interface statistics
2. -n EDEV: Network interface errors
3. -n TCP: TCP statistics
4. -n ETCP: TCP error statistics
5. -n SOCK: Socket usage
6. -n IP: IP datagram statistics
7. -n EIP: IP error statistics

[![Network - sar-tcp](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/network-sar-tcp.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/network-sar-tcp.png)

### _tcptop_
It is a tool that emits TCP throughput by host and its processes

[![Network - tcptop](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/network-tcptop.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/network-tcptop.png)

- *RX_KB* : Received traffic in KB
- *TX_KB* : Traffic sent in KB

## Disk I/O
Disk I/O can have huge impact on system performance and this in turn may lead to high latency of application. A very common scenario that might lead to performance issues - System is waiting for Disk I/O to get completed and as a result CPU is idle due to blocking I/O operation

Key tools that can help in gaining kernel level metrics
### _df_
It mainly shows all the file systems mounted along with its size and used space

[![Disk - df](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/disk-df.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/disk-df.png)

### _iostat_
Emits summary of I/O statistics per disk

[![Disk - iostat](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/disk-iostat.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/disk-iostat.png)

Note -
- Use -x to get extended metrics
- Few key metrics to understand the throughput / avg size of read/write - *r/s*, *w/s*, *rKb/s*, *rKb/s*
- *aqu-sz* : Measurement of saturation which indicates queue length of requests
- *%util* : Indicates utilization i.e. percentage of time device spent doing some work. A high value **may** indicate saturation

### _biotop_
Emits summary of disk I/O by process

[![Disk - biotop](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/disk-biotop.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/disk-biotop.png)

It mainly helps to identify source of load being written to disk is coming from. It in a way helps to identify the root cause of high consumption of resources

## Operating System
Operating System is a program on which application along with its binaries runs. It also manages system along with hardware, CPU, memory etc.

Few tools that can help to get key metrics -

### _opensnoop_
Allows you to see all the files being opened by all the processes on your system, in almost near to real time

{{< highlight bash >}}
  sudo opensnoop-bpfcc
{{< /highlight >}}

### _execsnoop_
It list of all the processes being started on the machine by listening for call to *exec* variant in the kernel

{{< highlight bash >}}
  sudo execsnoop-bpfcc
{{< /highlight >}}


## Other handy Linux tools from Performance Engineering standpoint

### *Uptime*
It mainly indicates - 
- how long system has been up
- average load on the system which mainly helps in understanding the pattern (increasing or decreasing load)

[![Misc - uptime](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/misc-uptime.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/misc-uptime.png)

Note:
1. The three numbers in Load Average represent a moving window sum average of processes competing over CPU time to run over 1, 5, and 15 minutes. The numbers are exponentially scaled, so a number twice as large doesn’t mean twice as much load
2. If the numbers are decreasing sharply, it **might** mean that the program which was eating up all the resources has already completed its execution
3. Increasing number would **mostly** indicate rising load on the system

### Exit code of applications
One can return exit code returned by previous command

{{< highlight bash >}}
  echo $?
{{< /highlight >}}

[![Misc - exitcode](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/misc-exitcode.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/misc-exitcode.png)

Note - codes in range 128-192 are decoded by using 128 + n, where n is the number of the kill signal

[![Misc - exitcode kill](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/misc-exitcode-kill.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/misc-exitcode-kill.png)

### Quick check on memory and CPU utilization
{{< highlight bash >}}
  top -n1 -o+%MEM
{{< /highlight >}} 

[![Misc - Top mem cpu](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/misc-topomem.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/misc-topomem.png)

### _dmesg_
Linux utility that displays kernel messages

misc-dmesg
[![Misc - dmesg](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/misc-dmesg.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/misc-dmesg.png)

# Conclusion
Architecting and developing low latency, high throughput enterprise applications needs an overall understanding of not only software but also about the underlying OS behavior along with its key components viz. CPU, RAM, Network and Disk. In this article I tried to show how quickly one can use some of the commands to get insights from OS level whilst troubleshooting performance issues.

P.S :
List of commands and their output shown above are based on [Chaos Engineering](https://www.amazon.in/Chaos-Engineering-reliability-controlled-disruption/dp/1617297755)