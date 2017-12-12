#### 自旋锁其他种类

在自旋锁中 另有三种常见的锁形式:TicketLock ，CLHlock 和MCSlock

##### TicketLock

* 1.主要解决访问顺序的问题,发生在多核cpu上
* 2.代码如下

````java
package cn.linuxcrypt.raspi.lock;

import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * 功能：
 * 作者：刘柏勋
 * 联系：wmsjhappy@gmail.com
 * 时间：17-12-1 下午6:11
 * 更新：
 * 备注：
 */
public class TicketLock {
    private AtomicInteger serviceNum = new AtomicInteger();
    private AtomicInteger ticketNum = new AtomicInteger();
    private static final int size = 100;

    private static final ThreadLocal<Integer> LOCAL = new ThreadLocal<>();

    /**
     * 先获取凭证, 并将获取到凭证存储到当前线程中即(ThreadLocal)中
     * 然后判断本次凭证是否与服务串号相同,如果相同即进入,否则自旋.
     * 服务串号类似在叫号系统.每个人都可以提前取号,服务一个然后在执行下一个
     */
    public void lock() {
        int myticket = ticketNum.getAndIncrement();
        LOCAL.set(myticket);
        while (myticket != serviceNum.get()) {
//            System.out.println("while...");
        }
    }

    public void unlock() {
        int myticket = LOCAL.get();
        serviceNum.compareAndSet(myticket, myticket + 1);
    }

    public Integer ticket(){
        return LOCAL.get();
    }

    public static void main(String[] args) {
        TicketLock ticketLock = new TicketLock();

        Thread[] threads = new Thread[size];
        CyclicBarrier barrier = new CyclicBarrier(size);
        CountDownLatch mainThread = new CountDownLatch(size);
        Long start = System.nanoTime();

        for (int i = 0; i < size; i++) {
            threads[i] = new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        barrier.await();
                        System.out.println(ticketLock.ticket());
                        ticketLock.lock();
                        System.out.println("ticket searil num: " + ticketLock.ticket());
                        mainThread.countDown();
                    } catch (InterruptedException e) {
                    } catch (BrokenBarrierException e) {
                    } finally {
                        ticketLock.unlock();
                    }
                }
            });
            threads[i].start();
        }

        try {
            mainThread.await();
            System.out.println("TicketLock 运行时间: "+(System.nanoTime() - start));
        } catch (InterruptedException e) {

        }

    }
}
````

每次都要查询一个serviceNum 服务号，影响性能（必须要到主内存读取，并阻止其他cpu修改）。

##### CLHLock和MCSLock

是两种类型相似的公平锁，采用链表的形式进行排序，

###### CLHlock

````java
package cn.linuxcrypt.raspi.lock;

import java.util.Date;
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.atomic.AtomicReferenceFieldUpdater;

public class CLHLock {
    private final static int size = 100;

    public static class CLHNode {
        private volatile boolean isLocked = true;
    }

    @SuppressWarnings("unused")
    private volatile CLHNode tail;
    private static final ThreadLocal<CLHNode> LOCAL = new ThreadLocal<CLHNode>();
    private static final AtomicReferenceFieldUpdater<CLHLock, CLHNode>
            UPDATER = AtomicReferenceFieldUpdater.newUpdater(CLHLock.class, CLHNode.class, "tail");

    public void lock() {
        CLHNode node = new CLHNode();
        LOCAL.set(node);

        /**
         * 以原子方式返回旧值, 内部实现时do while循环处理,条件是cas
         * public V getAndSet(T obj, V newValue) {
         *      V prev;
         *      do {
         *          prev = get(obj);
         *      } while (!compareAndSet(obj, prev, newValue));
         *      return prev;
         *  }
         */
        CLHNode preNode = UPDATER.getAndSet(this, node);
//        System.out.println(preNode);
        if (preNode != null) {
            while (preNode.isLocked) {
            }
            preNode = null;
            LOCAL.set(node);
        }
    }

    public void unlock() {
        CLHNode node = LOCAL.get();
        if (!UPDATER.compareAndSet(this, node, null)) {
            node.isLocked = false;
        }
        node = null;
    }

    public static void main(String[] args) {
        CLHLock clhLock = new CLHLock();
        CyclicBarrier barrier = new CyclicBarrier(size);
        Long start = System.nanoTime();
        CountDownLatch mainThreadExit = new CountDownLatch(size);
        Thread[] threads = new Thread[size];
        for (int i = 0; i < size; i++) {
            final int index = i;
            threads[i] = new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        barrier.await();
                        clhLock.lock();
                        System.out.println(index+ "running...." + new Date().getTime());

                    } catch (InterruptedException e) {

                    } catch (BrokenBarrierException e) {

                    } finally {
                        clhLock.unlock();
                        mainThreadExit.countDown();
                    }
                }
            });
            threads[i].start();
        }

        try {
            mainThreadExit.await();
            /**
             * 1 秒 == 1000毫秒
             * 1毫秒 == 1000 微妙
             * 1微妙 == 1000 毫微妙
             * 1毫微妙 == 1纳秒
             * 1纳秒 == 10埃秒
             */
            System.out.println("消耗的时间为: "+(System.nanoTime() - start)+"毫微秒");

        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
````

####### MCSLock
MCSLock则是对本地变量的节点进行循环。不存在CLHlock 的问题。

````java
package cn.linuxcrypt.raspi.lock;

import java.util.concurrent.atomic.AtomicReferenceFieldUpdater;

public class MCSLock {
    public static class MCSNode {
        volatile MCSNode next;
        volatile boolean isLocked = true;
    }

    private static final ThreadLocal<MCSNode> NODE = new ThreadLocal<MCSNode>();
    @SuppressWarnings("unused")
    private volatile MCSNode queue;
    private static final AtomicReferenceFieldUpdater<MCSLock, MCSNode> UPDATER = AtomicReferenceFieldUpdater.newUpdater(MCSLock.class,
            MCSNode.class, "queue");

    public void lock() {
        MCSNode currentNode = new MCSNode();
        NODE.set(currentNode);
        MCSNode preNode = UPDATER.getAndSet(this, currentNode);
        if (preNode != null) {
            preNode.next = currentNode;
            while (currentNode.isLocked) {

            }
        }
    }

    public void unlock() {
        MCSNode currentNode = NODE.get();
        if (currentNode.next == null) {
            if (UPDATER.compareAndSet(this, currentNode, null)) {

            } else {
                while (currentNode.next == null) {
                }
            }
        } else {
            currentNode.next.isLocked = false;
            currentNode.next = null;
        }
    }
}
````

从代码上 看，CLH 要比 MCS 更简单，

CLH 的队列是隐式的队列，没有真实的后继结点属性。

MCS 的队列是显式的队列，有真实的后继结点属性。

JUC ReentrantLock 默认内部使用的锁 即是 CLH锁（有很多改进的地方，将自旋锁换成了阻塞锁等等）。
