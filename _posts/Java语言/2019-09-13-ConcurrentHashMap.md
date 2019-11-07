---
layout: post
title: ConcurrentHashMap（1.8）
category: Java语言
tags: ConcurrentHashMap
keywords: java,ConcurrentHashMap
---
## 数据结构

```
	//存放key，value数据
    transient volatile Node<K,V>[] table;
	//为了扩容操作时，不影响读操作而设立，因此不保证读写实时一致性
    private transient volatile Node<K,V>[] nextTable;
	//基础计数，使用CAS操作更新
    private transient volatile long baseCount;
	//扩容阈值，触发扩容时为-1或者其他值小于0的数值
    private transient volatile int sizeCtl;
	//扩容时转移的index
    private transient volatile int transferIndex;
	//CASlock，当扩容或者新建counterCells时需要
    private transient volatile int cellsBusy;
	//每个线程写数据计数，与baseCount结合进行size计数
    private transient volatile CounterCell[] counterCells;

    @sun.misc.Contended static final class CounterCell {
        volatile long value;
        CounterCell(long x) { value = x; }
    }
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        volatile V val;
        volatile Node<K,V> next;

        Node(int hash, K key, V val, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.val = val;
            this.next = next;
        }

        public final K getKey()       { return key; }
        public final V getValue()     { return val; }
        public final int hashCode()   { return key.hashCode() ^ val.hashCode(); }
        public final String toString(){ return key + "=" + val; }
        public final V setValue(V value) {
            throw new UnsupportedOperationException();
        }

        public final boolean equals(Object o) {
            Object k, v, u; Map.Entry<?,?> e;
            return ((o instanceof Map.Entry) &&
                    (k = (e = (Map.Entry<?,?>)o).getKey()) != null &&
                    (v = e.getValue()) != null &&
                    (k == key || k.equals(key)) &&
                    (v == (u = val) || v.equals(u)));
        }

        /**
         * Virtualized support for map.get(); overridden in subclasses.
         */
        Node<K,V> find(int h, Object k) {
            Node<K,V> e = this;
            if (k != null) {
                do {
                    K ek;
                    if (e.hash == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                } while ((e = e.next) != null);
            }
            return null;
        }
    }
```
- 数组存储，数组元素嵌套单链表
- 单链表在一定条件下会转化为红黑树（带标记的平衡二叉树）

## get
```
	public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());
        
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            
            //落入的hash槽位有值
            if ((eh = e.hash) == h) {
            	//hash槽位头节点即为所找的值
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            
            else if (eh < 0)
            	//hash槽位头节点不为所找的值
            	//hash头节点小于0 有3种情况MOVED（-1，表明此时正在扩容），TREEBIN（-2，树的头节点），RESERVED（-3，暂留）
                return (p = e.find(h, key)) != null ? p.val : null;

			//eh>0,即该头节点构成链表，从链表中查找值
			//为何此查找方法不与eh<0时节点查找方法和并？此方法是while，eh<0时节点查找是do while
			//合并之后存在一个问题，槽位节点之后为空，会有空指针异常的情况（比如，一个线程在删除节点，另一个线程在读）
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }

	//find方法
    Node<K,V> find(int h, Object k) {
            Node<K,V> e = this;
            if (k != null) {
                do {
                    K ek;
                    if (e.hash == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                } while ((e = e.next) != null);
            }
            return null;
        }
```
1. 计算出key的hashCode h
2. 计算key 的hash槽位落点
3. 比较hash槽位落点的hash值eh，如相等，返回value值
4. eh小于0，有3种情况 （-1，表明此时正在扩容），TREEBIN（-2，树的头节点），RESERVED（-3，暂留），调用find方法查找节点
5. eh大于0，表明该节点未单链表，从单链表查找值

## size
```
    public int size() {
        long n = sumCount();
        return ((n < 0L) ? 0 :
                (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
                (int)n);
    }
   final long sumCount() {
        CounterCell[] as = counterCells; CounterCell a;
        long sum = baseCount;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                	//统计每个单元的value值
                    sum += a.value;
            }
        }
        return sum;
    }
```
- 统计baseCount和各线程CounterCell的计数器总和

## put
```
  public V put(K key, V value) {
        return putVal(key, value, false);
    }
  final V putVal(K key, V value, boolean onlyIfAbsent) {
		//key和value不能为空
        if (key == null || value == null) throw new NullPointerException();
        
        int hash = spread(key.hashCode());//计算hash值
        int binCount = 0;//hash槽位元素计数
        
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            
            if (tab == null || (n = tab.length) == 0)
            	//table初始化
                tab = initTable();
                
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
				//hash槽位无值，基于CAS替换元素，入槽位，无锁操作
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                  
            }
            
            else if ((fh = f.hash) == MOVED)
            	//该hash槽位有值且该值表明正在扩容，MOVED表示ConcurrentHashMap正在扩容，帮忙扩容
                tab = helpTransfer(tab, f);
            else {
            
            	//该hash槽位有值且没在扩容，元素入槽
                V oldVal = null;
                
                synchronized (f) {//同步块开始
                	//f为槽位头节点，对hash槽位头元素加锁
                	//像是1.7版本分段锁的进一步优化，粒度更细，直接对槽位加锁，同一槽位的操作可看作单线程进行
                	//如果hash冲突严重，有可能转化为重量级锁，这种情况可以重写hashcode和equal方法
                    if (tabAt(tab, i) == f) {
                    	//再次判断槽位的值有没有发生改变
                        if (fh >= 0) {
                        	//该槽位为链表的形式
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                     //链表中key值存在，直接替换value值
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                	//遍历到单链表尾，把新值放到链表尾部
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {
                        	//该槽位为红黑树形式
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }//同步块结束
                
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                    	//槽位链表长度大于等于8且table.length大于64，调整成红黑树结构
                    	//调整过程也是对槽位头元素加锁处理
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }//循环结束
        
        //自旋循环外部，统计大小或者扩容
        addCount(1L, binCount);
        return null;
    }
```
1. 检查key，value，key和value有为空抛出异常，检查table初始化情况，未初始化进行初始化
2. 计算key的hash值和hash槽位
3. hash槽位为空，通过CAS操作替换槽位值
4. hash槽位不为空，根据hash槽位hash值判断状态，为MOVED状态，进入扩容函数
5. hash槽位不为空且没在扩容，对槽位头元素加锁，进入同步代码块
6. 往单链表或红黑树中放入新元素
7. 检查单链表是否需要转化为红黑树，转化条件槽位链表长度大于等于8且table.length大于64
8. addCount函数更新计数器，并判断是否需要扩容

## get
```
    public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
        Node<K,V> find(int h, Object k) {
            Node<K,V> e = this;
            if (k != null) {
                do {
                    K ek;
                    if (e.hash == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                } while ((e = e.next) != null);
            }
            return null;
        }
```
- 计算hash值和hash槽位落点，获取槽位值或从单链表红黑树中获取元素值

## initTable
```
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // 让出处理器的使用权
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                //CAS成功，进行table初始化
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```
1. 判断table的状态，sizeCtl小于0，表明初始化由其他线程在做，并让出处理器的使用权
2. sizeCtl大于0，通过CAS替换sizeCtl，替换成功进行table初始化
3. 初始化默认容量为16，并修改sizeCtl为容量的3/4，方便进行扩容判断

## addCount
```
private final void addCount(long x, int check) {
        CounterCell[] as; long b, s;

		//size计数部分，原理类似LongAdder
        if ((as = counterCells) != null ||
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            //CAS更新baseCount不成功，转为采用CounterCell进行计数
            CounterCell a; long v; int m;
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
            s = sumCount();
        }

		//扩容检查及扩容,如putVal放入的hash槽位无值，不会触发扩容检查
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;

			//检查当前size是否大于sizeCtl
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                int rs = resizeStamp(n);
                if (sc < 0) {
                
                	//处于扩容状态，检查扩容完毕与否等
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    
                    //当前线程加入扩容处理流程    
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    	//扩容
                        transfer(tab, nt);
                }

				//当前线程首先触发扩容
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
                s = sumCount();
            }
        }
    }
```
1. 通过CAS更新baseCount或者利用LongAdder原理更新计数器，保证size方法的有效性
2. 扩容检查：当此线程放入元素刚好放入在hash槽位上，不会进行扩容检查
3. 扩容条件：容量大于sizeCtl即容量的3/4
4. 判断扩容状态，利用transfer方法进行扩容

## transfer
```
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
	int n = tab.length, stride;
	
	if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
		stride = MIN_TRANSFER_STRIDE;
	 
	 //触发扩容，初始化nextTable 
	if (nextTab == null) {         
		try {
			@SuppressWarnings("unchecked")
			Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];//扩容大小为table的2倍
			nextTab = nt;
		} catch (Throwable ex) {      // try to cope with OOME
			sizeCtl = Integer.MAX_VALUE;
			return;
		}
		nextTable = nextTab;
		transferIndex = n;//从table末槽位开始操作
	}

	//扩容进行时
	int nextn = nextTab.length;
	ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);//标记节点，标记table的槽位，槽位转移结束的标记
	boolean advance = true;
	boolean finishing = false; // to ensure sweep before committing nextTab
	
	for (int i = 0, bound = 0;;) {
		Node<K,V> f; int fh;
		
		//自旋替换transferIndex，操作下一个槽位
		while (advance) {
			int nextIndex, nextBound;
			if (--i >= bound || finishing)
				advance = false;
			else if ((nextIndex = transferIndex) <= 0) {
				i = -1;
				advance = false;
			}
			else if (U.compareAndSwapInt
					 (this, TRANSFERINDEX, nextIndex,
					  nextBound = (nextIndex > stride ?
								   nextIndex - stride : 0))) {
				bound = nextBound;
				i = nextIndex - 1;
				advance = false;
			}               
		}

		if (i < 0 || i >= n || i + n >= nextn) {
			int sc;
			//循环结束条件判定，扩容线程必运行
			if (finishing) {
				nextTable = null;
				table = nextTab;
				sizeCtl = (n << 1) - (n >>> 1);
				return;
			}
			if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
				if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
					return;
				finishing = advance = true;
				i = n; // recheck before commit
			}
		}
		else if ((f = tabAt(tab, i)) == null)
			advance = casTabAt(tab, i, null, fwd);//hash槽位为空，放入标记节点
		else if ((fh = f.hash) == MOVED)
			advance = true; //空槽位或者扩容已完成无需转移
		else {
			//非空槽位，头节点加锁，逐步进行节点转移，table->nextTable
			//hash槽位对应的节点进行扩容，即元素转移至nextTable
			synchronized (f) {
				if (tabAt(tab, i) == f) {
					//同一个槽位扩容后可拆分成两条链表，存放于不同位置（i，i+n）
					Node<K,V> ln, hn;
					if (fh >= 0) {
						//这个槽位存放的为链表
						int runBit = fh & n;//n是2的倍数，runBit只存在两个值0或n
						Node<K,V> lastRun = f;
						
						//这段代码有点费解，其实主要为处理一些特殊情况而不用重新new新节点，从而复用以前的节点
						//比如该槽位节点在nextTable的槽位都为i，那么不需要生成新链表，只需将原来槽位头节点替换即可
						//比如该槽位前半段在nextTable的新槽位为i，后半段在nextTable的槽位为i+n，则后半段不需要new新节点了
						//感觉是为了节省内存空间，减轻GC的压力
						for (Node<K,V> p = f.next; p != null; p = p.next) {
							int b = p.hash & n;
							if (b != runBit) {
								runBit = b;
								lastRun = p;
							}
						}                            
						if (runBit == 0) {
							ln = lastRun;
							hn = null;
						}
						else {
							hn = lastRun;
							ln = null;
						}
						for (Node<K,V> p = f; p != lastRun; p = p.next) {
							int ph = p.hash; K pk = p.key; V pv = p.val;
							if ((ph & n) == 0)
								ln = new Node<K,V>(ph, pk, pv, ln);
							else
								hn = new Node<K,V>(ph, pk, pv, hn);
						}
						setTabAt(nextTab, i, ln);
						setTabAt(nextTab, i + n, hn);
						setTabAt(tab, i, fwd);//将该槽位标记为转移完成的状态
						advance = true;
					}
					else if (f instanceof TreeBin) {

						//该槽位形成红黑树的结构
						TreeBin<K,V> t = (TreeBin<K,V>)f;
						TreeNode<K,V> lo = null, loTail = null;
						TreeNode<K,V> hi = null, hiTail = null;
						int lc = 0, hc = 0;
						for (Node<K,V> e = t.first; e != null; e = e.next) {
							int h = e.hash;
							TreeNode<K,V> p = new TreeNode<K,V>
								(h, e.key, e.val, null, null);
							if ((h & n) == 0) {
								if ((p.prev = loTail) == null)
									lo = p;
								else
									loTail.next = p;
								loTail = p;
								++lc;
							}
							else {
								if ((p.prev = hiTail) == null)
									hi = p;
								else
									hiTail.next = p;
								hiTail = p;
								++hc;
							}
						}
						ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
							(hc != 0) ? new TreeBin<K,V>(lo) : t;
						hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
							(lc != 0) ? new TreeBin<K,V>(hi) : t;
						setTabAt(nextTab, i, ln);
						setTabAt(nextTab, i + n, hn);
						setTabAt(tab, i, fwd);//将该槽位标记为转移完成的状态
						advance = true;
					}
				}
			}
		}
	}
}
```
1. 判断函数传入的nextTab是否为空，为空先初始化nextTab，并把nextTable置为nextTab，只有单个线程会初始化nextTable，由上层函数进行控制
2. 此线程通过自旋和CAS替换获取一段槽位或者单个槽位的操作权
3. 槽位的元素转移正式开始，先进行结束检查，已结束重置sizeCtl为正式容量的3/4
4. 槽位为空，插入扩容标记节点
5. 槽位为扩容标记节点，直接略过，表明有其他线程正在操作该节点或者原节点为空槽位或者该槽位转移完成
6. 槽位不为空且不是标记节点，进行槽位上的元素转移
7. 槽位元素转移，对头元素加锁，保证线程安全性
8. 单链表或者红黑树元素转移，转移完成将头节点标记为MOVED