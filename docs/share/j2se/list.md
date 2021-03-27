
# List源码

## 1.1 ArrayList

## 1.2 LinkedList

## 1.3 CopyOnWriteArrayList

Redis4之后支持RDB-AOF混合持久化的方式

bgsave操作使用CopyOnWrite机制进行写时复制，是由一个子进程将内存中的最新数据遍历写入临时文件，此时父进程仍旧处理客户端的操作，当子进程操作完毕后再将该临时文件重命名为dump.rdb替换掉原来的dump.rdb文件，因此无论bgsave是否成功，dump.rdb都不会受到影响



### 1.3.1 CopyOnWrite容器实现原理

CopyOnWrite容器即写时复制的容器，是一种用于程序设计中的优化策略，同样属性的还有`CopyOnWriteArraySet`

写时复制（读写分离的思想）：读取时不加锁只是写入和删除时加锁实现机制（volatile + ReentrantLock）

往一个容器添加元素的时候，不直接往当前容器添加，而是先将当前容器进行Copy，复制出一个新的容器，然后新的容器里添加元素，添加完元素之后，再将原容器的引用指向新的容器

CopyOnWrite容器只能保证数据的最终一致性，不能保证数据的实时一致性。所以如果你希望写入的的数据，马上能读到，请不要使用CopyOnWrite容器。【当执行add或remove操作没完成时，get获取的仍然是旧数组的元素】


### 1.3.2 初始化

```java
public CopyOnWriteArrayList() {
    setArray(new Object[0]);    //创建一个大小为0的Object数组作为array初始值
}
public CopyOnWriteArrayList(E[] toCopyIn) {
    //创建一个list，其内部元素是toCopyIn的的副本
    setArray(Arrays.copyOf(toCopyIn, toCopyIn.length, Object[].class));
}
//将传入参数集合中的元素复制到本list中
public CopyOnWriteArrayList(Collection<? extends E> c) {
    Object[] elements;
    if (c.getClass() == CopyOnWriteArrayList.class)
        elements = ((CopyOnWriteArrayList<?>)c).getArray();
    else {
        elements = c.toArray();
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elements.getClass() != Object[].class)
            elements = Arrays.copyOf(elements, elements.length, Object[].class);
    }
    setArray(elements);
}
```

### 1.3.2 添加元素

`add(E e)` 往集合末尾添加一个元素

```java
public boolean add(E e) {
    // 使用juc的可重入锁保证线程安全
    // 其中lock在CopyOnWriteArrayList的定义：final transient ReentrantLock lock = new ReentrantLock();
    final ReentrantLock lock = this.lock;
    // 线程得到锁，同时只能有一个线程操作
    lock.lock();
    try {
        // 获取当前的数组，和ArrayList一样。底层就是一个数组。
        Object[] elements = getArray();
        // 得到数组的长度
        int len = elements.length;
        // 直接copy出一个新数组，传入的参数是旧数组和新数组的容量。返回一个新数组，新数组数据就是就数组的数组，并且新数组的容量+1
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        // 往elements.length添加一个新元素。也就是末尾添加一个元素。
        newElements[len] = e;
        // 用新数组覆盖旧数组
        setArray(newElements);
        // 添加成功，返回true
        return true;
    } finally {
        // 当前线程操作完成，释放锁。
        lock.unlock();
    }
}
```

`add(int index, E element)` 往集合指定位置添加一个元素
```java
public void add(int index, E element) {
    // 上面add(E e)一样的逻辑就不再赘述
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        // 判断插入的位置是否合法。
        if (index > len || index < 0)
            throw new IndexOutOfBoundsException("Index: "+index+", Size: "+len);
        Object[] newElements;
        int numMoved = len - index;
        // 如果插入的是数组的最后一个位置，直接copy一个新数组，并且新数组的容量+1
        if (numMoved == 0)
            newElements = Arrays.copyOf(elements, len + 1);
        else {
            // 如果插入的不是最后一个位置，则new一个新数组，新数组的容量在原来的旧数组容量上+1
            newElements = new Object[len + 1];
            // 把旧数组直接拷贝到空的新数组中。
            // System.arraycopy参数解释：
            // 第一个参数是需要被拷贝的旧数组，第二个参数从旧数组的什么位置开始拷贝，
            // 第三个参数是拷贝到的目标数组，第四个参数是从旧数组的第几个参数开始拷贝，
            // 第五个参数是从旧数组拷贝多少个元素到新数组。
            // 第一次拷贝：把旧数组拷贝到新数组，把要插入的位置的index之前的数据拷贝到新数组。
            System.arraycopy(elements, 0, newElements, 0, index);
            // 第二次拷贝：把旧数组从要插入的位置开始拷贝到新数组要插入的位置+1开始，把剩下的旧数组数据拷贝到新数组。
            // 这里index+1就在新数组中预留了一个插入位置。
            System.arraycopy(elements, index, newElements, index + 1, numMoved);
        }
        // 往index的位置插入元素
        newElements[index] = element;
        // 覆盖旧数组
        setArray(newElements);
    } finally {
        // 线程执行完毕，解锁
        lock.unlock();
    }
}
```

### 1.3.3 修改元素

```java
public E set(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock();    //加锁
    try {
        Object[] elements = getArray();
        E oldValue = get(elements, index);  //先得到要修改的旧值

        if (oldValue != element) {  //值确实修改了
            int len = elements.length;
            //将array复制到新数组，并进行修改，并设置array为新数组
            Object[] newElements = Arrays.copyOf(elements, len);			
            newElements[index] = element;
            setArray(newElements);
        } else {
            // 虽然值确实没改，但要保证volatile语义，需重新设置array
            setArray(elements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```

### 1.3.4 删除元素

```java
public E remove(int index) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        E oldValue = get(elements, index);    //得到要删除的元素
        int numMoved = len - index - 1;
        if (numMoved == 0)
            setArray(Arrays.copyOf(elements, len - 1));
        else {
            Object[] newElements = new Object[len - 1];
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index + 1, newElements, index,
                                numMoved);
            setArray(newElements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```

### 1.3.5 迭代器的弱一致性

弱一致性是指返回迭代器后，其它线程对list的增删改对迭代器是不可见的，迭代器返回对象是快照

```java
public Iterator<E> iterator() {
        return new COWIterator<E>(getArray(), 0);
}
static final class COWIterator<E> implements ListIterator<E> {
        //array的快照
        private final Object[] snapshot;
        //数组下标
        private int cursor;
        private COWIterator(Object[] elements, int initialCursor) {
            cursor = initialCursor;
            snapshot = elements;
        }
        public boolean hasNext() {
            return cursor < snapshot.length;
        }
        public boolean hasPrevious() {
            return cursor > 0;
        }
        @SuppressWarnings("unchecked")
        public E next() {
            if (! hasNext())
                throw new NoSuchElementException();
            return (E) snapshot[cursor++];
        }
}
```