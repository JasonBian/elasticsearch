#Elasticsearch培训教程

##认识Apache Lucene
为了更深入地理解Elasticsearch的工作原理，特别是索引和查询这两个过程，理解Lucene的工作原理至关重要。本质上，Elasticsearch是用Lucene来实现索引的查询功能的。

###Lucene架构基本概念
- Document:  它是在索引和搜索过程中数据的主要表现形式，或者称“载体”，承载着我们索引和搜索的数据,它由一个或者多个域(Field)组成。
- Field:   它是Document的组成部分，由两部分组成，名称(name)和值(value)。
- Term:  它是搜索的基本单位，其表现形式为文本中的一个词。
- Token:  它是单个Term在所属Field中文本的呈现形式，包含了Term内容、Term类型、Term在文本中的起始及偏移位置。
	

---
Apache Lucene把所有的信息都写入到一个称为**倒排索引**的数据结构中。这种数据结构把索引中的每个Term与相应的Document映射起来，这与关系型数据库存储数据的方式有很大的不同。读者可以把倒排索引想象成这样的一种数据结构：数据以Term为导向，而不是以Document为导向。下面看看一个简单的倒排索引是什么样的，假定我们的Document只有title域(Field)被编入索引，Document如下：

- ElasticSearch Servier (document 1)
- Mastering ElasticSearch (document 2)
- Apache Solr 4 Cookbook (document 3)

所以索引(以一种直观的形式)展现如下：

Term | count | Docs
---|---
4|1|<3>
Apache|1|<3>
Cookbook|1|<3>
ElasticSearch|2|<1> <2>
Mastering|1|<1>
Server|1|<1>
Solr|1|<3>

正如所看到的那样，每个词都指向它所在的文档号(Document Number/Document ID)。这样的存储方式使得高效的信息检索成为可能，比如基于词的检索(term-based query)。此外，每个词映射着一个数值(Count)，它代表着Term在文档集中出现的频繁程度。

当然，Lucene创建的真实索引远比上文复杂和先进。这是因为在Lucene中，词向量(由单独的一个Field形成的小型倒排索引，通过它能够获取这个特殊Field的所有Token信息)可以存储；所有Field的原始信息可以存储；删除Document的标记信息可以存储……。核心在于了解数据的组织方式，而非存储细节。

每个索引被分成了多个段(Segment)，段具有一次写入，多次读取的特点。只要形成了，段就无法被修改。例如：被删除文档的信息被存储到一个单独的文件，但是其它的段文件并没有被修改。


需要注意的是，多个段是可以合并的，这个合并的过程称为segments merge。经过强制合并或者Lucene的合并策略触发的合并操作后，原来的多个段就会被Lucene创建的更大的一个段所代替了。很显然，段合并的过程是一个I/O密集型的任务。这个过程会清理一些信息，比如会删除.del文件。除了精减文件数量，段合并还能够提高搜索的效率，毕竟同样的信息，在一个段中读取会比在多个段中读取要快得多。但是，由于段合并是I/O密集型任务，建议不好强制合并，小心地配置好合并策略就可以了。

### 分析你的文本
问题到这里就变得稍微复杂了一些。传入到Document中的数据是如何转变成倒排索引的？查询语句是如何转换成一个个Term使高效率文本搜索变得可行？这种转换数据的过程就称为文本分析(analysis)


文本分析工作由analyzer组件负责。analyzer由一个分词器(tokenizer)和0个或者多个过滤器(filter)组成,也可能会有0个或者多个字符映射器(character mappers)组成。

Lucene中的tokenizer用来把文本拆分成一个个的Token。Token包含了比较多的信息，比如Term在文本的中的位置及Term原始文本，以及Term的长度。文本经过tokenizer处理后的结果称为token stream。token stream其实就是一个个Token的顺序排列。token stream将等待着filter来处理。


除了tokenizer外，Lucene的另一个重要组成部分就是filter链，filter链将用来处理Token Stream中的每一个token。这些处理方式包括删除Token,改变Token，甚至添加新的Token。Lucene中内置了许多filter，读者也可以轻松地自己实现一个filter。有如下内置的filter：

- Lowercase filter：把所有token中的字符都变成小写
- ASCII folding filter：去除tonken中非ASCII码的部分
- Synonyms filter：根据同义词替换规则替换相应的token
- Multiple language-stemming filters：把Token(实际上是Token的文本内容)转化成词根或者词干的形式

所以通过Filter可以让analyzer有几乎无限的处理能力：因为新的需求添加新的Filter就可以了。

### 索引和查询

在我们用Lucene实现搜索功能时，也许会有读者不明觉历：上述的原理是如何对索引过程和搜索过程产生影响？

索引过程：Lucene用用户指定好的analyzer解析用户添加的Document。当然Document中不同的Field可以指定不同的analyzer。如果用户的Document中有title和description两个Field，那么这两个Field可以指定不同的analyzer。

搜索过程：用户的输入查询语句将被选定的查询解析器(query parser)所解析,生成多个Query对象。当然用户也可以选择不解析查询语句，使查询语句保留原始的状态。在ElasticSearch中，有的Query对象会被解析(analyzed)，有的不会，比如：前缀查询(prefix query)就不会被解析，精确匹配查询(match query)就会被解析。对用户来说，理解这一点至关重要。

对于索引过程和搜索过程的数据解析这一环节，我们需要把握的重点在于：倒排索引中词应该和查询语句中的词正确匹配。如果无法匹配，那么Lucene也不会返回我们喜闻乐见的结果。举个例子：如果在索引阶段对文本进行了转小写(lowercasing)和转变成词根形式(stemming)处理，那么查询语句也必须进行相同的处理，不然就搜索结果就会是竹篮打水——一场空。

##认识Elasticsearch
Elasticsearch是建立在全文搜索引擎库Apache Lucene之上的开源搜索引擎。截止目前，Lucene可以说是最先进的、高性能的、全功能的搜索引擎库。

但是Lucene仅仅只是一个库，要充分利用Lucene的力量，你可能需要运用java将你的应用程序直接集成Lucene，更糟糕的是你可能会需要某种程度的信息检索，了解它是如何工作的，Lucene还是非常复杂的。

Elasticsearch是用java编写的，并使用Lucene内部所有的索引和搜索机制，Elasticsearch隐藏了Lucene背后复杂的索引过程，使得全文搜索变得简单。
然而Elasticsearch远不止仅仅只是Lucene的全文搜索，它可以简单地描述为：
- 	一个分布式文档存储，每一个field都是可以索引和搜索的
- 	分布式搜索引擎，支持实时分析
- 	服务器支持水平扩展，支持PB级别的结构化和非结构化数据

### Elasticsearch基本概念
#### 索引(Index)
ElasticSearch把数据存放到一个或者多个索引(indices)中。如果用关系型数据库模型对比，索引(index)的地位与数据库实例(database)相当。索引存放和读取的基本单元是文档(Document)。我们也一再强调，ElasticSearch内部用Apache Lucene实现索引中数据的读写。读者应该清楚的是：在ElasticSearch中被视为单独的一个索引(index)，在Lucene中可能不止一个。这是因为在分布式体系中，ElasticSearch会用到分片(shards)和备份(replicas)机制将一个索引(index)存储多份。
#### 文档(Document)
在ElasticSearch的世界中，文档(Document)是主要的存在实体(在Lucene中也是如此)。所有的ElasticSearch应用需求到最后都可以统一建模成一个检索模型：检索相关文档。文档(Document)由一个或者多个域(Field)组成，每个域(Field)由一个域名(此域名非彼域名)和一个或者多个值组成(有多个值的值称为多值域(multi-valued))。在ElasticSeach中，每个文档(Document)都可能会有不同的域(Field)集合；也就是说文档(Document)是没有固定的模式和统一的结构。文档(Document)之间保持结构的相似性即可(Lucene中的文档(Document)也秉持着相同的规定)。实际上，ElasticSearch中的文档(Document)就是Lucene中的文档(Document)。从客户端的角度来看，文档(Document)就是一个JSON对象。所有的文档(Document)在存储之前都必须经过分析(analyze)流程。用户可以配置输入文本分解成Token的方式；哪些Token应该被过滤掉；或者其它的的处理流程，比如去除HTML标签。此外，ElasticSearch提供的各种特性，比如排序的相关信息。保存上述的配置信息，这就是参数映射(Mapping)在ElasticSearch中扮演的角色。尽管ElasticSearch可以根据域的值自动识别域的类型(field type)，在生产应用中，都是需要自己配置这些信息以避免一些奇的问题发生。要保证应用的可控性。
#### 文档类型(Type)
每个文档在ElasticSearch中都必须设定它的类型。文档类型使得同一个索引中在存储结构不同文档时，只需要依据文档类型就可以找到对应的参数映射(Mapping)信息，方便文档的存取。
#### 节点(Node)
单独一个ElasticSearch服务器实例称为一个节点。对于许多应用场景来说，部署一个单节点的ElasticSearch服务器就足够了。但是考虑到容错性和数据过载，配置多节点的ElasticSearch集群是明智的选择。
#### 集群(Cluster)
集群是多个ElasticSearch节点的集合。这些节点齐心协力应对单个节点无法处理的搜索需求和数据存储需求。集群同时也是应对由于部分机器(节点)运行中断或者升级导致无法提供服务这一问题的利器。ElasticSearch提供的集群各个节点几乎是无缝连接(所谓无缝连接，即集群对外而言是一个整体，增加一个节点或者去掉一个节点对用户而言是透明的<个人理解，仅供参考>)。在ElasticSearch中配置一个集群非常简单，在我们看来，这是在与同类产品中竞争所体现出的最大优势。
#### 分片索引(Shard)
前面已经提到，集群能够存储超出单机容量的信息。为了实现这种需求，ElasticSearch把数据分发到多个存储Lucene索引的物理机上。这些Lucene索引称为分片索引，这个分发的过程称为索引分片(Sharding)。在ElasticSearch集群中，索引分片(Sharding)是自动完成的，而且所有分片索引(Shard)是作为一个整体呈现给用户的。需要注意的是，尽管索引分片这个过程是自动的，但是在应用中需要事先调整好参数。因为集群中分片的数量需要在索引创建前配置好，而且服务器启动后是无法修改的，至少目前无法修改。
#### 索引副本(Replica)
通过索引分片机制(Sharding)可以向ElasticSearch集群中导入超过单机容量的数据，客户端操作任意一个节点即可实现对集群数据的读写操作。当集群负载增长，用户搜索请求阻塞在单个节点上时，通过索引副本(Replica)机制就可以解决这个问题。索引副本(Replica)机制的的思路很简单：为索引分片创建一份新的拷贝，它可以像原来的主分片一样处理用户搜索请求。同时也顺便保证了数据的安全性。即如果主分片数据丢失，ElasticSearch通过索引副本使得数据不丢失。索引副本可以随时添加或者删除，所以用户可以在需要的时候动态调整其数量。
#### 时间之门(Gateway)
在运行的过程中，ElasticSearch会收集集群的状态、索引的参数等信息。这些数据被存储在Gateway中。

## Elasticsearch背后的核心理念
ElasticSearch是构建在极少数的几个概念之上的。ElasticSearch的开发团队希望它能够快速上手，可扩展性强。而且这些核心特性体现在ElasticSearch的各个方面。从架构的角度来看，这些主要特性是：
- 开箱即用。安装好ElasticSearch后，所有参数的默认值都自动进行了比较合理的设置，基本不需要额外的调整。包括内置的发现机制(比如Field类型的自动匹配)和自动化参数配置。
- 天生集群。ElasticSearch默认工作在集群模式下。节点都将视为集群的一部分，而且在启动的过程中自动连接到集群中。
- 自动容错。ElasticSearch通过P2P网络进行通信，这种工作方式消除了单点故障。节点自动连接到集群中的其它机器，自动进行数据交换及以节点之间相互监控。索引分片
- 扩展性强。无论是处理能力和数据容量上都可以通过一种简单的方式实现扩展，即增添新的节点。
- 近实时搜索和版本控制。由于ElasticSearch天生支持分布式，所以延迟和不同节点上数据的短暂性不一致无可避免。ElasticSearch通过版本控制(versioning)的机制尽量减少问题的出现。

## Elasticsearch入门
### 安装Elasticsearch
安装Elasticsearch只有一个要求，就是需要依赖JDK。
Elasticsearch最新版本可以从 http://www.elastic.co/downloads/elasticsearch 下载；

下载之后解压安装包：

unzip elasticsearch-$VERSION.zip

### 运行Elasticsearch
cd elasticsearch-$VERSION
./bin/elasticsearch

如果你想让它在后台保持运行的话可以在命令后面再加一个 -d

启动elasticsearch服务后你就可以使用浏览器打开http://localhost:9200,在浏览器看到如下内容：

```
{
  "status" : 200,
  "name" : "Aegis",
  "cluster_name" : "elasticsearch_bianzexin",
  "version" : {
    "number" : "1.7.1",
    "build_hash" : "b88f43fc40b0bcd7f173a1f9ee2e97816de80b19",
    "build_timestamp" : "2015-07-29T09:54:16Z",
    "build_snapshot" : false,
    "lucene_version" : "4.10.4"
  },
  "tagline" : "You Know, for Search"
}
```
这就说明你的Elasticsearch已经正常work了，这时你就可以尽情地完虐Elasticsearch了。

另外在服务器上也可以通过servicewrapper这个es插件来启动，它支持通过参数，指定是在后台或前台运行es，并支持启动、停止、重启es服务。
先从http://github.com/elasticsearch/elasticsearch-servicewrapper 下载servicewrapper插件，解压之后将service文件夹放置es的bin目录下。具体命令集合如下：

```
bin/service/elasticsearch +
console 在前台运行es
start   在后台运行es
stop    停止es
install 使es作为服务在服务器启动时启动
remove  取消启动时自动启动
restart 重启es
```
### 初识Elasticsearch配置文件
Elasticsearch的config文件夹下有两个配置文件:elasticsearch.yml和logging.yml,第一个是es的基本配置文件，第二个是日志配置文件，es也是使用log4j来记录日志的，所以logging.yml里的设置按普通的log4j配置文件来设置就行了。下面对elasticsearch.yml这个文件做一个比较详细的说明。
- cluster.name: elasticsearch

配置es的集群名称，默认是elasticsearch，es会自动发现在同一网段下的es，如果在同一个网段下有多个集群，就可以通过这个属性来区分不同的集群。

- node.name: "Franz Kafka"

节点名，默认随机指定一个name列表中的名字，该列表在es的jar包种的config文件夹里name.txt文件中。

- node.master: true

指定该节点是否有资格被选举为master，默认是true，es是默认集群中的第一台机器为master，如果这台机器挂了就会重新选举master。

- node.data: true

指定该节点是否存储索引数据，默认是true。

- index.number_of_shards: 5

设置默认索引分片个数，默认为5片。

- index.number_of_replicas: 1

设置默认索引副本个数，默认为1个副本。

- path.conf: /path/to/conf

设置配置文件的存储路径，默认是es根目录下地config文件夹。

- path.data: /path/to/data

设置索引数据的存储路径，默认是es根目录下地data文件夹，可以设置多个存储路径，用逗号隔开，例如：path.data: /path/to/data1,/path/to/data2

- path.work: /path/to/work

设置临时文件的存储路径，默认是es根目录下的work文件夹。

- path.logs: /path/to/logs

设置日志文件的存储路径，默认是es根目录下的logs文件夹

- path.plugins: /path/to/plugins

设置插件的存放路径，默认是es根目录下的plugins文件夹

- bootstrap.mlockall: true

设置为true来锁住内存。因为当jvm开始swapping时es的效率 会降低，所以要保证它不swap，可以把ES_MIN_MEM和ES_MAX_MEM两个环境变量设置成同一个值，并且保证机器有足够的内存分配给es。 同时也要允许elasticsearch的进程可以锁住内存，linux下可以通过`ulimit -l unlimited`命令。

- network.bind_host: 192.168.0.1

设置绑定的ip地址，可以是ipv4或ipv6的，默认为0.0.0.0。

- network.publish_host: 192.168.0.1

设置其它节点和该节点交互的ip地址，如果不设置它会自动判断，值必须是个真实的ip地址。

- network.host: 192.168.0.1

这个参数是用来同时设置bind_host和publish_host上面两个参数。

- transport.tcp.port: 9300

设置节点间交互的tcp端口，默认是9300。

- transport.tcp.compress: true

设置是否压缩tcp传输时的数据，默认为false，不压缩。

- http.port: 9200

设置对外服务的http端口，默认为9200。

- http.max_content_length: 100mb

设置内容的最大容量，默认100mb

- http.enabled: false

是否使用http协议对外提供服务，默认为true，开启。

- gateway.type: local

gateway的类型，默认为local即为本地文件系统，可以设置为本地文件系统，分布式文件系统，hadoop的HDFS，和amazon的s3服务器。

- gateway.recover_after_nodes: 1

设置集群中N个节点启动时进行数据恢复，默认为1。

- gateway.recover_after_time: 5m

设置初始化数据恢复进程的超时时间，默认是5分钟。

- gateway.expected_nodes: 2

设置这个集群中节点的数量，默认为2，一旦这N个节点启动，就会立即进行数据恢复。

- cluster.routing.allocation.node_initial_primaries_recoveries: 4

初始化数据恢复时，并发恢复线程的个数，默认为4。

- cluster.routing.allocation.node_concurrent_recoveries: 2

添加删除节点或负载均衡时并发恢复线程的个数，默认为4。

- indices.recovery.max_size_per_sec: 0

设置数据恢复时限制的带宽，如入100mb，默认为0，即无限制。

- indices.recovery.concurrent_streams: 5

设置这个参数来限制从其它分片恢复数据时最大同时打开并发流的个数，默认为5。

- discovery.zen.minimum_master_nodes: 1

设置这个参数来保证集群中的节点可以知道其它N个有master资格的节点。默认为1，对于大的集群来说，可以设置大一点的值（2-4）

- discovery.zen.ping.timeout: 3s

设置集群中自动发现其它节点时ping连接超时时间，默认为3秒，对于比较差的网络环境可以高点的值来防止自动发现时出错。

- discovery.zen.ping.multicast.enabled: false

设置是否打开多播发现节点，默认是true。

- discovery.zen.ping.unicast.hosts: ["host1:port", "host2:port"]

设置集群中master节点的初始列表，可以通过这些节点来自动发现新加入集群的节点。

- 下面是一些查询时的慢日志参数设置


```
index.search.slowlog.level: TRACE
index.search.slowlog.threshold.query.warn: 10s
index.search.slowlog.threshold.query.info: 5s
index.search.slowlog.threshold.query.debug: 2s
index.search.slowlog.threshold.query.trace: 500ms

index.search.slowlog.threshold.fetch.warn: 1s
index.search.slowlog.threshold.fetch.info: 800ms
index.search.slowlog.threshold.fetch.debug:500ms
index.search.slowlog.threshold.fetch.trace: 200ms
```

##索引你的数据

前面我们已经讲述了如何部署Elasticsearch集群并使Elasticsearch集群正常运行。那接下来我们就来讲讲如何通过Elasticsearch REST API来索引数据、删除数据以及检索数据。

```
Elasticsearch官方提供了很多种语言的客户端，详见https://www.elastic.co/guide/index.html

```
###创建索引
当我们在Elasticsearch集群创建我们的第一个document的时候，我们不需要完全关心整个创建索引的过程。我们仅仅只需要使用如下命令即可：

```
curl -XPUT http://localhost:9200/blog/
```

可以看到我们使用命令行curl来与Elasticsearch通信，通过9200端口由HTTP向Elasticsearch的RESTful API传送json。

如果上述脚本中的index不存在，Elasticsearch会为我们自动创建这个jindex。这样我们就仅仅告诉Elasticsearch我们只想创建一个名为blog的index。如果一切正常，你可以看到Elasticsearch会返回如下响应：

```
{"acknowledged":true}
```
###配置新创建的index
我们手动创建的index同样需要我们去设定一些配置参数，比如分片数、副本数，例如：

```
curl -XPUT http://localhost:9200/blog/ -d '{
    "settings" : {
        "number_of_shards" : 1,
        "number_of_replicas" : 2
    }
}'
```
这样命令执行后会创建一个只有一个分片，2份副本的名为blog的index，事实上这样一共创建3份物理的Lucene索引。
另外其他的一些参数也可以通过这样的方式来进行设置。

这样，我们已经拥有我们自己的全新的index了。但是这里有一个问题，我们忘记提供描述index结构的mappings。那我们可以怎么做呢？对于我们完全没有任何数据的index，我们可以简单地delete the index。我们只需要执行如下命令：

```
curl –XDELETE http://localhost:9200/posts

```
和先前执行的结果一样，你可以看到：

```
{"acknowledged":true}
```
###在index中创建新的Document

```
curl -XPUT http://localhost:9200/blog/article/1 -d '{"title": "New
version of Elasticsearch released!", "content": "...", "tags":
["announce", "elasticsearch", "release"] }'
```

###Mappings 配置
如果你使用过SQL数据库，你就会知道在我们往数据库插入数据前，你先要创建schema用来描述你的数据到底是什么样的。尽管Elasticsearch是schema-less的搜索引擎，但我们还是可以按照我们自己的方式来定义data structure。

#### Index structure mapping
schema mapping是用来定义索引结构的，其实无非就是设置document包含哪些field，然后对每一个field个性化的设置类型、是否存储，以及设置索引分析器和查询使用的分析器。

#### 剖析mapping
一个mapping由一个或多个analyzer组成， 一个analyzer又由一个或多个filter组成的。当ES索引文档的时候，它把字段中的内容传递给相应的analyzer，analyzer再传递给各自的filters。

filter的功能很容易理解：一个filter就是一个转换数据的方法， 输入一个字符串，这个方法返回另一个字符串，比如一个将字符串转为小写的方法就是一个filter很好的例子。

一个analyzer由一组顺序排列的filter组成，执行分析的过程就是按顺序一个filter一个filter依次调用， ES存储和索引最后得到的结果。

总结来说， mapping的作用就是执行一系列的指令将输入的数据转成可搜索的索引项。

####Core types
Elasticsearch中每个filed的type都可以指定，Elasticsearch包含的core types包含以下几种：

- String
- Number
- Date
- Boolean
- Binary

####公共属性
每种Elasticsearch的type都会有不同的属性，但有一些通用的属性是所有type都具备的。

- index_name:

This defines the name of the field that will be stored in theindex. If this is not defined, the name will be set to the name of the objectthat the field is defined with.
- index:
This can take the values analyzed and no. Also, for string-basedfields, it can also be set to not_analyzed. If set to analyzed, the field willbe indexed and thus searchable. If set to no, you won't be able to search onsuch a field. The default value is analyzed. In the case of string-based fields,there is an additional option, not_analyzed. This, when set, will mean thatthe field will be indexed but not analyzed. So, the field is written in theindex as it was sent to Elasticsearch and only a perfect match will be countedduring a search. Setting the index property to no will result in the disablingof the include_in_all property of such a field. 
- store: 
This can take the values yes and no and specifies if the original valueof the field should be written into the index. The default value is no, whichmeans that you can't return that field in the results (although, if you use the_source field, you can return the value even if it is not stored), but if you haveit indexed, you can still search the data on the basis of it.
- boost:The default value of this attribute is 1. Basically, it defines howimportant the field is inside the document; the higher the boost, the moreimportant the values in the field.
- null_value:This attribute specifies a value that should be written into theindex in case that field is not a part of an indexed document. The defaultbehavior will just omit that field.
- copy_to:

This attribute specifies a field to which all field values will be copied.

- include_in_all:

This attribute specifies if the field should be includedin the _all field. By default, if the _all field is used, all the fields will beincluded in it.

#### 默认analyzer
```
mappings: {  
    item: {  
        properties: {  
            description: {  
                type: string  
            }  
            name: {  
                type: string  
            }  
        }  
    }  
}
```
ES猜测description字段是string类型，于是默认创建一个string类型的mapping，它使用默认的全局analyzer， 默认的analyzer是标准analyzer, 这个标准analyzer有三个filter：token filter, lowercase filter和stop token filter。
我们可以在做查询的时候键入_analyze关键字查看分析的过程。使用以下指令查看description字段的转换过程：

```
curl -X GET "http://localhost:9200/test/_analyze?analyzer=standard&pretty=true" -d "A Pretty cool guy."  
   
{  
  "tokens" : [ {  
    "token" : "pretty",  
    "start_offset" : 2,  
    "end_offset" : 8,  
    "type" : "<ALPHANUM>",  
    "position" : 2  
  }, {  
    "token" : "cool",  
    "start_offset" : 9,  
    "end_offset" : 13,  
    "type" : "<ALPHANUM>",  
    "position" : 3  
  }, {  
    "token" : "guy",  
    "start_offset" : 14,  
    "end_offset" : 17,  
    "type" : "<ALPHANUM>",  
    "position" : 4  
  } ]
```
看看以单词a来搜索的结果：

```
$ curl -X GET "http://localhost:9200/test/_search?pretty=true" -d '{  
    "query" : {  
        "text" : { "description": "a" }  
    }  
}'  
   
{  
  "took" : 29,  
  "timed_out" : false,  
  "_shards" : {  
    "total" : 5,  
    "successful" : 5,  
    "failed" : 0  
  },  
  "hits" : {  
    "total" : 0,  
    "max_score" : null,  
    "hits" : [ ]  
  }  
}
```
#### 配置mapping方式
有两种添加mapping的方法，一种是定义在配置文件中，一种是运行是手动提交。

- 配置文件修改方式

把[mapping名].json文件放到config/mappings/[索引名]目录下，这个目录要自己创建，一个mapping和一个索引对应，你也可以定义一个默认的mapping，把自己定义的default-mapping.json放到config目录下就行

- 通过Java API手动提交mapping方式


```
XContentBuilder mapping = XContentFactory.jsonBuilder()
	.startObject().startObject("userindex")
	.startObject("_source").field("enabled", "false").endObject()
	.startObject("properties")
	.startObject("appId").field("type","string").field("index", 		"not_analyzed").endObject()
	.startObject("cell").field("type","string").field("index", 		"not_analyzed").endObject()
	.startObject("phoneType").field("type","string").field("index", 		"not_analyzed").endObject()
	.startObject("province").field("type","string").field("index", 		"not_analyzed").endObject()
	.startObject("tags").field("type","string").field("index_analyzer", 			"whitespace").field("search_analyzer", "whitespace").endObject()
	.startObject("city").field("type","string").field("index", 		"not_analyzed").endObject()
	.startObject("sex").field("type","double").field("index", 		"analyzed").endObject()
	.startObject("price").field("type","double").field("index", 		"analyzed").endObject()
	.startObject("usertype").field("type","string").field("index_analyzer", 		"whitespace").field("search_analyzer", "whitespace").endObject()
	.startObject("mobiler").field("type","string").field("index", 		"analyzed").endObject()
	.endObject().endObject().endObject();
PutMappingRequest mappingRequest = Requests.putMappingRequest("userindex")
	.type("Aaaa").source(mapping);  
client.admin().indices().putMapping(mappingRequest).actionGet();
```
#### Elasticsearch mapping示例

```
{
	"_default_":{
		"properties":{
			"appId":{
				"type":"string",
				"index":"not_analyzed" 
			},
			"cell":{
				"type":"string",
				"index":"not_analyzed" 
			},
			"phoneType":{
				"type":"string",
				"index":"not_analyzed" 
			},
			"province":{
				"type":"string",
				"index":"not_analyzed" 
			},
			"tags":{
				"type":"string",
				"index_analyzer":"whitespace",
				"search_analyzer":"whitespace" 
			},
			"city":{
				"type":"string",
				"index":"not_analyzed"
			},
			"sex":{
				"type":"double",
				"index":"analyzed"
			},
			"price":{
				"type":"double",
				"index":"analyzed"
			},
			"usertype":{
				"type":"string",
				"index_analyzer":"whitespace",
				"search_analyzer":"whitespace"
			},
			"mobiler":{
				"type":"string",
				"index":"analyzed"
			},
			"userTag":{
				"type":"string",
				"index_analyzer":"whitespace",
				"search_analyzer":"whitespace"
			}
		}
	}
}
```

##搜索你的数据
前面我们讲了如何通过RESTful API创建索引，那么如何搜索已经存在的索引数据呢？

###搜索API
Elasticsearch提供了两种基本的方式进行搜索：一种是通过REST请求URI传递搜索参数，另一种是通过REST请求体传递参数。搜索的REST API是”_search”,下面的命令是要搜索bank中的所有数据:

```
curl ‘localhost:9200/userindex/_search?q=*&pretty’
```
响应的结果(部分)如下：

```
{
	"took": 663,
	"timed_out": false,
	"_shards": {
		"total": 16,
		"successful": 16,
		"failed": 0
	},
	"hits": {
		"total": 307843698,
		"max_score": 1,
		"hits": [
			{
				"_index": "userindex",
				"_type": "951",
				"_id": "95131620",
				"_score": 1,
				"_source": {
					"uid": 95131620,
					"cell": "1725895131620",
					"appId": "951",
					"phoneType": "ANDROID",
					"province": "北京",
					"tags": "1 2 3 4 5”,
					"city": "11000000",
					"sex": 70,
					"price": 1000,
					"usertype": "10001",
					"mobiler": "1",
					"userTag": ""
				}
			}
		]
	}
}
```

其中，部分字段的含义如下：

took：花费的时间

time_out：显示是否超时

_shards：搜索了多少分片，成功/失败搜索的分片数量

hits：搜索命中的结果

hits.total：命中的数量

_score和max_score：匹配得分

第二种搜索方式，使用请求体：

```
curl –XPOST ‘localhost:9200/bank/_search?pretty’ –d ‘
{
    “query”: {“match_all”: {} }
}’
```

###查询简介

对上面的查询语句，”query”部分表示要进行查询操作，而”match_all”表示得到索引中的全部文档。除了”query”参数，还可以增加其他参数，比如还是要搜索全部文档，不过只返回第一个文档：

```
curl –XPOST ‘localhost:9200/bank/_search?pretty’ –d ‘
{
      “query”: { “match_all”: {} },
      “size”: 1
}’
```

如果”size”未特别说明，默认返回10条结果。下面的例子返回文档11到20：

```
curl –XPOST ‘localhost:9200/bank/_search?pretty’ –d ‘
{
       “query”: { “match_all”: {} },
       “from”: 10,
       “size”: 1
}’
```

当对搜索结果分页时，这个功能非常有用。下面的例子返回10条结果(默认size)并根据balance降序排序：

```
curl –XPOST ‘localhost:9200/bank/_search?pretty’ –d ’
{
       “query”: {“match_all”: {} },
       “sort”: {“balance”: {“orde”: “desc”}}
}’
```

###查询操作

之前的查询都将源数据全部返回，现在介绍只让我们想要的字段返回，比如下面的例子只返回两个字段”account_number”和”balance”：

```
curl –XPOST ‘localhost:9200/bank/_search?pretty’ –d  ‘
{      
       “query”: {“match_all”: {} } ,
       “_source”: [“account_number”, “balance”]
}’
```

然后再看”query”字段的匹配，比如下面的例子返回编号是20的账户：

```
curl –XPOST ‘localhost:9200/bank/_search?pretty’ –d ‘
{
      “query”: {“match”:{ “account_number”:20 } }
}’
```

然后返回地址中包含”mill”的账户:

```
curl –XPOST ‘localhost:9200/bank/_search?pretty’ –d ‘
{
      “query”: {“match”: {“address”: “mill”}}
}’
```

下面的例子返回的结果是地址中包含”mill”或”lane”的账户：

```
$curl –XPOST ‘localhost:9200/bank/_search?pretty’ –d ‘
{
      “query”: {“match”: {“address”: ”mill lane”}}
}’
```

区别上面的例子，使用”match_phrase”关键词，下面的例子返回包含词组”mill lane”的账户：

```
curl –XPOST ‘localhost:9200/bank/_search?pretty’ –d
{
       “query” : {“match_phrase”: {“address”:”mill lane”}}
}’
```

下面在介绍一些布尔查询的例子，首先介绍与查询：

```
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
     "query": {
           "bool": {
                "must": [
                     { "match": { "address": "mill" } },
                     { "match": { "address": "lane" } }
                 ]
           }
     }
}'
```

返回地址中既包含”mill”又包含”lane”的账户。然后或查询，地址中包含”mill”或”lane”：

```
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
      "query": {
            "bool": {
                 "must": [
                      { "match": { "address": "mill" } },
                      { "match": { "address": "lane" } }
                 ]
            }
     }
}'
```

既不包含”mill”又不包含”lane”:

```
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
      "query": {
            "bool": {
                 "must_not": [
                           { "match": { "address": "mill" } },
                           { "match": { "address": "lane" } }
                  ]
             }
       }
}'
```

在布尔查询中，可以随意组合布尔子句，例如下面的子句查询年龄40岁，但不在ID州的账户：

```
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
      "query": {
            "bool": {
                 "must": [
                       { "match": { "age": "40" } }
                 ],
                 "must_not": [
                       { "match": { "state": "ID" } }
                 ]
             }
       }
}'
```

###使用过滤器

前面介绍过查询结果中会回馈一个字段”_score”，这是查询和文档相关度的一个度量，值越大表示相关度越大，而值越小表示相关度越小。但有时，Elasticsearch提供了另一查询功能——过滤器。过滤器和查询的概念相似，但却比查询要快，基于下面两个原因：

(1)   过滤器不需要评分

(2)   过滤器通过缓存使得重复检索速度非常快

过滤器可以和查询一起使用，例如下面的例子查询余额在20000到30000之间(包含)的账号：

```
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
      "query": {
            "filtered": {
                   "query": { "match_all": {} },
                         "filter": {
                                "range": {
                                      "balance": {
                                            "gte": 20000,
                                            "lte": 30000
                                       }
                                 }
                        }
           		}
      	}
}'
```

这个命令包含了一个”match_all”查询和一个范围过滤器

###使用聚合

聚合提供了分组和统计的能力，关于聚合的理解，可以类比SQL中的GROUP BY或SQL聚合函数。下面的例子将所有的账户按照州聚合，并且按照默认值返回排名前10的州，并按照默认的降序排列：

```
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
       "size": 0,
            "aggs": {
                 "group_by_state": {
                         "terms": {
                               "field": "state"
                         }
                 }
            }
}'
```

上面的命令类似于SQL中的：

```
SELECT COUNT(*) FROM bank GROUP BY state ORDER BY COUNT(*) DESC
```



##Elasticsearch插件
插件作为一种普遍使用的，用来增强系统核心功能的机制，得到了广泛地使用，elasticsearch也不例外。

### 安装Elasticsearch插件

从elasticsearch-0.90.2开始，elasticsearch插件安装变得很简单，有三种方式：
- 在确保网络顺畅的情况下，执行如下格式的命令即可：

```
plugin --install <org>/<user/component>/<version>
```
- 如果网络不太顺畅，可以下载好插件的压缩包以如下方式安装：

```
bin/plugin --url file://path/to/plugin --install plugin-name
```
- 你可以直接将插件的相关文件拷贝到plugins目录下面，需要注意的是，这种方式需要特别留意插件的种类。

### 如何查看当前已经加载的插件

```
curl  -XGET 'http://localhost:9200/_nodes/plugin'
```
或者可以指定某个实例
```
curl  -XGET 'http://localhost:9200/_nodes/10.0.0.1/plugin'
```
### 推荐的插件
- BigDesk
该插件可以查看集群的JVM信息，磁盘IO，索引创建删除信息等，适合查找系统瓶颈，监控集群状态等。

```
bin/plugin -install lukas-vlcek/bigdesk
```
进入http://localhost:9200/_plugin/bigdesk/ 如下图:
![image description](http://images.cnitblog.com/blog/161247/201402/091102075964081.jpg)
![image description](http://images.cnitblog.com/blog/161247/201402/091102488363684.jpg)

- Head

elasticsearch-head是一个elasticsearch的集群管理工具，它是完全由html5编写的独立网页程序，你可以通过插件把它集成到es。
```
bin/plugin -install mobz/elasticsearch-head
```
进入http://localhost:9200/_plugin/head/ 如下图:
![image description](http://images.cnitblog.com/blog/161247/201402/091059467131077.jpg)


# Elasticsearch进阶篇
## Elasticsearch集群
### 分布式集群
Elasticsearch可以在分布式环境中运行，你可以随时根据你的需求扩展Elasticsearch，你可以购买配置更好的主机或者购买更多地主机来达到扩展的目的。

硬件越强大，Elasticsearch运行的也就越快，但是垂直扩展方式也有它的局限性。真正的扩展来自于横向扩展方式，在集群中添加更多的节点，这样能在节点之间分配负载。

对于大多数的数据库来说，横向扩展意味着你的程序往往需要大改，以充分使用这些新添加的设备。相比而言，Elasticsearch自带分布式功能：他知道如何管理多个节点并提供高可用性。这也就意味着你的程序根本不需要为扩展做任何事情。

### 空集群
如果我们启用一个既没有数据，也没有索引的单一节点，那我们的集群看起来就像是这样
![image description](https://fuxiaopang.gitbooks.io/learnelasticsearch/content/images/02-01_cluster.png)

节点是Elasticsearch运行中的实例，而集群则包含一个或多个具有相同cluster.name的节点，它们协同工作，共享数据，并共同分担工作负荷。由于节点是从属集群的，集群会自我重组来均匀地分发数据。

集群中的一个节点会被选为master节点，它将负责管理集群范畴的变更，例如创建或删除索引，添加节点到集群或从集群删除节点。master节点无需参与文档层面的变更和搜索，这意味着仅有一个master节点并不会因流量增长而成为瓶颈。任意一个节点都可以成为master节点。我们例举的集群只有一个节点，因此它会扮演master节点的角色。

作为用户，我们可以访问包括master节点在内的集群中的任一节点。每个节点都知道各个文档的位置，并能够将我们的请求直接转发到拥有我们想要的数据的节点。无论我们访问的是哪个节点，它都会控制从拥有数据的节点收集响应的过程，并返回给客户端最终的结果。这一切都是由Elasticsearch 透明管理的。

### 集群健康
在 Elasticsearch集群中可以监控统计很多信息，其中最重要的就是：集群健康(cluster health)。它的 status 有 green、yellow、red 三种；
```
GET /_cluster/health
```
在一个没有索引的空集群中，它将返回如下信息：
```
{
   "cluster_name":          "elasticsearch",
   "status":                "green",
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
- status 是我们最应该关注的字段。
status可以告诉我们当前集群是否处于一个可用的状态。三种颜色分别代表：

状态|意义
---|---
green|所有主分片和从分片都可用
yellow|所有主分片可用，但存在不可用的从分片
red|存在不可用的主要分片

### 添加索引
为了将数据添加到Elasticsearch，我们需要索引(index) ——存储关联数据的地方。实际上，索引只是一个逻辑命名空间(logical namespace)，它指向一个或多个分片(shards)。

分片(shard)是工作单元(worker unit)底层的一员，它只负责保存索引中所有数据的一小片。分片是一个独立的Lucene实例既可，并且它自身也是一个完整的搜索引擎。我们的文档存储并且被索引在分片中，但是我们的程序并不会直接与它们通信。取而代之，它们直接与索引进行通信的。

在 elasticsearch中，分片用来分配集群中的数据。把分片想象成一个数据的容器。数据被存储在分片中，然后分片又被分配在集群的节点上。当你的集群扩展或者缩小时，elasticsearch会自动的在节点之间迁移分配分片，以便集群保持均衡。

分片分为主分片(primary shard) 以及从分片(replica shard)两种。在你的索引中，每一个文档都属于一个主分片，所以具体有多少主分片取决于你的索引能存储多少数据。

虽然理论上主分片对存储多少数据是没有限制的。分片的最大数量完全取决于你的实际状况：硬件的配置、文档的大小和复杂度、如何索引和查询你的文档，以及你期望的响应时间。


从分片只是主分片的一个副本，它用于提供数据的冗余副本，在硬件故障时提供数据保护，同时服务于搜索和检索这种只读请求。

索引中的主分片的数量在索引创建后就固定下来了，但是从分片的数量可以随时改变。

接下来，我们在空的单节点集群中上创建一个叫做 blogs的索引。一个索引默认设置了5个主分片，但是为了演示，我们这里只设置3个主分片和一组从分片（每个主分片有一个从分片对应）：

```
PUT /blogs
{
   "settings" : {
      "number_of_shards" : 3,
      "number_of_replicas" : 1
   }
}
```
现在，我们的集群看起来就像下图所示了有索引的单节点集群，这三个主分片都被分配在 Node 1。
![image description](https://fuxiaopang.gitbooks.io/learnelasticsearch/content/images/02-02_one_node.png)

如果我们现在查看 集群健康(cluster-health) ，我们将得到如下信息：
```
{
   "cluster_name":          "elasticsearch",
   "status":                "yellow",
   "timed_out":             false,
   "number_of_nodes":       1,
   "number_of_data_nodes":  1,
   "active_primary_shards": 3,
   "active_shards":         3,
   "relocating_shards":     0,
   "initializing_shards":   0,
   "unassigned_shards":     3 <2>
}
```

集群的 status 为 yellow.我们的三个从分片还没有被分配到节点上。集群的健康状况yellow意味着所有的主分片(primary shards)启动并且运行了，这时集群已经可以成功的处理任意请求，但是从分片(replica shards)没有完全被激活。事实上，当前这三个从分片都处于unassigned（未分配）的状态，它们还未被分配到节点上。在同一个节点上保存相同的数据副本是没有必要的，如果这个节点故障了，就等同于所有的数据副本也丢失了。现在我们的集群已经可用了，但是依旧存在因硬件故障而导致数据丢失的风险。

### 增加故障转移
在单一节点上运行意味着有单点故障的风险，没有数据冗余备份。幸运的是，我们可以启用另一个节点来保护我们的数据。

只要第二个节点与第一个节点的cluster.name相同（参见./config/elasticsearch.yml文件中的配置），它就能自动发现并加入到第一个节点的集群中。如果没有，请结合日志找出问题所在。这可能是多播（multicast）被禁用，或者防火墙阻止了节点间的通信。

如果我们启动了第二个节点，这个集群应该叫做双节点集群(cluster-two-nodes)，双节点集群——所有的主分片和从分片都被分配: 
![image description](https://fuxiaopang.gitbooks.io/learnelasticsearch/content/images/02-03_two_nodes.png)

当第二个节点加入后，就产生了三个从分片(replica shards) ，它们分别于三个主分片一一对应。也就意味着即使有一个节点发生了损坏，我们可以保证数据的完整性。

所有被索引的新文档都会先被存储在主分片中，之后才会被平行复制到关联的从分片上。这样可以确保我们的文档在主节点和从节点上都能被检索。

当前，cluster-health 的状态为green，这意味着所有的6个分片（三个主分片和三个从分片）都已激活：

```
{
   "cluster_name":          "elasticsearch",
   "status":                "green",
   "timed_out":             false,
   "number_of_nodes":       2,
   "number_of_data_nodes":  2,
   "active_primary_shards": 3,
   "active_shards":         6,
   "relocating_shards":     0,
   "initializing_shards":   0,
   "unassigned_shards":     0
}
```
集群的 status 是 green.
我们的集群不仅功能齐全的，并且具有高可用性。

### 横向扩展
随着应用需求的增长，我们该如何扩展？如果我们启动第三个节点，集群内会自动重组，这时便成为了三节点集群(cluster-three-nodes)。
分片已经被重新分配以平衡负载：
![image description](https://fuxiaopang.gitbooks.io/learnelasticsearch/content/images/02-04_three_nodes.png)

在Node 1和Node 2中分别会有一个分片被移动到Node 3上，这样一来，每个节点上就都只有两个分片了。这意味着每个节点的硬件资源（CPU、RAM、I/O）被更少的分片共享，所以每个分片就会有更好的性能表现。

分片本身就是一个非常成熟的搜索引擎，它可以使用单个节点的所有资源。我们一共有6个分片（3个主分片和3个从分片），因此最多可以扩展到6个节点，每个节点上有一个分片，这样每个分片都可以使用到所在节点100%的资源了。

### 扩展更多
但是如果我们想要扩展到六个节点以上应该怎么办？

主分片的数量在索引创建的时候就已经指定了，实际上，这个数字定义了能存储到索引中的数据最大量（具体的数量取决于你的数据，硬件的使用情况）。例如，读请求——搜索或者文档恢复就可以由主分片或者从分片来执行，所以当你拥有更多份数据的时候，你就拥有了更大的吞吐量。

从分片的数量可以在运行的集群中动态的调整，这样我们就可以根据实际需求扩展或者缩小规模。接下来，我们来增加一下从分片组的数量：

```
PUT /blogs/_settings
{
   "number_of_replicas" : 2
}
```

增加number_of_replicas到2： 
![image description](https://fuxiaopang.gitbooks.io/learnelasticsearch/content/images/02-05_replicas.png)
从图中可以看出，现在 blogs 的索引总共有9个分片：3个主分片和6个从分片。也就是说，现在我们就可以将总节点数扩展到9个，就又会变成一个节点一个分片的状态了。最终我们得到了三倍搜索性能的三节点集群。

### 故障恢复
前文我们已经提到过Elasticsearch可以应对节点故障。让我们来尝试一下。如果我们把第一个节点杀掉，我们的集群就会如下图所示：
![image description](https://fuxiaopang.gitbooks.io/learnelasticsearch/content/images/02-06_node_failure.png)

被杀掉的节点是主节点。而为了集群的正常工作必须需要一个主节点，所以首先进行的进程就是从各节点中选择了一个新的主节点：Node 2。

主分片1和2 在我们杀掉Node 1后就丢失了，我们的索引在丢失主节点的时候是不能正常工作的。如果我们在这个时候检查集群健康状态，将会显示 red：存在不可用的主节点！

幸运的是，丢失的两个主分片的完整拷贝在存在于其他的节点上，所以新的主节点所完成的第一件事情就是将这些在 Node 2 和 Node 3 上的从分片提升为主分片，然后集群的健康状态就变回至 yellow。这个提升的进程是瞬间完成了，就好像按了一下开关。

那么为什么集群健康状态依然是是 yellow而不是green 呢？是因为现在我们有3个主分片，但是我们之前设定了1个主分片有2个从分片，但是现在却只有1份从分片，所以状态无法变为green，不过我们可以不用太担心这里：当我们再次杀掉 Node 2的时候，我们的程序依旧可以在没有丢失任何数据的情况下运行，因为 Node 3中依旧拥有每个分片的备份。

如果我们重启Node 1，集群就能够重新分配丢失的从分片，这样结果就会与三节点两从集群一致。如果 Node 1依旧还有旧节点的内容，系统会尝试重新利用他们，并只会复制在故障期间的变更数据。

## Elasticsearch JVM详解
### Elasticsearch JVM 配置概览

JVM参数|Elasticsearch默认值|Environment变量
---|---
-Xms|256m|ES_MIN_MEM
-Xmx|1g|ES_MAX_MEM
-Xms and -Xmx	||ES_HEAP_SIZE
-Xmn||ES_HEAP_NEWSIZE
-XX:MaxDirectMemorySize||ES_DIRECT_SIZE
-Xss|256k|	
-XX:UseParNewGC|+|	
-XX:UseConcMarkSweepGC|+|	
-XX:CMSInitiatingOccupancyFraction|75|	
-XX:UseCMSInitiatingOccupancyOnly|+|	
-XX:UseCondCardMark|(commented out)|

我们可以注意到ES JVM Heap内存设置为在256M在1GB之间.这个设置是为在开发和示范环境中使用的,开发人员可以通过简单地安装ES就可以使用了，但是这样的内存设置在很多情况下都是不够用的，我在需要设置更大的值。

![image description](http://pic002.cnblogs.com/images/2012/155807/2012121110055763.jpg)

ES_MIN_MEM/ES_MAX_MEM用于控制jvm的堆内存，另外还有ES_HEAP_SEIZ,这样我可以设置更多的堆内存用于ES,另外建议不在启动内存堆平衡，因为这样会浪费很大的性能。

ES_HEAP_NEWSIZE这个参数用于控制堆内存的子集，即新生代堆控制。

ES_DIRECT_SIZE，我们可以对应到Direct Memory Size这个参数，在JVM管理数据中使用的是NIO，本机内存可以映射到虚拟地址空间，在X64的架构上更有效，在ES中没有选择进行设置，但是有一个问题，本机直接内存的分配不会受到Java堆大小的限制，但是即然是内存那肯定还是要受到本机物理内存（包括SWAP区或者Windows虚拟内存）的限制的，一般服务器管理员配置JVM参数时，会根据实际内存设置-Xmx等参数信息，但经常忽略掉直接内存，使得各个内存区域总和大于物理内存限制（包括物理的和操作系统级的限制），而导致动态扩展时出现OutOfMemoryError异常。

下面例出一些JVM参数设置

JVM parameter | Garbage collector
---|---
-XX:+UseSerialGC | serial collector
-XX:+UseParallelGC|parallel collector
-XX:+UseParallelOldGC|Parallel compacting collector
-XX:+UseConcMarkSweepGC|Concurrent-Mark-Sweep ( CMS ) collector
-XX:+UseG1GC|Garbage-First collector (G1)

- UseParNewGC和UseConcMarkSweepGC是结并并行和行发性的垃圾回收机制，在JAVA6中将默认为UserParNewGC和UseGoncMarkSweepGC并禁用串行收集器。
- CMSInitiatingOccupancyFraction垃圾回收，这个75是指，到heap占用到75%时开发进行清理，我们知道堆分为新生代和老年代两块可新生代一块为老年代的两倍，也许在没有达到75%时也可能进行垃圾回收。
- UseCondCardMark  将在在高度并发的情况下，将些值注释掉

总结：

```
1、修改MAX 和MIN Heap大小设置。
2、设置垃圾回收百分比
3、如果在JAVA7中禁用默认的G1垃圾回收机制。
```
### JVM进程的内存结构
![image description](http://pic002.cnblogs.com/images/2012/155807/2012121110062359.jpg)

JVM内存分为如下几段：
- JVM CODE用于内部代码存放
- Noe-heap memory用于加载类
- Stack memory 用于存放本地变量和线程操作数
- Heap memory 存放引用类型对象
- Direct Buffer，缓冲输入，输出数据 

Heap memory大小设置是非常重要的，因为java的运行取决于一个合理的heap的大小,如果设置太小，在许多垃圾回收或是高性能的情况下就会出现OutOfMemory异常。如果堆太大，垃圾回收将需要更大的数据，该算法将要面对更高数量的存活堆，这样操作系统也会面对较大的压力。

Non-heap内存分配是由java应用程序自动设置的，没有办法控制这个参数，因为它是由JAVA应用程序代码决定的。

### 垃圾回收与Lucene段
在ES中的垃圾回收器是集用的CMS垃圾回收，这种回收器不是提高敢回收的效率可是降低了回收的次数，但是面对比较大的数据集合时，这种回收可能需要的时间更长。
而这种大的数据集合主要是在Lucene的索引中，因些可以将索引的段进行一行调优工作，提高GC的效率。
index.merge.policy.segments_per_tier

### 减少分页
在大堆内存的情况下，如果内存不足时会与操作系统的SWAP空间进行分页数据的交换，但是这种交换是非常慢的，这种会降低整体性能。

### 垃圾回收器的选择
JAVA 7中的默认是G1垃圾回收器，这种回收器和CMS回收相对，他在于处理吞吐量，但是如果在大堆的情况下CMS回收器在性能上将超过G1.

### 性能调优策略

1、  收集日志

2、  对日志进行分析

3、  选择你要优化的目标

4、  计划优化

5、  应用新有设置

6、  监控程序在新设置后的运行情况

7、  反复试尝

### ES 垃圾回收日志分析

[es-date-1224] [gc][young][3402090][244044] duration [887ms],collections [1]/[1.5s], total [887ms]/[3.3h], memory [4.5gb]->[4gb]/[6.9gb],all_pools {[young] [499.4mb]->[782.8kb]/[532.5mb]}{[survivor][32.7mb]->[30.2mb]/[66.5mb]}{[old] [3.9gb]->[3.9gb]/[6.3gb]}


上面这个例子的情况无须紧张，只是young gc，并且只用了887ms，对于Elasticsearch而言，没有啥影响。唯一需要留心的是，如果在日志中出现连续的和长时间的young gc，则需要引起警觉，可能是你的Heap内存分配不够。

[es-data-1224] [gc][old][76581][22] duration [3.1m], collections[2]/[3.1m], total [3.1m]/[3.1m], memory [3gb]->[1.2gb]/[3.4gb], all_pools{[young] [251mb]->[74.9mb]/[266.2mb]}{[survivor][25.8mb]->[0b]/[33.2mb]}{[old] [2.8gb]->[1.1gb]/[3.1gb]}

如果这种JVM出现，则你的节点一定被踢出了集群。old gc是比较耗时，上面这个例子用了3.1分钟，一定是出了啥大事，要不是然“世界”不会停转这么久的，呵呵！

### 建议

1、ES不要运行在6U22之前因之多版本的JDK存在许多的bug,尽量使用Sun/Oracle比较最出的JDK6-7因为里面修复很多bug.如果在JAVA7正式发布的情况下最好使用JDK7(不过要到2013了)

2、考虑到ES是一个比较新的软件，利用最先的技术来获取性能，尽量从JVM中来挤压性能，另外检索您的操作系统是否是最新版的，尽量使用最新版的操作系统。

3、做好随时更新JAVA版本和ES的版本的情况，因为每季度或是每年都会有新的版本出来。所以在做好版本更新的准备

4、测试从小到大，因为ES的强在多个节点的部署，一个节点是不足以测试出其性能，一个生产系统至少在三个节点以上。

5、测试JVM

6、如果索引有更新请记住对索引段的操作(index.merge.policy.segments_per_tier)

7、在性能调优之前，请先确定系统的最大性能和最大吞吐量

8、启用日志记录对JAVA垃圾回怍机制，有助于更好的诊断，以至于来调整你的系统

9、提高CMS垃圾收集器,您可以添加一个合理的- xx:CMSWaitDuration参数

10、如果堆大小趣过6-8GB,请选择使用CMS


## Elasticsearch的快照与恢复
快照和恢复模块可以将单个索引或者整个集群做一个快照并存放到远程仓库上。目前支持共享文件系统仓库和官方通过插件方式提供的其他仓库。

### 共享文件系统仓库
1、仓库注册

在创建或恢复仓库数据之前，首先需要到ElasticSearch里进行注册，如下命令用my_backup注册一个共享文件系统，快照数据将存放在/mount/backups/my_bakup上：

```
$ curl -XPUT 'http://localhost:9200/_snapshot/my_backup' -d '
{
    "type": "fs",（共享的文件系统仓库）
    "settings": {
        "location": "/mount/backups/my_backup", （快照存放位置）
        "compress": true （快照是否压缩，默认true）

    }
}'
```

2、查看仓库信息：

```
$ curl -XGET 'http://localhost:9200/_snapshot/my_backup?pretty'
{
  "my_backup" : {
    "type" : "fs",
    "settings" : {
      "compress" : "true",
      "location" : "/mount/backups/my_backup"
    }
  }
}
```

3、创建快照

同一个集群中，一个仓库中可以存放多个快照。快照在集群中的名称是唯一的。使用下面命令创建快照名为snapshot_1的快照：


```
$ curl -XPUT "localhost:9200/_snapshot/my_backup/snapshot_1"-d '{
    "indices": "index_1,index_2",
    "ignore_unavailable": "true",
    "include_global_state": false
}'
```

4、查看快照信息：


```
$ curl -XGET "localhost:9200/_snapshot/my_backup/snapshot_1"
```


5、快照恢复


```
$ curl -XPOST "localhost:9200/_snapshot/my_backup/snapshot_1/_restore"-d '{
    "indices": "index_1,index_2",
    "ignore_unavailable": "true",
    "include_global_state": false,
    "rename_pattern": "index_(.+)",
    "rename_replacement": "restored_index_$1"
}'
```

6、监控快照创建或恢复过程

```
$ curl -XGET "localhost:9200/_snapshot/my_backup/snapshot_1/_status"
```

### HDFS插件
ES的Hadoop插件,总共有3个,我们要使用的是hadoop HDFS Snapshot/Restore plugin,它主要用于备份ES数据到HDFS,或者从HDFS恢复数据,也就是ES的snapshot/restore特性。还原可以还原到别的集群,集群名字和节点数量不一样都可以,也就是可以做数据迁移。
- elasticsearch-hadoop
- repository-hdfs
- elasticsearch-hdfs

安装插件

```
bin/plugin -i elasticsearch/elasticsearch-repository-hdfs/2.0.2
```
这里hadoop的版本一定要对应好，否则后面会失败的。如果自动不能安，可以手动去库里面下载，然后放到ES插件目录下面。
[插件库地址](https://oss.sonatype.org/content/repositories/snapshots/org/elasticsearch/elasticsearch-repository-hdfs/)

- 创建配置

```
curl -XPUT 'http://localhost:9200/_snapshot/my_backup' -d '{
  "type": "hdfs",
  "settings": {
    "uri": "hdfs://myhadoop:8020",
    "path": "/es",
    "conf_location": "hdfs-site.xml"
  }
}'
```

```
{"acknowledged":true} 返回这个表示创建成功
```

- 通过下面的命令，可以查看所有创建的配置

```
curl http://localhost:9200/_snapshot/_all
```

- 备份数据

```
curl -XPUT "localhost:9200/_snapshot/my_backup/snapshot_1?wait_for_completion=true"
```

备份成功，我们可以在 http://myhadoop:50070/explorer.html#/es 下面看到我们创建的备份
![image description](http://static.oschina.net/uploads/space/2014/1113/220651_fiKZ_99041.png)

- 还原数据

```
curl -XPOST "localhost:9200/_snapshot/my_backup/snapshot_1/_restore?wait_for_completion=true"
```
## Elasticsearch升级

### 操作流程
Elasticsearch官方提供的Elasticsearch版本升级流程可参考：[https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-upgrade.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-upgrade.html)
<br/>

在实际操作过程中我们首先遇到的一个问题是：[https://github.com/elastic/elasticsearch/pull/7210](https://github.com/elastic/elasticsearch/pull/7210 ) ，简单来说就是elasticsearch的低版本和高版本之间采用的压缩处理有所不同，导致低版本的分片在高版本的集群中进行allocation时出错。这个问题在elasticsearch-1.3.2版本中已经解决，在官方的文档中有如下说明：
[The sequence of events needed to trigger this bug occurs rarely. It may be responsible for the occasional case of corruption that we have seen reported, but which has remained unexplained.  We advise users to upgrade, but in the meantime you can avoid this bug completely by disabling compression with the following request](https://www.elastic.co/blog/elasticsearch-1-3-2-released):

具体操作如下：

```
curl -XPUT "http://localhost:9200/_cluster/settings" -d'
{
  "persistent": {
    "indices.recovery.compress": false
  }
}'
```


解决不同版本间的压缩问题后，我们才开始进行elasticsearch-1.2.2升级到Elasticsearch-1.5.2的操作。

以下测试均在测试网18.33和18.36两台服务器上操作完成，初始状态下两台服务器上分别部署了两个master节点和data节点
![](http://i.imgur.com/TwQsMw3.png)

###集群节点Elasticsearch升级
- 操作流程

1.首先执行Elasticsearch-1.2.2集群的索引数据备份

2.关闭elasticsearch-1.2.2集群的recovery.compress
		
```
curl -XPUT "http://localhost:9200/_cluster/settings" -d'
		{
  			"persistent": {
   			"indices.recovery.compress": false
  			}
		}'
```

3.关闭shard reallocation
		
```
curl -XPUT localhost:9200/_cluster/settings -d '{
        "transient" : {
            "cluster.routing.allocation.enable" : "none"
        	}
		}'
```

4.关闭集群中需要升级的节点

```
curl -XPOST 'http://localhost:9200/_cluster/nodes/_local/_shutdown'
```

5.确认被关闭节点上的分片正确重新分配到集群中还在运行的节点上

6.在服务器上安装好elasticsearch-1.5.2的实例，
将elasticsearch-1.2.2实例的配置文件覆盖elasticsearch-1.5.2的配置文件；
同时elasticsearch-1.5.2节点的data目录和elasticsearch-1.2.2的data目录做一个symbolic link：

```
cp /app/IDC/KT-ES/elasticsearch-1.2.2-data/config/* ./config/
cp -r /app/IDC/KT-ES/elasticsearch-1.2.2-master/bin/service* ./bin/service
ln -s /app/IDC/KT-ES/elasticsearch-1.2.2-data/data/ ./data
```

7.启动新升级的elasticsearch-1.5.2节点，确认其正常加入cluster
8.恢复分片的reallocation
		
```
curl -XPUT localhost:9200/_cluster/settings -d '{
        	"transient" : {
           		"cluster.routing.allocation.enable" : "all"
        	}
		}'
```

9.观察所有分片可能在所有的节点上allocated。分片balance会花费一些时间

10.针对所有剩下的节点，重复上述操作步骤。

11.待所有节点操作完成后，执行如下命令：
	  
```
curl -XPUT localhost:9200/_cluster/settings -d '{
	    "persistent" : {
	        "cluster.routing.allocation.disable_allocation" : true
	        }
	   }
```
同时整个集群进行重启；

12.待所有节点操作完成后，打开indices.recovery.compress，同时整个集群进行重启。

# Elasticsearch实战篇
## Filtered Query和Post Filter如何选择
在Elasicsearch的官网文档中：（[http://www.elastic.co/guide/en/elasticsearch/guide/current/_filtered_query.html](http://www.elastic.co/guide/en/elasticsearch/guide/current/_filtered_query.html)），介绍了两种查询方式，一个是Filtered Query，一个是Post Filter。前者是先使用filter，过滤后的结果再query；后者是先query，query的结果再过滤。而在es中，filter的操作使用了cache，速度比query快，因而官网文档提出了一个关于性能的警告（翻译如下）:
只有当你需要对搜索结果和聚合使用不同的过滤方式时才考虑使用Post_filter。有时一些用户会直接在常规搜索中使用Post_filter。不要这样做！Post_filter会在查询之后才会被执行，因此会失去过滤在性能上帮助(比如缓存)。Post_filter应该只和聚合一起使用，并且仅当你使用了不同的过滤条件时。

- 测试一
查询条件：

```
boolFilter.must(FilterBuilders.termFilter("appId", "R6Z7Ews31c8O4BTidZaWyA"))
```
目标数：10010077hits,测试结果如下（耗时:ms）：

Filtered Query | Post Filter
---|---
176880|171064
161548|152789
164953|174562
150026|155928
177425|159847

结果分析：对单个查询条件共10010077条目标的查询测试。两者对比发现消耗的时间有大有小，不稳定。且由于Filtered Query第一次查询是没有cache的，该次消耗时间偏大。

- 测试二
查询条件：

```
boolFilter.must(FilterBuilders.termFilter("appId", "R6Z7Ews31c8O4BTidZaWyA"));
boolFilter.should(FilterBuilders.termFilter("province", "上海"))
		.should(FilterBuilders.termFilter("province", "山西"))
		.should(FilterBuilders.regexpFilter("city", "14.*"))
		.should(FilterBuilders.regexpFilter("city", "3307.*"))
		.should(FilterBuilders.regexpFilter("city", "40.*"));
```

目标数：9010477hits，测试结果（单位:ms）：

Post Filter | Filtered Query
---|---
157410|141416
159537|149571
151644|145090
151364|155600

结果分析：此次使用了较为复杂的正则查询，且组合了多种条件。但目标总数与上次差不多，结果对比发现Filtered Query的查询平均速度看起来比使用Post Filter 快了一点但仍不稳定，不能稳定的重现。

- 测试三
查询条件:

```
boolFilter.must(FilterBuilders.termFilter("appId", "R6Z7Ews31c8O4BTidZaWyA"))
		.should(FilterBuilders.termFilter("province", "北京"));
```

目标数：999599hits，测试结果（单位:ms）：

Post Filter | Filtered Query
---|---
24631|19260
25485|21232
25206|20373
23271|19046

结果分析：此次尝试缩小样本量，发现Filtered Query能明显的比Post Filter查询快。快了4、5秒左右，且较为稳定。


- 测试四
查询条件：

```
boolFilter.must(FilterBuilders.termFilter("appId", "R6Z7Ews31c8O4BTidZaWyA"))
		.should(FilterBuilders.termFilter("province", "山西"));
```

目标数：11073hits，测试结果（单位:ms）：

Post Filter | Filtered Query
---|---
1883|547
1563|563
1660|499
1612|604

结果分析：此次样本量缩小至1w+，发现FilteredQuery性能优势更加明显了，比PostFilter快了2到3倍。

- 测试五
查询条件：

```
boolFilter.must(FilterBuilders.termFilter("appId", "R6Z7Ews31c8O4BTidZaWyA"))
		.should(FilterBuilders.termFilter("city", "404"));
```

目标数：1003hits，测试结果(单位:ms)：

Post Filter  | Filtered Query
---|---
1616|279
1304|277
1657|302
1339|296

结果分析：此次的目标量只有1003条，两者方式的性能对比最明显，Filtered Query的搜索速度5倍左右于Post Filter 。

## Elasticsearch的get操作为什么那么慢

在我们具体的业务操作过程中，我们会通过id进行get操作。
例如：

```
GetResponse response = client.prepareGet("twitter", "tweet", "1")
    .execute().actionGet();
```
但是我们发现Elasticsearch1.2.2版本进行get操作非常的慢。
我们通过JVM分析工具进行分析：

<img src="/Users/bianzexin/Documents/work/Elasticsearch/个推Elasticsearch培训文档/benryanconversion.png" />
<img src="/Users/bianzexin/Documents/work/Elasticsearch/个推Elasticsearch培训文档/benryanconversion-2.png" />
<img src="/Users/bianzexin/Documents/work/Elasticsearch/个推Elasticsearch培训文档/benryanconversion-3.png" />

通过工具的分析结果，我们可以看到节点服务器的cpu基本全部跑满，主要是因为每次es的get请求的条件都不一样，在ES集群的 queryResult caches 中获取不到结果，所以每次都得调用identityHashCode来重新计算hash值，然后放入 WeakIdentityMap 中，这些操作占用了绝大部分cpu时间，导致服务器的cpu占用率非常高。

通过对比Elasticsearch1.2.2版本和后续Elasticsearch1.5.2版本，我们发现在Elasticsearch1.5.2版本中，get操作已经弃用了bloomfilter的hash算法处理，使用 ConcurrentMap<IndexReader,CloseableThreadLocal<PerThreadIDAndVersionLookup>> lookupStates 缓存了上一次根据_uid进行reader的信息。
如果在lookupStates中找不到，InternalEngine.get方法返回 GetResult.NOT_EXISTS ，继而ShardGetService. innerGet 直接返回
new GetResult(shardId.index().name(), type, id, -1, false, null, null)。
<img src="/Users/bianzexin/Documents/work/Elasticsearch/个推Elasticsearch培训文档/benryanconversion-4.png" />

但是通过后期业务运营发现虽然Elasticsearch1.5.2改进了get操作，但性能还是很慢。
因为在我们的业务中，通过_id去直接get索引的source信息的，而_id默认是not indexed(默认不索引)，索引导致get操作那么慢。

## doc_value的使用
在我们实际的业务中，使用复杂的条件进行搜索时，经常出现Out Of Memory的错误，
<img src="/Users/bianzexin/Documents/work/Elasticsearch/个推Elasticsearch培训文档/docvalues.jpg" />
主要是目前我们所有的fielddata都放在内存中计算导致的，此类问题可以通过doc_values来解决。
###什么是doc_value
In-memory模式中，fielddata受到heap内存大小限制，虽然这个问题可以通过集群的横向扩容解决，可能需要经常增加节点，而且即使加了，你还是会发现在其他一些资源利用不充分的节点上，在排序和聚合查询的时候仍然会消耗你大量的heap空间。

fielddata字段数据默认下，会频繁的在内存当中加载。但这不是唯一的方法，在索引数据时，fielddata字段数据还可以被写在磁盘中，这种方式可以提供和加载到内存中的一样的功能，而不会占用heap的内存空间。

Docvalues是在1.0.0之后加入到elasticsearch中的。通过基准测试与性能分析，各类的瓶颈——无论是elasticsearch还是lucene，都已经被发现并且已经被移除了。但是直到现在还是远远慢于字段数据放内存的in-memory方式。

- 存放在磁盘代替存放内存，可以允许你的集群负载大量fielddata字段数据而不会超量使用内存。这样，你的heap space就可以设置小一些了，对垃圾回收的速度有所帮助，理所当然的，节点也会更稳定些。

- DocValues是索引时建立的，而不是搜索的时候。当通过非反向的反向索引搜索时候，in-memory方式，内存中的字段数据必须被频繁的读写，而doc vales是预先建立的，并能更快的初始化。

像trade-off这种索引量大的搜索，访问fielddata会比较缓慢，设置docvalues会有显著的效果，在大量请求的时候，你甚至不会注意到你的搜索会变慢。结合更快的垃圾回收机制和初始化时间，你会留意到搜索性能会得到有效提升。

文件系统的缓存空间越多，docvalue的性能会越好。如果文件都是docvalues并且都位于文件系统的缓存中，那么访问这些文件的速度几乎与访问内存媲美的。而文件系统缓存由内核控制而非JVM。

###启动DocValues

numeric, date, Boolean, binary, and geo-point这些字段以及not_analyzed的字符串可以设置Docvalues属性。一般不处理analyze的字符串字段。Docvalues在mapping重的每个字段属性中定义，这样对于不同的字段，可以结合in-memory与docvalues使用。

```
PUT /music/_mapping/song

{

	“properties” : {

		“tag”: {

			“type”:       “string”,

			“index” :     “not_analyzed”,

			“doc_values”: true

		}

	}

}
```

设置了docvalues之后，字段创建时，就是用磁盘存储fielddata而不是内存存储fielddata了。

就这样！不需要再配置其他东西，查询，聚合，排序，以及一般的功能脚本；他们现在都会用到docvalues了。

你可以随便的使用docvalues这参数。用得越多，你使用的heap内存空间就可以越少。在不久的将来，docvalues应该会变成默认的参数。





