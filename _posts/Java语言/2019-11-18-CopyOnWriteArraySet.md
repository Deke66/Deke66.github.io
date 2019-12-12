---
layout: post
title: CopyOnWriteArraySet（1.8）
category: Java语言
tags: CopyOnWriteArraySet
keywords: java,CopyOnWriteArraySet
---
## 类

```
public class CopyOnWriteArraySet<E> extends AbstractSet<E>
        implements java.io.Serializable {
```

1. 继承AbstractSet，具有Set集合的特性
2. 实现Serializable，可进行序列化

## 成员变量

```
    private final CopyOnWriteArrayList<E> al;
```
1. CopyOnWriteArrayList保存元素

## 构造方法

```
    public CopyOnWriteArraySet() {
        al = new CopyOnWriteArrayList<E>();
    }
	
    public CopyOnWriteArraySet(Collection<? extends E> c) {
        if (c.getClass() == CopyOnWriteArraySet.class) {
            @SuppressWarnings("unchecked") CopyOnWriteArraySet<E> cc =
                (CopyOnWriteArraySet<E>)c;
            al = new CopyOnWriteArrayList<E>(cc.al);
        }
        else {
            al = new CopyOnWriteArrayList<E>();
            al.addAllAbsent(c);
        }
    }	
```

- 构造方法主要是对成员变量CopyOnWriteArrayList进行初始化

## contains

```
    public boolean contains(Object o) {
        return al.contains(o);
    }
```

## add

```
    public boolean add(E e) {
        return al.addIfAbsent(e);
    }
```

## remove

```
    public boolean remove(Object o) {
        return al.remove(o);
    }
```

## 小结
1. 线程安全性的保证机制-使用CopyOnWriteArrayList
2. 适用场景-读多写少，存储不重复的元素









