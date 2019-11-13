---
layout: post
title: 代理模式
category: Java语言
tags: proxy
keywords: java,proxy
---
## 代理模式

### 概念
- 代理模式：为其他对象提供一种代理以控制对这个对象的访问
- 静态代理是由程序员创建或工具生成代理类的源码，再编译代理类。所谓静态也就是在程序运行前就已经存在代理类的字节码文件，代理类和委托类的关系在运行前就确定了。
- 动态代理是在实现阶段不用关心代理类，而在运行阶段才指定哪一个对象。

### 优点
1. 职责清晰
	- 真实的角色就是实现实际的业务逻辑，不用关心其他非本职责的事务，通过后期的代理完成一件完成事务，附带的结果就是编程简洁清晰。
2. 代理对象可以在客户端和目标对象之间起到中介的作用，这样起到了中介的作用和保护了目标对象的作用。
3. 高扩展性



## 静态代理

### 代码实现

```
public class StaticProxy {
	
    private interface Target{
        void sayHello();
    }
	
	//
    private static class TargetImpl implements Target{

        @Override
        public void sayHello() {
            System.out.println("say hello");
        }
    }
	
    private static class ProxyTarget implements Target{

        private Target target;

        public ProxyTarget(Target target){
            this.target = target;
        }

        @Override
        public void sayHello() {
            this.target.sayHello();
        }
    }

    public static void main(String[] args) {
        ProxyTarget proxyTarget = new ProxyTarget(new TargetImpl());
        proxyTarget.sayHello();
    }
    
}
```
1. 代理类和被代理类需要实现同一接口
2. 被代理类持有代理对象
3. 被代理类可以对代理对象调用方法添加前后处理
4. 代理对象是对被代理对象方法的增强
5. 编译期程序逻辑已确定


## JDK动态代理

### 代码实现

```
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

public class JDKProxy {
    private interface Target{
        void sayHello();
    }

    private static class TargetImpl implements Target {

        @Override
        public void sayHello() {
            System.out.println("say hello");
        }
    }

    private static class JDKDynamicProxy implements InvocationHandler{

        private Object target;

        public JDKDynamicProxy(Object target){
            this.target = target;
        }

        /**
         * 获取被代理接口实例对象
         * @param <T>
         * @return
         */
        public <T> T getProxy() {
            return (T) Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), this);
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            Object result = method.invoke(target, args);
            return result;
        }
    }

    public static void main(String[] args) {
        //保存生成的字节码文件
        System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
        Target target = new JDKDynamicProxy(new TargetImpl()).getProxy();
        target.sayHello();
    }


}
```
1. 代理类和被代理类需要实现同一接口
2. 被代理类不需要持有代理对象
3. 代理对象是对被代理对象方法的增强
4. 代理对象在程序运行期间生成

## Cglib动态代理

### 代码实现

```
package com.example.demo.proxy;

import org.springframework.cglib.core.DebuggingClassWriter;
import org.springframework.cglib.proxy.Enhancer;
import org.springframework.cglib.proxy.MethodInterceptor;
import org.springframework.cglib.proxy.MethodProxy;

import java.lang.reflect.Method;

public class CglibProxy {
    public static class Target{
        public void sayHello() {
            System.out.println("say hello");
        }
    }

    private static class CglibProxyClassMake implements MethodInterceptor {

        @Override
        public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
            Object result = methodProxy.invokeSuper(o,objects);
            return result;
        }

        public <T> T  getProxy(Class<T> target){
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(target);
            enhancer.setCallback(this);
            return  (T)enhancer.create();
        }
    }

    public static void main(String[] args) {
        //将生成的代理类写入文件
        System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "D:/code");
        CglibProxyClassMake cglibProxyClassMake = new CglibProxyClassMake();
        Target target = cglibProxyClassMake.getProxy(Target.class);
        target.sayHello();
    }

}
```
1. 代理类和被代理类不需要实现同一接口
2. 被代理类不可为final类型的类
3. 代理类在程序运行期生成
4. 生成的代理类继承于被代理类






