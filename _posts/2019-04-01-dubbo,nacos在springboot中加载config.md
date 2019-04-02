---
    layout: post
    title: dubbo,nacos在springboot中加载配置
---


## spring-boot starter
- 一个完整的Spring Boot starter可能包含以下组件：
    * autoconfigure模块，包含自动配置类的代码。
    * starter模块，提供自动配置模块及其他有用的依赖，简而言之，添加本starter就能开始使用该library。


## dubbo
- dubbo-spring-boot-autoconfigure.jar  
    ```java
    /**
    * The lowest precedence {@link EnvironmentPostProcessor} processes
    * {@link SpringApplication#setDefaultProperties(Properties) Spring Boot default properties} for Dubbo
    * as late as possible before {@link ConfigurableApplicationContext#refresh() application context refresh}.
    */
    public class DubboDefaultPropertiesEnvironmentPostProcessor implements EnvironmentPostProcessor, Ordered {


        @Override
        public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
            MutablePropertySources propertySources = environment.getPropertySources();
            Map<String, Object> defaultProperties = createDefaultProperties(environment);
            if (!CollectionUtils.isEmpty(defaultProperties)) {
                addOrReplace(propertySources, defaultProperties);
            }
        }
    }
    ```

## nacos-config
- nacos-spring-context:NacosConfigurationPropertiesBindingPostProcessor

    ```java
    public class NacosConfigurationPropertiesBindingPostProcessor implements BeanPostProcessor, ApplicationContextAware {

        

        @Override
        public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {

            NacosConfigurationProperties nacosConfigurationProperties = findAnnotation(bean.getClass(), NacosConfigurationProperties.class);

            if (nacosConfigurationProperties != null) {
                bind(bean, beanName, nacosConfigurationProperties);
            }

            return bean;
        }

        private void bind(Object bean, String beanName, NacosConfigurationProperties nacosConfigurationProperties) {

            NacosConfigurationPropertiesBinder binder = new NacosConfigurationPropertiesBinder(applicationContext);

            binder.bind(bean, beanName, nacosConfigurationProperties);

        }
    }

    ```

- BeanPostProcessor：在bean initialization之前执行该回调
    ```java

    /**
    * Factory hook that allows for custom modification of new bean instances,
    * e.g. checking for marker interfaces or wrapping them with proxies.
    */
    public interface BeanPostProcessor {

        /**
        * Apply this BeanPostProcessor to the given new bean instance <i>before</i> any bean
        * initialization callbacks (like InitializingBean's {@code afterPropertiesSet}
        * or a custom init-method). The bean will already be populated with property values.
        * The returned bean instance may be a wrapper around the original.
        * <p>The default implementation returns the given {@code bean} as-is.
        * @param bean the new bean instance
        * @param beanName the name of the bean
        * @return the bean instance to use, either the original or a wrapped one;
        * if {@code null}, no subsequent BeanPostProcessors will be invoked
        * @throws org.springframework.beans.BeansException in case of errors
        * @see org.springframework.beans.factory.InitializingBean#afterPropertiesSet
        */
        @Nullable
        default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
            return bean;
        }
    }
    ```
- nacos config: tomcatServletWebServerFactory 这个bean init引起的
    * 执行链路：
        ```java
        "main@1" prio=5 tid=0x1 nid=NA runnable
        java.lang.Thread.State: RUNNABLE
            at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:576)
            at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:498)
            at org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:320)
            at org.springframework.beans.factory.support.AbstractBeanFactory$$Lambda$132.992086987.getObject(Unknown Source:-1)
            at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:222)
            at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:318)
            at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:204)
            at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.getWebServerFactory(ServletWebServerApplicationContext.java:216)
            at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.createWebServer(ServletWebServerApplicationContext.java:180)
            at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.onRefresh(ServletWebServerApplicationContext.java:154)
            at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:540)
            - locked <0x16fd> (a java.lang.Object)
            at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.refresh(ServletWebServerApplicationContext.java:142)
            at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:775)
            at org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:397)

            at org.springframework.boot.SpringApplication.run(SpringApplication.java:1248)

        ```

    * 执行处：AbstractAutowireCapableBeanFactory.applyBeanPostProcessorsBeforeInitialization()
        ```java

        @Override
            public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
                    throws BeansException {

                Object result = existingBean;
                for (BeanPostProcessor processor : getBeanPostProcessors()) {
                    Object current = processor.postProcessBeforeInitialization(result, beanName);
                    if (current == null) {
                        return result;
                    }
                    result = current;
                }
                return result;
            }

        ```
v