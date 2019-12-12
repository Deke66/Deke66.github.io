---
layout: post
title: ThreadLocal（1.8）
category: Java语言
tags: ThreadLocal
keywords: java,ThreadLocal
---

## 成员变量

```
    private final int threadLocalHashCode = nextHashCode();
    private static AtomicInteger nextHashCode =
        new AtomicInteger();
    private static final int HASH_INCREMENT = 0x61c88647;
```
1. 每个 ThreadLocal 对象初始化HashCode，保存在 Thread 中的ThreadLocal对象

## 内部类 ThreadLocalMap

### Entry

```
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```
1. key是弱引用，垃圾回收发生时被回收，value是强引用

### 成员变量

```

        private static final int INITIAL_CAPACITY = 16;
        private Entry[] table;
        private int size = 0;
		// 扩容相关
        private int threshold; // Default to 0
```
1. 初始化 table 容量大小为16
2. threshold 为扩容负载因子，size超过时进行扩容，一般为 size 的

### 构造方法

#### 构造新Map

```
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
```
1. 初始化table
2. 计算落点位置
3. 设置负载因子


#### 构造新Map从继承的ThreadLocal

```
        private ThreadLocalMap(ThreadLocalMap parentMap) {
            Entry[] parentTable = parentMap.table;
            int len = parentTable.length;
            setThreshold(len);
            table = new Entry[len];

            for (int j = 0; j < len; j++) {
                Entry e = parentTable[j];
                if (e != null) {
                    @SuppressWarnings("unchecked")
                    ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
                    if (key != null) {
                        Object value = key.childValue(e.value);
                        Entry c = new Entry(key, value);
                        int h = key.threadLocalHashCode & (len - 1);
                        while (table[h] != null)
                            h = nextIndex(h, len);
                        table[h] = c;
                        size++;
                    }
                }
            }
        }
```
1. 把入参 ThreadLocalMap 的table 复制至新的 ThreadLocalMap 的 table 里

### getEntry(ThreadLocal<?> key)

```
        private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }
```
1. 计算key落点位置
2. 如果落点key值为空，获取落点后面的 Entry

### set(ThreadLocal<?> key, Object value)

```
        private void set(ThreadLocal<?> key, Object value) {


            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

				// key 值对的上直接返回
                if (k == key) {
                    e.value = value;
                    return;
                }

				// key 为空，说明GC过，key是弱引用被回收了
				// 清理强引用 value 减少内存泄漏
                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```
1. 计算ThreadLocal 对象的hash落点
2. 从 table 查找key，判断找到的key值是否未空
	1. 不为空设置值直接返回
	2. 为空先进行一边value强引用置空，之后再设置值
3. 检查是否需要扩容


### remove(ThreadLocal<?> key)

```
        private void remove(ThreadLocal<?> key) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                if (e.get() == key) {
                    e.clear();
                    expungeStaleEntry(i);
                    return;
                }
            }
        }
```

### rehash()

```
        private void rehash() {
            expungeStaleEntries();

            // Use lower threshold for doubling to avoid hysteresis
            if (size >= threshold - threshold / 4)
                resize();
        }
		
        private void resize() {
            Entry[] oldTab = table;
            int oldLen = oldTab.length;
            int newLen = oldLen * 2;
            Entry[] newTab = new Entry[newLen];
            int count = 0;

            for (int j = 0; j < oldLen; ++j) {
                Entry e = oldTab[j];
                if (e != null) {
                    ThreadLocal<?> k = e.get();
                    if (k == null) {
                        e.value = null; // Help the GC
                    } else {
                        int h = k.threadLocalHashCode & (newLen - 1);
                        while (newTab[h] != null)
                            h = nextIndex(h, newLen);
                        newTab[h] = e;
                        count++;
                    }
                }
            }

            setThreshold(newLen);
            size = count;
            table = newTab;
        }		
```


## set

### set

```
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }	
```
1. 获取线程引用
2. 根据线程引用获取ThreadLocalMap
3. 如果ThreadLocalMap为空，初始化，不为空则设置value

- 小结
	1. 每个Thread持有ThreadLocalMap的引用，ThreadLocalMap持有弱引用key和强引用value
	2. 弱引用key指向ThreadLocal引用

### createMap

```
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```
1. ThreadLocalMap 的 key 为 当前 ThreadLocalMap 对象

## get

```
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```
1. 获取线程的 ThreadLocalMap 对象，若为空，则初始化 ThreadLocalMap ， 返回 null 值
2. 从 ThreadLocalMap 获取 Entry，若 Entry 为空，则初始化 Entry ，放回 null 值

## 小结
1. 引用关系
	1. Thread.ThreadLocalMap.key -> ThreadLocal (弱引用）
	2. Thread.ThreadLocalMap.value -> value （强引用）
2. 使用注意事项
	1. value 不使用及时 remove，可以有效防止内存泄漏












