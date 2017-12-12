#### 可重入锁

本文里面讲的是广义上的可重入锁，而不是单指JAVA下的ReentrantLock。

可重入锁，也叫做递归锁，指的是同一线程外层函数获得锁之后，内层递归函数仍然有获取该锁的代码，但不受影响。在JAVA环境下ReentrantLock和synchronized都是可重入锁

下面是使用实例

````java
package cn.linuxcrypt.raspi.lock;

import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicReference;
import java.util.concurrent.locks.ReentrantLock;

/**
 * 功能：
 * 作者：刘柏勋
 * 联系：wmsjhappy@gmail.com
 * 时间：17-12-12 上午11:01
 * 更新：
 * 备注：
 */
public class ReentrantLockTest implements Runnable {
    ReentrantLock lock = new ReentrantLock();
    public synchronized void get(){
        lock.lock();
        System.out.println(Thread.currentThread().getId());
        set();
        lock.unlock();
    }

    public synchronized void set(){
        lock.lock();
        System.out.println(Thread.currentThread().getId());
        lock.unlock();
    }

    @Override
    public void run() {
        get();
    }
    public static void main(String[] args) {
        ReentrantLockTest ss=new ReentrantLockTest();
        new Thread(ss).start();
        new Thread(ss).start();

        SynchronizedLock sl = new SynchronizedLock();
        new Thread(sl).start();
        new Thread(sl).start();

        SpinLock2 sp2 = new SpinLock2();
        new Thread(sp2).start();
        new Thread(sp2).start();

        SpinLock3 sp3 = new SpinLock3();
        new Thread(sp3).start();
        new Thread(sp3).start();
        new Thread(sp3).start();
        new Thread(sp3).start();

    }
}

class SynchronizedLock implements Runnable{
    public synchronized void get(){
        System.out.println(Thread.currentThread().getId());
        set();
    }

    public synchronized void set(){
        System.out.println(Thread.currentThread().getId());
    }

    @Override
    public void run() {
        get();
    }
}

/**
 * 自旋锁， 依据锁持有者是否为当前线程
 */
class SpinLock2 implements Runnable{
    private AtomicReference<Thread> owner = new AtomicReference<>();

    /**
     * 当同一个线程再次获取lock()会产生死锁，需要标记当前的持有者是否为本人
     */
    public void lock(){
        Thread thread = Thread.currentThread();
        // 当其他线程是否时（将owner设置为null）进行自旋竞争
        while(!owner.compareAndSet(null, thread)){
            System.out.println("...");
        }
    }

    /**
     * 当同一线程多次释放也会产生死锁，当lock()可以重入后，进行unlock()也同样会出发死锁，需要增加一个计数器用于标记，防止产生死锁
     */
    public void unlock(){
        Thread thread = Thread.currentThread();
        owner.compareAndSet(thread, null);
    }
    @Override
    public void run() {
        /**
         * 此处调用会产生死锁
        this.lock();
        System.out.println(Thread.currentThread().getId());
        this.lock();
        System.out.println(Thread.currentThread().getId());
        this.unlock();
        System.out.println(Thread.currentThread().getId());
        this.unlock();
        */
        /**
         * lock() --> 处理函数 --> unlcok()
         */
        this.lock();
        System.out.println(Thread.currentThread().getId());
        this.unlock();
    }
}

/**
 * 改良版
 * 增加线程是否与持有者为用一个人
 * 增加一个计数器，防止解锁产生死锁
 */
class SpinLock3 implements Runnable{
    private AtomicReference<Thread> owner = new AtomicReference<>();
    private AtomicInteger count = new AtomicInteger(0);

    public void lock(){
        Thread thread = Thread.currentThread();
        if (thread ==  owner.get()){
            count.getAndIncrement();
            return;
        }
        while(!owner.compareAndSet(null, thread)){
            System.out.println(",,,");
        }
    }
    public void unlock(){
        Thread thread = Thread.currentThread();
        if(thread == owner.get()){
            if (count.get() != 0){
               count.decrementAndGet();
               return;
            }
        }
        owner.compareAndSet(thread, null);
    }
    @Override
    public void run() {
        /**
         * 支持重入
         */
        this.lock();
        System.out.println(Thread.currentThread().getId());
        this.lock();
        System.out.println(Thread.currentThread().getId());
        this.unlock();
        System.out.println(Thread.currentThread().getId());
        this.unlock();
    }
}

````
