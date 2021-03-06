### 自旋锁

让当前线程不停地在循环体内执行实现(一般获取可进入权限),当循环的条件被修改(即其他线程已经释放)时,才会进入临界区.

直接源码展示

```java
package cn.linuxcrypt.lock;

import java.util.Date;
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.atomic.AtomicReference;

/**
 * 功能：自旋锁
 * 作者：刘柏勋
 * 联系：wmsjhappy@gmail.com
 * 时间：17-12-1 下午2:31
 * 更新：
 * 备注：
 */
public class SpinLock {
    private final static int size = 100;
    private AtomicReference<Thread> sign = new AtomicReference<>();

    public void lock() {
        Thread current = Thread.currentThread();
        while (!sign.compareAndSet(null, current)) {
            // System.out.println("消耗cpu....");
        }
    }

    public void unlock() {
        Thread current = Thread.currentThread();
        sign.compareAndSet(current, null);
    }
    public static void main(String[] args) {
        SpinLock spin = new SpinLock();
        CyclicBarrier barrier = new CyclicBarrier(size);
        Long start = System.nanoTime();
        CountDownLatch mainThreadExit = new CountDownLatch(size);
        Thread[] threads = new Thread[size];
        for (int i = 0; i < size; i++) {
            threads[i] = new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        barrier.await();
                        spin.lock();
                        // System.out.println("running...." + new Date().getTime());
                    } catch (InterruptedException e) {

                    } catch (BrokenBarrierException e) {

                    } finally {
                        spin.unlock();
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
```

当获取lock时,使用CAS原子操作,将持有者设置为当前线程(Thread.currentThread()), 需要从null状态设置为当前状态

当第二个线程调用lock时, 在将持有者设置为本次线程(即第二个线程)时, 此使用cas判断原值为null,因为第一个线程暂未释放,需要等待,到时while一直循环,直至null的临界区进入;

自旋锁,当线程不断增加时,性能下降明显. 主要原因在while对cpu的消耗.一下对测试结果的记录

注意需要将System.out.println()注释,io消耗性能, 测试环境 jdk8u151 i3 4160 -Xmx1024m -Xms1024m

|线程数|消耗时间(豪微妙)|
|-----|-------------|
|100|21573219|
|200|25891846|
|300|49183070|
|400|56480389|
|500|76267453|
|1000|168965962|
|5000|1629772230|
