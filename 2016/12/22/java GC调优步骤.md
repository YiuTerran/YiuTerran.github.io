#java GC调优步骤
tags: [java,gc,jvm]

---
0.增加GC相关选项：
```
-verbose:gc
-XX:+UseGCLogFileRotation
-XX:NumberOfGCLogFiles=5
-XX:GCLogFileSize=512K
-XX:+PrintGCDetails
-XX:+PrintGCTimeStamps
-XX:+PrintGCDateStamps
-XX:+PrintTenuringDistribution
-XX:+PrintGCApplicationStoppedTime
-Xloggc:/var/app/log/Push-server/gc.log
```
1. 如果不能确定所需内存，使用自动jvm自动调优；
2. 大致确定所需内存后，使用-Xmx -Xms设置堆大小；
3. 观察GC log确定FullGC后剩余堆大小（即为活跃数据大小）；
4. 整个堆大小宜为老年代活跃数据大小的3-4倍；
5. 永久带大小应该比永久带活跃数据大1.2~1.5倍；
6. 新生代空间应该为老年代空间活跃数据的1~1.5倍；
7. 通过top命令观察栈占用空间、直接内存占用空间，决定所需机器内存大小；
8. 新生代大小决定了Minor GC的周期和时长，缩短新生代大小可以减少停顿时长，但是增加了GC频率；在调整新生代大小时，尽量保持老年代大小不变；
9. 老年代大小不应该小于活跃数据的1.5倍；新生代空间至少为java堆大小的10%；增加堆大小时，注意不要超过可用物理内存数；
10. 从throughput收集器迁移到CMS时，需要将老年代空间增加20%~30%；
11. 新生代分为Eden和Survivor两部分，Survivor可以通过`-XX:SurvivorRatio=xx`来控制，对应的大小为`-Xmn<value>/(ratio+2)`；
12. 通过`-XX:MaxTenuringThreshold=<n>`来指定晋升阈值（年龄），n为0~15之间；
13. 期望Survivor空间为剩余总存活对象大小的2倍(age=1；
14. 注意调节Survivor大小时，保持Eden大小不变；
15. 如果Survivor空间足够大，且对象大部分并未到达老年代，那么就可以将晋升年纪指定的足够大（15）。在Eden与Survivor之间复制和CMS老年代空间压缩之间，我们宁愿选择前者；
16. CMS必须能以对象从新生代提升到老年代的同等速度对老年代中的对象进行收集，否则，就会失速；
17. 如果观察到'concurrent mode failures'，意味着失速已经发生，必须减少`-XX:CMSInitiatingOccupancyFraction=<percent>`的值；
18. 使用上述选项的同时，最好同时使用`-XX:+UseCMSInitiatingOccupancyOnly`，强制使用该比例,该比例的大小应该大于老年代占用空间和活跃数据大小之比，一般而言`老年代大小*该比例>1.5*老年代活跃数据大小`；
20. 使用`-XX:+ExplicitGCInvokesConcurrentAndUnloadsCloasses`可以使用CMS进行显式垃圾回收（`System.gc()`)；通过`-XX:+DisableExplicitGC`关闭显示垃圾回收（慎用）；
21. 使用`-XX:+CMSClassUnloadingEnabled`打开永久带垃圾回收，使用`-XX:+CMSPermGenSweepingEnabled`打开CMS对永久带的扫描；使用`-XX:CMSInitiatingPermOccupancyFraction=<perscent>`激活回收比例阈值；
22. 使用`-XX:ParallelGCThreads=<n>`控制扫描线程数；使用`-XX:+CMSScavengeBeforeRemark`强制重新标记前进行一次MinorGC；如果由大量的引用对象或可终结对象要处理，使用`-XX:+ParallelRefProcEnabled`；
23. CMS包括Minor GC所带来的开销应该小于10%；
24. 如果缺少长时间调优的条件，安全起见，可以使用G1，仅设置如下参数即可：
```
-d64
-Xmx5g
-Xms5g
-XX:PermSize=100m
-XX:MaxPermSize=100m
-XX:MaxDirectMemorySize=1g
-XX:+UseG1GC
-XX:MaxGCPauseMillis=80
```
G1不必明确设置新生代大小，其自动调优也十分可靠，对于停顿时间往往在长时间运行后可以达到预期效果；对吞吐量优先的应用，可能不是那么明显。






