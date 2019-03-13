### bean的加载过程

![img](C:/Users/xiechenglin.INVAULT/AppData/Local/YNote/data/qq62D1266CF704E0F9820D534BA21B1D72/2365ce68be87487cb931f3d1bf4a440e/clipboard.png) 





1.入口

```
ITestBean testBean = (ITestBean) factory.getBean("proxyFactory1");
```

2.对bean的名称进行解析

```
		//取回实际beanName
		final String beanName = transformedBeanName(name);
```

3.从缓存中来获取bean

```
	/**
		 * 1.首先是在缓存中获取
		 * 2.如果没有查询该bean的状态，如果是正在创建状态，在早期引用中获取
		 * 3.如果早期引用中没有则是否创建早期引用，如果可以获取该bean的工厂，创建早期引用并返回
		 */
		Object sharedInstance = getSingleton(beanName);
```

这里面主要介绍几个缓存map，从这几个缓存map中获取到的bean的实例对象有可能是bean的实例工程，也有可能就是bean实例的本身，所以需要进行bean的对象获取流程

```
	/** Cache of singleton objects: bean name to bean instance. */
	//以bean的名称作为键，然后以bean的实例作为值得缓存
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
	
	/** Cache of early singleton objects: bean name to bean instance. */
	/*以bean的名称作为键，然后以bean的实例作为值得缓存,不过这个缓存的作用在于早起引用，主要防止循环依赖		*的发生
	 */
	private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
	
	/** Cache of singleton factories: bean name to ObjectFactory. */
	/** 以bean的名称作为键，然后以bean的工厂实例作为值得缓存,这个应用场景主要是用于在允许创建早期引用，	  *	并且前面两个缓存都获取失败的情况下
	  */
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
```

4.bean对象的获取

其实这个方法的目的就是获取我们指定名称的object对象，但我们制定的名称有可能是直接的名称如：object也有可能是想要获取该bean的工厂类&object,所以它先过滤掉空工厂类和不符合调的工厂类，然后如果该实例是object类型或者是想要获取的是工厂类，这样就直接返回，如果不是则就是从工厂中获取的。

```
protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

		// 这里面是工厂引用，所需要的是一个工厂，对工厂条件的一种过滤，如果为空或者说该bean不是工厂实例则过滤掉，即直接返回
		if (BeanFactoryUtils.isFactoryDereference(name)) {
			//如果该工厂为空则直接返回
			if (beanInstance instanceof NullBean) {
				return beanInstance;
			}
			//如果该beanInstance不是工厂实例则抛异常
			if (!(beanInstance instanceof FactoryBean)) {
				throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
			}
		}
		
		//根据上面一层的过滤，如果说我们的beanInstance不是工厂bean或者想要获取的是工厂bean{这里面的工厂bean是不为空的}直接返回
		if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
			return beanInstance;
		}
		/**
		 * 根据上面两层走下来，可以确定它是工厂类，并且它想获取的是它的实例object，而非工厂实例
		 */
		//到这里可以证明该bean是个工厂bean
		//在缓存中加载工厂bean
		Object object = null;
		if (mbd == null) {
			//由FactoryBean创建的单例对象的缓存:FactoryBean名称到对象。factoryBeanObjectCache
			object = getCachedObjectForFactoryBean(beanName);
		}
		if (object == null) {
			// 从工厂返回bean实例.
			FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
			// 缓存从FactoryBean获得的对象(如果它是单例的).
			//bean定义对象的映射，按bean名称键控。
			if (mbd == null && containsBeanDefinition(beanName)) {
				mbd = getMergedLocalBeanDefinition(beanName);
			}
			boolean synthetic = (mbd != null && mbd.isSynthetic());
			//从给定的FactoryBean获取要公开的对象。
			object = getObjectFromFactoryBean(factory, beanName, !synthetic);
		}
		return object;
	}

```

以上4步骤是我们从缓存中获取实例对象的，但如果我们无法从缓存中获取对象呢？

有两种方式：方式一：从beanFactory中获取，beanFactory的作用在于获取bean和bean各种属性信息。

​		       方式二：获取beanDefinition解决好依赖后根据不同的条件进行创建beanInstances然后进行获取。