---
layout: post
title: Java性能分析器VisualVM简介
categories: Test
description: VisualVM 通过检测 JVM 中加载的类和对象信息等帮助我们分析内存使用情况，我们可以通过 VisualVM 的监视标签对应用程序进行内存分析。
keywords: VisualVM, Java, Analyzer 
---

# Java性能分析器VisualVM

## 准备工作

自从 JDK 6 Update 7 以后已经作为 Oracle JDK 的一部分，位于 JDK 根目录的 bin 文件夹下，无需安装，直接运行即可。

 

## 内存分析篇

VisualVM 通过检测 JVM 中加载的类和对象信息等帮助我们分析内存使用情况，我们可以通过 VisualVM 的监视标签对应用程序进行内存分析。

### **1）内存堆Heap**

首先我们来看内存堆Heap使用情况，我本机eclipse的进程在visualVM显示如下：

![031621407646615 (1)](https://tva1.sinaimg.cn/large/007DFXDhgy1g6065z8b2rj30wy0my42y.jpg)


随便写个小程序占用内存大的，运行一下

程序如下：

```java
package jvisualVM;

public class JavaHeapTest {
    public final static int OUTOFMEMORY = 200000000;
    
    private String oom;

    private int length;
    
    StringBuffer tempOOM = new StringBuffer();

    public JavaHeapTest(int leng) {
        this.length = leng;
       
        int i = 0;
        while (i < leng) {
            i++;
            try {
                tempOOM.append("a");
            } catch (OutOfMemoryError e) {
               e.printStackTrace();
               break;
            }
        }
        this.oom = tempOOM.toString();

    }

    public String getOom() {
        return oom;
    }

    public int getLength() {
        return length;
    }

    public static void main(String[] args) {
        JavaHeapTest javaHeapTest = new JavaHeapTest(OUTOFMEMORY);
        System.out.println(javaHeapTest.getOom().length());
    }

}
```


查看VisualVM Monitor tab, 堆内存变大了

![img](https://tva4.sinaimg.cn/large/007DFXDhgy1g6066en66mj30ru0jo769.jpg)

 

在程序运行结束之前， 点击Heap Dump 按钮， 等待一会儿，得到dump结果，可以看到一些Summary信息

点击Classes， 发现char[]所占用的内存是最大的

![img](https://tva2.sinaimg.cn/large/007DFXDhgy1g606bx8j72j30rq0jmq5x.jpg)

 

双击它，得到如下Instances结果

![img](https://tva4.sinaimg.cn/large/007DFXDhly1g606hg8vd3j30rp0jp779.jpg)

 Instances是按Size由大到小排列的

第一个就是最大的， 展开Field区域的 values

![img](https://ws2.sinaimg.cn/large/007DFXDhly1g606hgf32qj30l107a74m.jpg)

StringBuffer类型的 全局变量 tempOOM 占用内存特别大， 注意局部变量是无法通过 堆dump来得到分析结果的。

另外，对于“堆 dump”来说，在远程监控jvm的时候，VisualVM是没有这个功能的，只有本地监控的时候才有。



### 2)永久保留区域PermGen

其次来看下永久保留区域PermGen使用情况

运行一段类加载的程序，代码如下：


```java
package jvisualVM;

import java.io.File;
import java.lang.reflect.Method;
import java.net.MalformedURLException;
import java.net.URL;
import java.net.URLClassLoader;
import java.util.ArrayList;
import java.util.List;

public class TestPermGen {
    
    private static List<Object> insList = new ArrayList<Object>();

    public static void main(String[] args) throws Exception {

        permLeak();
    }

    private static void permLeak() throws Exception {
        for (int i = 0; i < 1000; i++) {
            URL[] urls = getURLS();
            URLClassLoader urlClassloader = new URLClassLoader(urls, null);
            Class<?> logfClass = Class.forName("org.apache.commons.logging.LogFactory", true,urlClassloader);
            Method getLog = logfClass.getMethod("getLog", String.class);
            Object result = getLog.invoke(logfClass, "TestPermGen");
            insList.add(result);
            System.out.println(i + ": " + result);
        }
    }

    private static URL[] getURLS() throws MalformedURLException {
        File libDir = new File("C:/Users/wadexu/.m2/repository/commons-logging/commons-logging/1.1.1");
        File[] subFiles = libDir.listFiles();
        int count = subFiles.length;
        URL[] urls = new URL[count];
        for (int i = 0; i < count; i++) {
            urls[i] = subFiles[i].toURI().toURL();
        }
        return urls;
    }

    
}
```

一个类型装载之后会创建一个对应的java.lang.Class实例，这个实例本身和普通对象实例一样存储于堆中，我觉得之所以说是这是一种特殊的实例，某种程度上是因为其充当了访问PermGen区域中类型信息的代理者。



运行一段时间后抛OutOfMemoryError了， VisualVM监控结果如下：

![img](https://ws1.sinaimg.cn/large/007DFXDhly1g606hgqm28j30rn0jo40j.jpg)

 

结论：PermGen区域分配的堆空间过小，我们可以通过设置-XX: PermSize参数和-XX:MaxPermSize参数来解决。

关于PermGen OOM深入分析请参考[这篇文章](http://www.cnblogs.com/zhuxing/articles/1247621.html)

关于Perform GC, 请参考[这篇文章](http://webcache.googleusercontent.com/search?q=cache:P1YTQVABpOEJ:www.coderli.com/log4j-permgen-space-leak/+&cd=4&hl=en&ct=clnk&gl=us)

这部分知识还是比较深入的，有空还要继续研究。

 



## CPU分析篇

CPU 性能分析的主要目的是统计函数的调用情况及执行时间，或者更简单的情况就是统计应用程序的 CPU 使用情况。

没有程序运行时的 CPU 使用情况如下图：

![img](https://ws4.sinaimg.cn/large/007DFXDhly1g606hgyg35j30oa0j0t9y.jpg)

 

运行一段 占用CPU 的小程序，代码如下


```java
package jvisualVM;

public class MemoryCpuTest {

    public static void main(String[] args) throws InterruptedException {

        cpuFix();
    }


    /**
     * cpu 运行固定百分比
     * 
     * @throws InterruptedException
     */
    public static void cpuFix() throws InterruptedException {
        // 80%的占有率
        int busyTime = 8;
        // 20%的占有率
        int idelTime = 2;
        // 开始时间
        long startTime = 0;
        
        while (true) {
            // 开始时间
            startTime = System.currentTimeMillis();
            
            /*
             * 运行时间
             */
            while (System.currentTimeMillis() - startTime < busyTime) {
                ;
            }
            
            // 休息时间
            Thread.sleep(idelTime);
        }
    }
}
```



查看监视页面 Monitor tab

![img](https://ws4.sinaimg.cn/large/007DFXDhly1g606hhg23sj30wy0myju5.jpg)

 

过高的 CPU 使用率可能是由于我们的项目中存在低效的代码；

在我们对程序施压的时候，过低的 CPU 使用率也有可能是程序的问题。

 

点击取样器Sampler， 点击“CPU”按钮， 启动CPU性能分析会话，VisualVM 会检测应用程序所有的被调用的方法，

在CPU samples tab 下可以看到我们的方法cpufix() 的自用时间最长， 如下图：

![img](https://ws4.sinaimg.cn/large/007DFXDhly1g606hi958nj30wy0my0x5.jpg)

切换到Thread CPU Time 页面下，我们的 main 函数这个进程 占用CPU时间最长， 如下图：

![img](https://ws4.sinaimg.cn/large/007DFXDhly1g606hixfv6j30wy0myjuh.jpg)



线程分析篇

Java 语言能够很好的实现多线程应用程序。当我们对一个多线程应用程序进行调试或者开发后期做性能调优的时候，往往需要了解当前程序中所有线程的运行状态，是否有死锁、热锁等情况的发生，从而分析系统可能存在的问题。

在 VisualVM 的监视标签内，我们可以查看当前应用程序中所有活动线程（Live threads）和守护线程（Daemon threads）的数量等实时信息。

 

运行一段小程序，代码如下：

```java
package jvisualVM;

public class MyThread extends Thread{
    
    public static void main(String[] args) {
        
        MyThread mt1 = new MyThread("Thread a");
        MyThread mt2 = new MyThread("Thread b");
        
        mt1.setName("My-Thread-1 ");
        mt2.setName("My-Thread-2 ");
        
        mt1.start();
        mt2.start();
    }
    
    public MyThread(String name) {
    }

    public void run() {
        
        while (true) {
            
        }
    }
    

}
```

 

Live threads 从11增加两个 变成13了

Daemon threads从8增加两个 变成10了 

![img](https://ws1.sinaimg.cn/large/007DFXDhly1g606hjlabqj30wy0myad4.jpg)

 

VisualVM 的线程标签提供了三种视图，默认会以时间线的方式展现， 如下图：

可以看到两个我们run的程序里启的线程：My-Thread-1 和 My-Thread-2

![img](https://ws2.sinaimg.cn/large/007DFXDhly1g606hk91emj30wy0my0w4.jpg)

 

另外还有两种视图分别是表视图和详细信息视图， 这里看一下每个Thread的详细视图：

![img](https://ws3.sinaimg.cn/large/007DFXDhly1g606hkyty9j30zh0owwie.jpg)



再来一段死锁的程序，看VisualVM 能否分析出来

```java
package jvisualVM;

public class DeadLock {
    public static void main(String[] args) {
        Resource r1 = new Resource();
        Resource r0 = new Resource();

        Thread myTh1 = new LockThread1(r1, r0);
        Thread myTh0 = new LockThread0(r1, r0);

        myTh1.setName("DeadLock-1 ");
        myTh0.setName("DeadLock-0 ");

        myTh1.start();
        myTh0.start();
    }
}

    class Resource {
        private int i;
    
        public int getI() {
            return i;
        }
    
        public void setI(int i) {
            this.i = i;
        }
        
    }

    class LockThread1 extends Thread {
        private Resource r1, r2;
    
        public LockThread1(Resource r1, Resource r2) {
            this.r1 = r1;
            this.r2 = r2;
        }
    
        @Override
        public void run() {
            int j = 0;
            while (true) {
                synchronized (r1) {
                    System.out.println("The first thread got r1's lock " + j);
                    synchronized (r2) {
                        System.out.println("The first thread got r2's lock  " + j);
                    }
                }
                j++;
            }
        }
    
    }

    class LockThread0 extends Thread {
        private Resource r1, r2;
    
        public LockThread0(Resource r1, Resource r2) {
            this.r1 = r1;
            this.r2 = r2;
        }
    
        @Override
        public void run() {
            int j = 0;
            while (true) {
                synchronized (r2) {
                    System.out.println("The second thread got r2's lock  " + j);
                    synchronized (r1) {
                        System.out.println("The second thread got r1's lock" + j);
                    }
                }
                j++;
            }
        }
    
    }
```

 

打开VisualVM检测到的JVM进程，我们可以看到这个tab在闪，VisualVM已经检测到我这个package下面的DeadLock类出错了

切换到Thread tab， 可以看到死锁了， Deadlock detected!

另外可以点击Thread Dump 线程转储，进一步分析，在这里就不赘述了，有兴趣的读者可以自行实验。

 ![img](https://ws3.sinaimg.cn/large/007DFXDhly1g606hm5q8pj30zh0owq9c.jpg)

 

## 参考文献

http://www.ibm.com/developerworks/cn/java/j-lo-visualvm/