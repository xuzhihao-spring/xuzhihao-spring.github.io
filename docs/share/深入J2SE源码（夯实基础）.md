# 深入J2SE源码（夯实基础）

## Map源码

### 1.7HashMap源码
### 1.8HashMap源码
### 红黑树原理
### 1.7ConcurrentHashMap源码
### 1.7ConcurrentHashMap源码
### LinkedHashMap源码

## List源码

### ArrayList
### LinkedList
### CopyOnWriteArrayList

## Set源码
### HashSet
### TreeSet
### LinkedHashSet

## Queue源码
BlockingQueue的核心方法：
```java
public interface BlockingQueue<E> extends Queue<E> {

    //将给定元素设置到队列中，如果设置成功返回true, 否则抛出异常。如果是往限定了长度的队列中设置值，推荐使用offer()方法。
    boolean add(E e);

    //将给定的元素设置到队列中，如果设置成功返回true, 否则返回false. e的值不能为空，否则抛出空指针异常。
    boolean offer(E e);

    //将元素设置到队列中，如果队列中没有多余的空间，该方法会一直阻塞，直到队列中有多余的空间。
    void put(E e) throws InterruptedException;

    //将给定元素在给定的时间内设置到队列中，如果设置成功返回true, 否则返回false.
    boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException;

    //从队列中获取值，如果队列中没有值，线程会一直阻塞，直到队列中有值，并且该方法取得了该值。
    E take() throws InterruptedException;

    //在给定的时间里，从队列中获取值，时间到了直接调用普通的poll方法，为null则直接返回null。
    E poll(long timeout, TimeUnit unit)
        throws InterruptedException;

    //获取队列中剩余的空间。
    int remainingCapacity();

    //从队列中移除指定的值。
    boolean remove(Object o);

    //判断队列中是否拥有该值。
    public boolean contains(Object o);

    //将队列中值，全部移除，并发设置到给定的集合中。
    int drainTo(Collection<? super E> c);

    //指定最多数量限制将队列中值，全部移除，并发设置到给定的集合中。
    int drainTo(Collection<? super E> c, int maxElements);
}
```

操作 | 抛出异常 | 特殊值 | 阻塞请求 | 超时
----|----|----|----|----
插入、添加 | add(e) | offer(e) | put(e) | offer(e, time, unit)
移除、获取 | remove() | poll() | take() | poll(time, unit)
检查 | element() | peek() | 不可用 | 不可用




### ArrayBlockingQueue 数组阻塞队列
- 数据结构：静态数组固定长度无扩容机制
- 存取同一把锁互斥 操作同一个对象
- 阻塞：出队为0时，无元素可取，阻塞在notEmpty 入队为数组长度时，无法刚入，阻塞在notFull
- 入队：队首添加元素记录putIndex 唤醒 notEmpty 
- 出队：队首获取元素记录takeIndex 唤醒 notFull
- 先进先出 读写互相排斥

### LinkedBlockingQueue 链表阻塞队列
- 数据结构：链表 可以指定容量默认Integer.MAX_VALUE
- 锁分离存取互不排斥takeLock、putLock
- 阻塞同ArrayBlockingQueue
- 入队：队尾入队 记录last节点
- 出队：队首出队 记录head节点
- 删除的时候两把锁一起加
- 先进先出
  
### SynchronousQueue 同步队列

### PriorityBlockingQueue 优先级阻塞队列 
- 小于64则翻倍，大于64增加一半 最小二叉堆
- 

### DelayQueue 延迟队列
  
### LinkedBlockingDeque 链表阻塞双端队列

### LinkedTransferQueue 由链表结构组成的无界阻塞

## JDK特性

### 语法糖

#### 默认构造器
```java
```
#### 自动拆装箱 JDK 5 
```java
```
#### 泛型集合取值 JDK 5 
```java
```
#### 可变参数 JDK 5 
```java
```
#### foreach 循环 JDK 5  能够配合数组，以及所有实现了 Iterable 接口的集合类一起使用
```java
```
#### switch 字符串 JDK 7 
```java
```
#### switch 枚举 JDK 7 
```java
```
#### 枚举类
```java
```
#### try-with-resources JDK 7  addSuppressed异常压制
```java
```
#### 方法重写时的桥接方法 子类返回值可以是父类返回值的子类
```java
```
#### 匿名内部类
```java
```

### JDK8

### JDK9

### JDK10

### JDK11

### JDK12

### JDK13