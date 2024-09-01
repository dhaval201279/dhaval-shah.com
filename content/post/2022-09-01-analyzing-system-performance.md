---
title: Linux tools for analyzing System Performance
author: Dhaval Shah
type: post
date: 2022-09-03T07:00:50+00:00
url: /linux-tools-4-analyzing-system-performance/
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
In today's contemporary world of enterprise software where massively used applications are expected to scale and run seamlessly at **extreme high loads** e.g. [Scaling Hotstar for 25.3 million users](https://youtu.be/QjvyiyH4rr0), *system performance* becomes one of the key tenant of architecting high throughput, low latency applications along with capability of ease in scaling as per business / end consumer needs .

*System performance* is a very broad term as it would encompass entire gambit of computer system i.e. all the software and hardware components that comes within the path of a user request. In this article I will be mainly covering key tools to analyze system resources -
1. Four main logical components of a physical computer - CPU, RAM, Networking, and Disk I/O
2. Operating System

[![App - Top Down View](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/app-sw-os-top-down-view.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/app-sw-os-top-down-view.png)

## CPU
CPU is mainly responsible for executing all software and are most often probable hot spot for system performance issue. Key Linux tools that can help in finding deeper insights -

### *top*
Shows top CPU consuming processes along with its CPU usage

[![CPU - top](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/cpu-top.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/cpu-top.png)

Above numbers pertaining to CPU utilization means - 
- us (user time)—The percentage of time the CPU spent in user space.
- sy (system time)—The percentage of time the CPU spent in kernel space.
- ni (nice time)—The percentage of time spent on low-priority processes.
- id (idle time)—The percentage of time the CPU spent doing literally nothing. (It can’t stop!)
- wa (I/O wait time)—The percentage of time the CPU spent waiting on I/O.
- hi (hardware interrupts)—The percentage of time the CPU spent servicing hardware interrupts.
- si (software interrupts)—The percentage of time the CPU spent servicing software interrupts.
- st (steal time)—The percentage of time a hypervisor stole the CPU to give it to someone else. This kicks in only in virtualized environments.

#### Niceness
It mainly indicates how happy a process is to give CPU cycles to other more high priority process i.e. how nice it is with others :) Allowed values range from -20 to 19
### *mpstat*
Its multi processor statistics tool tha can emit metrics per CPU. It is similar to _top_ but with load split separately for each processor.

[![CPU - mpstat](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/cpu-mpstat.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/cpu-mpstat.png)

### Note
When your application slows down considerably due to other background processes than one may use _niceness_ property to set higher relative property for application process. This has major limitation, as it does not control how much CPU to allocate to the process
In order to overcome above limitation one can use *control groups* which mainly allows user to specify exact magnitude of resources i.e. CPU, Memory, I/O that kernel should allocate to group of processes

## RAM
RAM determines the operating capacity of a system at any given time. Read speed of RAM determines two things -
1. How fast your CPU can load data into its cache
2. How much of system performance drops when CPU is forced to read from RAM instead of from its cache

Key Linux tools that can provide deeper insights -

### _free_
It shows utilization of RAM.

[![RAM - free](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/ram-free.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/ram-free.png)

'Available' column as shown above is a key metric to check RAM utilization

### _top_
It gives an overview of the memory along with CPU utilization (as discussed in previous section) of system.  By default, the output is sorted by the value of field _%CPU_ utilization of process

[![RAM - top](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/ram-top.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/ram-top.png)

### _vmstat_
It mainly emits virtual memory statistics

[![RAM - vmstat](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/ram-vmstat.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/ram-vmstat.png)

Key fields to understand from performance standpoint : 
- r -> Indicates no. of processes running / waiting to run which implicitly helps us to know the saturation level of the system
- b -> Indicates no. of processes
- in -> Total no. of interrupts
- cs -> Total no. of context switches

### _oomkill-bpfcc_
It basically works by tracing *oom_kill_process* and emitting information whenever out of memory happens. In order to leverage this utility, open a terminal and execute below command and keep the terminal open.

{{< highlight bash >}}
  sudo oomkill-bpfcc
{{< /highlight >}}

Whenever OOM occurs below output can be seen

[![RAM - ram-oomkil-bpfcc](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/ram-oomkil-bpfcc.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2022/09/ram-oomkil-bpfcc.png)

## Network
With Distributed Systems, network plays a huge role in System Performance. Typically speaking network would mainly comprise of -
1. Hardware - which would mainly include routers, NIC, switches etc
2. Software - which would mainly include [OS](https://en.wikipedia.org/wiki/Operating_system) [kernel](https://en.wikipedia.org/wiki/Kernel_(operating_system)) that may comprise of device drivers, protocols etc.

Note - At software level, protocols can be further categorized  into 2 :
1. Protocols at kernel level i.e. TCP, UDP etc
2. Protocols at application level i.e. HTTP
One can refer my older posts w.r.t application level protocols and ways to optimize it -
- [Blocking HTTP client via Spring's RestTemplate](https://www.dhaval-shah.com/rest-client-with-desired-nfrs-using-springs-resttemplate/)
- [Reactive HTTP client via Spring's WebClient](https://www.dhaval-shah.com/performant-and-optimal-spring-webclient/)
- [RSocket Vs Webflux](https://www.dhaval-shah.com/performance-comparison-rsocket-webflux/)

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