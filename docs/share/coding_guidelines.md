# Java 开发手册

Java 开发规约IDE插件 https://github.com/alibaba/p3c

## 1. 编程规约

## 2. 异常日志

## 3. 单元测试

## 4. 安全规约

## 5. 数据库

## 6. 工程结构

## 7. 设计结构

## 8. 名词解释

1. `CAS`（Compare And Swap）: 。解决多线程并行情况下使用锁造成性能损耗的一种机制，这是硬件实现的原子操作。CAS 操作包含三个操作数：内存位置、预期原值和新值。如果内存位置的值与预期原值相匹配，那么处理器会自动将该位置值更新为新值。否则，处理器不做任何操作。
2. `DO`（Data Object）: 阿里巴巴专指数据库表一一对应的 POJO 类。
3. `GAV`（GroupId、ArtifactId、Version）: Maven 坐标，是用来唯一标识 jar 包。
4. `OOP`（Object Oriented Programming）: 本文泛指类、对象的编程处理方式。
5. `AQS`（AbstractQueuedSynchronizer）: 利用先进先出队列实现的底层同步工具类，它是很多上层同步实现类的基础，比如：ReentrantLock、CountDownLatch、Semaphore 等，它们通过继承 AQS 实现其模版方法，然后将 AQS 子类作为同步组件的内部类，通常命名为 Sync。
6. `ORM`（Object Relation Mapping）: 对象关系映射，对象领域模型与底层数据之间的转换，本文泛指 iBATIS, mybatis 等框架。
7. `POJO`（Plain Ordinary Java Object）: 在本规约中，POJO 专指只有 setter/getter/toString 的简单类，包括 DO/DTO/BO/VO 等。
8. `AO`（Application Object）: 阿里巴巴专指 Application Object，即在 Service 层上，极为贴近业务的复用代码。
9. `NPE`（java.lang.NullPointerException）: 空指针异常。
10. `OOM`（Out Of Memory）: 源于 java.lang.OutOfMemoryError，当 JVM 没有足够的内存来为对象分配空间并且垃圾回收器也无法回收空间时，系统出现的严重状况。
11. `一方库`: 本工程内部子项目模块依赖的库（jar 包）。
12. `二方库`: 公司内部发布到中央仓库，可供公司内部其它应用依赖的库（jar 包）。
13. `三方库`: 公司之外的开源库（jar 包）。

## 9. 错误码列表