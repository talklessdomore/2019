### 容器的工程扩展

applicationContext包含了对beanFactory的所有功能，并且做了相应的扩展。

1.入口

```
public ClassPathXmlApplicationContext(String... configLocations) throws BeansException {
   this(configLocations, true, null);
}
```

2.对xml配置地址的解析

```
//1.加载配置文件第一步
setConfigLocations(configLocations);
```

3.整体概要

```
@Override
public void refresh() throws BeansException, IllegalStateException {
   synchronized (this.startupShutdownMonitor) {
      // 准备刷新的上下文环境
      /**
       * 对系统属性和环境变量初始化验证
       */
      prepareRefresh();

      // 初始化beanFactory,并进行对xml文件进行读取
      /**
       * 1.创建DefaultListableBeanFactory
       * 2.为了序列化指定id
       * 3.设置相关属性，包括是否可以覆盖同名的但不同定义的对象，以及循环依赖
       * 4.为该beanFactory创建XmlBeanDefinitionReader并读取所有bean的定义文件
       */
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      // 对beanFactory进行各种功能的补充
      /*增添功能
            * 1.增加了对spel语言的支持
            * 2.增加了对属性编辑器的支持
            * 3.增加了一些内置类比如对EnvironmentAware和MessageSourceAware的信息注入
            * 4.设置了依赖功能可忽略接口
            * 5.注册一些固定依赖的属性
            * 6.增加aspectj支持
            * 7.将相关环境变量和属性注册以单例模式注册
         **/
      prepareBeanFactory(beanFactory);

      try {
         // 子类覆盖方法做额外处理
         postProcessBeanFactory(beanFactory);

         // 激活各种beanFactory处理器
         invokeBeanFactoryPostProcessors(beanFactory);

         // 注册拦截bean创建的bean处理器，这里只是注册，真正调用是在getBean的时候
         registerBeanPostProcessors(beanFactory);

         // 为上下文初始化message源，即不同语言的消息体，国际化处理
         initMessageSource();

         // Initialize event multicaster for this context.
         //初始化应用消息广播器并放入“applicationEvenMulticaster”中
         initApplicationEventMulticaster();

         // Initialize other special beans in specific context subclasses.
         //留给子类来初始化其他bean
         onRefresh();

         // Check for listener beans and register them.
         //在所有的bean中找listener Bean将其注册到消息广播器中
         registerListeners();

         // Instantiate all remaining (non-lazy-init) singletons.
         //初始化剩下的单实例
         /**
          * conversionService的设置
          * 配置冻结
          * 非延迟加载,实例化所有的单例bean
          */
         finishBeanFactoryInitialization(beanFactory);

         // Last step: publish corresponding event.
         //完成刷新过程，通知生命周期处理器lifeCycleProcessor刷新过程，同事发出contextRefreshEvent通知别人
         finishRefresh();
      }

      catch (BeansException ex) {
         if (logger.isWarnEnabled()) {
            logger.warn("Exception encountered during context initialization - " +
                  "cancelling refresh attempt: " + ex);
         }

         // Destroy already created singletons to avoid dangling resources.
         destroyBeans();

         // Reset 'active' flag.
         cancelRefresh(ex);

         // Proparefreshgate exception to caller.
         throw ex;
      }

      finally {
         // Reset common introspection caches in Spring's core, since we
         // might not ever need metadata for singleton beans anymore...
         resetCommonCaches();
      }
   }
}
```

4.对系统环境参数验证

```
/**
 * 对系统属性和环境变量初始化验证
 */
prepareRefresh();

//这里的this是getEnvirment(),requiredProperties是必要的参数集合
	@Override
	public void validateRequiredProperties() {
		MissingRequiredPropertiesException ex = new MissingRequiredPropertiesException();
		for (String key : this.requiredProperties) {
			if (this.getProperty(key) == null) {
				ex.addMissingRequiredProperty(key);
			}
		}
		if (!ex.getMissingRequiredProperties().isEmpty()) {
			throw ex;
		}
	}
```

5.对beanFactory的获取

```
// 初始化beanFactory,并进行对xml文件进行读取
/**
 * 1.创建DefaultListableBeanFactory
 * 2.为了序列化指定id
 * 3.设置相关属性，包括是否可以覆盖同名的但不同定义的对象，以及循环依赖
 * 4.为该beanFactory创建XmlBeanDefinitionReader并读取所有bean的定义文件
 */
ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
```

```
//创建configurableListableBeanFactory
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
   //初始化beanFactory并读取xml文件，并将得到的beanFactory记录在当前实体的属性中
   refreshBeanFactory();
   //返回当前实体的beanFactory属性
   return getBeanFactory();
}
```

![1552543883746](C:\Users\XIECHE~1.INV\AppData\Local\Temp\1552543883746.png)

6.对beanFactory进行各种功能的补充

```
// 对beanFactory进行各种功能的补充
/*增添功能
      * 1.增加了对spel语言的支持
      * 2.增加了对属性编辑器的支持
      * 3.增加了一些内置类比如对EnvironmentAware和MessageSourceAware的信息注入
      * 4.设置了依赖功能可忽略接口
      * 5.注册一些固定依赖的属性
      * 6.增加aspectj支持
      * 7.将相关环境变量和属性注册以单例模式注册
   **/
prepareBeanFactory(beanFactory);
```

6.1对el表达式的支持

进行注入是在属性填充的时候会调用applyPropertyValues()

解析是在beanDefinitionValueResolver类型实例valueResolver来解析

6.2对属性编辑器支持的两种方式：

​	方式一：使用自定义属性编辑器，继承propertyEditorSupport,重写setText方法，然后注册到CustomerEditorConfigrer的customerEditors属性中。

​	方式二：使用spring知道的属性编辑器CustomerDateEditor,定义属性编辑器重写PropertyEditorRegistars中的注册方法registerCustomerEditors()注册到CustomerEditorConfigrer的propertyEditorRegistrars中。

6.3添加applicationContextAwareProcessor处理器，就是将某些实现了***Aware的接口获取相应的资源。

6.4设置几个自动忽略的接口，形如***Aware

6.5设置依赖注入

