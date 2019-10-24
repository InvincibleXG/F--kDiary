# Dubbo 学习之旅

[Dubbo-Demo Repository](https://github.com/InvincibleXG/Dubbo-Demo)

终于有幸接触到鼎鼎大名的分布式RPC框架 `Dubbo` 啦！我也建立了一个 Repository for Dubbo-Demo Project 去记录我学习过程中写下的代码并方便以后使用。此文作为参考随记，会记下一些值得注意的 Point ！



1. Repo 中的 Dubbo-Demo 项目中的 `Provider` 模块「Dubbo-Service」使用的依赖版本为 **2.5.3**，集成的 Spring 版本为 **3.2.4**，并且从 Dubbo 依赖中 **抠出了自带的 Spring** 。 **由于Dubbo的注解扫描会与Spring的AOP冲突（源码决定）** ，因此Dubbo使用 「xml文件」 进行服务配置管理。
2. Dubbo 官方给出的启动方式 Main.main(); 需要将配置文件放置于默认的 **resources/META-INF/spring/** 目录下。
3. 注册中心 `Registry` 使用 Zookeeper ，使用时应当先保证 **Zookeeper 启动** 。如果Zookeeper 没有起将导致服务无法注册，那么 「Dubbo 启动会陷入死循环」一直请求注册。(Zookeeper可以只配置一个，启动会报一个ERROR但仍然可 提供服务，client port 设为 **2181**)
4. Dubbo 的 `Consumer` 由于不必让Dubbo代理接口（不编写DAO直接调用Provider），因此可以使用Spring的相关注解。 `Consumer` 在 pom 中需要依赖相关的 `Provider` 模块，并且使用时对声明的 「Provider接口对象」 使用 **@Reference** 注解( 这样spring容器启动时不会注入此dubbo依赖，而是通过RPC方式进行调用 )。
5. `Monitor` 可使用 Dubbo-Admin.war 不过由于一些以来原因我没有试验成功，另外也有 ZookeeperInspector.jar 可查看注册中心中的可用服务。
6. 关于 `Consumer` 与 `Provider` 的RPC调用，由于使用 `dubbo协议` 因此要保证可以在 `Registry` 中发现 `Provider` 提供的服务，理所当然， `Consumer` 与 `Provider` 中都要配正确的 zookeeper URL。这样消费者就可以远程依赖于生产者，从而 **将DAO与事务操作解耦给生产者** ，消费者的业务逻辑可以只进行数据的处理，而不需要跟持久层交互了。
7.  **发布方式**：`Registry` 的发布方式我这边了解的就是 「Zookeeper集群」，当然我在测试 Demo 时并没有做任何复杂配置，如有兴趣可自行了解；`Consumer` 可以就是标准的 J2EE 项目的部署方式，发布到常见 Java Web Container 中；而这边 `Provider` 的发布方式有一点复杂，需要依赖于 「Assembly 插件」，插件配置的获取渠道也是特别暧昧了，百度上很难找到，而我正好整理齐了放在 这「Dubbo-Demo的Repo」 里了，一旦配置好 Assembly，对 `Provider` 模块 maven install 即可自动生成发布包( 一个jar文件和一个tar.gz压缩包 )，解压 tar.gz 后，可以看到 [dubbo-service-impl-1.0-SNAPSHOT](https://github.com/InvincibleXG/Dubbo-Demo/tree/master/dubbo-service-impl-1.0-SNAPSHOT) 类似的目录结构，刚开始可能会没有 log 目录，咱们直接启动 /bin/startup.sh(或bat)，如果 /conf 目录下配置没问题的话( 我自己这边清空了 dubbo.properties 的配置，因为实际使用的是默认位置下的 xml )，就可以直接在控制台或者 /logs/stdout.log 中看到 「Dubbo service server started!」 说明 `Provider` 启动成功了。这样再启动 `Consumer` 后就可以正确调用到 `Provider` 的服务。