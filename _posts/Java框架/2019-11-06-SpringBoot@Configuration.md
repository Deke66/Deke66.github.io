---
layout: post
title: SpringBoot @Configuration
category: Java框架
tags: spring boot
keywords: spring boot
---
## BeanDefinition注册
- @Configuration标记的class在ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry注册BeanDefinition到beanFactory
- @Configuration标记的class内的@Bean注解方法需要生成的bean同样在ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry会注册BeanDefinition，并指定实例化策略

## 处理入口-ConfigurationClassPostProcessor.postProcessBeanFactory

```
	@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		int factoryId = System.identityHashCode(beanFactory);
		if (this.factoriesPostProcessed.contains(factoryId)) {
			throw new IllegalStateException(
					"postProcessBeanFactory already called on this post-processor against " + beanFactory);
		}
		this.factoriesPostProcessed.add(factoryId);
		if (!this.registriesPostProcessed.contains(factoryId)) {
			// BeanDefinitionRegistryPostProcessor hook apparently not supported...
			// Simply call processConfigurationClasses lazily at this point then.
			processConfigBeanDefinitions((BeanDefinitionRegistry) beanFactory);
		}

		enhanceConfigurationClasses(beanFactory);
		beanFactory.addBeanPostProcessor(new ImportAwareBeanPostProcessor(beanFactory));
	}
```

1. 此方法执行在注册BeanDefinition之后
2. 通过enhanceConfigurationClasses对@Configuration标记的class进行Cglib动态代理增强

## Cglib增强-enhanceConfigurationClasses

```
	public void enhanceConfigurationClasses(ConfigurableListableBeanFactory beanFactory) {
		Map<String, AbstractBeanDefinition> configBeanDefs = new LinkedHashMap<>();
		for (String beanName : beanFactory.getBeanDefinitionNames()) {
			BeanDefinition beanDef = beanFactory.getBeanDefinition(beanName);
			if (ConfigurationClassUtils.isFullConfigurationClass(beanDef)) {
				if (!(beanDef instanceof AbstractBeanDefinition)) {
					throw new BeanDefinitionStoreException("Cannot enhance @Configuration bean definition '" +
							beanName + "' since it is not stored in an AbstractBeanDefinition subclass");
				}
				else if (logger.isInfoEnabled() && beanFactory.containsSingleton(beanName)) {
					logger.info("Cannot enhance @Configuration bean definition '" + beanName +
							"' since its singleton instance has been created too early. The typical cause " +
							"is a non-static @Bean method with a BeanDefinitionRegistryPostProcessor " +
							"return type: Consider declaring such methods as 'static'.");
				}
				configBeanDefs.put(beanName, (AbstractBeanDefinition) beanDef);
			}
		}
		if (configBeanDefs.isEmpty()) {
			// nothing to enhance -> return immediately
			return;
		}

		ConfigurationClassEnhancer enhancer = new ConfigurationClassEnhancer();
		for (Map.Entry<String, AbstractBeanDefinition> entry : configBeanDefs.entrySet()) {
			AbstractBeanDefinition beanDef = entry.getValue();
			// If a @Configuration class gets proxied, always proxy the target class
			beanDef.setAttribute(AutoProxyUtils.PRESERVE_TARGET_CLASS_ATTRIBUTE, Boolean.TRUE);
			try {
				// Set enhanced subclass of the user-specified bean class
				Class<?> configClass = beanDef.resolveBeanClass(this.beanClassLoader);
				if (configClass != null) {
					//增强方法
					Class<?> enhancedClass = enhancer.enhance(configClass, this.beanClassLoader);
					if (configClass != enhancedClass) {
						if (logger.isTraceEnabled()) {
							logger.trace(String.format("Replacing bean definition '%s' existing class '%s' with " +
									"enhanced class '%s'", entry.getKey(), configClass.getName(), enhancedClass.getName()));
						}
						beanDef.setBeanClass(enhancedClass);
					}
				}
			}
			catch (Throwable ex) {
				throw new IllegalStateException("Cannot load configuration class: " + beanDef.getBeanClassName(), ex);
			}
		}
	}
```
1. 从beanFactory获取所有BeanDefinition
2. 过滤剩余@Configuration标记的BeanDefinition
3. 运用Cglib对符合条件的BeanDefinition进行增强

## 	class生成-enhance
- ConfigurationClassEnhancer.enhancer->newEnhancer

```
	private Enhancer newEnhancer(Class<?> configSuperClass, @Nullable ClassLoader classLoader) {
		Enhancer enhancer = new Enhancer();
		enhancer.setSuperclass(configSuperClass);
		enhancer.setInterfaces(new Class<?>[] {EnhancedConfiguration.class});
		enhancer.setUseFactory(false);
		enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
		enhancer.setStrategy(new BeanFactoryAwareGeneratorStrategy(classLoader));
		enhancer.setCallbackFilter(CALLBACK_FILTER);
		enhancer.setCallbackTypes(CALLBACK_FILTER.getCallbackTypes());
		return enhancer;
	}
```
1. 定义生成的父类
2. 增加了EnhancedConfiguration的接口，这个接口继承BeanFactoryAware
3. 定名命令规则
4. 定义class生成策略
5. 设置CallbackFilter与CallbackTypes，主要设置了3个CallBack
	1. BeanMethodInterceptor
	2. BeanFactoryAwareMethodInterceptor
	3. NoOp.INSTANCE
	
## 小结
- @Configuration标记的class在class实例化之前会通过Cglib动态代理的形式增强特定的方法
- 动态代理增强主要改变了@Bean注解方法和setBeanFactory方法，改变了bean的实例化策略

	




