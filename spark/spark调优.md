# ����
�����ǰ����ͬ�·�����һ��spark����ִ�к�����2.5Сʱ���ϣ������Һܲ��ȶ���ʧ�ܸ�����50%�����������ֳ��ԣ�����partition�����ڴ桢vcore������û��Ч����

# ���л���
```
spark2.1.1��yarn clusterģʽ��
hadoop2.7.3
```

# ԭ��spark�������
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
join numPartitions 6000��joinʱָ��������
```

# �Ų����
## �����߼�
�������֣�����ĺ����߼��ܼ򵥣����������join������һ����Ĵ�СԼ2TB��200������¼��shuffle������5T��������һ����Ĵ�С��50G��1������¼����

## ������־
����ȥweb ui�鿴��־������web ui����ʾ����ʧ�ܵ�ԭ����
```
org.apache.spark.shuffle.FetchFailedException: Failed to connect to hadoop1301.xxx.org/10.xxx.76:7337
```
������ĳЩexecutor OOM�������޷��ӽ����fetch���ݡ����ǲ鿴��Ӧ��executor��־����û�з���OOM�������Ҷ�Ӧ��executor�������ڴ�ӡ�µ���־�����Զ�Ӧ��executor���Ǵ��״̬��û�йҵ��������ڳ��Ե���executor�ڴ�֮��������Ȼ���ڣ�Ҳ֤����OOM�Ĳ²��Ǵ���ġ�

����������־��������������־��������
```
1. TransportClientFactory: DNS resolution for hadoop2105.xxx.org/10.xxx.xxx.193:7337 took 5004 ms
2. NettyRpcEndpointRef: Error sending message [message = Heartbeat(2402,[Lscala.Tuple2;@25caf03a,BlockManagerId(2402, hadoop616.xxx.org, 26041, None))] in 1 attempts org.apache.spark.rpc.RpcTimeoutException: Futures timed out after [10 seconds]. This timeout is controlled by spark.executor.heartbeatInterval
3. No route to host: hadoop616.xxx.org
```
- �д�����executor��������ʱ�ı�����һ��Ӧ�²���driver���ڴ�/cpu/�������������⵼�¡����ǰ�yarn clientģʽ��Ϊyarn clusterģʽ����ֹdriver��executor����ͬһ���Σ�Ӱ���������ܣ�����driver�ڴ������8G�����������2G����vcores����Ϊ2������ִ�г��򡣷���������Ȼ���ڣ����ҹ۲�driver���ڻ�����cpu���ڴ桢�����أ����ֲ�û�����Ա仯��
- ��Ȼdriverû�����⣬�ǻ����ǲ���ĳЩ����������������̫�֮ࣨǰ��������������������������executor lost����SA�İ����£��鿴�˶�Ӧʱ��㲿��excutor���ڻ���������״̬����������ָ�꣨�������������ȣ�Ҳû�����Ա仯��cpu���ڴ�״̬��û���쳣��
- ��Ȼ�������־����ʾ����������صı������԰��ص����·Żػ��������ϡ���û�п�����shuffle fetch�����д���������DNS���󣬰�DNS����򱬣���ȥ��SAȷ�ϣ�SA����DNS����������
- �������ܣ���������·������

# ��������
��ȻĿǰ�Ѿ�ȷ����������ص����⣬���Ҷ�ʱ���ڲ��ܽ�һ��ȷ��ԭ���Ǿ���ʱ�ȿ��ǽ�������������޸����ߺ��ٷ��ؼ����Ų�����ԭ��
��ǰspark����ʹ����1000��executor������ÿ��executor��ֻ��һ�����ӣ���ô������Ⱥ����1000\*999�����ӡ��Բ��������µ���
```
executor-memory 16G
spark.yarn.executor.memoryOverhead 4G
spark.dynamicAllocation.maxExecutors 250
spark.executor.cores 4
```
Ҳ����˵,�����executor����С��ԭ����1/4���ܵ�vcore�����䡢�ڴ治�䡣Ŀ���Ǽ�С��Ⱥ�ڵ��������Լ�Զ��fetch������������spark.executor.vcores����һ��ͬʱexecutor�����������£�shuffle�������������һ����shuffle�������������ԭ��1/4��û������spark.reducer.maxReqsInFlight��spark.reducer.maxBlocksInFlightPerAddress��ǰ���£���
�������ִ�к󣬷��ֳ����ȶ�ִ�У�����ִ��ʱ���2.5Сʱ��С����1.3Сʱ��

# GC����
�����ڷ���spark��־�����У�����gcʱ��ռ�ȴﵽ38%
![](ͼƬ���ӵ�ַ)
��gc����Ϊg1��gcʱ���С��ԭ����13%
![](ͼƬ���ӵ�ַ)


# �ܽ�
�ھ����������š�gc���ź�spark�����ִ��ʱ���2.5Сʱ���ٵ�1Сʱ���Ӳ��ȶ����ȶ���˵��spark�����á�����������Ӱ��ܴ󣬶���spark�������ԣ�������Ҫȥ���ԡ����������ܽ�ļ���sparkʹ�á�����ԭ��
1. �ȰѴ������������ݵ���ͨ�����ٿ��ǵ���
2. �࿴��־��������־����spark����ʧ�ܡ�ִ�л�����ԭ�򣬶����ǳ���������ڴ桢����vcore
3. ��Դ��������Խ��Խ�ã�����̫����Դ���ܻᵼ�����硢cpuѹ��̫�󣬷���Ӱ���������ִ��
4. ����nodemanger���á���Ⱥ��Դ�����������spark.executor.vcores
5. ��executor gcʱ��ռ��̫�ߣ�10%���ϣ�����gcʱ��̫��ʱ����Ҫ����gc����

# TODO
1. ������������ʧ�ܵĸ���ԭ���Ƿ�����Ϊ��DNS����������ࣿ
2. �о�spark.shuffle.spill.numElementsForceSpillThreshold������shuffle��Ӱ��
3. �о�spark.reducer.maxBlocksInFlightPerAddress���߼���ʹ�÷���
4. Ŀǰ������ʹ��Ĭ�����õ�g1 gc����Ҫ�о�gc�Ż�
