---
    layout: post
    title: spring_aop的一些实现
---

## Spring aop的实现
- 动态代理工具比较成熟的产品有：JDK自带的，ASM，CGLIB(基于ASM包装)，JAVAASSIST， 
  
-  > 对于这个过程，一般分为动态织入和静态织入，动态织入的方式是在运行时动态将要增强的代码织入到目标类中，这样往往是通过动态代理技术完成的，如Java JDK的动态代理(Proxy，底层通过反射实现)或者CGLIB的动态代理(底层通过继承实现)，Spring AOP采用的就是基于运行时增强的代理技术，这点后面会分析，这里主要重点分析一下静态织入，ApectJ采用的就是静态织入的方式。ApectJ主要采用的是编译期织入，在这个期间使用AspectJ的acj编译器(类似javac)把aspect类编译成class字节码后，在java目标类编译时织入，即先编译aspect类再编译目标类。

- Spring 只是使用了与 AspectJ 5 一样的注解，但仍然没有使用 AspectJ 的编译器，底层依是动态代理技术的实现，因此并不依赖于 AspectJ 的编译器

- spring AOP的实现原理是基于动态织入的动态代理技术:Java JDK动态代理和CGLIB动态代理，前者是基于反射技术的实现，后者是基于继承的机制实现  

    * jdk动态代理:dubbo中JdkProxyFactory#getProxy  

    ```java  
    @Overridev
    @SuppressWarnings("unchecked")
    public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
        return (T) Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), interfaces, new InvokerInvocationHandler(invoker));
    }
    ```

    * cglib：spring中CglibSubclassingInstantiationStrategy#createEnhancedSubclass    

    ```java  
     /**
		 * Create an enhanced subclass of the bean class for the provided bean
		 * definition, using CGLIB.
		 */
		private Class<?> createEnhancedSubclass(RootBeanDefinition beanDefinition) {
			Enhancer enhancer = new Enhancer();
			enhancer.setSuperclass(beanDefinition.getBeanClass());
			enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
			if (this.owner instanceof ConfigurableBeanFactory) {
				ClassLoader cl = ((ConfigurableBeanFactory) this.owner).getBeanClassLoader();
				enhancer.setStrategy(new ClassLoaderAwareGeneratorStrategy(cl));
			}
			enhancer.setCallbackFilter(new MethodOverrideCallbackFilter(beanDefinition));
			enhancer.setCallbackTypes(CALLBACK_TYPES);
			return enhancer.createClass();
		}
    ```

## spring bean 实例化
- 默认策略是懒加载(lazy-init),beanFactory.getBean()时触发doCreateBean(),通过组装好的BeanDefinition 实例化出bean对象。 要是不是懒加载，容器refresh后就是进行了，见👇

- AbstractApplicationContext#refresh->finishBeanFactoryInitialization()->   DefaultListableBeanFactory#preInstantiateSingletons-> AbstractBeanFactory#doGetBean-> AbstractAutowireCapableBeanFactory#createBean  

- instance的策略默认是cglib的   
  ```java   
    public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory {

		/** Strategy for creating bean instances */
		private InstantiationStrategy instantiationStrategy = new CglibSubclassingInstantiationStrategy();
		}  
  ```

- CglibSubclassingInstantiationStrategy,createEnhancedSubclass有cglib代理生成proxy class  

  ```java  

  public class CglibSubclassingInstantiationStrategy extends SimpleInstantiationStrategy;

  SimpleInstantiationStrategy
	@Override
	public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
		// Don't override the class with CGLIB if no overrides.
		if (bd.getMethodOverrides().isEmpty()) {
			Constructor<?> constructorToUse;
			synchronized (bd.constructorArgumentLock) {
				constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
				if (constructorToUse == null) {
					final Class<?> clazz = bd.getBeanClass();
					if (clazz.isInterface()) {
						throw new BeanInstantiationException(clazz, "Specified class is an interface");
					}
					try {
						if (System.getSecurityManager() != null) {
							constructorToUse = AccessController.doPrivileged(
									(PrivilegedExceptionAction<Constructor<?>>) () ->
											clazz.getDeclaredConstructor());
						}
						else {
							constructorToUse =	clazz.getDeclaredConstructor();
						}
						bd.resolvedConstructorOrFactoryMethod = constructorToUse;
					}
					catch (Throwable ex) {
						throw new BeanInstantiationException(clazz, "No default constructor found", ex);
					}
				}
			}
			return BeanUtils.instantiateClass(constructorToUse);
		}
		else {
			// Must generate CGLIB subclass.
			return instantiateWithMethodInjection(bd, beanName, owner);
		}
	}  
  ```


## spring aop的bean织入
   - AbstractAutowireCapableBeanFactory#initializeBean ->#applyBeanPostProcessorsAfterInitialization()

   - AbstractAdvisorAutoProxyCreator extends AbstractAutoProxyCreator#postProcessAfterInitialization()

   - AbstractAutoProxyCreator#wrapIfNecessary->#createProxy()

   


## spring aop自调用问题
   - spring容器中管理的是经过aop织入的代理对象，注入使用时调用对象方法，aop自然生效
   - bean内部方法调用，this.some(),this自然不会是代理对象(能不能注入this的代理对象或者直接直接从容器中取？这样应该就能解决这个自调用了)
   - 一些方法：
		* AopContext.currentProxy()
		* @PostConstruct  ApplicationContext.getBean()
		* BeanPostProcessor set Proxy




## spring bean生命周期
* BeanPostProcessor#postProcessBeforeInitialization(),#postProcessAfterInitialization() difference?
	* spring bean lifecycle 
		- BeanDefinition 
		- BeanFactoryPostProcessor.postProcessBeanFactory()
		- instantiate
		- populate properties
		- BeanPostProcessor.postProcessBeforeInitialization() 
		- @PostConstruct(JSR-250)
		- InitializingBean.afterPropertiesSet()
		- BeanPostProcessor.postProcessAfterInitialization