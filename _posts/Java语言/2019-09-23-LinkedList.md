---
layout: post
title: LinkedList（1.8）
category: Java语言
tags: LinkedList
keywords: java,LinkedList
---
## 类
```
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```
- 实现List接口，使用List存储数据
- 实现Deque接口，可作为双端队列
- 实现Cloneable接口，可将属性进行复制
- 实现Serializable接口，可序列化

## 数据结构

```
	//指向列表的第一个元素
    transient Node<E> first;

	//指向列表的最后一个元素
    transient Node<E> last;
	//Node节点
    private static class Node<E> {
        E item;
        Node<E> next;//next指针
        Node<E> prev;//prev指针

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```
- 双向链表存放数据，包含头元素指针和尾元素指针

## add
```
    public boolean add(E e) {
        linkLast(e);
        return true;
    }
      void linkLast(E e) {
        final Node<E> l = last;
        //新节点
        final Node<E> newNode = new Node<>(l, e, null);
        //把新加入的节点当作最后的节点
        last = newNode;
        if (l == null)
            first = newNode;//l为空，说明原来无节点，此新加入的节点既为首节点，也为尾节点
        else
            l.next = newNode;//新节点加在尾节点之后
        size++;
        modCount++;
    }
```
- 创建新节点，往尾元素插入新元素，新元素为新的尾元素

## remove
```
    public E remove(int index) {
        checkElementIndex(index);
        return unlink(node(index));
    }
	Node<E> node(int index) {
        // assert isElementIndex(index);

        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
    E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;
        final Node<E> next = x.next;
        final Node<E> prev = x.prev;

        if (prev == null) {
            first = next;
        } else {
            prev.next = next;
            x.prev = null;
        }

        if (next == null) {
            last = prev;
        } else {
            next.prev = prev;
            x.next = null;
        }

        x.item = null;
        size--;
        modCount++;
        return element;
    }	
```
1. 检查index是否溢出
2. 查找index的节点，index<size的一半，从头元素开始查找，否从未元素开始查找
3. 从双链表中删除该元素并返回

## get
```
    public E get(int index) {
        checkElementIndex(index);
        return node(index).item;
    }
     Node<E> node(int index) {
        if (index < (size >> 1)) {
        	//从前往后找
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
        	//从后往前找
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    } 
```
1. 检查index是否溢出
2. 获取index元素

## set
```
    public E set(int index, E element) {
        checkElementIndex(index);
        Node<E> x = node(index);
        E oldVal = x.item;
        x.item = element;
        return oldVal;
    }
```
- 检查index是否溢出，不溢出查找到元素再修改

## iterator
```
    public Iterator<E> iterator() {
        return listIterator();
    }
    public ListIterator<E> listIterator(final int index) {
        rangeCheckForAdd(index);

        return new ListItr(index);
    }
```
- ListItr遍历删除元素调用本类方法

## 与ArrayList对比
- 存储结构不同：ArrayList使用Object数组，LinkedList使用双链表，存储同样的数据LinkedList比ArrayList更耗费空间，因此LinkedList更容易造成内存溢出
- 使用场景不同：ArrayList的随机访问效率高，但增加（扩容时元素需要进行复制）或者删除（可能存在很多元素需要移动）的效率低；LinkedList随机访问效率低，增加和删除操作快；对于少量数据或者大量但不经常增删的数据，比较适合用ArrayList，对于大量且经常需要增删的数据建议用LinkedList
- 线程安全性：都不是线程安全的集合
- 实现的接口：都实现了Collection，List等接口，LinkedList还实现了Deque接口，可以方便用作双端队列


