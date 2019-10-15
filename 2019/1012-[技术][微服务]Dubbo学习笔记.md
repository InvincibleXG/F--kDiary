# Dubbo 学习之旅

[Dubbo-Demo Repository](https://github.com/InvincibleXG/Dubbo-Demo)

终于有幸接触到鼎鼎大名的分布式RPC框架 `Dubbo` 啦！我也建立了一个 Repository for Dubbo-Demo Project 去记录我学习过程中写下的代码并方便以后使用。此文作为参考随记，会记下一些值得注意的 Point ！



1. Repo 中的 Dubbo-Demo 项目中的 `Provider` 模块「Dubbo-Service」使用的依赖版本为 **2.5.3**，集成的 Spring 版本为 **3.2.4**，并且从 Dubbo 依赖中 **抠出了自带的 Spring** 。 **由于Dubbo的注解扫描会与Spring的AOP冲突（源码决定）** ，因此Dubbo使用 「xml文件」 进行服务配置管理。

2. Dubbo 官方给出的启动方式 Main.main(); 需要将配置文件放置于默认的 **resources/META-INF/spring/** 目录下。
3. 注册中心 `Registry` 使用 Zookeeper ，使用时应当先保证 **Zookeeper 启动** 。如果Zookeeper 没有起将导致服务无法注册，那么 「Dubbo 启动会陷入死循环」一直请求注册。(Zookeeper可以只配置一个，启动会报一个ERROR但仍然可 提供服务，client port 设为 **2181**)
4. Dubbo 的 `Consumer` 由于不必让Dubbo代理接口（不编写DAO直接调用Provider），因此可以使用Spring的相关注解。 `Consumer` 在 pom 中需要依赖相关的 `Provider` 模块，并且使用时对声明的 「Provider接口对象」 使用 **@Reference** 注解。
5. `Monitor` 可使用 Dubbo-Admin.war 不过由于一些以来原因我没有试验成功，另外也有 ZookeeperInspector.jar 可查看注册中心中的可用服务。