# FutureTask isDone 返回 false

大家好，我是烤鸭：

​	&nbsp;&nbsp;&nbsp;&nbsp;今天看一下 FutureTask源码。好吧，其实遇到问题了，哪里不会点哪里。



## 伪代码

```
package src.executor;

import org.springframework.scheduling.annotation.AsyncResult;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.*;

/**
 *@program: leetcode
 *@description: 测试
 *@author:  
 *@email: 
 *@create: 2021/07/07 11:35
 */
public class FutureAndLatchTest {
    static ThreadPoolTaskExecutor initTaskPool(){
        ThreadPoolTaskExecutor taskExecutor = new ThreadPoolTaskExecutor();
        taskExecutor.setCorePoolSize(15);
        taskExecutor.setMaxPoolSize(60);
        taskExecutor.setQueueCapacity(200);
        taskExecutor.setKeepAliveSeconds(60);
        taskExecutor.setThreadNamePrefix("test-");
        taskExecutor.setWaitForTasksToCompleteOnShutdown(true);
        taskExecutor.setRejectedExecutionHandler(new ThreadPoolExecutor.AbortPolicy());
        taskExecutor.setAwaitTerminationSeconds(60);
        taskExecutor.initialize();
        return taskExecutor;
    }
    
    public static void main(String[] args) {
        ThreadPoolTaskExecutor taskPool = initTaskPool();
        for (int i = 0; i < 20; i++) {
            CountDownLatch latch = new CountDownLatch(2);
            Future<Integer> f1 = taskPool.submit(() ->future(latch));
            Future<Integer> f2 = taskPool.submit(() ->future(latch));
            try {
                latch.await(200, TimeUnit.MILLISECONDS);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            if(!f1.isDone()){
                System.out.println("f1 is not done");
            }
            if(!f2.isDone()){
                System.out.println("f2 is not done");
            }
        }
        System.out.println("taskPool finish");
    }
    
    private static Integer future(CountDownLatch latch) {
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            latch.countDown();
        }
        return 1;
    }
}

```

这段代码维护了一个线程池，执行两个线程用CountDownLatch做超时控制，再判断线程是否完成，这段代码会输出 is not done么。看下实际结果。(有概率复现)

```
输出:
f1 is not done
f1 is not done
f2 is not done
f1 is not done
f1 is not done
f2 is not done
f1 is not done
taskPool finish
```



## 原因分析

看一下 Future 的方法注释，就是方法是否执行完成，理论上没问题啊。

```
/**
 * Returns {@code true} if this task completed.
 *
 * Completion may be due to normal termination, an exception, or
 * cancellation -- in all of these cases, this method will return
 * {@code true}.
 *
 * @return {@code true} if this task completed
 */
boolean isDone();
```

而实现调用的 FutureTask

```
public boolean isDone() {
    return state != NEW;
}
```

出现这个state还得再看下源码，state用来维护线程状态的，注释也说明了几种状态的流转。

```
/**
 * The run state of this task, initially NEW.  The run state
 * transitions to a terminal state only in methods set,
 * setException, and cancel.  During completion, state may take on
 * transient values of COMPLETING (while outcome is being set) or
 * INTERRUPTING (only while interrupting the runner to satisfy a
 * cancel(true)). Transitions from these intermediate to final
 * states use cheaper ordered/lazy writes because values are unique
 * and cannot be further modified.
 *
 * Possible state transitions:
 * NEW -> COMPLETING -> NORMAL
 * NEW -> COMPLETING -> EXCEPTIONAL
 * NEW -> CANCELLED
 * NEW -> INTERRUPTING -> INTERRUPTED
 */
private volatile int state;
private static final int NEW          = 0;
private static final int COMPLETING   = 1;
private static final int NORMAL       = 2;
private static final int EXCEPTIONAL  = 3;
private static final int CANCELLED    = 4;
private static final int INTERRUPTING = 5;
private static final int INTERRUPTED  = 6;
```

新建 -> 进行中 -> 完成、新建 -> 进行中 -> 异常、新建 -> 取消、新建 ->等待 ->取消

更多详细的可以看下这篇文章。

https://blog.csdn.net/qq_35067322/article/details/104872102

看下2012年的这个提问吧，和有位大神的回复。

https://stackoverflow.com/questions/9604713/future-isdone-returns-false-even-if-the-task-is-done

![1](.\1.png)

简单来说，就是子线程里调用finally 执行 countdownlatch.countdown()的时候，主线程发现 latch 变成0了就继续执行，但是这个时候 futureTask还在finally里，state没变过来。就是毫秒级别的线程切换，主线程在那一瞬间优先执行。



## 优化

原来代码里是想监听多线程的执行结果，执行完成后再去执行其他的操作。怎么样才能监听到实际结果呢，改为 Future.get();

```
try {
	f1.get(5,TimeUnit.MILLISECONDS);
	f2.get(5,TimeUnit.MILLISECONDS);
} catch (Exception e) {
	e.printStackTrace();
}
```

get 方法为啥没问题呢，看下源码。

```
/**
 * @throws CancellationException {@inheritDoc}
 */
public V get(long timeout, TimeUnit unit)
    throws InterruptedException, ExecutionException, TimeoutException {
    if (unit == null)
        throw new NullPointerException();
    int s = state;
    if (s <= COMPLETING &&
        (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
        throw new TimeoutException();
    return report(s);
}
```

状态没完成，就等他完成就好了。妥妥的CAS 乐观锁实现。

```
/**
 * Awaits completion or aborts on interrupt or timeout.
 *
 * @param timed true if use timed waits
 * @param nanos time to wait, if timed
 * @return state upon completion
 */
private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    boolean queued = false;
    for (;;) {
        if (Thread.interrupted()) {
            removeWaiter(q);
            throw new InterruptedException();
        }

        int s = state;
        if (s > COMPLETING) {
            if (q != null)
                q.thread = null;
            return s;
        }
        else if (s == COMPLETING) // cannot time out yet
            Thread.yield();
        else if (q == null)
            q = new WaitNode();
        else if (!queued)
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                 q.next = waiters, q);
        else if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                removeWaiter(q);
                return state;
            }
            LockSupport.parkNanos(this, nanos);
        }
        else
            LockSupport.park(this);
    }
}
```



