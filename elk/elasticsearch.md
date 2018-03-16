https://www.elastic.co/

### words
```
indices(index的复数形式)
elect n. 被选的人；特殊阶层；上帝的选民 adj. 选出的；当选的；卓越的 vi. 作出选择；进行选举 vt. 选举；选择；推选
loopback addresse 回送地址 127.0.0.1 访问这个地址会吧消息回送到自己主机
rotate vt.& vi.（使某物）旋转;使转动;使轮流，轮换;交替
sufficient adj.足够的;充足的;充分的
```

### 问题
> Collector [cluster-stats-collector] timed out when collecting data

```
可能需要用 -XX:+UseSerialGC
而不是 -XX:+UseParallelGC 
反正我是 jvm.options 中加了-XX:+UseSerialGC后就OK了
```

[elastic官网的文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/_use_serial_collector_check.html)

[串行，并行的GC介绍](http://blog.csdn.net/macyang/article/details/8731313)

[介绍虚拟内存](http://blog.csdn.net/tysforwork/article/details/56671701)

[也可能是需要关闭虚拟内存的swap设置](https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-configuration-memory.html#bootstrap-memory_lock)。根据文档来说是 gc的时候如果被swap到了disk上，那么需要swap回来，会花时间，可能会导致gc time out

* Linux

The vm.max_map_count setting should be set permanently in /etc/sysctl.conf:

>$ grep vm.max_map_count /etc/sysctl.conf
vm.max_map_count=262144

To apply the setting on a live system type: sysctl -w vm.max_map_count=262144



1. 版本
```
-oos    The oss flavor does not include X-Pack, and contains only open-source Elasticsearch.

-baisc  The basic flavor, which is the default, ships with X-Pack Basic features pre-installed and automatically activated with a free licence

-platinum The platinum flavor features all X-Pack functionally under a 30-day trial licence. 


```

2. x-pack
```
X-Pack是一个Elastic Stack的扩展，将安全，警报，监视，报告和图形功能包含在一个易于安装的软件包中。
在Elasticsearch 5.0.0之前，您必须安装单独的Shield，Watcher和Marvel插件才能获得在X-Pack中所有的功能
```