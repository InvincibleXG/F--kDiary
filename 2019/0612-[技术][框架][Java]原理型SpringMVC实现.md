# 手撸 简易Spring MVC
----
`Spring`可以说是当今 `J2EE` 开发最为流行的框架了，它代表了一种高级设计模式的编程思想。由于某个契机，本人也有幸学习了一下框架的简单原理，并进行了实现，在这里分享下一些感想和心得，希望能对你有所启发。

先从基础的`servlet`编程说起吧，作为低等后端开发者，主要编写的是业务逻辑代码，也就是应用层代码。一个一般的 `J2EE` 的`请求/响应`过程是：

1. 请求`request`到达服务器，这时候可能就已经生成了对应的`response`对象
2. 服务器根据路由，找到对应的`servlet`来处理这个请求
3. 一般都是以流的形式把处理结果封装了写入`response`对象，响应返回

这里面就会涉及几个明显的问题：

1. `servlet`编程针对不同的路由请求，需要编写大量不同的`servlet方法`进行响应，如果不以某种形式的解耦将这些业务逻辑处理的方法分离开，代码会显得非常臃肿；如果使用`web.xml`配置，很繁琐
2. 无论是`servlet`还是所依赖的其他`java bean`，如果没有良好的设计模式来解耦，会重复大量地申请内存或者引起一些潜在的线程安全问题，性能上会大大受影响
3. 很难进行优雅的编程，尤其是某些适用`AOP`的场景，会出现大量重复代码，维护也增加了难度

因此，Spring 类 的框架，主要就是解决这几点，我们也可以针对这几点去理解并手撸出我们自己的 SpringMVC 框架。

----

而实现思路和过程就是:

1. 以 `EmbededTomcat` 来构建一个内嵌的服务器( SpringBoot 就是这么干的)，项目启动时，启动 Tomcat 实例并为其添加 `DispatcherServlet` 用于分发请求给其他的 `servlet(controller method)`
2. `DispatcherServlet` 实现 `Servlet interface` ，其 service 方法 的逻辑就是根据已经注册的`requestMapping`，按路由匹配请求与控制器方法(也可以叫 servlet或action )，实现请求分发
3. 创建自定义的控制器注解 @Controller、路由注解 @RequestMapping，在项目启动时，通过启动类的包名让类加载器扫描到所有的类
4. 遍历所有的类，将被 @Controller 注解的类拎出来，反射遍历其 `Method`，如果方法被 @RequestMapping注解了，就获取注解配置的 value，生成一个保存 controller action 的对象，暂称作 `MappingHandler`
5. 每次请求分发时，遍历每个 `MappingHandler`，如果请求的 URI 与此 handler 匹配，那么就将请求的参数传入给反射方法 `method.invoke(controllerObject, parameters)`，执行结果写入 response 流，响应此请求
6. 目前已经实现好了 servlet 的“请求分发——处理响应”，接下来考虑到业务层级是复杂的，因此 controller 会依赖各种 service，而 service 又有其中的 bean 依赖，故我们需要实现“控制反转/依赖注入”
7. 创建自定义注解 @Bean 和自动装备注解 @Autowired，在 `bean` 上使用 @Bean，使用引用声明时在字段上使用 @Autowired
8. 创建 `BeanFactory` 对象，内嵌 Tomcat 启动后以工厂模式创建好所有的 `bean`
9. 在 `第4步` 遍历所有类时，通过 @Bean 注解获取 `bean` 类，工厂将以反射构造的方式生成对象，并存入单例的缓存 `Map<Class<?>, Object>` 中，以供后续直接取用，这里简化了将所有类作为单例模式，而实际 Spring 是以 BeanName 作为 key 的；对于此类中的 `Field` ，如果被 @Autowired 注解了，那么就从缓存中获取生成好的 `bean`，注入即可
10. 但是逻辑思维完备的小fo伴会想到，这里面可能产生循环依赖： A依赖B，B依赖A ，那么可能创建A对象时，B对象还未生成，这种情况Spring是使用了三级缓存机制来解决的，**文末详述**
11. 到这里，我们的流程**已经实现**了解决`问题1`和`问题2`的 MiniSpringMVC ，至于 AOP 如何实现，等我有空了再详细了解


> 这三级缓存分别指： 
> Map<String, ObjectFactory<?>> singletonFactories ： 下层-单例对象工厂的cache 
> Map<String, Object> earlySingletonObjects ： 中层-提前曝光的单例对象的cache 
> Map<String, Object> singletonObjects ： 上层-单例对象的cache

> 让我们来分析一下 “A 的某个 field 是 B 的实例对象，同时 B 的某个 field 是 A 的实例对象” 这种循环依赖的情况。A 首先完成了初始化的第一步(被工厂创建出来)，并且将自己提前曝光到 singletonFactories 的ObjectFactory<A\> 中，此时进行初始化的第二步，发现自己依赖对象 B，此时就尝试去 get(B)，发现 B 还没有被 create，所以走create流程，B 在初始化第二步的时候发现自己依赖了对象 A，于是尝试 get(A)，尝试一级缓存 singletonObjects (肯定没有，因为A还没初始化完全)，尝试二级缓存 earlySingletonObjects（也没有），尝试三级缓存 singletonFactories ，由于 A 被 ObjectFactory 生产出来后曝光到 singletonFactories，所以此时能够通过 ObjectFactory.getObject 拿到 A 对象，B 拿到 A 对象后顺利完成了初始化阶段 2、3 ，完全初始化之后将自己放入到 一级缓存 singletonObjects 中。此时返回 A 中，A 此时能拿到 B 的对象顺利完成自己的初始化阶段 2、3，最终 A 也完成了完整的创建，存入一级缓存singletonObjects 中。 因为 B 拿到了 A 的对象引用，所以 B 中注入的 A 对象当然也完成了完整的初始化。 


#### 项目源码可以到 https://github.com/InvincibleXG/mini-spring 中查看