---
title: 并发编程总结之synchronized细节问题
categories: 并发编程
date: 2016-11-26 21:00:00 +0800
toc: true
tags:
- 并发编程
- 多线程
---

**摘要**:本节主要介绍了并发编程下怎么避免数据脏读和什么是synchronized的可重入锁，synchronized的可重入锁的几种使用场景下，是线程安全的。以及一些细节的synchronized使用问题和synchronized常见代码块示例Code可以直接Copy运行。

---

## 脏读
### 什么是脏读
   对于对象的同步和异步方法，我们在设计程序，一定要考虑问题的整体性，不然会出现数据不一致的错误，最经典的错误就是脏读(DirtyRead)。

### 示例Code
业务整体需要使用完整的synchronized，保持业务的原子性。
<!--more-->
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
<!--more-->
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

### synchronized常见代码块

1. synchronized可以使用任意的Object进行加锁， 使用synchronized代码块加锁,比较灵活，如下代码所示:

```java
public class ObjectLock {

    public void method1() {
        // 对this当前ObjectLock实例对象加锁
        synchronized (this) {
            try {
                System.out.println("do method1..");
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public void method2() {
        // 对ObjectLock类加锁
        synchronized (ObjectLock.class) {
            try {
                System.out.println("do method2..");
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    // 任何对象锁
    private Object anyObjectlock = new Object();

    public void method3() {
        synchronized (anyObjectlock) {
            try {
                System.out.println("do method3..");
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {

        final ObjectLock objLock = new ObjectLock();
        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                objLock.method1();
            }
        });
        Thread t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                objLock.method2();
            }
        });
        Thread t3 = new Thread(new Runnable() {
            @Override
            public void run() {
                objLock.method3();
            }
        });

        t1.start();
        t2.start();
        t3.start();

    }

}
```
2.使用synchronized声明的方法在某些情况下，是有弊端的，比如A线程调用同步的方法执行一个很长时间的任务,那么B线程就必须等待很长的时间才可以执行，这样情况下可以使用synchronize的去优化代码执行时间，也就是我们通常所说的减小锁的粒度。

```java
   public class Optimize {

       public void doLongTimeTask() {
           try {

               System.out.println("当前线程开始：" + Thread.currentThread().getName() + ", 正在执行一个较长时间的业务操作，其内容不需要同步");
               Thread.sleep(2000);
               // 使用synchronized代码块减小锁的粒度，提高性能
               synchronized (this) {
                   System.out.println("当前线程：" + Thread.currentThread().getName() + ", 执行同步代码块，对其同步变量进行操作");
                   Thread.sleep(1000);
               }
               System.out.println("当前线程结束：" + Thread.currentThread().getName() + ", 执行完毕");

           } catch (InterruptedException e) {
               e.printStackTrace();
           }
       }

       public static void main(String[] args) {
           final Optimize otz = new Optimize();
           Thread t1 = new Thread(new Runnable() {
               @Override
               public void run() {
                   otz.doLongTimeTask();
               }
           }, "t1");
           Thread t2 = new Thread(new Runnable() {
               @Override
               public void run() {
                   otz.doLongTimeTask();
               }
           }, "t2");
           t1.start();
           t2.start();
       }

   }
```
 执行结果:

```
当前线程开始：t1, 正在执行一个较长时间的业务操作，其内容不需要同步
当前线程开始：t2, 正在执行一个较长时间的业务操作，其内容不需要同步
当前线程：t2, 执行同步代码块，对其同步变量进行操作
当前线程结束：t2, 执行完毕
当前线程：t1, 执行同步代码块，对其同步变量进行操作
当前线程结束：t1, 执行完毕
```

3.注意就是不要使用String的常量加锁，会出现死循环问题。

   synchronized代码块对字符串的锁，注意String常量池的缓存功能,示例代码如下:

   ```java
   public class StringLock {

   	public void method() {
   		synchronized ("字符串常量") {
   			try {
   				while(true){
   					System.out.println("当前线程 : "  + Thread.currentThread().getName() + "开始");
   					Thread.sleep(1000);		
   					System.out.println("当前线程 : "  + Thread.currentThread().getName() + "结束");
   				}
   			} catch (InterruptedException e) {
   				e.printStackTrace();
   			}
   		}
   	}
   	
   	public static void main(String[] args) {
   		final StringLock stringLock = new StringLock();
   		Thread t1 = new Thread(new Runnable() {
   			@Override
   			public void run() {
   				stringLock.method();
   			}
   		},"t1");
   		Thread t2 = new Thread(new Runnable() {
   			@Override
   			public void run() {
   				stringLock.method();
   			}
   		},"t2");
   		
   		t1.start();
   		t2.start();
   	}
   }
 ```

 **提示**:运行结果是:**t1线程一直死循环**。**t2线程不执行**。修改为如下代码,t1和t2线程交替执行

```java 
    public void method() {
           //把synchronized ("字符串常量") 修改为synchronized (new String("字符串常量"))
           synchronized (new String("字符串常量")) {
               try {
                   while (true) {
                       System.out.println("当前线程 : " + Thread.currentThread().getName() + "开始");
                       Thread.sleep(1000);
                       System.out.println("当前线程 : " + Thread.currentThread().getName() + "结束");
                   }
               } catch (InterruptedException e) {
                   e.printStackTrace();
               }
           }
       }
  ```
4.锁对象的改变问题:
   当使用一个对象进行加锁的时候，要注意对象本身发生变化的时候，那么持有的锁就不同。如果对象本身不发生改变，那么依然是同步的，即使是对象的属性发生了变化。

   4.1 示例代码1:`对象本身发生变化的时候,那么对象持有的锁就发生变化`

   ```java 
   public class ChangeLock {

       private String lock = "lock";

       private void method() {
           synchronized (lock) {
               try {
                   System.out.println("当前线程 : " + Thread.currentThread().getName() + "开始");
                   // 这里把锁的内容改变了，因此t1,t2线程基本同时进来，而不是t1休眠2秒后，t2进来
                   lock = "change lock";
                   Thread.sleep(2000);
                   System.out.println("当前线程 : " + Thread.currentThread().getName() + "结束");
               } catch (InterruptedException e) {
                   e.printStackTrace();
               }
           }
       }

       public static void main(String[] args) {

           final ChangeLock changeLock = new ChangeLock();
           Thread t1 = new Thread(new Runnable() {
               @Override
               public void run() {
                   changeLock.method();
               }
           }, "t1");
           Thread t2 = new Thread(new Runnable() {
               @Override
               public void run() {
                   changeLock.method();
               }
           }, "t2");
           t1.start();
           try {
               Thread.sleep(100);
           } catch (InterruptedException e) {
               e.printStackTrace();
           }
           t2.start();
       }

   }
   ```
   4.2 示例代码2:`同一对象属性的修改不会影响锁的情况`

   ```java
   public class ModifyLock {

       private String name;
       private int age;

       public String getName() {
           return name;
       }

       public void setName(String name) {
           this.name = name;
       }

       public int getAge() {
           return age;
       }

       public void setAge(int age) {
           this.age = age;
       }

       public synchronized void changeAttributte(String name, int age) {
           try {
               System.out.println("当前线程 : " + Thread.currentThread().getName() + " 开始");
               this.setName(name);
               this.setAge(age);

               System.out.println("当前线程 : " + Thread.currentThread().getName() + " 修改对象内容为： " + this.getName() + ", "
                       + this.getAge());

               Thread.sleep(2000);
               System.out.println("当前线程 : " + Thread.currentThread().getName() + " 结束");
           } catch (InterruptedException e) {
               e.printStackTrace();
           }
       }

       public static void main(String[] args) {
           final ModifyLock modifyLock = new ModifyLock();
           Thread t1 = new Thread(new Runnable() {
               @Override
               public void run() {
                   modifyLock.changeAttributte("许进", 25);
               }
           }, "t1");
           Thread t2 = new Thread(new Runnable() {
               @Override
               public void run() {
                   modifyLock.changeAttributte("李四X", 21);
               }
           }, "t2");

           t1.start();
           try {
               Thread.sleep(100);
           } catch (InterruptedException e) {
               e.printStackTrace();
           }
           t2.start();
       }

   }
   ```
   运行结果:
   ```
   当前线程 : t1 开始
   当前线程 : t1 修改对象内容为： 许进, 25
   当前线程 : t1 结束
   当前线程 : t2 开始
   当前线程 : t2 修改对象内容为： 李四X, 21
   当前线程 : t2 结束
   ```

