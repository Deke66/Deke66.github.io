---
layout: post
title: Java模板方法模式
category: Java语言
tags: Template
keywords: Template
---
## 模板方法模式
- 一个抽象类公开定义了执行它的方法的方式/模板。它的子类可以按需要重写方法实现，但调用将以抽象类中定义的方式进行。

## 模板类

```
public abstract class HummerModel { 
 
 protected abstract void start(); 
  
 //能发动，那还要能停下来，那才是真本事 
 protected abstract void stop(); 
  
 //喇叭会出声音，是滴滴叫，还是哔哔叫 


 protected abstract void alarm(); 
  
 //引擎会轰隆隆的响，不响那是假的 
 protected abstract void engineBoom(); 
  
 //那模型应该会跑吧，别管是人推的，还是电力驱动，总之要会跑  
 final public void run() { 
   
  //先发动汽车 
  this.start(); 
   
  //引擎开始轰鸣 
  this.engineBoom(); 
   
  //喇嘛想让它响就响，不想让它响就不响 
  if(this.isAlarm()){ 
   this.alarm(); 
  } 
     
  //到达目的地就停车 
  this.stop(); 
 } 
  
 //钩子方法，默认喇叭是会响的 
 protected  boolean isAlarm(){ 
  return true; 
 } 
}
```

- final方法run为主要方法，其他方法在子类实现

## 模板类实现类

```
public class HummerH2Model extends HummerModel { 
 
 @Override 
 protected void alarm() { 
  System.out.println("悍马H2鸣笛..."); 
 } 
 
 @Override 
 protected void engineBoom() {  
  System.out.println("悍马H2引擎声音是这样在..."); 
 } 
 
 @Override 
 protected void start() { 
  System.out.println("悍马H2发动..."); 
 } 
 
 @Override 
 protected void stop() { 
  System.out.println("悍马H1停车..."); 
 } 
  
  
 //默认没有喇叭的 
 @Override 
 protected boolean isAlarm() {   
  return false; 
 } 
  
} 

```

## 测试

```
public class Client { 
 
 public static void main(String[] args) { 
  HummerH2Model h2 = new HummerH2Model(); 
  h2.run();  //H2型号的悍马跑起来 
 } 
 
} 
```

## Spring中模板方法
- RedisTemplate，JdbcTemplate等

## 总结
- 模板方法模式是父类定义模板，子类实现具体方法，以达到子类可以复用父类的代码


## 参考
- 设计模式之禅















