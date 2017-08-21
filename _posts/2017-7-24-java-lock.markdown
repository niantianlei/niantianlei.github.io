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
  




  * [公平锁与非公平锁](#jump1)
  * [共享锁与排他锁(读写锁)](#jump2)



#### <span id = "jump1">公平锁与非公平锁</span>  
公平锁表示线程获取锁的顺序是按照线程加锁的顺序来完成分配，即FIFO先进先出的顺序。  
而非公平锁就是一种获取锁的抢占机制，是随机获得锁的，先来的不一定先得到锁，也可能导致某些线程一直拿不到锁，结果就不公平了。  
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


#### <span id = "jump2">共享锁与排他锁</span>  
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
