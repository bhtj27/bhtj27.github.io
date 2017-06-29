---
layout: post
title: 2017-06-29-threadpool.md 
date: Thu Jun 29 2017 16:30:02 GMT+0800 (中国标准时间)
tags: [线程池,Java]
categories: [Java笔记]
author: junjuntao
excerpt: "线程池框架分析，主要有Executor，ExecutorService，AbstractExecutorService，ThreadPoolExecutor等"
---

1.线程池框架类：
Executor：接口只有一个方法execute，该接口是一个顶层接口，传入参数为Runnable型
ExecutorService：该接口继承Execute，还扩展了几个方法shutdown，submit，shutdownNow，invokeAny等
AbstractExecutorService：抽象类实现了ExecuteService的大部分方法
ThreadPoolExecutor：继承了AbstractExecutorService，Executor中的execute方法在该类中具体实现；ThreadPoolExecutor类中有几个重要的方法，execute，submit，shutdown，shutdownNow

![enter description here][1]

2.线程核心类ThreadPoolExecutor
ThreadPoolExecutor中的execute方法向线程池提交一个任务，交由线程池去执行；submit犯法则是在AbstractExecutorService方法中实现的，也是向线程池提交一个任务，通过Future获取结果，shutdown和shutdownNow则是用来关闭线程池的。

创建线程池后，线程池处于running状态；
如果调用了shutdown方法，则线程池处理shurdown状态，此时线程池不再接受新的任务，他会等待所有的任务执行完毕；
如果调用了shutdownNow方法，则线程池处于stop状态，此时线程池不再接受新任务，并且尝试终止正在执行的任务；
当线程池处于shutdown或shutdownNow状态，并且所有的工作线程已经销毁，任务缓存队列已经清空或执行结束后，线程池被设置为terminated状态

3.ThreadPoolExecutor类几个重要的参数和方法
private final BlockingQueue<Runnable> workQueue;				//任务缓存队列，用来存放等待执行的任务
private final ReentrantLock mainLock=new ReentrantLock();		//线程池状态锁，对线程池状态的改变都要使用这个锁
private final HashSet<Worker> workers=new HashSet<Worker>();		//用来存放工作集

private volatile long keepAliveTime;		//线程存活时间
private volatile boolean allowCoreThreadTimeOut;		//是否允许核心线程设置存活时间
private volatile int corePoolSize;		//核心线程池大小，当线程池中的线程数大于该值是，提交的任务会被放到缓存队列中
private volatile int maximumPoolSize;		//线程池最大线程数
private volatile int poolSize;		//线程池当中的线程数
private volatile RejectedExecutionHandler handler;		//任务拒绝策略
private volatile ThreadFactory threadFactory;		//线程工厂

private int largetPoolSize;		//记录线程池曾今出现过的最大线程数
private long completedTaskCount;		//记录已经执行完的任务个数

4.任务缓存队列及排队策略
BlockingQueue是一个特殊的队列，如果该队列为空，从该队列取值时会被阻塞进入等待状态，知道队列中有数据才唤醒，如果该队列是满的，往里面加数据也会被阻塞，知道有空间才被唤醒。BlockingQueue是一个阻塞队列接口

具体实现由以下几种：
ArrayBlockingQueue：基于数组的阻塞队列，构造函数需传入数组大小，按先进先出的规则；
LinkedBlockingQueue：基于链表的阻塞队列，该链表有大小限制，构造函数若不传大小，则大小默认为Interger.Max_VALUE；按先进先出的规则；
PriorityBlockingQueue：类似LinkedBlockingQueue队列，但是排序规则不是先进先出，而是按对象的自然排序或者构造函数的Compatator决定；
SynchronousQueue：特殊的BlockingQueue，对其操作必须必须是存和取交替完成；

5.任务拒绝策略：



  [1]: ./images/a-1_1.png "a-1"