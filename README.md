Partial Key Grouping实现
===

An implementation and example of Partial Key Grouping for Apache Storm.
Partial Key Grouping is a load balancing strategy for distributed stream processing systems.
**Partial Key grouping** : 和**Fields Grouping**类似，多了load balance，解决data skew **`数据倾斜`**的问题。
## 适用场景
在使用 **Fields Grouping** 的时候，按照字段 `Fields` 进行分发，若data中存在某些`key`值数据有大量的倾斜，会导致某个 **task** 上的数据处理量十分多，造成处理效率降低。这就是数据倾斜。
**数据倾斜** 具体在手机阅读BI数据中的表现：
- 存在大量电话号码是 *000000* 的话单数据。如果按照**Fields Grouping**取`MSISDN`进行分发数据，就会出现**data skew**。
- 某个省份的话单数据远远多于其它省份，比如北京的data 远远多于西藏等偏远地区。如果按照**Fields Grouping**取`ProvinceID`分发数据，就会出现**data skew**。

## 数据倾斜表现
- 在UI界面可以发现某个**task**处理的数据量远远大于其它的**task**。影响了Storm的处理速度。

## 解决办法
**Partial Key Grouping** ： 类似**Fields grouping** , 同时具有 load balanced between two downstream bolts，有效的解决了输入数据的倾斜问题.   (这篇[paper](https://melmeric.files.wordpress.com/2014/11/the-power-of-both-choices-practical-load-balancing-for-distributed-stream-processing-engines.pdf) 解释了它的工作原理和优点。)

## 实现原理
[yahoo Lab](https://github.com/HQebupt/partial-key-grouping) 实现了一个简单的示例。
- 实现 `CustomStreamGrouping`, `Serializable`来自定义流的分组策略
- code line 28 - 38 是核心代码。采用了2个不同的 `HashFunction` 来对第0个`Fields`进行Hash的分发，数据倾斜的数据会比较均衡的分发到2个`boltIds`(Task)里面，这样不会出现数据倾斜。
> 注意：相同的数据不是说完全均衡的分发到2个`boltIds`,看代码就容易明白。有一个很重要的`long`型数组记录每个`boltIds`当前处理的`tuple`个数，算法会保证每个**boltIds**之间的数据相对均衡，但是不是我们理解的**shuffle** 的全均衡方式。（若不明白，请看代码实现）

## 使用
自定义流分组策略，如何使用呢？
由于采用**PKG** 会出现相同的`key`会出现在2个`task`里面，所以我们需要一个**聚合bolt**来将这些数据给聚合起来。可以参考[示例WordCount](https://github.com/HQebupt/partial-key-grouping/blob/master/src/main/java/com/yahoo/labs/slb/WordCountPartialKeyGrouping.java)。

## 改进
这个简单的实现，可以对自己的应用定制化的改进数据倾斜。如果想对这个组件进行更好的通用化设计，要和**Fields Grouping**的接口方式看齐，就是说可以传入`Fields`参数。代码对比见分晓:
``` java
builder.setBolt("counter", new CounterBolt(), 10).customGrouping("split", new PartialKeyGrouping());
builder.setBolt("aggregator", new AggregatorBolt(), 1).fieldsGrouping("counter", new Fields("word"));
```