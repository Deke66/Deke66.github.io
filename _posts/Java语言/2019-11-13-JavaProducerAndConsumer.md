---
layout: post
title: Java多线程生产者和消费者模型具体实现
category: Java语言
tags: 多线程
keywords: 生产者消费者模型
---
## 生产者消费者模型
- 在一个系统中，存在生产者和消费者两种角色，他们通过内存缓冲区进行通信，生产者生产产品，消费者消费产品。
	1. 生产者生产数据到缓冲区中，消费者从缓冲区中取数据。
	2. 如果缓冲区已经满了，则生产者线程不能生产数据。
	3. 如果缓冲区为空，消费者无法从缓冲区获取数据。

## 接口定义

### 消息

```
public interface IMessage {
    String read();
    boolean write(String msg);
}
```

### 消息队列

```
public interface IMessageQueue {
    boolean produceMessage(IMessage message);
    IMessage consumeMessage();
}
```

### 生产者

```
public interface IProducer {
    boolean produce(IMessage message);
}
```

### 消费者

```
public interface IConsumer {
    IMessage consume();
}
```

## 具体实现

### 消息

```
public class Message implements IMessage {
    private String msg;
    @Override
    public String read() {
        return msg;
    }

    @Override
    public boolean write(String msg) {
        this.msg = msg;
        return true;
    }
}
```

### 消息队列

```
public abstract class AbstractMessageQueue implements IMessageQueue {

    static final int MAXIMUM_CAPACITY = 1 << 30;

    protected final int capacity;

    protected volatile IMessage[] messages;

    protected final String id;

    public AbstractMessageQueue(int capacity,String id){
        this.capacity = messagesSizeFor(capacity);
        this.id = id;
        messages = new IMessage[this.capacity];
    }

    /**
     * Returns a power of two size for the given target capacity.
     */
    static final int messagesSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }


}
```


1. wait-notify机制
	1. 生产者和消费者共用同一把锁
	2. 缓冲区满了，阻塞生产者
	3. 缓冲区为空，阻塞消费者
	4. 唤醒的时候把生产者和消费者一同唤醒
	
```
public class MessageQueueWaitNotify extends AbstractMessageQueue {

    private final Object LOCK = new Object();

	//当前要消费消息的index，消息用数组对象存储
    private volatile int consumeIndex;

	//当前要生产消息的index，消息用数组对象存储
    private volatile int produceIndex;

    public MessageQueueWaitNotify(int capacity, String id) {
		//父类主要将capacity设置成2的倍数
        super(capacity, id);
    }

    @Override
    public boolean produceMessage(IMessage message) {
        synchronized (LOCK){
			//判断缓冲区是否满了
            if(produceIndex-consumeIndex < capacity){
                messages[produceIndex&(capacity-1)] = message;
                produceIndex++;
                LOCK.notifyAll();
                return true;
            }else {
                try {
                    LOCK.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
        return false;
    }

    @Override
    public IMessage consumeMessage() {
        IMessage message = null;
        synchronized (LOCK){
		
			//缓冲区是否为空
            if (consumeIndex >= produceIndex){
                try {
                    LOCK.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }else {
                message = messages[consumeIndex++&(capacity-1)];
                LOCK.notifyAll();

            }

        }
        return message;
    }
```


2. JDK阻塞队列实现
	- BlockingQueue put和take为阻塞方法

```
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class MessageBlockQueue implements IMessageQueue {

    private final BlockingQueue<IMessage> blockingQueue;

    private final String id;

    private final int capacity;

    public MessageBlockQueue(int capacity, String id){
        this.blockingQueue = new ArrayBlockingQueue<>(capacity);
        this.id = id;
        this.capacity = capacity;
    }

    @Override
    public boolean produceMessage(IMessage message) {
        try {
            this.blockingQueue.put(message);
            return true;
        } catch (Exception e) {
        }
        return false;
    }

    @Override
    public IMessage consumeMessage() {
        try {
            return this.blockingQueue.take();
        } catch (Exception e) {
        }
        return null;
    }
```

3. 可重入锁ReentrantLock实现
	1. 生产者和消费者各自享有自己的锁
	2. 没有采用阻塞唤醒机制，会消耗CPU时间

```
import sun.misc.Unsafe;

import java.lang.reflect.Field;
import java.security.AccessController;
import java.security.PrivilegedExceptionAction;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;


public class CustomizeMessageQueue extends AbstractMessageQueue {

    public volatile int consumeIndex;

    private Lock read = new ReentrantLock();

    private Lock write = new ReentrantLock();

    private Object readLock = new Object();

    private Object writeLock = new Object();

    public volatile int produceIndex;

	//volatile数组元素不具有可见性，用Unsafe类的方法解决
    private static  Unsafe U = null;
    private static final long ABASE;
    private static final int ASHIFT;

    static {

        try {
            final PrivilegedExceptionAction<Unsafe> action = new PrivilegedExceptionAction<Unsafe>() {
                public Unsafe run() throws Exception {
                    Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
                    theUnsafe.setAccessible(true);
                    return (Unsafe) theUnsafe.get(null);
                }
            };
            U = AccessController.doPrivileged(action);
        }
        catch (Exception e){
            throw new RuntimeException("Unable to load unsafe", e);
        }

        Class<?> ak = IMessage[].class;
        ABASE = U.arrayBaseOffset(ak);
        int scale = U.arrayIndexScale(ak);
        if ((scale & (scale - 1)) != 0)
            throw new Error("data type scale not a power of two");
        ASHIFT = 31 - Integer.numberOfLeadingZeros(scale);
    }

    public CustomizeMessageQueue(int capacity, String id){
        super(capacity, id);
    }


    @Override
    public  boolean produceMessage(IMessage message){

        write.lock();
        try{
            if(produceIndex-consumeIndex < capacity){
                messages[produceIndex&(capacity-1)] = message;
                setTabAt(messages,produceIndex&(capacity-1),message);
                produceIndex++;
                return true;
            }
        } finally {
            write.unlock();
        }

//            synchronized (writeLock){
//                if(produceIndex-consumeIndex < capacity){
//                    messages[produceIndex&(capacity-1)] = message;
//                    produceIndex++;
//                    return true;
//                }
//            }
            return false;

    }


    @Override
    public IMessage consumeMessage() {
        IMessage message = null;
        read.lock();
        try{
            if(consumeIndex < produceIndex) {
                message = tabAt(messages,consumeIndex&(capacity-1));
                consumeIndex++;
            }
        } finally {
            read.unlock();
        }

//        synchronized (readLock){
//            if(consumeIndex < produceIndex) {
//                message = messages[consumeIndex&(capacity-1)];
//                consumeIndex++;
//            }
//        }
        return message;
    }
	
	//解决volatile数组元素不具有volatile变量的特点
    static final  IMessage tabAt(IMessage[] tab, int i) {
        return (IMessage)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }

    static final void setTabAt(IMessage[] tab, int i, IMessage v) {
        U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
    }


}
```

- 小结（主要针对2和3实现方式待改进）
	1. 消息消费按照先进先出的方式，消费了的消息没有移出消息队列，有可能造成内存泄露
	2. 没有对index初始化的操作，有可能出现int数据溢出
	3. 消息消费不能指定某条消息进行消费
	4. 一条消息只能被一个消费者进行消费

### 生产者

```
public class Producer implements IProducer {
    private final IMessageQueue messageQueue;

    public Producer(IMessageQueue messageQueue){
        this.messageQueue= messageQueue;
    }
    @Override
    public boolean produce(IMessage message) {
        return messageQueue.produceMessage(message);
    }
}
```

### 消费者

```
public class Consumer implements IConsumer {

    private final IMessageQueue messageQueue;
    public Consumer(IMessageQueue messageQueue){
        this.messageQueue = messageQueue;
    }
    @Override
    public IMessage consume() {
        return this.messageQueue.consumeMessage();
    }
}
```

## 测试

```
public class Test {

    private volatile static boolean stop = false;

    public static void main(String[] args) {
		//消息队列大小
        int messageQueueCapacity = 1000000;
		//消费者数量
        int consumerNum = 100;
		//生产者数量
        int producerNum = 100;
		//消息总量
        int messageNum = 50000000;

        IMessageQueue messageQueue = new CustomizeMessageQueue(messageQueueCapacity,"message-queue-1");
		
		//采用ConcurrentHashMap辅助判定消息是否重复消费
        ConcurrentHashMap<String,Object> concurrentHashMap = new ConcurrentHashMap<>(messageNum);
        final Object object = new Object();

		//创建消息
        IMessage[] message = new IMessage[messageNum];
        for (int i = 0; i < messageNum; i++) {
            message[i] = new Message();
            message[i].write(String.valueOf(i));
        }

		//创建生产者及设定每个生产者运送消息的数量
        int num = messageNum/producerNum;
        for (int i = 0; i < producerNum; i++) {
            final int index = i;
             new Thread(()->{
                IProducer producer = new Producer(messageQueue);
                for (int j = num*index; j < num*(index+1); j++) {
                    while (!stop){
                        if (producer.produce(message[j])) break;
                        else {
                            try {
                                Thread.sleep(5);
                            } catch (InterruptedException e) {
                                e.printStackTrace();
                            }
                        }
                    }
                }
            }).start();
        }


		//创建消费者
        for (int i = 0; i < consumerNum; i++) {
            new Thread(()->{
                IConsumer consumer = new Consumer(messageQueue);
                IMessage msg = null;
                while (!stop){
                    msg = consumer.consume();
                    if (msg != null) {
                        if (concurrentHashMap.putIfAbsent(msg.read(),object) != null) System.out.println("消息被重复消费了"+msg.read()+":");
                    }else {
                        try {
                            Thread.sleep(5);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                }

            }).start();
        }

		//统计执行时间
        long start = System.currentTimeMillis();
        while (concurrentHashMap.size()<messageNum) {
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        stop = true;
        long end = System.currentTimeMillis();
        System.out.println(end-start);
        System.out.println(concurrentHashMap.size());
    }
```













