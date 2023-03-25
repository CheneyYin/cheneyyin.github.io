---
layout: post
title: "常见的Out of Memory Error"
subtitle: "介绍几种常见的导致OOM的场景"
date: 2022-07-02
author: "Cheney.Yin"
header-img: "//imgloc.com/i/iADI63"
tags: 
 - Java
 - JVM
 - OOM
 - Heap
 - Stack
 - Stack Frame
 - 栈帧
 - 常量池
 - Direct内存
---

# Out of Memory Error

## Heap内存OOM

```java
import java.util.ArrayList;
import java.util.List;

//  "java.debug.settings.vmArgs": "-Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError"
public class HeapOutMemory {

    static class MemoryObject {}

    public static void main(String[] args) {
        List<MemoryObject> list = new ArrayList<>();
        while(true) {
            list.add(new MemoryObject());
        }
    }
}

```

结果：

```shell
 /usr/bin/env /usr/lib/jvm/java-11-openjdk/bin/java -Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError -cp /home/cheney/expr/java/OutOfMemoryError/bin HeapOutMemory 
java.lang.OutOfMemoryError: Java heap space
Dumping heap to java_pid46145.hprof ...
Heap dump file created [28212769 bytes in 0.123 secs]
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
        at java.base/java.util.Arrays.copyOf(Arrays.java:3720)
        at java.base/java.util.Arrays.copyOf(Arrays.java:3689)
        at java.base/java.util.ArrayList.grow(ArrayList.java:238)
        at java.base/java.util.ArrayList.grow(ArrayList.java:243)
        at java.base/java.util.ArrayList.add(ArrayList.java:486)
        at java.base/java.util.ArrayList.add(ArrayList.java:499)
        at HeapOutMemory.main(HeapOutMemory.java:12)
```

## Stack过深启发StackOverflow

```java
//  "java.debug.settings.vmArgs": "-Xss136k"
public class StackOverflow {
    private long stackDepth = 0l;

    public void stackPush() {
        this.stackDepth++;
        this.stackPush();
    }

    public static void main(String[] args) {
        StackOverflow sof = new StackOverflow();
        try {
            sof.stackPush();
        } catch (Throwable e) {
            System.out.println("stack depth:" + sof.stackDepth);
            throw e;
        }
    }


}

```

结果：

```shell
/usr/bin/env /usr/lib/jvm/java-11-openjdk/bin/java -Xss136k -cp /home/cheney/expr/java/OutOfMemoryError/bin StackOverflow 
stack depth:365
Exception in thread "main" java.lang.StackOverflowError
        at StackOverflow.stackPush(StackOverflow.java:6)
        at StackOverflow.stackPush(StackOverflow.java:7)
			......
        at StackOverflow.stackPush(StackOverflow.java:7)
        at StackOverflow.main(StackOverflow.java:13)
```

## Stack栈帧过大引发StackOverflow

```java

// "java.debug.settings.vmArgs": "-Xss136k"
public class StackFrameOverflow {
    private long stackDepth = 0l;

    public void stackPushLargeFrame() {
        long unused0,unused1,unused2,unused3,unused4,unused5,unused6,unused7,unused8,unused9;
        long unused10,unused11,unused12,unused13,unused14,unused15,unused16,unused17,unused18,unused19;
        long unused20,unused21,unused22,unused23,unused24,unused25,unused26,unused27,unused28,unused29;
        long unused30,unused31,unused32,unused33,unused34,unused35,unused36,unused37,unused38,unused39;
        long unused40,unused41,unused42,unused43,unused44,unused45,unused46,unused47,unused48,unused49;
        long unused50,unused51,unused52,unused53,unused54,unused55,unused56,unused57,unused58,unused59;
        long unused60,unused61,unused62,unused63,unused64,unused65,unused66,unused67,unused68,unused69;
        long unused70,unused71,unused72,unused73,unused74,unused75,unused76,unused77,unused78,unused79;
        long unused80,unused81,unused82,unused83,unused84,unused85,unused86,unused87,unused88,unused89;
        long unused90,unused91,unused92,unused93,unused94,unused95,unused96,unused97,unused98,unused99;
    
        unused0 = unused1 = unused2 = unused3 = unused4 = unused5 = unused6 = unused7 = unused8 = unused9 = 0l;
        unused10 = unused11 = unused12 = unused13 = unused14 = unused15 = unused16 = unused17 = unused18 = unused19 = 0l;
        unused20 = unused21 = unused22 = unused23 = unused24 = unused25 = unused26 = unused27 = unused28 = unused29 = 0l;
        unused30 = unused31 = unused32 = unused33 = unused34 = unused35 = unused36 = unused37 = unused38 = unused39 = 0l;
        unused40 = unused41 = unused42 = unused43 = unused44 = unused45 = unused46 = unused47 = unused48 = unused49 = 0l;
        unused50 = unused51 = unused52 = unused53 = unused54 = unused55 = unused56 = unused57 = unused58 = unused59 = 0l;
        unused60 = unused61 = unused62 = unused63 = unused64 = unused65 = unused66 = unused67 = unused68 = unused69 = 0l;
        unused70 = unused71 = unused72 = unused73 = unused74 = unused75 = unused76 = unused77 = unused78 = unused79 = 0l;
        unused80 = unused81 = unused82 = unused83 = unused84 = unused85 = unused86 = unused87 = unused88 = unused89 = 0l;
        unused90 = unused91 = unused92 = unused93 = unused94 = unused95 = unused96 = unused97 = unused98 = unused99 = 0l;
        
        this.stackDepth++;
        this.stackPushLargeFrame();
    }

    public static void main(String[] args) {
        StackFrameOverflow sfo = new StackFrameOverflow();
        try {
            sfo.stackPushLargeFrame();
        } catch (Throwable e) {
            System.out.println("Stack depth:" + sfo.stackDepth);
            throw e;
        }
        
    }
    
}

```

结果：

```shell
/usr/bin/env /usr/lib/jvm/java-11-openjdk/bin/java -Xss136k -cp /home/cheney/expr/java/OutOfMemoryError/bin StackFrameOverflow 
Stack depth:20
Exception in thread "main" java.lang.StackOverflowError
        at StackFrameOverflow.stackPushLargeFrame(StackFrameOverflow.java:18)
        at StackFrameOverflow.stackPushLargeFrame(StackFrameOverflow.java:30)
        at StackFrameOverflow.stackPushLargeFrame(StackFrameOverflow.java:30)
        at StackFrameOverflow.stackPushLargeFrame(StackFrameOverflow.java:30)
        at StackFrameOverflow.stackPushLargeFrame(StackFrameOverflow.java:30)
        at StackFrameOverflow.stackPushLargeFrame(StackFrameOverflow.java:30)
        at StackFrameOverflow.stackPushLargeFrame(StackFrameOverflow.java:30)
        at StackFrameOverflow.stackPushLargeFrame(StackFrameOverflow.java:30)
        at StackFrameOverflow.stackPushLargeFrame(StackFrameOverflow.java:30)
        at StackFrameOverflow.stackPushLargeFrame(StackFrameOverflow.java:30)
        at StackFrameOverflow.stackPushLargeFrame(StackFrameOverflow.java:30)
        at StackFrameOverflow.stackPushLargeFrame(StackFrameOverflow.java:30)
        at StackFrameOverflow.stackPushLargeFrame(StackFrameOverflow.java:30)
        at StackFrameOverflow.stackPushLargeFrame(StackFrameOverflow.java:30)
        at StackFrameOverflow.stackPushLargeFrame(StackFrameOverflow.java:30)
        at StackFrameOverflow.stackPushLargeFrame(StackFrameOverflow.java:30)
        at StackFrameOverflow.stackPushLargeFrame(StackFrameOverflow.java:30)
        at StackFrameOverflow.stackPushLargeFrame(StackFrameOverflow.java:30)
        at StackFrameOverflow.stackPushLargeFrame(StackFrameOverflow.java:30)
        at StackFrameOverflow.stackPushLargeFrame(StackFrameOverflow.java:30)
        at StackFrameOverflow.stackPushLargeFrame(StackFrameOverflow.java:30)
        at StackFrameOverflow.main(StackFrameOverflow.java:36)
```

和上一例子相比，`-Xss136k`栈大小一直，但本例的栈深度更小。

## 创建线程引发OOM

```java
public class StackOverflowByThread {

    public void loop() {
        while(true){
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                // TODO Auto-generated catch block
                System.out.println("Cancel Thread.");
            }
        }
    }

    public void threads() {
        Thread[] threadsArray = new Thread[10000];
        for (int i = 0; i < threadsArray.length; i++) {
            Thread thread = new Thread(()->{
                this.loop();
            });
            thread.start();
            threadsArray[i] = thread;
            System.out.println("Start #" + i + " thread.");
        }

        System.out.println("Has start all threads.");

        for (Thread thread : threadsArray) {
            thread.interrupt();
        }

        System.out.println("Has send cancel all threads.");

    }
    
    public static void main(String[] args) {
        StackOverflowByThread soft = new StackOverflowByThread();
        try {
            soft.threads();
        } catch (Throwable e) {
            throw e;
        }
    }
}

```

为了方便测试，建立开启内存控制器的`cgroup`组，

```
mkdir /sys/fs/cgroup/A
echo '33554432' > /sys/fs/cgroup/A/memory.max
echo $$ > /sys/fs/cgroup/A/cgroup.procs
```

```shell
java -Xss136k -Xmx20m -cp ./bin StackOverflowByThread
...
...
Start #1249 thread.
Start #1250 thread.
[1]    44247 killed     java -Xss136k -Xmx20m -cp ./bin StackOverflowByThread
```

## 常量池启发OOM

```java
import java.util.HashSet;
import java.util.Set;

// "java.debug.settings.vmArgs": "-Xmx20m -Xms20m -XX:MaxMetaspaceSize=10m"
public class ConstantPoolOOM {

    public void constantPoolOOM() {
        Set<String> poolSet = new HashSet<>();
        for(int i = 0; i < 100000000; i++) {
            poolSet.add(String.valueOf(i).intern());
        }
    }

    public static void main(String[] args) {
        ConstantPoolOOM cpOOM = new ConstantPoolOOM();
        cpOOM.constantPoolOOM();
    }
    
}

```

结果：

```shell
/usr/lib/jvm/java-11-openjdk/bin/java -Xmx20m -Xms20m -XX:MaxMetaspaceSize=10m -cp /home/cheney/expr/java/OutOfMemoryError/bin ConstantPoolOOM 
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
        at java.base/java.util.HashMap.resize(HashMap.java:699)
        at java.base/java.util.HashMap.putVal(HashMap.java:658)
        at java.base/java.util.HashMap.put(HashMap.java:607)
        at java.base/java.util.HashSet.add(HashSet.java:220)
        at ConstantPoolOOM.constantPoolOOM(ConstantPoolOOM.java:9)
        at ConstantPoolOOM.main(ConstantPoolOOM.java:15)
```

## Direct内存OOM

后续补充
