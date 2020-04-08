---
title: java多线程基础-3
date: 2019-08-03 11:51:45
tags:
- Java
- Concurrent
categories:
- Java
- Concurrent
---
# Java多线程基础

### ReentrantLock

#### 概述

> ​	`Reentrantlock`是一个可重入且独占式的锁，它具有与使用`synchronized`监视器锁相同的基本行为和语义，但与`synchronized`关键字相比，它更灵活、更强大，增加了轮询、超时、中断等高级功能。
>
> ​	`ReentrantLock`，顾名思义，它是支持可重入锁的锁，是一种递归无阻塞的同步机制。除此之外，该锁还支持获取锁时的公平和非公平选择。

------

#### 使用方法

```java
public class Reentrantlock2 {
	Lock lock = new ReentrantLock();

	void m1() {
		try {
			lock.lock();
			for (int i = 0; i < 10; i++) {
				TimeUnit.SECONDS.sleep(1);
				System.out.println(Thread.currentThread().getName()+" i == "+i);
			}
		} catch (InterruptedException e) {
			e.printStackTrace();
		}finally {
			lock.unlock();
		}
	}
	synchronized void m2() {
		lock.lock();
		System.out.println(Thread.currentThread().getName()+"m2....");
		lock.unlock();
	}
	public static void main(String[] args) {
		Reentrantlock2 r1 = new Reentrantlock2();
		new Thread(() -> r1.m1(), "m1").start();
		try {
			TimeUnit.SECONDS.sleep(1);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		new Thread(r1::m2, "m2").start();
	}
}
```

需要注意的是，`ReentrantLock`必须要手动释放锁。使用`synchronized`锁定的话如果遇到异常，`jvm`会自动释放锁，但`ReentrantLock`要手动释放 ，因此经常在finally中进行锁的释放。

------

`trylock()`方法，进行尝试锁定，不管锁定与否，方法都将继续进行，可以通过`trylock()`的返回值判断。

```java
public class Reentrantlock3 {
	Lock lock = new ReentrantLock();

	void m1() {
		try {
			lock.lock();
			for (int i = 0; i < 10; i++) {
				TimeUnit.SECONDS.sleep(1);
				System.out.println(i);
			}
		} catch (InterruptedException e) {
			e.printStackTrace();
		}finally {
			lock.unlock();
		}
	}

	synchronized void m2() {
		boolean locked = false;
		try {
			locked = lock.tryLock(5, TimeUnit.SECONDS);
			System.out.println("m2...."+locked);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}finally {
			if(locked) lock.unlock();
		}
	}

	public static void main(String[] args) {
		Reentrantlock3 r1 = new Reentrantlock3();
		new Thread(() -> r1.m1(), "m1").start();
		try {
			TimeUnit.SECONDS.sleep(1);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		new Thread(r1::m2, "m2").start();
	}
}
```

------

`lockInterruptibly()`方法使线程对`interrupt()`方法做出响应，在一个线程等待锁的过程中，可以被打断。

```java
public class Reentrantlock4 {
	
	public static void main(String[] args) {
		Lock lock = new ReentrantLock();
		Thread t1 = new Thread(()->{
			try {
				lock.lockInterruptibly();
				System.out.println("t1 start");
				TimeUnit.SECONDS.sleep(Integer.MAX_VALUE);
				System.out.println("t1 end");
			} catch (InterruptedException e) {
				System.out.println("t1 Interrupted");
			}finally {
				lock.unlock();
			}
		}) ;
		t1.start();
		Thread t2 = new Thread(()->{
			boolean locked = lock.tryLock();
			try {
				//lock.lock();
				lock.lockInterruptibly();//可以对interrupt方法做出响应
				System.out.println("t2 start");
				TimeUnit.SECONDS.sleep(5);
				System.out.println("t2 end");
			} catch (InterruptedException e) {
				System.out.println("t2 Interrupted");
			}finally {
				if(locked)	 lock.unlock();
			}
		});
		t2.start();
		try {
			TimeUnit.SECONDS.sleep(1);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		t2.interrupt();//打断线程等待
		try {
			TimeUnit.SECONDS.sleep(1);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		t1.interrupt();
	}
}
```



------



`Reentrantlock`还可以指定为公平锁，公平性与否是针对锁获取顺序而言的，如果一个锁是公平的，那么锁的获取顺序就应该符合FIFO原则。

`ReentrantLock()`

```java
   /**
     * Creates an instance of {@code ReentrantLock}.
     * This is equivalent to using {@code ReentrantLock(false)}.
     */
    public ReentrantLock() {
        sync = new NonfairSync();
    }
	/**
     * Creates an instance of {@code ReentrantLock} with the
     * given fairness policy.
     *
     * @param fair {@code true} if this lock should use a fair ordering policy
     */
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

```java
public class Reentrantlock5 extends Thread {
	private  ReentrantLock lock = new ReentrantLock(true);//参数为true 公平锁
	public void run() {
		for(int i = 0; i < 100; i++) {
			lock.lock();
			try {				System.out.println(Thread.currentThread().getName()+"获得锁");
			}finally {
				lock.unlock();
			}
		}
	}
	public static void main(String[] args) {
		Reentrantlock5 r5 = new Reentrantlock5();
		Thread t1 = new Thread(r5);
		Thread t2 = new Thread(r5);
		t1.start();
		t2.start();
	}
}
```

声明为公平锁后，公平锁根据等待时间，时间长的优先得到锁 ，t1、t2交替打印。