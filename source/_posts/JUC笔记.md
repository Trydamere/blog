---
title: JUC笔记
date: 2022-05-01 20:58:32
tags:
- 原创
categories:
- Java
description: 本博客是根据黑马JUC课程所做的笔记
---

# 并发编程


根据[黑马JUC](https://www.bilibili.com/video/BV1TE41177mP?share_source=copy_web)整理的笔记



# 1.概览



{% asset_img 目录.png 目录 %}





# 2.进程与线程



## 2.1 进程与线程



### 进程

- 程序由指令和数据组成，但这些指令要运行，数据要读写，就必须将指令加载至 CPU，数据加载至内存。在
  指令运行过程中还需要用到磁盘、网络等设备。进程就是用来加载指令、管理内存、管理 IO 的
- 当一个程序被运行，从磁盘加载这个程序的代码至内存，这时就开启了一个进程。
- 进程就可以视为程序的一个实例。大部分程序可以同时运行多个实例进程（例如记事本、画图、浏览器
  等），也有的程序只能启动一个实例进程（例如网易云音乐、360 安全卫士等）



### 线程

- 一个进程之内可以分为一到多个线程。
- 一个线程就是一个指令流，将指令流中的一条条指令以一定的顺序交给 CPU 执行
- Java 中，线程作为最小调度单位，进程作为资源分配的最小单位。 在 windows 中进程是不活动的，只是作为线程的容器





### 二者对比



- 进程基本上相互独立的，而线程存在于进程内，是进程的一个子集 
- 进程拥有共享的资源，如内存空间等，供其内部的线程共享 
- 进程间通信较为复杂 
  - 同一台计算机的进程通信称为 IPC（Inter-process communication） 
  - 不同计算机之间的进程通信，需要通过网络，并遵守共同的协议，例如 HTTP 
- 线程通信相对简单，因为它们共享进程内的内存，一个例子是多个线程可以访问同一个共享变量 
- 线程更轻量，线程上下文切换成本一般上要比进程上下文切换低



## 2.2 并行与并发



### 并发

在单核 cpu 下，线程实际还是串行执行的。操作系统中有一个组件叫做任务调度器，将 cpu 的时间片分给不同的程序使用，只是由于 cpu 在线程间（时间片很短）的切换非常快，人类感觉是同时运行的。 一般会将这种线程轮流使用 CPU 的做法称为并发(concurrent)

{% asset_img 并发.png 并发 %}

### 并行

多核 cpu下，每个核（core） 都可以调度运行线程，这时候线程可以是并行的，不同的线程同时使用不同的cpu在执行。

{% asset_img 并行.png 并行 %}



### 应用



####  同步和异步的概念

以调用方的角度讲，如果

- 需要等待结果返回，才能继续运行就是同步
- 不需要等待结果返回，就能继续运行就是异步



#### 应用之提高效率

1. 单核 cpu 下，多线程不能实际提高程序运行效率，只是为了能够在不同的任务之间切换，不同线程轮流使用cpu ，不至于一个线程总占用 cpu，别的线程没法干活
2. 多核 cpu 可以并行跑多个线程，但能否提高程序运行效率还是要分情况的

   - 有些任务，经过精心设计，将任务拆分，并行执行，当然可以提高程序的运行效率。但不是所有计算任务都能拆分（参考后文的【阿姆达尔定律】）
   - 也不是所有任务都需要拆分，任务的目的如果不同，谈拆分和效率没啥意义
3. IO 操作不占用 cpu，只是我们一般拷贝文件使用的是【阻塞 IO】，这时相当于线程虽然不用 cpu，但需要一直等待 IO 结束，没能充分利用线程。所以才有后面的【非阻塞 IO】和【异步 IO】优化



# 3.Java线程



## 3.1 创建和运行线程



### 方法一，直接使用 Thread

```
// 构造方法的参数是给线程指定名字，，推荐给线程起个名字
Thread t1 = new Thread("t1") {
	@Override
 	// run 方法内实现了要执行的任务
 	public void run() {
 		log.debug("hello");
 	}
 };
t1.start();
```



###  方法二，使用 Runnable 配合 Thread

把【线程】和【任务】（要执行的代码）分开

- Thread 代表线程
- Runnable 可运行的任务（线程要执行的代码）



```
Runnable runnable = new Runnable() {
	public void run(){
	// 要执行的任务
	}
};
// 创建线程对象
Thread t = new Thread( runnable );
// 启动线程
t.start();
```



### 小结

- 方法1 是把线程和任务合并在了一起，方法2 是把线程和任务分开了
- 用 Runnable 更容易与线程池等高级 API 配合
- 用 Runnable 让任务类脱离了 Thread 继承体系，更灵活



### 方法三，FutureTask 配合 Thread



FutureTask 能够接收 Callable 类型的参数，用来处理有返回结果的情况

```
// 创建任务对象
FutureTask<Integer> task3 = new FutureTask<>(() -> {
	log.debug("hello");
	return 100;
});

// 参数1 是任务对象; 参数2 是线程名字，推荐
new Thread(task3, "t3").start();

// 主线程阻塞，同步等待 task 执行完毕的结果
Integer result = task3.get();
log.debug("结果是:{}", result);
```

Future就是对于具体的Runnable或者Callable任务的执行结果进行取消、查询是否完成、获取结果。必要时可以通过get方法获取执行结果，该方法会阻塞直到任务返回结果。

```
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

Future提供了三种功能： 　　

1. 判断任务是否完成； 　　
2. 能够中断任务； 　　
3. 能够获取任务执行结果。

[FutureTask是Future和Runable的实现](https://gitee.com/link?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FRX5rVuGr6Ab0SmKigmZEag)



## 3.2 观察多个线程同时运行

主要是理解

- 交替执行
- 谁先谁后，不由我们控制



## 3.3 查看进程线程的方法



### linux

- `ps -fe` 查看所有进程
- `ps -fT -p <PID> `查看某个进程（PID）的所有线程
- `kill` 杀死进程
- `top` 按大写 H 切换是否显示线程
- `top -H -p <PID>` 查看某个进程（PID）的所有线程



## 3.4 线程运行原理



### 栈与栈帧


每个线程启动后，虚拟机就会为其分配一块栈内存。每个方法被执行的时候都会同时创建一个**栈帧(stack frame)**用于存储**局部变量表、操作数栈、动态链接、方法出口**等信息，是属于线程的私有的。当java中使用多线程时，每个线程都会维护它自己的栈，每个栈由多个栈帧（Frame）组成，对应着每次方法调用时所占用的内存。每个线程只能有一个活动栈帧，对应着当前正在执行的那个方法。





### 线程上下文切换

因为以下一些原因导致 cpu 不再执行当前的线程，转而执行另一个线程的代码

- 线程的 cpu 时间片用完

- 垃圾回收

- 有更高优先级的线程需要运行

- 线程自己调用了 sleep、yield、wait、join、park、synchronized、lock 等方法

当 Context Switch 发生时，需要由操作系统保存当前线程的状态，并恢复另一个线程的状态，Java 中对应的概念就是程序计数器，它的作用是记住下一条 jvm 指令的执行地址，是线程私有的

- 状态包括程序计数器、虚拟机栈中每个栈帧的信息，如局部变量、操作数栈、返回地址等
- Context Switch 频繁发生会影响性能





## 3.5 常见方法



{% asset_img 常见方法.png 常见方法 %}





## 3.6 start 与 run



- 直接调用 run 是在主线程中执行了 run，没有启动新的线程
- 使用 start 是启动新的线程，通过新的线程间接执行 run 中的代码





## 3.7 sleep 与 yield



**sleep**

1. 调用 sleep 会让当前线程从 Running 进入 Timed Waiting 状态（阻塞）

2. 其它线程可以使用 interrupt 方法打断正在睡眠的线程，这时 sleep 方法会抛出 `InterruptedException`

3. 睡眠结束后的线程未必会立刻得到执行

4. 建议用 TimeUnit 的 sleep 代替 Thread 的 sleep 来获得更好的可读性

  

**yield**

1. 调用 yield 会让当前线程从 Running 进入 Runnable 就绪状态，然后调度执行其它线程
2. 具体的实现依赖于操作系统的任务调度器



**线程优先级**

- 线程优先级会提示（hint）调度器优先调度该线程，但它仅仅是一个提示，调度器可以忽略它
- 如果 cpu 比较忙，那么优先级高的线程会获得更多的时间片，但 cpu 闲时，优先级几乎没作用



## 3.8 join



用于等待某个线程结束。哪个线程内调用join()方法，就等待哪个线程结束，然后再去执行其他线程。

如在主线程中调用ti.join()，则是主线程等待t1线程结束



## 3.9 interrupt



用于打断**阻塞**(sleep wait join)的线程。 处于阻塞状态的线程，CPU不会给其分配时间片

- 如果一个线程在在运行中被打断，打断标记会被置为true。线程**不会停止**，会继续执行。如果要让线程在被打断后停下来，需要**使用打断标记来判断**。
- 如果是打断因sleep wait join方法而被阻塞的线程，会将打断标记置为false。线程抛出异常InterruptedException





### **interrupt方法的应用**——两阶段终止模式



当我们在执行线程一时，想要终止线程二，这是就需要使用interrupt方法来**优雅**的停止线程二。



{% asset_img 两阶段终止模式.png 两阶段终止模式 %}

**代码**

```
public class Test7 {
	public static void main(String[] args) throws InterruptedException {
		Monitor monitor = new Monitor();
		monitor.start();
		Thread.sleep(3500);
		monitor.stop();
	}
}

class Monitor {

	Thread monitor;

	/**
	 * 启动监控器线程
	 */
	public void start() {
		//设置线控器线程，用于监控线程状态
		monitor = new Thread() {
			@Override
			public void run() {
				//开始不停的监控
				while (true) {
                    //判断当前线程是否被打断了
					if(Thread.currentThread().isInterrupted()) {
						System.out.println("处理后续任务");
                        //终止线程执行
						break;
					}
					System.out.println("监控器运行中...");
					try {
						//线程休眠
						Thread.sleep(1000);
					} catch (InterruptedException e) {
						e.printStackTrace();
						//如果是在休眠的时候被打断，不会将打断标记设置为true，这时要重新设置打断标记
						Thread.currentThread().interrupt();
					}
				}
			}
		};
		monitor.start();
	}

	/**
	 * 	用于停止监控器线程
	 */
	public void stop() {
		//打断线程
		monitor.interrupt();
	}
}
```



## 3.10 sleep，yiled，wait，join 对比


- sleep，join，yield，interrupted是Thread类中的方法
- wait/notify是object中的方法
- sleep 不释放锁、释放cpu
- join 释放锁、join的线程抢占cpu，如t1.join(), t1抢占cpu
- yield 不释放锁、释放cpu
- wait 释放锁、释放cpu



## 3.11 主线程与守护线程



默认情况下，Java 进程需要等待所有线程都运行结束，才会结束。有一种特殊的线程叫做守护线程，只要其它非守护线程运行结束了，即使守护线程的代码没有执行完，也会强制结束。



> 注意
> 垃圾回收器线程就是一种守护线程



## 3.12 五种状态

{% asset_img 五种状态.png 五种状态 %}



- 【初始状态】仅是在语言层面创建了线程对象，还未与操作系统线程关联
- 【可运行状态】（就绪状态）指该线程已经被创建（与操作系统线程关联），可以由 CPU 调度执行
- 【运行状态】指获取了 CPU 时间片运行中的状态
  - 当 CPU 时间片用完，会从【运行状态】转换至【可运行状态】，会导致线程的上下文切换
- 【阻塞状态】
  - 如果调用了阻塞 API，如 BIO 读写文件，这时该线程实际不会用到 CPU，会导致线程上下文切换，进入【阻塞状态】
  - 等 BIO 操作完毕，会由操作系统唤醒阻塞的线程，转换至【可运行状态】
  - 与【可运行状态】的区别是，对【阻塞状态】的线程来说只要它们一直不唤醒，调度器就一直不会考虑调度它们
- 【终止状态】表示线程已经执行完毕，生命周期已经结束，不会再转换为其它状态



## 3.13 六种状态



这是从 **Java API** 层面来描述的
根据 Thread.State 枚举，分为六种状态

{% asset_img 六种状态.png 六种状态 %}

- `NEW` 线程刚被创建，但是还没有调用 `start()` 方法
- `RUNNABLE` 当调用了 `start()` 方法之后，注意，**Java API** 层面的 RUNNABLE 状态涵盖了 **操作系统** 层面的【可运行状态】、【运行状态】和【阻塞状态】（由于 BIO 导致的线程阻塞，在 Java 里无法区分，仍然认为是可运行）
- `BLOCKED`， `WAITING` ， `TIMED_WAITING` 都是 Java API 层面对【阻塞状态】的细分，后面会在状态转换一节详述
- `TERMINATED` 当线程代码运行结束



# 4. 共享模型之管程



## 4.1 共享带来的问题



### 临界区 Critical Section

- 一个程序运行多个线程本身是没有问题的
- 问题出在多个线程访问**共享资源**
  - 多个线程读**共享资源**其实也没有问题
  - 在多个线程对**共享资源**读写操作时发生指令交错，就会出现问题
- 一段代码块内如果存在对**共享资源**的多线程读写操作，称这段代码块为**临界区**



### 竞态条件 Race Condition

多个线程在临界区内执行，由于代码的**执行序列不同**而导致结果无法预测，称之为发生了**竞态条件**





## 4.2 synchronized 解决方案



为了避免临界区的竞态条件发生，有多种手段可以达到目的。

- 阻塞式的解决方案：synchronized，Lock
- 非阻塞式的解决方案：原子变量



本次课使用阻塞式的解决方案：synchronized，来解决上述问题，即俗称的【对象锁】，它采用互斥的方式让同一时刻至多只有一个线程能持有【对象锁】，其它线程再想获取这个【对象锁】时就会阻塞住。这样就能保证拥有锁的线程可以安全的执行临界区内的代码，不用担心线程上下文切换



> 注意
>
> 虽然 java 中互斥和同步都可以采用 synchronized 关键字来完成，但它们还是有区别的：
>
> - 互斥是保证临界区的竞态条件发生，同一时刻只能有一个线程执行临界区代码
> - 同步是由于线程执行的先后、顺序不同、需要一个线程等待其它线程运行到某个点



### synchronized

语法

```
synchronized(对象) {
	//临界区
}
```



## 4.3 方法上的 synchronized



- 加在成员方法上，锁this对象

```
public class Demo {
	//在方法上加上synchronized关键字
	public synchronized void test() {
	
	}
	//等价于
	public void test() {
		synchronized(this) {
		
		}
	}
}
```



- 加在静态方法上，锁Class对象

```
public class Demo {
	//在静态方法上加上synchronized关键字
	public synchronized static void test() {
	
	}
	//等价于
	public void test() {
		synchronized(Demo.class) {
		
		}
	}
}
```



## 4.4 变量的线程安全分析



### 成员变量和静态变量是否线程安全？

- 如果它们没有共享，则线程安全

- 如果它们被共享了，根据它们的状态是否能够改变，又分两种情况

  - 如果只有读操作，则线程安全

  - 如果有读写操作，则这段代码是临界区，需要考虑线程安全



### 局部变量是否线程安全？

- 局部变量是线程安全的--(每个方法都在对应线程的栈中创建栈帧，不会被其他线程共享)
- 但局部变量引用的对象则未必
  - 如果该对象没有逃离方法的作用访问，它是线程安全的
  - 如果该对象逃离方法的作用范围，需要考虑线程安全



### 常见线程安全类



- String
- Integer
- StringBuffer
- Random
- Vector
- Hashtable
- java.util.concurrent 包下的类

这里说它们是线程安全的是指，多个线程调用它们**同一个实例的某个方法**时，是线程安全的

- 它们的每个方法是原子的（都被加上了synchronized）
- 但注意它们**多个方法的组合不是原子**的，所以可能会出现线程安全问题



#### 不可变类线程安全性

String、Integer 等都是**不可变类**，因为其内部的状态不可以改变，因此它们的方法都是线程安全的



有同学或许有疑问，String 有 replace，substring 等方法【可以】改变值啊，那么这些方法又是如何保证线程安 全的呢？

这是因为这些方法的返回值都**创建了一个新的对象**，而不是直接改变String、Integer对象本身。



## 4.6 Monitor 概念



### 原理之Monitor

Monitor 被翻译为**监视器**或**管程**



每个 Java 对象都可以关联一个 Monitor 对象，如果使用 synchronized 给对象上锁（重量级）之后，该对象头的
Mark Word 中就被设置指向 Monitor 对象的指针



Monitor 结构如下

{% asset_img monitor原理.png monitor原理 %}

- 刚开始 Monitor 中 Owner 为 null
- 当 Thread-2 执行 synchronized(obj) 就会将 Monitor 的所有者 Owner 置为 Thread-2，Monitor中只能有一个 Owner
- 在 Thread-2 上锁的过程中，如果 Thread-3，Thread-4，Thread-5 也来执行 synchronized(obj)，就会进入EntryList BLOCKED
- Thread-2 执行完同步代码块的内容，然后唤醒 EntryList 中等待的线程来竞争锁，竞争的时是非公平的
- 图中 WaitSet 中的 Thread-0，Thread-1 是之前获得过锁，但条件不满足进入 WAITING 状态的线程，后面讲wait-notify 时会分析

> 注意：
>
> - synchronized 必须是进入同一个对象的 monitor 才有上述的效果
> - 不加 synchronized 的对象不会关联监视器，不遵从以上规则



### synchronized 原理进阶



#### 对象头格式

{% asset_img 对象头格式.png 对象头格式 %}



#### 1. 轻量级锁

轻量级锁的使用场景：如果一个对象虽然有多线程要加锁，但加锁的时间是错开的（也就是没有竞争），那么可以使用轻量级锁来优化。
轻量级锁对使用者是透明的，即语法仍然是 `synchronized`
假设有两个方法同步块，利用同一个对象加锁

```
static final Object obj = new Object();
public static void method1() {
	synchronized( obj ) {
	// 同步块 A
		method2();
	}
}
public static void method2() {
	synchronized( obj ) {
		// 同步块 B
	}
}
```

- 创建锁记录（Lock Record）对象，每个线程都的栈帧都会包含一个锁记录的结构，内部可以存储锁定对象的Mark Word

{% asset_img 创建锁记录.png 创建锁记录 %}

- 让锁记录中 Object reference 指向锁对象，并尝试用 cas 替换Object 的 Mark Word，将 Mark Word 的值存入锁记录

{% asset_img 替换markword.png 替换markword %}

- 如果 cas 替换成功，对象头中存储了锁记录地址和状态 00 ，表示由该线程给对象加锁，这时图示如下

{% asset_img 替换成功.png 替换成功 %}

- 如果 cas 失败，有两种情况
  - 如果是其它线程已经持有了该 Object 的轻量级锁，这时表明有竞争，进入锁膨胀过程
  - 如果是自己执行了 synchronized 锁重入，那么再添加一条 Lock Record 作为重入的计数

{% asset_img cas失败.png cas失败 %}

- 当退出 synchronized 代码块（解锁时）如果有取值为 null 的锁记录，表示有重入，这时重置锁记录，表示重入计数减一

{% asset_img 重入计数减一.png 重入计数减一 %}

- 当退出 synchronized 代码块（解锁时）锁记录的值不为 null，这时使用 cas 将 Mark Word 的值恢复给对象头
  - 成功，则解锁成功
  - 失败，说明轻量级锁进行了锁膨胀或已经升级为重量级锁，进入重量级锁解锁流程



#### 2. 锁膨胀

如果在尝试加轻量级锁的过程中，CAS 操作无法成功，这时一种情况就是有其它线程为此对象加上了轻量级锁（有竞争），这时需要进行锁膨胀，将轻量级锁变为重量级锁。



- 当 Thread-1 进行轻量级加锁时，Thread-0 已经对该对象加了轻量级锁

{% asset_img 加轻量级锁失败.png 加轻量级锁失败 %}

- 这时 Thread-1 加轻量级锁失败，进入锁膨胀流程
  - 即为 Object 对象申请 Monitor 锁，让 Object 指向重量级锁地址
  - 然后自己进入 Monitor 的 EntryList BLOCKED

{% asset_img 锁膨胀.png 锁膨胀 %}

- Thread-0 退出同步块解锁时，使用 cas 将 Mark Word 的值恢复给对象头，失败。这时会进入重量级解锁流程，即按照 Monitor 地址找到 Monitor 对象，设置 Owner 为 null，唤醒 EntryList 中 BLOCKED 线程



#### 3. 自旋优化



重量级锁竞争的时候，还可以使用自旋来进行优化，如果当前线程**自旋成功**（即这时候持锁线程已经退出了同步块，**释放了锁**），这时当前线程就可以避免阻塞。



自旋会占用 CPU 时间，单核 CPU 自旋就是浪费，多核 CPU 自旋才能发挥优势。



#### 4. 偏向锁



轻量级锁在没有竞争时（就自己这个线程），每次重入仍然需要执行 CAS 操作。
引入了偏向锁来做进一步优化：只有第一次使用 CAS 将线程 ID 设置到对象的 Mark Word 头，之后发现这个线程 ID 是自己的就表示没有竞争，不用重新 CAS。以后只要不发生竞争，这个对象就归该线程所有

{% asset_img 偏向锁.png 偏向锁 %}



#### 偏向状态

- Normal：一般状态，没有加任何锁，前面62位保存的是对象的信息，**最后2位为状态（01），倒数第三位表示是否使用偏向锁（未使用：0）**
- Biased：偏向状态，使用偏向锁，前面54位保存的当前线程的ID，**最后2位为状态（01），倒数第三位表示是否使用偏向锁（使用：1）**
- Lightweight：使用轻量级锁，前62位保存的是锁记录的指针，**最后两位为状态（00）**
- Heavyweight：使用重量级锁，前62位保存的是Monitor的地址指针，**后两位为状态(10)**

{% asset_img 对象头格式.png 对象头格式 %}

一个对象创建时：

- 如果开启了偏向锁（默认开启），那么对象创建后，markword 值为 0x05 即最后 3 位为 101，这时它的thread、epoch、age 都为 0
- 偏向锁是默认是延迟的，不会在程序启动时立即生效，如果想避免延迟，可以加 VM 参数 -XX:BiasedLockingStartupDelay=0 来禁用延迟
- 如果没有开启偏向锁，那么对象创建后，markword 值为 0x01 即最后 3 位为 001，这时它的 hashcode、age 都为 0，第一次用到 hashcode 时才会赋值



## 4.7 wait notify

{% asset_img wait-notify.png wait-notify %}

- Owner 线程发现条件不满足，调用 wait 方法，即可进入 WaitSet 变为 WAITING 状态
- BLOCKED 和 WAITING 的线程都处于阻塞状态，不占用 CPU 时间片。但是有所区别：
  - BLOCKED状态的线程是在竞争对象时，发现Monitor的Owner已经是别的线程了，此时就会进入EntryList中，并处于BLOCKED状态
  - WAITING状态的线程是获得了对象的锁，但是自身因为某些原因需要进入阻塞状态时，锁对象调用了wait方法而进入了WaitSet中，处于WAITING状态
- BLOCKED 线程会在 Owner 线程释放锁时唤醒
- WAITING 线程会在 Owner 线程调用 notify 或 notifyAll 时唤醒，但唤醒后并不意味者立刻获得锁，仍需进入EntryList 重新竞争



**注：只有当对象被锁以后，才能调用wait和notify方法**



## 4.8 wait notify 的正确姿势

### Wait与Sleep的区别

**不同点**

- Sleep是Thread类的静态方法，Wait是Object的方法，Object又是所有类的父类，所以所有类都有Wait方法。
- Sleep在阻塞的时候不会释放锁，而Wait在阻塞的时候会释放锁
- Sleep不需要与synchronized一起使用，而Wait需要与synchronized一起使用（对象被锁以后才能使用）

**相同点**

- 阻塞状态都为**TIMED_WAITING**



### 优雅地使用wait/notify

**什么时候适合使用wait**

- 当线程**不满足某些条件**，需要暂停运行时，可以使用wait。这样会将**对象的锁释放**，让其他线程能够继续运行。如果此时使用sleep，会导致所有线程都进入阻塞，导致所有线程都没法运行，直到当前线程sleep结束后，运行完毕，才能得到执行。

**使用wait/notify需要注意什么**

- 当有**多个**线程在运行时，对象调用了wait方法，此时这些线程都会进入WaitSet中等待。如果这时使用了**notify**方法， 只能随机唤醒一个 WaitSet 中的线程，这时如果有其它线程也在等待，那么就可能唤醒不了正确的线程，改为 notifyAll
- **虚假唤醒**（唤醒的不是满足条件的等待线程），**在wait端必须使用while来等待条件变量而不能使用if语句**

```
synchronized (LOCK) {
	while(//不满足条件，一直等待，避免虚假唤醒) {
		LOCK.wait();
	}
	//满足条件后再运行
}

synchronized (LOCK) {
	//唤醒所有等待线程
	LOCK.notifyAll();
}
```



## 模式之保护性暂停



### 1. 定义

即 Guarded Suspension，用在一个线程等待另一个线程的执行结果



要点

- 有一个结果需要从一个线程传递到另一个线程，让他们关联同一个 GuardedObject
- 如果有结果不断从一个线程到另一个线程那么可以使用消息队列（见生产者/消费者）
- JDK 中，join 的实现、Future 的实现，采用的就是此模式
- 因为要等待另一方的结果，因此归类到同步模式

{% asset_img 两阶段终止模式.png 两阶段终止模式 %}



### join源码——使用保护性暂停模式

```
public final synchronized void join(long millis)
    throws InterruptedException {
        long base = System.currentTimeMillis();
        long now = 0;

        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (millis == 0) {
            while (isAlive()) {
                wait(0);
            }
        } else {
            while (isAlive()) {
                long delay = millis - now;
                if (delay <= 0) {
                    break;
                }
                wait(delay);
                now = System.currentTimeMillis() - base;
            }
        }
    }
```



## 模式之生产者消费者

### 实现

```
public class Test21 {

    public static void main(String[] args) {
        MessageQueue queue = new MessageQueue(2);

        for (int i = 0; i < 3; i++) {
            int id = i;
            new Thread(() -> {
                queue.put(new Message(id , "值"+id));
            }, "生产者" + i).start();
        }

        new Thread(() -> {
            while(true) {
                sleep(1);
                Message message = queue.take();
            }
        }, "消费者").start();
    }

}

// 消息队列类 ， java 线程之间通信
@Slf4j(topic = "c.MessageQueue")
class MessageQueue {
    // 消息的队列集合
    private LinkedList<Message> list = new LinkedList<>();
    // 队列容量
    private int capcity;

    public MessageQueue(int capcity) {
        this.capcity = capcity;
    }

    // 获取消息
    public Message take() {
        // 检查队列是否为空
        synchronized (list) {
            while(list.isEmpty()) {
                try {
                    log.debug("队列为空, 消费者线程等待");
                    list.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            // 从队列头部获取消息并返回
            Message message = list.removeFirst();
            log.debug("已消费消息 {}", message);
            list.notifyAll();
            return message;
        }
    }

    // 存入消息
    public void put(Message message) {
        synchronized (list) {
            // 检查对象是否已满
            while(list.size() == capcity) {
                try {
                    log.debug("队列已满, 生产者线程等待");
                    list.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            // 将消息加入队列尾部
            list.addLast(message);
            log.debug("已生产消息 {}", message);
            list.notifyAll();
        }
    }
}

final class Message {
    private int id;
    private Object value;

    public Message(int id, Object value) {
        this.id = id;
        this.value = value;
    }

    public int getId() {
        return id;
    }

    public Object getValue() {
        return value;
    }

    @Override
    public String toString() {
        return "Message{" +
                "id=" + id +
                ", value=" + value +
                '}';
    }
}
```



## 4.9 Park & Unpark



### 基本使用

是LockSupport类中的的方法

```
//暂停线程运行
LockSupport.park;

//恢复线程运行
LockSupport.unpark(thread);
```

### 特点

**与wait/notify的区别**

- wait，notify 和 notifyAll 必须配合**Object Monitor**一起使用，而park，unpark不必
- park ，unpark 是以**线程为单位**来**阻塞**和**唤醒**线程，而 notify 只能随机唤醒一个等待线程，notifyAll 是唤醒所有等待线程，就不那么精确
- park & unpark 可以**先unpark**，而 wait & notify 不能先 notify
- **park不会释放锁**，而wait会释放锁

### 原理

每个线程都有一个自己的**Park对象**，并且该对象**_counter, _cond,__mutex**组成

- 先调用park再调用unpark时
  - 先调用park
    - 线程运行时，会将Park对象中的**_counter的值设为0**；
    - 调用park时，会先查看counter的值是否为0，如果为0，则将线程放入阻塞队列cond中
    - 放入阻塞队列中后，会**再次**将counter设置为0
  - 然后调用unpark
    - 调用unpark方法后，会将counter的值设置为1
    - 去唤醒阻塞队列cond中的线程
    - 线程继续运行并将counter的值设为0



{% asset_img 先调用park再调用unpark(1).png 先调用park再调用unpark(1) %}



{% asset_img 先调用park再调用unpark(2).png 先调用park再调用unpark(2) %}

- 先调用unpark，再调用park
  - 调用unpark
    - 会将counter设置为1（运行时0）
  - 调用park方法
    - 查看counter是否为0
    - 因为unpark已经把counter设置为1，所以此时将counter设置为0，但**不放入**阻塞队列cond中



{% asset_img 先调用unpark，再调用park.png 先调用unpark，再调用park %}



## 4.10 重新理解线程状态转换

{% asset_img 重新理解线程状态转换.png 重新理解线程状态转换 %}



### 情况1 `NEW –> RUNNABLE`

- 当调用了`t.start()`方法时，由`NEW –> RUNNABLE`

### 情况2 `RUNNABLE <–> WAITING`

- t 线程用 `synchronized(obj)`获取了对象锁后
  - 调用 `obj.wait()` 方法时，t 线程从 `RUNNABLE –> WAITING`
  - 调用 `obj.notify()` ， `obj.notifyAll()` ， `t.interrupt() `时
    - 竞争锁成功，t 线程从 `WAITING –> RUNNABLE`
    - 竞争锁失败，t 线程从 `WAITING –> BLOCKED`

### 情况3 `RUNNABLE <–> WAITING`

- 当前线程调用`t.join()`方法时，当前线程从 `RUNNABLE –> WAITING`
- t 线程**运行结束**，或调用了**当前线程**的 `interrupt()` 时，当前线程从 `WAITING –> RUNNABLE`

### 情况4 `RUNNABLE <–> WAITING`

- 当前线程调用 `LockSupport.park()` 方法会让当前线程从 `RUNNABLE –> WAITING`
- 调用` LockSupport.unpark(目标线程)` 或调用了线程 的 `interrupt()` ，会让目标线程从 `WAITING –> RUNNABLE`

### 情况5 `RUNNABLE <–>TIMED_WAITING`

t 线程用 `synchronized(obj)` 获取了对象锁后

- 调用 `obj.wait(long n)` 方法时，t 线程从 `RUNNABLE –>TIMED_WAITING`
- t 线程等待时间超过了 n 毫秒，或调用 `obj.notify()` ， `obj.notifyAll()` ， `t.interrupt()` 时
  - 竞争锁成功，t 线程从 TIMED_WAITING –> RUNNABLE
  - 竞争锁失败，t 线程从 TIMED_WAITING –> BLOCKED

### 情况6 `RUNNABLE <–> TIMED_WAITING`

- 当前线程调用 `t.join(long n)` 方法时，当前线程从 `RUNNABLE –> TIMED_WAITING`
- 当前线程等待时间超过了 n 毫秒，或t 线程运行结束，或调用了当前线程的 `interrupt()` 时，当前线程从 `TIMED_WAITING –> RUNNABLE`

### 情况7 `RUNNABLE <–> TIMED_WAITING`

- 当前线程调用 `Thread.sleep(long n)` ，当前线程从 `RUNNABLE –> TIMED_WAITING`
- 当前线程等待时间超过了 n 毫秒，当前线程从 `TIMED_WAITING –> RUNNABLE`

### 情况8 `RUNNABLE <–> TIMED_WAITING`

- 当前线程调用 `LockSupport.parkNanos(long nanos)` 或 `LockSupport.parkUntil(long millis)` 时，当前线 程从`RUNNABLE –> TIMED_WAITING`
- 调用 `LockSupport.unpark(目标线程)` 或调用了线程的 `interrupt()`，或是等待超时，会让目标线程从 `TIMED_WAITING–>RUNNABLE`

### 情况9 `RUNNABLE <–> BLOCKED`

- t 线程用 `synchronized(obj)` 获取了对象锁时如果**竞争失败**，从 `RUNNABLE –> BLOCKED`
- 持 obj 锁线程的同步代码块执行完毕，会唤醒该对象上所有 `BLOCKED` 的线程重新竞争，如果其中 t 线程竞争 成功，从 `BLOCKED –> RUNNABLE` ，其它**失败**的线程仍然 `BLOCKED`

### 情况10 `RUNNABLE <–> TERMINATED`

当前线**程所有代码运行完毕**，进入 `TERMINATED`



## 4.11 多把锁



将锁的粒度细分

- 好处，是可以增强并发度
- 坏处，如果一个线程需要同时获得多把锁，就容易发生死锁



```
class BigRoom {
    //额外创建对象来作为锁
	private final Object studyRoom = new Object();
	private final Object bedRoom = new Object();
}
```

## 4.12 活跃性



### 死锁

有这样的情况：一个线程需要同时获取多把锁，这时就容易发生死锁

`t1 线程` 获得 `A对象` 锁，接下来想获取 `B对象` 的锁 `t2 线程` 获得 `B对象` 锁，接下来想获取 `A对象` 的锁

```
public static void main(String[] args) {
		final Object A = new Object();
		final Object B = new Object();
		new Thread(()->{
			synchronized (A) {
				try {
					Thread.sleep(2000);
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
				synchronized (B) {

				}
			}
		}).start();

		new Thread(()->{
			synchronized (B) {
				try {
					Thread.sleep(1000);
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
				synchronized (A) {

				}
			}
		}).start();
	}
```



#### 发生死锁的必要条件

- 互斥条件
  - 在一段时间内，一种资源只能被一个进程所使用
- 请求和保持条件
  - 进程已经拥有了至少一种资源，同时又去申请其他资源。因为其他资源被别的进程所使用，该进程进入阻塞状态，并且不释放自己已有的资源
- 不可抢占条件
  - 进程对已获得的资源在未使用完成前不能被强占，只能在进程使用完后自己释放
- 循环等待条件
  - 发生死锁时，必然存在一个进程——资源的循环链。



### 哲学家就餐问题



有五位哲学家，围坐在圆桌旁。

- 他们只做两件事，思考和吃饭，思考一会吃口饭，吃完饭后接着思考。
- 吃饭时要用两根筷子吃，桌上共有 5 根筷子，每位哲学家左右手边各有一根筷子。
- 如果筷子被身边的人拿着，自己就得等待



#### 避免死锁的方法

在线程使用锁对象时**，顺序加锁**即可避免死锁



### 活锁

活锁出现在两个线程互相改变对方的结束条件，最后谁也无法结束。



#### 避免活锁的方法

在线程执行时，中途给予**不同的间隔时间**即可。





#### 死锁与活锁的区别

- 死锁是因为线程互相持有对象想要的锁，并且都不释放，最后**线程阻塞**，**停止运行**的现象。
- 活锁是因为线程间修改了对方的结束条件，而导致代码**一直在运行**，却一直**运行不完**的现象。



### 饥饿

某些线程因为优先级太低，导致一直无法获得资源的现象。

在使用顺序加锁时，可能会出现饥饿现象。



## 4.13 ReentrantLock



相对于 synchronized 它具备如下特点

- 可中断
- 可以设置超时时间
- 可以设置为公平锁
- 支持多个条件变量

与 synchronized 一样，都支持可重入

基本语法

```
//获取ReentrantLock对象
private ReentrantLock lock = new ReentrantLock();
//加锁
lock.lock();
try {
	//需要执行的代码
}finally {
	//释放锁
	lock.unlock();
}
```



### 可重入

可重入是指同一个线程如果首次获得了这把锁，那么因为它是这把锁的拥有者，因此有权利再次获取这把锁
如果是不可重入锁，那么第二次获得锁时，自己也会被锁挡住



### 可打断

如果某个线程处于阻塞状态，可以调用其interrupt方法让其停止阻塞，获得锁失败

**简而言之**就是：处于阻塞状态的线程，被打断了就不用阻塞了，直接停止运行



```
public static void main(String[] args) {
		ReentrantLock lock = new ReentrantLock();
		Thread t1 = new Thread(()-> {
			try {
				//加锁，可打断锁
				lock.lockInterruptibly();
			} catch (InterruptedException e) {
				e.printStackTrace();
				log.debug("等锁过程被打断");
				return;
			}finally {
				//释放锁
				lock.unlock();
			}
		});

		lock.lock();
		log.debug("获得锁");
		t1.start();
		try {
			Thread.sleep(1000);
			//打断
			t1.interrupt();
			log.debug("执行打断");
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			lock.unlock();
		}
	}
```



### 锁超时

使用**lock.tryLock**方法会返回获取锁是否成功。如果成功则返回true，反之则返回false。



并且tryLock方法可以**指定等待时间**，参数为：tryLock(long timeout, TimeUnit unit), 其中timeout为最长等待时间，TimeUnit为时间单位



**简而言之**就是：获取失败了、获取超时了或者被打断了，不再阻塞，直接停止运行



不设置等待时间

```
public static void main(String[] args) {
		ReentrantLock lock = new ReentrantLock();
		Thread t1 = new Thread(()-> {
            //未设置等待时间，一旦获取失败，直接返回false
			if(!lock.tryLock()) {
				System.out.println("获取失败");
                //获取失败，不再向下执行，返回
				return;
			}
			System.out.println("得到了锁");
			lock.unlock();
		});


		lock.lock();
		try{
			t1.start();
			Thread.sleep(3000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			lock.unlock();
		}
	}
```

设置等待时间

```
public static void main(String[] args) {
		ReentrantLock lock = new ReentrantLock();
		Thread t1 = new Thread(()-> {
			try {
				//判断获取锁是否成功，最多等待1秒
				if(!lock.tryLock(1, TimeUnit.SECONDS)) {
					System.out.println("获取失败");
					//获取失败，不再向下执行，直接返回
					return;
				}
			} catch (InterruptedException e) {
				e.printStackTrace();
				//被打断，不再向下执行，直接返回
				return;
			}
			System.out.println("得到了锁");
			//释放锁
			lock.unlock();
		});


		lock.lock();
		try{
			t1.start();
			//打断等待
			t1.interrupt();
			Thread.sleep(3000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			lock.unlock();
		}
	}
```

使用 tryLock 解决哲学家就餐问题

```
package cn.itcast.n4.deadlock.v2;

import cn.itcast.n2.util.Sleeper;
import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.locks.ReentrantLock;

public class TestDeadLock {
    public static void main(String[] args) {
        Chopstick c1 = new Chopstick("1");
        Chopstick c2 = new Chopstick("2");
        Chopstick c3 = new Chopstick("3");
        Chopstick c4 = new Chopstick("4");
        Chopstick c5 = new Chopstick("5");
        new Philosopher("苏格拉底", c1, c2).start();
        new Philosopher("柏拉图", c2, c3).start();
        new Philosopher("亚里士多德", c3, c4).start();
        new Philosopher("赫拉克利特", c4, c5).start();
        new Philosopher("阿基米德", c5, c1).start();
    }
}

@Slf4j(topic = "c.Philosopher")
class Philosopher extends Thread {
    Chopstick left;
    Chopstick right;

    public Philosopher(String name, Chopstick left, Chopstick right) {
        super(name);
        this.left = left;
        this.right = right;
    }

    @Override
    public void run() {
        while (true) {
            //　尝试获得左手筷子
            if (left.tryLock()) {
                try {
                    // 尝试获得右手筷子
                    if (right.tryLock()) {
                        try {
                            eat();
                        } finally {
                            right.unlock();
                        }
                    }
                } finally {
                    left.unlock();
                }
            }
        }
    }

    private void eat() {
        log.debug("eating...");
        Sleeper.sleep(1);
    }
}



class Chopstick extends ReentrantLock {
    String name;

    public Chopstick(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "筷子{" + name + '}';
    }
}

```

### 公平锁

在线程获取锁失败，进入阻塞队列时，**先进入**的会在锁被释放后**先获得**锁。这样的获取方式就是**公平**的。

```
//默认是不公平锁，需要在创建时指定为公平锁
ReentrantLock lock = new ReentrantLock(true);
```

### 条件变量

synchronized 中也有条件变量，就是我们讲原理时那个 waitSet 休息室，当条件不满足时进入 waitSet 等待
ReentrantLock 的条件变量比 synchronized 强大之处在于，它是支持多个条件变量的，这就好比

- synchronized 是那些不满足条件的线程都在一间休息室等消
- 而 ReentrantLock 支持多间休息室，有专门等烟的休息室、专门等早餐的休息室、唤醒时也是按休息室来唤醒

使用要点：

- await 前需要获得锁
- await 执行后，会释放锁，进入 conditionObject 等待
- await 的线程被唤醒（或打断、或超时）取重新竞争 lock 锁
- 竞争 lock 锁成功后，从 await 后继续执行

```
static Boolean judge = false;
public static void main(String[] args) throws InterruptedException {
	ReentrantLock lock = new ReentrantLock();
	//获得条件变量
	Condition condition = lock.newCondition();
	new Thread(()->{
		lock.lock();
		try{
			while(!judge) {
				System.out.println("不满足条件，等待...");
				//等待
				condition.await();
			}
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			System.out.println("执行完毕！");
			lock.unlock();
		}
	}).start();

	new Thread(()->{
		lock.lock();
		try {
			Thread.sleep(1);
			judge = true;
			//释放
			condition.signal();
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			lock.unlock();
		}

	}).start();
}
```

### 通过Lock与AQS实现可重入锁

```
public class MyLock implements Lock {
   private static class Sync extends AbstractQueuedSynchronizer {
      @Override
      protected boolean tryAcquire(int arg) {
         if (getExclusiveOwnerThread() == null) {
            if (compareAndSetState(0, 1)) {
               setExclusiveOwnerThread(Thread.currentThread());
               return true;
            }
            return false;
         }

         if (getExclusiveOwnerThread() == Thread.currentThread()) {
            int state = getState();
            compareAndSetState(state, state + 1);
            return true;
         }

         return false;
      }

      @Override
      protected boolean tryRelease(int arg) {
         if (getState() <= 0) {
            throw new IllegalMonitorStateException();
         }

         if (getExclusiveOwnerThread() != Thread.currentThread()) {
            throw new IllegalMonitorStateException();
         }

         int state = getState();
         if (state == 1) {
            setExclusiveOwnerThread(null);
            compareAndSetState(state, 0);
         } else {
            compareAndSetState(state, state - 1);
         }
         return true;
      }

      @Override
      protected boolean isHeldExclusively() {
         return getState() >= 1;
      }

      public Condition newCondition() {
         return new ConditionObject();
      }

   }

   Sync sync = new Sync();

   @Override
   public void lock() {
      sync.acquire(1);
   }

   @Override
   public void lockInterruptibly() throws InterruptedException {
      sync.acquireInterruptibly(1);
   }

   @Override
   public boolean tryLock() {
      return sync.tryAcquire(1);
   }

   @Override
   public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
      return sync.tryAcquireNanos(1, time);
   }

   @Override
   public void unlock() {
      sync.release(1);
   }

   @Override
   public Condition newCondition() {
      return sync.newCondition();
   }
}

class Main {
   static int num = 0;
   public static void main(String[] args) throws InterruptedException, IOException {
      MyLock lock = new MyLock();

      Object syncLock = new Object();

      Thread t1 = new Thread(() -> {
         for (int i = 0; i < 10000; i++) {
            lock.lock();
            try {
               lock.lock();
               try {
                  lock.lock();
                  try {
                     num++;
                  } finally {
                     lock.unlock();
                  }
               } finally {
                  lock.unlock();
               }
            } finally {
               lock.unlock();
            }
         }
      });

      Thread t2 = new Thread(() -> {
         for (int i = 0; i < 10000; i++) {
            lock.lock();
            try {
               lock.lock();
               try {
                  lock.lock();
                  try {
                     num--;
                  } finally {
                     lock.unlock();
                  }
               } finally {
                  lock.unlock();
               }
            } finally {
               lock.unlock();
            }
         }
      });

      t1.start();
      t2.start();
      t1.join();
      t2.join();

      int x = 0;
   }
}
```

## 4.14 ThreadLocal

### 简介

ThreadLocal是JDK包提供的，它提供了线程本地变量，也就是如果你创建了一个ThreadLocal变量，那么**访问这个变量的每个线程都会有这个变量的一个本地副本**。当多个线程操作这个变量时，实际操作的是自己本地内存里面的变量，从而避免了线程安全问题

### 使用

```
public class ThreadLocalStudy {
   public static void main(String[] args) {
      // 创建ThreadLocal变量
      ThreadLocal<String> stringThreadLocal = new ThreadLocal<>();
      ThreadLocal<User> userThreadLocal = new ThreadLocal<>();

      // 创建两个线程，分别使用上面的两个ThreadLocal变量
      Thread thread1 = new Thread(()->{
         // stringThreadLocal第一次赋值
         stringThreadLocal.set("thread1 stringThreadLocal first");
         // stringThreadLocal第二次赋值
         stringThreadLocal.set("thread1 stringThreadLocal second");
         // userThreadLocal赋值
         userThreadLocal.set(new User("Nyima", 20));

         // 取值
         System.out.println(stringThreadLocal.get());
         System.out.println(userThreadLocal.get());
          
          // 移除
		 userThreadLocal.remove();
		 System.out.println(userThreadLocal.get());
      });

      Thread thread2 = new Thread(()->{
         // stringThreadLocal第一次赋值
         stringThreadLocal.set("thread2 stringThreadLocal first");
         // stringThreadLocal第二次赋值
         stringThreadLocal.set("thread2 stringThreadLocal second");
         // userThreadLocal赋值
         userThreadLocal.set(new User("Hulu", 20));

         // 取值
         System.out.println(stringThreadLocal.get());
         System.out.println(userThreadLocal.get());
      });

      // 启动线程
      thread1.start();
      thread2.start();
   }
}

class User {
   String name;
   int age;

   public User(String name, int age) {
      this.name = name;
      this.age = age;
   }

   @Override
   public String toString() {
      return "User{" +
            "name='" + name + '\'' +
            ", age=" + age +
            '}';
   }
}
```

**运行结果**

```
thread1 stringThreadLocal second
thread2 stringThreadLocal second
User{name='Nyima', age=20}
User{name='Hulu', age=20}
null
```

从运行结果可以看出

- 每个线程中的ThreadLocal变量是每个线程私有的，而不是共享的
  - 从线程1和线程2的打印结果可以看出
- ThreadLocal其实就相当于其泛型类型的一个变量，只不过是每个线程私有的
  - stringThreadLocal被赋值了两次，保存的是最后一次赋值的结果
- ThreadLocal可以进行以下几个操作
  - set 设置值
  - get 取出值
  - remove 移除值

### 原理

#### Thread中的threadLocals

```
public class Thread implements Runnable {
 ...

 ThreadLocal.ThreadLocalMap threadLocals = null;

 // 放在后面说
 ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;

 ...
}
```

```
static class ThreadLocalMap {
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
```

可以看出Thread类中有一个threadLocals和一个inheritableThreadLocals，它们都是ThreadLocalMap类型的变量，而ThreadLocalMap是一个定制化的Hashmap。在默认情况下，每个线程中的这两个变量都为null。此处先讨论threadLocals，inheritableThreadLocals放在后面讨论

#### **总结**

在每个线程内部都有一个名为threadLocals的成员变量，该变量的类型为HashMap，其中**key为我们定义的ThreadLocal变量的this引用，value则为我们使用set方法设置的值**。每个线程的本地变量存放在线程自己的内存变量threadLocals中

只有当前线程**第一次调用ThreadLocal的set或者get方法时才会创建threadLocals**（inheritableThreadLocals也是一样）。其实每个线程的本地变量不是存放在ThreadLocal实例里面，而是存放在调用线程的threadLocals变量里面

## 4.15 InheritableThreadLocal

### 简介

从ThreadLocal的源码可以看出，无论是set、get、还是remove，都是相对于当前线程操作的

```
Thread.currentThread()
```

所以ThreadLocal无法从父线程传向子线程，所以InheritableThreadLocal出现了，**它能够让父线程中ThreadLocal的值传给子线程。**

也就是从main所在的线程，传给thread1或thread2

### 使用

```
public class Demo1 {
   public static void main(String[] args) {
      ThreadLocal<String> stringThreadLocal = new ThreadLocal<>();
      InheritableThreadLocal<String> stringInheritable = new InheritableThreadLocal<>();

      // 主线程赋对上面两个变量进行赋值
      stringThreadLocal.set("this is threadLocal");
      stringInheritable.set("this is inheritableThreadLocal");

      // 创建线程
      Thread thread1 = new Thread(()->{
         // 获得ThreadLocal中存放的值
         System.out.println(stringThreadLocal.get());

         // 获得InheritableThreadLocal存放的值
         System.out.println(stringInheritable.get());
      });

      thread1.start();
   }
}
```

**运行结果**

```
null
this is inheritableThreadLocal
```

可以看出InheritableThreadLocal的值成功从主线程传入了子线程，而ThreadLocal则没有

### 原理

#### InheritableThreadLocal

```
public class InheritableThreadLocal<T> extends ThreadLocal<T> {
    // 传入父线程中的一个值，然后直接返回
    protected T childValue(T parentValue) {
        return parentValue;
    }

  	// 返回传入线程的inheritableThreadLocals
    // Thread中有一个inheritableThreadLocals变量
    // ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }

 	// 创建一个inheritableThreadLocals
    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
}
```

由如上代码可知，InheritableThreadLocal继承了ThreadLocal，并重写了三个方法。InheritableThreadLocal重写了**createMap方法**，那么现在当第一次调用set方法时，创建的是当前线程的inheritableThreadLocals变量的实例而不再是threadLocals。当调用**getMap方法**获取当前线程内部的map变量时，获取的是inheritableThreadLocals而不再是threadLocals

#### childValue(T parentValue)方法的调用

在主函数运行时，会调用Thread的默认构造函数（**创建主线程**，也就是父线程），所以我们先看看Thread的默认构造函数

```
public Thread() {
    init(null, null, "Thread-" + nextThreadNum(), 0);
}
```

```
private void init(ThreadGroup g, Runnable target, String name,
                  long stackSize, AccessControlContext acc,
                  boolean inheritThreadLocals) {
   	...
        
	// 获得当前线程的，在这里是主线程
    Thread parent = currentThread();
   
    ...
    
    // 如果父线程的inheritableThreadLocals存在
    // 我们在主线程中调用set和get时，会创建inheritableThreadLocals
    if (inheritThreadLocals && parent.inheritableThreadLocals != null)
        // 设置子线程的inheritableThreadLocals
        this.inheritableThreadLocals =
            ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
    
    /* Stash the specified stack size in case the VM cares */
    this.stackSize = stackSize;

    /* Set thread ID */
    tid = nextThreadID();
}
```

```
static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
    return new ThreadLocalMap(parentMap);
}
```

在createInheritedMap内部使用父线程的inheritableThreadLocals变量作为构造函数创建了一个新的ThreadLocalMap变量，然后赋值给了子线程的inheritableThreadLocals变量

```
private ThreadLocalMap(ThreadLocalMap parentMap) {
    Entry[] parentTable = parentMap.table;
    int len = parentTable.length;
    setThreshold(len);
    table = new Entry[len];

    for (int j = 0; j < len; j++) {
        Entry e = parentTable[j];
        if (e != null) {
            @SuppressWarnings("unchecked")
            ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
            if (key != null) {
                // 这里调用了 childValue 方法
                // 该方法会返回parent的值
                Object value = key.childValue(e.value);
                
                Entry c = new Entry(key, value);
                int h = key.threadLocalHashCode & (len - 1);
                while (table[h] != null)
                    h = nextIndex(h, len);
                table[h] = c;
                size++;
            }
        }
    }
}
```

在该构造函数内部把父线程的inheritableThreadLocals成员变量的值复制到新的ThreadLocalMap对象中

### 总结

InheritableThreadLocal类通过重写getMap和createMap，让本地变量保存到了具体线程的inheritableThreadLocals变量里面，那么线程在通过InheritableThreadLocal类实例的set或者get方法设置变量时，就会创建当前线程的inheritableThreadLocals变量。

**当父线程创建子线程时，构造函数会把父线程中inheritableThreadLocals变量里面的本地变量复制一份保存到子线程的inheritableThreadLocals变量里面。**



# 5. 共享模型之内存



## 5.1 Java 内存模型

JMM 即 Java Memory Model，它定义了主存、工作内存抽象概念，底层对应着 CPU 寄存器、缓存、硬件内存、CPU 指令优化等。
JMM 体现在以下几个方面

- 原子性 - 保证指令不会受到线程上下文切换的影响
- 可见性 - 保证指令不会受 cpu 缓存的影响
- 有序性 - 保证指令不会受 cpu 指令并行优化的影响



## 5.2 可见性



### 退不出的循环

先来看一个现象，main 线程对 run 变量的修改对于 t 线程不可见，导致了 t 线程无法停止：

```
static Boolean run = true;
	public static void main(String[] args) throws InterruptedException {
		new Thread(()->{
			while (run) {
				//如果run为真，则一直执行
			}
		}).start();

		Thread.sleep(1000);
		System.out.println("改变run的值为false");
		run = false;
	}
```

**为什么无法退出该循环**

- 初始状态， t 线程刚开始从**主内存**读取了 run 的值到**工作内存**。

{% asset_img 退不出的循环(1).png 退不出的循环(1) %}

- 因为 t 线程要频繁从主内存中读取 run 的值，JIT 编译器会将 run 的值**缓存至自己工作内存**中的高速缓存中， 减少对主存中 run 的访问，提高效率

{% asset_img 退不出的循环(2).png 退不出的循环(2) %}

- 1 秒之后，main 线程修改了 run 的值，并同步至主存，而 t 是从自己工作内存中的高速缓存中读取这个变量 的值，结果永远是**旧值**

{% asset_img 退不出的循环(3).png 退不出的循环(3) %}

### 解决方法
volatile（易变关键字）

它可以用来修饰成员变量和静态成员变量，他可以避免线程从自己的工作缓存中查找变量的值，必须到主存中获取它的值，线程操作 volatile 变量都是直接操作主存



### 可见性 vs 原子性

前面例子体现的实际就是**可见性**，它保证的是在多个线程之间，一个线程对**volatile变量**的修改对另一个线程可见， **不能**保证原子性，仅用在**一个写**线程，**多个读**线程的情况

- 注意 synchronized 语句块既可以保证代码块的**原子性**，也同时保证代码块内变量的**可见性**。

- 但缺点是 synchronized 是属于**重量级**操作，性能相对更低。

- 如果在前面示例的死循环中加入 System.out.println() 会发现即使不加 volatile 修饰符，线程 t 也能正确看到 对 run 变量的修改了，想一想为什么？

  - 因为使用了**synchronized**关键字

  ```
  public void println(String x) {
  		//使用了synchronized关键字
          synchronized (this) {
              print(x);
              newLine();
          }
      }
  ```



## 5.3 有序性

JVM 会在不影响正确性的前提下，可以调整语句的执行顺序，思考下面一段代码

```
static int i;
static int j;
// 在某个线程内执行如下赋值操作
i = ...;
j = ...;
```

可以看到，至于是先执行 i 还是 先执行 j ，对最终的结果不会产生影响。所以，上面代码真正执行时，既可以是

```
i = ...;
j = ...;
```

也可以是

```
j = ...;
i = ...;
```

这种特性称之为『指令重排』，多线程下『指令重排』会影响正确性。



### 指令重排序优化

- 事实上，现代处理器会设计为一个时钟周期完成一条执行时间长的 CPU 指令。为什么这么做呢？可以想到指令还可以再划分成一个个更小的阶段，例如，每条指令都可以分为： **取指令 - 指令译码 - 执行指令 - 内存访问 - 数据写回** 这5 个阶段
- 在不改变程序结果的前提下，这些指令的各个阶段可以通过**重排序**和**组合**来实现**指令级并行**
- 指令重排的前提是，重排指令**不能影响结果**



### 解决办法

**volatile** 修饰的变量，可以**禁用**指令重排



### volatile 原理

volatile的底层实现原理是**内存屏障**，Memory Barrier（Memory Fence）

- 对 volatile 变量的写指令后会加入写屏障
- 对 volatile 变量的读指令前会加入读屏障



#### 如何保证可见性

- 写屏障（sfence）保证在该屏障之前的，对共享变量的改动，都同步到主存当中

- 而读屏障（lfence）保证在该屏障之后，对共享变量的读取，加载的是主存中新数据



#### 如何保证有序性

- 写屏障会确保指令重排序时，不会将写屏障之前的代码排在写屏障之后

- 读屏障会确保指令重排序时，不会将读屏障之后的代码排在读屏障之前



**但是不能解决指令交错问题**

- 写屏障仅仅是保证之后的读能够读到新的结果，但不能保证读跑到它前面去
- 而有序性的保证也只是保证了**本线程内**相关代码不被重排序



#### double-checked locking 问题

```
public final class Singleton {
	private Singleton() { }
	private static Singleton INSTANCE = null;
	public static Singleton getInstance() {
		if(INSTANCE == null) { // t2
			// 首次访问会同步，而之后的使用没有 synchronized
			synchronized(Singleton.class) {
				if (INSTANCE == null) { // t1
					INSTANCE = new Singleton();
				}
			}
		}
	return INSTANCE;
	}
}
```

以上的实现特点是：

- 懒惰实例化
- 首次使用 getInstance() 才使用 synchronized 加锁，后续使用时无需加锁
- 有隐含的，但很关键的一点：第一个 if 使用了 INSTANCE 变量，是在同步块之外



但在多线程环境下，上面的代码是有问题的



发生指令重排，线程t1还未完全将构造方法构造完毕，此时线程t2拿到的是一个未初始化的单例。



对 INSTANCE 使用 volatile 修饰即可，可以禁用指令重排



#### double-checked locking 解决

```
public final class Singleton {
	private Singleton() { }
	private static volatile Singleton INSTANCE = null;
	public static Singleton getInstance() {
		// 实例没创建，才会进入内部的 synchronized代码块
		if (INSTANCE == null) {
			synchronized (Singleton.class) { // t2
			// 也许有其它线程已经创建实例，所以再判断一次
				if (INSTANCE == null) { // t1
					INSTANCE = new Singleton();
				}
			}
		}
	return INSTANCE;
	}
}
```

读写 volatile 变量时会加入内存屏障（Memory Barrier（Memory Fence）），保证下面两点：

- 可见性
  - 写屏障（sfence）保证在该屏障之前的 t1 对共享变量的改动，都同步到主存当中
  - 而读屏障（lfence）保证在该屏障之后 t2 对共享变量的读取，加载的是主存中最新数据
- 有序性
    - 写屏障会确保指令重排序时，不会将写屏障之前的代码排在写屏障之后
    - 读屏障会确保指令重排序时，不会将读屏障之后的代码排在读屏障之前
- 更底层是读写变量时使用 lock 指令来多核 CPU 之间的可见性与有序性



### happens-before

happens-before 规定了对共享变量的写操作对其它线程的读操作可见，它是可见性与有序性的一套规则总结，抛开以下 happens-before 规则，JMM 并不能保证一个线程对共享变量的写，对于其它线程对该共享变量的读可见

- 线程解锁 m 之前对变量的写，对于接下来对 m 加锁的其它线程对该变量的读可见

```
	static int x;
	
	static Object m = new Object();
	
	new Thread(()->{
	    synchronized(m) {
	        x = 10;
	    }
	},"t1").start();
	
	new Thread(()->{
	    synchronized(m) {
	        System.out.println(x);
	    }
	},"t2").start();

// 10

```

- 线程对 volatile 变量的写，对接下来其它线程对该变量的读可见

```
volatile static int x;
new Thread(()->{
	x = 10;
},"t1").start();
new Thread(()->{
	System.out.println(x);
},"t2").start();
```

- 线程 start 前对变量的写，对该线程开始后对该变量的读可见

```
static int x;
x = 10;
new Thread(()->{
System.out.println(x);
},"t2").start();
```

- 线程结束前对变量的写，对其它线程得知它结束后的读可见（比如其它线程调用 t1.isAlive() 或 t1.join()等待它结束）

```
static int x;
Thread t1 = new Thread(()->{
	x = 10;
},"t1");
t1.start();
t1.join();
System.out.println(x);
```

- 线程 t1 打断 t2（interrupt）前对变量的写，对于其他线程得知 t2 被打断后对变量的读可见（通过t2.interrupted 或 t2.isInterrupted）

```java
static int x;

public static void main(String[] args) {
    Thread t2 = new Thread(() -> {
        while (true) {
            if (Thread.currentThread().isInterrupted()) {
                System.out.println(x);
                break;
            }
        }
    }, "t2");
    t2.start();
    new Thread(() -> {
        sleep(1);
        x = 10;
        t2.interrupt();
    }, "t1").start();
    while (!t2.isInterrupted()) {
        Thread.yield();
    }
    System.out.println(x);
}
```

- 对变量默认值（0，false，null）的写，对其它线程对该变量的读可见
- 具有传递性，如果 x hb-> y 并且 y hb-> z 那么有 x hb-> z ，配合 volatile 的防指令重排

```Java
volatile static int x;
static int y;
new Thread(()->{
    y = 10;
    x = 20;
},"t1").start();
new Thread(()->{
    // x=20 对 t2 可见, 同时 y=10 也对 t2 可见
    System.out.println(x);
},"t2").start();
```

> 变量都是指成员变量或静态成员变量



# 6. 共享模型之无锁



## 6.2 CAS 与 volatile

`AtomicInteger` 内部并没有用锁来保护共享变量的线程安全。那么它是如何实现的呢？



其中的**关键是 compareAndSwap**（比较并设置值），它的**简称就是 CAS** （也有 Compare And Swap 的说法），它必须是**原子操作**。

{% asset_img cas操作.png cas操作 %}



### **工作流程**

- 当一个线程要去修改Account对象中的值时，先获取值pre（调用get方法），然后再将其设置为新的值next（调用cas方法）。在调用cas方法时，会将pre与Account中的余额进行比较。
  - 如果**两者相等**，就说明该值还未被其他线程修改，此时便可以进行修改操作。
  - 如果**两者不相等**，就不设置值，重新获取值pre（调用get方法），然后再将其设置为新的值next（调用cas方法），直到修改成功为止。

**注意**

- 其实 CAS 的底层是 **lock cmpxchg** 指令（X86 架构），在单核 CPU 和多核 CPU 下都能够保证【比较-交换】的**原子性**。
- 在多核状态下，某个核执行到带 lock 的指令时，CPU 会让总线锁住，当这个核把此指令执行完毕，再开启总线。这个过程中不会被线程的调度机制所打断，保证了多个线程对内存操作的准确性，是原子的。

### volatile

获取共享变量时，为了保证该变量的**可见性**，需要使用 **volatile** 修饰。
它可以用来修饰成员变量和静态成员变量，他可以避免线程从自己的工作缓存中查找变量的值，必须到**主存**中获取 它的值，线程操作 volatile 变量都是直接操作主存。即一个线程对 volatile 变量的修改，对另一个线程可见。

> **注意**
>
> volatile 仅仅保证了共享变量的可见性，让其它线程能够看到新值，但不能解决指令交错问题（不能保证原子性）

**CAS 必须借助 volatile** 才能读取到共享变量的新值来实现【比较并交换】的效果

### 效率问题

一般情况下，使用无锁比使用加锁的**效率更高。**

- 无锁情况下，即使重试失败，线程始终在高速运行，没有停歇，而 synchronized 会让线程在没有获得锁的时候，发生上下文切换，进入阻塞。
- 但无锁情况下，因为线程要保持运行，需要额外 CPU 的支持，虽然不会进入阻塞，但由于没有分到时间片，仍然会进入可运行状态，还是会导致上下文切换。



### CAS特点

结合 CAS 和 volatile 可以实现**无锁并发**，适用于**线程数少、多核 CPU** 的场景下。

- CAS 是基于**乐观锁**的思想：乐观的估计，不怕别的线程来修改共享变量，就算改了也没关系，我吃亏点再重试呗。
- synchronized 是基于悲观锁的思想：悲观的估计，得防着其它线程来修改共享变量，我上了锁你们都别想改，我改完了解开锁，你们才有机会。
- CAS 体现的是无锁并发、无阻塞并发
  - 因为没有使用 synchronized，所以线程不会陷入阻塞，这是效率提升的因素之一
  - 但如果竞争激烈，可以想到重试必然频繁发生，反而效率会受影响



## ABA 问题及解决



### ABA 问题

```Java
public class Demo3 {
    static AtomicReference<String> str = new AtomicReference<>("A");
    public static void main(String[] args) {
        new Thread(() -> {
            String pre = str.get();
            System.out.println("change");
            try {
                other();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            //把str中的A改为C
            System.out.println("change A->C " + str.compareAndSet(pre, "C"));
        }).start();
    }

    static void other() throws InterruptedException {
        new Thread(()-> {
            System.out.println("change A->B " + str.compareAndSet("A", "B"));
        }).start();
        Thread.sleep(500);
        new Thread(()-> {
            System.out.println("change B->A " + str.compareAndSet("B", "A"));
        }).start();
    }
}
```

主线程仅能判断出共享变量的值与初值 A **是否相同**，不能感知到这种从 A 改为 B 又 改回 A 的情况，如果主线程希望：
只要有其它线程【**动过了**】共享变量，那么自己的 **cas 就算失败**，这时，仅比较值是不够的，需要再加一个**版本号**

### AtomicStampedReference

```
public class Demo3 {
	//指定版本号
	static AtomicStampedReference<String> str = new AtomicStampedReference<>("A", 0);
	public static void main(String[] args) {
		new Thread(() -> {
			String pre = str.getReference();
			//获得版本号
			int stamp = str.getStamp();
			System.out.println("change");
			try {
				other();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			try {
				Thread.sleep(1000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			//把str中的A改为C,并比对版本号，如果版本号相同，就执行替换，并让版本号+1
			System.out.println("change A->C stamp " + stamp + str.compareAndSet(pre, "C", stamp, stamp+1));
		}).start();
	}

	static void other() throws InterruptedException {
		new Thread(()-> {
			int stamp = str.getStamp();
			System.out.println("change A->B stamp " + stamp + str.compareAndSet("A", "B", stamp, stamp+1));
		}).start();
		Thread.sleep(500);
		new Thread(()-> {
			int stamp = str.getStamp();
			System.out.println("change B->A stamp " + stamp +  str.compareAndSet("B", "A", stamp, stamp+1));
		}).start();
	}
}
```

AtomicStampedReference 可以给原子引用加上**版本号**，追踪原子引用整个的变化过程，如： `A -> B -> A ->C` ，通过AtomicStampedReference，我们可以知道，引用变量中途被更改了几次。

### AtomicMarkableReference

但是有时候，并不关心引用变量更改了几次，只是单纯的关心**是否更改过**，所以就有了AtomicMarkableReference

```
public class Demo4 {
	//指定版本号
	static AtomicMarkableReference<String> str = new AtomicMarkableReference<>("A", true);
	public static void main(String[] args) {
		new Thread(() -> {
			String pre = str.getReference();
			System.out.println("change");
			try {
				other();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			try {
				Thread.sleep(1000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			//把str中的A改为C,并比对版本号，如果版本号相同，就执行替换，并让版本号+1
			System.out.println("change A->C mark " +  str.compareAndSet(pre, "C", true, false));
		}).start();
	}

	static void other() throws InterruptedException {
		new Thread(() -> {
			System.out.println("change A->A mark " + str.compareAndSet("A", "A", true, false));
		}).start();
	}
}
```

### 两者的区别

- **AtomicStampedReference** 需要我们传入**整型变量**作为**版本号**，来判定是否被更改过
- **AtomicMarkableReference**需要我们传入**布尔变量**作为**标记**，来判断是否被更改过



## 6.7 原子累加器



### 原理之伪共享

其中 Cell 即为累加单元

```Java
// 防止缓存行伪共享
@sun.misc.Contended
    static final class Cell {
        volatile long value;
        Cell(long x) { value = x; }
        // 最重要的方法, 用来 cas 方式进行累加, prev 表示旧值, next 表示新值
        final boolean cas(long prev, long next) {
            return UNSAFE.compareAndSwapLong(this, valueOffset, prev, next);
        }
        // 省略不重要代码
    }
```

因为 CPU 与 内存的速度差异很大，需要靠预读数据至**缓存**来提升效率。
而缓存以**缓存行**为单位，每个缓存行对应着一块内存，一般是 **64 byte**（8 个 long）
缓存的加入会造成数据副本的产生，即同一份数据会缓存在不同核心的缓存行中
CPU 要保证数据的**一致性**，如果某个 CPU 核心**更改**了数据，其它 CPU 核心对应的整个缓存行必须**失效**

{% asset_img 缓存行失效.png 缓存行失效 %}

因为 Cell 是数组形式，在内存中是连续存储的，一个 Cell 为 24 字节（16 字节的对象头和 8 字节的 value），因 此缓存行可以存下 2 个的 Cell 对象。这样问题来了：

- Core-0 要修改 Cell[0]
- Core-1 要修改 Cell[1]

无论谁修改成功，都会导致对方 Core 的缓存行失效，

比如 Core-0 中 Cell[0]=6000, Cell[1]=8000 要累加 Cell[0]=6001, Cell[1]=8000 ，这时会让 Core-1 的缓存行失效

@sun.misc.Contended 用来解决这个问题，它的原理是在使用此注解的对象或字段的**前后各增加 128 字节大小的 padding**（空白），从而让 CPU 将对象预读至缓存时**占用不同的缓存行**，这样，不会造成对方缓存行的失效

{% asset_img 解决缓存行失效.png 解决缓存行失效 %}



# 7. 共享模型之不可变

## 7.1 不可变

如果一个对象在**不能够修**改其内部状态（属性），那么它就是线程安全的，因为不存在并发修改。

## 7.2 不可变设计

### String类中不可变的体现

```Java
public final class String implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];

    /** Cache the hash code for the string */
    private int hash; // Default to 0

    //....
}
}
```

### final 的使用

发现该类、类中所有属性都是 final 的

- 属性用 final 修饰保证了该属性是只读的，不能修改
- 类用 final 修饰保证了该类中的方法不能被覆盖，防止子类无意间破坏不可变性



### 保护性拷贝

构造新字符串对象时，会生成新的 char[] value，对内容进行复制 。这种通过创建副本对象来避免共享的手段称之为【保护性拷贝（defensive copy）】



### 模式之享元

#### 1. 简介

定义 英文名称：Flyweight pattern. 当需要重用数量有限的同一类对象时



#### 2.设计连接池

```
public class Test3 {
    public static void main(String[] args) {
        Pool pool = new Pool(2);
        for (int i = 0; i < 5; i++) {
            new Thread(() -> {
                Connection conn = pool.borrow();
                try {
                    Thread.sleep(new Random().nextInt(1000));
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                pool.free(conn);
            }).start();
        }
    }
}

@Slf4j(topic = "c.Pool")
class Pool {
    // 1. 连接池大小
    private final int poolSize;

    // 2. 连接对象数组
    private Connection[] connections;

    // 3. 连接状态数组 0 表示空闲， 1 表示繁忙
    private AtomicIntegerArray states;

    // 4. 构造方法初始化
    public Pool(int poolSize) {
        this.poolSize = poolSize;
        this.connections = new Connection[poolSize];
        this.states = new AtomicIntegerArray(new int[poolSize]);
        for (int i = 0; i < poolSize; i++) {
            connections[i] = new MockConnection("连接" + (i+1));
        }
    }

    // 5. 借连接
    public Connection borrow() {
        while(true) {
            for (int i = 0; i < poolSize; i++) {
                // 获取空闲连接
                if(states.get(i) == 0) {
                    if (states.compareAndSet(i, 0, 1)) {
                        log.debug("borrow {}", connections[i]);
                        return connections[i];
                    }
                }
            }
            // 如果没有空闲连接，当前线程进入等待
            synchronized (this) {
                try {
                    log.debug("wait...");
                    this.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    // 6. 归还连接
    public void free(Connection conn) {
        for (int i = 0; i < poolSize; i++) {
            if (connections[i] == conn) {
                states.set(i, 0);
                synchronized (this) {
                    log.debug("free {}", conn);
                    this.notifyAll();
                }
                break;
            }
        }
    }
}

class MockConnection implements Connection {

    private String name;

    public MockConnection(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "MockConnection{" +
                "name='" + name + '\'' +
                '}';
    }

    @Override
    public Statement createStatement() throws SQLException {
        return null;
    }

    @Override
    public PreparedStatement prepareStatement(String sql) throws SQLException {
        return null;
    }

    @Override
    public CallableStatement prepareCall(String sql) throws SQLException {
        return null;
    }

    @Override
    public String nativeSQL(String sql) throws SQLException {
        return null;
    }

    @Override
    public void setAutoCommit(boolean autoCommit) throws SQLException {

    }

    @Override
    public boolean getAutoCommit() throws SQLException {
        return false;
    }

    @Override
    public void commit() throws SQLException {

    }

    @Override
    public void rollback() throws SQLException {

    }

    @Override
    public void close() throws SQLException {

    }

    @Override
    public boolean isClosed() throws SQLException {
        return false;
    }

    @Override
    public DatabaseMetaData getMetaData() throws SQLException {
        return null;
    }

    @Override
    public void setReadOnly(boolean readOnly) throws SQLException {

    }

    @Override
    public boolean isReadOnly() throws SQLException {
        return false;
    }

    @Override
    public void setCatalog(String catalog) throws SQLException {

    }

    @Override
    public String getCatalog() throws SQLException {
        return null;
    }

    @Override
    public void setTransactionIsolation(int level) throws SQLException {

    }

    @Override
    public int getTransactionIsolation() throws SQLException {
        return 0;
    }

    @Override
    public SQLWarning getWarnings() throws SQLException {
        return null;
    }

    @Override
    public void clearWarnings() throws SQLException {

    }

    @Override
    public Statement createStatement(int resultSetType, int resultSetConcurrency) throws SQLException {
        return null;
    }

    @Override
    public PreparedStatement prepareStatement(String sql, int resultSetType, int resultSetConcurrency) throws SQLException {
        return null;
    }

    @Override
    public CallableStatement prepareCall(String sql, int resultSetType, int resultSetConcurrency) throws SQLException {
        return null;
    }

    @Override
    public Map<String, Class<?>> getTypeMap() throws SQLException {
        return null;
    }

    @Override
    public void setTypeMap(Map<String, Class<?>> map) throws SQLException {

    }

    @Override
    public void setHoldability(int holdability) throws SQLException {

    }

    @Override
    public int getHoldability() throws SQLException {
        return 0;
    }

    @Override
    public Savepoint setSavepoint() throws SQLException {
        return null;
    }

    @Override
    public Savepoint setSavepoint(String name) throws SQLException {
        return null;
    }

    @Override
    public void rollback(Savepoint savepoint) throws SQLException {

    }

    @Override
    public void releaseSavepoint(Savepoint savepoint) throws SQLException {

    }

    @Override
    public Statement createStatement(int resultSetType, int resultSetConcurrency, int resultSetHoldability) throws SQLException {
        return null;
    }

    @Override
    public PreparedStatement prepareStatement(String sql, int resultSetType, int resultSetConcurrency, int resultSetHoldability) throws SQLException {
        return null;
    }

    @Override
    public CallableStatement prepareCall(String sql, int resultSetType, int resultSetConcurrency, int resultSetHoldability) throws SQLException {
        return null;
    }

    @Override
    public PreparedStatement prepareStatement(String sql, int autoGeneratedKeys) throws SQLException {
        return null;
    }

    @Override
    public PreparedStatement prepareStatement(String sql, int[] columnIndexes) throws SQLException {
        return null;
    }

    @Override
    public PreparedStatement prepareStatement(String sql, String[] columnNames) throws SQLException {
        return null;
    }

    @Override
    public Clob createClob() throws SQLException {
        return null;
    }

    @Override
    public Blob createBlob() throws SQLException {
        return null;
    }

    @Override
    public NClob createNClob() throws SQLException {
        return null;
    }

    @Override
    public SQLXML createSQLXML() throws SQLException {
        return null;
    }

    @Override
    public boolean isValid(int timeout) throws SQLException {
        return false;
    }

    @Override
    public void setClientInfo(String name, String value) throws SQLClientInfoException {

    }

    @Override
    public void setClientInfo(Properties properties) throws SQLClientInfoException {

    }

    @Override
    public String getClientInfo(String name) throws SQLException {
        return null;
    }

    @Override
    public Properties getClientInfo() throws SQLException {
        return null;
    }

    @Override
    public Array createArrayOf(String typeName, Object[] elements) throws SQLException {
        return null;
    }

    @Override
    public Struct createStruct(String typeName, Object[] attributes) throws SQLException {
        return null;
    }

    @Override
    public void setSchema(String schema) throws SQLException {

    }

    @Override
    public String getSchema() throws SQLException {
        return null;
    }

    @Override
    public void abort(Executor executor) throws SQLException {

    }

    @Override
    public void setNetworkTimeout(Executor executor, int milliseconds) throws SQLException {

    }

    @Override
    public int getNetworkTimeout() throws SQLException {
        return 0;
    }

    @Override
    public <T> T unwrap(Class<T> iface) throws SQLException {
        return null;
    }

    @Override
    public boolean isWrapperFor(Class<?> iface) throws SQLException {
        return false;
    }
}
```





# 8. 共享模型之工具

## 8.1 线程池

### 1. 自定义线程池

{% asset_img 自定义线程池.png 自定义线程池 %}

```
@Slf4j(topic = "c.TestPool")
public class TestPool {
    public static void main(String[] args) {
        ThreadPool threadPool =new ThreadPool(1, 1,
                1000, TimeUnit.MILLISECONDS, (queue, task)->{
            //1. 死等
            queue.put(task);
            //2. 带超时等待
//            queue.offer(task, 1500, TimeUnit.MILLISECONDS);
            //3. 让调用者放弃任务执行
//            log.debug("放弃{}", task);
            //4. 让调用者抛出异常
//            throw new RuntimeException("执行任务失败 "+ task);
            //5. 让调用者自己执行任务
//            task.run();
        });
        for (int i = 0; i < 4; i++) {
            int j = i;
            threadPool.execute(() -> {
                try {
                    Thread.sleep(1000L);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                log.debug("{}", j);
            });
        }
    }

}

//拒绝策略
@FunctionalInterface
interface RejectPolicy<T> {
    void reject(BlockingQueue<T> queue, T task);
}

//线程池
@Slf4j(topic = "c.ThreadPool")
class ThreadPool {
    //线程数
    private int coreSize;

    //任务队列
    private BlockingQueue<Runnable> taskQueue;

    //线程集合
    private HashSet<Worker> workers = new HashSet<>();

    //获取任务时的超时时间
    private long timeout;

    private TimeUnit timeUnit;

    private RejectPolicy<Runnable> rejectPolicy;

    //执行任务
    public void execute(Runnable task) {
        //当任务数没有超过coreSize时，直接交给worker对象执行
        //当任务数超过coreSize时，加入任务队列暂存
        synchronized (workers) {
            if (workers.size() < coreSize) {
                Worker worker = new Worker(task);
                log.debug("新增 worker{}, {}", worker, task);
                workers.add(worker);
                worker.start();
            } else {
                //1. 死等
                //2. 带超时等待
                //3. 调用者放弃执行任务
                //4. 调用者抛出异常
                //5. 让调用者自己执行任务
                
                //使用策略模式，用户传入策略避免策略写死
                taskQueue.tryPut(rejectPolicy, task);
            }
        }
    }

    public ThreadPool(int coreSize, int queueSize, long timeout, TimeUnit timeUnit, RejectPolicy<Runnable> rejectPolicy) {
        this.coreSize = coreSize;
        this.taskQueue = new BlockingQueue<>(queueSize);
        this.timeout = timeout;
        this.timeUnit = timeUnit;
        this.rejectPolicy = rejectPolicy;
    }

    //自定义线程
    class Worker extends Thread {
        private Runnable task;

        public Worker(Runnable task) {
            this.task = task;
        }

        @Override
        public void run() {
            //执行任务
            //1. 当task不为空，执行task
            //2. 当task为空，从任务队列中获取任务并执行
            while (task != null || (task = taskQueue.poll(timeout, timeUnit)) != null) {
                try {
                    log.debug("正在执行...{}", task);
                    task.run();     
                } catch (Exception e) {
                    e.printStackTrace();
                } finally {
                    task = null;
                }
            }
            synchronized (workers) {
                log.debug("worker被移除{}", this);
                workers.remove(this);
            }
        }
    }
}

//任务队列
@Slf4j(topic = "c.BlockingQueue")
class BlockingQueue<T> {

    //任务数
    private int capacity;

    //任务队列
    private Deque<T> queue = new ArrayDeque<>();

    //锁
    private ReentrantLock lock = new ReentrantLock();

    //生产者条件变量
    private Condition producer = lock.newCondition();

    //消费者条件变量
    private Condition consumer = lock.newCondition();

    public BlockingQueue(int capacity) {
        this.capacity = capacity;
    }

    //带超时阻塞获取
    public T poll(long timeout, TimeUnit unit) {
        lock.lock();
        try {
            long nanos = unit.toNanos(timeout);
            while (queue.isEmpty()) {
                try {
                    if (nanos <= 0) {
                        return null;
                    }
                    nanos = consumer.awaitNanos(nanos);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            T task = queue.removeFirst();
            producer.signal();
            return task;
        } finally {
            lock.unlock();
        }
    }

    //阻塞获取
    public T take() {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                try {
                    consumer.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            T task = queue.removeFirst();
            producer.signal();
            return task;
        } finally {
            lock.unlock();
        }
    }

    //阻塞添加
    public void put(T task) {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                try {
                    log.debug("等待加入任务队列{}...", task);
                    producer.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            log.debug("加入任务队列{}", task);
            queue.addLast(task);
            consumer.signal();
        } finally {
            lock.unlock();
        }
    }

    //带超时时间阻塞添加
    public boolean offer(T task, long timeout, TimeUnit unit) {
        lock.lock();
        try {
            long nanos = unit.toNanos(timeout);
            while (queue.size() == capacity) {
                if (nanos <= 0) {
                    return false;
                }
                try {
                    log.debug("等待加入任务队列{}...", task);
                    nanos = producer.awaitNanos(nanos);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            log.debug("加入任务队列{}", task);
            queue.addLast(task);
            consumer.signal();
            return true;
        } finally {
            lock.unlock();
        }
    }

    public int size() {
        lock.lock();
        try {
            return queue.size();
        } finally {
            lock.unlock();
        }
    }

    //带拒绝策略的添加
    public void tryPut(RejectPolicy<T> rejectPolicy, T task) {
        lock.lock();
        try {
            if (queue.size() == capacity) {
                rejectPolicy.reject(this, task);
            } else {
                log.debug("加入任务队列{}", task);
                queue.addLast(task);
                consumer.signal();
            }
        } finally {
            lock.unlock();
        }
    }
}

```

- 阻塞队列BlockingQueue用于暂存来不及被线程执行的任务
  - 也可以说是平衡生产者和消费者执行速度上的差异
  - 里面的获取任务和放入任务用到了**生产者消费者模式**
- 线程池中对线程Thread进行了再次的封装，封装为了Worker
  - 在调用任务的run方法时，线程会去执行该任务，执行完毕后还会**到阻塞队列中获取新任务来执行**
- 线程池中执行任务的主要方法为execute方法
  - 执行时要判断正在执行的线程数是否大于了线程池容量



### 2. ThreadPoolExecutor

{% asset_img ThreadPoolExecutor继承关系.png ThreadPoolExecutor继承关系 %}



## 8.2 J.U.C

### 8.2.1 AQS 原理

#### 1. 概述

全称是 AbstractQueuedSynchronizer，是阻塞式锁和相关的同步器工具的框架

特点：

- 用 state 属性来表示资源的状态（分独占模式和共享模式），子类需要定义如何维护这个状态，控制如何获取锁和释放
   - getState - 获取 state 状态
   - setState - 设置 state 状态
   - compareAndSetState - cas 机制设置 state 状态
   - 独占模式是只有一个线程能够访问资源，而共享模式可以允许多个线程访问资源
- 提供了基于 FIFO 的等待队列，类似于 Monitor 的 EntryList
- 条件变量来实现等待、唤醒机制，支持多个条件变量，类似于 Monitor 的 WaitSet



子类主要实现这样一些方法（默认抛出 UnsupportedOperationException）

- tryAcquire
- tryRelease
- tryAcquireShared
- tryReleaseShared
- isHeldExclusively



###  8.2.2 ReentrantLock 原理

{% asset_img ReentrantLock继承关系.png ReentrantLock继承关系 %}



可以看到ReentrantLock提供了两个同步器，实现公平锁和非公平锁，默认是非公平锁！



###  8.2.3 读写锁

####  1. ReentrantReadWriteLock

当读操作远远高于写操作时，这时候使用读写锁让读-读可以并发，提高性能。读-写，写-写都是相互互斥的！

注意事项

1. 读锁不支持条件变量
2. 重入时升级不支持：即持有读锁的情况下去获取写锁，会导致获取写锁永久等待

###  8.2.4 Semaphore

####  基本使用

信号量，用来限制能同时访问共享资源的线程上限。



###  8.2.5 CountdownLatch

CountDownLatch允许 count 个线程阻塞在一个地方，直至所有线程的任务都执行完毕。在 Java 并发中，countdownlatch 的概念是一个常见的面试题，所以一定要确保你很好的理解了它。



CountDownLatch是共享锁的一种实现,它默认构造 AQS 的 state 值为 count。当线程使用countDown方法时,其实使用了`tryReleaseShared`方法以CAS的操作来减少state,直至state为0就代表所有的线程都调用了countDown方法。当调用await方法的时候，如果state不为0，就代表仍然有线程没有调用countDown方法，那么就把已经调用过countDown的线程都放入阻塞队列Park,并自旋CAS判断state == 0，直至最后一个线程调用了countDown，使得state == 0，于是阻塞的线程便判断成功，全部往下执行。



用来进行线程同步协作，等待所有线程完成倒计时。 其中构造参数用来初始化等待计数值，await() 用来等待计数归零。



###  8.2.6 CyclicBarrier

yclicBarrier循环栅栏，用来进行线程协作，等待线程满足某个计数。构造时设置『计数个数』，每个线程执行到某个需要“同步”的时刻调用 await() 方法进行等待，当等待的线程数满足『计数个数』时，继续执行。跟CountdownLatch一样，但这个可以重用

