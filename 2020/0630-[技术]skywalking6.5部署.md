# Skywalking 6.5 与 ElasticSearch 6.3.2

刚开始，是一套 skywalking 5.0.0-GA 和 ElasticSearch 5.6.16，配置监控 dubbo服务 的调用链关系。然而呢，这个版本视图不给力，前端相当垃圾，而且 skywalking-agent 反正也是需要定制化开发跟踪相关的东西，就索性升级为 6.5 好了。

刚开始少不更事，被某个博主带飞了，给了个 ES 7.0.0 的下载链接，我还用了，ES配置改完起来了，skywalking配置也改对了，但是就铁马的一起就报错，错误是个什么 bad request，排查了好久查不出原因。最后去查了 gayhub 上的 issue，作者对此现象的回答是：

> your ES version is wrong

我好奇的是为啥不是 fucking wrong, bloody wrong ？反正我是挺气愤的，他妈的skywalking 6.5 不能配合 ES 7.0.0 使用，而某博主却热心地给了 7.0.0 的链接，不得不说，有些人既是傻也是坏。

-----------

闲话不多说了，我们选择官方文档中提及的 ElasticSearch 6.3.2 版本，修改其配置文件 `elasticsearch.yml` ，只给出关键参数，其余是否修改按个人需求决定，以下配置皆同此思想，不再赘述。

```yaml
cluster.name: XXXXX
cluster.routing.allocation.disk.threshold_enabled: false

node.name: node-1

bootstrap.memory_lock: false
bootstrap.system_call_filter: false

network.host: 0.0.0.0
```

再让 skywalking 以 standalone 模式启动，修改其配置文件 `application.yml`

```yaml
cluster:
  standalone:
  # Please check your ZooKeeper is 3.5+, However, it is also compatible with ZooKeeper 3.4.x. Replace the ZooKeeper 3.5+
  # library the oap-libs folder with your ZooKeeper 3.4.x library.
#  zookeeper:
#    nameSpace: ${SW_NAMESPACE:""}
#    hostPort: ${SW_CLUSTER_ZK_HOST_PORT:xxxxxx}
#    #Retry Policy
#    baseSleepTimeMs: ${SW_CLUSTER_ZK_SLEEP_TIME:1000} # initial amount of time to wait between retries
#    maxRetries: ${SW_CLUSTER_ZK_MAX_RETRIES:3} # max number of times to retry
#    # Enable ACL
#    enableACL: ${SW_ZK_ENABLE_ACL:false} # disable ACL in default
#    schema: ${SW_ZK_SCHEMA:digest} # only support digest schema
#    expression: ${SW_ZK_EXPRESSION:skywalking:skywalking}
#  kubernetes:
#    watchTimeoutSeconds: ${SW_CLUSTER_K8S_WATCH_TIMEOUT:60}
#    namespace: ${SW_CLUSTER_K8S_NAMESPACE:default}
#    labelSelector: ${SW_CLUSTER_K8S_LABEL:app=collector,release=skywalking}
#    uidEnvName: ${SW_CLUSTER_K8S_UID:SKYWALKING_COLLECTOR_UID}
#  consul:
#    serviceName: ${SW_SERVICE_NAME:"SkyWalking_OAP_Cluster"}
#     Consul cluster nodes, example: 10.0.0.1:8500,10.0.0.2:8500,10.0.0.3:8500
#    hostPort: ${SW_CLUSTER_CONSUL_HOST_PORT:localhost:8500}
#  nacos:
#    serviceName: ${SW_SERVICE_NAME:"SkyWalking_OAP_Cluster"}
#    hostPort: ${SW_CLUSTER_NACOS_HOST_PORT:localhost:8848}
#  # Nacos Configuration namespace
#    namespace: 'public'
#  etcd:
#    serviceName: ${SW_SERVICE_NAME:"SkyWalking_OAP_Cluster"}
#     etcd cluster nodes, example: 10.0.0.1:2379,10.0.0.2:2379,10.0.0.3:2379
---------
storage:
  elasticsearch:
    nameSpace: ${SW_NAMESPACE:"XXXXX"}
    clusterNodes: ${SW_STORAGE_ES_CLUSTER_NODES:xx.xx.xxx.xxx:9200}
    protocol: ${SW_STORAGE_ES_HTTP_PROTOCOL:"http"}
    trustStorePath: ${SW_SW_STORAGE_ES_SSL_JKS_PATH:"../es_keystore.jks"}
    trustStorePass: ${SW_SW_STORAGE_ES_SSL_JKS_PASS:""}
    user: ${SW_ES_USER:""}
    password: ${SW_ES_PASSWORD:""}
    indexShardsNumber: ${SW_STORAGE_ES_INDEX_SHARDS_NUMBER:2}
    indexReplicasNumber: ${SW_STORAGE_ES_INDEX_REPLICAS_NUMBER:0}
    # Those data TTL settings will override the same settings in core module.
    recordDataTTL: ${SW_STORAGE_ES_RECORD_DATA_TTL:7} # Unit is day
    otherMetricsDataTTL: ${SW_STORAGE_ES_OTHER_METRIC_DATA_TTL:45} # Unit is day
    monthMetricsDataTTL: ${SW_STORAGE_ES_MONTH_METRIC_DATA_TTL:18} # Unit is month
    # Batch process setting, refer to https://www.elastic.co/guide/en/elasticsearch/client/java-api/5.5/java-docs-bulk-processor.html
    bulkActions: ${SW_STORAGE_ES_BULK_ACTIONS:1000} # Execute the bulk every 1000 requests
    bulkSize: ${SW_STORAGE_ES_BULK_SIZE:20} # flush the bulk every 20mb
    flushInterval: ${SW_STORAGE_ES_FLUSH_INTERVAL:10} # flush the bulk every 10 seconds whatever the number of requests
    concurrentRequests: ${SW_STORAGE_ES_CONCURRENT_REQUESTS:2} # the number of concurrent requests
    resultWindowMaxSize: ${SW_STORAGE_ES_QUERY_MAX_WINDOW_SIZE:10000}
    metadataQueryMaxSize: ${SW_STORAGE_ES_QUERY_MAX_SIZE:5000}
    segmentQueryMaxSize: ${SW_STORAGE_ES_QUERY_SEGMENT_SIZE:200}
#  h2:
#    driver: ${SW_STORAGE_H2_DRIVER:org.h2.jdbcx.JdbcDataSource}
#    url: ${SW_STORAGE_H2_URL:jdbc:h2:mem:skywalking-oap-db}
#    user: ${SW_STORAGE_H2_USER:sa}
#    metadataQueryMaxSize: ${SW_STORAGE_H2_QUERY_MAX_SIZE:5000}
```

因为 6.5 里 前端配置分离出来了，再修改 `webapp.yml`

```yaml
server:
  port: 8081

collector:
  path: /graphql
  ribbon:
    ReadTimeout: 10000
    # Point to all backend's restHost:restPort, split by ,
    listOfServers: xx.xx.xxx.xxx:12800
```

配完以后可以知道，ES起在9200端口，SW backend 监听 11800 和 12800，SW web 用12800 与 backend 通信，agent默认就推送到 11800 端口给 backend。

java 启动的时候 带上参数 -DSW_AGENT_NAME=appName 即可覆盖 agent.config 里的应用名，实现 agent 复用。

