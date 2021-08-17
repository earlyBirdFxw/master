---
layout: post
title:  "reference包源码分析"
date:   2020-06-10 23:06:06
categories: java
tags: java
---

## 1,引用类型


|  类型	  | 对应类	|   特征 |
| ----  | ----  | ----  |
| 强引用	  |    		|	强引用的对象绝对不会被gc回收	   |	
|软引用	|SoftReference|如果物理内存充足则不会被gc回收,如果物理内存不充足则会被gc回收。|
|弱引用	|WeakReference|一旦被gc扫描到则会被回收|
|虚引用	|PhantomReference|不会影响对象的生命周期，形同于无，任何时候都可能被gc回收|
|	|FinalReference|用于收尾机制(finalization)|

![reference关系](https://kunge2013.go123.live/images/frame/jdk/reference/reference.png)

## 2, FinalReference

FinalReference访问权限为package，并且只有一个子类Finalizer，同时Finalizer 是final修饰的类，所以无法继承扩展。

​ 与Finalizer相关联的则是Object中的finalize()方法，在类加载的过程中，如果当前类有覆写finalize()方法，则其对象会被标记为finalizer类，这种类型的对象被回收前会先调用其finalize()。

​ 具体的实现机制是，在gc进行可达性分析的时候，如果当前对象是finalizer类型的对象，并且本身不可达（与GC Roots无相连接的引用），则会被加入到一个ReferenceQueue类型的队列(F-Queue)中。而系统在初始化的过程中，会启动一个FinalizerThread实例的守护线程(线程名Finalizer)，该线程会不断消费F-Queue中的对象，并执行其finalize()方法(runFinalizer)，并且runFinalizer方法会捕获Throwable级别的异常，也就是说finalize()方法的异常不会导致FinalizerThread运行中断退出。对象在执行finalize()方法后，只是断开了与Finalizer的关联，并不意味着会立即被回收，还是要等待下一次GC，而每个对象的finalize()方法都只会执行一次，不会重复执行。

​ finalize()方法是对象逃脱死亡命运的最后一次机会，如果在该方法中将对象本身(this关键字) 赋值给某个类变量或者对象的成员变量，那在第二次标记时它将被移出"即将回收的集合"。

——《深入理解java虚拟机》

注意：finalize()使用不当会导致内存泄漏和内存溢出，比如SocksSocketImpl之类的服务会在finalize()中加入close()操作用于释放资源，但是如果FinalizerThread一直没有执行的话就会导致资源一直无法释放，从而出现内存泄漏。还有如果某对象的finalize()方法执行时间太长或者陷入死循环，将导致F-Queue一直堆积，从而造成内存溢出(oom)。
	
### 2.1, Finalizer

-  FinalizerThread
	
		  //消费ReferenceQueue并执行对应元素对象的finalize()方法
		    private static class FinalizerThread extends Thread {
		        ......
		        public void run() {
		            ......
		            final JavaLangAccess jla = SharedSecrets.getJavaLangAccess();
		            running = true;
		            for (;;) {
		                try {
		                    Finalizer f = (Finalizer)queue.remove();
		                    f.runFinalizer(jla);
		                } catch (InterruptedException x) {
		                }
		            }
		        }
		    }
	
		//初始化的时候启动FinalizerThread(守护线程)
		    static {
		        ThreadGroup tg = Thread.currentThread().getThreadGroup();
		        for (ThreadGroup tgn = tg;
		             tgn != null;
		             tg = tgn, tgn = tg.getParent());
		        Thread finalizer = new FinalizerThread(tg);
		        finalizer.setPriority(Thread.MAX_PRIORITY - 2);
		        finalizer.setDaemon(true);
		        finalizer.start();
		    }
-   add
	在jvm启动的时候就会启动一个守护线程去消费引用队列，并调用引用队列指向对象的finalize()方法。
	jvm在注册finalize()方法被覆写的对象的时候会创建一个Finalizer对象，并且将该对象加入一个双向链表中：


	    static void register(Object finalizee) {
	        new Finalizer(finalizee);
	    }
	    private Finalizer(Object finalizee) {
	        super(finalizee, queue);
	        add();
	    }
	    private void add() { 
	        synchronized (lock) { //头插法构建Finalizer对象的链表
	            if (unfinalized != null) {
	                this.next = unfinalized;
	                unfinalized.prev = this;
	            }
	            unfinalized = this;
	        }
	    }


另外还有两个附加线程用于消费Finalizer链表以及队列:
Runtime.runFinalization()会调用runFinalization()用于消费Finalizer队列，而java.lang.Shutdown则会在jvm退出的时候(jvm关闭钩子)调用runAllFinalizers()用于消费Finalizer链表。


## 3, SoftReference

系统将要发生内存溢出(oom)之前，会回收软引用的对象，如果回收后还没有足够的内存，抛出内存溢出异常；

使用SoftReference类，将要软引用的对象最为参数传入；

构造方法传入ReferenceQueue队列的时候，如果引用的对象被回收，则将其加入该队列。

	public SoftReference(T referent);	//根据传入的引用创建软引用					
	public SoftReference(T referent, ReferenceQueue<? super T> q); //根据传入的引用和注册队列创建软引用

使用示例：

	   ReferenceQueue<String> referenceQueue = new ReferenceQueue<>();
	        SoftReference<String> softReference = new SoftReference<>("abc", referenceQueue);
	        System.gc();
	        System.out.println(softReference.get());
	        Reference<? extends String> reference = referenceQueue.poll();
	        System.out.println(reference);
	      
运行结果如下：


	abc
	null

软引用可用来实现内存敏感的高速缓存

## 4, WeakReference

WeakReference与SoftReference类似，区别在于WeakReference的生命周期更短，一旦发生GC就会被回收，不过由于gc的线程优先级比较低，所以WeakReference不会很快被GC发现并回收。

使用WeakReference类，将要弱引用的对象最为参数传入；

构造方法传入ReferenceQueue队列的时候，如果引用的对象被回收，则将其加入该队列。

	WeakReference(T referent);			//根据传入的引用创建弱引用
	WeakReference(T referent, ReferenceQueue<? super T> q); //根据传入的引用和注册队列创建弱引用

使用示例:

	public class WeakReferenceTest {
	    public static void main(String[] args) {
	        ReferenceQueue<String> rq = new ReferenceQueue<>();
	        //这里必须用new String构建字符串，而不能直接传入字面常量字符串
	        Reference<String> r = new WeakReference<>(new String("java"), rq);
	        Reference rf;
	        //一次System.gc()并不一定会回收A，所以要多试几次
	        while((rf=rq.poll()) == null) {
	            System.gc();
	        }
	        System.out.println(rf);
	        if (rf != null) {
	            //引用指向的对象已经被回收，存入引入队列的是弱引用本身,所以这里最终返回null
	            System.out.println(rf.get());
	        }
	    }
	}


运行结果：

	java.lang.ref.WeakReference@5a07e868
	null

## 5, PhantomReference

虚引用是引用中最弱的引用类型，有些形同虚设的意味。不同于软引用和弱引用，虚引用不会影响对象的生命周期，如果一个对象仅持有虚引用，那么它就相当于无引用指向，不可达，被gc扫描到就会被回收，虚引用无法通过get()方法来获取目标对象的强引用从而使用目标对象，虚引用中get()方法永远返回null。

​ 虚引用必须和引用队列(ReferenceQueue)联合使用，当gc回收一个被虚引用指向的对象时，会将虚引用加入相关联的引用队列中。虚引用主要用于追踪对象gc回收的活动，通过查看引用队列中是否包含对象所对应的虚引用来判断它是否即将被回收。

​ 虚引用的一个应用场景是用来追踪gc回收对应对象的活动。
	
	public PhantomReference(T referent, ReferenceQueue<? super T> q) // 创建弱引用

示例：

	public class PhantomReferenceTest {
	
	    public static void main(String[] args) {
	        ReferenceQueue<String> rq = new ReferenceQueue<>();
	        PhantomReference<String> reference = new PhantomReference<>(new String("cord"), rq);
	        System.out.println(reference.get());
	        System.gc();
	        System.runFinalization();
	        System.out.println(rq.poll() == reference);
	    }
	}
	
运行结果：
	
	null
	true


## 6, ReferenceQueue

ReferenceQueue内部数据结构是一个链表，链表里的元素是加入进去的Reference实例，然后通过wait和notifyAll与对象锁实现生产者和消费者，通过这种方式模拟一个队列。

ReferenceQueue是使用wati()和notifyAll()实现生产者和消费者模式的一个具体场景。

ReferenceQueue重点源码解析：

-  NULL和ENQUEUED

	    static ReferenceQueue<Object> NULL = new Null<>();
	    static ReferenceQueue<Object> ENQUEUED = new Null<>();

这两个静态属性主要用于标识加入引用队列的引用的状态，NULL标识该引用已被当前队列移除过，ENQUEUED标识该引用已加入当前队列。

-  enqueue(Reference<? extends T> r)

	 boolean enqueue(Reference<? extends T> r) { /* Called only by Reference class */
	        synchronized (lock) {
	            //检查该引用是否曾从当前队列移除过或者已经加入当前队列了，如果有则直接返回
	            ReferenceQueue<?> queue = r.queue;
	            if ((queue == NULL) || (queue == ENQUEUED)) {
	                return false;
	            }
	            assert queue == this;
	            r.queue = ENQUEUED;//将引用关联的队列统一标识为ENQUEUED
	            r.next = (head == null) ? r : head;//当前引用指向head
	            head = r; //将head指向当前引用(链表新增节点采用头插法)
	            queueLength++; //更新链表长度
	            if (r instanceof FinalReference) {
	                sun.misc.VM.addFinalRefCount(1); //
	            }
	            lock.notifyAll(); //通知消费端
	            return true;
	        }
	    }
	    
-   remove(long timeout)

remove尝试移除队列中的头部元素，如果队列为空则一直等待直至达到指定的超时时间。

	 public Reference<? extends T> remove(long timeout)
	        throws IllegalArgumentException, InterruptedException
	    {
	        if (timeout < 0) {
	            throw new IllegalArgumentException("Negative timeout value");
	        }
	        synchronized (lock) {
	            Reference<? extends T> r = reallyPoll();
	            if (r != null) return r; //如果成功移除则直接返回
	            long start = (timeout == 0) ? 0 : System.nanoTime();
	            for (;;) {
	                lock.wait(timeout); //释放当前线程锁，等待notify通知唤醒
	                r = reallyPoll();
	                if (r != null) return r;
	                if (timeout != 0) {   //如果超时时间不为0则校验超时
	                    long end = System.nanoTime();
	                    timeout -= (end - start) / 1000_000;
	                    if (timeout <= 0) return null;  //如果剩余时间小于0则返回
	                    start = end;
	                }
	            }
	        }
	    }

## 7，Cleaner

​ Cleaner是PhantomReference的一个子类实现，提供了比finalization(收尾机制)更轻量级和健壮的实现，因为Cleaner中的清理逻辑是由Reference.ReferenceHandler 直接调用的，而且由于是虚引用的子类，它完全不会影响指向的对象的生命周期。

​ 一个Cleaner实例记录了一个对象的引用，以及一个包含了清理逻辑的Runnable实例。当Cleaner指向的引用被gc回收后，Reference.ReferenceHandler会不断消费引用队列中的元素，当元素为Cleaner类型的时候就会调用其clean()方法。

​ Cleaner不是用来替代finalization的，只有在清理逻辑足够轻量和直接的时候才适合使用Cleaner，繁琐耗时的清理逻辑将有可能导致ReferenceHandler线程阻塞从而耽误其它的清理任务。

重点源码解析：

	public class Cleaner extends PhantomReference<Object>
	{
	    //一个统一的空队列，用于虚引用构造方法，Cleaner的trunk会被直接调用不需要通过队列
	    private static final ReferenceQueue<Object> dummyQueue = new ReferenceQueue<>();
	
	    //Cleaner内部为双向链表,防止虚引用本身比它们引用的对象先被gc回收,此为头节点
	    static private Cleaner first = null;
	
	    //添加节点
	    private static synchronized Cleaner add(Cleaner cl) {
	        if (first != null) {    //头插法加入节点
	            cl.next = first;
	            first.prev = cl;
	        }
	        first = cl;
	        return cl;
	    }
	    //移除节点
	    private static synchronized boolean remove(Cleaner cl) {
	
	        //指向自己说明已经被移除
	        if (cl.next == cl)
	            return false;
	
	        //移除头部节点
	        if (first == cl) {
	            if (cl.next != null)
	                first = cl.next;
	            else
	                first = cl.prev;
	        }
	        if (cl.next != null)//下一个节点指向前一个节点
	            cl.next.prev = cl.prev;
	        if (cl.prev != null)//前一个节点指向下一个节点
	            cl.prev.next = cl.next;
	
	        //自己指向自己标识已被移除
	        cl.next = cl;
	        cl.prev = cl;
	        return true;
	
	    }
	
	    //清理逻辑runnable实现
	    private final Runnable thunk;
	
	    ...
	
	    //调用清理逻辑
	    public void clean() {
	        if (!remove(this))
	            return;
	        try {
	            thunk.run();
	        } catch (final Throwable x) {
	            ...
	        }
	    }
	}


Cleaner可以用来实现对堆外内存进行管理，DirectByteBuffer就是通过Cleaner实现堆外内存回收的:

	    DirectByteBuffer(int cap) { //构造方法中创建引用对象相关联的Cleaner对象                 
	        ...
	        cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
	        att = null;
	    }
	    
	    private static class Deallocator implements Runnable {
	        ...
	        public void run() { //内存回收的逻辑(具体实现参看源码此处不展开)
	        ...
	        }
	
	    }     
  
 
## 8, Reference

Reference是上面列举的几种引用包括Cleaner的共同父类，一些引用的通用处理逻辑均在这里面实现。

### 引用实例的几个状态
 
-  Active

当处于Active状态，gc会特殊处理引用实例，一旦gc检测到其可达性发生变化，gc就会更改其状态。此时分两种情况，如果该引用实例创建时有注册引用队列，则会进入pending状态，否则会进入inactive状态。新创建的引用实例为Active。



-  Pending

当前为pending-Reference列表中的一个元素，等待被ReferenceHandler线程消费并加入其注册的引用队列。如果该引用实例未注册引用队列，则永远不会处理这个状态。

-  Enqueued

该引用实例创建时有注册引用队列并且当前处于入队列状态，属于该引用队列中的一个元素。当该引用实例从其注册引用队列中移除后其状态变为Inactive。如果该引用实例未注册引用队列，则永远不会处理这个状态。

-  Inactive

当处于Inactive状态，无需任何处理，一旦变成Inactive状态则其状态永远不会再发生改变。

整体迁移流程图如下：

![reference关系](https://kunge2013.go123.live/images/frame/jdk/reference/引用实例生命周期.png)


### 重点源码解析

-  1，Reference中的几个关键属性

	 //关联的对象的引用,根据引用类型不同gc针对性处理
	    private T referent;       
	    //引用注册的队列,如果有注册队列则回收引用会加入该队列
	    volatile ReferenceQueue<? super T> queue;
	
	    //上面引用队列referenceQueue中保存引用的链表
	    /*    active:     NULL //未加入队列前next指向null
	     *    pending:    this
	     *    Enqueued:   next reference in queue (or this if last)
	     *    Inactive:   this
	     */
	    Reference next;
	
	
	    /* When active:   由gc管理的引用发现链表的下一个引用
	     *     pending:   pending链表中的下一个元素
	     *   otherwise:   NULL
	     */
	    transient private Reference<T> discovered;  /* used by VM */
	
	    /* 
	     *等待入队列的引用链表，gc往该链表加引用对象，Reference-handler线程消费该链表。
	     * 它通过discovered连接它的元素 
	     */     
	    private static Reference<Object> pending = null;
	    
	    
-   2，ReferenceHandler

		private static class ReferenceHandler extends Thread {
				...
		        public void run() {
		            while (true) {
		                tryHandlePending(true); //无限循环调用tryHandlePending
		            }
		        }
		    }
		    static {
				... jvm启动时以守护线程运行ReferenceHandler
		        Thread handler = new ReferenceHandler(tg, "Reference Handler");
		        handler.setPriority(Thread.MAX_PRIORITY);
		        handler.setDaemon(true);
		        handler.start();
		        //注册JavaLangRefAccess匿名实现,堆外内存管理会用到(Bits.reserveMemory)
		        SharedSecrets.setJavaLangRefAccess(new JavaLangRefAccess() {
		            @Override
		            public boolean tryHandlePendingReference() {
		                return tryHandlePending(false);
		            }
		        });
		    }

		 //消费pending队列
		    static boolean tryHandlePending(boolean waitForNotify) {
		        Reference<Object> r;
		        Cleaner c;
		        try {
		            synchronized (lock) {
		                if (pending != null) {
		                    r = pending;
		                    // 'instanceof' might throw OutOfMemoryError sometimes
		                    // so do this before un-linking 'r' from the 'pending' chain...
		                    //判断是否为Cleaner实例
		                    c = r instanceof Cleaner ? (Cleaner) r : null;
		                   //将r从pending链表移除
		                    pending = r.discovered;
		                    r.discovered = null;
		                } else {
		                    // The waiting on the lock may cause an OutOfMemoryError
		                    // because it may try to allocate exception objects.
		                    //如果pending没有元素可消费则等待通知
		                    if (waitForNotify) {
		                        lock.wait();
		                    }
		                    // retry if waited
		                    return waitForNotify;
		                }
		            }
		        } catch (OutOfMemoryError x) {
		            //释放cpu资源
		            Thread.yield();
		            // retry
		            return true;
		        } catch (InterruptedException x) {
		            // retry
		            return true;
		        }
		
		        //调用Cleaner清理逻辑(可参考前面的7，Cleaner段落)
		        if (c != null) {
		            c.clean();
		            return true;
		        }
		        //如果当前引用实例有注册引用队列则将其加入引用队列
		        ReferenceQueue<? super Object> q = r.queue;
		        if (q != ReferenceQueue.NULL) q.enqueue(r);
		        return true;
		    }
	 
 
### 总结

​ jvm中引用有好几种类型的实现，gc针对这几种不同类型的引用有着不同的回收机制，同时它们也有着各自的应用场景, 比如SoftReference可以用来做高速缓存, WeakReference也可以用来做一些普通缓存(WeakHashMap), 而PhantomReference则用在一些特殊场景，比如Cleaner就是一个很好的应用场景，它可以用来回收堆外内存。与此同时，SoftReference, WeakReference, PhantomReference这几种弱类型引用还可以与引用队列结合使用，使得可以在关联引用回收之后可以做一些额外处理，甚至于Finalizer(收尾机制)都可以在对象回收过程中改变对象的生命周期。

参考链接：

https://www.ibm.com/developerworks/cn/java/j-fv/index.html

https://www.infoq.cn/article/jvm-source-code-analysis-finalreference

https://www.ibm.com/developerworks/cn/java/j-lo-langref/index.html

https://www.cnblogs.com/duanxz/p/10275778.html

《深入理解java虚拟机》

https://blog.csdn.net/mazhimazh/article/details/19752475

https://www.tuicool.com/articles/AZ7Fvqb

https://blog.csdn.net/aitangyong/article/details/39455229

https://www.cnblogs.com/duanxz/p/6089485.html

https://www.throwable.club/2019/02/16/java-reference/#Reference的状态集合

http://imushan.com/2018/08/19/java/language/JDK%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB-Reference/


 