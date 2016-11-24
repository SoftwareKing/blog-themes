---
title: 跟我学并发编程之多线程上篇
categories: 跟我学并发编程
date: 2016-11-23 21:00:00 +0800
toc: true
tags:
- 并发编程
- 多线程
---
## 线程安全
### 线程安全概念
 1. 当多个线程访问访问某一个`类(对象或方法)`时，这个类或对象或方法`始终`能表现出`正确的行为`或我们想要的`结果`，那么这个类(对象或方法)就是`线程安全`的。
 2. synchronized：可以在`任意的对象`及`方法`上加锁，而加锁的这段代码称之为`互斥区`或者`临界区`。
### 代码示例说明
 运行main方法，main方法里有5个线程t1到t5，同一时间启动去访问MyThread类的Run方法。
<!--more-->
1. 不加synchronized关键字修饰run()方法的代码
```java
package org.xujin.multithread;

/**
 * 线程安全概念：当多个线程访问某一个类（对象或方法）时，这个对象始终都能表现出正确的行为，那么这个类（对象或方法）就是线程安全的。
 * synchronized：可以在任意对象及方法上加锁，而加锁的这段代码称为"互斥区"或"临界区"
 * @author xujin
 */
public class MyThread extends Thread{
	
	private int count = 5 ;
	
	public void run(){
		count--;
		System.out.println(this.currentThread().getName() + " count = "+ count);
	}
	
	public static void main(String[] args) {
		/**
		 * 分析：当多个线程访问myThread的run方法时，以排队的方式进行处理（这里排对是按照CPU分配的先后顺序而定的），
		 * 		一个线程想要执行synchronized修饰的方法里的代码：
		 * 		1 尝试获得锁
		 * 		2 如果拿到锁，执行synchronized代码体内容；拿不到锁，这个线程就会不断的尝试获得这把锁，直到拿到为止，
		 * 		   而且是多个线程同时去竞争这把锁。（也就是会有锁竞争的问题）
		 */
		MyThread myThread = new MyThread();
		Thread t1 = new Thread(myThread,"t1");
		Thread t2 = new Thread(myThread,"t2");
		Thread t3 = new Thread(myThread,"t3");
		Thread t4 = new Thread(myThread,"t4");
		Thread t5 = new Thread(myThread,"t5");
		t1.start();
		t2.start();
		t3.start();
		t4.start();
		t5.start();
	}
}

```
运行结果如下，没有出现我们想要的结果,打印出来的线程名是无序的 count值也没按正常--，运行多次不能保证count打印的值每次一致，因此出现了线程安全问题。
```
t1 count = 2
t2 count = 2
t5 count = 0
t3 count = 2
t4 count = 1
```
2. 当我们加上synchronized关键字修饰run()方法后，代码如下。
```java
public class MyThread extends Thread{
	
	private int count = 5 ;
	
	public synchronized void run(){
		count--;
		System.out.println(this.currentThread().getName() + " count = "+ count);
	}
	
	public static void main(String[] args) {
		/**
		 * 分析：当多个线程访问myThread的run方法时，以排队的方式进行处理（这里排对是按照CPU分配的先后顺序而定的），
		 * 		一个线程想要执行synchronized修饰的方法里的代码：
		 * 		1 尝试获得锁
		 * 		2 如果拿到锁，执行synchronized代码体内容；拿不到锁，这个线程就会不断的尝试获得这把锁，直到拿到为止，
		 * 		   而且是多个线程同时去竞争这把锁。（也就是会有锁竞争的问题）
		 */
		MyThread myThread = new MyThread();
		Thread t1 = new Thread(myThread,"t1");
		Thread t2 = new Thread(myThread,"t2");
		Thread t3 = new Thread(myThread,"t3");
		Thread t4 = new Thread(myThread,"t4");
		Thread t5 = new Thread(myThread,"t5");
		t1.start();
		t2.start();
		t3.start();
		t4.start();
		t5.start();
	}
}
```
加上`synchronized`运行结果如下，线程名无序，无论你执行多少次程序，count--的值都是显示我们想要的正确结果。
```
t1 count = 4
t3 count = 3
t4 count = 2
t5 count = 1
t2 count = 0
```
小结:当多个线程访问Mythread的run方法时，以排队的方式进行处理(排队的方式是按照CPU分配的饿先后顺序而定的)，一个线程想要执行synchronized修饰的方法里的代码，首先尝试获得锁，如果拿到锁，执行synchronized中代码体内容;拿不到锁，这个线程就会不断的尝试获得这把锁，直到拿到为止，而且是多个线程同时去竞争这把锁，也就是会有竞争锁的问题。

### 多个线程多个锁
多个线程多个锁:多个线程，每个线程都可以拿到自己指定的锁，分别获得锁之后，执行synchronized方法体的内容。

关键字synchronized取得的锁都是对象锁，而不是把一段代码或方法当做锁，所以示例中代码中的哪个线程先执行synchronized关键字的方法，哪个线程就持有该方法对象的锁，也就是Lock,两个对象，线程获得的就是两个不同的锁，他们互不影响。
   有一种情况则是相同的锁，即在静态方法上加synchronized关键字，表示锁定.class类，类一级别的锁独占.class类。
-----未完待续-------