# Dubbo RPC调用的 timeout 机制

> ```xml
> <dubbo:service interface="xxx" ref="xxx" protocol="rest" timeout="2000" connections="10"/>
> ```
>
> ```xml
> <dubbo:reference id="xxx" interface="xxx" timeout="2000" connections="10"/>
> ```

其实在 dubbo 2.7 的配置中，对于 provider 和 consumer 都有一个 timeout 属性。那么它们在 RPC调用 中是如何生效的呢？

首先要明确一点，服务对于timeout的发生，是后验式的，必须等代码执行完毕后，检测到发生了超时，这个时候才会抛出TimeoutException。

并且，默认retry次数为2次，算上超时时间，其实最多客户端会等待 `maxRetryTimes * timeout ` 的时间才会收到服务端的返回。



而 provider.timeout 与 consumer.timeout 是如何发挥作用、他们的优先级是如何的呢？

> However, we generally recommend configuring the service provider to provide such a configuration. According to the official dubbo documentation, “Provider should configure the properties of the Consumer side as much as possible. Let the Provider implementer think about the service features and service quality of the Provider from the beginning.”

1. 当服务端 provider.timeout 没有配置时，客户端的 consumer.timeout 会通过拦截器生效

2. 当服务端 provider.timeout 配置了，而客户端没有配置，则理所应当 provider.timeout 生效

3. 当服务端 provider.timeout 配置了，并且 客户端的 consumer.timeout 也配置了，则 provider.timeout 优先生效。

   

   参考官方文档的说明，意思是 dubbo provider 会对方法的性能问题更加有所把握，所以 provider.timeout 优先生效。

