    public class ReentrantReadWriteLock extends Object implements ReadWriteLock, Serializable

[Java Doc 地址] (http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/locks/ReentrantReadWriteLock.html)

`ReentrantReadWriteLock`是`ReadWriteLock`接口的实现，提供类似`ReentrantLock`的语义（作者觉得是指可重入）

这个类具有有下面的一些属性:

- __获取锁的顺序__

    这个类不强制要求读线程或者写线程获得锁的顺序。但是它提供一个可选的公平策略。

 - __*非公平模式 (默认)*__  
    当用非公平模式构造`ReentrantReadWriteLock`(默认构造器)，进入读锁和写锁的顺序是没有指定的，以重入约束为准。一个非公平锁即:不断竞争可能会使一个或多个读（写）线程无限期延迟，但是通常能提供比公平锁更高的吞吐量。

 - __*公平模式*__    
    当用公平模式构造`ReentrantReadWriteLock`, 线程使用一个近似的到达顺序策略来竞争锁。当当前保持的锁被释放，可能发生以下情况：  
    - 等待最长的单一的写线程将被分配写锁；  
    - 如果有一组读线程等待的时间比所有的等待中的写线程要长，这组读线程将被分配读锁。  

    如果有写锁保持或者存在一个正在等待的写线程，那么一个线程试图获取一个公平的读锁（非重入获取）将会被阻塞。这个线程将无法获得读锁，直到最早等待的写线程获取到锁并且释放写锁。当然，如果一个等待的写线程放弃等待，留下一个或多个读线程作为队列里面等待时间最长的等待者等待写锁释放，然后这些线程将被分配读锁。
一个线程尝试获取一个公平的写锁（非重入获取）将会阻塞除非所有的读锁和写锁都释放（意味着没有等待的线程）。(需要注意的是非阻塞方法`ReentrantReadWriteLock.ReadLock.tryLock()`和`ReentrantReadWriteLock.WriteLock.tryLock()`不履行这个公平设置并且将会获得锁，不管是否有等待的线程。)

- __重入__

    这个锁像`ReentrantLock`一样，允许读写线程两者都可以重新获取读锁或者写锁（笔者注释：读线程只能重新获取读锁；写线程可以获取读锁和写锁）。非重入的读线程不允许进入锁，除非所有写线程保持的写锁都被释放。
此外，一个写线程可以获取到读锁，但是反过来则不行。在某些应用中，当在调用或者回调的方法在读锁下执行读操作的时候写锁一直保持，重入很有用。

- __锁降级__

 重入同样允许写锁降级为读锁，通过获取写锁，然后获取读锁，然后释放写锁。然而从读锁升级成写锁是不可能的。

- __锁中断__

 读锁和写锁在锁定的时候都支持中断。

- __条件支持__

 写锁为相同行为提供了一个`Condition`的实现，和由`ReentrantLock.newCondition()`为`ReentrantLock`提供的`Condition`实现一样。这个条件只能被写锁使用。

 锁不支持条件并且`readLock().newCondition()`抛出`UnsupportedOperationException`异常。

- __植入?__

 这个类支持让方法决定是继续保持锁还是竞争。这些方法被设计来做系统状态监控，不是为了同步控制。

此类行为同样以内置锁的方式序列化：反序列化的锁处于解锁状态，不管序列化的时候是什么状态。

__示例用法.__ 这里有一个代码草图演示如何在更新一个缓存后执行锁降级（当非嵌套方式处理多个锁的时候异常处理尤其棘手）：

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
	 
ReentrantReadWriteLocks can be used to improve concurrency in some uses of some kinds of Collections. This is typically worthwhile only when the collections are expected to be large, accessed by more reader threads than writer threads, and entail operations with overhead that outweighs synchronization overhead. For example, here is a class using a TreeMap that is expected to be large and concurrently accessed.

    class RWDictionary {
	    private final Map<String, Data> m = new TreeMap<String, Data>();
	    private final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
	    private final Lock r = rwl.readLock();
	    private final Lock w = rwl.writeLock();
	
	    public Data get(String key) {
	        r.lock();
	        try { return m.get(key); }
	        finally { r.unlock(); }
	    }
	    public String[] allKeys() {
	        r.lock();
	        try { return m.keySet().toArray(); }
	        finally { r.unlock(); }
	    }
	    public Data put(String key, Data value) {
	        w.lock();
	        try { return m.put(key, value); }
	        finally { w.unlock(); }
	    }
	    public void clear() {
	        w.lock();
	        try { m.clear(); }
	        finally { w.unlock(); }
	    }
	 }

__Implementation Notes__

This lock supports a maximum of 65535 recursive write locks and 65535 read locks. Attempts to exceed these limits result in Error throws from locking methods.
