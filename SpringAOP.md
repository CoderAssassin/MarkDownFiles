[TOC]
## 什么是AOP
AOP(Aspect-Oriented Programming，面向切面编程)是Spring的两大特性之一。在C语言中，代码都要写在main函数中，代码多了以后可读性很差；后来就有了模块化的语言，将不同的模块分离开来，使得代码组织起来更加整齐和清晰，开发效率高了很多。但是虽然利用面向对象的方法可以很好地组织代码，也可以通过继承关系实现代码重用，但是程序中总会出现一些重复的代码，而且不太方便使用，虽然可以将这些重复的代码封装成公共函数，但是通过显示调用它们不是很方便，需要维护代码，不是很灵活。因此就有了AOP，AOP将这些重复的代码抽离出来单独维护，在需要使用的时候统一调用，而且使用的时候更加灵活。

## AOP的主要组成
#### Advice通知
Advice定义了在连接点做什么，为切面增强提供织入接口。具体的通知类型有BeforeAdvice、AfterAdvice和ThrowAdvice等
#### Pointcut切点
Pointcut决定Advice通知应该作用于哪个连接点，通过Pointcut来定义需要增强的方法的集合。
Pointcut的接口继承体系比较多，在根接口Pointcut中定义了需要返回**MethodMatcher对象，它主要是用来判断方法是否匹配的**，而具体的如何判断方法是否匹配的方式由子类去实现，比如JdkRegexpMethodPointcut通过正则表达式对方法名进行匹配，AnnotationMatchingPointcut通过定义注解的方式进行匹配，NameMatchMethodPointcut通过方法名进行匹配。在**MethodMatcher中定义了matches()接口方法，这个方法是具体的匹配接口方法。**
#### Advisor通知器
Advisor定义应该使用哪个通知并在哪个关注点使用它，是Advice和Pointcut的结合。

## AOP的设计
说到AOP，就会想到代理，**AOP的核心技术是动态代理**，而动态代理主要有两种，一种是对接口进行代理的**JDK动态代理**，一种是对非final类进行代理的**CGLIB动态代理**，在JDK动态代理中，具体是通过**反射**来调用对象的方法，CGLIB动态代理是基于ASM字节码技术来实现的，是对字节码文件直接操作的。和JDK动态代理紧密相关的一个接口是**HandlerInvocation**，里面定义了回调方法**invoke()**；和CGLIB动态代理密切相关的接口是**MethodInterception**，里面主要的方法是**intercept()**。

## AOP代理对象设计原理
![ProxyFactory](https://github.com/CoderAssassin/markdownImg/blob/master/Spring/AOP_ProxyFactory.png?raw=true)
上图是AOP代理核心的类继承体系。
ProxyConfig里是一些基本的方法和数据；AdvisedSupport封装了AOP对通知和通知器的相关操作；ProxyCreatorSupport是子类创建AOP代理的一个辅助；AspectJProxyFactory是将Spring和AspectJ结合；ProxyFactory是Spring编程式的代理方式；ProxyFactoryBean是Spring的声明式代理。

**下面的原理讲解都是基于spring 5.0.5版本的源码！！！**

## ProxyFactoryBean原理
ProxyFactoryBean支持声明式的生成代理对象的方式，因此比编程式的更加灵活，这里以ProxyFactoryBean为例讲解。
#### 基本XML配置方式
``` xml
<bean id="MyAdvisor" class="com.wy.MyAdvisor"/>
<bean id="MyAOP" class="org.springframework.aop.ProxyFactoryBean">
	<property name="proxyInterface">
    	<value>com.wy.MyInterface</value>
    </property>
    <property name="target">
    	<bean class="com.wy.MyTartget"/>
    </property>
    <property name="interceptorNames">
    	<list>
        	<value>
            	MyAdvisor
            </value>
        </list>
    </property>
</bean>
```
配置一个Advisor通知器，一个ProxyFactoryBean，在ProxyFactoryBean里有3个配置属性，一个是代理的对象接口，一个是目标对象，还一个是通知器。

#### 生成AopProxy
有了配置信息，但是底层是如何生成代理对象的我们还不清楚，接下来我们一层层剖析ProxyFactoryBean是如何生成代理对象的。首先，我们要知道底层是通过JDK动态代理或者CGLUB动态代理来实现的。
**ProxyFactoryBean通向底层代理对象生成的总入口是getObject()方法。**
``` java
public Object getObject() throws BeansException {
		//初始化通知器链
        this.initializeAdvisorChain();
        //生成单例的代理对象
        if (this.isSingleton()) {
            return this.getSingletonInstance();
        } else {
            if (this.targetName == null) {
                this.logger.warn("Using non-singleton proxies with singleton targets is often undesirable. Enable prototype proxies by setting the 'targetName' property.");
            }
			//创建新的代理对象实例
            return this.newPrototypeInstance();
        }
    }
```
从总入口getObject()可以看到，这里总的做了两件事：

* 初始化通知器链，也就是一系列的拦截器，从配置文件中读取
* 创建代理对象，这里分两种方式创建
	* 获取单例代理对象
	* 创建新的代理对象

接下来进入ProxyFactoryBean的**initializeAdvisorChain()方法**看下具体是怎么获取通知器链的：
``` java
private synchronized void initializeAdvisorChain() throws AopConfigException, BeansException {
		//通知器链只初始化一次，已经初始化过的不需要再进行初始化
        if (!this.advisorChainInitialized) {
            if (!ObjectUtils.isEmpty(this.interceptorNames)) {
                if (this.beanFactory == null) {
                    throw new IllegalStateException("No BeanFactory available anymore (probably due to serialization) - cannot resolve interceptor names " + Arrays.asList(this.interceptorNames));
                }

                if (this.interceptorNames[this.interceptorNames.length - 1].endsWith("*") && this.targetName == null && this.targetSource == EMPTY_TARGET_SOURCE) {
                    throw new AopConfigException("Target required after globals");
                }

                String[] var1 = this.interceptorNames;
                int var2 = var1.length;
				//遍历每个拦截器名字
                for(int var3 = 0; var3 < var2; ++var3) {
                    String name = var1[var3];
                    if (this.logger.isTraceEnabled()) {
                        this.logger.trace("Configuring advisor or advice '" + name + "'");
                    }
					//名字以*结尾的是全局拦截器，添加匹配的全局拦截器
                    if (name.endsWith("*")) {
                        if (!(this.beanFactory instanceof ListableBeanFactory)) {
                            throw new AopConfigException("Can only use global advisors or interceptors with a ListableBeanFactory");
                        }

                        this.addGlobalAdvisor((ListableBeanFactory)this.beanFactory, name.substring(0, name.length() - "*".length()));
                    } else {
                        Object advice;
                        //原型类型
                        if (!this.singleton && !this.beanFactory.isSingleton(name)) {
                            advice = new ProxyFactoryBean.PrototypePlaceholderAdvisor(name);
                        } else {//单例类型
                            advice = this.beanFactory.getBean(name);
                        }
						//添加通知器到通知器链
                        this.addAdvisorOnChainCreation(advice, name);
                    }
                }
            }

            this.advisorChainInitialized = true;
        }
    }
```
上面代码是初始化通知器链的过程，总结一下就是遍历配置好的通知器名字，然后对于每个通知器，若是以\*结尾的全局通知器，那么获取匹配的全局通知器加入；否则的话，若是单例那么调用getBean()，这个就是获取bean的入口，若是非单例的话，new一个。
了解了initializeAdvisorChain()的原理后，接下来主要看**ProxyFactoryBean的生成单例的AopProxy对象的入口函数getSingletonInstance()**：
``` java
private synchronized Object getSingletonInstance() {
        if (this.singletonInstance == null) {
            this.targetSource = this.freshTargetSource();
            if (this.autodetectInterfaces && this.getProxiedInterfaces().length == 0 && !this.isProxyTargetClass()) {
                Class<?> targetClass = this.getTargetClass();
                if (targetClass == null) {
                    throw new FactoryBeanNotInitializedException("Cannot determine target class for proxy");
                }

                this.setInterfaces(ClassUtils.getAllInterfacesForClass(targetClass, this.proxyClassLoader));
            }

            super.setFrozen(this.freezeProxy);
            this.singletonInstance = this.getProxy(this.createAopProxy());//关键方法
        }

        return this.singletonInstance;
    }
```
getSingletonInstance()里主要是**this.singletonInstance = this.getProxy(this.createAopProxy())**这一行，createAopProxy()是创建一个AopProxy对象，getProxy()是获取一个AopProxy对象。
**createAopProxy()是在ProxyFactoryBean的父类ProxyCreatorSupport里实现的：**
``` java
protected final synchronized AopProxy createAopProxy() {
        if (!this.active) {
            this.activate();
        }

        return this.getAopProxyFactory().createAopProxy(this);
    }
```
这里有两个方法，一个是getAopProxyFactory()获取**DefaultAopProxyFactory**；一个是调用createAopProxy()从工厂里获取具体的AopProxy对象，这里会根据代理方式生成不同的代理对象。
来到DefaultAopProxyFactory看一下对createAopProxy()方法的实现：
``` java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
        if (!config.isOptimize() && !config.isProxyTargetClass() && !this.hasNoUserSuppliedProxyInterfaces(config)) {
            return new JdkDynamicAopProxy(config);//jdk动态代理
        } else {
            Class<?> targetClass = config.getTargetClass();//取得代理的目标对象
            if (targetClass == null) {
                throw new AopConfigException("TargetSource cannot determine target class: Either an interface or a target is required for proxy creation.");
            } else {
                return (AopProxy)(!targetClass.isInterface() && !Proxy.isProxyClass(targetClass) ? new ObjenesisCglibAopProxy(config) : new JdkDynamicAopProxy(config));//根据代理目标是接口还是类创建不同的代理对象
            }
        }
    }
```
这里主要是根据代理目标是接口还是类分别创建**JdkDynamicAopProxy和ObjenesisCglibAopProxy代理对象，ObjenesisCglibAopProxy主要是在父类CglibAopProxy中创建代理对象。**
那么，根据不同的代理类型创建完代理对象后，接下来回到getSingletonInstance()方法中的**getProxy()方法，还是在ProxyFactoryBean类中**：
``` java
protected Object getProxy(AopProxy aopProxy) {
		return aopProxy.getProxy(this.proxyClassLoader);
	}
```
真正实现在AopProxy具体代理类中的getProxy()，传入的是代理类加载器，这里就将任务交给JdkDynamicAopProxy和ObjenesisCglibAopProxy来实现获取代理类：
``` java
//JdkDynamicAopProxy中生成代理类对象
public Object getProxy(@Nullable ClassLoader classLoader) {
		if (logger.isDebugEnabled()) {
			logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
		}
		Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
		findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
		return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);//关键方法
	}
    //CglibAopProxy中生成代理对象
    public Object getProxy(@Nullable ClassLoader classLoader) {
		if (logger.isDebugEnabled()) {
			logger.debug("Creating CGLIB proxy: target source is " + this.advised.getTargetSource());
		}

		try {
			Class<?> rootClass = this.advised.getTargetClass();//代理目标类
			Assert.state(rootClass != null, "Target class must be available for creating a CGLIB proxy");

			Class<?> proxySuperClass = rootClass;
            //代理的目标是类
			if (ClassUtils.isCglibProxyClass(rootClass)) {
				proxySuperClass = rootClass.getSuperclass();//获取目标类父类
				Class<?>[] additionalInterfaces = rootClass.getInterfaces();//获取目标类实现的接口
				for (Class<?> additionalInterface : additionalInterfaces) {
					this.advised.addInterface(additionalInterface);//添加接口到通知器内
				}
			}

			// 验证
			validateClassIfNecessary(proxySuperClass, classLoader);

			// 下面就是创建Enhancer，CGLIB代理的核心类，和平时使用的时候差不多
			Enhancer enhancer = createEnhancer();
			if (classLoader != null) {
				enhancer.setClassLoader(classLoader);//设置目标类加载器
				if (classLoader instanceof SmartClassLoader &&
						((SmartClassLoader) classLoader).isClassReloadable(proxySuperClass)) {
					enhancer.setUseCache(false);
				}
			}
			enhancer.setSuperclass(proxySuperClass);
			enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
			enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
			enhancer.setStrategy(new ClassLoaderAwareUndeclaredThrowableStrategy(classLoader));
			//获取回调，回调也就是通知
			Callback[] callbacks = getCallbacks(rootClass);
			Class<?>[] types = new Class<?>[callbacks.length];
			for (int x = 0; x < types.length; x++) {
				types[x] = callbacks[x].getClass();
			}
			// 回调过滤器
			enhancer.setCallbackFilter(new ProxyCallbackFilter(
					this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
			enhancer.setCallbackTypes(types);

			// 创建代理类实例，里面还是调用的enhancer.create()方法
			return createProxyClassAndInstance(enhancer, callbacks);
		}
		catch (CodeGenerationException | IllegalArgumentException ex) {
			throw new AopConfigException("Could not generate CGLIB subclass of class [" +
					this.advised.getTargetClass() + "]: " +
					"Common causes of this problem include using a final class or a non-visible class",
					ex);
		}
		catch (Throwable ex) {
			throw new AopConfigException("Unexpected AOP exception", ex);
		}
	}
```
上面的两个代码段分别是JdkDynamicAopProxy和CglibAopProxy中创建代理对象的地方，和我们平时使用这两种代理差不多。在CglibAopProxy的getProxy()中有个getCallbacks()，这个方法返回的是Callback数组，也就是封装了我们的通知，**每个Callback其实是一个DynamicAdvisedInterceptor对象**，在CGLIB中是通过DynamicAdvisedInterceptor的intercept()方法回调目标对象的被代理的方法，而在JDK代理中，使用的是InvocationHandler的invoke()方法实现回调的。

## Aop拦截器调用
上面的工作完成了对AopProxy对象的获取，入口是在ProxyFactoryBean的getObject()方法中，最终是根据Jdk动态代理和CGLIB动态代理两种方式获取AopProxy的实例对象。那么，这个AopProxy实例对象中现在封装了一系列的拦截器(通知)和对被代理的方法的回调，具体是怎么调用实现的就是接下来要讨论的问题。
#### JdkDynamicAopProxy的invoke
在JdkDynamicAopProxy的getProxy()方法最后使用**Proxy.newProxyInstance(classLoader, proxiedInterfaces, this)**来创建Proxy实例(类似平时使用JDK代理一样)，需要传入目标实现类的类加载器、代理的目标接口和HandlerInvocation实例对象(因为JdkDynamicAopProxy实现了该接口，所以传入this)，而拦截的回调入口是HandlerInvocation的invoke()方法。
``` java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		MethodInvocation invocation;
		Object oldProxy = null;
		boolean setProxyContext = false;

		TargetSource targetSource = this.advised.targetSource;
		Object target = null;

		try {
			if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
				// The target does not implement the equals(Object) method itself.
				return equals(args[0]);
			}
			else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
				// The target does not implement the hashCode() method itself.
				return hashCode();
			}
			else if (method.getDeclaringClass() == DecoratingProxy.class) {
				// There is only getDecoratedClass() declared -> dispatch to proxy config.
				return AopProxyUtils.ultimateTargetClass(this.advised);
			}
			else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
					method.getDeclaringClass().isAssignableFrom(Advised.class)) {
				// Service invocations on ProxyConfig with the proxy config...
				return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
			}

			Object retVal;

			if (this.advised.exposeProxy) {
				// Make invocation available if necessary.
				oldProxy = AopContext.setCurrentProxy(proxy);
				setProxyContext = true;
			}

			// 获取代理的目标对象
			target = targetSource.getTarget();
			Class<?> targetClass = (target != null ? target.getClass() : null);

			//获取方法的拦截器链
			List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

			//如果拦截器链是空的，直接调用目标方法
			if (chain.isEmpty()) {
				Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
				retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
			}
			else {
				// 创建ReflectiveMethodInvocation对Aop功能进行封装
				invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
				retVal = invocation.proceed();
			}

			// Massage return value if necessary.
			Class<?> returnType = method.getReturnType();
			if (retVal != null && retVal == target &&
					returnType != Object.class && returnType.isInstance(proxy) &&
					!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
				retVal = proxy;
			}
			else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
				throw new AopInvocationException(
						"Null return value from advice does not match primitive return type for: " + method);
			}
			return retVal;
		}
		finally {
			if (target != null && !targetSource.isStatic()) {
				targetSource.releaseTarget(target);
			}
			if (setProxyContext) {
				AopContext.setCurrentProxy(oldProxy);
			}
		}
	}
```
上述代码关键是创建了**ReflectiveMethodInvocation对象**，该对象封装了AOP的实现。而proceed()方法会逐个运行拦截器的拦截方法：
``` java
public Object proceed() throws Throwable {
		if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
			return invokeJoinpoint();
		}

		//获取拦截器
		Object interceptorOrInterceptionAdvice =
				this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
		if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
			//动态匹配切点
			InterceptorAndDynamicMethodMatcher dm =
					(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
			if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
				return dm.interceptor.invoke(this);
			}
			else {
				// 动态匹配失败，跳过当前拦截器
				return proceed();
			}
		}
		else {
			// 若是interceptor，直接调用其对应的方法
			return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
		}
	}
```
上面是对拦截器的处理过程，interceptorsAndDynamicMethodMatchers里面有存放interceptorOrInterceptionAdvice的一个列表，每次调用proceed()会对列表中的下一个拦截器进行处理。而这个列表是在invoke()回调方法中获取的，具体是`List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);`这行代码获取，**getInterceptorsAndDynamicInterceptionAdvice()是在AdvisedSupport中实现的。**
``` java
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, @Nullable Class<?> targetClass) {
		MethodCacheKey cacheKey = new MethodCacheKey(method);//List保存的是MethodCacheKey对象
		List<Object> cached = this.methodCache.get(cacheKey);
		if (cached == null) {
			cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
					this, method, targetClass);//缓存中没有拦截器，自己生成
			this.methodCache.put(cacheKey, cached);
		}
		return cached;
	}
    //下面是在DefaultAdvisorChainFactory中生成
    public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
			Advised config, Method method, @Nullable Class<?> targetClass) {

		List<Object> interceptorList = new ArrayList<>(config.getAdvisors().length);
		Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
		boolean hasIntroductions = hasMatchingIntroductions(config, actualClass);
		AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();//获取注册器
		//遍历每个配置的通知器
		for (Advisor advisor : config.getAdvisors()) {
			if (advisor instanceof PointcutAdvisor) {
				// Add it conditionally.
				PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
				if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
					MethodInterceptor[] interceptors = registry.getInterceptors(advisor);//通过注册器加入拦截器链
					MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
					if (MethodMatchers.matches(mm, method, actualClass, hasIntroductions)) {
						if (mm.isRuntime()) {
							// 添加新的通知
							for (MethodInterceptor interceptor : interceptors) {
								interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
							}
						}
						else {
							interceptorList.addAll(Arrays.asList(interceptors));
						}
					}
				}
			}
			else if (advisor instanceof IntroductionAdvisor) {
				IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
				if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
					Interceptor[] interceptors = registry.getInterceptors(advisor);
					interceptorList.addAll(Arrays.asList(interceptors));
				}
			}
			else {
				Interceptor[] interceptors = registry.getInterceptors(advisor);
				interceptorList.addAll(Arrays.asList(interceptors));
			}
		}

		return interceptorList;
	}
```
上面首先是创建了一个List用来保存拦截器，然后创建了一个**AdvisorAdapterRegistry**用来注册拦截器，这个主要是用来将通知织入。接下来是利用AdvisorAdapterRegistry对每个通知进行适配，得到拦截器，加入到List中。

#### CglibAopProxy的intercept方法
在CglibAopProxy的getProxy()方法中我们介绍了，这里的Callback是DynamicAdvisedInterceptor对象，对AOP的拦截调用的回调是在DynamicAdvisedInterceptor(CglibAopProxy的内部类)中的intercept()方法实现的：
``` java
public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
			Object oldProxy = null;
			boolean setProxyContext = false;
			Object target = null;
			TargetSource targetSource = this.advised.getTargetSource();
			try {
				if (this.advised.exposeProxy) {
					oldProxy = AopContext.setCurrentProxy(proxy);
					setProxyContext = true;
				}
				// 获取代理目标类
				target = targetSource.getTarget();
				Class<?> targetClass = (target != null ? target.getClass() : null);
				List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
				Object retVal;
				// 拦截器链为空，直接调用目标方法
				if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
					Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
					retVal = methodProxy.invoke(target, argsToUse);
				}
				else {
					// 创建CglibMethodInvocation对象，封装Aop实现
					retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
				}
				retVal = processReturnType(proxy, target, method, retVal);
				return retVal;
			}
			finally {
				if (target != null && !targetSource.isStatic()) {
					targetSource.releaseTarget(target);
				}
				if (setProxyContext) {
					AopContext.setCurrentProxy(oldProxy);
				}
			}
		}
```
和JdkDynamicAopProxy的invoke类似，这里也是获取拦截器链，**使用CglibMethodInvocation对Aop进行封装**，CglibMethodInvocation是ReflectiveMethodInvocation的子类，这里对拦截器链的调用proceed()方法和Jdk代理一样。

## Advice通知的实现
上面讲述了AopProxy对象的生成和从通知配置中获取拦截器链，以及调用目标方法时拦截器的拦截调用和目标对象方法调用的实现，接下来要分析的是通知是如何对目标对象进行增强的。
最开始的时候，我们看到Advice通知分为前置通知、后置通知、异常通知等，对于配置的通知，我们需要根据他们的类型进行分类，在DefaultAdvisorAdapterRegister里默认会注册这些通知类型，然后在getInterceptors()里将具体的通知转换为对应类型的拦截器：
``` java
public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
		List<MethodInterceptor> interceptors = new ArrayList<>(3);
		Advice advice = advisor.getAdvice();//获得通知器的通知
		if (advice instanceof MethodInterceptor) {//是方法拦截器
			interceptors.add((MethodInterceptor) advice);
		}
        //遍历每个默认的适配器，和当前通知进行匹配
		for (AdvisorAdapter adapter : this.adapters) {
			if (adapter.supportsAdvice(advice)) {
				interceptors.add(adapter.getInterceptor(advisor));
			}
		}
		if (interceptors.isEmpty()) {
			throw new UnknownAdviceTypeException(advisor.getAdvice());
		}
		return interceptors.toArray(new MethodInterceptor[0]);
	}
```
转换拦截器的过程，就是获取通知，判断通知是哪一类的通知，用对应的通知进行封装，在Spring中是设计不同的拦截器来封装不同类型的通知的，具体对应的类是MethodBeforeAdviceAdapter、AfterReturningAdviceInterceptor和ThrowsAdviceInterceptor等。

