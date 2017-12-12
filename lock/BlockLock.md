### 阻塞锁

阻塞锁，与自旋锁不同，改变了线程的运行状态。

阻塞锁的优势在于，阻塞的线程不会占用cpu时间，不会导致cpu占用率过高，但进入时间以及恢复时间都要比自旋锁略慢。

在竞争激烈的情况下阻塞锁的性能要明显高于自旋锁。

理想的情况则是;在线程竞争不激烈的情况下，使用自旋锁，竞争激烈的情况下使用，阻塞锁。

java线程一下状态

* 1.新建状态
* 2.就绪状态
* 3.运行状态
* 4.阻塞状态
* 5.死亡状态

阻塞锁，可以说是让线程进入阻塞状态进行等待，当获得相应的信号（唤醒，时间） 时，才可以进入线程的准备就绪状态，准备就绪状态的所有线程，通过竞争，进入运行状态。

JAVA中，能够进入\退出、阻塞状态或包含阻塞锁的方法有:
* synchronized 关键字（其中的重量锁）
* ReentrantLock
* Object.wait()|notify()
* LockSupport.park()|unpart()(j.u.c经常使用), LockSupport是JDK中比较底层的类，用来创建锁和其他同步工具类的基本线程阻塞原语。java锁和同步器框架的核心AQS:AbstractQueuedSynchronizer，就是通过调用LockSupport.park()和LockSupport.unpark()实现线程的阻塞和唤醒的。LockSupport很类似于二元信号量(只有1个许可证可供使用)，如果这个许可还没有被占用，当前线程获取许可并继续执行；如果许可已经被占用，当前线程阻塞，等待获取许可。

  > LockSupport是不可重入, 如果一个线程连续2次调用LockSupport.park()，那么该线程一定会一直阻塞下去。注意，当无法获取许可可能会出现死锁

  > LockSupport.park(), 许可默认是被占用的

  ````java
  public static void main(String[] args)
  {
       LockSupport.park();
       System.out.println("block.");//进入阻塞, 因为许可默认是被占用的，调用park()时获取不到许可，所以进入阻塞状态
  }
  ````

#### 阻塞例子

````java
package cn.linuxcrypt.raspi.lock;

import org.junit.Test;

import java.util.concurrent.atomic.AtomicReferenceFieldUpdater;
import java.util.concurrent.locks.LockSupport;

/**
 * 功能：
 * 作者：刘柏勋
 * 联系：wmsjhappy@gmail.com
 * 时间：17-12-11 上午10:55
 * 更新：
 * 备注：
 *      阻塞锁的优势在于，阻塞的线程不会占用cpu时间， 不会导致 cpu占用率过高，但进入时间以及恢复时间都要比自旋锁略慢。
 *      在竞争激烈的情况下 阻塞锁的性能要明显高于 自旋锁。
 *      理想的情况则是; 在线程竞争不激烈的情况下，使用自旋锁，竞争激烈的情况下使用，阻塞锁。
 */
public class BlockLock {
    public static class BlockNode {
        private volatile Thread isLocked;
    }

    private volatile BlockNode tail;
    private static final ThreadLocal<BlockNode> LOCAL = new ThreadLocal<>();
    private static final AtomicReferenceFieldUpdater<BlockLock, BlockNode> UPDATER =
            AtomicReferenceFieldUpdater.newUpdater(BlockLock.class, BlockNode.class, "tail");

    public void lock(){
        BlockNode blockNode = new BlockNode();
        LOCAL.set(blockNode);
        BlockNode preNode = UPDATER.getAndSet(this, blockNode);
        System.out.println(Thread.currentThread().getName());

        if(preNode != null){
            preNode.isLocked = Thread.currentThread();
            LockSupport.park(this);
            preNode = null;
            LOCAL.set(blockNode);
            System.out.println("proNode: "+preNode + " thread name: "+ preNode.isLocked.getName());
        }
    }

    public void unlock(){
        BlockNode blockNode = LOCAL.get();
        if (!UPDATER.compareAndSet(this, blockNode, null)){
            System.out.println("unlock: "+blockNode.isLocked);
            LockSupport.unpark(blockNode.isLocked);
        }
        blockNode = null;
    }

    @Test
    public void test(){
        Thread thread = Thread.currentThread();
        /**
         * 如果不将当前线程释放，会进入阻塞
         */
        LockSupport.unpark(thread); // 释放许可
        LockSupport.park();// 获取许可
        System.out.println("b");
    }

    public static void main(String[] args) throws InterruptedException {
        BlockLock bl = new BlockLock();
        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                bl.lock();
                System.out.println("1:--------------------running..");
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {

                }
                bl.unlock();
            }
        });
        t1.setName("t1");

        Thread t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                bl.lock();
                System.out.println("2:--------------------running..");
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {

                }
                bl.unlock();
            }
        });
        t2.setName("t2");

        t1.start();
        t2.start();
        bl.lock();
        Thread.sleep(3000);
        System.out.println("3:--------------------running..");
        bl.unlock();
    }
}

````
