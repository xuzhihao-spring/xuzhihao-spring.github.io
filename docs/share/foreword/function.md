#  算法

## 1. Unsafe的使用

```java

package com.xuzhihao.test;
import java.lang.reflect.Field;
import sun.misc.Unsafe;
public class UnsafeTest {

	private int i = 0;
	private static sun.misc.Unsafe unsafe;
	private static long offset;
	private String[] table = { "1", "2", "3", "4" };
	static {
		try {
			Field field = Unsafe.class.getDeclaredField("theUnsafe");
			field.setAccessible(true);
			unsafe = (Unsafe) field.get(null);
			offset = unsafe.objectFieldOffset(UnsafeTest.class.getDeclaredField("i"));
		} catch (Exception e) { // Internal reference
			e.printStackTrace();
		}
	}
    //unsafe方式修改数组中值
	public static void main2(String[] args) {
		final UnsafeTest po = new UnsafeTest();
		// 数组中存储的对象的对象头大小
		int ns = unsafe.arrayIndexScale(String[].class);
		// 数组中第一个元素的起始位置
		int base = unsafe.arrayBaseOffset(String[].class);
		System.out.println(unsafe.getObject(po.table, base + 3 * ns));
	}
    //多线程并发修改
	public static void main(String[] args) {
		final UnsafeTest po = new UnsafeTest();
		new Thread(new Runnable() {
			@Override
			public void run() {
				while (true) {
					boolean b = unsafe.compareAndSwapInt(po, offset, po.i, po.i + 1);
					if (b)
						System.out.println(unsafe.getIntVolatile(po, offset));
					try {
						Thread.sleep(500);
					} catch (InterruptedException e) {
						// TODO Auto-generated catch block
						e.printStackTrace();
					}
				}
			}
		}).start();
		new Thread(new Runnable() {
			@Override
			public void run() {
				while (true) {
					boolean b = unsafe.compareAndSwapInt(po, offset, po.i, po.i + 1);
					if (b)
						System.out.println(unsafe.getIntVolatile(po, offset));
					try {
						Thread.sleep(500);
					} catch (InterruptedException e) {
						// TODO Auto-generated catch block
						e.printStackTrace();
					}
				}
			}
		}).start();
	}
}
```

## 2. 排序算法

### 2.1 冒泡排序

取第一个元素与后一个比较，如果大于后者，就与后者互换位置，不大于，就保持位置不变。再拿第二个元素与后者比较，如果大于后者，就与后者互换位置。一轮比较之后，最大的元素就移动到末尾。相当于最大的就冒出来了。再进行第二轮，第三轮，直到排序完毕

冒泡排序：比较 （N-1)+(N-2)+...+2+1 = N*(N-1)/2=N2/2

　　　　　交换  0——N2/2 = N2/4

　　　　　总时间 3/4*N2

```java
/**
* 冒泡排序
*/
public static void bubble() {
	int[] arr = { 10, 6, 3, 2, 1, 7 };
	for (int i = 0; i < arr.length - 1; i++) {// 外层循环n-1
		for (int j = 0; j < arr.length - i - 1; j++) {// 内层循环n-i-1
			if (arr[j] > arr[j + 1]) {// 从第一个开始，往后两两比较大小，如果前面的比后面的大，交换位置
				int tmp = arr[j];
				arr[j] = arr[j + 1];
				arr[j + 1] = tmp;
			}
		}
	}
	System.out.println(Arrays.toString(arr));
}
```

### 2.2 插入排序

把元素分为已排序的和未排序的。每次从未排序的元素取出第一个，与已排序的元素从尾到头逐一比较，找到插入点，将之后的元素都往后移一位，腾出位置给该元素

```java
/*
* 插入排序方法
*/
public static  void doInsertSort() {
	int[] array = { 8, 5, 10, 2, 19, 1 };
	for (int index = 1; index < array.length; index++) {// 外层向右的index，即作为比较对象的数据的index
		int temp = array[index];// 用作比较的数据
		int leftindex = index - 1;
		while (leftindex >= 0 && array[leftindex] > temp) {// 当比到最左边或者遇到比temp小的数据时，结束循环
			array[leftindex + 1] = array[leftindex];
			leftindex--;
		}
		array[leftindex + 1] = temp;// 把temp放到空位上
		System.out.println(Arrays.toString(array));
	}
}
```

### 2.3 选择排序

从数组中找到最小的元素，和第一个位置的元素互换。 

从第二个位置开始，找到最小的元素，和第二个位置的元素互换。

........

直到选出array.length-1个较小元素，剩下的最大的元素自动排在最后一位

```java
public static void selectrSort() {
	int[] arr = { 3, 6, 10, 2, 19, 1 };
	for (int i = 0; i < arr.length - 1; i++) {
		int minPos = i;
		for (int j = i; j < arr.length; j++) {
			if (arr[j] < arr[minPos]) {
				minPos = j;// 找出当前最小元素的位置
			}
		}
		if (arr[minPos] != arr[i]) {
			swap(arr, minPos, i);
		}
	}
}

public static void swap(int[] arr, int a, int b) {
	int temp = arr[a];
	arr[a] = arr[b];
	arr[b] = temp;
}
```

## 3. 变量交换

```java
	int a = 3, b = 5;
	a = a + b;// a==8,b==5
	b = a - b;// a==8,b==3
	a = a - b;// a==5,b==3
	System.out.println(a + "," + b);
	a = a ^ b;
	b = a ^ b;
	a = a ^ b;
	System.out.println(a + "," + b);
```

## 4. 

## 5. 