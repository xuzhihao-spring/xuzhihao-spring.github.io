# Set源码

## HashSet
HashMap初始化容量大小与阿里巴巴Java开发手册约定计算公式一致

即initialCapacity = (需要存储的元素个数 / 负载因子) + 1

```java
public HashSet(Collection<? extends E> c) {
    map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
    addAll(c);
}
```
## TreeSet

## LinkedHashSet
