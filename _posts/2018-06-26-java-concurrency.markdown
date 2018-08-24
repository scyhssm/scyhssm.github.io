---
layout:    post
title:      "Java高并发"
subtitle:   "Java Concurrency"
date:       2018-06-26 12:00:00
author:     "scyhssm"
header-img: "img/Java.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Java
---

> 记录Java高并发的笔记

1.线程状态转换图
￼![ThreadStatus](/img/Thread-Status.png)

2.线程的几个状态

新建：线程被创建，但是没有启动

可运行：可能正在运行也可能在等待时间片，Running和Ready两种状态

阻塞：等待获取一个排他锁解除阻塞状态，等待拥有此锁的线程释放该锁，Blocking

无限期等待：等待其他线程显式唤醒，否则不会被分配CPU时间片，wait列表

进入方法|退出方法
----|---
没有设置 Timeout 参数的 Object.wait() 方法|Object.notify() / Object.notifyAll()
没有设置 Timeout 参数的 Thread.join() 方法|被调用的线程执行完毕
LockSupport.park() 方法|-

限期等待：Timed Waiting，无需等待其他线程显式唤醒，在一定时间后会被系统自动唤醒。调用Thread.sleep()方法使线程进入期限等待，使线程睡眠。Object.wait使线程进入期限等待或者无限期等待，挂起一个线程。睡眠和挂起用来描述行为，阻塞和等待用来描述状态。睡眠是Sleep自动恢复，挂起是wait需要主动唤醒。阻塞和等待区别是阻塞被动，而等待（也是阻塞）是主动的。

进入方法|退出方法
----|----
Thread.sleep() 方法|时间结束
设置了 Timeout 参数的 Object.wait() 方法|时间结束 / Object.notify() / Object.notifyAll()
设置了 Timeout 参数的 Thread.join() 方法|时间结束 / 被调用的线程执行完毕
LockSupport.parkNanos() 方法|-
LockSupport.parkUntil() 方法|-

死亡：线程结束任务后结束或者由于各种异常结束。
￼
3.有三种使用线程的方法，Runnable接口，Callable接口，继承Thread类。实际上只有Thread类才是真正创建了线程，Runnable和Callable只是在类中实现了能在线程中运行的任务，还是需要线程驱动运行。

4.Runnable实现run方法，通过Thread的start来启动线程。

5.Callable相比Runnable可以有返回值，返回值通过FutureTask封装。
```
public class MyCallable implements Callable<Integer> {
    public Integer call() {
        return 123;
    }
}
public static void main(String[] args) throws ExecutionException, InterruptedException {
    MyCallable mc = new MyCallable();
    FutureTask<Integer> ft = new FutureTask<>(mc);
    Thread thread = new Thread(ft);
    thread.start();
    System.out.println(ft.get());
}
```

6.继承Thread类继承重写run方法，直接通过新建实例start方法启动线程。

7.实现接口和继承Thread对比
* Java不支持多重继承，继承了Thread类无法继承其他类，但是可以实现多个接口
* 类只要求可执行，继承整个Thread类开销太大

8.Executor可以管理多个异步任务的执行，无需程序员显式管理线程生命周期。三种Executor，CachedThreadPool，一个任务创建一个线程，FixedThreadPool，所有任务使用固定大小的线程，SingleThreadExecutor大小为1的FixedThreadPool。

9.Daemon是程序运行时在后台提供服务的线程，setDaemon方法将线程设置为守护线程。

10.sleep方法休眠当前线程，可能会抛出InterruptedException，异常不能传回main，必须在本地处理，即Thread中规定的run方法并没有抛出异常，所以run中不能抛出异常，只能在run方法内处理异常。

11.Thread.yield声明了当前线程完成了最重要的部分可以切换给其它线程执行，可以让出时间片。线程调度器可以选择继续执行当前线程或者是切换到其它线程运行。

12.线程中断
* 通过调用线程的interrupt中断线程，如果线程处于阻塞、限期等待或者是无限期等待就会抛出InterruptedException，提前结束线程。不能中断I/O阻塞和synchronized锁阻塞。
* Run方法是无限循环就不会抛出InterruptedException，interrupt无法提前结束。interrupt方法会设置线程的中断标记，调用interrupted方法会返回true。在循环体中使用interrupted方法判断线程是否处于中断状态，是的话结束循环体，结束线程，见代码：
```
public class InterruptExample {

    private static class MyThread2 extends Thread {
        @Override
        public void run() {
            while (!interrupted()) {
                // ..
            }
            System.out.println("Thread end");
        }
    }
}
public static void main(String[] args) throws InterruptedException {
    Thread thread2 = new MyThread2();
    thread2.start();
    thread2.interrupt();
}
```
* Executor中断，shutdown会等待线程都执行完毕后关闭，如果是shutdownNow相当于调用每个线程的interrupt方法。也可以只中断一个线程，通过submit方法提交线程，他会返回Future对象，通过这个对象的cancel方法中断线程。

13.Java两种锁机制控制线程对共享资源的互斥访问，一个是JVM实现的synchronized，一个是JDK实现的ReentrantLock。

14.synchronized可以同步一个对象 synchronized(this)、同步一个类 synchronized(XXX.class)、同步一个方法(作用于this对象) public synchronized void func ()、同步一个静态方法(作用与XXX.class) public synchronized static void fun()

15.ReentrantLock是 java.util.concurrent（J.U.C）包中的锁.
```
public class LockExample {

    private Lock lock = new ReentrantLock();

    public void func() {
        lock.lock();
        try {
            for (int i = 0; i < 10; i++) {
                System.out.print(i + " ");
            }
        } finally {
            lock.unlock(); // 确保释放锁，从而避免发生死锁。
        }
    }
}
```
16.对synchronized和ReentrantLock比较
* synchronized是JVM实现，ReentrantLock是JDK实现
* 新版本Java对synchronized进行很多优化，如自旋锁，synchronized和ReentrantLock大致效率相同。
* ReentrantLock可以在持有锁的线程长期不释放锁的时候中断处理其它事情，而synchronized不行
* 多个线程等待同一个锁，必须按照申请锁的时间顺序依次获得锁，synchronized锁非公平，ReentrantLock非公平，但是可以是公平的
* 一个ReentrantLock可以同时绑定多个Condition对象
一般情况下优先使用synchronized，synchronized是JVM实现的一种锁机制，JVM原生支持，ReentrantLock不是所有JDK都支持。JVM会确保锁释放，ReentrantLock要担心没有释放锁导致的死锁。

17.join方法会挂起当前线程，直到目标线程结束，即令目标线程先运行。

18.wait使线程等待某个条件满足，如果条件满足其他线程会调用notify或者notifyAll唤醒挂起的线程，wait、notify、notifyAll都属于Object一部分，而不属于Thread。如果使用wait挂起线程，线程会释放锁，如果没有释放锁，其他线程无法进入对象的同步方法或者同步控制块，无法执行notify或者notifyAll唤醒线程。
```
public class WaitNotifyExample {
    public synchronized void before() {
        System.out.println("before");
        notifyAll();
    }

    public synchronized void after() {
        try {
            wait();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("after");
    }
}
```
19.wait是Object方法，会释放锁，而sleep是Thread的静态方法，不会释放锁

20.java.util.concurrent类库中提供了Condition实现线程间的协调。创建了ReentrantLock对象，在ReentrantLock中创建Condition，Condition 将 Object 监视器方法（wait、notify 和 notifyAll）分解成截然不同的对象，以便通过将这些对象与任意 Lock 实现组合使用，为每个对象提供多个等待 set （wait-set）。可以在Condition上调用await方法使线程等待，其他线程调用signal或signalAll方法唤醒等待线程。想比如wait，await可以指定等待条件，更灵活。
```
public class AwaitSignalExample {
    private Lock lock = new ReentrantLock();
    private Condition condition = lock.newCondition();

    public void before() {
        lock.lock();
        try {
            System.out.println("before");
            condition.signalAll();
        } finally {
            lock.unlock();
        }
    }

    public void after() {
        lock.lock();
        try {
            condition.await();
            System.out.println("after");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
```
21.java.util.concurrent（J.U.C）大大提高了并发性能，AQS（AbstractQueuedSynchronizer） 被认为是 J.U.C 的核心。

22.J.U.C AQS组件，CountdownLatch，用来控制一个线程等待多个线程。维护了一个计数器 cnt，每次调用 countDown() 方法会让计数器的值减 1，减到 0 的时候，那些因为调用 await() 方法而在等待的线程就会被唤醒。
```
public class CountdownLatchExample {

    public static void main(String[] args) throws InterruptedException {
        final int totalThread = 10;
        CountDownLatch countDownLatch = new CountDownLatch(totalThread);
        ExecutorService executorService = Executors.newCachedThreadPool();
        for (int i = 0; i < totalThread; i++) {
            executorService.execute(() -> {
                System.out.print("run..");
                countDownLatch.countDown();
            });
        }
        countDownLatch.await();
        System.out.println("end");
        executorService.shutdown();
    }
}
```
23.J.U.C AQS组件， CyclicBarrier，用来控制多个线程互相等待，只有当多个线程都到达时，这些线程才会继续执行。和 CountdownLatch 相似，都是通过维护计数器来实现的。但是它的计数器是递增的，每次执行 await() 方法之后，计数器会加 1，直到计数器的值和设置的值相等，等待的所有线程才会继续执行。和 CountdownLatch 的另一个区别是，CyclicBarrier 的计数器可以循环使用，所以它才叫做循环屏障。
```
public class CyclicBarrierExample {
    public static void main(String[] args) throws InterruptedException {
        final int totalThread = 10;
        CyclicBarrier cyclicBarrier = new CyclicBarrier(totalThread);
        ExecutorService executorService = Executors.newCachedThreadPool();
        for (int i = 0; i < totalThread; i++) {
            executorService.execute(() -> {
                System.out.print("before..");
                try {
                    cyclicBarrier.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
                System.out.print("after..");
            });
        }
        executorService.shutdown();
    }
}
```
24.J.U.C AQS组件，Semaphore，Semaphore 就是操作系统中的信号量，可以控制对互斥资源的访问线程数。

以下代码模拟了对某个服务的并发请求，每次只能有 3 个客户端同时访问，请求总数为 10：
```
public class SemaphoreExample {
    public static void main(String[] args) {
        final int clientCount = 3;
        final int totalRequestCount = 10;
        Semaphore semaphore = new Semaphore(clientCount);
        ExecutorService executorService = Executors.newCachedThreadPool();
        for (int i = 0; i < totalRequestCount; i++) {
            executorService.execute(()->{
                try {
                    semaphore.acquire();
                    System.out.print(semaphore.availablePermits() + " ");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    semaphore.release();
                }
            });
        }
        executorService.shutdown();
    }
}
```
25.J.U.C 其他组件，FutureTask，Callable有返回值，返回值通过 Future 进行封装。FutureTask 实现了 RunnableFuture 接口，该接口继承自 Runnable 和 Future 接口，这使得 FutureTask 既可以当做一个任务执行，也可以有返回值。
```
public class FutureTask<V> implements RunnableFuture<V>
public interface RunnableFuture<V> extends Runnable, Future<V>
```
FutureTask 可用于异步获取执行结果或取消执行任务的场景。当一个计算任务需要执行很长时间，那么就可以用 FutureTask 来封装这个任务，主线程在完成自己的任务之后再去获取结果。
```
public class FutureTaskExample {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        FutureTask<Integer> futureTask = new FutureTask<Integer>(new Callable<Integer>() {
            @Override
            public Integer call() throws Exception {
                int result = 0;
                for (int i = 0; i < 100; i++) {
                    Thread.sleep(10);
                    result += i;
                }
                return result;
            }
        });

        Thread computeThread = new Thread(futureTask);
        computeThread.start();

        Thread otherThread = new Thread(() -> {
            System.out.println("other task is running...");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        otherThread.start();
        System.out.println(futureTask.get());  //futureTask.get会阻塞main方法，直到get到数据
    }
}
```
26.J.U.C 其他组件，java.util.concurrent.BlockingQueue 接口有以下阻塞队列的实现：
* FIFO 队列 ：LinkedBlockingQueue、ArrayBlockingQueue（固定长度）
* 优先级队列 ：PriorityBlockingQueue
用ArrayBlockingQueue实现生产者消费者
```
public class ProducerConsumer {

    private static BlockingQueue<String> queue = new ArrayBlockingQueue<>(5);

    private static class Producer extends Thread {
        @Override
        public void run() {
            try {
                queue.put("product");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.print("produce..");
        }
    }

    private static class Consumer extends Thread {

        @Override
        public void run() {
            try {
                String product = queue.take();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.print("consume..");
        }
    }
}
```
提供了阻塞的take和put方法：如果队列为空take阻塞，直到队列中有内容；如果队列为满put将阻塞，直到队列有空闲。

27.J.U.C 其他组件，ForkJoin，主要用于并行计算中，和 MapReduce 原理类似，都是把大的计算任务拆分成多个小任务并行计算。
```
public class ForkJoinExample extends RecursiveTask<Integer> {
    private final int threhold = 5;
    private int first;
    private int last;

    public ForkJoinExample(int first, int last) {
        this.first = first;
        this.last = last;
    }

    @Override
    protected Integer compute() {
        int result = 0;
        if (last - first <= threhold) {
            // 任务足够小则直接计算
            for (int i = first; i <= last; i++) {
                result += i;
            }
        } else {
            // 拆分成小任务
            int middle = first + (last - first) / 2;
            ForkJoinExample leftTask = new ForkJoinExample(first, middle);
            ForkJoinExample rightTask = new ForkJoinExample(middle + 1, last);
            leftTask.fork();
            rightTask.fork();
            result = leftTask.join() + rightTask.join();
        }
        return result;
    }
}
public static void main(String[] args) throws ExecutionException, InterruptedException {
    ForkJoinExample example = new ForkJoinExample(1, 10000);
    ForkJoinPool forkJoinPool = new ForkJoinPool();
    Future result = forkJoinPool.submit(example);
    System.out.println(result.get());
}
```
ForkJoin 使用 ForkJoinPool 来启动，它是一个特殊的线程池，线程数量取决于 CPU 核数。
```
public class ForkJoinPool extends AbstractExecutorService
```
ForkJoinPool 实现了工作窃取算法来提高 CPU 的利用率。每个线程都维护了一个双端队列，用来存储需要执行的任务。工作窃取算法允许空闲的线程从其它线程的双端队列中窃取一个任务来执行。窃取的任务必须是最晚的任务，避免和队列所属线程发生竞争。例如下图中，Thread2 从 Thread1 的队列中拿出最晚的 Task1 任务，Thread1 会拿出 Task2 来执行，这样就避免发生竞争。但是如果队列中只有一个任务时还是会发生竞争。
![ForkJoinPool](/img/fork-join-pool.jpg)
￼
28.Java内存模型试图屏蔽各种硬件和操作系统的内存访问差异，以实现让Java程序在各种平台下达到一致的内存访问效果。现在计算机，cpu在计算时并不总是从内存读取数据，读取数据优先级：寄存器-高速缓存-内存。如果多个缓存共享同一块住内存区域，多个缓存的数据可能会不一致。
￼
29.所有的变量都存储在主内存中，每个线程还有自己的工作内存，工作内存存储在高速缓存或者寄存器中，保存了该线程使用的变量的主内存副本拷贝。线程只能直接操作工作内存中的变量，不同线程之间的变量值传递需要通过主内存来完成。
![JavaMemory](/img/java-memory-model.png)
￼
30.Java内存模型的8个操作：
![JavaMemoryOperation](/img/java-memory-eight-operation.png)
* Read:把变量的值从住内存传输到工作内存
* Load：在read后执行，把read得到的值放入工作内存的变量副本中
* Use：把工作内存中一个变量的值传递给执行引擎
* Assign：把一个从执行引擎接收到的值赋给工作内存的变量
* Store：把工作内存的一个变量传到住内存中
* Write：在store后执行，把store得到的值放入住内存的变量中
* Lock：作用于主内存的变量
* Unlock：作用于主内存的变量，解锁变量

31.原子性

Java 内存模型保证了 read、load、use、assign、store、write、lock 和 unlock 操作具有原子性，例如对一个 int 类型的变量执行 assign 赋值操作，这个操作就是原子性的。但是 Java 内存模型允许虚拟机将没有被 volatile 修饰的 64 位数据（long，double）的读写操作划分为两次 32 位的操作来进行，即 load、store、read 和 write 操作可以不具备原子性。好像在最新的JDK中，JVM已经保证对64位数据的读取和赋值也是原子性操作了。

有一个错误认识就是，int 等原子性的变量在多线程环境中不会出现线程安全问题。前面的线程不安全示例代码中，cnt 变量属于 int 类型变量，1000 个线程对它进行自增操作之后，得到的值为 997 而不是 1000。

为了方便讨论，将内存间的交互操作简化为 3 个：load、assign、store。

下图演示了两个线程同时对 cnt 变量进行操作，load、assign、store 这一系列操作整体上看不具备原子性，那么在 T1 修改 cnt 并且还没有将修改后的值写入主内存，T2 依然可以读入该变量的值。可以看出，这两个线程虽然执行了两次自增运算，但是主内存中 cnt 的值最后为 1 而不是 2。因此对 int 类型读写操作满足原子性只是说明 load、assign、store 这些单个操作具备原子性。
￼
![WrongPlus](/img/wrong-plus-plus.png)
AtomicInteger 能保证多个线程修改的原子性。

![AtomicInteger](/img/atomicInteger.png)￼

除了使用原子类之外，也可以使用 synchronized 互斥锁来保证操作的完整性，它对应的内存间交互操作为：lock 和 unlock，在虚拟机实现上对应的字节码指令为 monitorenter 和 monitorexit。

32.可见性

可见性指当一个线程修改了共享变量的值，其它线程能够立即得知这个修改。Java 内存模型是通过在变量修改后将新值同步回主内存，在变量读取前从主内存刷新变量值来实现可见性的。

volatile 可保证可见性。synchronized 也能够保证可见性，对一个变量执行 unlock 操作之前，必须把变量值同步回主内存。final 关键字也能保证可见性：被 final 关键字修饰的字段在构造器中一旦初始化完成，并且没有发生 this 逃逸（其它线程可以通过 this 引用访问到初始化了一半的对象），那么其它线程就能看见 final 字段的值。

对前面的线程不安全示例中的 cnt 变量用 volatile 修饰，不能解决线程不安全问题，因为 volatile 并不能保证操作的原子性。

33.缓存一致性协议。最出名的就是Intel 的MESI协议，MESI协议保证了每个缓存中使用的共享变量的副本是一致的。它核心的思想是：当CPU写数据时，如果发现操作的变量是共享变量，即在其他CPU中也存在该变量的副本，会发出信号通知其他CPU将该变量的缓存行置为无效状态，因此当其他CPU需要读取这个变量时，发现自己缓存中缓存该变量的缓存行是无效的，那么它就会从内存重新读取。

34.有序性是指：在本线程内观察，所有操作都是有序的。在一个线程观察另一个线程，所有操作都是无序的，无序是因为发生了指令重排序。

在 Java 内存模型中，允许编译器和处理器对指令进行重排序，重排序过程不会影响到单线程程序的执行，却会影响到多线程并发执行的正确性。

volatile 关键字通过添加内存屏障的方式来禁止指令重排，即重排序时不能把后面的指令放到内存屏障之前。

也可以通过 synchronized 来保证有序性，它保证每个时刻只有一个线程执行同步代码，相当于是让线程顺序执行同步代码。

35.volatile修饰变量：
* 使用volatile关键字会强制将修改的值立即写入主存；
* 使用volatile关键字的话，当线程2进行修改时，会导致线程1的工作内存中缓存变量stop的缓存行无效（反映到硬件层的话，就是CPU的L1或者L2缓存中对应的缓存行无效）；
* 由于线程1的工作内存中缓存变量stop的缓存行无效，所以线程1再次读取变量stop的值时会去主存读取。
在线程2修改stop值时（当然这里包括2个操作，修改线程2工作内存中的值，然后将修改后的值写入内存），会使得线程1的工作内存中缓存变量stop的缓存行无效，然后线程1读取时，发现自己的缓存行无效，它会等待缓存行对应的主存地址被更新之后，然后去对应的主存读取最新的值。线程1读取到的就是最新的正确的值。

Volatile可以保证原子性吗？例子：
```
public class Test {
    public volatile int inc = 0;
     
    public void increase() {
        inc++;
    }
     
    public static void main(String[] args) {
        final Test test = new Test();
        for(int i=0;i<10;i++){
            new Thread(){
                public void run() {
                    for(int j=0;j<1000;j++)
                        test.increase();
                };
            }.start();
        }
         
        while(Thread.activeCount()>1)  //保证前面的线程都执行完
            Thread.yield();
        System.out.println(test.inc);
    }
}
```
实际上上面结果每次运行都不一致，是一个小于10000的数字。volatile能保证可见行，但是不能保证原子性，也就是将一个自增操作拆分成三个原子操作，load，assign，store，假如说运行顺序是下图
￼![WrongPlus](/img/wrong-plus-plus.png)

T1的修改在T2读取之后进行，不会使T2的缓存行无效（读取时还有效，而后面操作可以一直使用，不会因为T1后面再修改无效而失效，失效只是针对在读取的时候发现失效从主存中读取）。最后inc值只增加了1.
Volatile保证有序性，禁止指令重排序的意思：
* 当执行到volatile变量的读或者写，其前面操作的更改已经全部进行，结果对后面操作可见，后面的操作还没有进行
* 指令优化不能将对volatile变量访问的语句放后面执行，不能把volatile变量后面的语句放到前面执行
```
//x、y为非volatile变量
//flag为volatile变量
 
x = 2;        //语句1
y = 0;        //语句2
flag = true;  //语句3
x = 4;         //语句4
y = -1;       //语句5
```
由于flag变量为volatile变量，那么在进行指令重排序的过程的时候，不会将语句3放到语句1、语句2前面，也不会将语句3放到语句4、语句5后面。但是要注意语句1和语句2的顺序、语句4和语句5的顺序是不作任何保证的。

并且volatile关键字能保证，执行到语句3时，语句1和语句2必定是执行完毕了的，且语句1和语句2的执行结果对语句3、语句4、语句5是可见的。
```
//线程1:
context = loadContext();   //语句1
inited = true;             //语句2
 
//线程2:
while(!inited ){
  sleep()
}
doSomethingwithconfig(context);
```

提到有可能语句2会在语句1之前执行，那么久可能导致context还没被初始化，而线程2中就使用未初始化的context去进行操作，导致程序出错。这里如果用volatile关键字对inited变量进行修饰，就不会出现这种问题了，因为当执行到语句2时，必定能保证context已经初始化完毕。

观察加入volatile关键字和没有加入volatile关键字时所生成的汇编代码发现，加入volatile关键字时，会多出一个lock前缀指令。lock前缀指令实际上相当于一个内存屏障（也成内存栅栏），内存屏障（内存栅栏）会提供3个功能：
* 它确保指令重排序时不会把其后面的指令排到内存屏障之前的位置，也不会把前面的指令排到内存屏障的后面；即在执行到内存屏障这句指令时，在它前面的操作已经全部完成；
* 它会强制将对缓存的修改操作立即写入主存；
* 如果是写操作，它会导致其他CPU中对应的缓存行无效。

36.先行发生原则

单一线程原则，在一个线程内，在程序前面的操作先行发生于后面的操作。

管程锁定规则，一个 unlock 操作先行发生于后面对同一个锁的 lock 操作。

volatile 变量规则，对一个 volatile 变量的写操作先行发生于后面对这个变量的读操作。

线程启动规则，Thread 对象的 start() 方法调用先行发生于此线程的每一个动作。

线程加入规则，Thread 对象的结束先行发生于 join() 方法返回。

线程中断规则，对线程 interrupt() 方法的调用先行发生于被中断线程的代码检测到中断事件的发生，可以通过 interrupted() 方法检测到是否有中断发生。

对象终结规则，一个对象的初始化完成（构造函数执行结束）先行发生于它的 finalize() 方法的开始。

传递性，如果操作 A 先行发生于操作 B，操作 B 先行发生于操作 C，那么操作 A 先行发生于操作 C。

37.线程安全，一个类在可以被多个线程安全调用时就是线程安全的。将共享数据按照安全程度强弱分成：不可变、绝对线程安全、相对线程安全、线程兼容和线程对立。

38.不可变Immutable）的对象一定是线程安全的，无论是对象的方法实现还是方法的调用者，都不需要再采取任何的线程安全保障措施，只要一个不可变的对象被正确地构建出来，那其外部的可见状态永远也不会改变，永远也不会看到它在多个线程之中处于不一致的状态。不可变的类型：
* Final关键字修饰的基本数据类型
* String
* 枚举类型
* Number 部分子类，如 Long 和 Double 等数值包装类型，BigInteger 和 BigDecimal 等大数据类型。但同为 Number 的子类型的原子类 AtomicInteger 和 AtomicLong 则并非不可变的。
* 对于集合类型，可以使用 Collections.unmodifiableXXX() 方法来获取一个不可变的集合。Collections.unmodifiableXXX() 先对原始的集合进行拷贝，需要对集合进行修改的方法都直接抛出异常。

多线程环境下，应当尽量使对象成为不可变，来满足线程安全。

39.绝对线程安全，不管运行时环境如何，调用者都不需要任何额外的同步措施。

40.相对线程安全，相对的线程安全需要保证对这个对象单独的操作是线程安全的，在调用的时候不需要做额外的保障措施，但是对于一些特定顺序的连续调用，就可能需要在调用端使用额外的同步手段来保证调用的正确性。

Java 语言中，大部分的线程安全类都属于这种类型，例如 Vector、HashTable、Collections 的 synchronizedCollection() 方法包装的集合等。

对于下面的代码，如果删除元素的线程删除了一个元素，而获取元素的线程试图访问一个已经被删除的元素，那么就会抛出 ArrayIndexOutOfBoundsException。
```
public class VectorUnsafeExample {
    private static Vector<Integer> vector = new Vector<>();

    public static void main(String[] args) {
        while (true) {
            for (int i = 0; i < 100; i++) {
                vector.add(i);
            }
            ExecutorService executorService = Executors.newCachedThreadPool();
            executorService.execute(() -> {
                for (int i = 0; i < vector.size(); i++) {
                    vector.remove(i);
                }
            });
            executorService.execute(() -> {
                for (int i = 0; i < vector.size(); i++) {
                    vector.get(i);
                }
            });
            executorService.shutdown();
        }
    }
}
```
如果要保证上面的代码能正确执行下去，就需要对删除元素和获取元素的代码进行同步。
```
executorService.execute(() -> {
    synchronized (vector) {
        for (int i = 0; i < vector.size(); i++) {
            vector.remove(i);
        }
    }
});
executorService.execute(() -> {
    synchronized (vector) {
        for (int i = 0; i < vector.size(); i++) {
            vector.get(i);
        }
    }
});
```

41.线程兼容，线程兼容是指对象本身并不是线程安全的，但是可以通过在调用端正确地使用同步手段来保证对象在并发环境中可以安全地使用，我们平常说一个类不是线程安全的，绝大多数时候指的是这一种情况。Java API 中大部分的类都是属于线程兼容的，如与前面的 Vector 和 HashTable 相对应的集合类 ArrayList 和 HashMap 等。

42.线程对立，线程对立是指无论调用端是否采取了同步措施，都无法在多线程环境中并发使用的代码。由于 Java 语言天生就具备多线程特性，线程对立这种排斥多线程的代码是很少出现的，而且通常都是有害的，应当尽量避免。

43.互斥同步，synchronized 和 ReentrantLock。

44.非阻塞同步，互斥同步最主要的问题就是进行线程阻塞和唤醒所带来的性能问题，因此这种同步也称为阻塞同步。

互斥同步属于一种悲观的并发策略，总是认为只要不去做正确的同步措施，那就肯定会出现问题。论共享数据是否真的会出现竞争，它都要进行加锁（这里讨论的是概念模型，实际上虚拟机会优化掉很大一部分不必要的加锁）、用户态核心态转换、维护锁计数器和检查是否有被阻塞的线程需要唤醒等操作。

随着硬件指令集的发展，我们可以使用基于冲突检测的乐观并发策略：先进行操作，如果没有其它线程争用共享数据，那操作就成功了，否则采取补偿措施（不断地重试，直到成功为止）。这种乐观的并发策略的许多实现都不需要把线程挂起，因此这种同步操作称为非阻塞同步。

乐观锁需要操作和冲突检测这两个步骤具备原子性，这里就不能再使用互斥同步来保证了，只能靠硬件来完成。

硬件支持的原子性操作最典型的是：比较并交换（Compare-and-Swap，CAS）。CAS 指令需要有 3 个操作数，分别是内存地址 V、旧的预期值 A 和新值 B。当执行操作时，只有当 V 的值等于 A，才将 V 的值更新为B。

J.U.C 包里面的整数原子类 AtomicInteger，其中的 compareAndSet() 和 getAndIncrement() 等方法都使用了 Unsafe 类的 CAS 操作。

ABA:如果一个变量初次读取的时候是 A 值，它的值被改成了 B，后来又被改回为 A，那 CAS 操作就会误认为它从来没有被改变过。J.U.C 包提供了一个带有标记的原子引用类. AtomicStampedReference 来解决这个问题，它可以通过控制变量值的版本来保证 CAS 的正确性。大部分情况下 ABA 问题不会影响程序并发的正确性，如果需要解决 ABA 问题，改用传统的互斥同步可能会比原子类更高效。

45.无同步方案，要保证线程安全，并不是一定就要进行同步，两者没有因果关系。同步只是保证共享数据争用时的正确性的手段，如果一个方法本来就不涉及共享数据，那它自然就无须任何同步措施去保证正确性，因此会有一些代码天生就是线程安全的。

* 可重入代码，这种代码也叫做纯代码（Pure Code），可以在代码执行的任何时刻中断它，转而去执行另外一段代码（包括递归调用它本身），而在控制权返回后，原来的程序不会出现任何错误。可重入代码有一些共同的特征，例如不依赖存储在堆上的数据和公用的系统资源、用到的状态量都由参数中传入、不调用非可重入的方法等。
* 栈封闭，多个线程访问同一个方法的局部变量时，不会出现线程安全问题，因为局部变量存储在栈中，属于线程私有的。
* 线程本地存储，如果一段代码中所需要的数据必须与其他代码共享，那就看看这些共享数据的代码是否能保证在同一个线程中执行。如果能保证，我们就可以把共享数据的可见范围限制在同一个线程之内，这样，无须同步也能保证线程之间不出现数据争用的问题。可以使用 java.lang.ThreadLocal 类来实现线程本地存储功能。
```
public class ThreadLocalExample1 {
    public static void main(String[] args) {
        ThreadLocal threadLocal1 = new ThreadLocal();
        ThreadLocal threadLocal2 = new ThreadLocal();
        Thread thread1 = new Thread(() -> {
            threadLocal1.set(1);
            threadLocal2.set(1);
        });
        Thread thread2 = new Thread(() -> {
            threadLocal1.set(2);
            threadLocal2.set(2);
        });
        thread1.start();
        thread2.start();
    }
}
```
对应底层结构：

![ThreadLocal](/img/thread-local.png)
￼
每个 Thread 都有一个 ThreadLocal.ThreadLocalMap 对象，Thread 类中就定义了 ThreadLocal.ThreadLocalMap 成员。ThreadLocal.ThreadLocalMap threadLocals = null;
当调用一个 ThreadLocal 的 set(T value) 方法时，先得到当前线程的 ThreadLocalMap 对象，然后将 ThreadLocal->value 键值对插入到该 Map 中。
```
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
get() 方法类似。
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

ThreadLocal 从理论上讲并不是用来解决多线程并发问题的，因为根本不存在多线程竞争。在一些场景 (尤其是使用线程池) 下，由于 ThreadLocal.ThreadLocalMap 的底层数据结构导致 ThreadLocal 有内存泄漏的情况，尽可能在每次使用 ThreadLocal 后手动调用 remove()，以避免出现 ThreadLocal 经典的内存泄漏甚至是造成自身业务混乱的风险。

46.锁优化，这里的锁优化主要是指虚拟机对 synchronized 的优化。

47.自旋锁，互斥同步的进入阻塞状态的开销都很大，应该尽量避免。在许多应用中，共享数据的锁定状态只会持续很短的一段时间。自旋锁的思想是让一个线程在请求一个共享数据的锁时执行忙循环（自旋）一段时间，如果在这段时间内能获得锁，就可以避免进入阻塞状态。

自旋锁虽然能避免进入阻塞状态从而减少开销，但是它需要进行忙循环操作占用 CPU 时间，它只适用于共享数据的锁定状态很短的场景。

在 JDK 1.6 中引入了自适应的自旋锁。自适应意味着自旋的次数不再固定了，而是由前一次在同一个锁上的自旋次数及锁的拥有者的状态来决定。

48.锁消除，锁消除是指对于被检测出不可能存在竞争的共享数据的锁进行消除。

锁消除主要是通过逃逸分析来支持，如果堆上的共享数据不可能逃逸出去被其它线程访问到，那么就可以把它们当成私有数据对待，也就可以将它们的锁进行消除。

对于一些看起来没有加锁的代码，其实隐式的加了很多锁。例如下面的字符串拼接代码就隐式加了锁：
```
public static String concatString(String s1, String s2, String s3) {
    return s1 + s2 + s3;
}
```
String 是一个不可变的类，编译器会对 String 的拼接自动优化。在 JDK 1.5 之前，会转化为 StringBuffer 对象的连续 append() 操作：
```
public static String concatString(String s1, String s2, String s3) {
    StringBuffer sb = new StringBuffer();
    sb.append(s1);
    sb.append(s2);
    sb.append(s3);
    return sb.toString();
}
```

每个 append() 方法中都有一个同步块。虚拟机观察变量 sb，很快就会发现它的动态作用域被限制在 concatString() 方法内部。也就是说，sb 的所有引用永远不会“逃逸”到 concatString() 方法之外，其他线程无法访问到它，因此可以进行消除。

48.锁粗化，如果一系列的连续操作都对同一个对象反复加锁和解锁，频繁的加锁操作就会导致性能损耗。

上一节的示例代码中连续的 append() 方法就属于这类情况。如果虚拟机探测到由这样的一串零碎的操作都对同一个对象加锁，将会把加锁的范围扩展（粗化）到整个操作序列的外部。对于上一节的示例代码就是扩展到第一个 append() 操作之前直至最后一个 append() 操作之后，这样只需要加锁一次就可以了。

49.轻量级锁，JDK 1.6 引入了偏向锁和轻量级锁，从而让锁拥有了四个状态：无锁状态（unlocked）、偏向锁状态（biasble）、轻量级锁状态（lightweight locked）和重量级锁状态（inflated）。
轻量级锁是相对于传统的重量级锁而言，它使用 CAS 操作来避免重量级锁使用互斥量的开销。对于绝大部分的锁，在整个同步周期内都是不存在竞争的，因此也就不需要都使用互斥量进行同步，可以先采用 CAS 操作进行同步，如果 CAS 失败了再改用互斥量进行同步。

当尝试获取一个锁对象时，如果锁对象标记为 0 01，说明锁对象的锁未锁定（unlocked）状态。此时虚拟机在当前线程栈中创建 Lock Record，然后使用 CAS 操作将对象的 Mark Word 更新为 Lock Record 指针。如果 CAS 操作成功了，那么线程就获取了该对象上的锁，并且对象的 Mark Word 的锁标记变为 00，表示该对象处于轻量级锁状态。

如果 CAS 操作失败了，虚拟机首先会检查对象的 Mark Word 是否指向当前线程的虚拟机栈，如果是的话说明当前线程已经拥有了这个锁对象，那就可以直接进入同步块继续执行，否则说明这个锁对象已经被其他线程线程抢占了。如果有两条以上的线程争用同一个锁，那轻量级锁就不再有效，要膨胀为重量级锁。

50.偏向锁，偏向锁的思想是偏向于让第一个获取锁对象的线程，这个线程在之后获取该锁就不再需要进行同步操作，甚至连 CAS 操作也不再需要。当锁对象第一次被线程获得的时候，进入偏向状态，标记为 1 01。同时使用 CAS 操作将线程 ID 记录到 Mark Word 中，如果 CAS 操作成功，这个线程以后每次进入这个锁相关的同步块就不需要再进行任何同步操作。当有另外一个线程去尝试获取这个锁对象时，偏向状态就宣告结束，此时撤销偏向（Revoke Bias）后恢复到未锁定状态或者轻量级锁状态。

偏向锁的释放：

偏向锁只有遇到其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁，线程不会主动去释放偏向锁。偏向锁的撤销，需要等待全局安全点（在这个时间点上没有字节码正在执行），它会首先暂停拥有偏向锁的线程，判断锁对象是否处于被锁定状态，撤销偏向锁后恢复到未锁定（标志位为“01”）或轻量级锁（标志位为“00”）的状态。偏向锁最好在锁无竞争的情况下使用，一旦有了竞争就升级为轻量级锁，升级为轻量级锁的时候需要撤销偏向锁，在有锁的竞争时，偏向锁会多做很多额外操作，尤其是撤销偏向锁的时候会导致进入安全点，安全点会导致stw，导致性能下降，这种情况下应当禁用。

51.多线程开发良好实践
* 给线程起个有意义的名字，这样可以方便找 Bug。
* 缩小同步范围，例如对于 synchronized，应该尽量使用同步块而不是同步方法。
* 多用同步类少用 wait() 和 notify()。首先，CountDownLatch, Semaphore, CyclicBarrier 和 Exchanger 这些同步类简化了编码操作，而用 wait() 和 notify() 很难实现对复杂控制流的控制。其次，这些类是由最好的企业编写和维护，在后续的 JDK 中它们还会不断优化和完善，使用这些更高等级的同步工具你的程序可以不费吹灰之力获得优化。
* 多用并发集合少用同步集合。并发集合比同步集合的可扩展性更好，例如应该使用 ConcurrentHashMap 而不是 Hashtable。
* 使用本地变量和不可变类来保证线程安全。
* 使用线程池而不是直接创建 Thread 对象，这是因为创建线程代价很高，线程池可以有效地利用有限的线程来启动任务。
* 使用 BlockingQueue 实现生产者消费者问题。

52.markword,markword数据的长度在32位和64位的虚拟机（未开启压缩指针）中分别为32bit和64bit，它的最后2bit是锁状态标志位，用来标记当前对象的状态，对象的所处的状态，决定了markword存储的内容.

53.加锁流程：

偏向所锁，轻量级锁都是乐观锁，重量级锁是悲观锁。

一个对象刚开始实例化的时候，没有任何线程来访问它的时候。它是可偏向的，意味着，它现在认为只可能有一个线程来访问它，所以当第一个线程来访问它的时候，它会偏向这个线程，此时，对象持有偏向锁。偏向第一个线程，这个线程在修改对象头成为偏向锁的时候使用CAS操作，并将对象头中的ThreadID改成自己的ID，之后再次访问这个对象时，只需要对比ID，不需要再使用CAS在进行操作。

一旦有第二个线程访问这个对象，因为偏向锁不会主动释放，所以第二个线程可以看到对象时偏向状态，这时表明在这个对象上已经存在竞争了，检查原来持有该对象锁的线程是否依然存活，如果挂了，则可以将对象变为无锁状态，然后重新偏向新的线程，如果原来的线程依然存活，则马上执行那个线程的操作栈，检查该对象的使用情况，如果仍然需要持有偏向锁，则偏向锁升级为轻量级锁，（偏向锁就是这个时候升级为轻量级锁的）。如果不存在使用了，则可以将对象回复成无锁状态，然后重新偏向。

轻量级锁认为竞争存在，但是竞争的程度很轻，一般两个线程对于同一个锁的操作都会错开，或者说稍微等待一下（自旋），另一个线程就会释放锁。 但是当自旋超过一定的次数，或者一个线程在持有锁，一个在自旋，又有第三个来访时，轻量级锁膨胀为重量级锁，重量级锁使除了拥有锁的线程以外的线程都阻塞，防止CPU空转。
