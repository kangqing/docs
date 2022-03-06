## 自我介绍
您好，我叫杨旭，2017年毕业于天津理工大学软件工程专业，毕业后到现在一直从事Java后端开发的工作，平时用到的技术栈主要有 Spring、SpringBoot、Mybatis,MySql,redis等等，熟练使用集合框架、多线程开发，本人有较强的自学能力，能够较快适应各种工作环境，善于与其他人沟通共同推进项目进程。

## 说一下你的项目
上家公司做的是一个银行内部使用的一个提工单、写文章、写想法、绩效考核、兴趣部落的这么一个系统，主要使用的技术是 SpringBoot、ORM框架用的Mybatis、数据库使用的是MySQL、Redis用来做缓存、项目部署使用的是 Jenkins、版本控制用的git。

## 项目中的亮点

在项目中有这么一个模块，就是考评模块，给一个部门进行考评打分，然后这个考评会跟工资奖金挂钩，我们需要收集几个负责人打分的Excel表格，然后导入全部评价之后进行统计分析，这个过程一开始写的是一个个的导入，确认全部导入之后再进行考评逻辑，后来考虑到使用 CountDownLatch 同步类，等待 6 个负责人的 Excel 读取完毕之后，也就是 countdown() 6次之后，唤醒主逻辑进行考评统计。这样做的好处是，让 6 个考评人的打分能够以多线程异步的形式进行录入，节约了操作时间。


在写年报模块的时候，在项目中发现一个问题，由于用户量比较多，导致年报的计算录入的定时任务接口执行的比较慢，由于是定时任务，部署之后多实例只能有一个实例能够成功执行，否则就重复执行了，所以当时我经过查阅资料，了解到 xxl_job 和 elastic_job 两个分布式任务调度平台，能够对这种大数据量的定时任务执行分片，于多实例上分片执行。调研之后了解到使用 xxl_job 的公司比较多，上手简单，不依赖 zookeeper，虽然最后没有用到项目中（考虑到年报定时任务一年只执行一次，项目经理考虑到没有必要，就还是用的原来的 Schduling 定时任务），但是，我自己利用下班时间，在家里自己学习构建了 xxl_job 分布式任务调度平台。

## 多线程的使用

例如在年报的模块代码编写时，因为要统计员工在这一年的各种操作，例如写文章、写想法、回答问题等等一系列操作，如果一个个串行查询这些不相关的操作，则会叠加响应时间，这时候则可以使用线程池，把各个不相关的查询多线程并发执行，则能够明显缩短响应时间。

## 英文面试简介

Good morning, ladies and gentlemen! 

It is really my honor to have this opportunity for this interview. 

I hope I can make a good performance today. 

Now, I will introduce myself.

My name is Yangxu.I am 28 years old, 

I graduated from  Tianjin University of Technology. 

I mastered basic professional knowledge.

Becoming an Java Development Engineer is my dream .

I long for the chance to use my talents.

I think Ericsson is an international company, and I can get more in such a working environment.
 
That is the reason that I come here for the interview.

I think I'm a good team player.

I am very confident that I am qualified for the position of engineer in the company.

Thank you for giving me this opportunity.
