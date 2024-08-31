---
tags:
  - Java基础
  - 编程
author: tong
layout: post
title: 记多线程 addAll 问题
---

# 现象

数据量较大时，使用了多线程去获取数据，并通过 addAll 的方式将它们添加至同一个 result (ArrayList) 集合，我们发现在这个集合中出现了 null 元素，导致下面的处理程序出现空指针问题
# 根本原因

我们查看 ArrayList 的源码，可以知道 addAll 方法并不是原子方法

```java
public boolean addAll(Collection<? extends E> c) {
    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);  // Increments modCount
    System.arraycopy(a, 0, elementData, size, numNew);
    size += numNew;
    return numNew != 0;
}
```

那么当多线程进行处理操作时，有可能出现以下问题
1. 元素被覆盖
2. 元素为 null
3. 数组越界问题

## 元素被覆盖

1. 列表大小为 9，即 size=9
2. 线程 A 开始进入 `addAll` 方法，这时它获取到 size 的值为 9，调用 ` ensureCapacityInternal ` 方法进行容量判断。可以容纳返回
3. 线程 B 此时也进入 `addAll` 方法，它获取到 size 的值也为 9，也开始调用 ` ensureCapacityInternal ` 方法。可以容纳返回
6. 线程 A 开始进行设置值操作， `System.arraycopy` 操作，它的 size 值为 9。此时复制的数组元素开始地址是 9
7. 线程 B 也开始进行设置值操作，它尝试设置  `System.arraycopy` 它的 size 值也为 9。此时复制的数组元素开始地址是 9，将线程 A 复制过去的元素覆盖

## 元素为 null 

1. 列表大小为 9，即 size=9
2. 线程 A 开始进入 `addAll` 方法，这时它获取到 size 的值为 9，调用 ` ensureCapacityInternal ` 方法进行容量判断。可以容纳返回
3. 线程 B 此时也进入 `addAll` 方法，它获取到 size 的值也为 9，也开始调用 ` ensureCapacityInternal ` 方法。可以容纳返回
6. 线程 A 开始进行设置值操作， `System.arraycopy` 操作，它的 size 值为 9。此时复制的数组元素开始地址是 9
7. 线程 B 也开始进行设置值操作，它尝试设置  `System.arraycopy` 它的 size 值也为 9。此时复制的数组元素开始地址是 9，将线程 A 复制过去的元素覆盖
8. 线程 A 执行 `size += numNew;` 后 size 为 10
9. 线程 B 执行 `size += numNew;` 后 size 为 11
10. 下一个线程进来时将从 idx 11 开始赋值，导致 idx 为 10 的元素为 null

## 数组越界

1. 列表大小为 9，即 size=9
2. 线程 A 开始进入 `addAll` 方法，这时它获取到 size 的值为 9，调用 ` ensureCapacityInternal ` 方法进行容量判断。可以容纳返回
3. 线程 B 此时也进入 `addAll` 方法，它获取到 size 的值也为 9，也开始调用 ` ensureCapacityInternal ` 方法。可以容纳返回
6. 线程 A 开始进行设置值操作， `System.arraycopy` 操作，而 elementData 没有进行过扩容，它的可容纳长度为 10。执行 `size += numNew;` 后 size 为 10
7. 线程 B 也开始进行设置值操作，它尝试设置  `System.arraycopy`，而 elementData 没有进行过扩容，它的可容纳长度为 10。此时复制的数组元素开始地址是 size 值 10，将线程 A 复制后超出数组长度 10，导致 `IndexOutOfBoundsException` 异常

# 解决方案

避免多线程调用 `addAll()` 方法
1. 通过 Collections 的 synchronizedList 方法将 ArrayList 转换成线程安全的容器后再使用。
		`List<Object> list =Collections.synchronizedList(new ArrayList<Object>);`
2. 使用线程安全的 CopyOnWriteArrayList 代替线程不安全的 ArrayList。
		`List<Object> list1 = new CopyOnWriteArrayList<Object>();`

注意：这两种方式都是通过加锁的方式来实现线程安全的，都将会导致性能的部分下降


---

[back](../编程相关文章汇总)

[home](../../../index)