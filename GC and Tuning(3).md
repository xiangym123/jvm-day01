# GC和GC Tuning

作者：马士兵教育 http://mashibing.com

### GC的基础知识

#### 1.什么是垃圾

> C语言申请内存：malloc free
>
> C++： new delete
>
> c/C++ 手动回收内存
>
> Java: new ？
>
> 自动内存回收，编程上简单，系统不容易出错，手动释放内存，容易出两种类型的问题：
>
> 1. 忘记回收
> 2. 多次回收

没有任何引用指向的一个对象或者多个对象（循环引用）

#### 2.如何定位垃圾

1. 引用计数
2. 根可达算法

#### 3.常见的垃圾回收算法

1. 标记清除 - 位置不连续 产生碎片 效率偏低（两遍扫描）
2. 拷贝算法 - 没有碎片，浪费空间
3. 标记压缩 - 没有碎片，效率偏低（两遍扫描，指针需要调整）

#### 4.JVM内存分代模型（用于分代垃圾回收算法）

1. 部分垃圾回收器使用的模型
2. 新生代 + 老年代 + 永久代（1.7）/ 元数据区(1.8) Metaspace
   1. 永久代 元数据 - Class
   2. 永久代必须指定大小限制 ，元数据可以设置，也可以不设置，无上限（受限于物理内存）
   3. 字符串常量 1.7 - 永久代，1.8 - 堆
   4. MethodArea逻辑概念 - 永久代、元数据
3. 新生代 = Eden + 2个suvivor区 
   1. YGC回收之后，大多数的对象会被回收，活着的进入s0
   2. 再次YGC，活着的对象eden + s0 -> s1
   3. 再次YGC，eden + s1 -> s0
   4. 年龄足够 -> 老年代 （15 CMS 6）
   5. s区装不下 -> 老年代
4. 老年代
   1. 顽固分子
   2. 老年代满了FGC Full GC
5. GC Tuning (Generation)
   1. 尽量减少FGC
   2. MinorGC = YGC
   3. MajorGC = FGC

#### 5.常见的垃圾回收器

1. Serial 年轻代 串行回收
2. PS 年轻代 并行回收
3. ParNew 年轻代 配合CMS的并行回收
4. SerialOld 
5. ParallelOld
6. ConcurrentMarkSweep 老年代 并发的， 垃圾回收和应用程序同时运行，降低STW的时间(200ms)
7. G1(10ms)
8. ZGC (1ms) PK C++
9. Shenandoah
10. Eplison

1.8默认的垃圾回收：PS + ParallelOld

#### 6.JVM调优第一步，了解生产环境下的垃圾回收器组合

* JVM的命令行参数参考：https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html

* HotSpot参数分类

  > 标准： - 开头，所有的HotSpot都支持
  >
  > 非标准：-X 开头，特定版本HotSpot支持特定命令
  >
  > 不稳定：-XX 开头，下个版本可能取消

  java -version

  java -X

  
  
  试验用程序：
  
  ```java
  import java.util.List;
  import java.util.LinkedList;
  
  public class HelloGC {
    public static void main(String[] args) {
      System.out.println("HelloGC!");
      List list = new LinkedList();
      for(;;) {
        byte[] b = new byte[1024*1024];
        list.add(b);
      }
    }
  }
  ```
  
  
  
  1. java -XX:+PrintCommandLineFlags HelloGC
  2. java -Xmn10M -Xms40M -Xmx60M -XX:+PrintCommandLineFlags HelloGC
  3. java -XX:+UseConcMarkSweepGC -XX:+PrintCommandLineFlags HelloGC
  4. java -XX:+PrintFlagsInitial 默认参数值
      java -XX:+PrintFlagsInitial | wc -l
	  java -XX:+PrintFlagsInitial | more
  5. java -XX:+PrintFlagsFinal 最终参数值
  6. java -XX:+PrintFlagsFinal | grep xxx 找到对应的参数
  7. java -XX:+PrintFlagsFinal -version |grep GC
   java -Xms200M -Xmx200M -XX:+PrintGCDetails   T15_FullGC_Problem01

* 常见垃圾回收器组合参数设定：(1.8)
  * -XX:+UseSerialGC = Serial New (DefNew) + Serial Old
    * 小型程序。默认情况下不会是这种选项，HotSpot会根据计算及配置和JDK版本自动选择收集器
  * -XX:+UseParNewGC = ParNew + SerialOld
    * 这个组合已经很少用（在某些版本中已经废弃）
    * https://stackoverflow.com/questions/34962257/why-remove-support-for-parnewserialold-anddefnewcms-in-the-future
  * UseConc<font color=red>(urrent)</font>MarkSweepGC = ParNew + CMS + Serial Old
  * UseParallelGC = Parallel Scavenge + Parallel Old (1.8默认) 【PS + SerialOld】
  * UseParallelOldGC = Parallel Scavenge + Parallel Old
  * UseG1GC = G1
  
* Linux中没找到默认GC的查看方法，而windows中会打印UseParallelGC 
  * java +XX:+PrintCommandLineFlags -version
  * 通过GC的日志来分辨
  
* Linux下1.8版本默认的垃圾回收器到底是什么？

  * 1.8.0_181 默认（看不出来）Copy MarkCompact
  * 1.8.0_222 默认 PS + PO

#### 7.调优，从规划开始

* 调优，从业务场景开始，没有业务场景的调优都是耍流氓
* 无监控，不调优
* 步骤：
  1. 熟悉业务场景（没有最好的垃圾回收器，只有最合适的垃圾回收器）
     1. 响应时间、停顿时间
     2. 吞吐量 = 用户时间 / 用户时间 + GC时间
  2. 选择回收器组合
  3. 计算内存需求
  4. 设定年代大小、升级年龄
  5. 设定日志参数
     1. -Xloggc:/opt/xxx/logs/xxx-xxx-gc-%t.log -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=20M -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCCause
  6. 观察日志情况

#### 8.GC日志详解

heap dump部分：

```java
eden space 5632K, 94% used [0x00000000ff980000,0x00000000ffeb3e28,0x00000000fff00000)
                            后面的内存地址指的是，起始地址，使用空间结束地址，整体空间结束地址
```

#### 9.实战操作

1. 测试代码：

   ```java
   package com.mashibing.jvm.gc;
   
   import java.math.BigDecimal;
   import java.util.ArrayList;
   import java.util.Date;
   import java.util.List;
   import java.util.concurrent.ScheduledThreadPoolExecutor;
   import java.util.concurrent.ThreadPoolExecutor;
   import java.util.concurrent.TimeUnit;
   
   /**
    * 从数据库中读取信用数据，套用模型，并把结果进行记录和传输
    */
   
   public class T15_FullGC_Problem01 {
   
       private static class CardInfo {
           BigDecimal price = new BigDecimal(0.0);
           String name = "张三";
           int age = 5;
           Date birthdate = new Date();
   
           public void m() {}
       }
   
       private static ScheduledThreadPoolExecutor executor = new ScheduledThreadPoolExecutor(50,
               new ThreadPoolExecutor.DiscardOldestPolicy());
   
       public static void main(String[] args) throws Exception {
           executor.setMaximumPoolSize(50);
   
           for (;;){
               modelFit();
               Thread.sleep(100);
           }
       }
   
       private static void modelFit(){
           List<CardInfo> taskList = getAllCardInfo();
           taskList.forEach(info -> {
               // do something
               executor.scheduleWithFixedDelay(() -> {
                   //do sth with info
                   info.m();
   
               }, 2, 3, TimeUnit.SECONDS);
           });
       }
   
       private static List<CardInfo> getAllCardInfo(){
           List<CardInfo> taskList = new ArrayList<>();
   
           for (int i = 0; i < 100; i++) {
               CardInfo ci = new CardInfo();
               taskList.add(ci);
           }
   
           return taskList;
       }
   }
   
   ```

2. java -Xms200M -Xmx200M com.mashibing.jvm.gc.T15_FullGC_Problem01

3. top命令观察到问题：内存不断增长 CPU占用率居高不下

4. jps定位具体java进程

5. jstat -gc 4655 500 动态观察gc情况 / 阅读GC日志发现频繁GC / arthas观察 / jconsole

6. jmap -dump:format=b,file=xxx pid / jmap -histo
   jmap -dump:format=b,file=a1.dump 2833  生成a.dump文件
   jhat a.dump
   jmap -histo 2833
7. java -Xms20M -Xmx20M -XX:+UseParallelGC -XX:+HeapDumpOnOutOfMemoryError com.mashibing.jvm.gc.T15_FullGC_Problem01

8. 使用MAT / jhat进行dump文件分析

9. 找到代码的问题

#### 实战：jconsole远程连接

1. 程序启动加入参数：

   > ```shell
   > java -Djava.rmi.server.hostname=192.168.17.11 -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=11111 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false XXX
   > ```

2. 如果遭遇 Local host name unknown：XXX的错误，修改/etc/hosts文件，把XXX加入进去

   > ```java
   > 192.168.17.11 basic localhost localhost.localdomain localhost4 localhost4.localdomain4
   > ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
   > ```

3. 关闭linux防火墙（实战中应该打开对应端口）

   > ```shell
   > service iptables stop
   > chkconfig iptables off #永久关闭
   > ```

4. windows上打开 jconsole远程连接 192.168.17.11:11111

#### 作业：

1. -XX:MaxTenuringThreshold控制的是什么？
   	A: 对象升入老年代的年龄
      	B: 老年代触发FGC时的内存垃圾比例
   
2. 生产环境中，倾向于将最大堆内存和最小堆内存设置为：（为什么？）
   	A: 相同 B：不同
   	
3. JDK1.8默认的垃圾回收器是：
   	A: ParNew + CMS
      	B: G1
      	C: PS + ParallelOld
      	D: 以上都不是
   
4. 什么是响应时间优先？

5. 什么是吞吐量优先？

6. ParNew和PS的区别是什么？

7. ParNew和ParallelOld的区别是什么？（年代不同，算法不同）

8. 长时间计算的场景应该选择：A：停顿时间 B: 吞吐量

9. 大规模电商网站应该选择：A：停顿时间 B: 吞吐量

10. HotSpot的垃圾收集器最常用有哪些？

11. 常见的HotSpot垃圾收集器组合有哪些？

12. JDK1.7 1.8 1.9的默认垃圾回收器是什么？如何查看？

13. 所谓调优，到底是在调什么？

14. 如果采用PS + ParrallelOld组合，怎么做才能让系统基本不产生FGC

15. 如果采用ParNew + CMS组合，怎样做才能够让系统基本不产生FGC

     1.加大JVM内存

     2.加大Young的比例

     3.提高Y-O的年龄

     4.提高S区比例

     5.避免代码内存泄漏

16. G1是否分代？G1垃圾回收器会产生FGC吗？

17. 如果G1产生FGC，你应该做什么？

     1. 扩内存
     2. 提高CPU性能（回收的快，业务逻辑产生对象的速度固定，垃圾回收越快，内存空间越大）



### 参考资料

1. [https://blogs.oracle.com/
    ](https://blogs.oracle.com/jonthecollector/our-collectors)[jonthecollector](https://blogs.oracle.com/jonthecollector/our-collectors)[/our-collectors](https://blogs.oracle.com/jonthecollector/our-collectors)
2. https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html
3. http://java.sun.com/javase/technologies/hotspot/vmoptions.jsp
4.  JVM调优参考文档：https://docs.oracle.com/en/java/javase/13/gctuning/introduction-garbage-collection-tuning.html#GUID-8A443184-7E07-4B71-9777-4F12947C8184 