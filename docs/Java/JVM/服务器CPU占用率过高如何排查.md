问题：
公司参加HW期间，项目两台双活的jboss服务器频繁触发cpu利用率过高告警，cpu利用率长时间在90%以上。
排查思路：
第一步：在两台Linux服务器上，执行top命令，并按大写P以cpu利用率排序，确定cpu占用最高的进程为 java进程
那么，java进程cpu占用过高该如何排查呢，我们从两个角度出发：
（1）执行任务的java线程本身存在bug，死循环或者操作本身耗cpu，导致cpu占用过高
（2）jvm发生频繁gc，导致cpu过高
第二步：先排查java任务线程本身，确定什么线程cpu占用过高
（1）方法一：ps -mp [java进程id] -o THREAD,tid,time | sort -n   找到cpu占用最大的线程id![20210407100513581.png](https://raw.githubusercontent.com/danmuking/image/main/6df538307ec36475b219b48cfb531ea5.png)
_**注意**，该方法在生产环境测试，发现无法找到cpu占用高的线程，显示所有线程cpu占用均为0，因此实际排查采用方法二_![20210407100658287.png](https://raw.githubusercontent.com/danmuking/image/main/476b2a9e61a050c627d5fe783226361c.png)
（2）使用top -H -p [java进程id]，找到cpu占用较高的线程id，如下图所示，左边红框标注的PID列为线程id
![20210407100752520.png](https://raw.githubusercontent.com/danmuking/image/main/a1c205d6687cfd6504f84884dc8f0ebc.png)
**第三步：**计算java线程id的16进制值，因为后续用jstack看到的线程快照中，线程id为小写十六进制值
（1）可百度在线进制转换
（2）可使用windows自带的计算器，程序员模式，可转换十六进制
（3）Linux可使用命令：**printf "%x\n" [线程_id]**![20210407101534309.png](https://raw.githubusercontent.com/danmuking/image/main/607402d59180d7d82f551124d33903bc.png)
**第四步：使用命令 jstack [java进程pid] | grep [线程id十六进制值] -A 30**（-A 30表示向下打印30行）![2021040710285629.png](https://raw.githubusercontent.com/danmuking/image/main/27f55d176b9668c2195b8c4631e5f67e.png)
分析上图可知，线程在做正则匹配，查看完整堆栈可定位到具体代码位置，分析后，确定在做url的正则匹配。继续分析其他线程，发现基本都在做url正则匹配。
后续分析，则需要结合业务代码分析，正则匹配是否耗时。
正则分析参考博客：[https://blog.csdn.net/weixin_33023873/article/details/114740786](https://blog.csdn.net/weixin_33023873/article/details/114740786)
第五步：从gc角度出发，是否存在大量gc，首先确定当前内存消耗情况，使用top命令或者查看设备监控管理系统，确定内存利用率达97%：
![2021040710273567.png](https://raw.githubusercontent.com/danmuking/image/main/f0f8743aa4d7686af50846aab7a5a889.png)
**第六步：**确认gc次数，使用命令 jstat -gc [java进程ID]：![2021040710434242.png](https://raw.githubusercontent.com/danmuking/image/main/f74567380718ad8e2aff0be48f62f650.png)
**_YGC，表示 Young GC，也就是Minor GC，发生在新生代中的GC_**
**_FGC，表示 Full GC，发生在老年代中的GC_**
**_S0C：第一个幸存区的大小_**
**_S1C：第二个幸存区的大小_**
**_S0U：第一个幸存区的使用大小_**
**_S1U：第二个幸存区的使用大小_**
**_EC：伊甸园区的大小_**
**_EU：伊甸园区的使用大小_**
**_OC：老年代大小_**
**_OU：老年代使用大小_**
**_MC：方法区大小_**
**_MU：方法区使用大小_**
**_CCSC:压缩类空间大小_**
**_CCSU:压缩类空间使用大小_**
**_YGCT：年轻代垃圾回收消耗时间_**
**_FGCT：老年代垃圾回收消耗时间_**
**_GCT：垃圾回收消耗总时间 _**
**_如何计算gc频率，参考：_**[**_https://blog.csdn.net/qq_18671415/article/details/104446568_**](https://blog.csdn.net/qq_18671415/article/details/104446568)
**_结合上图可知，程序运行以来共发生7436次YGC，63次FGC，gc次数较多_**
**_基本可以说明存在频繁GC导致cpu占用高的问题_**
**_第七步：使用命令dump 内存堆存储快照：jmap -dump:format=b,file=/tmp/my.hprof [java进程id]_**
**_第八步：使用内存分析工具，如Eclipse Memory Analyzer等分析my.hprof文件，分析内存那块占用大，存在内存泄露，导致空间无法释放。_**
**_总结：_**
**_cpu利用率过高排查，需要从两个角度排查，一是自身任务线程是否存在bug，二是是否内存泄露导致触发频繁gc；然后利用top、jstack、jmap等工具，定位出问题的代码位置，然后针对性分析修改。_**

## 参考资料
[java进程cpu占用高如何排查_java web应用服务器cpu90-CSDN博客](https://blog.csdn.net/jwentao01/article/details/115461695)
