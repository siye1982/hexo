layout: photo
title: Elasticsearch使用积累(持续更新...)
tags:
  - Elasticsearch
  - 原创
  - 分布式
photos:
date: 2015-09-17 11:15:01
categories: Elasticsearch
---

## 写在前面
接触Elasticsearch已经有几年的时间, 从一开始在自己的普通台式机上搭建单实例来简单记录测试环境的日志, 到现在生产上用9个Elasticsearch实例搭成的集群来记录商城后台服务的全量日志. 使用场景没变,都是用了记录日志, 并做实时检索和统计. 但是随着每天记录的数据量快速递增,QPS快速递增,我们经常会碰到一些性能问题,这些问题一直在驱动我们对Elasticsearch集群进行持续优化, 这里是有必要记录一下我们在使用ES(后续文章中用ES代替Elasticsearch)的过程中积累的经验.
<!--more-->


## 常用插件

* [Head](https://github.com/mobz/elasticsearch-head)
	查看分片情况,操作简单api
* [Bigdesk](https://github.com/lukas-vlcek/bigdesk)
   监控所在机器的CPU,IO,JVM等指标,简单分片概览
* [KOPF](https://github.com/lmenezes/elasticsearch-kopf) 
   查看集群gc回收磁盘性能, 分片情况, 简单操作api, 感觉该插件较Head更实用一些
* [Sql](https://github.com/NLPchina/elasticsearch-sql)
	可以通过sql进行聚合检索, 可以将sql语句翻译成ES的JSON检索语句
	
	
## ES集群优雅停止,启动

在一开始使用ES的时候, 都是通过 `kill <pid>` (不是Kill -9)来关闭ES实例. 但是每回重启后, 都会发现有很长时间的分片同步(即使没有手动删除数据等操作). 后来发现ES默认是开启自动分片均衡的. 那么如果想要我们在停止,启动某个ES实例后, 可以快速将集群状态变更为Green. 我们最好可以采用如下步骤进行:

* **暂停集群的分片自动均衡(在集群中任一台ES实例上执行都可以,只需要执行一次)**

	```
	curl -XPUT http://127.0.0.1:9200/_cluster/settings -d' 
	{ 
		"transient" : { 
			"cluster.routing.allocation.enable" : "none" 
		} 
	}'
	```
* **优雅停止你要升级或检测的ES实例(不到万不得已,绝对不要用Kill -9)**

	```
	curl -XPOST http://127.0.0.1:9200/_cluster/nodes/_local/_shutdown
	```

	备注: 在2.0版本之后将废弃该api
	
	```
	The _shutdown API has been removed without a replacement. Nodes should be managed via the operating system and the provided start/stop scripts.
	```
	
* **升级重启该节点，并确认该节点重新加入到了集群中**
* **重复上面的2,3步，操作其他集群内其他ES实例**
* **重新启动集群的分片自动均衡(在集群中任一ES实例中执行一次即可)**

	```
	curl -XPUT http://127.0.0.1:9200/_cluster/settings -d' 
	{ 
		"transient" : { 
			"cluster.routing.allocation.enable" : "all" 
		} 
	}'
	```
备注: **如果在集群中关闭自动均衡, 业务程序在插入数据时,不会自动创建索引, 需要预先创建好索引**

## ES集群JVM调优积累

在最开始, 我们使用的ES实例,没有涉及到很大的数据写入和检索(每天几百万数据). 当时我们配置ES的JVM(Xms=Xmx=8G)的垃圾回收器主要是CMS,具体配置如下:

```
# reduce the per-thread stack size
JAVA_OPTS="$JAVA_OPTS -Xss256k"

JAVA_OPTS="$JAVA_OPTS -XX:+UseParNewGC"
JAVA_OPTS="$JAVA_OPTS -XX:+UseConcMarkSweepGC"

JAVA_OPTS="$JAVA_OPTS -XX:CMSInitiatingOccupancyFraction=75"
JAVA_OPTS="$JAVA_OPTS -XX:+UseCMSInitiatingOccupancyOnly"
```

这个在小数据量,小内存时运行良好, 基本很少会出现Full gc.

后来我们的数据量为了一天1亿多, 为了加快检索, 我们将Xms,Xmx同时调整为32G. 并按天进行索引,后来发现随着索引数增加,写入数据量增加, Full GC已经影响到了我们的写入和检索(这个时候我们的ES集群是三个实例).我们决定将G1作为垃圾回收期,但是[官方并不推荐使用G1](https://www.elastic.co/guide/en/elasticsearch/guide/current/_don_8217_t_touch_these_settings.html#_garbage_collector), 我们还是想尝试一下换成G1, 并借此机会把JDK从1.7升级到1.8(升级很平滑), G1的具体配置如下: 

```
JAVA_OPTS="$JAVA_OPTS -XX:+UseG1GC "
#init_globals()末尾打印日志
JAVA_OPTS="$JAVA_OPTS -XX:+PrintFlagsFinal "
#打印gc引用
JAVA_OPTS="$JAVA_OPTS -XX:+PrintReferenceGC "
#输出虚拟机中GC的详细情况.
JAVA_OPTS="$JAVA_OPTS -verbose:gc "
JAVA_OPTS="$JAVA_OPTS -XX:+PrintGCDetails "
#Enables printing of time stamps at every GC. By default, this option is disabled.
JAVA_OPTS="$JAVA_OPTS -XX:+PrintGCTimeStamps "
#Enables printing of information about adaptive generation sizing. By default, this option is disabled.
JAVA_OPTS="$JAVA_OPTS -XX:+PrintAdaptiveSizePolicy "
# unlocks diagnostic JVM options
JAVA_OPTS="$JAVA_OPTS -XX:+UnlockDiagnosticVMOptions "
#to measure where the time is spent
JAVA_OPTS="$JAVA_OPTS -XX:+G1SummarizeConcMark "
#设置触发标记周期的 Java 堆占用率阈值。默认占用率是整个 Java 堆的 45%。
#JAVA_OPTS="$JAVA_OPTS -XX:InitiatingHeapOccupancyPercent=45 "
```

> TODO 之后会在这补充替换为G1后,ES集群的表现

## ES集群健康说明
在Elasticsearch集群中可以监控统计很多信息，其中最重要的就是：集群健康(cluster health)。它的 status 有 `green`、`yellow`、`red` 三种；

可以通过如下命令获取:

```
GET http://127.0.0.1:9200/_cluster/health
```

在一个没有索引的空集群中，它将返回如下信息：

```
{
   "cluster_name":          "elasticsearch",
   "status":                "green", <1>
   "timed_out":             false,
   "number_of_nodes":       1,
   "number_of_data_nodes":  1,
   "active_primary_shards": 0,
   "active_shards":         0,
   "relocating_shards":     0,
   "initializing_shards":   0,
   "unassigned_shards":     0
}
```

`status` 是我们最应该关注的字段。
`status` 可以告诉我们当前集群是否处于一个可用的状态。三种颜色分别代表：

**状态说明**
green	: 所有主分片和从分片都可用
yellow: 	所有主分片可用，但存在不可用的从分片
red:	存在不可用的主要分片


## ES配置文件详解

```
cluster.name: elasticsearch
#配置es的集群名称，默认是elasticsearch，es会自动发现在同一网段下的es，如果在同一网段下有多个集群，就可以用这个属性来区分不同的集群。

node.name: "Franz Kafka"
#节点名，默认随机指定一个name列表中名字，该列表在es的jar包中config文件夹里name.txt文件中，其中有很多作者添加的有趣名字。

node.master: true
#指定该节点是否有资格被选举成为node，默认是true，es是默认集群中的第一台机器为master，如果这台机挂了就会重新选举master。

node.data: true
#指定该节点是否存储索引数据，默认为true。

index.number_of_shards: 5
#设置默认索引分片个数，默认为5片。

index.number_of_replicas: 1
#设置默认索引副本个数，默认为1个副本。

path.conf: /path/to/conf
#设置配置文件的存储路径，默认是es根目录下的config文件夹。

path.data: /path/to/data
#设置索引数据的存储路径，默认是es根目录下的data文件夹，可以设置多个存储路径，用逗号隔开，例：
#path.data: /path/to/data1,/path/to/data2

path.work: /path/to/work
#设置临时文件的存储路径，默认是es根目录下的work文件夹。

path.logs: /path/to/logs
#设置日志文件的存储路径，默认是es根目录下的logs文件夹

path.plugins: /path/to/plugins
#设置插件的存放路径，默认是es根目录下的plugins文件夹

bootstrap.mlockall: true
#设置为true来锁住内存。因为当jvm开始swapping时es的效率会降低，所以要保证它不swap，可以把#ES_MIN_MEM和ES_MAX_MEM两个环境变量设置成同一个值，并且保证机器有足够的内存分配给es。同时也要#允许elasticsearch的进程可以锁住内存，linux下可以通过`ulimit -l unlimited`命令。

network.bind_host: 192.168.0.1
#设置绑定的ip地址，可以是ipv4或ipv6的，默认为0.0.0.0。


network.publish_host: 192.168.0.1
#设置其它节点和该节点交互的ip地址，如果不设置它会自动判断，值必须是个真实的ip地址。

network.host: 192.168.0.1
#这个参数是用来同时设置bind_host和publish_host上面两个参数。

transport.tcp.port: 9300
#设置节点间交互的tcp端口，默认是9300。

transport.tcp.compress: true
#设置是否压缩tcp传输时的数据，默认为false，不压缩。

http.port: 9200
#设置对外服务的http端口，默认为9200。

http.max_content_length: 100mb
#设置内容的最大容量，默认100mb

http.enabled: false
#是否使用http协议对外提供服务，默认为true，开启。

gateway.type: local
#gateway的类型，默认为local即为本地文件系统，可以设置为本地文件系统，分布式文件系统，hadoop的#HDFS，和amazon的s3服务器，其它文件系统的设置方法下次再详细说。

gateway.recover_after_nodes: 1
#设置集群中N个节点启动时进行数据恢复，默认为1。

gateway.recover_after_time: 5m
#设置初始化数据恢复进程的超时时间，默认是5分钟。

gateway.expected_nodes: 2
#设置这个集群中节点的数量，默认为2，一旦这N个节点启动，就会立即进行数据恢复。

cluster.routing.allocation.node_initial_primaries_recoveries: 4
#初始化数据恢复时，并发恢复线程的个数，默认为4。

cluster.routing.allocation.node_concurrent_recoveries: 2
#添加删除节点或负载均衡时并发恢复线程的个数，默认为4。

indices.recovery.max_size_per_sec: 0
#设置数据恢复时限制的带宽，如入100mb，默认为0，即无限制。

indices.recovery.concurrent_streams: 5
#设置这个参数来限制从其它分片恢复数据时最大同时打开并发流的个数，默认为5。

discovery.zen.minimum_master_nodes: 1
#设置这个参数来保证集群中的节点可以知道其它N个有master资格的节点。默认为1，对于大的集群来说，可以设置大一点的值（2-4）

discovery.zen.ping.timeout: 3s
#设置集群中自动发现其它节点时ping连接超时时间，默认为3秒，对于比较差的网络环境可以高点的值来防止自动发现时出错。

discovery.zen.ping.multicast.enabled: false
#设置是否打开多播发现节点，默认是true。

discovery.zen.ping.unicast.hosts: ["host1", "host2:port", "host3[portX-portY]"]
#设置集群中master节点的初始列表，可以通过这些节点来自动发现新加入集群的节点。
```

## ES使用中碰到的问题

* **rejected execution (queue capacity 50) on org.elasticsearch.action.support.replication.TransportShar** 

	该问题需要在配置文件中添加Thread pool参数.
	
	```
	threadpool:
    #这是批量插入时需要做的配置
     bulk:
        type: fixed
        # 设置的和cpu数量一致
        size: 24
        queue_size: 10000

    #检索时需要做的配置
    search:
        tyep: fixed
        # 设置的和cpu数量一致
        size: 24
        queue_size: 10000
	```
	整个数据将会被处理它的节点载入内存中，所以如果请求量很大的话，留给其他请求的内存空间将会很少。bulk应该有一个最佳的限度。超过这个限制后，性能不但不会提升反而可能会造成宕机。

	最佳的容量并不是一个确定的数值，它取决于你的硬件，你的文档大小以及复杂性，你的索引以及搜索的负载。幸运的是，这个平衡点 很容易确定：

	试着去批量索引越来越多的文档。当性能开始下降的时候，就说明你的数据量太大了。一般比较好初始数量级是1000到5000个文档，或者你的文档很大，你就可以试着减小队列。 有的时候看看批量请求的物理大小是很有帮助的。1000个1KB的文档和1000个1MB的文档的差距将会是天差地别的。比较好的初始批量容量是5-15MB。

* **同一天机器上不同的es实例, 同一个索引的主副分片被分在了同一台机器上**  
	
	在构建es集群的前期, 机器数量比较少, 但是机器配置还不错,可能会在一个机器上挂好几个硬盘, 开启多个es实例. 这样少量的机器,可以创建超过机器数量的es实例, 充分利用集群硬件性能,同时也具备更好的可拓展性. 
	
	一台机器上多个es实例, 可能会使同一个索引的主副分片被分在同一台机器上,这样就降低了这个索引分片的健壮性,一旦这台机器挂了, 这个索引分片上的数据就是不可用的.
	
	基于上述的描述, es配置文件中需要对所有es实例进行分组,以保证同一个索引的主副分片不在同一台集群中, 具体配置如下:
	
	```
	#通过node.group建立一个属性为group
	node.group: eagleye_19
	#引用属性group的值
	cluster.routing.allocation.awareness.attributes: group
	```
	
* **java客户端可以检索句子的方法**

	之前因为存储的内容是中英文混合的, 所以分词的时候采用最小力度分词, 这样如果要检索句子的时候, 不知道为什么, 检索不出来, 最后使用了, 如下方式可以直接检索句子了:
	
```
BoolFilterBuilder bqb = FilterBuilders.boolFilter();
if(vo.getQueryStr() != null && !vo.getQueryStr().trim().equals("")){    
bqb.must(FilterBuilders.queryFilter(QueryBuilders.queryStringQuery( 
ESReservedCharsUtil.removeReservedChars(vo.getQueryStr())).defaultField("body").defaultOperator(QueryStringQueryBuilder.Operator.AND)).cache(true));
// 该种方式不能检索句子
// for(String _s : vo.getQueryStr().split("\\s+")) {
//     bqb.must(FilterBuilders.termFilter("body", _s));
// }
}
```
	
----

	
## 其他

* **批量删除索引**	
	
	```
	#可用批量删除名字以xxx-log-2015.06开头的索引
	curl -XDELETE http://127.0.0.1:9201/xxx-log-2015.06*
	```
* **保证最大限度的使用内存而不引起OutOfMemory**
	设置es的缓存类型为Soft Reference，它的主要特点是据有较强的引用功能。只有当内存不够的时候，才进行回收这类内存，因此在内存足够的时候，它们通常不被回收。另外，这些引 用对象还能保证在Java抛出OutOfMemory 异常之前，被设置为null。它可以用于实现一些常用图片的缓存，实现Cache的功能.在es的配置文件中做如下修改:

	```
	index.cache.field.type: soft
	```
* **通过修改Template,缩小索引大小,减小系统内存使用**
	在ES的Template中做如下调整
	
	```
	"_source": {'enabled': false}, //如果不需要展示一些字段信息,只是用统计功能, 可以直接关闭
	"_all" : { "enabled" : false }, //不对所有字段进行自动索引,可以减小索引数量
	"pid": {
        //not_analyzed可以不分词, no:不索引, 去除index则会根据分词器进行分词并索引
	   	"index": "not_analyzed",
	   	"store": "false", //如果不需要存储, 不需要显示, 可以直接关闭
	   	//设置了doc_values为true之后，字段创建时，就是用磁盘存储fielddata而不是内存中, fielddata在lucene中就是正排索引, 可以非常快速的帮助我们做数据聚合,排序使用, 倒排索引帮助我们快速检索. 正常如果doc_values为false, 正排索引是放在内存中, 比较消化内存资源. 如果设为true则会放在磁盘中.另外,有一个设置 indices.fielddata.cache.expire 可以设置fielddata 进入内存中的数据多久自动过期。注意，因为 ES 的 fielddata 本身是一种数据结构，而不是简单的缓存，所以过期删除 fielddata 是一个非常消耗资源的操作。ES 官方在文档中特意说明，这个参数绝对绝对不要设置！
	   	"doc_values": true
	    "type": "string"
	 },
	```
* **加快调整分片和副本时的恢复进度**
	副本配置和分片配置不一样，是可以随时调整的。有些较大的索引，甚至可以在做 optimize 前，先把副本全部取消掉，等 optimize 完后，再重新开启副本，节约单个 segment 的重复归并消耗。

* **ES 1.X 的版本升级非常平滑,向前兼容做的不错, 但是 从 1.x到2.x会有一些改动**	
* **index设置合理的刷新时间**
	建立的索引，不会立马查到，这是为什么elasticsearch为near-real-time的原因
需要配置index.refresh_interval参数，默认是1s。在template中设置如下:

	```
	{
  "logstash" : {
    "order" : 0,
    "template" : "eagleye-log*",
    "settings" : {
      "index.refresh_interval" : "2s"
    },
    "mappings" : {
      "log" : {
        "_all" : { "enabled" : false },
        ... ...
	```
	
* **字段缓存(Field cache)设置**

	当对字段排序或者对字段做聚合（如 facet ）时，字段缓存（ Field cache ）非常重要。 Es 会将这些待排序或者聚合字段都加载到内存，以提高对这些字段的快速访问。注意，将字段都加载到内存是非常耗费资源的，所以，你应该保证 field cache 足够大，以足以将所有的结果都缓存起来，下次排序或 facet 时不用再次从磁盘进行加载。
	
	可以通过设置 indices.fielddata.cache.size 为具体的大小，比如 2GB ，或者可用内存的百分比，比如 40% 。请注意，这个属性是 node 级别 ( 不是 index 级别的 ). 当这个缓存不够用时，为了跟新的缓存对象腾出空间，原来缓存的字段会被挤出来，这会导致系统性能下降。所以，请保证这个值足够大，能够满足业务需求。另外，如果你没有设置这个值， es 默认缓存可以无限大。所以，在生产环境注意要设置这个值。

	```
	indices.fielddata.cache.size:  20% 
	```
	
* **查看集群健康状况的命令**

	```
	curl -s localhost:9201/_cat/indices?v | sort -k3  | more
	```

* **减小es集群脑裂的配置优化**
	* master尽量不作为data节点
	* discovery.zen.ping_timeout（默认值是3秒）修改,默认情况下，一个节点会认为，如果master节点在3秒之内没有应答，那么这个节点就是死掉了，而增加这个值，会增加节点等待响应的时间，从一定程度上会减少误判。
	* discovery.zen.minimum_master_nodes（默认是1）：这个参数控制的是，一个节点需要看到的具有master节点资格的最小数量，然后才能在集群中做操作。官方的推荐值是(N/2)+1，其中N是具有master资格的节点的数量（我们的情况是3，因此这个参数设置为2，但对于只有2个节点的情况，设置为2就有些问题了，一个节点DOWN掉后，你肯定连不上2台服务器了，这点需要注意）。
	* 加快master发现的速度,可以将data节点的默认的master发现方式有multicast修改为unicast,并指定具体的master地址
		
		```
		discovery.zen.ping.multicast.enabled: false
		discovery.zen.ping.unicast.hosts: ["master1", "master2", "master3"]  
		```

* **有次出现某些分片长期处于UNASSIGNED状态，我们就可以手动分配分片到指定节点上**
	默认情况下只允许手动分配副本分片，所以如果是主分片故障，需要单独加一个allow_primary选项, 注意，如果是历史数据的话，请提前确认一下哪个节点上保留有这个分片的实际目录，且目录大小最大。然后手动分配到这个节点上。以此减少数据丢失。(我们在测试环境发现有有一天早上六点有几个分片是UNASSIGNED状态, 采用下面的命令可以使集群重回green状态)：
	  
	  ```
	  curl -XPOST 127.0.0.1:9200/_cluster/reroute -d '{
		  "commands": [
		    {
		      "allocate": {
		        "index": "eagleye_bigindex_2015.12.29",
		        "shard": 1,
		        "node": "eagleye_es_236",
		        "allow_primary": true
		      }
		    }
		  ]
		}'
	  ```
	  
* **因为负载过高，磁盘利用率过高，服务器下线，更换磁盘等原因，可以会需要从节点上移走部分分片**

		```
		curl -XPOST 127.0.0.1:9200/_cluster/reroute -d '{
		  "commands": [
		    {
		      "move": {
		        "index": "eagleye_bigindex_2015.12.29",
		        "shard": 1,
		        "from_node": "eagleye_es_235",
		        "to_node": "eagleye_es_237"
		      }
		    }
		  ]
		}'
		```


<div style="margin-top: 15px; font-size: 11px;color: #cc0000;"><p align="center"><strong>（转载本站文章请注明作者和出处 <a href="http://siye1982.github.io">Panda</a>）</strong></p></div>

