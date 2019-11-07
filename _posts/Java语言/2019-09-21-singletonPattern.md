---
layout: post
title: 单例模式
category: Java语言
tags: 单例模式
keywords: java,单例模式
---
## 概念
- 一个类有且仅有一个实例，并且自行实例化向整个系统提供。

## 线程安全式实现方式

### 饿汉式（非静态代码块）

```
public class Singleton {
    private static  Singleton singleton = new Singleton();

    private Singleton(){
    }

    public static Singleton getInstance(){
        return singleton;
    }
}
```
- 在类加载时期就进行初始化，线程安全的实现方式
- 如果该类有其他静态属性或静态方法时，调用静态方法或属性时也会触发Singleton的实例生成

###  饿汉式（静态代码块）
```
public class Singleton {
    private static  Singleton singleton;
	
	static{
		singleton = new Singleton()
	}

    private Singleton(){
    }

    public static Singleton getInstance(){
        return singleton;
    }
}
```
- 在类加载时期就进行初始化，线程安全的实现方式

###  饿汉式（静态内部类）
```
public class SingletonOuter {

    private static class Singleton{
        private static Singleton singleton = new Singleton();
    }

    public static Singleton getInstance(){
        return Singleton.singleton;
    }
    
}
```
- 不调用getInstance方法不会触发单例的实例化，达到延迟初始化的效果

### 懒汉式（双重检查）
```
public class Singleton {
    private static volatile Singleton singleton = null;
    private Singleton(){
    }

    public static Singleton getInstance(){
        if (null == singleton){
            synchronized (Singleton.class){
                if (singleton==null)
                    singleton = new Singleton();
            }
        }
        return singleton;
    }
}
```
- 在类加载阶段不会生成单例，程序不调用即不会生成实例化对象
- 延迟性初始化，第一次初始化较慢，并且增加了null值判断，一定程度上降低了代码的执行效率






