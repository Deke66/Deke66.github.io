---
layout: post
title: CopyOnWriteArrayList（1.8）
category: Java语言
tags: CopyOnWriteArrayList
keywords: java,CopyOnWriteArrayList
---
## 类

```
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
```

1. 实现List<E>, RandomAccess, Cloneable, java.io.Serializable，支持List存储，随机访问，复制和序列化

## 成员变量

```
	//可重入锁
    final transient ReentrantLock lock = new ReentrantLock();
	// 存储元素的数组，只有getArray/setArray可以访问
    private transient volatile Object[] array;
```
1. 为保证并发线程安全使用了可重入锁ReentrantLock

## 构造方法

```
    public CopyOnWriteArrayList() {
        setArray(new Object[0]);
    }
	
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

- 构造方法主要是对成员变量array进行初始化

## contains

```
    public boolean contains(Object o) {
        Object[] elements = getArray();
        return indexOf(o, elements, 0, elements.length) >= 0;
    }
```
- 直接遍历原数组，查找是否存在与入参相等的元素

## get

```
    public E get(int index) {
        return get(getArray(), index);
    }
```
1. 获取对应下标的元素，不考虑index溢出的情况

## set

```
    public E set(int index, E element) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            E oldValue = get(elements, index);

            if (oldValue != element) {
                int len = elements.length;
                Object[] newElements = Arrays.copyOf(elements, len);
                newElements[index] = element;
                setArray(newElements);
            } else {
                // Not quite a no-op; ensures volatile write semantics
                setArray(elements);
            }
            return oldValue;
        } finally {
            lock.unlock();
        }
    }
```

1. 加锁
2. 如果index下的元素与原来的元素不相等替换原来的元素
3. 复制原来的数组，得到新的数组，改变array引用，以保证数组元素的可见性

## add

```
    public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
```
1. 加可重入锁
2. 获取数组元素的长度，复制数组得新数组
3. 改变array引用，以保证数组元素的可见性

## remove

```
    public E remove(int index) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            E oldValue = get(elements, index);
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
1. 加锁
2. 获取原index下的元素
3. 如果被删除元素是最后一个元素，复制原数组中index前的所有元素
4. 如果不是，则进行两次复制，先进行index前的元素复制，后进行index后的元素复制

## 小结
1. 线程安全性的保证机制
	1. 在进行写操作加可重入锁，读的时候不对数组元素index溢出做检查
	2. 写操作，如果数组元素发生变动，直接复制原数组元素（System.arraycopy），改变array的引用指向触发volatile语义保证数组元素多线程间的可见性
2. 适用场景-读多写少
	1. 数组元素发生变动，直接调用System.arraycopy进行元素复制，那么旧数组相当于直接抛弃
	2. 写的时候加入了可重入锁，读不加锁








