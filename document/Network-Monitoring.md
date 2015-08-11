#Per-container的网络监控和隔离

Mesos  在  Linux  中提供支持  per-container  的网络监控和隔离。网络隔离防止单个容器用尽可用的网络端口，消费不公平的网络宽带或者推迟包传输。每个活动的容器的网络统计数据被发布通过设置在`/monitor/statistics.json`  在slave 的端点。网络隔离是透明的对于大多数的  task  在一个  slave  中（那些绑定到0端口， 让  kernel 分配端口）。


##安装

默认情况下  Per-container 不支持网络监视和隔离。要想实现这个功能，你需要安装附加依赖和配置在构建过程中。


##Prerequisites

Per-container的网络监控和隔离只支持  版本3.6 或者更高的Linux。另外，kernel  必须包含这些补丁（兼容kernel版本3.15）



- [6a662719c9868b3d6c7d26b3a085f0cd3cc15e64](https://github.com/torvalds/linux/commit/6a662719c9868b3d6c7d26b3a085f0cd3cc15e64)


- [0d5edc68739f1c1e0519acbea1d3f0c1882a15d7](https://github.com/torvalds/linux/commit/0d5edc68739f1c1e0519acbea1d3f0c1882a15d7)


- [e374c618b1465f0292047a9f4c244bd71ab5f1f0](https://github.com/torvalds/linux/commit/e374c618b1465f0292047a9f4c244bd71ab5f1f0)


- [25f929fbff0d1bcebf2e92656d33025cd330cbf8](https://github.com/torvalds/linux/commit/25f929fbff0d1bcebf2e92656d33025cd330cbf8)


以下包是对  slave  的要求：



- [libnl3](http://www.infradead.org/~tgr/libnl/) >= 3.2.26


- [iproute](http://www.linuxfoundation.org/collaborate/workgroups/networking/iproute2) >= 2.6.39 is advised for debugging purpose but not required.

[iproute](http://www.linuxfoundation.org/collaborate/workgroups/networking/iproute2) >= 2.6.39建议用于使用，不是必须要求的。

另外， 如果你正在构建从  source， 你还需要编译  libnl3  依赖包。



- [libnl3-devel / libnl3-dev](http://www.infradead.org/~tgr/libnl/) >= 3.2.26



#构建

 通过  per-container网络监控和隔离的支持构建  Mesos， 你需要增加一个配置选项：

    $ ./configure --with-network-isolator
    $ make



#配置

在  slave  中启用  Per-container  网络监控和隔离， 需要在  slave  命令行  `--isolation`  中添加  `network/port_mapping` 。

    `--isolation="network/port_mapping"`

如果这个  slave  不支持编译  per-container  网络监控和隔离，它将拒绝开始并打印一个错误：

    I0708 00:17:08.080271 44267 containerizer.cpp:111] Using isolation: network/port_mapping
    Failed to create a containerizer: Could not create MesosContainerizer: Unknown or unsupported
    isolator: network/port_mapping




#配置网络端口

没有网络隔离， 在一个主机上的所有容器将分享  slave  的公开  IP  地址， 可以绑定到操作系统允许的任何端口。

当启用网络隔离功能时， slave  中的每个容器都有一个单独的网络堆栈（via Linux [network namespaces](http://lwn.net/Articles/580893/)）。全部的容器仍分享  slave  中的同一个公开 IP（使得服务发现的机器不需要改变）。 slave  会分配给每个容器不重复的端口（在端口范围内），并且仅有包可以从这些分配的端口范围内被交付。请求 kernel 分配端口（通过绑定到端口0）的应用程序将从容器分配的范围给定端口。应用程序能够绑定到容器指定范围外的端口，但是从这些端口的数据包将被主机丢弃。

Mesos 提供两种范围的端口容器：


- OS allocated “[ephemeral](https://en.wikipedia.org/wiki/Ephemeral_port)” ports are assigned by the OS in a range specified for each container by Mesos.



- Mesos allocated “non-ephemeral” ports are acquired by a framework using the same Mesos resource offer mechanism used for cpu, memory etc. for allocation to executors/tasks as required.


另外， 主机本身将需要临时的网络通信端口。你需要在主机上配置3个不重复的端口范围。

#临时端口范围

当前，通过使用`sysctl net.ipv4.ip_local_port_range `命令，配置主机的临时端口范围在任何时间都能被发现。 如果端口需要预留  slave  容器， 临时端口范围可以被更新在`/etc/sysctl.conf`中.

在更新后重新启动，端口已经被其他进程正在使用。例如， 通过增加以下部分：

    # net.ipv4.ip_local_port_range defines the host ephemeral port range, by
    # default 32768-61000.  We reduce this range to allow the Mesos slave to
    # allocate ports 32768-57344
    # net.ipv4.ip_local_port_range = 32768 61000
    net.ipv4.ip_local_port_range = 57345 61000


#容器端口范围


容器的临时和非临时的端口范围通过使用  `--resources`  进行配置。 非临时的端口范围被提供给  master， 任何提供给分配的  framework。

临时端口的范围被  slave  再细分， 通过`ephemeral_ports_per_container` （默认是1024）给每一个容器。在 slave  中最大数量的容器将因此被限制在大约：

    number of ephemeral_ports / ephemeral_ports_per_container

 当临时端口范围已经被用尽时， 使用  `--max_executors_per_slave` 标记防止分配更多的  executors 在一个  slave。

建议通过把  `ephemeral_ports_per_container`  设置成2的整数幂和绑定低的临时端口范围的倍数`ephemeral_ports_per_container`做法，最大限度的减少CPU的开销在包的进程中。例如：


    --resources=ports:[31000-32000];ephemeral_ports:[32768-57344] \
    --ephemeral_ports_per_container=512



#容器运输的限速

通过对容器到网络的出站流量限制速率，以防止一个单独容器消费全部的可用网络资源，会对同一主机上其他容器有不利的影响。` --egress_rate_limit_per_container` 这个标记用于限制每容器的指定带宽。网络流量导致此限制被突破，被延迟以后传输。 TCP  协议将会调解增长的延迟和减少传输的速率确保没有数据包被丢弃。

    --egress_rate_limit_per_container=100MB

我们不限制传入的速率，因为我们只能修改已接收后由主机和任何拥堵已经发生的网络流量。



#出口流量的隔离


稍后传送延迟的网络数据可以增加延迟和抖动（变化）的接口上的所有流量。通过使用流分类和隔离在容器端口范围，Mesos能够减少碰撞在同一主机中的其他容器。 使用`--egress_unique_flow_per_container` 标记启动。


    --egress_unique_flow_per_container

#全部放在一起

一个完成的  slave  命令行能够启动网络隔离，预留端口57345-61000作为主机临时端口， 32768-57344作为容器的临时端口， 31000-32000端口由框架分配的非临时端口，限制容器的传输带宽到300兆位/秒， 启动命令是：


    mesos-slave \
    --isolation=network/port_mapping \
    --resources=ports:[31000-32000];ephemeral_ports:[32768-57344] \
    --ephemeral_ports_per_container=1024 \
    --egress_rate_limit_per_container=37500KB \
    --egress_unique_flow_per_container

#监控容器网络统计

Mesos  公开统计数据来自  Linux   网络栈对每一个容器网络在  `/monitor/statistics.json`  的 slave  端点。


从容器内部的网络接口， 根据  `statistics`  报告以下的数据：

Metric&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Description&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Type
----------


`net_rx_bytes`&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;接收的字节&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Counter

----------

`net_rx_dropped`&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;接收时丢掉数据包&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Counter

----------

`net_rx_errors`&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;接收到的错误报告&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Counter

----------
`net_rx_packets`&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;接收到的数据包&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Counter

----------
`net_tx_bytes`&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;发送的字节&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Counter

----------
`net_tx_dropped`&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;发送时丢掉数据包&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Counter

----------
`net_tx_errors`&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;错误报告发送&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Counter

----------
`net_tx_packets`&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;发送包&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Counter


----------
此外：[Linux Traffic Control](http://tldp.org/HOWTO/Traffic-Control-HOWTO/intro.html) 报告以下统计数据元素，可以实现带宽限制和减少膨胀通过`statistics/net_traffic_control_statistics`。每一个元素包括下列内容：

Metric&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Description&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Type
----------

 `backlog`&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;字节排队等待传输[1]&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Gauge


----------
 `bytes`&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;发送字节&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Counter

----------
`drops`&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;发送时丢掉数据包&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;  Counter

----------
`packets`&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;数据包发送&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;  Counter

----------
`qlen`&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;包排队等待传输&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;  Gauge

----------
`ratebps`&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;传输速率[3]（字节/秒）&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Gauge

----------
`ratepps`&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;传输速率[3]（包/秒）&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Gauge

----------
`requeues`&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;由于资源冲突，数据包发送失败&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Counter

----------

[1] Backlog 仅仅报告了 bloat_reduction 接口。

[2] Overlimits 仅仅报告了  bw_limit  接口。

[3] 目前始终报告是 0  由底层交通控制元素。

 例如， 这些都是统计数据， 你可用通过  `/monitor/statistics.json`  打开网络监控在一个  slave ：

    $ curl -s http://localhost:5051/monitor/statistics.json | python2.6 -mjson.tool
    [
    {
    "executor_id": "job.1436298853",
    "executor_name": "Command Executor (Task: job.1436298853) (Command: sh -c 'iperf ....')",
    "framework_id": "20150707-195256-1740121354-5150-29801-0000",
    "source": "job.1436298853",
    "statistics": {
    "cpus_limit": 1.1,
    "cpus_nr_periods": 16314,
    "cpus_nr_throttled": 16313,
    "cpus_system_time_secs": 2667.06,
    "cpus_throttled_time_secs": 8036.840845388,
    "cpus_user_time_secs": 123.49,
    "mem_anon_bytes": 8388608,
    "mem_cache_bytes": 16384,
    "mem_critical_pressure_counter": 0,
    "mem_file_bytes": 16384,
    "mem_limit_bytes": 167772160,
    "mem_low_pressure_counter": 0,
    "mem_mapped_file_bytes": 0,
    "mem_medium_pressure_counter": 0,
    "mem_rss_bytes": 8388608,
    "mem_total_bytes": 9945088,
    "net_rx_bytes": 10847,
    "net_rx_dropped": 0,
    "net_rx_errors": 0,
    "net_rx_packets": 143,
    "net_traffic_control_statistics": [
    {
    "backlog": 0,
    "bytes": 163206809152,
    "drops": 77147,
    "id": "bw_limit",
    "overlimits": 210693719,
    "packets": 107941027,
    "qlen": 10236,
    "ratebps": 0,
    "ratepps": 0,
    "requeues": 0
    },
    {
    "backlog": 15481368,
    "bytes": 163206874168,
    "drops": 27081494,
    "id": "bloat_reduction",
    "overlimits": 0,
    "packets": 107941070,
    "qlen": 10239,
    "ratebps": 0,
    "ratepps": 0,
    "requeues": 0
    }
    ],
    "net_tx_bytes": 163200529816,
    "net_tx_dropped": 0,
    "net_tx_errors": 0,
    "net_tx_packets": 107936874,
    "perf": {
    "duration": 0,
    "timestamp": 1436298855.82807
    },
    "timestamp": 1436300487.41595
    }
    }
    ]
