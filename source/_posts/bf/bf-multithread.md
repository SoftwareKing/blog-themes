---
title: 并发编程总结之多线程基础
categories: 并发编程
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
### 代码示例说明1
 运行main方法，main方法里有5个线程t1到t5，同一时间启动去访问MyThread类的Run方法。
<!--more-->
1. 不加synchronized关键字修饰run()方法的代码
```java
package org.xujin.multithread;

public class MyThread extends Thread {

    private int count = 5;

    public void run() {
        count--;
        System.out.println(this.currentThread().getName() + " count = " + count);
    }

    public static void main(String[] args) {
        /**
         * 分析：当多个线程访问myThread的run方法时，以排队的方式进行处理（这里排对是按照CPU分配的先后顺序而定的）， 一个线程想要执行synchronized修饰的方法里的代码： 1 尝试获得锁 2
         * 如果拿到锁，执行synchronized代码体内容；拿不到锁，这个线程就会不断的尝试获得这把锁，直到拿到为止， 而且是多个线程同时去竞争这把锁。（也就是会有锁竞争的问题）
         */
        MyThread myThread = new MyThread();
        Thread t1 = new Thread(myThread, "t1");
        Thread t2 = new Thread(myThread, "t2");
        Thread t3 = new Thread(myThread, "t3");
        Thread t4 = new Thread(myThread, "t4");
        Thread t5 = new Thread(myThread, "t5");
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
### 代码示例说明2
1. 当我们加上synchronized关键字修饰run()方法后，代码如下。
```java
public class MyThread extends Thread {

    private int count = 5;

    public synchronized void run() {
        count--;
        System.out.println(this.currentThread().getName() + " count = " + count);
    }

    public static void main(String[] args) {
        /**
         * 分析：当多个线程访问myThread的run方法时，以排队的方式进行处理（这里排对是按照CPU分配的先后顺序而定的）， 一个线程想要执行synchronized修饰的方法里的代码： 1 尝试获得锁 2
         * 如果拿到锁，执行synchronized代码体内容；拿不到锁，这个线程就会不断的尝试获得这把锁，直到拿到为止， 而且是多个线程同时去竞争这把锁。（也就是会有锁竞争的问题）
         */
        MyThread myThread = new MyThread();
        Thread t1 = new Thread(myThread, "t1");
        Thread t2 = new Thread(myThread, "t2");
        Thread t3 = new Thread(myThread, "t3");
        Thread t4 = new Thread(myThread, "t4");
        Thread t5 = new Thread(myThread, "t5");
        t1.start();
        t2.start();
        t3.start();
        t4.start();
        t5.start();
    }
}
```
2. 加上`synchronized`运行结果如下，线程名无序，无论你执行多少次程序，count--的值都是显示我们想要的正确结果。
```
t1 count = 4
t3 count = 3
t4 count = 2
t5 count = 1
t2 count = 0
```
### 小结
 当多个线程访问Mythread的run方法时，以排队的方式进行处理(排队的方式是按照CPU分配的饿先后顺序而定的)，一个线程想要执行synchronized修饰的方法里的代码，首先尝试获得锁，如果拿到锁，执行synchronized中代码体内容;拿不到锁，这个线程就会不断的尝试获得这把锁，直到拿到为止，而且是多个线程同时去竞争这把锁，也就是会有竞争锁的问题。

## 多个线程多个锁
多个线程多个锁:多个线程，每个线程都可以拿到自己指定的锁，分别获得锁之后，执行synchronized方法体的内容。
### 代码示例说明1
  1. 两个线程t1,t2分别依次start，访问两个对象的synchronized修饰的printNum方法，Code如下:
```java
/**
 * 关键字synchronized取得的锁都是对象锁，而不是把一段代码（方法）当做锁，
 * 所以代码中哪个线程先执行synchronized关键字的方法，哪个线程就持有该方法所属对象的锁（Lock），
 * 
 * 在静态方法上加synchronized关键字，表示锁定.class类，类一级别的锁（独占.class类）。
 * @author xujin
 *
 */
public class MultiThread {

    private int num = 0;

    public synchronized void printNum(String tag) {
        try {

            if (tag.equals("a")) {
                num = 100;
                System.out.println("tag a, set num over!");
                Thread.sleep(1000);
            } else {
                num = 200;
                System.out.println("tag b, set num over!");
            }

            System.out.println("tag " + tag + ", num = " + num);

        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    // 注意观察run方法输出顺序
    public static void main(String[] args) {

        // 两个不同的对象
        final MultiThread m1 = new MultiThread();
        final MultiThread m2 = new MultiThread();

        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                m1.printNum("a");
            }
        });

        Thread t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                m2.printNum("b");
            }
        });

        t1.start();
        t2.start();

    }
}
```
2. 执行结果如下:
```
tag a, set num over!
tag b, set num over!
tag b, num = 200
tag a, num = 100
```
>关键字synchronized取得的锁都是对象锁，而不是把一段代码（方法）当做锁，
 所以代码中哪个线程先执行synchronized关键字的方法，哪个线程就持有该方法所属对象的锁（Lock）

### 代码示例说明2
 1. 在静态方法上printNum（）加一个synchronized关键字修饰的话，那这个线程调用printNum()获得锁，就是这个类级别的锁。这是时候无论你实例化出多少个对象m1,m2都是没有任何关系的，代码Demo如下所示：
```java
public class MultiThread {

    // ②修改为static关键字修饰
    private static int num = 0;

    // ①修改为static修饰该方法
    public static synchronized void printNum(String tag) {
        try {

            if (tag.equals("a")) {
                num = 100;
                System.out.println("tag a, set num over!");
                Thread.sleep(1000);
            } else {
                num = 200;
                System.out.println("tag b, set num over!");
            }
            System.out.println("tag " + tag + ", num = " + num);

        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    // 注意观察run方法输出顺序
    public static void main(String[] args) {

        // 俩个不同的对象
        final MultiThread m1 = new MultiThread();
        final MultiThread m2 = new MultiThread();

        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                m1.printNum("a");
            }
        });

        Thread t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                m2.printNum("b");
            }
        });
        t1.start();
        t2.start();
    }
}
```
2. 运行结果如下,可以看出`t1执行完了`，然后`执行t2`,他们之间有一个顺序
```
tag a, set num over!
tag a, num = 100
tag b, set num over!
tag b, num = 200
```
### 多个线程多个锁小结
1. 关键字`synchronized`取得的`锁`都是`对象锁`，而不是把一段代码或方法当做`锁`，所以示例中代码中的`哪个线程先执行synchronized关键字的方法`，`哪个线程就持有该方法对象的锁`，也就是Lock,两个对象，线程获得的就是两个不同的锁，他们互不影响。
2. 有一种情况则是相同的锁，即在静态方法上加`synchronized`关键字，表示锁定`.class类`，类一级别的锁独占.class类。

## 对象锁的同步和异步
### 锁同步和异步的概念
1. 同步-synchronized
    同步的概念就是共享，需要记住共享这个概念，如果不是共享的资源，就没有必要同步。
2. 异步-asynchronized
    异步是相互独立的，相互之间不受任何约制，类似于http中的Ajax请求。

>同步的目的就是为了线程安全，其实对于线程安全来说，需要满足两个特性：`原子性`，`可见性`。

### 代码示例1

```java
public class TestObject {

    /** synchronized */
    public synchronized void method1() {
        try {
            System.out.println(Thread.currentThread().getName());
            //休眠4秒
            Thread.sleep(4000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    public void method2() {
        System.out.println(Thread.currentThread().getName());
    }

    public static void main(String[] args) {

        final TestObject mo = new TestObject();

        /**
         * 分析： t1线程先持有TestObject对象的Lock锁，t2线程可以以异步的方式调用对象中的非synchronized修饰的方法
         * t1线程先持有TestObject对象的Lock锁，t2线程如果在这个时候调用对象中的同步（synchronized）方法则需等待，也就是同步
         */
        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                mo.method1();
            }
        }, "t1");

        Thread t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                mo.method2();
            }
        }, "t2");

        t1.start();
        t2.start();

    }

}
```
运行结果如下，因为t1,t2两个线程访问TestObject对象的mo的method1，method2方法是异步的，所以直接打出。
```
t2
t1
```
>分析： t1线程若先持有TestObject对象的Lock锁，t2线程可以以异步的方式调用对象中的非synchronized修饰的方法，这就是异步。

### 代码示例2
把上面代码中的method2，也加上`synchronized`去修饰，代码如下:
```java
public class TestObject {

    public synchronized void method1() {
        try {
            System.out.println(Thread.currentThread().getName());
            Thread.sleep(4000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    /** 加上synchronized修饰method2 */
    public synchronized void method2() {
        System.out.println(Thread.currentThread().getName());
    }

    public static void main(String[] args) {

        final TestObject mo = new TestObject();

        /**
         * 分析： t1线程先持有TestObject对象的Lock锁，t2线程可以以异步的方式调用对象中的非synchronized修饰的方法
         * t1线程先持有TestObject对象的Lock锁，t2线程如果在这个时候调用对象中的同步（synchronized）方法则需等待，也就是同步
         */
        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                mo.method1();
            }
        }, "t1");

        Thread t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                mo.method2();
            }
        }, "t2");

        t1.start();
        t2.start();

    }

}
```

打印结果如下,由于CPU随机分配，若t1线程先执行，先打印t1,然后t1线程先休眠4s，后释放了Lock，然后打印t2。
```
t1
t2
```
> t1线程先持有TestObject对象的Lock锁，t2线程如果在这个时候调用对象中的同步（synchronized）方法则需等待，也就是同步
