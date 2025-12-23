# Java并发编程核心

## 📚 一、核心概念前置理解

### 1. JMM（Java内存模型）
Java内存模型是定义多线程环境下内存访问规范的抽象模型，它解决了多线程并发中的三大核心问题：

#### 内存结构
```java
┌─────────────┐    读取/写入     ┌─────────────┐
│   主内存     │ ←──────────→  │  工作内存    │
│ 共享变量X=1  │                 │ 变量副本X=1  │
└─────────────┘                 └─────────────┘
       ↑                               ↑
   所有线程共享                    每个线程独有
```

#### 三大核心问题
- **可见性**：线程对共享变量的修改能否被其他线程及时看到
- **原子性**：一个或多个操作要么全部执行，要么全部不执行
- **有序性**：程序执行的顺序按照代码的先后顺序

#### 内存屏障示例
```java
public class MemoryBarrierExample {
    private int x = 0;
    private volatile boolean flag = false;
    
    public void writer() {
        x = 42;           // 普通写
        flag = true;      // volatile写，插入StoreStore屏障
    }
    
    public void reader() {
        if (flag) {       // volatile读，插入LoadLoad屏障
            System.out.println(x); // 保证看到x=42
        }
    }
}
```

### 2. volatile关键字
volatile是轻量级的同步机制，适用于一写多读的场景：

#### 工作原理
```java
public class VolatileDemo {
    private volatile boolean running = true;
    
    public void stop() {
        running = false;  // 修改后立即刷新到主内存
    }
    
    public void work() {
        while (running) { // 每次读取都从主内存获取最新值
            // 工作逻辑
        }
    }
}
```

#### volatile vs synchronized
```java
// volatile适用场景：状态标志位
class SafeShutdown {
    private volatile boolean shutdownRequested;
    
    public void shutdown() { shutdownRequested = true; }
    
    public void doWork() {
        while (!shutdownRequested) {
            // 执行任务
        }
    }
}

// 不适用场景：复合操作
class Counter {
    private volatile int count = 0;
    
    public void increment() {
        count++; // 非原子操作：1.读取 2.加1 3.写入
                 // 多个线程同时执行时可能丢失更新
    }
}
```

### 3. CAS（Compare-And-Swap）
无锁编程的核心思想，基于硬件指令实现：

#### CAS工作原理
```java
public class CASOperation {
    // 伪代码展示CAS流程
    public boolean compareAndSwap(int expectedValue, int newValue) {
        // 1. 读取当前内存值
        int currentValue = getFromMemory();
        
        // 2. 比较内存值与预期值
        if (currentValue == expectedValue) {
            // 3. 相等则更新为新值
            writeToMemory(newValue);
            return true;
        }
        return false;
    }
}
```

#### 实际应用示例
```java
import java.util.concurrent.atomic.AtomicInteger;

public class CASExample {
    private AtomicInteger counter = new AtomicInteger(0);
    
    public void safeIncrement() {
        int oldValue, newValue;
        do {
            oldValue = counter.get();      // 读取当前值
            newValue = oldValue + 1;       // 计算新值
        } while (!counter.compareAndSet(oldValue, newValue)); // CAS循环
    }
    
    // 解决ABA问题
    private AtomicStampedReference<Integer> stampedRef = 
        new AtomicStampedReference<>(0, 0);
    
    public void safeUpdate() {
        int[] stampHolder = new int[1];
        int oldValue = stampedRef.get(stampHolder);
        int newStamp = stampHolder[0] + 1;
        int newValue = oldValue + 10;
        
        stampedRef.compareAndSet(oldValue, newValue, 
                                stampHolder[0], newStamp);
    }
}
```

### 4. 线程池详解
线程池是并发编程的核心组件，合理使用可以显著提升性能：

#### 线程池工作流程
```
任务提交 → 核心线程 → 任务队列 → 非核心线程 → 拒绝策略
    ↓         ↓           ↓           ↓          ↓
  execute   running    waiting      expand     handler
```

#### 线程池创建示例
```java
import java.util.concurrent.*;

public class ThreadPoolDemo {
    
    // 1. 标准ThreadPoolExecutor
    public static ExecutorService createStandardPool() {
        return new ThreadPoolExecutor(
            5,                      // corePoolSize: 核心线程数
            10,                     // maximumPoolSize: 最大线程数
            60L, TimeUnit.SECONDS,  // keepAliveTime: 空闲线程存活时间
            new LinkedBlockingQueue<>(100), // workQueue: 任务队列
            Executors.defaultThreadFactory(), // threadFactory: 线程工厂
            new ThreadPoolExecutor.AbortPolicy() // handler: 拒绝策略
        );
    }
    
    // 2. 不同类型的线程池
    public static void differentPools() {
        // 固定大小线程池
        ExecutorService fixedPool = Executors.newFixedThreadPool(10);
        
        // 缓存线程池（自动扩容）
        ExecutorService cachedPool = Executors.newCachedThreadPool();
        
        // 单线程池（保证顺序执行）
        ExecutorService singlePool = Executors.newSingleThreadExecutor();
        
        // 调度线程池
        ScheduledExecutorService scheduledPool = 
            Executors.newScheduledThreadPool(5);
    }
    
    // 3. 任务提交方式
    public static void submitTasks(ExecutorService executor) {
        // 无返回值任务
        executor.execute(() -> System.out.println("Execute task"));
        
        // 有返回值任务
        Future<String> future = executor.submit(() -> {
            Thread.sleep(1000);
            return "Task Result";
        });
        
        try {
            String result = future.get(2, TimeUnit.SECONDS);
            System.out.println("Result: " + result);
        } catch (Exception e) {
            future.cancel(true); // 取消任务
        }
    }
}
```

### 5. 锁的核心分类
```java
// 锁分类示意代码
public class LockClassification {
    
    // 1. 悲观锁 vs 乐观锁
    public void pessimisticVsOptimistic() {
        // 悲观锁：假设会有竞争，先加锁
        synchronized(this) {
            // 执行操作
        }
        
        // 乐观锁：假设无竞争，失败重试
        AtomicInteger atomicInt = new AtomicInteger(0);
        atomicInt.incrementAndGet(); // 基于CAS
    }
    
    // 2. 可重入锁演示
    public class ReentrantExample {
        private final Object lock = new Object();
        
        public void outer() {
            synchronized(lock) {
                System.out.println("Outer lock");
                inner(); // 可以重入
            }
        }
        
        public void inner() {
            synchronized(lock) { // 同一线程可重入
                System.out.println("Inner lock");
            }
        }
    }
    
    // 3. 公平锁 vs 非公平锁
    public void fairVsNonFair() {
        // 公平锁：按等待顺序获取
        ReentrantLock fairLock = new ReentrantLock(true);
        
        // 非公平锁：抢占式获取（性能更好）
        ReentrantLock nonFairLock = new ReentrantLock(false);
    }
}
```

## 🔄 二、生产者消费者模型实现

### 1. synchronized版本（最经典）
```java
public class ProducerConsumerSynchronized {
    private final Queue<Integer> buffer = new LinkedList<>();
    private final int maxSize;
    
    public ProducerConsumerSynchronized(int maxSize) {
        this.maxSize = maxSize;
    }
    
    public synchronized void produce(int value) throws InterruptedException {
        // 缓冲区满时等待
        while (buffer.size() == maxSize) {
            System.out.println("缓冲区满，生产者等待...");
            wait();
        }
        
        buffer.offer(value);
        System.out.println("生产: " + value + "，缓冲区大小: " + buffer.size());
        
        // 通知消费者
        notifyAll();
    }
    
    public synchronized int consume() throws InterruptedException {
        // 缓冲区空时等待
        while (buffer.isEmpty()) {
            System.out.println("缓冲区空，消费者等待...");
            wait();
        }
        
        int value = buffer.poll();
        System.out.println("消费: " + value + "，缓冲区大小: " + buffer.size());
        
        // 通知生产者
        notifyAll();
        return value;
    }
}
```

### 2. ReentrantLock + Condition版本（更灵活）
```java
import java.util.concurrent.locks.*;

public class ProducerConsumerReentrantLock {
    private final Queue<Integer> buffer = new LinkedList<>();
    private final int maxSize;
    private final Lock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();  // 缓冲区未满条件
    private final Condition notEmpty = lock.newCondition(); // 缓冲区非空条件
    
    public ProducerConsumerReentrantLock(int maxSize) {
        this.maxSize = maxSize;
    }
    
    public void produce(int value) throws InterruptedException {
        lock.lock();
        try {
            while (buffer.size() == maxSize) {
                System.out.println("缓冲区满，生产者等待...");
                notFull.await(); // 等待"未满"条件
            }
            
            buffer.offer(value);
            System.out.println("生产: " + value + "，缓冲区大小: " + buffer.size());
            
            notEmpty.signal(); // 唤醒等待"非空"的消费者
        } finally {
            lock.unlock();
        }
    }
    
    public int consume() throws InterruptedException {
        lock.lock();
        try {
            while (buffer.isEmpty()) {
                System.out.println("缓冲区空，消费者等待...");
                notEmpty.await(); // 等待"非空"条件
            }
            
            int value = buffer.poll();
            System.out.println("消费: " + value + "，缓冲区大小: " + buffer.size());
            
            notFull.signal(); // 唤醒等待"未满"的生产者
            return value;
        } finally {
            lock.unlock();
        }
    }
}
```

### 3. BlockingQueue版本（最简单）
```java
import java.util.concurrent.*;

public class ProducerConsumerBlockingQueue {
    private final BlockingQueue<Integer> queue;
    
    public ProducerConsumerBlockingQueue(int capacity) {
        this.queue = new ArrayBlockingQueue<>(capacity);
    }
    
    // 生产者线程
    class Producer implements Runnable {
        private final int id;
        
        public Producer(int id) {
            this.id = id;
        }
        
        @Override
        public void run() {
            try {
                for (int i = 0; i < 10; i++) {
                    int value = id * 100 + i;
                    queue.put(value); // 队列满时会自动阻塞
                    System.out.printf("生产者%d生产: %d%n", id, value);
                    Thread.sleep((int)(Math.random() * 100));
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
    
    // 消费者线程
    class Consumer implements Runnable {
        private final int id;
        
        public Consumer(int id) {
            this.id = id;
        }
        
        @Override
        public void run() {
            try {
                while (true) {
                    Integer value = queue.take(); // 队列空时会自动阻塞
                    if (value == null) break;
                    System.out.printf("消费者%d消费: %d%n", id, value);
                    Thread.sleep((int)(Math.random() * 150));
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
    
    public void start() {
        // 创建线程池
        ExecutorService executor = Executors.newCachedThreadPool();
        
        // 启动生产者
        for (int i = 0; i < 3; i++) {
            executor.execute(new Producer(i));
        }
        
        // 启动消费者
        for (int i = 0; i < 2; i++) {
            executor.execute(new Consumer(i));
        }
        
        executor.shutdown();
    }
}
```

### 4. 测试主程序
```java
public class ProducerConsumerDemo {
    public static void main(String[] args) {
        System.out.println("=== 生产者消费者模型演示 ===\n");
        
        // 1. synchronized版本测试
        testSynchronizedVersion();
        
        // 2. ReentrantLock版本测试
        testReentrantLockVersion();
        
        // 3. BlockingQueue版本测试
        testBlockingQueueVersion();
    }
    
    private static void testSynchronizedVersion() {
        System.out.println("\n1. synchronized版本:");
        ProducerConsumerSynchronized pc = new ProducerConsumerSynchronized(5);
        
        Thread producer = new Thread(() -> {
            try {
                for (int i = 1; i <= 10; i++) {
                    pc.produce(i);
                    Thread.sleep(100);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        Thread consumer = new Thread(() -> {
            try {
                for (int i = 1; i <= 10; i++) {
                    pc.consume();
                    Thread.sleep(150);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        producer.start();
        consumer.start();
        
        try {
            producer.join();
            consumer.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    
    private static void testReentrantLockVersion() {
        System.out.println("\n\n2. ReentrantLock版本:");
        ProducerConsumerReentrantLock pc = new ProducerConsumerReentrantLock(5);
        
        // 多生产者多消费者
        ExecutorService executor = Executors.newFixedThreadPool(4);
        
        for (int i = 0; i < 2; i++) {
            executor.execute(() -> {
                try {
                    for (int j = 0; j < 5; j++) {
                        pc.produce(j);
                        Thread.sleep(50);
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }
        
        for (int i = 0; i < 2; i++) {
            executor.execute(() -> {
                try {
                    for (int j = 0; j < 5; j++) {
                        pc.consume();
                        Thread.sleep(80);
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }
        
        executor.shutdown();
        try {
            executor.awaitTermination(5, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    
    private static void testBlockingQueueVersion() {
        System.out.println("\n\n3. BlockingQueue版本:");
        ProducerConsumerBlockingQueue pc = new ProducerConsumerBlockingQueue(5);
        pc.start();
    }
}
```

## ⚖️ 三、synchronized与ReentrantLock深度对比

### 对比表格

| **维度** | **synchronized** | **ReentrantLock** |
|---------|-----------------|------------------|
| **实现层次** | JVM内置，通过monitor实现 | JDK实现，基于AQS |
| **锁类型** | 非公平锁（不可配置） | 可配置公平/非公平锁 |
| **锁的获取** | 自动获取/释放 | 手动lock()/unlock() |
| **可中断性** | 不支持中断等待 | 支持lockInterruptibly() |
| **超时机制** | 不支持超时 | 支持tryLock(timeout) |
| **条件变量** | 单个条件队列(wait/notify) | 多个Condition对象 |
| **锁状态** | 无法查询 | 可查询(isLocked等) |
| **性能** | JDK6后优化，低竞争下相当 | 高竞争下更优 |
| **重入性** | 支持可重入 | 支持可重入 |

### 代码示例对比

```java
public class SynchronizedVsReentrantLock {
    
    // 1. synchronized方式
    public class SynchronizedCounter {
        private int count = 0;
        
        public synchronized void increment() {
            count++;
        }
        
        public synchronized int getCount() {
            return count;
        }
        
        public void transfer(SynchronizedCounter target, int amount) {
            synchronized(this) {
                synchronized(target) {
                    // 可能产生死锁！
                    this.count -= amount;
                    target.count += amount;
                }
            }
        }
    }
    
    // 2. ReentrantLock方式（更灵活）
    public class ReentrantLockCounter {
        private int count = 0;
        private final ReentrantLock lock = new ReentrantLock();
        
        public void increment() {
            lock.lock();
            try {
                count++;
            } finally {
                lock.unlock();
            }
        }
        
        public int getCount() {
            lock.lock();
            try {
                return count;
            } finally {
                lock.unlock();
            }
        }
        
        // 避免死锁的转账方法
        public boolean tryTransfer(ReentrantLockCounter target, int amount) {
            boolean thisLocked = false;
            boolean targetLocked = false;
            
            try {
                // 尝试获取两个锁（带超时）
                thisLocked = lock.tryLock(100, TimeUnit.MILLISECONDS);
                if (thisLocked) {
                    targetLocked = target.lock.tryLock(100, TimeUnit.MILLISECONDS);
                    if (targetLocked) {
                        this.count -= amount;
                        target.count += amount;
                        return true;
                    }
                }
                return false;
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return false;
            } finally {
                if (targetLocked) target.lock.unlock();
                if (thisLocked) lock.unlock();
            }
        }
    }
    
    // 3. Condition的高级用法
    public class BoundedBuffer {
        private final String[] buffer;
        private int putPtr, takePtr, count;
        private final ReentrantLock lock = new ReentrantLock();
        private final Condition notFull = lock.newCondition();
        private final Condition notEmpty = lock.newCondition();
        
        public BoundedBuffer(int capacity) {
            buffer = new String[capacity];
        }
        
        public void put(String x) throws InterruptedException {
            lock.lock();
            try {
                while (count == buffer.length) {
                    notFull.await(); // 等待"不满"信号
                }
                buffer[putPtr] = x;
                if (++putPtr == buffer.length) putPtr = 0;
                ++count;
                notEmpty.signal(); // 发送"非空"信号
            } finally {
                lock.unlock();
            }
        }
        
        public String take() throws InterruptedException {
            lock.lock();
            try {
                while (count == 0) {
                    notEmpty.await(); // 等待"非空"信号
                }
                String x = buffer[takePtr];
                if (++takePtr == buffer.length) takePtr = 0;
                --count;
                notFull.signal(); // 发送"不满"信号
                return x;
            } finally {
                lock.unlock();
            }
        }
    }
}
```

## 🎯 四、最佳实践总结

### 1. 性能优化建议
```java
public class ConcurrencyBestPractices {
    
    // 1. 减小锁粒度
    public class FineGrainedLocking {
        private final Object[] locks;
        private final Object[] data;
        
        public FineGrainedLocking(int size) {
            locks = new Object[size];
            data = new Object[size];
            for (int i = 0; i < size; i++) {
                locks[i] = new Object();
            }
        }
        
        public void update(int index, Object value) {
            synchronized(locks[index]) { // 只锁需要的部分
                data[index] = value;
            }
        }
    }
    
    // 2. 使用读写锁提高读性能
    public class ReadWriteLockDemo {
        private final Map<String, Object> cache = new HashMap<>();
        private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
        
        public Object get(String key) {
            rwLock.readLock().lock();
            try {
                return cache.get(key);
            } finally {
                rwLock.readLock().unlock();
            }
        }
        
        public void put(String key, Object value) {
            rwLock.writeLock().lock();
            try {
                cache.put(key, value);
            } finally {
                rwLock.writeLock().unlock();
            }
        }
    }
    
    // 3. 使用ThreadLocal避免共享
    public class ThreadLocalExample {
        private static final ThreadLocal<SimpleDateFormat> dateFormat =
            ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
        
        public String formatDate(Date date) {
            return dateFormat.get().format(date); // 每个线程有自己的实例
        }
    }
}
```

### 2. 常见问题及解决方案

| **问题** | **现象** | **解决方案** |
|---------|---------|------------|
| **死锁** | 线程相互等待 | 1. 固定锁获取顺序<br>2. 使用tryLock<br>3. 设置超时时间 |
| **活锁** | 线程不断重试但无法前进 | 1. 引入随机退避<br>2. 增加重试间隔 |
| **饥饿** | 某些线程永远得不到执行 | 1. 使用公平锁<br>2. 合理设置线程优先级 |
| **竞态条件** | 执行结果依赖执行时序 | 1. 同步访问共享资源<br>2. 使用原子操作 |

### 3. 调试和监控
```java
public class ConcurrencyDebugTools {
    
    // 1. 使用ThreadMXBean监控死锁
    public static void detectDeadlock() {
        ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
        long[] deadlockedThreads = threadMXBean.findDeadlockedThreads();
        
        if (deadlockedThreads != null) {
            System.out.println("发现死锁线程:");
            for (long threadId : deadlockedThreads) {
                ThreadInfo threadInfo = threadMXBean.getThreadInfo(threadId);
                System.out.println(threadInfo.getThreadName());
            }
        }
    }
    
    // 2. 线程堆栈分析
    public static void printAllThreads() {
        Map<Thread, StackTraceElement[]> allStackTraces = 
            Thread.getAllStackTraces();
        
        for (Map.Entry<Thread, StackTraceElement[]> entry : 
             allStackTraces.entrySet()) {
            Thread thread = entry.getKey();
            System.out.println("\n线程: " + thread.getName() + 
                             " 状态: " + thread.getState());
            
            for (StackTraceElement element : entry.getValue()) {
                System.out.println("    " + element);
            }
        }
    }
}
```
