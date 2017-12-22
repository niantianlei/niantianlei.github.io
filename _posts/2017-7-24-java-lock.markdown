---
layout: post
title:  "Java锁的种类"
subtitle:  " \"various locks in Java\""
date:   2017-07-24 10:23:18 +0800
tags:
    - Java
author: Nian Tianlei
header-img: "img/post-bg-2016.jpg"
---

> This document is not completed and will be updated in the future.
  



  * [可重入锁](#jump3)
  * [公平锁与非公平锁](#jump1)
  * [读写锁](#jump2)
  * [排他锁/共享锁](#jump4)
  * [乐观锁/悲观锁](#jump5)
  * [自旋锁](#jump6)
  * [类锁/对象锁](#jump7)
  * [无锁/偏向锁/轻量级锁/重量级锁](#jump8)
  * [锁消除](#jump9)
  * [锁粗化](#jump10)
  * [分段锁](#jump11)

#### <span id = "jump3">可重入锁</span>  
能够支持一个线程对资源的重复加锁，即线程在获得锁之后能够再次获取锁而不会阻塞。  
锁需要去识别获取锁的线程是否为当前占据锁的线程。如果是，则再次成功获取。  
线程重复n次获取了锁，随后在第n次释放该锁后，其他线程能够获取到该锁。  

```
public class testSynchronized extends Thread {
	public static void main(String[] args) {
		new testSynchronized().start();
	}
	public void run() {
		new Helper().A();
	}
}
class Helper {
	synchronized void A() {
		System.out.println("进入A()");
		B();
	}
	synchronized void B() {
		System.out.println("进入B()");
	}
}
```
运行结果  
![可重入锁]({{ "/img/post/locks/reentrant.png" | prepend: site.baseurl }} )  
这里的synchronized锁的this即当前对象，因此根据可重入原理可以再进入B方法。  
#### <span id = "jump1">公平锁与非公平锁</span>  
公平锁表示线程获取锁的顺序是按照线程申请锁的顺序来完成分配，即FIFO先进先出的顺序。因此，公平锁能够减少“饥饿”发生的概率。（饥饿：一个线程因为CPU时间全部被其他线程抢走而得不到CPU运行时间）  
而非公平锁就是一种获取锁的抢占机制，是随机获得锁的，先来的不一定先得到锁，也可能导致某些线程一直拿不到锁，结果就不公平了。  
ReentrantLock默认为非公平锁。  
非公平锁不需维护队列，效率较高。  
对于内置锁`synchronized`，是一种非公平锁。但不像ReentrantLock是通过AQS实现线程调度，所以不能变成公平锁。  
例子：  
```
package concurrent;

import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class TestMain {  
    public static void main(String[] args) {  
        Service service = new Service(true);  
        Thread threadArray[] = new Thread[20];
        MyThread my = new MyThread(service);
        for(int i=0; i < 20; i++){  
        	threadArray[i] = new Thread(my);  
        }  
        for(int i=0; i < 20; i++) {
        	threadArray[i].start(); 
        }
          
    }  
}  
  
class Service {  
    public Lock lock;
    public Service(boolean isFair) {
    	lock = new ReentrantLock(isFair); 
    }  
  
    public void method() {
    	try {
    		lock.lock();
    		System.out.println(Thread.currentThread().getName() + " 获得锁定");  
    	} finally {
    		lock.unlock();
    	}
    }  
}  
  
class MyThread implements Runnable{  
    private Service service;  
      
    public MyThread(Service service) {  
        this.service = service;  
    }  
      
    public void run(){  
        System.out.println(Thread.currentThread().getName() + " 运行了");  
        service.method();  
    }  
  
}  
```
公平锁的结果如图，打印结果基本有序，先运行的先获得锁。  
![公平锁]({{ "/img/post/locks/1.png" | prepend: site.baseurl }} ) 
然后，创建lock对象时，参数为false。在本例中`Service service = new Service(false);` ,得到结果，可见是随机获得锁的      
![非公平锁]({{ "/img/post/locks/2.png" | prepend: site.baseurl }} ) 


#### <span id = "jump2">读写锁</span>  
ReentrantLock类具有完全互斥的作用，同一时间只有一个线程在执行锁内任务。这样虽然保证了实例变量的线程安全性，但效率很低。  
读写锁ReentrantReadWriteLock类可以加快运行效率，在不需要操作实例变量的方法中，可以使用它来提高运行速度。  
其中，读操作相关的锁称为共享锁；写操作相关的锁称为排他锁。多个读锁之间不互斥，读锁与写锁互斥，写锁与写锁互斥。即多个Thread可以同时进行读取操作但同一时刻只允许一个Thread进行写入操作。  
**读间共享:**  
```
package concurrent;

import java.util.concurrent.locks.ReentrantReadWriteLock;

public class TestMain {  
    public static void main(String[] args) {  
    	Service service = new Service();

		MyThread a = new MyThread("A", service);
		MyThread b = new MyThread("B", service);

		a.start();
		b.start();
    }  
}  
  
class Service {
	private ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

	public void read() {
		try {
			try {
				lock.readLock().lock();
				System.out.println("获得读锁, 时间: " + Thread.currentThread().getName()
						+ " " + System.currentTimeMillis());
				Thread.sleep(10000);
			} finally {
				lock.readLock().unlock();
			}
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}

}

class MyThread extends Thread {
	private Service service;
	
	public MyThread(String name, Service service) {
		super(name);
		this.service = service;
	}

	@Override
	public void run() {
		service.read();
	}
}
```
结果:  
![读共享]({{ "/img/post/locks/3.png" | prepend: site.baseurl }} )  
**写间互斥：**  
改变Service类代码中的方法  
```
	public void read() {
		try {
			try {
				lock.writeLock().lock();
				System.out.println("获得写锁, 时间: " + Thread.currentThread().getName()
						+ " " + System.currentTimeMillis());
				Thread.sleep(1000);
			} finally {
				lock.writeLock().unlock();
			}
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
```
结果:  
![写互斥]({{ "/img/post/locks/4.png" | prepend: site.baseurl }} )
明显看出有先后顺序，时间上并不一致。  

#### <span id = "jump4">排他锁/共享锁</span>   
如上面的读写锁提到的，排他锁是指该锁一次只能被一个线程所持有。共享锁是指该锁可被多个线程所持有。  
读锁是共享锁，写锁是排他锁。  
读锁的共享锁可保证并发读是非常高效的，读写，写读 ，写写的过程是互斥的。
排他锁与共享锁也是通过AQS来实现的。  
对于Synchronized而言，当然是排他锁。  

#### <span id = "jump5">乐观锁/悲观锁</span>   
悲观锁操作数据时，认为一定发生修改。因此会先加锁，再操作。悲观的认为，不加锁的并发操作一定会出问题。  
乐观锁认为操作数据时，是不会发生修改的。在更新数据的时候，会采用尝试更新，不断重新的方式更新数据。乐观的认为，不加锁的并发操作不会出问题。  

由此可知，悲观锁适合写操作非常多的场景，乐观锁适合读操作非常多的场景，不加锁会带来大量的性能提升。  
在Java中，悲观锁的实现就是利用各种锁。  
乐观锁的实现，是无锁编程，常常采用CAS算法，原子类AtomicBoolean、AtomicInteger、AtomicLong等，通过CAS自旋实现原子操作的更新。  

#### <span id = "jump6">自旋锁</span>   
在Java中，自旋锁是指尝试获取锁的线程不会进入睡眠，而是采用循环的方式去尝试获取锁，这样的好处是减少线程上下文切换的消耗，缺点是循环会消耗CPU。  
如果前一个线程占用锁的时间很长，那么自旋线程会白白浪费处理器资源。  
重量级锁中，线程不使用自旋。  

#### <span id = "jump7">类锁/对象锁</span>   
类锁：在方法上加上static synchronized的锁，或synchronized(xxx.class)的锁。  
对象锁：对象方法用synchronized修饰，或synchronized(对象)。  

#### <span id = "jump8">无锁/偏向锁/轻量级锁/重量级锁</span>   
这三种锁是指锁的状态，并且是针对Synchronized。  这三种锁的状态是通过对象监视器在对象头中的字段来表明的。  
偏向锁是指一段同步代码一直被一个线程所访问，那么该线程会自动获取锁。降低获取锁的代价。
轻量级锁是指当锁是偏向锁的时候，被另一个线程所访问，偏向锁就会升级为轻量级锁，其他线程会通过自旋的形式尝试获取锁，不会阻塞，提高性能。
重量级锁是指当锁为轻量级锁的时候，另一个线程虽然是自旋，但自旋不会一直持续下去，当自旋一定次数的时候，还没有获取到锁，就会进入阻塞，该锁膨胀为重量级锁。重量级锁会让其他申请的线程进入阻塞，性能降低。

#### <span id = "jump9">锁消除</span>   
锁消除是Java虚拟机在JIT编译是，通过对运行上下文的扫描，去除不可能存在共享资源竞争的锁，通过锁消除，可以节省毫无意义的请求锁时间。  

#### <span id = "jump10">锁粗化</span>  
锁粗化，如果虚拟机探测到有这样一串零碎的操作都对同一个对象加锁，将会把加锁同步的范围扩展到整个操作序列的外部，这样就只需要加锁一次就够了。  
例如：StringBuffer对象的append()操作对同一个对象多次加锁解锁，就会扩大锁同步的范围。  

#### <span id = "jump11">分段锁</span>   
待更新