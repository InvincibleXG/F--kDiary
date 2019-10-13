# Dubbo 学习之旅

[Dubbo-Demo Repository](https://github.com/InvincibleXG/Dubbo-Demo)

终于有幸接触到鼎鼎大名的分布式RPC框架 `Dubbo` 啦！我也建立了一个 Repository for Dubbo-Demo Project 去记录我学习过程中写下的代码并方便以后使用。此文作为参考随记，会记下一些值得注意的 Point ！



1. Repo 中的 Dubbo-Demo 项目使用的版本为 **2.5.3**，集成的 Spring 版本为 **3.2.4**，并且从 Dubbo 依赖中抠出了自带的 Spring。 **由于Dubbo的注解扫描会与Spring的AOP冲突（源码决定），因此Dubbo使用xml文件进行服务配置管理**。

2. Dubbo 官方给出的启动方式 Main.main(); 需要将配置文件放置于默认的 **resources/META-INF/spring/** 目录下。
3. 注册中心使用 Zookeeper ，如果 Zookeeper 没有起导致服务无法注册，那么 Dubbo 启动会陷入死循环一直请求注册。