# 并发常用工具类

线程通讯几种方式
- 1、`synchronized`加wait/notify 休眠唤醒方式
- 2、`ReentrantLock`加Condition方式
- 3、`CountDownLatch` 闭锁方式
- 4、`CyclicBarrier` 栅栏的方式
- 5、`Semaphore` 信号量的方式
  - join的使用
  - yield的使用

```java
public class addEvenDemo {
	private int i = 0;// 打印的数
	private Object obj = new Object();
	public void odd() {
		// 小于10打印
		while (i < 10) {
			synchronized (obj) {
				if (i % 2 == 1) {
					System.out.println("奇数" + i);
					i++;
					obj.notify();
				} else {
					try {
						obj.wait();
					} catch (InterruptedException e) {
						// TODO Auto-generated catch block
						e.printStackTrace();
					} // 等待偶数打印完毕
				}
			}
		}
	}
	public void even() {
		// 小于10打印
		while (i < 10) {
			synchronized (obj) {
				if (i % 2 == 0) {
					System.out.println("偶数" + i);
					i++;
					obj.notify();
				} else {
					try {
						obj.wait();
					} catch (InterruptedException e) {
						// TODO Auto-generated catch block
						e.printStackTrace();
					} // 等待奇数打印完毕
				}
			}
		}
	}
	public static void main(String[] args) {
		addEvenDemo addEvenDemo = new addEvenDemo();
		new Thread(() -> {
			addEvenDemo.odd();
		}).start();
		new Thread(() -> {
			addEvenDemo.even();
		}).start();
	}
}
```

```java
private int i = 0;// 打印的数
	private Lock lock = new ReentrantLock();
	private Condition condition = lock.newCondition();
	public void odd() {
		// 小于10打印
		while (i < 10) {
			lock.lock();
			try {
				if (i % 2 == 1) {
					System.out.println("奇数" + i);
					i++;
					condition.signal();
				} else {
					condition.await();
				}
			} catch (Exception e) {
			} finally {
				lock.unlock();
			}
		}
	}
	public void even() {
		// 小于10打印
		while (i < 10) {
			lock.lock();
			try {
				if (i % 2 == 0) {
					System.out.println("偶数" + i);
					i++;
					condition.signal();
				} else {
					condition.await();
				}
			} catch (Exception e) {
			} finally {
				lock.unlock();
			}
		}
	}
```

## CountDownLatch源码解析和使用场景
- countDownLatch这个类使一个线程等待其他线程各自执行完毕后再执行。
- 是通过一个计数器来实现的，计数器的初始值是线程的数量。每当一个线程执行完毕后，计数器的值就-1，当计数器的值为0时，表示所有线程都执行完毕，然后在闭锁上等待的线程就可以恢复工作了。

```java
//14个分中心都各自算完以后 再计算总分比提升比例等考核项
private CountDownLatch countDownLatch = new CountDownLatch(2);

	// 分中心计算
	public void fzx(String fzxId) {
		System.out.println(fzxId + "正在计算...");
		try {
			Thread.sleep(1000L);
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		countDownLatch.countDown();
	}

	// 总比计算
	public void cal() {
		// 等待所有分中心计算完毕
		String name = Thread.currentThread().getName();
		System.out.println(name + "正在等待...");
		try {
			countDownLatch.await();
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		System.out.println("分中心计算完毕，开始计算");
	}

	public static void main(String[] args) {
		CountDownLatchDemo countDownLatchDemo = new CountDownLatchDemo();
		new Thread(() -> {
			countDownLatchDemo.fzx("100");
		}).start();
		new Thread(() -> {
			countDownLatchDemo.fzx("200");
		}).start();
		new Thread(() -> {
			countDownLatchDemo.cal();
		}).start();

	}
```

## CyclicBarrier源码解析和使用场景
```java

	private CyclicBarrier cyclicBarrier = new CyclicBarrier(3);

	// 考核项组数据生成 必须同时准备好才能一起同步
	public void check(String propertyGroup) {
		System.out.println(propertyGroup + "正在准备...");
		try {
			cyclicBarrier.await();
		} catch (InterruptedException | BrokenBarrierException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		System.out.println(propertyGroup + "正在同步...");
	}

	public static void main(String[] args) {
		CyclicBarrierDemo cyclicBarrierDemo = new CyclicBarrierDemo();
		new Thread(() -> {
			cyclicBarrierDemo.check("及时处置项");
		}).start();
		new Thread(() -> {
			cyclicBarrierDemo.check("超时项");
		}).start();
		new Thread(() -> {
			cyclicBarrierDemo.check("延期项");
		}).start();
	}
```

## Semphore源码解析和使用场景
- 控制对某组资源的访问权限、秒杀车位、流量控制,一组线程40个，每10个一组执行
```java
static class Work implements Runnable {
		private int workerNum;// 工号
		private Semaphore Semaphore;// 机器数量

		public Work(int workerNum, java.util.concurrent.Semaphore semaphore) {
			this.workerNum = workerNum;
			Semaphore = semaphore;
		}

		@Override
		public void run() {
			try {
				// 获取机器资源
				Semaphore.acquire();
				System.out.println(Thread.currentThread().getName()+"获取到机器，开始工作");
				Thread.sleep(1000L);
				Semaphore.release();
				System.out.println(Thread.currentThread().getName()+"工作完毕，释放机器");
			} catch (InterruptedException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			}

		}
	}

	public static void main(String[] args) {
		int workers = 8;
		Semaphore Semaphore = new Semaphore(3);
		for (int i = 0; i < workers - 1; i++) {
			new Thread(new Work(i,Semaphore)).start();
		}
	}
```

## AtomicBoolean、AtomicInteger、AtomicLong(基本类型)

```java
public class AtomicBooleanTest {
	private static AtomicBoolean initialized = new AtomicBoolean(false);

	public void init() {
		if (initialized.compareAndSet(false, true)) {
			System.out.println("这里放置初始化代码");
		}
	}

	public static void main(String[] args) {
		AtomicBooleanTest t = new AtomicBooleanTest();
		new Thread(() -> {
			t.init();
		}).start();
		new Thread(() -> {
			t.init();
		}).start();
	}
}
```
```java
public class AtomicIntegerTest {
	public static void main(String[] args) {
		 AtomicInteger atomicInteger = new AtomicInteger(123);
	        System.out.println(atomicInteger.get());
	        int expectedValue = 123;
	        int newValue      = 234;
	        Boolean b =atomicInteger.compareAndSet(expectedValue, newValue);
	        System.out.println(b);
	        System.out.println(atomicInteger);
	}
}
```
>LongAdder的基本思路就是分散热点，将value值的新增操作分散到一个数组中，不同线程会命中到数组的不同槽中，各个线程只对自己槽中的那个value值进行CAS操作，这样热点就被分散了，冲突的概率就小很多

```java
/**
 * Atomic和LongAdder耗时测试
 *
 */
public class AtomicLongAdderTest {
	public static void main(String[] args) throws Exception {
		testAtomicLongAdder(1, 10000000);
		testAtomicLongAdder(10, 10000000);
		testAtomicLongAdder(100, 10000000);
	}

	static void testAtomicLongAdder(int threadCount, int times) throws Exception {
		System.out.println("threadCount: " + threadCount + ", times: " + times);
		long start = System.currentTimeMillis();
		testLongAdder(threadCount, times);
		System.out.println("LongAdder 耗时：" + (System.currentTimeMillis() - start) + "ms");
		System.out.println("threadCount: " + threadCount + ", times: " + times);
		long atomicStart = System.currentTimeMillis();
		testAtomicLong(threadCount, times);
		System.out.println("AtomicLong 耗时：" + (System.currentTimeMillis() - atomicStart) + "ms");
		System.out.println("----------------------------------------");
	}

	static void testAtomicLong(int threadCount, int times) throws Exception {
		AtomicLong atomicLong = new AtomicLong();
		List<Thread> list = new ArrayList<Thread>();
		for (int i = 0; i < threadCount; i++) {
			list.add(new Thread(() -> {
				for (int j = 0; j < times; j++) {
					atomicLong.incrementAndGet();
				}
			}));
		}
		for (Thread thread : list) {
			thread.start();
		}
		for (Thread thread : list) {
			thread.join();
		}
		System.out.println("AtomicLong value is : " + atomicLong.get());
	}

	static void testLongAdder(int threadCount, int times) throws Exception {
		LongAdder longAdder = new LongAdder();
		List<Thread> list = new ArrayList<Thread>();
		for (int i = 0; i < threadCount; i++) {
			list.add(new Thread(() -> {
				for (int j = 0; j < times; j++) {
					longAdder.increment();
				}
			}));
		}
		for (Thread thread : list) {
			thread.start();
		}
		for (Thread thread : list) {
			thread.join();
		}
		System.out.println("LongAdder value is : " + longAdder.longValue());
	}
}
```

## AtomicIntegerArray、AtomicLongArray、AtomicReferenceArray（数组类型）



## AtomicReference、AtomicStampedRerence、AtomicMarkableReference（引用类型）



## AtomicIntegerFieldUpdater、AtomicLongFieldUpdater、AtomicReferenceFieldUpdater（属性原子修改器）
