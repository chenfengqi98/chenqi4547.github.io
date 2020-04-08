---
title: java多线程基础-2
date: 2019-08-02 10:48:57
tags:
- Java
- Concurrent
categories:
- Java
- Concurrent
---
# Java多线程基础

### volatile关键字

#### 作用

> volatile关键字，使一个变量在多个线程中可见。

​	假设A，B线程都用到了一个变量，在Java中，默认是A线程从主内存中保留一份copy到自己的线程内存中，如果B线程修改了变量，A线程未必知道。使用volatile关键字，当变量值修改了之后，会通知其他线程的缓冲区的值过期。

```java
public class T {
	/*volatile*/ boolean running = true;//对比一下有无volatile的情况下，整个程序运行结果的区别
	 void m() {
		System.out.println(Thread.currentThread().getName()+"m Start");
		while(running) {
			
		}
		System.out.println("m end");
	}
	public static void main(String[] args) {
		T t = new T();
		new Thread(()->t.m(),"t1").start();
		try {
			TimeUnit.SECONDS.sleep(1);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		//使用volatile关键字，改了值之后，会通知其他线程缓冲区值过期
		t.running = false;
	}
}
```

在`running` 声明的时候如果不加`volatile` 关键字，线程t1启动后会进入死循环，尽管后面修改了`running`的值，加了`volatile` 关键字，就可以停止循环。

#### 性质

> volatile具有可见性，有序性，不具备原子性。

```java
public class T {
	/*volatile*/ int count = 0;
	/*synchronized*/ void m() {
		for(int i = 0; i < 10000; i++) {
			count++;
		}
	}
	public static void main(String[] args) {
		T t = new T();
		List<Thread> threads = new ArrayList<Thread>();
		for(int i = 0; i < 10; i++) {
			threads.add(new Thread(t::m,"thread"+i));
		}
		threads.forEach((o)->o.start());//启动线程
		threads.forEach((o)->{//求总和
			try {
				o.join();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		});
		//volatile count到不了10W
		System.out.println(t.count);
	}
}
```

在上面代码中，启动了十个线程，每个线程使`count`自增10000次。

- 不加`volatile`。

- 加`volatile`。

- 加`synchronized`。

  然后启动十个线程，可以看到结果，不加`volatile`的`count`值小于加`volatiel`的`count`值，但是都到不了10W，如果加了`synchronized`，`count`值等于10W。造成上面结果的原因是`count++`不是一个原子操作。

下面使用AtomicInteger类演示。

```java
public class T {
	AtomicInteger count = new AtomicInteger(0);//原子性
	void m() {
		for(int i = 0; i < 10000; i++) {
			count.incrementAndGet();//原子++
		}
	}
	public static void main(String[] args) {
		T t = new T();
		List<Thread> threads = new ArrayList<Thread>();
		for(int i = 0; i < 10; i++) {
			threads.add(new Thread(t::m,"thread"+i));
		}
		threads.forEach((o)->o.start());
		threads.forEach((o)->{
			try {
				o.join();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		});
		System.out.println(t.count);
	}
}
```

运行代码后可以看到`count`的值是10w，使用`AtomicInteger`的`incrementAndGet()`方法，就是一个原子操作，不可分。在没有加synchronized的情况下，也可以得到正确的`count`结果。

#### synchronized和volatile对比

- volatile

  - 可见性

    某线程对 volatile 变量的修改，对其他线程都是可见的。即获取 volatile 变量的值都是最新的。

  - 禁止指令重排

- synchronized

  - 可见性

  - 原子性

    

  

  

------



#### 实践

> 题目：
>
> 实现一个容器，有add 和 size方法 
>
> - 写两个线程，线程1 添加10个元素到容器中，线程2监视元素个数， 当size=5 线程2结束

------

方法1：`wait()`和`notify()`的使用

```java
public class MyContainer3 {
	volatile List lists = new ArrayList();
	public void add(Object o) {
		lists.add(o);
	}
	public int size() {
		return lists.size();
	}
	public static void main(String[] args) {
		MyContainer3 c = new MyContainer3();
		final Object lock = new Object();
		new Thread(()->{
			synchronized(lock) {
				System.out.println("t2 START");
				if(c.size() != 5) {
					try {
						lock.wait();//等待
					} catch (InterruptedException e) {
						e.printStackTrace();
					}
				}
				System.out.println(Thread.currentThread().getName()+"end");
				//通知t1继续执行
				lock.notify();
			}
		},"t2").start();
		try {
			TimeUnit.SECONDS.sleep(1);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		new Thread(()->{
			synchronized (lock) {
				System.out.println("t1 START");
				for(int i = 0; i< 10; i++) {
					c.add(new Object());
					System.out.println("add" + i);
					if(c.size() == 5) {
						lock.notify();//唤醒t2
						
						try {
							lock.wait();//释放锁
						} catch (InterruptedException e) {
							e.printStackTrace();
						}
						
					}
					try {
						TimeUnit.SECONDS.sleep(1);
					} catch (InterruptedException e) {
						e.printStackTrace();
					}
				}
			}
		},"t1").start();
	}
}
```

`t2`线程先启动，监视`size`的值，当`size`不等于5的时候，调用`wait()`等待，然后`t1`线程获得锁，开始添加元素，当`size`等于5，调用`notify()`方法通知t2，自己进入等待。然后线程`t2`执行完毕后通知`t1`，让`t1`继续执行。整个过程虽然满足了要求，但是通信过程繁琐。

------

方法2：使用门闩`CountDownLatch`

​	`CountDownLatch`是一个同步辅助类，在jdk5中引入，它允许一个或多个线程等待其他线程操作完成之后才执行。

​	` 	CountDownLatch`是通过计数器的方式来实现，计数器的初始值为线程的数量。每当一个线程完成了自己的任务之后，就会对计数器减1，当计数器的值为0时，表示所有线程完成了任务，此时等待在闭锁上的线程才继续执行，从而达到等待其他线程完成任务之后才继续执行的目的。

```java
public class MyContainer4 {
	volatile List lists = new ArrayList();
	public void add(Object o) {
		lists.add(o);
	}
	public int size() {
		return lists.size();
	}
	public static void main(String[] args) {
		MyContainer4 c = new MyContainer4();
		CountDownLatch latch = new CountDownLatch(1);//设置门闩，值为1
		new Thread(()->{
				System.out.println("t2 START");
				if(c.size() != 5) {
					try {
						latch.await();//门闩值为1 等待
					} catch (InterruptedException e) {
						e.printStackTrace();
					}
				}
				System.out.println(Thread.currentThread().getName()+"end");
		},"t2").start();
		try {
			TimeUnit.SECONDS.sleep(1);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		new Thread(()->{
				System.out.println("t1 START");
				for(int i = 0; i< 10; i++) {
					c.add(new Object());
					System.out.println("add" + i);
					if(c.size() == 5) {
						latch.countDown();//1-> 0
					}
					try {
						TimeUnit.SECONDS.sleep(1);
					} catch (InterruptedException e) {
						e.printStackTrace();
					}
				}
		},"t1").start();
	}
}
```

