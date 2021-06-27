# 设计模式

对修改关闭，对扩展开放

## 1. 单例模式Singleton

**目的：对象只被创建一次。**
1. 懒汉式，双重判断，synchronized影响了效率
2. 静态内部方式，jvm保证只会加载一次
3. 枚举方式，解决同步，防止反序列化，因为`枚举类没有构造方法`
  
```java
package com.xuzhihao.design.mode.singleton;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;

/**
 * 使用单例模式创建JDBC工具类 构造器私有化，自行创建实例，自行向系统提供这个实例
 */
//饿汉式：直接创建对象 不存在线程安全问题
//	 直接实例化
//	 枚举式
//	 静态代码块
//懒汉式：延迟创建对象
//	线程不安全 适用于单线程
//	线程安全 适用于多线程
//	静态内部类 适用于多线程
public final class JdbcUtilsSingleton {

	private static ThreadLocal<Connection> tl = new ThreadLocal<>();

	private static final String driver = "com.mysql.cj.jdbc.Driver";// mysql驱动
//	private static final String driver = "oracle.jdbc.driver.OracleDriver"; // oracle驱动
//	private static final String driver = "org.postgresql.Driver"; // postgresql驱动
//	private static final String driver = "com.microsoft.sqlserver.jdbc.SQLServerDriver"; // sqlserver驱动

	private static final String url = "jdbc:mysql://debug-registry:3306/mall";// mysql地址
//	private static final String url = "jdbc:oracle:thin:@debug-registry:1521:ORCL"; // oracle地址
//	private static final String url = "jdbc:postgresql://debug-registry:5432/VJSP10003182"; // postgresql地址
//	private static final String url = "jdbc:sqlserver://debug-registry:1433; DatabaseName=maptest"; // sqlserver地址

	private String username = "root";
	private String password = "root";

	// 懒汉式单例
	private static JdbcUtilsSingleton singleton;

	/**
	 * 构造器私用,防止直接创建对象, 通过反射或反序列化可以破解单例
	 */
	private JdbcUtilsSingleton() {

	}

	public static JdbcUtilsSingleton getInstance() {
		if (singleton == null) {
			// 加入有一个线程走到这里,然后又然给另一个线程执行完
			synchronized (JdbcUtilsSingleton.class) {
				if (singleton == null) {
					singleton = new JdbcUtilsSingleton();
				}
			}
		}
		return singleton;
	}

	static {
		try {
			Class.forName(driver);
		} catch (ClassNotFoundException e) {
			throw new ExceptionInInitializerError(e);
		}
	}

	/**
	 * 获取连接
	 * 
	 * @return
	 * @throws SQLException
	 */
	public synchronized Connection getConnection() throws SQLException {
		Connection conn = tl.get();
		// 如果容器中没有连接，就从连接池获取一个连接存到ThreadLocal中
		if (conn == null) {
			conn = DriverManager.getConnection(url, username, password);
			tl.set(conn);
		}
		return conn;
	}

	/**
	 * 释放资源
	 */
	public static void free(AutoCloseable... ios) {
		for (AutoCloseable io : ios) {
			try {
				if (io != null) {
					io.close();
				}
			} catch (Exception e) {
				e.printStackTrace();
			} finally {
				try {
					if (io != null) {
						io.close();
					}
				} catch (Exception e) {
					e.printStackTrace();
				} finally {
					if (io != null) {
						try {
							io.close();
						} catch (Exception e) {
							e.printStackTrace();
						}
					}
				}
			}
		}
	}
}
```

```java
package com.xuzhihao;

import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;

import com.xuzhihao.design.mode.singleton.JdbcUtilsSingleton;

public class JdbcTest {
	public static void main(String[] args) throws SQLException {

		// 首先需要获取实例,然后获取连接
		Connection conn = JdbcUtilsSingleton.getInstance().getConnection();
		// 创建语句
		Statement st = conn.createStatement();
		// 执行语句
		ResultSet rs = st.executeQuery("select * from ums_admin");
		// 处理结果集
		while (rs.next()) {
			System.out.println(
					rs.getObject(1) + "\t" + rs.getObject(2) + "\t" + rs.getObject(3) + "\t" + rs.getObject(4));
		}
		st.executeBatch();
		st.close();

		//批量插入
		String sql = "INSERT INTO ums_admin2 (username,password) VALUES(?,?)";
		PreparedStatement pstmt = conn.prepareStatement(sql);
		for (int i = 1; i <= 10; i++) {
			pstmt.setString(1, "username" + i);
			pstmt.setString(2, "password" + i);
			pstmt.addBatch();
		}
		pstmt.executeBatch();
		pstmt.close();
		conn.close();
	}
}
```
```java
/**
 * 静态内部类方式 JVM保证单例
 */
public class Sington {
	private Sington() {

	}

	private static class SingtonHolder {
		private final static Sington INSTANCE = new Sington();
	}

	public static Sington getInstance() {
		return SingtonHolder.INSTANCE;
	}

	public static void main(String[] args) {
		for (int i = 0; i < 10; i++) {
			new Thread(() -> {
				System.out.println(Sington.getInstance().hashCode());
			}).start();
		}
	}
}
```
```java
public enum Sington {

	INSTANCE;

	public static void main(String[] args) {
		for (int i = 0; i < 10; i++) {
			new Thread(() -> {
				System.out.println(Sington.INSTANCE.hashCode());
			}).start();
		}
	}
}
```

## 2. 策略模式Strategy

策略模式属于对象的行为模式。其用意是针对一组算法，将每一个算法封装到具有共同接口的独立的类中，从而使得它们可以相互替换。策略模式使得算法可以在不影响到客户端的情况下发生变化

策略模式是对算法的包装，是把使用算法的责任和算法本身分割开来，委派给不同的对象管理。策略模式通常把一个系列的算法包装到一系列的策略类里面，作为一个抽象策略类的子类

`缺点`：客户端必须知道所有策略的详细实现细节，使用ifelse语法进行选择性的注入对应实现类，代码如下

定义策略接口
```java
import java.math.BigDecimal;

/**
 * 优惠券类型；
 * 1. 直减券  2. 满减券 3. 折扣券  4. n元购
 * 1. 限时折扣 2. 限量折扣 3. 某类商品折扣 4. 某个商品折扣
 */
public interface CouponDiscountService {

	/**
	 * 优惠券折扣计算接口
	 */
	public BigDecimal discountAmount(BigDecimal orderPrice);
}
```

模拟折扣和优惠的实现
```java
public class ZKCouponDiscount implements UserPayDiscountService {

    /**
     * 折扣计算
     * 1. 使用商品价格乘以折扣比例，为最后支付金额
     * 2. 保留两位小数
     * 3. 最低支付金额1元
     */
    public BigDecimal discountAmount(Double couponInfo, BigDecimal skuPrice) {
        BigDecimal discountAmount = skuPrice.multiply(new BigDecimal(couponInfo)).setScale(2, BigDecimal.ROUND_HALF_UP);
        if (discountAmount.compareTo(BigDecimal.ZERO) < 1) return BigDecimal.ONE;
        return discountAmount;
    }
}

public class ZJCouponDiscount implements UserPayDiscountService {

    /**
     * 直减计算
     * 1. 使用商品价格减去优惠价格
     * 2. 最低支付金额1元
     */
    public BigDecimal discountAmount(Double couponInfo, BigDecimal skuPrice) {
        BigDecimal discountAmount = skuPrice.subtract(new BigDecimal(couponInfo));
        if (discountAmount.compareTo(BigDecimal.ZERO) < 1) return BigDecimal.ONE;
        return discountAmount;
    }
}
```

下单
```java
public BigDecimal calPrice(BigDecimal orderPrice,User user) {

     if (直减) {
        //伪代码：从Spring中获取策略对象
        CouponDiscountService strategy = Spring.getBean(ZJCouponDiscount.class);
        return strategy.discountAmount(orderPrice);
     }

     if (折扣) {
        CouponDiscountService strategy = Spring.getBean(ZKCouponDiscount.class);
        return strategy.discountAmount(orderPrice);
     }

     return 原价;
}
```

借助Spring和工厂模式，为了方便我们从Spring中获取CouponDiscountService的各个策略类，我们创建一个工厂类

```java
public class  CouponDiscountServiceStrategyFactory {

    private static Map<String, CouponDiscountService> services = new ConcurrentHashMap<String, CouponDiscountService>();

    public  static  CouponDiscountService getByDiscountType(String type){
        return services.get(type);
    }
    //通过每个实现类初始化的时候调用注册
    public static void register(String userType, CouponDiscountService  CouponDiscountService){
        Assert.notNull(userType,"userType can't be null");
        services.put(userType, CouponDiscountService);
    }
}
```

初始化策略类,`注意`此处也可使用spring的Aware机制进行实例注册
```java
@Service
public class ZKCouponDiscount implements UserPayDiscountService,InitializingBean {

    /**
     * 折扣计算
     * 1. 使用商品价格乘以折扣比例，为最后支付金额
     * 2. 保留两位小数
     * 3. 最低支付金额1元
     */
    public BigDecimal discountAmount(Double couponInfo, BigDecimal skuPrice) {
        BigDecimal discountAmount = skuPrice.multiply(new BigDecimal(couponInfo)).setScale(2, BigDecimal.ROUND_HALF_UP);
        if (discountAmount.compareTo(BigDecimal.ZERO) < 1) return BigDecimal.ONE;
        return discountAmount;
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        CouponDiscountServiceStrategyFactory.register("zKCouponDiscount",this);
    }
}
```

下单逻辑简化
```java
public Order calPrice(Order order) {

     String discountType = order.discountType();//获取折扣类型
     CouponDiscountService strategy = CouponDiscountServiceStrategyFactory.getByDiscountType(discountType);
     return strategy.discountAmount(order);
}
```

这并不是严格意义上的策略模式，因为没用使用到`Context`,而使用工厂模式给代替了，Context的存在的意思在于可以对返回的对应进行2次包装，再次减少因为修改策略而对下单逻辑代码进行的改造
```java
public class Context<T> {

    private ICouponDiscount couponDiscount;

    public Context(ICouponDiscount couponDiscount) {
        this.couponDiscount = couponDiscount;
    }

    public Order discountAmount(Order order) {
        return couponDiscount.discountAmount(order);
    }

}
```

```java
public Order calPrice(Order order) {
     String discountType = order.discountType();//获取折扣类型
     CouponDiscountService strategy = CouponDiscountServiceStrategyFactory.getByDiscountType(discountType);
     Context context = new Context(strategy);
     return context.discountAmount(order);
}

```


## 3. 工厂方法模式

产生对象的类和方法都可以叫做工厂，目的是灵活控制生产过程，例如`产品维度扩展`

## 4. 抽象工厂模式

形容词用接口，名词用抽象类，例如`产品一族扩展`,`一键换肤`

## 5. 中介者模式（行为型） 

是一种`行为设计模式`，能让你减少对象之间混乱无序的依赖和调用关系，该模式会限制对象之间的直接交互，迫使他们通过一个中间对象来进行合作，以达到解耦的目的。

```java
public class MyOneTask extends TimerTask {
    private static int num = 0;
    @Override
    public void run() {
        System.out.println("I'm MyOneTask " + ++num);
    }
}

public class MyTwoTask extends TimerTask {
    private static int num = 1000;
    @Override
    public void run() {
        System.out.println("I'm MyTwoTask " + num--);
    }
}
```

```java
public class TimerTest {
    public static void main(String[] args) {
        // 注意：多线程并行处理定时任务时，Timer运行多个TimeTask时，只要其中之一没有捕获抛出的异常，
        // 其它任务便会自动终止运行，使用ScheduledExecutorService则没有这个问题
        Timer timer = new Timer();
        timer.schedule(new MyOneTask(), 3000, 1000); // 3秒后开始运行，循环周期为 1秒
        timer.schedule(new MyTwoTask(), 3000, 1000);
    }
}
```

`Timer`的部分关键源码如下

```java
public class Timer {

    private final TaskQueue queue = new TaskQueue();
    private final TimerThread thread = new TimerThread(queue);
    
    public void schedule(TimerTask task, long delay) {
        if (delay < 0)
            throw new IllegalArgumentException("Negative delay.");
        sched(task, System.currentTimeMillis()+delay, 0);
    }
    
    public void schedule(TimerTask task, Date time) {
        sched(task, time.getTime(), 0);
    }
    
    private void sched(TimerTask task, long time, long period) {
        if (time < 0)
            throw new IllegalArgumentException("Illegal execution time.");

        if (Math.abs(period) > (Long.MAX_VALUE >> 1))
            period >>= 1;

        // 获取任务队列的锁(同一个线程多次获取这个锁并不会被阻塞,不同线程获取时才可能被阻塞)
        synchronized(queue) {
            // 如果定时调度线程已经终止了,则抛出异常结束
            if (!thread.newTasksMayBeScheduled)
                throw new IllegalStateException("Timer already cancelled.");

            // 再获取定时任务对象的锁(为什么还要再加这个锁呢?想不清)
            synchronized(task.lock) {
                // 判断线程的状态,防止多线程同时调度到一个任务时多次被加入任务队列
                if (task.state != TimerTask.VIRGIN)
                    throw new IllegalStateException(
                        "Task already scheduled or cancelled");
                // 初始化定时任务的下次执行时间
                task.nextExecutionTime = time;
                // 重复执行的间隔时间
                task.period = period;
                // 将定时任务的状态由TimerTask.VIRGIN(一个定时任务的初始化状态)设置为TimerTask.SCHEDULED
                task.state = TimerTask.SCHEDULED;
            }
            
            // 将任务加入任务队列
            queue.add(task);
            // 如果当前加入的任务是需要第一个被执行的(也就是他的下一次执行时间离现在最近)
            // 则唤醒等待queue的线程(对应到上面提到的queue.wait())
            if (queue.getMin() == task)
                queue.notify();
        }
    }
    
    // cancel会等到所有定时任务执行完后立刻终止定时线程
    public void cancel() {
        synchronized(queue) {
            thread.newTasksMayBeScheduled = false;
            queue.clear();
            queue.notify();  // In case queue was already empty.
        }
    }
    // ...
}
```

Timer 是中介者

## 6. 门面模式

门面（Facade）模式也被称为正面模式、外观模式，这种模式用于将一组复杂的类包装到一个简单的外部接口中。

例如：

| 日志门面（抽象层）| 日志实现 |
| ----- | ----- |
| JCL，SLF4j，Jboss-logging| JUL，log4j，log4j2，logback |

## 7. 装饰模式

装饰者模式(Decorator Pattern)：动态地给一个对象增加一些额外的职责，增加对象功能来说，装饰模式比生成子类实现更为灵活。装饰模式是一种对象结构型模式。

在装饰者模式中，为了让系统具有更好的灵活性和可扩展性，我们通常会定义一个抽象装饰类，而将具体的装饰类作为它的子类

源码分析装饰者模式的典型应用：

### 7.1 Java I/O中的装饰者模式
   
使用 Java I/O 的时候总是有各种输入流、输出流、字符流、字节流、过滤流、缓冲流等等各种各样的流，不熟悉里边的设计模式的话总会看得云里雾里的，现在通过设计模式的角度来看 Java I/O，Java中应用程序通过输入流（InputStream）的Read方法从源地址处读取字节，然后通过输出流（OutputStream）的Write方法将流写入到目的地址。

流的来源主要有三种：本地的文件（File）、控制台、通过socket实现的网络通信

下面的图可以看出Java中的装饰者类和被装饰者类以及它们之间的关系，这里只列出了InputStream中的关系：

![](../../images/share/designpattern/design/design17_1.png)

由上图可以看出只要继承了FilterInputStream的类就是装饰者类，可以用于包装其他的流，装饰者类还可以对装饰者和类进行再包装

下面看一下Java中包装流的实例：

```java
import java.io.BufferedInputStream;
import java.io.DataInputStream;
import java.io.FileInputStream;
import java.io.IOException;
public class StreamDemo {
    public static void main(String[] args) throws IOException{
        DataInputStream in=new DataInputStream(new BufferedInputStream(new  FileInputStream("D:\\hello.txt")));
        while(in.available()!=0) {
            System.out.print((char)in.readByte());
        }
        in.close();
    }
}
```

输出结果

```bash
hello world!
hello Java I/O!
```

上面程序中对流进行了两次包装，先用 BufferedInputStream将FileInputStream包装成缓冲流也就是给FileInputStream增加缓冲功能，再DataInputStream进一步包装方便数据处理

如果要实现一个自己的包装流，根据上面的类图，需要继承抽象装饰类 FilterInputStream,譬如来实现这样一个操作的装饰者类：将输入流中的所有小写字母变成大写字母

```java
import java.io.FileInputStream;
import java.io.FilterInputStream;
import java.io.IOException;
import java.io.InputStream;

public class UpperCaseInputStream extends FilterInputStream {
    protected UpperCaseInputStream(InputStream in) {
        super(in);
    }

    @Override
    public int read() throws IOException {
        int c = super.read();
        return (c == -1 ? c : Character.toUpperCase(c));
    }

    @Override
    public int read(byte[] b, int off, int len) throws IOException {
        int result = super.read(b, off, len);
        for (int i = off; i < off + result; i++) {
            b[i] = (byte) Character.toUpperCase((char) b[i]);
        }
        return result;
    }

    public static void main(String[] args) throws IOException {
        int c;
        InputStream in = new UpperCaseInputStream(new FileInputStream("D:\\hello.txt"));
        try {
            while ((c = in.read()) >= 0) {
                System.out.print((char) c);
            }
        } finally {
            in.close();
        }
    }
}
```

整个Java IO体系都是基于字符流(InputStream/OutputStream) 和 字节流(Reader/Writer)作为基类，下面画出OutputStream、Reader、Writer的部分类图

OutputStream类图

![](../../images/share/designpattern/design/design17_2.png)

Reader类图

![](../../images/share/designpattern/design/design17_3.png)

Writer类图

![](../../images/share/designpattern/design/design17_4.png)

### 7.2 spring cache 中的装饰者模式

看 `org.springframework.cache.transaction` 包下的 `TransactionAwareCacheDecorator` 这个类

```java
public class TransactionAwareCacheDecorator implements Cache {
    private final Cache targetCache;
    
    public TransactionAwareCacheDecorator(Cache targetCache) {
        Assert.notNull(targetCache, "Target Cache must not be null");
        this.targetCache = targetCache;
    }
    
    public <T> T get(Object key, Class<T> type) {
        return this.targetCache.get(key, type);
    }

    public void put(final Object key, final Object value) {
        // 判断是否开启了事务
        if (TransactionSynchronizationManager.isSynchronizationActive()) {
            // 将操作注册到 afterCommit 阶段
            TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
                public void afterCommit() {
                    TransactionAwareCacheDecorator.this.targetCache.put(key, value);
                }
            });
        } else {
            this.targetCache.put(key, value);
        }
    }
    // ...省略...
}
```

该类实现了`Cache`接口，同时将Cache组合到类中成为了成员属性`targetCache`，所以可以大胆猜测`TransactionAwareCacheDecorator`是一个装饰类，不过这里并没有抽象装饰类，且`TransactionAwareCacheDecorator`没有子类，这里的装饰类关系并没有Java I/O 中的装饰关系那么复杂

spring cache中类图关系

![](../../images/share/designpattern/design/design17_5.png)

该类的主要功能：通过 Spring 的`TransactionSynchronizationManager`将其`put/evict/clear`操作与 Spring 管理的事务同步，仅在成功的事务的`after-commit`阶段执行实际的缓存`put/evict/clear`操作。如果没有事务是`active`的，将立即执行`put/evict/clear`操作

### 7.3 spring session 中的装饰者模式

>注意：适配器模式的结尾也可能是 Wrapper

类 ServletRequestWrapper 的代码如下：

```java
public class ServletRequestWrapper implements ServletRequest {
    private ServletRequest request;
    
    public ServletRequestWrapper(ServletRequest request) {
        if (request == null) {
            throw new IllegalArgumentException("Request cannot be null");
        }
        this.request = request;
    }
    
    @Override
    public Object getAttribute(String name) {
        return this.request.getAttribute(name);
    }
    //...省略...
}    
```

可以看到该类对 ServletRequest 进行了包装，这里是一个装饰者模式，再看下图，spring session 中 SessionRepositoryFilter 的一个内部类 SessionRepositoryRequestWrapper 与 ServletRequestWrapper 的关系

![](../../images/share/designpattern/design/design17_6.png)

可见 ServletRequestWrapper 是第一层包装，HttpServletRequestWrapper 通过继承进行包装，增加了 HTTP 相关的功能，SessionRepositoryRequestWrapper 又通过继承进行包装，增加了 Session 相关的功能

###  7.4 Mybatis 缓存中的装饰者模式

`org.apache.ibatis.cache` 包的文件结构如下所示

![](../../images/share/designpattern/design/design17_7.png)

我们通过类所在的包名即可判断出该类的角色，Cache 为抽象构件类，PerpetualCache 为具体构件类，decorators 包下的类为装饰类，没有抽象装饰类

通过名称也可以判断出装饰类所要装饰的功能

总结

**装饰模式的主要优点如下：**

1. 对于扩展一个对象的功能，装饰模式比继承更加灵活性，不会导致类的个数急剧增加。
2. 可以通过一种动态的方式来扩展一个对象的功能，通过配置文件可以在运行时选择不同的具体装饰类，从而实现不同的行为。
3. 可以对一个对象进行多次装饰，通过使用不同的具体装饰类以及这些装饰类的排列组合，可以创造出很多不同行为的组合，得到功能更为强大的对象。
4. 具体构件类与具体装饰类可以独立变化，用户可以根据需要增加新的具体构件类和具体装饰类，原有类库代码无须改变，符合 “开闭原则”。

**装饰模式的主要缺点如下：**

1. 使用装饰模式进行系统设计时将产生很多小对象，这些对象的区别在于它们之间相互连接的方式有所不同，而不是它们的类或者属性值有所不同，大量小对象的产生势必会占用更多的系统资源，在一定程序上影响程序的性能。
2. 装饰模式提供了一种比继承更加灵活机动的解决方案，但同时也意味着比继承更加易于出错，排错也很困难，对于多次装饰的对象，调试时寻找错误可能需要逐级排查，较为繁琐。

**适用场景：**

1. 在不影响其他对象的情况下，以动态、透明的方式给单个对象添加职责。
2. 当不能采用继承的方式对系统进行扩展或者采用继承不利于系统扩展和维护时可以使用装饰模式。不能采用继承的情况主要有两类：第一类是系统中存在大量独立的扩展，为支持每一种扩展或者扩展之间的组合将产生大量的子类，使得子类数目呈爆炸性增长；第二类是因为类已定义为不能被继承（如Java语言中的final类）

## 8. 适配器模式（结构型）

适配器模式（Adapter Pattern）是作为两个不兼容的接口之间的桥梁

- tomcat CoyoteAdapter

## 9. 原型模式

## 10. 建造者模式

## 11. 结构型模式

## 12. 代理模式

## 13. 桥接模式

## 14. 享元模式

享元模式（Flyweight Pattern）主要用于减少创建对象的数量，以减少内存占用和提高性能。

## 15. 组合模式

## 16. 行为型模式

## 17. 模板方法模式

- tomcat LifecycleBase initInternal startInternal

## 18. 命令模式

## 19. 责任链模式

## 20. 状态模式

## 21. 观察者模式

## 22. 迭代器模式

## 23. 访问者模式

## 24. 备忘录模式

## 25. 解释器模式
