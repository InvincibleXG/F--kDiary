# Prometheus 指标采集与查询

> [Prometheus](https://github.com/prometheus) 是由前 Google 工程师从 2012 年开始在 [Soundcloud](http://soundcloud.com/) 以开源软件的形式进行研发的系统监控和告警工具包，自此以后，许多公司和组织都采用了 Prometheus 作为监控告警工具。Prometheus 的开发者和用户社区非常活跃，它现在是一个独立的开源项目，可以独立于任何公司进行维护。
>
> 主要优势有：
>
> 由指标名称和和键/值对标签标识的时间序列数据组成的多维[数据模型](https://github.com/yangchuansheng/prometheus-handbook/tree/c6e1e12588ec63c20345090368b37654ef30922a/2-concepts/data_model.html)。
>
> 强大的[查询语言 PromQL](https://github.com/yangchuansheng/prometheus-handbook/tree/c6e1e12588ec63c20345090368b37654ef30922a/4-prometheus/basics.html)。
>
> 不依赖分布式存储；单个服务节点具有自治能力。
>
> 时间序列数据是服务端通过 HTTP 协议主动拉取获得的。
>
> 也可以通过中间网关来[推送时间序列数据](https://github.com/yangchuansheng/prometheus-handbook/tree/c6e1e12588ec63c20345090368b37654ef30922a/5-instrumenting/pushing.html)。
>
> 可以通过静态配置文件或服务发现来获取监控目标。
>
> 支持多种类型的图表和仪表盘。

### ***适用场景***

Prometheus在记录纯数字时间序列方面表现非常好。它既适用于面向服务器等硬件指标的监控，也适用于高动态的面向服务架构的监控。对于现在流行的微服务，Prometheus的多维度数据收集和数据筛选查询语言也是非常的强大。

Prometheus是为服务的可靠性而设计的，当服务出现故障时，它可以使你快速定位和诊断问题。它的搭建过程对硬件和服务没有很强的依赖关系。

### ***不适用场景***

Prometheus，它的价值在于可靠性，甚至在很恶劣的环境下，你都可以随时访问它和查看系统服务各种指标的统计信息。 如果你对统计数据需要100%的精确，它并不适用，例如：它不适用于实时计费系统

-----

### ***指标类型***

Prometheus客户库提供了四个核心的metrics类型。

#### ***Counter(计数器)***

**counter** 是一个累计度量指标，它是一个只能递增的数值。计数器主要用于统计服务的请求数、任务完成数和错误出现的次数等等。计数器是一个递增的值。反例：统计goroutines的数量。

#### ***Gauge(测量器)***

**gauge**是一个度量指标，它表示一个既可以递增, 又可以递减的值。

测量器主要测量类似于温度、当前内存使用量等，也可以统计当前服务运行随时增加或者减少的Goroutines数量

#### ***Histogram(柱状图)***

**histogram**是柱状图，在Prometheus系统中的查询语言中，有三种作用：

1. 对每个采样点进行统计，打到各个分类值中(bucket)

2. 对每个采样点值累计和(sum)

3. 对采样点的次数累计和(count)

度量指标名称: [basename]的柱状图, 上面三类的作用度量指标名称

[basename]_bucket{le="上边界"}, 这个值为小于等于上边界的所有采样点数量

[basename]_sum

[basename]_count

#### ***Summary******分位图***

类似**histogram**柱状图，**summary**是采样点分位图统计，(通常的使用场景：请求持续时间和响应大小)。 它也有三种作用：

1. 对于每个采样点进行统计，并形成分位图。（如：正态分布一样，统计低于60分不及格的同学比例，统计低于80分的同学比例，统计低于95分的同学比例）

2. 统计班上所有同学的总成绩(sum)

3. 统计班上同学的考试总人数(count)

带有度量指标的[basename]的summary 在抓取时间序列数据展示。

观察时间的φ-quantiles (0 ≤ φ ≤ 1), 显示为[basename]{分位数="[φ]"}

[basename]_sum， 是指所有观察值的总和

[basename]_count, 是指已观察到的事件计数值

------

###***指标采集***

使用 `push_gateway` 作为数据采集 Endpoint，在代码中构建普罗米修斯对象，并且将采集到的 Metrics 推送到 `push_gateway` 上。普罗米修斯服务端会定时从配置好的 `Targets` 中拉取数据。

一般可以通过 `http://xxx.xxx.xxx.xxx:port/mertrics` 来访问普罗米修斯或者push_gateway上存储的指标。

*以下心得摘自CSDN，建议看看。

> 在使用的过程中，还有以下小细节问题记录。Prometheus通过配置scrape_interval和evaluation_interval参数来设置向exporter采集数据的间隔时间。采集数据的间隔和数据生成的时间间隔大小不同，会对数据采集结果有影响。
>
> 3.1 采集到重复数据
> 假如采集的间隔时间小于数据生成的间隔时间，就会采集到重复的数据。
>
> 即：假设Prometheus是每5秒采集一次，但是exporter是每11秒生成一次数据，那么第一次采集成功后，第二次采集的数据还是之前的历史数据，并未更新。
>
> 3.2 丢失更新数据
> 假如采集的时间间隔大于数据生成的时间间隔，那么就会丢失更新数据。上一次的数据还未被采集，就被新的数据覆盖了。
>
> 简而言之：Prometheus每次向exporter获取数据的时候，都是获取的当前值，不会取到历史数据。
>
> 同样的：在使用pushgetway做中转的时候，也有同样的问题，并不能获取到历史数据，或者避免重复数据。
>
> 3.3 时区问题
> 通过prometheus的UI查询界面查询的到数据在查看graph的时候会发现时间对不上，经过观察比对，会发现时间永远是比当前时间小8个小时。检查Prometheus所在主机的时区以及业务bis_exporter的主机的时区以及时间都相同，但是查看的时候时间数据时间对不上，不过，最怪异的是通过Grafana展示的数据曲线却是正确的，时间也正确。
>
> 最后经过搜索发现，Prometheus为了保证时序的准确性，内部所有组件都是会用UTC时间，查了下出的日志文件时间也是少8个小时，这就对的上了。
> ————————————————
> 版权声明：本文为CSDN博主「凉茶冰」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
> 原文链接：https://blog.csdn.net/liangcha007/java/article/details/88558389

###***指标查询***

普罗米修斯默认会暴露 `RESTful接口`  「IP:Port/api/v1/query?query={JSON}」

在命令行中查询，如 `curl -X POST -g 'http://127.0.0.1:9090/api/v1/query?query={name="hello"}[5m]'`

具体如何使用，需要了解其查询语言

*以下内容引用自简书

> 作者：小孩真笨
> 链接：https://www.jianshu.com/p/3bdc4cfa08da
> 来源：简书
> 著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

「这边贴过来方便学习」

## Prometheus 查询语言

PromQL（Prometheus Query Language）是 Prometheus 自己开发的表达式语言，语言表现力很丰富，内置函数也很多。使用它可以对时序数据进行筛选和聚合。

### 1. PromQL 语法

#### 1.1 数据类型

PromQL 表达式计算出来的值有以下几种类型：

- 瞬时向量 (Instant vector): 一组时序，每个时序只有一个采样值
- 区间向量 (Range vector): 一组时序，每个时序包含一段时间内的多个采样值
- 标量数据 (Scalar): 一个浮点数
- 字符串 (String): 一个字符串，暂时未用

#### 1.2 时序选择器

##### 1.2.1 瞬时向量选择器

瞬时向量选择器用来选择**一组时序在某个采样点的采样值**。

最简单的情况就是指定一个度量指标，选择出所有属于该度量指标的时序的当前采样值。比如下面的表达式：

```undefined
http_requests_total
```

可以通过在后面添加用大括号包围起来的一组标签键值对来对时序进行过滤。比如下面的表达式筛选出了 `job` 为 `prometheus`，并且 `group` 为 `canary` 的时序：

```csharp
http_requests_total{job="prometheus", group="canary"}
```

匹配标签值时可以是**等于**，也可以**使用正则表达式**。总共有下面几种匹配操作符：

- `=`：完全相等
- `!=`： 不相等
- `=~`： 正则表达式匹配
- `!~`： 正则表达式不匹配

下面的表达式筛选出了 environment 为 staging 或 testing 或 development，并且 method 不是 GET 的时序：

```bash
http_requests_total{environment=~"staging|testing|development",method!="GET"}
```

度量指标名可以使用内部标签 `__name__` 来匹配，表达式 `http_requests_total` 也可以写成 `{__name__="http_requests_total"}`。表达式 `{__name__=~"job:.*"}` 匹配所有度量指标名称以 `job:` 打头的时序。

##### 1.2.2 区间向量选择器

区间向量选择器类似于瞬时向量选择器，不同的是它选择的是**过去一段时间的采样值**。可以通过在瞬时向量选择器后面添加包含在 `[]` 里的时长来得到区间向量选择器。比如下面的表达式选出了所有度量指标为 `http_requests_total` 且 `job` 为 `prometheus` 的时序在过去 5 分钟的采样值。

```bash
http_requests_total{job="prometheus"}[5m]
```

**说明：时长的单位可以是下面几种之一：**

- s：seconds
- m：minutes
- h：hours
- d：days
- w：weeks
- y：years

##### 1.2.3 偏移修饰器

前面介绍的选择器默认都是以当前时间为基准时间，偏移修饰器用来调整基准时间，使其往前偏移一段时间。偏移修饰器紧跟在选择器后面，使用 offset 来指定要偏移的量。比如下面的表达式选择度量名称为 `http_requests_total` 的所有时序在 `5` 分钟前的采样值。

```undefined
http_requests_total offset 5m
```

下面的表达式选择 `http_requests_total` 度量指标在 `1` 周前的这个时间点过去 5 分钟的采样值。

```css
http_requests_total[5m] offset 1w
```

### 2. PromQL 操作符

#### 2.1 二元操作符

PromQL 的二元操作符支持基本的逻辑和算术运算，包含**算术类**、**比较类**和**逻辑类**三大类。

##### 2.1.1 算术类二元操作符

算术类二元操作符有以下几种：

- `+`：加
- `-`：减
- `*`：乘
- `/`：除
- `%`：求余
- `^`：乘方

算术类二元操作符可以使用在标量与标量、向量与标量，以及向量与向量之间。

**二元操作符上下文里的向量特指瞬时向量，不包括区间向量。**

- 标量与标量之间，结果很明显，跟通常的算术运算一致。
- 向量与标量之间，相当于把标量跟向量里的每一个标量进行运算，这些计算结果组成了一个新的向量。
- 向量与向量之间，会稍微麻烦一些。运算的时候首先会为左边向量里的每一个元素在右边向量里去寻找一个匹配元素（匹配规则后面会讲），然后对这两个匹配元素执行计算，这样每对匹配元素的计算结果组成了一个新的向量。如果没有找到匹配元素，则该元素丢弃。

##### 2.1.2 比较类二元操作符

比较类二元操作符有以下几种：

- `==` (equal)
- `!=` (not-equal)
- `>` (greater-than)
- `<` (less-than)
- `>=` (greater-or-equal)
- `<=` (less-or-equal)

比较类二元操作符同样可以使用在标量与标量、向量与标量，以及向量与向量之间。默认执行的是过滤，也就是保留值。

也可以通过在运算符后面跟 bool 修饰符来使得返回值 0 和 1，而不是过滤。

- 标量与标量之间，必须跟 bool 修饰符，因此结果只可能是 0（false） 或 1（true）。
- 向量与标量之间，相当于把向量里的每一个标量跟标量进行比较，结果为真则保留，否则丢弃。如果后面跟了 bool 修饰符，则结果分别为 1 和 0。
- 向量与向量之间，运算过程类似于算术类操作符，只不过如果比较结果为真则保留左边的值（包括度量指标和标签这些属性），否则丢弃，没找到匹配也是丢弃。如果后面跟了 bool 修饰符，则保留和丢弃时结果相应为 1 和 0。

##### 2.1.3 逻辑类二元操作符

逻辑操作符仅用于向量与向量之间。

- `and`：交集
- `or`：合集
- `unless`：补集

具体运算规则如下：

- `vector1 and vector2` 的结果由在 vector2 里有匹配（标签键值对组合相同）元素的 vector1 里的元素组成。
- `vector1 or vector2` 的结果由所有 vector1 里的元素加上在 vector1 里没有匹配（标签键值对组合相同）元素的 vector2 里的元素组成。
- `vector1 unless vector2` 的结果由在 vector2 里没有匹配（标签键值对组合相同）元素的 vector1 里的元素组成。

##### 2.1.4 二元操作符优先级

PromQL 的各类二元操作符运算优先级如下：

1. `^`
2. `*, /, %`
3. `+, -`
4. `==, !=, <=, <, >=, >`
5. `and, unless`
6. `or`

#### 2.2 向量匹配

前面算术类和比较类操作符都需要在向量之间进行匹配。共有两种匹配类型，`one-to-one` 和 `many-to-one` / `one-to-many`。

##### 2.2.1 One-to-one 向量匹配

这种匹配模式下，两边向量里的元素如果其标签键值对组合相同则为匹配，并且只会有一个匹配元素。可以使用 `ignoring` 关键词来忽略不参与匹配的标签，或者使用 `on` 关键词来指定要参与匹配的标签。语法如下：

```xml
<vector expr> <bin-op> ignoring(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) <vector expr>
```

比如对于下面的输入：

```bash
method_code:http_errors:rate5m{method="get", code="500"}  24
method_code:http_errors:rate5m{method="get", code="404"}  30
method_code:http_errors:rate5m{method="put", code="501"}  3
method_code:http_errors:rate5m{method="post", code="500"} 6
method_code:http_errors:rate5m{method="post", code="404"} 21

method:http_requests:rate5m{method="get"}  600
method:http_requests:rate5m{method="del"}  34
method:http_requests:rate5m{method="post"} 120
```

执行下面的查询：

```bash
method_code:http_errors:rate5m{code="500"} / ignoring(code) method:http_requests:rate5m
```

得到的结果为：

```cpp
{method="get"}  0.04            //  24 / 600
{method="post"} 0.05            //   6 / 120
```

也就是每一种 method 里 code 为 500 的请求数占总数的百分比。由于 method 为 put 和 del 的没有匹配元素所以没有出现在结果里。

##### 2.2.2 Many-to-one / one-to-many 向量匹配

这种匹配模式下，某一边会有多个元素跟另一边的元素匹配。这时就需要使用 `group_left` 或 `group_right` 组修饰符来指明哪边匹配元素较多，左边多则用 `group_left`，右边多则用 `group_right`。其语法如下：

```xml
<vector expr> <bin-op> ignoring(<label list>) group_left(<label list>) <vector expr>
<vector expr> <bin-op> ignoring(<label list>) group_right(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) group_left(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) group_right(<label list>) <vector expr>
```

**组修饰符只适用于算术类和比较类操作符。**

对于前面的输入，执行下面的查询：

```undefined
method_code:http_errors:rate5m / ignoring(code) group_left method:http_requests:rate5m
```

将得到下面的结果：

```cpp
{method="get", code="500"}  0.04            //  24 / 600
{method="get", code="404"}  0.05            //  30 / 600
{method="post", code="500"} 0.05            //   6 / 120
{method="post", code="404"} 0.175           //  21 / 120
```

也就是每种 method 的每种 code 错误次数占每种 method 请求数的比例。这里匹配的时候 ignoring 了 code，才使得两边可以形成 Many-to-one 形式的匹配。由于左边多，所以需要使用 group_left 来指明。

Many-to-one / one-to-many 过于高级和复杂，要尽量避免使用。很多时候通过 ignoring 就可以解决问题。

#### 2.3 聚合操作符

PromQL 的聚合操作符用来将向量里的元素聚合得更少。总共有下面这些聚合操作符：

- sum：求和
- min：最小值
- max：最大值
- avg：平均值
- stddev：标准差
- stdvar：方差
- count：元素个数
- count_values：等于某值的元素个数
- bottomk：最小的 k 个元素
- topk：最大的 k 个元素
- quantile：分位数

聚合操作符语法如下：

```xml
<aggr-op>([parameter,] <vector expression>) [without|by (<label list>)]
```

其中 `without` 用来指定不需要保留的标签（也就是这些标签的多个值会被聚合），而 `by` 正好相反，用来指定需要保留的标签（也就是按这些标签来聚合）。

下面来看几个示例：

```undefined
sum(http_requests_total) without (instance)
```

http_requests_total 度量指标带有 application、instance 和 group 三个标签。上面的表达式会得到每个 application 的每个 group 在所有 instance 上的请求总数。效果等同于下面的表达式：

```csharp
sum(http_requests_total) by (application, group)
```

下面的表达式可以得到所有 application 的所有 group 的所有 instance 的请求总数。

```undefined
sum(http_requests_total)
```

### 3. 函数

Prometheus 内置了一些函数来辅助计算，下面介绍一些典型的。

- abs()：绝对值
- sqrt()：平方根
- exp()：指数计算
- ln()：自然对数
- ceil()：向上取整
- floor()：向下取整
- round()：四舍五入取整
- delta()：计算区间向量里每一个时序第一个和最后一个的差值
- sort()：排序



