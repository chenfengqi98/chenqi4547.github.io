---
title: java多线程基础-1
date: 2019-08-01 11:36:41
tags:
- Java
- Concurrent
categories:
- Java
- Concurrent
---
## Java多线程基础

### synchronized关键字

#### 作用

> synchronized关键字 对某个对象加锁，一个synchronized代码块是一个原子操作。

#### 使用方法

1. 对Object o上锁

```java
public class T {
	private int count = 10;
	private Object o = new Object();
	public void m() {
		synchronized (o) {//任何线程要执行下面的代码，都必须拿到对象o的锁
			count--;
			System.out.println(Thread.currentThread().getName()+"count = "+count);
		}
	}
}
```

2. 对this上锁

   ```java
   public class T {
   	private int count = 10;
   	public void m() {
   		synchronized (this) {//任何线程要执行下面的代码，都必须拿到this的锁
   			count--;
   			System.out.println(Thread.currentThread().getName()+"count = "+count);
   		}
   	}
   }
   ```

3. 方法上加synchronized

   ```java
   public class T {
   	private int count = 10;
   	public synchronized void m() {//等同于在方法的代码执行时要synchronized(this)
   		//锁定的是当前对象  不是代码块
   			count--;
   			System.out.println(Thread.currentThread().getName()+"count = "+count);
   	}
   }
   ```

4. 静态方法

   ```java
   public class T {
   	private static int count = 10;
   	public synchronized static void m() {//等同于在方法的代码执行时要synchronized(c04.T.class)
   			count--;
   			System.out.println(Thread.currentThread().getName()+"count = "+count);
   	}
   	public static void mm() {
   		synchronized (T.class) {//不可以使用synchronized(this)，因为静态的方法不需要new对象，就没有this
   			count--;
   			System.out.println(Thread.currentThread().getName()+"count = "+count);
   		}
   	}
   }
   ```

------



#### 实例

------

1. synchronized将代码块变成原子操作，不可分。

```java
public class T implements Runnable{
	private int count = 10;
	@Override
	public synchronized void run() {
		count--;
		System.out.println(Thread.currentThread().getName()+"count =="+count);
	}
	public static void main(String[] args) {
		T t = new T();
		for(int i = 0; i < 5; i++) {
			new Thread(t, "THREAD" + i).start();
		}
	}
}
```

在上面的代码中，class T继承了Runnable接口，实现了Run方法，使count自减并打印count的值。然后启动5个线程，如果run方法不加 synchronized 值打印会有相同的 ，出现了线程重用。

------

2. 在一个同步方法执行过程中，非同步方法也可以执行

   ```java
   public class T {
   	public synchronized void m1() {
   		System.out.println(Thread.currentThread().getName() + "m1 start");
   		try {
   			Thread.sleep(6000);
   		} catch (InterruptedException e) {
   			e.printStackTrace();
   		}
   		System.out.println(Thread.currentThread().getName() + "m1 end");
   	}
   	public void m2() {
   		try {
   			Thread.sleep(3000);
   		} catch (InterruptedException e) {
   			e.printStackTrace();
   		}
   		System.out.println(Thread.currentThread().getName() + "m2 start");
   	}
   	public static void main(String[] args) {
   		T t = new T();
   		//java8 lambda表达式 
   		new Thread(()->t.m1(),"t1").start();
   		new Thread(new Runnable() {
   			@Override
   			public void run() {
   				t.m2();
   			}},"t2").start();
   	}
   }
   ```

3. 模拟取钱

   ```java
   public class Account {
   	String name;
   	double balance;
   	public synchronized void set(String name,double balance) {
   		this.name = name;
   		try {
   			Thread.sleep(2000);
   		} catch (InterruptedException e) {
   			e.printStackTrace();
   		}
   		this.balance = balance;
   	}
   	public /*synchronized*/ double getBalance(String name) {
   		return this.balance;
   	}
   	public static void main(String[] args) {
   		Account a = new Account();
   		new Thread(()->a.set("zhangsan", 100.0)).start();
   		try {
   			TimeUnit.SECONDS.sleep(1);
   		} catch (InterruptedException e) {
   			e.printStackTrace();
   		}
   		System.out.println(a.getBalance("zhangsan"));
   		try {
   			TimeUnit.SECONDS.sleep(2);
   		} catch (InterruptedException e) {
   			e.printStackTrace();
   		}
   		System.out.println(a.getBalance("zhangsan"));
   	}
   }
   ```

   上面这段代码执行后，输出的两个值不同，原因是在同步方法set执行的过程中，非同步方法getBalance也可以执行，set方法中先设置name属性，然后线程睡了2s，还没设置balance属性，所以getBalance方法两次取得的结果不同。如果对getBalance加锁，就可以得到同样的结果。

#### 可重入锁

> synchronized获得的锁是可重入的

一个同步方法可以调用另外一个同步方法，一个线程已经拥有某个对象的锁，再次申请的时候仍然会得到该对象的锁。

```java
public class T {
	synchronized void m1() {
		System.out.println(Thread.currentThread().getName()+"m1 Start");
		try {
			TimeUnit.SECONDS.sleep(1);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		m2();//调用m2
		System.out.println(Thread.currentThread().getName()+"m1 end");
	}
	synchronized void m2() {
		try {
			TimeUnit.SECONDS.sleep(2);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println(Thread.currentThread().getName()+"m2 Start");
	}
	public static void main(String[] args) {
		T t =new T();
		new Thread(()->t.m1(),"t1").start();
	}
}
```

另一种重入锁是子类调用父类的同步方法，这是继承中可能发生的情形。

```java
public class T {
	synchronized void m1() {
		System.out.println(Thread.currentThread().getName()+"m1 Start");
		try {
			TimeUnit.SECONDS.sleep(1);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println(Thread.currentThread().getName()+"m1 end");
	}
	public static void main(String[] args) {
		TT tt = new TT();
		new Thread(()->tt.m1(),"tt").start();
		//new TT().m1();
	}
}
class TT extends T{
	@Override
	synchronized void m1() {
		System.out.println("child m1 Start");
		super.m1();
		System.out.println("child m1 end");
	}
}
```

#### 异常处理

程序执行过程中，如果出现异常，默认情况下锁会被释放，在并发处理的过程中，有异常要多加小心，不然可能会发生不一致的情况。在第一个线程中抛出异常，其他线程就会进入同步代码区，有可能访问到异常产生的数据。

```java
public class T {
	int count = 0;
	synchronized void m() {
		System.out.println(Thread.currentThread().getName() + " Start");
		while(true) {
			count++;
			System.out.println(Thread.currentThread().getName() + "count =" + count);
			try {
				TimeUnit.SECONDS.sleep(1);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			if(count == 5) {
				/*try {
					int i = 1/0;//这里抛出异常，锁将被释放，可以再这里进行catch，然后让循环继续。
				}catch(Exception e) {
					e.printStackTrace();
				}*/
				int i = 1/0;
			}
		}
	}
	public static void main(String[] args) {
		T t = new T();
		new Thread(()->t.m(),"t1").start();
		try {
			TimeUnit.SECONDS.sleep(3);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		new Thread(()->t.m(),"t2").start();
	}
}
```

在上面的的代码中，`t1`线程执行`count++`，当`count==5` 的时候,`int i = 1/0`会抛出异常，这时候锁就会被释放，然后线程`t2`就会进入到同步代码区，`t2`开始执行，count从之前的值开始增加。如果用`try catch`捕获异常，t1就会继续运行。

#### 模拟死锁

```java
public class DeadLock {
	private Object o1 = new Object();
	private Object o2 = new Object();
	public void m1() {
		synchronized(o1) {
			System.out.println(Thread.currentThread().getName()+"m1 Start,get o1 ");
			try {
				Thread.sleep(1000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			synchronized(o2) {
				System.out.println(Thread.currentThread().getName()+"m1 end,get o2 ");
			}
		}
	}
	public void m2() {
		synchronized(o2) {
			System.out.println(Thread.currentThread().getName()+"m2 Start,get o2 ");
			try {
				Thread.sleep(1000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			synchronized(o1) {
				System.out.println(Thread.currentThread().getName()+"m2 end,get o1 ");
			}
		}
	}
	public static void main(String[] args) {
		DeadLock dl = new DeadLock();
		new Thread(()->dl.m1(),"m1").start();
		new Thread(()->dl.m2(),"m2").start();
	}
}
```


