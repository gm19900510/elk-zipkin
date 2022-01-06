# 一、架构思路分析
一般日志分析场景：直接在日志文件中 `grep`、`awk` 就可以获得自己想要的信息。

在规模较大的场景中，此方法效率低下，面临问题包括日志量太大如何归档、文本搜索太慢怎么办、如何多维度查询。

需要集中化的日志管理，所有服务器上的日志收集汇总。常见解决思路是建立集中式日志收集系统，将所有节点上的日志统一收集，管理，访问。

一般大型系统是一个分布式部署的架构，不同的服务模块部署在不同的服务器上，问题出现时，大部分情况需要根据问题暴露的关键信息，定位到具体的服务器和服务模块，构建一套集中式日志系统，可以提高定位问题的效率。

一个完整的集中式日志系统，需要包含以下几个主要特点：

- 收集－能够采集多种来源的日志数据
- 传输－能够稳定的把日志数据传输到中央系统
- 存储－如何存储日志数据
- 分析－可以支持 UI 分析
- 警告－能够提供错误报告，监控机制

ELK提供了一整套解决方案，并且都是开源软件，之间互相配合使用，完美衔接，高效的满足了很多场合的应用。目前主流的一种日志系统。

# 二、ELK简介
`ELK`是三个开源软件的缩写，分别表示：`Elasticsearch` , `Logstash`, `Kibana` , 它们都是开源软件。新增了一个`FileBeat`，它是一个轻量级的日志收集处理工具，`Filebeat`占用资源少，适合于在各个服务器上搜集日志后传输给`Logstash`，官方也推荐此工具。

`Elasticsearch`是个开源分布式搜索引擎，提供搜集、分析、存储数据三大功能。它的特点有：分布式，零配置，自动发现，索引自动分片，索引副本机制，`restful`风格接口，多数据源，自动搜索负载等。

`Logstash` 主要是用来日志的搜集、分析、过滤日志的工具，支持大量的数据获取方式。一般工作方式为`c/s`架构，`client`端安装在需要收集日志的主机上，`server`端负责将收到的各节点日志进行过滤、修改等操作在一并发往`elasticsearch`上去。

`Kibana` 也是一个开源和免费的工具，`Kibana`可以为 `Logstash` 和 `ElasticSearch` 提供的日志分析友好的 `Web` 界面，可以帮助汇总、分析和搜索重要数据日志。

`Filebeat`隶属于`Beats`。目前`Beats`包含四种工具：

- Packetbeat - 搜集网络流量数据
- Topbeat - 搜集系统、进程和文件系统级别的 CPU 和内存使用情况等数据
- Filebeat - 搜集文件数据
- Winlogbeat - 搜集 Windows 事件日志数据

# 三、ELK架构
## 3.1 架构组合方式一
![在这里插入图片描述](https://img-blog.csdnimg.cn/042b6ea652b8462499671e5d07e4562f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)
这是最简单的一种`ELK`架构方式。

优点是搭建简单，易于上手；缺点是`Logstash`耗资源较大，运行占用`CPU`和内存高。另外没有消息队列缓存，存在数据丢失隐患。

此架构由`Logstash`分布于各个节点上搜集相关日志、数据，并经过分析、过滤后发送给远端服务器上的`Elasticsearch`进行存储。`Elasticsearch`将数据以分片的形式压缩存储并提供多种API供用户查询，操作。用户亦可以更直观的通过配置`Kibana Web`方便的对日志查询，并根据数据生成报表。

## 3.2 架构组合方式二
![在这里插入图片描述](https://img-blog.csdnimg.cn/e9beac553bbc45af820a6084d8764e7a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)
此种架构引入了消息队列机制，位于各个节点上的`Logstash Agent`先将数据/日志传递给`Kafka`（或者`Redis`），并将队列中消息或数据间接传递给`Logstash`，`Logstash`过滤、分析后将数据传递给`Elasticsearch`存储。最后由`Kibana`将日志和数据呈现给用户。

因为引入了`Kafka`（或者`Redis`），所以即使远端`Logstash server`因故障停止运行，数据将会先被存储下来，从而避免数据丢失。
## 3.3 架构组合方式3
![在这里插入图片描述](https://img-blog.csdnimg.cn/5cbc43b230d9424ab55fbf16eb5b1753.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)
此种架构将收集端`logstash`替换为`beats`，更灵活，消耗资源更少，扩展性更强。同时可配置`Logstash` 和`Elasticsearch` 集群用于支持大集群系统的运维日志数据监控和查询。

# 四、Filebeat工作原理
`Filebeat`由两个主要组件组成：`prospectors` 和 `harvesters`。这两个组件协同工作将文件变动发送到指定的输出中。
![在这里插入图片描述](https://img-blog.csdnimg.cn/e2164a39bdda4446934393967c599c88.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)
> `Harvester`（收割机）：负责读取单个文件内容。
 
每个文件会启动一个`Harvester`，每个`Harvester`会逐行读取各个文件，并将文件内容发送到制定输出中。

`Harvester`负责打开和关闭文件，意味在`Harvester`运行的时候，文件描述符处于打开状态，如果文件在收集中被重命名或者被删除，`Filebeat`会继续读取此文件。所以在`Harvester`关闭之前，磁盘不会被释放。

默认情况`Filebeat`会保持文件打开的状态，直到达到`close_inactive`（如果此选项开启，filebeat会在指定时间内将不再更新的文件句柄关闭，时间从`Filebeat`读取最后一行的时间开始计时。若文件句柄被关闭后，文件发生变化，则会启动一个新的`Filebeat`。关闭文件句柄的时间不取决于文件的修改时间，若此参数配置不当，则可能发生日志不实时的情况，由`scan_frequency`参数决定，默认`10s`。`Harvester`使用内部时间戳来记录文件最后被收集的时间。例如：设置`5m`，则在`Harvester`读取文件的最后一行之后，开始倒计时`5分钟`，若`5分钟`内文件无变化，则关闭文件句柄。默认`5m`）。

> `Prospector`（勘测者）：负责管理`Harvester`并找到所有读取源。

`Prospector`会为每个文件启动一个`Harvester`。`Prospector`会检查每个文件，看`Harvester`是否已经启动，是否需要启动，或者文件是否可以忽略。若`Harvester`关闭，只有在文件大小发生变化的时候`Prospector`才会执行检查。只能检测本地的文件。

>`Filebeat`如何记录文件状态 ?

将文件状态记录在文件中（默认在`/var/lib/filebeat/registry`）。此状态可以记住`Harvester`收集文件的偏移量。若连接不上输出设备，如`ES`等，`Filebeat`会记录发送前的最后一行，并再可以连接的时候继续发送。`Filebeat`在运行的时候，`Prospector`状态会被记录在内存中。`Filebeat`重启的时候，利用`registry`记录的状态来进行重建，用来还原到重启之前的状态。每个`Prospector`会为每个找到的文件记录一个状态，对于每个文件，`Filebeat`存储唯一标识符以检测文件是否先前被收集。

>`Filebeat`如何保证事件至少被输出一次 ?

`Filebeat`之所以能保证事件至少被传递到配置的输出一次，没有数据丢失，是因为`Filebeat`将每个事件的传递状态保存在文件中。在未得到输出方确认时，`Filebeat`会尝试一直发送，直到得到回应。若`Filebeat`在传输过程中被关闭，则不会再关闭之前确认所有时事件。任何在`Filebeat`关闭之前为确认的时间，都会在`Filebeat`重启之后重新发送。这可确保至少发送一次，但有可能会重复。可通过设置`shutdown_timeout` 参数来设置关闭之前的等待事件回应的时间（默认禁用）。

 

# 五、Logstash工作原理
`Logstash`事件处理有三个阶段：`inputs → filters → outputs`。是一个接收，处理，转发日志的工具。支持系统日志，错误日志，应用日志，总之包括所有可以抛出来的日志类型。
![在这里插入图片描述](https://img-blog.csdnimg.cn/ffd9db5ceef94fb7b54e071954a309dc.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)
![在这里插入图片描述](https://img-blog.csdnimg.cn/0ec5b39009d140abb9612ea2b0bf2162.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)


>`Input`：输入数据到`logstash`。

一些常用的输入为：

- `file`：从文件系统的文件中读取，类似于`tail -f`命令
- `syslog`：在`514`端口上监听系统日志消息，并根据`RFC3164`标准进行解析
- `redis`：从`redis service`中读取
- `beats`：从`filebeat`中读取

更多详细介绍可转阅：[https://www.cnblogs.com/dalianpai/p/11970810.html](https://www.cnblogs.com/dalianpai/p/11970810.html)

>`Filters`：数据中间处理，对数据进行操作。

一些常用的过滤器为：

- `grok`：解析任意文本数据，`grok`是 `Logstash` 最重要的插件。它的主要作用就是将文本格式的字符串，转换成为具体的结构化的数据，配合正则表达式使用。内置`120`多个解析语法。

	官方提供的grok表达式：[https://github.com/logstash-plugins/logstash-patterns-core/tree/master/patterns](https://github.com/logstash-plugins/logstash-patterns-core/tree/master/patterns)
	
	`grok`在线调试：[https://grokdebug.herokuapp.com/](https://grokdebug.herokuapp.com/)

- `mutate`：对字段进行转换。例如对字段进行删除、替换、修改、重命名等。

- `drop`：丢弃一部分`events`不进行处理。

- `clone`：拷贝 `event`，这个过程中也可以添加或移除字段。

- `geoip`：添加地理信息(为前台`kibana`图形化展示使用)

>`Outputs`：`outputs`是`logstash`处理管道的最末端组件。一个`event`可以在处理过程中经过多重输出，但是一旦所有的`outputs`都执行结束，这个`event`也就完成生命周期。

一些常见的`outputs`为：

- `elasticsearch`：可以高效的保存数据，并且能够方便和简单的进行查询。

- `file`：将`event`数据保存到文件中。

- `graphite`：将`event`数据发送到图形化组件中，一个很流行的开源存储图形化展示的组件。

- `Codecs`：`codecs` 是基于数据流的过滤器，它可以作为`input`，`output`的一部分配置。`Codecs`可以帮助你轻松的分割发送过来已经被序列化的数据。

	一些常见的`codecs`：
	
	`json`：使用json格式对数据进行编码/解码。
	
	`multiline`：将汇多个事件中数据汇总为一个单一的行。比如：java异常信息和堆栈信息。

>`Batcher`：负责批量的从`queue`中取数据。

> `Queue`分类：
	
`In Memory` ： 无法处理进程`Crash`、机器宕机等情况，会导致数据丢失。
`Persistent Queue In Disk`：可处理进程`Crash`等情况，保证数据不丢失，保证数据至少消费一次，充当缓冲区，可以替代`kafka`等消息队列的作用。

