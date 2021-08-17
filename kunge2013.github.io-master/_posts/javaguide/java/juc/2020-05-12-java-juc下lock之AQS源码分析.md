---
layout: post
title:  "juc下lock之AQS源码分析"
date:   2020-05-12 23:06:06
categories: java
tags: java
---

###### 一.AQS 之ReentrantLock源码分析

-   1.ReentrantLock(重入锁) 可以中断锁?
	
	可中断，即线程在运行中或者等待时可以调用线程中断方法，销毁线程而不影响程序的整体执行


-   2.ReentrantLock可以实现公平锁和非公平锁
	
	公平锁，即多个线程之间 可以轮询获取锁对象

-   3.什么是重入锁？
	
	重入锁可以多次在同一个线程内调用 lock.lock();释放也需要多次 调用lock.unlock()释放
	
	

-   4.下面一段代码体现可重入，可中断的特性

		package com.kframe;
		
		import java.util.concurrent.locks.LockSupport;
		import java.util.concurrent.locks.ReentrantLock;
		
		public class ReentrantLockDemo implements Runnable {
		
		// 公平锁
		private static ReentrantLock lock = new ReentrantLock(true);
	
		@Override
		public void run() {
			while (true) {
				/**
				 * 可重入锁 多加锁
				 */
				lock.lock();
				lock.lock();
				try {
					System.out.println(Thread.currentThread().getName() + " get lock");
					Thread.sleep(1000);
					
				} catch (Exception e) {
					e.printStackTrace();
				} finally {
					/**
					 * 可重入锁 多次解锁
					 */
					lock.unlock();
					lock.unlock();
				}
			}
		}
	
		public static void main(String[] args) {
	//		testFailSyn();//公平锁
			testInterupt();//可中断锁
		}
		
		/**
		 * 公平锁
		 */
		public static void testFailSyn() {
			ReentrantLockDemo rtld = new ReentrantLockDemo();
			Thread thread1 = new Thread(rtld);
			Thread thread2 = new Thread(rtld);
			thread1.start();
			thread2.start();
		}
		/**
		 * 可中断
		 */
		public static void testInterupt() {
			
			Thread thread1 =new Thread(() -> {
				lock.lock();
				while (true) {
					if (Thread.currentThread().isInterrupted()) {
						System.out.println(Thread.currentThread().getName() + "===线程中断");
						lock.unlock();
						break;
					}
					System.out.println(Thread.currentThread().getName() + "=====继续执行");
				}
				
			}) ;
			thread1.start();
			
			Thread thread2 =new Thread(() -> {
				lock.lock();
				System.out.println("线程  ===" + Thread.currentThread().getName());
				lock.unlock();
				
			}) ;
			thread2.start();
			thread1.interrupt();
		}
	}

---

###### 二.ReentrantLock可重入原理分析

-   1.ReentrantLock核心是根据 ReentrantLock.Sync来实现，而 Sync抽象类父类是 AbstractQueuedSynchronizer，Sync的子类有两个分别是 NonfairSync（非公平锁）和 FairSync（公平锁）
ReentrantLock默认无参构造方法是 NonfairSync（非公平锁）具体代码如下、

	
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



-   2.ReentrantLock NonfairSync（非公平锁原理）lock 源码  AQS调用时序图

<div align="left">  
	<img src="https://kunge2013.go123.live/images/frame/jdk/juc/非公平锁之AQS.png" width="100%"/>
</div>


-   3.ReentrantLock NonfairSync（非公平锁原理）unlock 源码  AQS调用时序图

<div align="left">  
	<img src="https://kunge2013.go123.live/images/frame/jdk/juc/非公平锁之AQS.png" width="100%"/>
</div>

-   3.代码如下
[相关源码](https://github.com/kunge2013/javasource.git)