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

```
beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
```

6.2对属性编辑器支持的两种方式：

```
beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));
```

	方式一：使用自定义属性编辑器，继承propertyEditorSupport,重写setText方法，然后注册到CustomerEditorConfigrer的customerEditors属性中。
	
	方式二：使用spring知道的属性编辑器CustomerDateEditor,定义属性编辑器重写PropertyEditorRegistars中的注册方法registerCustomerEditors()注册到CustomerEditorConfigrer的propertyEditorRegistrars中。

6.3添加applicationContextAwareProcessor处理器，就是将某些实现了***Aware的接口获取相应的资源。

```
beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
```

6.4设置几个自动忽略的接口，形如***Aware

```
//设置几个忽略自动装配的接口
beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
```

6.5设置依赖注入

```
beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
beanFactory.registerResolvableDependency(ResourceLoader.class, this);
beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
beanFactory.registerResolvableDependency(ApplicationContext.class, this);
```

6.6设置后处理器，对bean创建后和销毁前（ApplicationListenerDetector）

```
beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));
```

6.7设置对aspectJ支持

```
if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
   beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
   // Set a temporary ClassLoader for type matching.
   beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
}
```

6.8设置一些系统环境参数

```
//添加默认的系统环境bean
// Register default environment beans.
if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
   beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
}
if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
   beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
}
if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
   beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
}
```

7.对后处理器beanFactoryPostProcessor的应用

BeanDefinitionRegistryPostProcessor是BeanFactoryPostProcessor子类，因为postProcessBeanDefinitionRegistry是用来创建bean定义的，而postProcessBeanFactory是修改BeanFactory，当然postProcessBeanFactory也可以修改bean定义的。

```
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor 
```

执行顺序：首先执行硬编码的后处理器，即beanFactory是registryBeanDefinition并且后处理器是BeanDefinitionRegistryPostProcessor

​		   再获取容器中注册的类型是BeanDefinitionRegistryPostProcessor,按照priortity，order,普通类型执行

​		   再执行容器中的beanPostProcessor按照priorituty,order,普通类型执行

最后执行缓存中的后处理器。

```
public static void invokeBeanFactoryPostProcessors(
      ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

   // Invoke BeanDefinitionRegistryPostProcessors first, if any.
   Set<String> processedBeans = new HashSet<>();
   //对beanDefinitionRegistry类型进行处理
   if (beanFactory instanceof BeanDefinitionRegistry) {
      BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
      //记录通过硬编码方式注册的BeanFactoryPostProcessor
      List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
      //记录通过硬编码的方式注册的BeanDefinitionRegistryPostProcessor
      List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

      for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
         if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
            BeanDefinitionRegistryPostProcessor registryProcessor =
                  (BeanDefinitionRegistryPostProcessor) postProcessor;
            registryProcessor.postProcessBeanDefinitionRegistry(registry);
            registryProcessors.add(registryProcessor);
         }
         else {
            regularPostProcessors.add(postProcessor);
         }
      }

      // Do not initialize FactoryBeans here: We need to leave all regular beans
      // uninitialized to let the bean factory post-processors apply to them!
      // Separate between BeanDefinitionRegistryPostProcessors that implement
      // PriorityOrdered, Ordered, and the rest.
      //不要在这里初始化factorybean:我们需要保留所有常规bean
      //未初始化，让bean工厂的后处理器应用于它们!
      //实现的beandefinitionregistrypostprocessor之间是分开的
      //优先考虑、排序等等。
      List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

      // First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
      //首先，调用实现priorityordated的beandefinitionregistrypostprocessor。
      //这里就是配置的beandefinitionregistrypostprocessor
      String[] postProcessorNames =
            beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
      for (String ppName : postProcessorNames) {
         if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
            processedBeans.add(ppName);
         }
      }
      sortPostProcessors(currentRegistryProcessors, beanFactory);
      registryProcessors.addAll(currentRegistryProcessors);

      invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
      currentRegistryProcessors.clear();
      // Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
      //接下来，调用实现Ordered的beandefinitionregistrypostprocessor。
      postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
      for (String ppName : postProcessorNames) {
         if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
            currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
            processedBeans.add(ppName);
         }
      }
      sortPostProcessors(currentRegistryProcessors, beanFactory);
      registryProcessors.addAll(currentRegistryProcessors);
      invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
      currentRegistryProcessors.clear();

      // Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
      //最后，调用所有其他beandefinitionregistrypostprocessor，直到不再出现其他处理器为止。
      boolean reiterate = true;
      while (reiterate) {
         reiterate = false;
         postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
         for (String ppName : postProcessorNames) {
            if (!processedBeans.contains(ppName)) {
               currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
               processedBeans.add(ppName);
               reiterate = true;
            }
         }
         sortPostProcessors(currentRegistryProcessors, beanFactory);
         registryProcessors.addAll(currentRegistryProcessors);
         invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
         currentRegistryProcessors.clear();
      }

      // 现在，调用到目前为止处理的所有处理器的postProcessBeanFactory回调。
      invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
      invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
   }

   else {
      // Invoke factory processors registered with the context instance.
      //调用在上下文实例中注册的工厂处理器。
      invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
   }

   // Do not initialize FactoryBeans here: We need to leave all regular beans
   // uninitialized to let the bean factory post-processors apply to them!
   //不要在这里初始化factory bean:我们需要保留所有常规bean
   //未初始化，让bean工厂的后处理器应用于它们!
   String[] postProcessorNames =
         beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

   // Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
   // Ordered, and the rest.
   //实现priorityorated的beanfactorypostprocessor之间的分离，
   List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
   List<String> orderedPostProcessorNames = new ArrayList<>();
   List<String> nonOrderedPostProcessorNames = new ArrayList<>();
   for (String ppName : postProcessorNames) {
      if (processedBeans.contains(ppName)) {
         // skip - already processed in first phase above
      }
      else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
         priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
      }
      else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
         orderedPostProcessorNames.add(ppName);
      }
      else {
         nonOrderedPostProcessorNames.add(ppName);
      }
   }

   // First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
   //首先，调用实现priorityorated的beanfactorypostprocessor。
   sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
   invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

   // Next, invoke the BeanFactoryPostProcessors that implement Ordered.
   //接下来，调用实现Ordered的beanfactorypostprocessor。
   List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>();
   for (String postProcessorName : orderedPostProcessorNames) {
      orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
   }
   sortPostProcessors(orderedPostProcessors, beanFactory);
   invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

   // Finally, invoke all other BeanFactoryPostProcessors.
   //最后，调用所有其他beanfactorypostprocessor。
   List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
   for (String postProcessorName : nonOrderedPostProcessorNames) {
      nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
   }
   invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

   // Clear cached merged bean definitions since the post-processors might have
   // modified the original metadata, e.g. replacing placeholders in values...
   //清除缓存的合并bean定义，因为后处理程序可能有
   //修改原始元数据，例如在值中替换占位符……
   beanFactory.clearMetadataCache();
}
```

8.对beanPostProcessor进行注册操作

```
/** BeanPostProcessors to apply in createBean. */
//beanPostprocessors存储的位置，即进行注册
private final List<BeanPostProcessor> beanPostProcessors = new CopyOnWriteArrayList<>();
```

```
// 注册拦截bean创建的bean处理器，这里只是注册，真正调用是在getBean的时候
registerBeanPostProcessors(beanFactory);
```

将beanPostProcessor按照事先接口priority,order,普通这三种方式进行注册。

9.初始化消息资源{国际化配置操作}

ResourceBundleMessageSources和ReloadbleResourceBundleMessageSources可以通过资源名来加载国际化资源。默认是英文

```
/**
 * Initialize the MessageSource.
 * Use parent's if none defined in this context.
 * 初始化MessageSource。
 * 如果在这个上下文中没有定义，请使用父类
 */
protected void initMessageSource() {
   ConfigurableListableBeanFactory beanFactory = getBeanFactory();
   if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {
      //如果配置中已经配置了messageSources,那么将message提取并防止在this.messageSources中
      this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
      // Make MessageSource aware of parent MessageSource.
      //使MessageSource感知父MessageSource。
      if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
         HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
         if (hms.getParentMessageSource() == null) {
            // Only set parent context as parent MessageSource if no parent MessageSource
            // registered already.
            //如果没有父MessageSource，则只将父上下文设置为父MessageSource
            //注册了
            hms.setParentMessageSource(getInternalParentMessageSource());
         }
      }
      if (logger.isTraceEnabled()) {
         logger.trace("Using MessageSource [" + this.messageSource + "]");
      }
   }
   else {
      // Use empty MessageSource to be able to accept getMessage calls.
      //使用空MessageSource能够接受getMessage调用。
      //如果用户并没有定义配置文件，那么使用临时的DelegatingMessageSource以便于作为getMessage()方法的返回
      DelegatingMessageSource dms = new DelegatingMessageSource();
      dms.setParentMessageSource(getInternalParentMessageSource());
      this.messageSource = dms;
      beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
      if (logger.isTraceEnabled()) {
         logger.trace("No '" + MESSAGE_SOURCE_BEAN_NAME + "' bean, using [" + this.messageSource + "]");
      }
   }
}
```

10注册监听器ApplicationEventMulticaster{主要功能是添加删除监听器，发布事件给响应的监听器}

```
/**
 * Initialize the ApplicationEventMulticaster.
 * Uses SimpleApplicationEventMulticaster if none defined in the context.
 * 初始化ApplicationEventMulticaster。
 * 如果上下文中没有定义，则使用simpleapplicationeventmultiaster。
 * @see org.springframework.context.event.SimpleApplicationEventMulticaster
 */
protected void initApplicationEventMulticaster() {
   ConfigurableListableBeanFactory beanFactory = getBeanFactory();
   if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
      this.applicationEventMulticaster =
            beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
      if (logger.isTraceEnabled()) {
         logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
      }
   }
   else {
      this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
      beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
      if (logger.isTraceEnabled()) {
         logger.trace("No '" + APPLICATION_EVENT_MULTICASTER_BEAN_NAME + "' bean, using " +
               "[" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
      }
   }
}
```

11.为ApplicationEventMulticaster容器添加监听器

```
/**
 * 添加实现ApplicationListener为侦听器的bean。
 * 不影响其他监听器，可以在不使用bean的情况下添加监听器。
 */
protected void registerListeners() {
   // Register statically specified listeners first.
   //首先注册静态指定的侦听器{硬编码方式}
   for (ApplicationListener<?> listener : getApplicationListeners()) {
      getApplicationEventMulticaster().addApplicationListener(listener);
   }

   // Do not initialize FactoryBeans here: We need to leave all regular beans
   // uninitialized to let post-processors apply to them!
   //不要在这里初始化factory bean:我们需要保留所有常规bean
   //未初始化以让后处理器应用于它们!{配置中获取}
   String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
   for (String listenerBeanName : listenerBeanNames) {
      getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
   }

   // Publish early application events now that we finally have a multicaster...
   //发布早期的应用程序事件，现在我们终于有了多主机……
   Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
   this.earlyApplicationEvents = null;
   if (earlyEventsToProcess != null) {
      for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
         getApplicationEventMulticaster().multicastEvent(earlyEvent);
      }
   }
}
```

12.冻结配置，提前实例化一些非抽象，单例，不是延迟加载的bean

```
finishBeanFactoryInitialization(beanFactory);
```

13.主要干两件事，第一件：初始化bean的生命周期处理器，第二件：发出contextRefreshEvent给相应的监听器

```
finishRefresh();
```