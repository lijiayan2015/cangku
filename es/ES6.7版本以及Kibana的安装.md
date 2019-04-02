### ES简介:
  - Elasticsearch(通常简称为ES)是一个高度可扩展的开源全文搜索和分析引擎。它允许您快速，近实时地存储，搜索和分析大量数据。它通常用作底层引擎/技术，为具有复杂搜索功能和要求的应用程序提供支持,本身扩展性很好，可以扩展到上百台服务器，处理PB级别的数据.
  - Lucene与ES关系:
    - Lucene只是一个库。想要使用它，你必须使用Java来作为开发语言并将其直接集成到你的应用中，更糟糕的是，Lucene非常复杂，你需要深入了解检索的相关知识来理解它是如何工作的。
    - Elasticsearch也使用Java开发并使用Lucene作为其核心来实现所有索引和搜索的功能，但是它的目的是通过简单的RESTful API来隐藏Lucene的复杂性，从而让全文搜索变得简单。

### ES的工作原理:
当ElasticSearch的节点启动后，它会利用多播(multicast)(或者单播，如果用户更改了配置)寻找集群中的其它节点，并与之建立连接。这个过程如下图所示：<br/>
![ES的工作原理](https://www.github.com/lijiayan2015/cangku/raw/master/小书匠/1553828035352.png)



### ES中的基础概念:
  - Cluster：集群
    - ES可以作为一个独立的单个搜索服务器。不过，为了处理大型数据集，实现容错和高可用性，ES可以运行在许多互相合作的服务器上。这些服务器的集合称为集群。
  - Node:节点
    - 形成集群的每个服务器称为节点。
  - Shard：分片
	- 当有大量的文档时，由于内存的限制、磁盘处理能力不足、无法足够快的响应客户端的请求等，一个节点可能不够。这种情况下，数据可以分为较小的分片。每个分片放到不同的服务器上。 
当你查询的索引分布在多个分片上时，ES会把查询发送给每个相关的分片，并将结果组合在一起，而应用程序并不知道分片的存在。即：这个过程对用户来说是透明的。
  - Replia：副本
    - 为提高查询吞吐量或实现高可用性，可以使用分片副本。 
    - 副本是一个分片的精确复制，每个分片可以有零个或多个副本。ES中可以有许多相同的分片，其中之一被选择更改索引操作，这种特殊的分片称为主分片。 
    - 当主分片丢失时，如：该分片所在的数据不可用时，集群将副本提升为新的主分片。
  - 全文检索
    - 全文检索就是对一篇文章进行索引，可以根据关键字搜索，类似于mysql里的like语句。 
全文索引就是把内容根据词的意义进行分词，然后分别创建索引，例如”你们的激情是因为什么事情来的” 可能会被分词成：“你们“，”激情“，“什么事情“，”来“ 等token，这样当你搜索“你们” 或者 “激情” 都会把这句搜出来



### 关系数据库MySQL对比
![ES与mysql的对比](https://www.github.com/lijiayan2015/cangku/raw/master/小书匠/1553828404753.png)
### ES 6.7安装
  - 环境:
	- Centos7
	- JDK8
	- ES 6.7
 - 管理ES的用户,建议使用一个单独的用户来管理ES集群.同时因为安全问题,es不让用es来启动集群,需要使用其他非root用户来启动.
 - 下载ES `wget  https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.7.0.tar.gz`
 - 解压:我的安装目录:`/soft/es`    解压安装包: `tar -zxvf elasticsearch-6.7.0.tar.gz`
 - 修改配置文件:elasticsearch.yml
  ``` xml
# ======================== Elasticsearch Configuration =========================
#
# NOTE: Elasticsearch comes with reasonable defaults for most settings.
#       Before you set out to tweak and tune the configuration, make sure you
#       understand what are you trying to accomplish and the consequences.
#
# The primary way of configuring a node is via this file. This template lists
# the most important settings you may want to configure for a production cluster.
#
# Please consult the documentation for further information on configuration options:
# https://www.elastic.co/guide/en/elasticsearch/reference/index.html
#
# ---------------------------------- Cluster -----------------------------------
#
# Use a descriptive name for your cluster:
# 集群名称
cluster.name: es-cluster
#
# ------------------------------------ Node ------------------------------------
#
# Use a descriptive name for the node:
# 节点名称,可以设置为当前节点所在的主机名
node.name: vhost1
#
# Add custom attributes to the node:
# 可以自定义节点属性
#node.attr.rack: r1
#
# ----------------------------------- Paths ------------------------------------
#
# Path to directory where to store the data (separate multiple locations by comma):
# 存储数据的目录,确保管理es集权的用户有权限读写该目录
path.data: /data/es-data
#
# Path to log files:
# log日志目录,确保管理es集权的用户有权限读写该目录
path.logs: /soft/es/elasticsearch-6.7.0/logs
#
# ----------------------------------- Memory -----------------------------------
#
# Lock the memory on startup:
#
#bootstrap.memory_lock: true
#
# Make sure that the heap size is set to about half the memory available
# on the system and that the owner of the process is allowed to use this
# limit.
#
# Elasticsearch performs poorly when the system is swapping the memory.
#
# ---------------------------------- Network -----------------------------------
#
# Set the bind address to a specific IP (IPv4 or IPv6):
# 当前节点的ip地址,可以使用主机名
network.host: vhost1
#
# Set a custom port for HTTP:
# 端口号,使用默认的9200 不用修改
#http.port: 9200
#
# For more information, consult the network module documentation.
#
# --------------------------------- Discovery ----------------------------------
#
# Pass an initial list of hosts to perform discovery when new node is started:
# The default list of hosts is ["127.0.0.1", "[::1]"]
# 定义可被发现的节点列表
discovery.zen.ping.unicast.hosts: ["vhost1:9300", "vhost2:9300", "vhost3:9300"]
#
# Prevent the "split brain" by configuring the majority of nodes (total number of master-eligible nodes / 2 + 1):
# master 选举最少的节点数，这个一定要设置为N/2+1，其中N是：具有master资格的节的数量，而不是整个集群节点个数
#discovery.zen.minimum_master_nodes: 
#
# For more information, consult the zen discovery module documentation.
#
# ---------------------------------- Gateway -----------------------------------
#
# Block initial recovery after a full cluster restart until N nodes are started:
#
#gateway.recover_after_nodes: 3
#
# For more information, consult the gateway module documentation.
#
# ---------------------------------- Various -----------------------------------
#
# Require explicit names when deleting indices:
#
#action.destructive_requires_name: true
#
# 其他配置
# 
# 是否存储数据
node.data: true

# 是否参与master的选举
node.master: true

```
 - 配置完以后,分发到其他节点
   - 在安装包的上层目录下,如我的则是在`/soft`下
   - 执行拷贝(之前执行确保其他节点又相同的用户)
     -  `scp -r es vhost2$PWD` 拷贝到vhost2节点,使用$PWD表示拷贝到相同路径下
     -  `scp -r es vhost3$PWD` 拷贝到vhost3节点
 - 拷贝完成后,修改拷贝后节点下的elasticsearch.yml文件
   - vhost2节点:`node.name: vhost2`   `network.host: vhost2`
   - vhost3节点:`node.name: vhost3`   `network.host: vhost3`
   
### 启动集群
进入到es的安装目录下,在每个节点分别执行`./bin/elasticsearch` 这个地方也可以自己写一个脚本来管理集群的启停.

启动集群时可能会报如下错误.
  - `org.elasticsearch.bootstrap.StartupException: java.lang.RuntimeException: can not run elasticsearch as root`
    - 解决方案:因为安全问题elasticsearch不让用root用户直接运行，所以要创建新用户
  - `ERROR: [3] bootstrap checks failed
     [1]: max file descriptors [4096] for elasticsearch process is too low, increase to   at least [65536]`
     - 解决方案:
       - `vi /etc/security/limits.conf`
       -  文件末尾追加:
          -   `es soft nofile 819200    #es为启动es的用户`
          -  ` es hard nofile 819200   #es为启动es的用户`
  - `ERROR: [1] bootstrap checks failed
[1]: max number of threads [3802] for user [es] is too low, increase to at least [4096]`
    - 解决方案:
      - `vim /etc/security/limits.d/20-nproc.conf`
      - 在文件末尾添加:
        - `soft	nproc	4096`
        - `hard	nproc	4096`
        - `root	soft	nproc	unlimited`

  - `ERROR: [1] bootstrap checks failed
[1]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]`
    - 解决方案:
      - 切换到root用户修改配置sysctl.conf 
      - vi /etc/sysctl.conf
      - 添加下面配置：`vm.max_map_count=655360` 
      - 并执行命令： `sysctl -p`

注意:如果修改的是系统文件,需要切换成root用户,修改后,启动的时候,记得切换为管理es的用户.

### 安装Kibana插件
#### Kibana插件
kibana是一个与elasticsearch一起工作的开源的分析和可视化的平台。使用kibana可以查询、查看并与存储在elasticsearch索引的数据进行交互操作。使用kibana能执行高级的数据分析，并能以图表、表格和地图的形式查看数据。 
kibana使得理解大容量的数据变得非常容易。它非常简单，基于浏览器的接口使我们能够快速的创建和分享显示elasticsearch查询结果实时变化的仪表盘。
#### 安装
  - 下载 Kibana插件,这里需要强调一下,Kibana的版本需要与ES的版本对应,否则连接ES的时候可能会报版本不匹配的错误
  - `wget https://artifacts.elastic.co/downloads/kibana/kibana-6.7.0-linux-x86_64.tar.gz`
  - 解压 `tar -zxvf kibana-6.7.0-linux-x86_64.tar.gz`
  - 进入到kibana目录,并修改config目录下面的配置文件:kibana.yml
  - `server.host: "vhost1"   #kibana服务所在的主机,我的当前解压到了vhost1机器上,所以这里为vhost1`
  - `elasticsearch.url: "http://vhost1:9200"   # kibana监听的es集群`
  - `启动kibana : ./bin/kibana,启动成功后,会显示kibana的访问地址:http://vhost1:5601`
  
  
  
  ### TODO
  好了,以上就是ES以及Kibana的安装简单教程.后面还会更新ES的各种操作.
  