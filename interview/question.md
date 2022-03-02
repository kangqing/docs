# 线上问题排查

## 查看正在运行的java程序
jps -l 查看所有正在运行的Java程序，同时显示启动类类名，获取到PID
```bash
29201 jdk.jcmd/sun.tools.jps.Jps
17912 org.jetbrains.jps.cmdline.Launcher
```

## 查看JVM参数
jinfo -flags PID 查看运行时进程参数与JVM参数
```bash
VM Flags:
-XX:CICompilerCount=4 -XX:ConcGCThreads=2 -XX:G1ConcRefinementThreads=8 -XX:G1HeapRegionSize=1048576 -XX:GCDrainStackTargetSize=64 -XX:InitialHeapSize=268435456 -XX:MarkStackSize=4194304 -XX:MaxHeapSize=4294967296 -XX:MaxNewSize=2576351232 -XX:MinHeapDeltaBytes=1048576 -XX:NonNMethodCodeHeapSize=5836300 -XX:NonProfiledCodeHeapSize=122910970 -XX:ProfiledCodeHeapSize=122910970 -XX:ReservedCodeCacheSize=251658240 -XX:+SegmentedCodeCache -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseFastUnorderedTimeStamps -XX:+UseG1GC 
```
## 查询虚拟机不稳定参数默认值

java -XX:+PrintFlagsInitial

## 查询虚拟机参数的最终生效值

java -XX:+PrintFlagsFinal

## 查看 GC 即时状态
jstat -gc PID 1000 10 每秒查看一次gc信息，共10次

S0C: 当前幸存者空间 0 容量 (kB)。

S1C：当前幸存者空间 1 容量 (kB)。

S0U: 幸存者空间 0 利用率 (kB)。

S1U：幸存者空间 1 利用率 (kB)。

EC: 当前伊甸园空间容量 (kB)。

EU：伊甸园空间利用率（kB）。

OC: 当前旧空间容量 (kB)。

OU: 旧空间利用率 (kB)。

MC：元空间容量（kB）。

MU：元空间利用率（kB）。

CCSC：压缩类空间容量（kB）。

CCSU：使用的压缩类空间（kB）。

YGC：年轻代垃圾回收事件的数量。

YGCT：年轻代垃圾回收时间。

FGC：完整 GC 事件的数量。

FGCT：完整的垃圾收集时间。

GCT：总垃圾收集时间。

## 内存问题
内存泄露导致OOM？内存占用异常的高？这是生产环境常常出现的问题，Java提供dump文件供我们对内存里发生过的事情进行了记录，我们需要借助一些工具从中获取有价值的信息。

- 导出Dump文件
提前对Java程序加上这些参数印dump文件 -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=./
对正在运行的程序使用jmap：jmap -dump:format=b,file=heap.hprof PID
- 分析Dump文件
如果Dump文件不太大的话，可以传到 http://heaphero.io/index.jsp 来分析

文件比较大，且想进行更加系统的分析，推荐使用[MAT](https://www.eclipse.org/mat/)分析，有如下几种常用查看方式

1. 首页中的【Leak Suspects】能推测出问题所在
2. 点击【Create a histogram from an arbitrary set of objects】查到所有对象的数量
3. 右键点击某个对象【Merge Shortest Paths to GC Roots】-> 【exclude all phantom/weak/soft etc. references】能查询到大量数量的某个对象是哪个GC ROOT引用的

## CPU 占用100%
任务长时间不退出？CPU 负载过高？很可能因为死循环或者死锁，导致某些线程一直执行不被中断，但是不报错是最烦人的，所以日志里看不到错误信息，并且又不能用dump文件分析，因为跟内存无关。这个时候就需要用线程分析工具来帮我们了。

```bash
#首先使用 top命令查看出 CPU 100% 的进程 PID

#查看当前 PID 的进程中有哪些线程
# 查看当前 pid 中有哪些线程
top -Hp 进程pid

#然后找到 CPU 较高的线程的 PID 然后转换成 16 进制
#转换成 16 进制
# 把 pid 转换成 16 进制
printf '%x' 线程pid

#使用 jstack 生成 进程 pid 到一个文件 x.txt 中去
jstack 进程pid > x.txt

#打开文件，搜索 16 进制的值
# 打开生成的文件
vim x.txt

# 搜索生成的16进制数据
/16进制
# 往下看，找到业务代码哪行导致的
```
