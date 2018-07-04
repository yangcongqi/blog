# 背景
半个月前遇到同事反馈，一个spark程序执行很慢（2.5小时以上），并且很不稳定（失败概率有50%），经过各种尝试，增加partition数、内存、vcore数，都没有效果。

# 运行环境
```
spark2.1.1（yarn cluster模式）
hadoop2.7.3
```

# 原有spark程序参数
```
spark.dynamicAllocation.enabled true
spark.shuffle.service.enabled true
driver-memory 4G
executor-memory 4G
spark.yarn.executor.memoryOverhead 1G
spark.dynamicAllocation.maxExecutors 1000
spark.executor.cores 1
conf spark.shuffle.io.maxRetries 3
conf spark.shuffle.io.retryWait 5s
conf spark.kryo.registrationRequired false
join numPartitions 6000（join时指定参数）
```

# 排查过程
## 代码逻辑
分析发现，代码的核心逻辑很简单，是两个表的join。其中一个表的大小约2TB（200亿条记录，shuffle数据量5T），另外一个表的大小是50G（1亿条记录）。

## 分析日志
首先去web ui查看日志，发现web ui上显示任务失败的原因是
```
org.apache.spark.shuffle.FetchFailedException: Failed to connect to hadoop1301.xxx.org/10.xxx.76:7337
```
怀疑是某些executor OOM，导致无法从结点上fetch数据。但是查看对应的executor日志，并没有发现OOM报错，并且对应的executor后续还在打印新的日志，所以对应的executor还是存活状态并没有挂掉。另外在尝试调大executor内存之后问题依然存在，也证明了OOM的猜测是错误的。

继续分析日志，发现有三种日志大量出现
```
1. TransportClientFactory: DNS resolution for hadoop2105.xxx.org/10.xxx.xxx.193:7337 took 5004 ms
2. NettyRpcEndpointRef: Error sending message [message = Heartbeat(2402,[Lscala.Tuple2;@25caf03a,BlockManagerId(2402, hadoop616.xxx.org, 26041, None))] in 1 attempts org.apache.spark.rpc.RpcTimeoutException: Futures timed out after [10 seconds]. This timeout is controlled by spark.executor.heartbeatInterval
3. No route to host: hadoop616.xxx.org
```
- 有大量的executor有心跳超时的报错，第一反应猜测是driver的内存/cpu/网络配置有问题导致。于是把yarn client模式改为yarn cluster模式（防止driver与executor不在同一网段，影响网络性能）、把driver内存调整到8G（正常情况下2G）、vcores设置为2，重新执行程序。发现问题依然存在，并且观察driver所在机器的cpu、内存、网络监控，发现并没有明显变化。
- 既然driver没有问题，那怀疑是不是某些机器创建的连接数太多（之前有遇到过）把网卡打满，导致executor lost？在SA的帮助下，查看了对应时间点部分excutor所在机器的网卡状态，发现网卡指标（连接数、流量等）也没有明显变化。cpu、内存状态都没有异常。
- 既然上面的日志中显示的是网络相关的报错，所以把重点重新放回基础网络上。有没有可能是shuffle fetch过程中创建大量的DNS请求，把DNS服务打爆？再去跟SA确认，SA反馈DNS服务正常。
- 其它可能：交换机、路由器？

# 参数调优
既然目前已经确定是网络相关的问题，并且短时间内不能进一步确认原因。那就暂时先考虑解决方案，问题修复上线后再返回继续排查问题原因。
当前spark程序使用了1000个executor，假如每个executor上只有一个连接，那么整个集群会有1000\*999个连接。对参数做以下调整
```
executor-memory 16G
spark.yarn.executor.memoryOverhead 4G
spark.dynamicAllocation.maxExecutors 250
spark.executor.cores 4
```
也就是说,把最大executor数减小到原来的1/4、总的vcore数不变、内存不变。目的是减小集群内的连接数以及远端fetch的数据量。在spark.executor.vcores增加一倍同时executor数减半的情况下，shuffle数据量最多会减少一半且shuffle连接数会减少至原来1/4（没有配置spark.reducer.maxReqsInFlight及spark.reducer.maxBlocksInFlightPerAddress的前提下）。
经过多次执行后，发现程序稳定执行，并且执行时间从2.5小时减小到了1.3小时。

# GC调优
另外在分析spark日志过程中，发现gc时间占比达到38%
![](图片链接地址)
把gc设置为g1后，gc时间减小到原来的13%
![](图片链接地址)


# 总结
在经过参数调优、gc调优后，spark程序的执行时间从2.5小时减少到1小时，从不稳定到稳定。说明spark的配置、参数对性能影响很大，对于spark开发而言，我们需要去尝试。以下是我总结的几个spark使用、调优原则：
1. 先把代码在少量数据调试通过，再考虑调优
2. 多看日志，根据日志分析spark任务失败、执行缓慢的原因，而不是出错就增加内存、增加vcore
3. 资源不是申请越多越好，申请太多资源可能会导致网络、cpu压力太大，反而影响程序正常执行
4. 根据nodemanger配置、集群资源配置情况设置spark.executor.vcores
5. 当executor gc时间占比太高（10%以上）或总gc时间太长时，需要考虑gc调优

# TODO
1. 分析本次任务失败的根本原因，是否是因为对DNS服务请求过多？
2. 研究spark.shuffle.spill.numElementsForceSpillThreshold参数对shuffle的影响
3. 研究spark.reducer.maxBlocksInFlightPerAddress的逻辑、使用方法
4. 目前仅仅是使用默认配置的g1 gc，需要研究gc优化
