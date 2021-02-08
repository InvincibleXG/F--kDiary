# Pulsar 在mbp上单机部署

1. 获取 `apache-pulsar-2.4.0-bin.tar.gz` 并解压到 3个 不同的目录

2. 部署 3个 `zookeeper`，经测试，`pulsar自带的zookeeper` 无法完成单机集群的部署，调试过程中发现选举端口只能正常监听 1个，另外两个 zk 的选举端口看不到活动，推测可能是 pulsar 中有某种限制会检测硬盘、mac地址等。而使用 3个 `zookeeper-3.4.6` 修改为与 `pulsar内置zk` 相同的配置，就可以成功建立集群。

3. 启动 zk集群

   1. 保证在 `dataDir` 目录下有不冲突的 myid
   2. zkServer start
   3. zkServer status 可以看到节点的主从标识即为集群启动成功

4. 部署 3个 `bookkeeper`，其中 主要修改 `bk_server.conf` 的：

   1. bookiePort
   2. httpServerPort
   3. storageserver.grpc.port
   4. zk集群节点信息 zkServers=localhost:12181,localhost:12182,localhost:12183
   5. metadataServiceUri 中的 zk地址 对应好上述集群节点「自行配置」
   6. journalDirectories=[defPath]/tmp/bk-txn
      ledgerDirectories=[defPath]/tmp/bk-data 这两个地址最好用绝对地址，因为 **相对的当前路径是 bk的根目录** ，而不是conf文件所在的目录，元数据如果存在了会起不来
   7. advertisedAddress 改成本机IP

5. 启动 bk集群：

   1. 配好以后现在本机任意一个 bk 的 bin 目录下初始化集群元数据 `bookkeeper shell metaformat`  「zk里面没有`ledgers\INSTANCEID`的话 也就是bk第一次启动要初始化集群元数据。后面如果多次初始化会造成INSTANCEID变更，也就是$journalDirectories$和$ledgerDirectories$中`current/VERSION`文件内容变更。这样很可能导致zk上的`ledgers\INSTANCEID`与当前INSTANCEID不一致，从而启动bookie报错 —— **不一致的话要么在启动bk前清除每个集群节点的current目录，要么初始化元数据以后用zk里`ledgers\INSTANCEID`的值修改每个集群节点的`current/VERSION`，要么修改`ledgers\INSTANCEID`的值为旧INSTANCEID**」
   2. bookkeeper bookie 启动每一个bk实例，构成集群（如果上述 4-6点 的目录配置有冲突启动就会报错not match）
   3. 启动无报错后，在任意实例的bin目录下进行测试 `bookkeeper shell bookiesanity`，测试succeed说明集群可用

6. 部署 3个 `broker`，主要修改 broker.conf 文件内容：

   1. brokerServicePort

   2. brokerServicePortTls

   3. webServicePort

   4. webServicePortTls 

   5. 添加 ZooKeeper 集群节点信息

      zookeeperServers=localhost:12181,localhost:12182,localhost:12183

      configurationStoreServers=localhost:12181,localhost:12182,localhost:12183

   6. 指定集群名 clusterName

7. 启动 broker集群

   1. 分别后台启动每个实例 pulsar-daemon start broker
   2. 测试 list 所有 brokers： pulsar-admin --admin-url http://`[webServicePort|webServicePortTls]` brokers list `[clusterName]` 

8. 测试 pulsar

   1. 如果你是 类unix系统，比如我是 macOS，那就需要在 `etc/hosts` 文件中，把本机的 IP 和 hostname 映射关系配置好，否则会出现 java 底层调用异常，无法解析主机

   2. 由于我们使用的 pulsar版本 是 2.4.0，因此 maven 依赖如下

      ```java
      		<dependency>
      			<groupId>org.apache.pulsar</groupId>
      			<artifactId>pulsar-client</artifactId>
      			<version>2.4.0</version>
      		</dependency>
      ```

   3. 启动消费者订阅 my-topic

      ```java
      public class PulsarConsumer {
          private static String localClusterUrl = "pulsar://127.0.0.1:6653";
      
          public static void main(String[] args) {
              try {
                  //将订阅消费者指定的主题消息
                  Consumer<byte[]> consumer = getClient().newConsumer()
                          .topic("persistent://my-tenant/my-namespace/my-topic")
                          .subscriptionName("my-subscription")
                          .subscribe();
                  while (true) {
                      Message msg = consumer.receive();
                      System.out.printf("consumer-Message received: %s. \n", new String(msg.getData()));
                      // 确认消息，以便broker删除消息
                      consumer.acknowledge(msg);
                  }
              } catch (Exception e) {
                  e.printStackTrace();
              }
          }
      
          public static PulsarClient getClient() throws Exception {
              PulsarClient client;
              client = PulsarClient.builder().serviceUrl(localClusterUrl).build();
              return client;
          }
      }
      ```

      

   4. 启动生产者发送消息到 my-topic，消费者每次都可以接收消息并打印出来。但是如果消费者先不启动，生产者发送完消息后，消费者再起，那么消费者是消费不到之前的消息的，这个可能跟订阅的模式或者消息的持久化有关，后续再研究。

      ```java
      public class PulsarProducer {
          // 连接集群 broker
          private static String localClusterUrl = "pulsar://127.0.0.1:6653";
      
          public static void main(String[] args) {
              try {
                  Producer<byte[]> producer = getProducer();
                  String msg = "hello world pulsar!";
      
                  Long start = System.currentTimeMillis();
                  MessageId msgId = producer.send(msg.getBytes());
                  System.out.println("spend=" + (System.currentTimeMillis() - start) + ";send a message msgId = " + msgId.toString());
              } catch (Exception e) {
                  e.printStackTrace();
              }
          }
      
          public static Producer<byte[]> getProducer() throws Exception {
              PulsarClient client;
              client = PulsarClient.builder().serviceUrl(localClusterUrl).build();
              Producer<byte[]> producer = client.newProducer().topic("persistent://my-tenant/my-namespace/my-topic").producerName("producerName").create();
              return producer;
          }
      }
      ```

      

   这样最简单的一个pulsar单机集群就部署好了，更多特性就可以接着慢慢尝试了。





# Pulsar Functions

1. 首先来修改 `function_works.yml`

   ```yaml
   workerId: worker6751
   workerHostname: 127.0.0.1
   workerPort: 6751
   workerPortTls: 16751
   
   # Configuration Store connection string
   configurationStoreServers: 127.0.0.1:12181,127.0.0.1:12182,127.0.0.1:12183
   
   
   # pulsar topics used for function metadata management
   
   pulsarFunctionsNamespace: my-tenant/functions
   pulsarFunctionsCluster: pulsar-wyf
   functionMetadataTopicName: metadata
   clusterCoordinationTopicName: coordinate
   # Number of threads to use for HTTP requests processing. Default is set to 8
   numHttpServerThreads: 8
   ```

   

   基本上也就是指定 `functions-worker Server` 的 Id、主机地址还有端口信息，配置存放在我们之前配置的zk集群上，并且将functions任务调度的 Cluster-Namespace 指定为我们想要的，这样 assignments Topic 就会生成在指定的 namespace 下

2. 然后 启动 `functions-worker Server` ， 在bin目录下键入「 pulsar functions-worker 」，如果配置都对的上的话大概率是会生成一个 server 的 URL：

   15:15:29.113 [main] INFO org.eclipse.jetty.server.Server - Started @4076ms

   15:15:29.113 [main] INFO org.apache.pulsar.functions.worker.rest.WorkerServer - Worker Server started at **http://127.0.0.1:6751/admin**

   15:15:29.113 [main] INFO org.apache.pulsar.functions.worker.Worker - Start worker server on port 6751...

3. 然后我们就可以将编写好的 functions 代码打包，放到指定位置，并创建function:

   ```java
   import org.apache.pulsar.client.api.Producer;
   import org.apache.pulsar.client.api.PulsarClient;
   import org.apache.pulsar.client.api.PulsarClientException;
   import org.apache.pulsar.functions.api.Context;
   import org.apache.pulsar.functions.api.Function;
   import org.slf4j.Logger;
   
   import java.util.HashMap;
   import java.util.Map;
   
   /**
    * in: persistent://my-tenant/my-namespace/my-topic
    * out: persistent://tenant0/namespace0/wyf-share
    * 这是一个将 输入topic中数值类型的字符串 过滤到输出topic 的function  如果在function上配置了outTopic 就会发双倍消息
    * @author wyf
    * @date 2021/2/7 09:01
    */
   public class FunctionTest implements Function<byte[], byte[]> {
       private static Logger log;
       private static PulsarClient client;
   
       private boolean initFlag;
   
       private Producer<byte[]> numberProducer;
   
       @Override
       public byte[] process(byte[] messageIn, Context context) throws Exception {
           if (!initFlag) {
               log = context.getLogger();
               //  do init
               init(context);
               initFlag = true;
           }
           try {
               String msg = new String(messageIn);
               Long.parseLong(msg);
               // 将number消息滤出来
               numberProducer.send(messageIn);
               context.getCurrentRecord().ack();
               // 如果在创建和更新function的时候指定了output topic 只要返回值不为null 就会自动发送
               return messageIn;
           } catch (NumberFormatException e) {
               if (log.isInfoEnabled()) {
                   log.info("消息格式不合规 直接滤除");
               }
               context.getCurrentRecord().ack();
           } catch (Throwable anyThrowable) {
               context.getCurrentRecord().fail();
           }
           return null;
       }
   
       private void init(Context context) throws PulsarClientException {
           Map<String, Object> userConfigMap = context.getUserConfigMap();
           if (null != userConfigMap && !userConfigMap.isEmpty()) {
               String topicInfoString = (String) userConfigMap.get("topic_info");
               //topic=persistent://tenant0/namespace0/wyf-share;url=pulsar://127.0.0.1:6651
               String[] configArray = topicInfoString.split(";");
               HashMap<String, String> map = new HashMap<>();
               for (String entry : configArray) {
                   String[] topSimple = entry.split("=");
                   map.put(topSimple[0], topSimple[1]);
               }
               if (log.isInfoEnabled()) {
                   log.info("configMap =>\n" + map);
               }
   
               if (client == null) {
                   try {
                       client = PulsarClient.builder().serviceUrl(map.get("url")).build();
                   } catch (PulsarClientException e) {
                       log.error("init pulsar client error", e);
                   }
               }
               numberProducer = client.newProducer().topic(map.get("topic")).create();
           }
       }
   }
   ```

   对应的shell为

   ```shell
   pulsar-admin --admin-url http://127.0.0.1:6751 functions create \
     --parallelism 2 \
     --processing-guarantees 'ATLEAST_ONCE' \
     --jar '/Users/wyf/opt/test-1.0-SNAPSHOT.jar' \
     --classname 'test.pulsar.function.FunctionTest' \
     --auto-ack false \
     --inputs 'persistent://my-tenant/my-namespace/my-topic' \
     --output 'persistent://tenant0/namespace0/wyf-share' \
     --user-config '{"topic_info":"topic=persistent://tenant0/namespace0/wyf-share;url=pulsar://127.0.0.1:6651"}' \
     --tenant 'my-tenant' \
     --namespace 'functions' \
     --name 'function_test'
     
    
    
    pulsar-admin --admin-url http://127.0.0.1:6751 functions start \
     --tenant 'my-tenant' \
     --namespace 'functions' \
     --name 'function_test'
   ```

   就可以启动function啦，执行一些流处理任务

4. 可以看看日志 

   more /Users/wyf/opt/apache-pulsar-2.4.0-1/logs/functions/my-tenant/functions/function_test/function_test-1.log

   > 14:32:16.551 [pulsar-timer-4-1] INFO org.apache.pulsar.client.impl.ConsumerStatsRecorderImpl - [persistent://my-tenant/my-namespace/my-topic-partition-0] [my-tenant/functions/function_test] [5fa1b] Prefetched messages: 0 --- Consume throughput received: 0.02 msgs/s --- 0.00 Mbit/s --- Ack sent rate: 0.03 ack/s --- Failed messages: 0 --- Failed acks: 0
   >
   > 14:32:16.557 [pulsar-timer-4-1] INFO org.apache.pulsar.client.impl.ConsumerStatsRecorderImpl - [persistent://my-tenant/my-namespace/my-topic-partition-1] [my-tenant/functions/function_test] [5fa1b] Prefetched messages: 0 --- Consume throughput received: 0.02 msgs/s --- 0.00 Mbit/s --- Ack sent rate: 0.02 ack/s --- Failed messages: 0 --- Failed acks: 0
   >
   > 14:32:16.563 [pulsar-timer-4-1] INFO org.apache.pulsar.client.impl.ConsumerStatsRecorderImpl - [persistent://my-tenant/my-namespace/my-topic-partition-2] [my-tenant/functions/function_test] [5fa1b] Prefetched messages: 0 --- Consume throughput received: 0.02 msgs/s --- 0.00 Mbit/s --- Ack sent rate: 0.03 ack/s --- Failed messages: 0 --- Failed acks: 0
   >
   > 14:32:17.042 [my-tenant/functions/function_test-1] INFO function-function_test - 消息格式不合规 直接滤除
   >
   > 14:32:19.062 [my-tenant/functions/function_test-1] INFO function-function_test - 消息格式不合规 直接滤除
   >
   > 14:32:21.081 [my-tenant/functions/function_test-1] INFO function-function_test - 消息格式不合规 直接滤除
   >
   > 14:32:23.098 [my-tenant/functions/function_test-1] INFO function-function_test - 消息格式不合规 直接滤除
   >
   > 14:32:30.161 [my-tenant/functions/function_test-1] INFO function-function_test - 消息格式不合规 直接滤除
   >
   > 14:32:31.172 [my-tenant/functions/function_test-1] INFO function-function_test - 消息格式不合规 直接滤除
   >
   > 14:32:36.221 [my-tenant/functions/function_test-1] INFO function-function_test - 消息格式不合规 直接滤除

反正最终现象就是，输入topic里的偶数时间戳字符串，被发送到了输出topic里双份，这也是意料之中的结果。



其他停止、更新、删除等等，自行了解吧。