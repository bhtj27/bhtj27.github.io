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

注意：
1).当线程池中当前线程个数poolSize<corePoolSize，线程池新增一个线程处理新提交的任务；
2).当线程池中的线程数达到corePoolSize，小于maxmumPoolSize时，新提交的任务进入阻塞队列BlockingQueue，
3).如果进入阻塞队列容量达到上限，并且当前线程大小没有达到maxmumPoolSize，则新增线程处理任务；
4).此时队列已满，线程还在增加，当线程数达到maxmumPoolSize时，此时线程池会拒绝新增加的任务，取决于拒绝策略RejectedExecutionHandler；

![enter description here][2]

![enter description here][3]

![enter description here][4]

上面的源代码中，有四个ThreadPoolExecutor的构造函数，参数如图所示。最后两个参数分别是线程池工厂和拒绝策略，若这个参数不输入则采用默认的线程工厂和拒绝策略，如上图，默认的拒绝策略为AbortPolicy

5.任务拒绝策略：
AbortPolicy：丢弃任务并抛出RejectedExecutionException异常；
DiscardPolicy：也是丢弃任务，但是不抛出异常；
DiscardOldEstPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复）；
CallerRunsPolicy：由调用线程处理该任务；

AbortPolicy等四种拒绝策略都是ThreadPoolExecutor的内部类：
``` stylus
public static class AbortPolicy implements RejectedExecutionHandler {
        /**
         * Creates an {@code AbortPolicy}.
         */
        public AbortPolicy() { }

        /**
         * Always throws RejectedExecutionException.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         * @throws RejectedExecutionException always
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException("Task " + r.toString() +
                                                 " rejected from " +
                                                 e.toString());
        }
    }
``` 

``` stylus
public static class DiscardPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code DiscardPolicy}.
         */
        public DiscardPolicy() { }

        /**
         * Does nothing, which has the effect of discarding task r.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        }
    }
``` 

``` stylus
     /**
         * Creates a {@code DiscardOldestPolicy} for the given executor.
         */
        public DiscardOldestPolicy() { }

        /**
         * Obtains and ignores the next task that the executor
         * would otherwise execute, if one is immediately available,
         * and then retries execution of task r, unless the executor
         * is shut down, in which case task r is instead discarded.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                e.getQueue().poll();
                e.execute(r);
            }
        }
    }
```

``` stylus
public static class CallerRunsPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code CallerRunsPolicy}.
         */
        public CallerRunsPolicy() { }

        /**
         * Executes task r in the caller's thread, unless the executor
         * has been shut down, in which case the task is discarded.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                r.run();
            }
        }
    }
```

6.线程池关闭
ThreadPoolExecutor提供了两个方法用于关闭线程池，分别是shutdown和shutdownNow;
shutdown：不会立即关闭线程池，而是等所有任务缓存队列中的任务都执行完毕才终止，但是也不会接收新任务；
shutdownNow：立即终止线程池，并尝试打断正在执行的任务，并清空任务缓存队列，返回尚未执行的任务

7.线程池容量的动态调整：
ThreadPoolExecutor类提供了动态调整线程池容量大小的方法：setCorePoolSize()和setMaximumPoolSize()，当上面参数有小变大时还可以立即创建线程

8.实例

``` stylus
package com.itao.study.test;

import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class ThreadPoolTest {

	public static void main(String[] args) {
		ThreadPoolExecutor executor=new ThreadPoolExecutor(5, 10, 200, TimeUnit.MILLISECONDS, 
				new ArrayBlockingQueue<Runnable>(5));
		
		for(int i=0;i<15;i++){
			MyTask mytask=new MyTask(i);
			executor.execute(mytask);
			try{
				Thread.sleep(10);
			}catch(Exception e){
				
			}
			System.out.println("线程池中线程数目："+executor.getPoolSize()+",队列中等待执行的任务数目："+
			executor.getQueue().size()+",已执行的线程数目："+executor.getCompletedTaskCount());
		}
		
		executor.shutdown();

	}

}

class MyTask implements Runnable {
	private int taskNum;
	
	public MyTask(int taskNum){
		this.taskNum=taskNum;
	}

	@Override
	public void run() {
		System.out.println("正在执行task："+taskNum);
		
		try{
			Thread.sleep(400000);
		}catch(Exception e){
			e.printStackTrace();
		}
		
		System.out.println("task "+taskNum+"执行完毕");
	}
	
	
	
}
```

![enter description here][5]

该实例中，核心线程数5，最大线程数为10，核心线程保持存活时间为200，单位是微秒，任务阻塞队列采用ArrayBolckingQueue，容量为5，默认线程工厂，默认线程池拒绝策略AbortPolicy

任务开始，for循环创建15个线程。
1).核心线程数由0增加到5个，此时核心线程已满；
2).再来的任务进入阻塞队列，阻塞队列此时由0开始增加，知道达到最大容量5；
3).再来任务时，由于任务队列已满，核心线程已满，但是线程数没有达到线程池的最大线程，此时线程池还会创建线程，直到达到最大线程数10（这时候阻塞队列中还有5个线程在等待的，不要忘了）

![enter description here][6]

这个图接上图，当task0任务完毕之后从任务阻塞队列中取出一个继续执行。

注意：
若将循环的线程数改为20，分析可知：核心线程数5，最大线程数10，阻塞队列容量5；还有5个线程是不能执行的，由于采用了AbortPolicy的拒绝策略，会抛出异常，如下所示：

``` stylus
正在执行task：0
线程池中线程数目：1,队列中等待执行的任务数目：0,已执行的线程数目：0
正在执行task：1
线程池中线程数目：2,队列中等待执行的任务数目：0,已执行的线程数目：0
正在执行task：2
线程池中线程数目：3,队列中等待执行的任务数目：0,已执行的线程数目：0
正在执行task：3
线程池中线程数目：4,队列中等待执行的任务数目：0,已执行的线程数目：0
正在执行task：4
线程池中线程数目：5,队列中等待执行的任务数目：0,已执行的线程数目：0
线程池中线程数目：5,队列中等待执行的任务数目：1,已执行的线程数目：0
线程池中线程数目：5,队列中等待执行的任务数目：2,已执行的线程数目：0
线程池中线程数目：5,队列中等待执行的任务数目：3,已执行的线程数目：0
线程池中线程数目：5,队列中等待执行的任务数目：4,已执行的线程数目：0
线程池中线程数目：5,队列中等待执行的任务数目：5,已执行的线程数目：0
正在执行task：10
线程池中线程数目：6,队列中等待执行的任务数目：5,已执行的线程数目：0
正在执行task：11
线程池中线程数目：7,队列中等待执行的任务数目：5,已执行的线程数目：0
正在执行task：12
线程池中线程数目：8,队列中等待执行的任务数目：5,已执行的线程数目：0
正在执行task：13
线程池中线程数目：9,队列中等待执行的任务数目：5,已执行的线程数目：0
正在执行task：14
线程池中线程数目：10,队列中等待执行的任务数目：5,已执行的线程数目：0
Exception in thread "main" java.util.concurrent.RejectedExecutionException: Task com.itao.study.test.MyTask@232204a1 rejected from java.util.concurrent.ThreadPoolExecutor@4aa298b7[Running, pool size = 10, active threads = 10, queued tasks = 5, completed tasks = 0]
	at java.util.concurrent.ThreadPoolExecutor$AbortPolicy.rejectedExecution(ThreadPoolExecutor.java:2047)
	at java.util.concurrent.ThreadPoolExecutor.reject(ThreadPoolExecutor.java:823)
	at java.util.concurrent.ThreadPoolExecutor.execute(ThreadPoolExecutor.java:1369)
	at com.itao.study.test.ThreadPoolTest.main(ThreadPoolTest.java:15)
task 0执行完毕
正在执行task：5
task 1执行完毕
正在执行task：6
task 2执行完毕
正在执行task：7
task 3执行完毕
正在执行task：8
task 4执行完毕
正在执行task：9
task 10执行完毕
task 11执行完毕
task 12执行完毕
task 13执行完毕
task 14执行完毕
task 5执行完毕
task 6执行完毕
task 7执行完毕
task 8执行完毕
task 9执行完毕

```




  [1]: ./images/a-1_2.png "a-1"
  [2]: ./images/a-2.png "a-2"
  [3]: ./images/a-3.png "a-3"
  [4]: ./images/a-4.png "a-4"
  [5]: ./images/a-5.png "a-5"
  [6]: ./images/a-6.png "a-6"