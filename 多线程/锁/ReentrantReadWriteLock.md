    public class ReentrantReadWriteLock extends Object implements ReadWriteLock, Serializable

[Java Doc 地址] (http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/locks/ReentrantReadWriteLock.html)

`ReentrantReadWriteLock`是`ReadWriteLock`接口的实现，提供类似`ReentrantLock`的语义（作者觉得是指可重入）

这个类具有有下面的一些属性:

> 获取锁的顺序

这个类不强制要求读线程或者写线程获得锁的顺序。但是它提供一个可选的公平策略。

> 非公平模式 (默认)  
当用非公平模式构造`ReentrantReadWriteLock`(默认构造器)，进入读锁和写锁的顺序是没有指定的，以重入约束为准。一个非公平锁即:不断竞争可能会使一个或多个读（写）线程无限期延迟，但是通常能提供比公平锁更高的吞吐量。

> 公平模式  
当用公平模式构造`ReentrantReadWriteLock`, 线程使用一个近似的到达顺序策略来竞争锁。当当前保持的锁被释放，可能发生以下情况：  
1. 等待最长的单一的写线程将被分配写锁；  
2. 如果有一组读线程等待的时间比所有的等待中的写线程要长，这组读线程将被分配读锁。  
如果有写锁保持或者存在一个正在等待的写线程，那么一个线程试图获取一个公平的读锁（非重入获取）将会被阻塞。这个线程将无法获得读锁，直到最早等待的写线程获取到锁并且释放写锁。当然，如果一个等待的写线程放弃等待，留下一个或多个读线程作为队列里面等待时间最长的等待者等待写锁释放，然后这些线程将被分配读锁。
一个线程尝试获取一个公平的写锁（非重入获取）将会阻塞除非所有的读锁和写锁都释放（意味着没有等待的线程）。(需要注意的是非阻塞方法`ReentrantReadWriteLock.ReadLock.tryLock()`和`ReentrantReadWriteLock.WriteLock.tryLock()`不履行这个公平设置并且将会获得锁，不管是否有等待的线程。)

> 重入  
这个锁像`ReentrantLock`一样，允许读写线程两者都可以重新获取读锁或者写锁（笔者注释：读线程只能重新获取读锁；写线程可以获取读锁和写锁）。没有重入的读线程不允许进入锁，除非所有写线程保持的写锁都被释放。

Additionally, a writer can acquire the read lock, but not vice-versa. Among other applications, reentrancy can be useful when write locks are held during calls or callbacks to methods that perform reads under read locks. If a reader tries to acquire the write lock it will never succeed.

> Lock downgrading  
Reentrancy also allows downgrading from the write lock to a read lock, by acquiring the write lock, then the read lock and then releasing the write lock. However, upgrading from a read lock to the write lock is not possible.

> Interruption of lock acquisition
The read lock and write lock both support interruption during lock acquisition.

> Condition support  
The write lock provides a Condition implementation that behaves in the same way, with respect to the write lock, as the Condition implementation provided by ReentrantLock.newCondition() does for ReentrantLock. This Condition can, of course, only be used with the write lock.

The read lock does not support a Condition and readLock().newCondition() throws UnsupportedOperationException.

> Instrumentation  
This class supports methods to determine whether locks are held or contended. These methods are designed for monitoring system state, not for synchronization control.

Serialization of this class behaves in the same way as built-in locks: a deserialized lock is in the unlocked state, regardless of its state when serialized.

> Sample usages. Here is a code sketch showing how to perform lock downgrading after updating a cache (exception handling is particularly tricky when handling multiple locks in a non-nested fashion):   

	class CachedData {
	   Object data;
	   volatile boolean cacheValid;
	   final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
	
	   void processCachedData() {
	     rwl.readLock().lock();
	     if (!cacheValid) {
	        // Must release read lock before acquiring write lock
	        rwl.readLock().unlock();
	        rwl.writeLock().lock();
	        try {
	          // Recheck state because another thread might have
	          // acquired write lock and changed state before we did.
	          if (!cacheValid) {
	            data = ...
	            cacheValid = true;
	          }
	          // Downgrade by acquiring read lock before releasing write lock
	          rwl.readLock().lock();
	        } finally {
	          rwl.writeLock().unlock(); // Unlock write, still hold read
	        }
	     }

	     try {
	       use(data);
	     } finally {
	       rwl.readLock().unlock();
	     }
     	   }
	}
