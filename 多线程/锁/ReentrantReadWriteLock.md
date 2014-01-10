    public class ReentrantReadWriteLock extends Object implements ReadWriteLock, Serializable

[Java Doc 地址] (http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/locks/ReentrantReadWriteLock.html)

`ReentrantReadWriteLock`是`ReadWriteLock`接口的实现，提供类似`ReentrantLock`的语义（作者觉得是指可重入）

这个类具有有下面的一些属性:

> 获取锁的顺序

这个类不强制要求读线程或者写线程获得锁的顺序。但是它提供一个可选的公平策略。

> 非公平模式 (默认)  
当用非公平模式构造`ReentrantReadWriteLock`(默认构造器)，进入读锁和写锁的顺序是没有指定的，以重入约束为准。一个非公平锁即:不断竞争可能会使一个或多个读（写）线程无限期延迟，但是通常能提供比公平锁更高的吞吐量。

> Fair mode  
When constructed as fair, threads contend for entry using an approximately arrival-order policy. When the currently held lock is released either the longest-waiting single writer thread will be assigned the write lock, or if there is a group of reader threads waiting longer than all waiting writer threads, that group will be assigned the read lock.
A thread that tries to acquire a fair read lock (non-reentrantly) will block if either the write lock is held, or there is a waiting writer thread. The thread will not acquire the read lock until after the oldest currently waiting writer thread has acquired and released the write lock. Of course, if a waiting writer abandons its wait, leaving one or more reader threads as the longest waiters in the queue with the write lock free, then those readers will be assigned the read lock.

A thread that tries to acquire a fair write lock (non-reentrantly) will block unless both the read lock and write lock are free (which implies there are no waiting threads). (Note that the non-blocking ReentrantReadWriteLock.ReadLock.tryLock() and ReentrantReadWriteLock.WriteLock.tryLock() methods do not honor this fair setting and will acquire the lock if it is possible, regardless of waiting threads.)
