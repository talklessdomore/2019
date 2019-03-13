### bean的创建过程

1.入口

这个是接着bean的加载过程进行分析

```
return createBean(beanName, mbd, args);
```

2.使用bean的定义mbd和传入参数beanName来获取class,这块可以参考一下AbstractBeanDefinition,这个对象中提供了获取class的方法getBeanClass()，完成参数的设置。

```
Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
```

3.这个方法就是获取该方法下所有overied方法，然后获取所有的接口和父类的方法，如果方法名大于1的该方法的overloaded属性就是true,进行记录。

```
mbdToUse.prepareMethodOverrides();

```

4.实例化前后处理程序已启动 对beanDefinition的一些参数进行处理

```
Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
```

上面4步可以总结为就是对该beanDefnition的一些预处理操作：如覆盖方法的属性标志，创建实例化前处理器属性的标志。

5.进行实际创建步骤

```
Object beanInstance = doCreateBean(beanName, mbdToUse, args);
```

6.首先是要从缓存中进行获取

```
// 实例化bean。
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			//未完成FactoryBean实例的缓存:将FactoryBean名称缓存到BeanWrapper{清除缓存}
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
```

这里面要解释一下BeanWrapper{Spring底层javabean基础设施的核心接口。 }

7.实例化一个bean的实例，实例化bean的实例基本上有三种方式：

第一种：使用该bean的工厂

第二种：带有参数的构造方法{这个比较麻烦，需要进行参数的解析和构造函数的解析和匹配}

第三种：默认参数的构造

```
/**
	 * 使用适当的实例化策略为指定的bean创建一个新实例:
	 * 工厂方法、构造函数自动连接或简单实例化。
	 * @param beanName bean的名字
	 * @param mbd bean的定义
	 * @param args 用于构造函数或工厂方法调用的显式参数
	 * @return 新实例的BeanWrapper
	 * @see #obtainFromSupplier
	 * @see #instantiateUsingFactoryMethod
	 * @see #autowireConstructor
	 * @see #instantiateBean
	 */
	protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
		// 确保bean类在这一点上得到了实际的解析。解析beanName获取到beanClass
		Class<?> beanClass = resolveBeanClass(mbd, beanName);

		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}
		//获取bean创建的回调如果有的话
		Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
		if (instanceSupplier != null) {
			return obtainFromSupplier(instanceSupplier, beanName);
		}
		//如果工厂方式不为空则用工厂方式创建
		if (mbd.getFactoryMethodName() != null) {
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

		// 重新创建相同bean时的快捷方式…
		boolean resolved = false;
		boolean autowireNecessary = false;
		//进行参数解析
		if (args == null) {
			synchronized (mbd.constructorArgumentLock) {
				// 一个类有多个构造函数，每个构造函数都有不同的参数，所以调用前需要先根据参数锁定构造函数或对应的工厂方法
				if (mbd.resolvedConstructorOrFactoryMethod != null) {
					resolved = true;
					autowireNecessary = mbd.constructorArgumentsResolved;
				}
			}
		}
		//如果已经解析过则使用解析好的构造函数方法不需要再次锁定
		if (resolved) {
			if (autowireNecessary) {
				//参数构造解析方式
				return autowireConstructor(beanName, mbd, null, null);
			}
			else {
				//默认解析方式
				return instantiateBean(beanName, mbd);
			}
		}

		// 解析构造参数
		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
			//构造函数自动注入
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		// Preferred constructors for default construction?
		ctors = mbd.getPreferredConstructors();
		if (ctors != null) {
			return autowireConstructor(beanName, mbd, ctors, null);
		}

		// 无需特殊处理:只需使用无参数构造函数。
		return instantiateBean(beanName, mbd);
	}
```

8.合并bean的定义

```
synchronized (mbd.postProcessingLock) {
   if (!mbd.postProcessed) {
      try {
         //将MergedBeanDefinitionPostProcessors应用于指定的bean定义
         applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
      }
      catch (Throwable ex) {
         throw new BeanCreationException(mbd.getResourceDescription(), beanName,
               "Post-processing of merged bean definition failed", ex);
      }
      mbd.postProcessed = true;
   }
}
```



9.是否添加到单例工厂

```
boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isTraceEnabled()) {
				logger.trace("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}
```

10.属性的注入，分为两种，一种是按名称注入，一种是按类型注入

```
populateBean(beanName, mbd, instanceWrapper);
```

①按名称注入

```
/**
 * 在传入的参数中找到该bean并实例化初始化属性参数
 * 用引用填充任何缺失的属性值
 * *如果autowire设置为“byName”，则此工厂中的其他bean。
 * @param beanName 我们正在连接的bean的名称。
 * Useful for debugging messages; not used functionally.
 * @param mbd 通过自动连接更新bean定义
 * @param bw 我们可以从中获取关于bean的信息的bean包装器
 * @param pvs 用于注册连接对象的PropertyValues
 */
protected void autowireByName(
      String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {

   String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
   for (String propertyName : propertyNames) {
      if (containsBean(propertyName)) {
         Object bean = getBean(propertyName);
         pvs.add(propertyName, bean);
         registerDependentBean(propertyName, beanName);
         if (logger.isTraceEnabled()) {
            logger.trace("Added autowiring by name from bean name '" + beanName +
                  "' via property '" + propertyName + "' to bean named '" + propertyName + "'");
         }
      }
      else {
         if (logger.isTraceEnabled()) {
            logger.trace("Not autowiring property '" + propertyName + "' of bean '" + beanName +
                  "' by name: no matching bean found");
         }
      }
   }
}
```

②按类型注入

```
/**
 * 抽象方法定义“按类型自动连接”(按类型定义bean属性)行为。
 * 这类似于PicoContainer默认值，其中必须只有一个bean
 * *表示bean工厂中的属性类型。这使得bean工厂非常简单
 * *配置小的名称空间，但不能像标准Spring那样工作
 * *大型应用程序的行为。
 * @param beanName the name of the bean to autowire by type
 * @param mbd the merged bean definition to update through autowiring
 * @param bw the BeanWrapper from which we can obtain information about the bean
 * @param pvs the PropertyValues to register wired objects with
 */
protected void autowireByType(
      String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {

   TypeConverter converter = getCustomTypeConverter();
   if (converter == null) {
      converter = bw;
   }

   Set<String> autowiredBeanNames = new LinkedHashSet<>(4);
   //寻找bw中需要进行依赖注入的属性
   String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
   for (String propertyName : propertyNames) {
      try {
         PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
         // Don't try autowiring by type for type Object: never makes sense,
         // even if it technically is a unsatisfied, non-simple property.
         if (Object.class != pd.getPropertyType()) {
            //探测指定属性的set方法
            MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
            // 在优先级后处理器的情况下，不允许立即初始化进行类型匹配。
            boolean eager = !PriorityOrdered.class.isInstance(bw.getWrappedInstance());
            DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager);
            Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter);
            // 解析指定 beanName 的属性所匹配的值，并把解析到的属性名称存储在
            // autowireBeanNames 中，当属性存在多个封装 bean 时，如:
            // @Autowired private List<A> aList; 将会找到所有匹配 A 类型
            // 的 bean 并将其注入
            if (autowiredArgument != null) {
               //将bean中属性匹配类型添加进去
               pvs.add(propertyName, autowiredArgument);
            }
            for (String autowiredBeanName : autowiredBeanNames) {
               //注册依赖
               registerDependentBean(autowiredBeanName, beanName);
               if (logger.isTraceEnabled()) {
                  logger.trace("Autowiring by type from bean name '" + beanName + "' via property '" +
                        propertyName + "' to bean named '" + autowiredBeanName + "'");
               }
            }
            autowiredBeanNames.clear();
         }
      }
      catch (BeansException ex) {
         throw new UnsatisfiedDependencyException(mbd.getResourceDescription(), beanName, propertyName, ex);
      }
   }
}
```

①根据名称注入，首先获取该bean所需要进行注入的所有属性

然后根据getBean方法循环递归实例化并加入到pvs中

②根据类型注入，首先获取该bean所需要进行注入的所有属性

然后循环获取每个属性的解释器

然后根据解释器探测每个属性的setter方法

解析指定 beanName 的属性所匹配的值，并把解析到的属性名称存储在autowireBeanNames，并加入到pvs中

最终会设置到beanWapper的properties参数中去。

11.bean的创建，这个过程会有初始化方法和一些后处理器的作用

```
exposedObject = initializeBean(beanName, exposedObject, mbd);
```

12.如果该bean存在早起的应用需要进行依赖的检测

```
if (earlySingletonExposure) {
   Object earlySingletonReference = getSingleton(beanName, false);
   if (earlySingletonReference != null) {
      //如果该bean没有被增强
      if (exposedObject == bean) {
         exposedObject = earlySingletonReference;
      }
      //如果没增强，则获取它所有的依赖处理
      //在循环引用时是否采用注入生bean实例;确定是否为给定名称注册了依赖bean
      //不允许注入原生的bean，并为给定名称注册依赖bean
      //进行的则是依赖检测
      else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
         String[] dependentBeans = getDependentBeans(beanName);
         Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
         for (String dependentBean : dependentBeans) {
            if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
               actualDependentBeans.add(dependentBean);
            }
         }
         if (!actualDependentBeans.isEmpty()) {
            throw new BeanCurrentlyInCreationException(beanName,
                  "Bean with name '" + beanName + "' has been injected into other beans [" +
                  StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                  "] in its raw version as part of a circular reference, but has eventually been " +
                  "wrapped. This means that said other beans do not use the final version of the " +
                  "bean. This is often the result of over-eager type matching - consider using " +
                  "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
         }
      }
   }
}
```

13.就是将disposableBeans添加一个记录

```
// 将bean按照scope进行注册
try {
   registerDisposableBeanIfNecessary(beanName, bean, mbd);
}
```

```
/** Disposable bean instances: bean name to disposable instance. */
private final Map<String, Object> disposableBeans = new LinkedHashMap<>();
```