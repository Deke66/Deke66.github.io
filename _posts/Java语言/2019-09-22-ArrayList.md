---
layout: post
title: ArrayList（1.8）
category: Java语言
tags: ArrayList
keywords: java,ArrayList
---
## 类
```
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```
- 实现List接口，支持数组数据结构存储数据
- 实现RandomAccess接口，标记此类支持快速随机访问
- 实现Cloneable接口，可将属性进行复制
- 实现Serializable接口，可序列化

## 数据结构

```
	//存放元素的数组，默认数组大小为10
    transient Object[] elementData; 
    //大小，非elementData的length，存入元素的总个数
    private int size;
```
- Object数组存储元素

## add
```
    public boolean add(E e) {
    	//保证数组不溢出，否则进行扩容操作
        ensureCapacityInternal(size + 1);  
        elementData[size++] = e;
        //返回值都为true
        return true;
    }
```
- 数组元素不溢出的情况下size++和赋值

## grow
```
    private void grow(int minCapacity) {
        int oldCapacity = elementData.length;
        //扩容大小为之前的1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
         //新的容量大小不能超过Integer.MAX_VALUE - 8
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
		//元素复制，主要采用System.arraycopy方法
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```
- 如果放入元素数组将溢出，进行扩容
- 扩容大小为原先容量的1.5倍

## remove
```
    public E remove(int index) {
        rangeCheck(index);

        modCount++;
        //返回旧元素
        E oldValue = elementData(index);
		
        int numMoved = size - index - 1;

		//如果非末尾元素，所有元素前移
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; 

        return oldValue;
    }
```
1. 判断index是否超出数组大小
2. 获取旧元素，并将元素前移

## get
```
    public E get(int index) {
        rangeCheck(index);

        return elementData(index);
    }
```
- 检查index是否溢出，返回index所在元素

## set
```
    public E set(int index, E element) {
        rangeCheck(index);

        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }
```
- 检查index是否溢出，不溢出修改数组元素

## iterator
```
    public Iterator<E> iterator() {
    	//访问指针的封装对象
        return new Itr();
    }
    private class Itr implements Iterator<E> {
        int cursor;        // 移动指针
        int lastRet = -1; // 上一个指针
        int expectedModCount = modCount;
      }
```
- 使用int作为指针移动访问元素


