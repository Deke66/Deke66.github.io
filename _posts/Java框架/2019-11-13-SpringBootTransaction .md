---
layout: post
title: Spring 事务
category: Java框架
tags: Transaction
keywords: Spring Transaction
---
## 事务隔离级别
1. TransactionDefinition.ISOLATION_DEFAULT
	- 使用后端数据库默认的隔离级别，Mysql 默认采用的 REPEATABLE_READ隔离级别 Oracle 默认采用的 READ_COMMITTED隔离级别.
2. TransactionDefinition.ISOLATION_READ_UNCOMMITTED 
	- 最低的隔离级别，允许读取尚未提交的数据变更，可能会导致脏读、幻读或不可重复读
3. TransactionDefinition.ISOLATION_READ_COMMITTED	
	- 允许读取并发事务已经提交的数据，可以阻止脏读，但是幻读或不可重复读仍有可能发生
4. TransactionDefinition.ISOLATION_REPEATABLE_READ	
	- 对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，可以阻止脏读和不可重复读，但幻读仍有可能发生。
5. TransactionDefinition.ISOLATION_SERIALIZABLE	
	- 最高的隔离级别，完全服从ACID的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，该级别可以防止脏读、不可重复读以及幻读。但是这将严重影响程序的性能。通常情况下也不会用到该级别
	
- 小结
	1. Spring事务定义了事务的隔离级别，但具体隔离级别的实现在数据库，如果数据库不支持该隔离级别，则使用默认的隔离级别

## 事务传播机制
- 支持当前事务的情况
	1. TransactionDefinition.PROPAGATION_REQUIRED： 如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。
	2. TransactionDefinition.PROPAGATION_SUPPORTS： 如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
	3. TransactionDefinition.PROPAGATION_MANDATORY： 如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。（mandatory：强制性）
- 不支持当前事务的情况
	1. TransactionDefinition.PROPAGATION_REQUIRES_NEW： 创建一个新的事务，如果当前存在事务，则把当前事务挂起。
	2. TransactionDefinition.PROPAGATION_NOT_SUPPORTED： 以非事务方式运行，如果当前存在事务，则把当前事务挂起。
	3. TransactionDefinition.PROPAGATION_NEVER： 以非事务方式运行，如果当前存在事务，则抛出异常。
- 其他情况
	1. TransactionDefinition.PROPAGATION_NESTED： 如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于TransactionDefinition.PROPAGATION_REQUIRED。
	
## Spring事务管理

### 编程式事务管理
1. 实现方式
	1. 使用TransactionTemplate
	2. 直接使用PlatformTransactionManager实现
2. 不采用注解的方式，不在运行期进行增强

### 声明式事务
1. 基于Spring AOP机制，在运行期间动态改变类的行为，添加事务拦截器
2. 主要通过运行期为被事务管理的类添加TransactionInterceptor








	




