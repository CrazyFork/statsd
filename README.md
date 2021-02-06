#

-> meta
created time: 2021-02-05 15:31:22
-> end meta


# overview
在接近完成这个repo的时候, 我才理解这个statsd的工作原理. 在我开始翻阅代码之后, 我才发现实际的情况可能要比我预想中的简单.

statsd 的作用就是ta会作为守护进程开启在对应的server端的instance(每个服务器)中. 由于ta的工作机理中的通信部分都是通过
socket来桥接的, 所以在架构上就比较灵活. 对应要发送数据的进程需要向statsd按规定格式(protocol)发送打点信息. 然后statsd那边
会根据配置的interval去计算/上报所有的metrics类型的信息.

# notes
`examples/go/statsd.go` 这个server的实现还是比较清晰的, 也比较简单. 而scala版本的 `examples/StatsD.scala` 对于message格式
的定义那块也是比较清晰.


## main components


* server
* client
* metric_types
* stats.js  
* backend
* management
* proxy


### server

用来监听client端发送的数据, 内置的2种实现(tcp/udp)都是基于socket的. 比较灵活. 这个地方statsd也允许通过配置override默认的server

### client

client 是指需要向statsd发送metric数据的进程, 具体client形式是根据server定的. 所以默认的client都是通过socket向server发送固定格式的信息.


### metric_types

statsd 支持的metric类型.

具体看 `docs/metric_types.md` 文档的描述. 


### stats.js

核心文件, 描述了statsd的主要工作方式. 即, 

读取配置-> 启动server -> 监听socket -> 读取client端发送过来的metric -> 处理对应metric (add count, calc timer, record statsd itself metric)
		-> 启动一个loop, 每次interval 会flush缓存计算的metrics到对应 backend 上
		-> 启动 management server, 处理management operation


### backend

statsd 将各个client端发送过来的数据处理之后, 需要将这些信息发送到某个地方. 要发送的地方就是backend. 显然backend是没个使用者需要根据自己的诉求
去寻找或者实现对应的backend. 


### management

statsd 提供了 management 功能, 也是通过暴露socket来实现的. 可以inspect metrics/del/read config etc...


### proxy



## core files

* `stats.js` 程序入口文件. 最核心的部分
* `lib/process_metrics.js` 里边记录了比较核心的 `timer` type是如何被处理的





# todo

done: 
* shell getopts
* 
pending:
* syslog





# dir

```
.
├── README.md
├── backends												# a various backends
│   ├── console.js
│   ├── graphite.js
│   └── repeater.js
├── docs													# docs
│   ├── additional_tools.md
│   ├── admin_interface.md
│   ├── backend.md											# !Important
│   ├── backend_interface.md
│   ├── client_implementations.md							# !Important
│   ├── cluster_proxy.md
│   ├── graphite.md
│   ├── graphite_pickle.md
│   ├── history.md
│   ├── metric_types.md										# !Important
│   ├── namespacing.md
│   ├── protocol.md
│   ├── server.md											# !Important
│   ├── server_implementations.md
│   └── server_interface.md
├── exampleConfig.js
├── exampleProxyConfig.js
├── examples												# example of how to send metrics to statsd
│   ├── go
│   │   └── statsd.go
├── lib														# src code resides
│   ├── config.js											# config
│   ├── helpers.js											# -
│   ├── logger.js											# logger, support console and syslog
│   ├── mgmt_console.js										# management console
│   ├── mgmt_server.js										# management server
│   ├── process_metrics.js									# handle calc metrics between each interval
│   ├── process_mgmt.js										# set title and handle exit for statsd process
│   └── set.js												# a set implementation
├── proxy.js
├── run_tests.js
├── servers													# builtin server implementations
│   ├── tcp.js
│   └── udp.js
├── stats.js												# main entry, statsd executable binary will link to this file

```

















[#](#) StatsD [![Build Status][travis-ci_status_img]][travis-ci_statsd] [![Join the chat at https://gitter.im/statsd/statsd](https://badges.gitter.im/statsd/statsd.svg)](https://gitter.im/statsd/statsd?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge) [![Docker Pulls](https://img.shields.io/docker/pulls/statsd/statsd)](https://hub.docker.com/r/statsd/statsd)

A network daemon that runs on the [Node.js][node] platform and
listens for statistics, like counters and timers, sent over [UDP][udp] or
[TCP][tcp] and sends aggregates to one or more pluggable backend services (e.g.,
[Graphite][graphite]).

## Key Concepts

* *buckets*

  Each stat is in its own "bucket". They are not predefined anywhere. Buckets
can be named anything that will translate to Graphite (periods make folders,
etc)

* *values*

  Each stat will have a value. How it is interpreted depends on modifiers. In
general values should be integers.

* *flush*

  After the flush interval timeout (defined by `config.flushInterval`,
  default 10 seconds), stats are aggregated and sent to an upstream backend service.


## Installation and Configuration

### Docker
StatsD supports docker in two ways:
* The official docker image on [docker hub](https://hub.docker.com/r/statsd/statsd)
* Building the image from the bundled [Dockerfile](./Dockerfile)

### Manual installation
 * Install Node.js (All [`Current` and `LTS` Node.js versions](https://nodejs.org/en/about/releases/) are supported.)
 * Clone the project
 * Create a config file from `exampleConfig.js` and put it somewhere
 * Start the Daemon:
   `node stats.js /path/to/config`

## Usage
The basic line protocol expects metrics to be sent in the format:

    <metricname>:<value>|<type>

So the simplest way to send in metrics from your command line if you have
StatsD running with the default UDP server on localhost would be:

    echo "foo:1|c" | nc -u -w0 127.0.0.1 8125

## More Specific Topics
* [Metric Types][docs_metric_types]
* [Graphite Integration][docs_graphite]
* [Supported Servers][docs_server]
* [Supported Backends][docs_backend]
* [Admin TCP Interface][docs_admin_interface]
* [Server Interface][docs_server_interface]
* [Backend Interface][docs_backend_interface]
* [Metric Namespacing][docs_namespacing]
* [StatsD Cluster Proxy][docs_cluster_proxy]

## Debugging
There are additional config variables available for debugging:

* `debug` - log exceptions and print out more diagnostic info
* `dumpMessages` - print debug info on incoming messages

For more information, check the `exampleConfig.js`.


## Tests
A test framework has been added using node-unit and some custom code to start
and manipulate StatsD. Please add tests under test/ for any new features or bug
fixes encountered. Testing a live server can be tricky, attempts were made to
eliminate race conditions but it may be possible to encounter a stuck state. If
doing dev work, a `killall statsd` will kill any stray test servers in the
background (don't do this on a production machine!).

Tests can be executed with `./run_tests.sh`.

## History
StatsD was originally written at [Etsy][etsy] and released with a
[blog post][blog post] about how it works and why we created it.

## Inspiration
StatsD was inspired (heavily) by the project of the same name at Flickr.
Here's a post where Cal Henderson described it in depth:
[Counting and timing][counting-timing].
Cal re-released the code recently:
[Perl StatsD][Flicker-StatsD]



[graphite]: http://graphite.readthedocs.org/
[etsy]: http://www.etsy.com
[blog post]: https://codeascraft.etsy.com/2011/02/15/measure-anything-measure-everything/
[node]: http://nodejs.org
[nodemods]: http://nodejs.org/api/modules.html
[counting-timing]: http://code.flickr.com/blog/2008/10/27/counting-timing/
[Flicker-StatsD]: https://github.com/iamcal/Flickr-StatsD
[udp]: http://en.wikipedia.org/wiki/User_Datagram_Protocol
[tcp]: http://en.wikipedia.org/wiki/Transmission_Control_Protocol
[docs_metric_types]: https://github.com/statsd/statsd/blob/master/docs/metric_types.md
[docs_graphite]: https://github.com/statsd/statsd/blob/master/docs/graphite.md
[docs_server]: https://github.com/statsd/statsd/blob/master/docs/server.md
[docs_backend]: https://github.com/statsd/statsd/blob/master/docs/backend.md
[docs_admin_interface]: https://github.com/statsd/statsd/blob/master/docs/admin_interface.md
[docs_server_interface]: https://github.com/statsd/statsd/blob/master/docs/server_interface.md
[docs_backend_interface]: https://github.com/statsd/statsd/blob/master/docs/backend_interface.md
[docs_namespacing]: https://github.com/etsy/statsd/blob/master/docs/namespacing.md
[docs_cluster_proxy]: https://github.com/etsy/statsd/blob/master/docs/cluster_proxy.md
[travis-ci_status_img]: https://travis-ci.org/statsd/statsd.svg?branch=master
[travis-ci_statsd]: https://travis-ci.org/statsd/statsd
