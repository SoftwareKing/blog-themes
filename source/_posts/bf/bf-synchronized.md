---
title: 并发编程总结之synchronized细节问题
categories: 并发编程
date: 2016-11-26 21:00:00 +0800
toc: true
tags:
- 并发编程
- 多线程
---

**摘要**:本节主要介绍了并发编程下怎么避免数据脏读和什么是synchronized的可重入锁，synchronized的可重入锁的几种使用场景下，是线程安全的。

---

## 脏读
### 什么是脏读
   对于对象的同步和异步方法，我们在设计程序，一定要考虑问题的整体性，不然会出现数据不一致的错误，最经典的错误就是脏读(DirtyRead)。
<!--more-->
### 示例Code
```java
/**
 * 业务整体需要使用完整的synchronized，保持业务的原子性。
 * 
 * @author xujin
 *
 */
public class DirtyRead {

    private String username = "xujin";
    private String password = "123";

    public synchronized void setValue(String username, String password) {
        this.username = username;
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        this.password = password;

        System.out.println("setValue最终结果：username = " + username + " , password = " + password);
    }
    //①这里getValue没有加synchronized修饰
    public void getValue() {
        System.out.println("getValue方法得到：username = " + this.username + " , password = " + this.password);
    }

    public static void main(String[] args) throws Exception {

        final DirtyRead dr = new DirtyRead();
        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                dr.setValue("张三", "456");
            }
        });
        t1.start();
        Thread.sleep(1000);

        dr.getValue();
    }

}
```
上面的Code中，getValue没有加synchronized修饰，打印结果如下,出现脏读
```
getValue方法得到：username = 张三 , password = 123
setValue最终结果：username = 张三 , password = 456
```
只需在getValue加synchronized修饰，如下:
```java
public synchronized void getValue() {
  System.out.println("getValue方法得到：username = " + this.username + " , password = " + this.password);
}
```
运行结果如下,没有造成数据脏读
```
setValue最终结果：username = 张三 , password = 456
getValue方法得到：username = 张三 , password = 456
```
### 小结
 在我们对对象中的一个方法加锁的时候，需要考虑业务的或程序的整体性，也就是为程序中的set和get方法同时加锁synchronized同步关键字，保证业务的(service层)的原子性，不然会出现数据错误，脏读。

## synchronized的重入
### 什么是synchronized的重入锁
1. synchronized,它拥有强制原子性的内置锁机制,是一个重入锁,所以在使用synchronized时,当一个线程请求得到一个对象锁后再次请求此对象锁,可以再次得到该对象锁,就是说在一个synchronized方法/块的内部调用本类的其他synchronized方法/块时，是永远可以拿到锁。
2. 当线程请求一个由其它线程持有的对象锁时，该线程会阻塞，而当线程请求由自己持有的对象锁时，如果该锁是重入锁,请求就会成功,否则阻塞.

>简单的说:关键字synchronized具有`锁重入`的功能，也就是在使用`synchronized时`，`当一个线程`得到一个`对象锁`的`锁后`，`再次请求此对象时`可以`再次`得到该`对象对应的锁`。

### 嵌套调用关系synchronized的重入

嵌套调用关系synchronized的重入也是线程安全的，下面是method1，method2，method3都被synchronized修饰，调用关系method1-->method2-->method3,也是线程安全的。
```java
/**
 * synchronized的重入
 * 
 * @author xujin
 *
 */
public class SyncReenTrant {

    public synchronized void method1() {
        System.out.println("method1..");
        method2();
    }

    public synchronized void method2() {
        System.out.println("method2..");
        method3();
    }

    public synchronized void method3() {
        System.out.println("method3..");
    }

    public static void main(String[] args) {
        final SyncReenTrant sd = new SyncReenTrant();
        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                sd.method1();
            }
        });
        t1.start();
    }
```
运行结果如下:
```
method1..
method2..
method3..
```
### 继承关系的synchronized的重入
简单 Code1：
```
public class Son extends Father {

    public synchronized void doSomething() {
        System.out.println("child.doSomething()");
        // 调用自己类中其他的synchronized方法
        doAnotherThing();

    }

    private synchronized void doAnotherThing() {
        // 调用父类的synchronized方法
        super.doSomething();
        System.out.println("child.doAnotherThing()");
    }

    public static void main(String[] args) {
        Son child = new Son();
        child.doSomething();
    }
}

class Father {
    public synchronized void doSomething() {
        System.out.println("father.doSomething()");
    }

}
```
运行结果:
```
child.doSomething()
father.doSomething()
child.doAnotherThing()
```
1. 这里的对象锁只有一个,就是child对象的锁,当执行child.doSomething时，该线程获得child对象的锁，在doSomething方法内执行doAnotherThing时再次请求child对象的锁，因为synchronized是重入锁，所以可以得到该锁，继续在doAnotherThing里执行父类的doSomething方法时第三次请求child对象的锁，同理可得到，如果不是重入锁的话，那这后面这两次请求锁将会被一直阻塞，从而导致死锁。
2. 所以在Java内部，同一线程在调用自己类中其他synchronized方法/块或调用父类的synchronized方法/块都不会阻碍该线程的执行，就是说同一线程对同一个对象锁是可重入的，而且同一个线程可以获取同一把锁多次，也就是可以多次重入。因为java线程是基于“每线程（per-thread）”，而不是基于“每调用（per-invocation）”的（java中线程获得对象锁的操作是以每线程为粒度的，per-invocation互斥体获得对象锁的操作是以每调用作为粒度的）

>我们再来看看重入锁是怎么实现可重入性的，其实现方法是为每个锁关联一个线程持有者和计数器，当计数器为0时表示该锁没有被任何线程持有，那么任何线程都可能获得该锁而调用相应的方法；当某一线程请求成功后，JVM会记下锁的持有线程，并且将计数器置为1；此时其它线程请求该锁，则必须等待；而该持有锁的线程如果再次请求这个锁，就可以再次拿到这个锁，同时计数器会递增；当线程退出同步代码块时，计数器会递减，如果计数器为0，则释放该锁。

```java
public class SyncExtends {

    // 父类
    static class Father {
        public int i = 10;

        public synchronized void operationSup() {
            try {
                i--;
                System.out.println("Father print i = " + i);
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    // 子类继承父类
    static class Son extends Father {
        public synchronized void operationSub() {
            try {
                while (i > 0) {
                    i--;
                    System.out.println("Son print i = " + i);
                    Thread.sleep(100);
                    this.operationSup();
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {

        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                Son sub = new Son();
                sub.operationSub();
            }
        });

        t1.start();
    }

}
```
运行结果如下:
```
Son print i = 9
Father print i = 8
Son print i = 7
Father print i = 6
Son print i = 5
Father print i = 4
Son print i = 3
Father print i = 2
Son print i = 1
Father print i = 0
```

参考文章:
http://blog.csdn.net/aigoogle/article/details/29893667