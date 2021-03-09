
# JDK特性

## 语法糖

### 默认构造器
```java
```
### 自动拆装箱 JDK 5 
```java
```
### 泛型集合取值 JDK 5 
```java
```
### 可变参数 JDK 5 

jdbc释放链接

```java
//释放资源
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
```
### foreach 循环 JDK 5  能够配合数组，以及所有实现了 Iterable 接口的集合类一起使用
```java
```
### switch 字符串 JDK 7 
```java
```
### switch 枚举 JDK 7 
```java
```
### 枚举类
```java
```
### try-with-resources JDK 7  addSuppressed异常压制
```java
```
### 方法重写时的桥接方法 子类返回值可以是父类返回值的子类
```java
```
### 匿名内部类
```java
```

## JDK8

### Stream使用和底层原理详解
### 函数式接口与Lambda 表达式详解
### 方法引用与Data API详解
### 其他特性详解

## JDK9

### modularity System 模块化系统
### HTTP/2详解
### JShell : 交互式 Java REPL
### 不可变集合工厂方法、私有接口方法详解
### 多版本兼容 JAR与I/O 流新特性

## JDK10

### 局部变量的类型推断
### GC改进和内存管理
### 线程本地握手
### 备用内存设备上的堆分配
### 其他Unicode语言 - 标记扩展
### 基于Java的实验性JIT编译器
### 开源根证书
### 根证书颁发认证（CA）
### 将JDK生态整合单个存储库
### 删除工具javah

## JDK11

### HTTP Client 详解
### Java Flight Recorder (JFR)
### JavaFX从JDK分离为独立模块
### Project ZGC

## JDK12

### Shenandoah
### Microbenchmark Suite
### JVM常量API
### 默认CDS档案
### G1的优化详解

## JDK13

### Dynamic CDS Archives
### ZGC详解
### Reimplement the Legacy Socket API
### Switch Expressions
### Text Blocks