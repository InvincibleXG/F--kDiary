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

   1. 配好以后现在本机任意一个 bk 的 bin 目录下初始化集群元数据 `bookkeeper shell metaformat`  「zk重启的话也就是bk第一次启动都要初始化集群元数据」
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

