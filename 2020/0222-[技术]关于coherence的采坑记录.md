# Oracle Coherence 遇坑记录

其实想真正认识和熟悉一个技术，必须成体系地去学习和实践，最起码要有能回答上三个问题：

1. 该技术是什么？
2. 为什么产生，能解决什么问题，其在这块的领域的特点是什么？
3. 如何正确使用？

-----

1. `Oracle Coherence`是一款基于`Java`的分布式缓存和内存数据网格。适用于需要高可用性，高可扩展性和低延迟的系统
2. 作用就是分布式缓存，特点是坑
3. 「 long story that I cannot 」

-----

记录下一些心得：

1. coherence 是 有 client - server 架构的，所以修改配置的时候要注意同步多端，有时候会忽略客户端的配置。
2. coherence 启动时是可以通过命令参数指定网卡的，如果连不上网格要检查下网卡是否选对。
3. 常见的coherence持久化要改的配置文件有：
   1. config/pof-config-domain.xml  用于配置 user-type bean 「EP POJO」
   2. configext/pof-config-proxy.xml & configext/pof-config-XXXX.xml 「CacheStore Bean」 一般都是开代理模式
   3. configext/pof-config-client-proxy.xml 「配置服务节点IP」





另可参考 [仇老师的博客](https://www.qiubinren.com/categories/coherence/)

