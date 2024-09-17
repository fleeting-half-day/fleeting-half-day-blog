---
title: "Spring 源码解析 （一）：容器刷新"
date: 2024-05-08
draft: true
metaAlignment: center
categories:
- Spring
tags:
- Spring 源码
- Spring 容器 refresh
---

<!--more-->

```java
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // 准备Context的刷新
        prepareRefresh();

        // 初始化 BeanFactory
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // 配置 BeanFactory
        // 设置 ClassLoader、语言解析器（StandardBeanExpressionResolver解析EL表达式）、属性编辑注册器（ResourceEditorRegistrar）
        // 注册 2 个 bean 后置处理器：ApplicationContextAwareProcessor、ApplicationListenerDetector
        // 注册 3 个默认环境 bean：environment、systemProperties、systemEnvironment
        // 忽略自动装配的几个接口、设置自动装配的特殊规则
        prepareBeanFactory(beanFactory);

        try {
            // 模板方法，默认实现为空，提供给子类对 BeanFactory 进行扩展
            postProcessBeanFactory(beanFactory);

            // 实例化和调用所有 BeanFactoryPostProcessor，包括其子类 BeanDefinitionRegistryPostProcessor
            // BeanDefinitionRegistryPostProcessor 继承自 BeanFactoryPostProcessor，
            // 比 BeanFactoryPostProcessor 具有更高的优先级，
            // 主要用来在常规的 BeanFactoryPostProcessor 激活之前注册一些 bean 定义。
            // 可以通过 BeanDefinitionRegistryPostProcessor 来注册一些常规的 BeanFactoryPostProcessor，
            // 因为此时所有常规的 BeanFactoryPostProcessor 都还没开始被处理。
            invokeBeanFactoryPostProcessors(beanFactory);

            // 注册所有的 BeanPostProcessor，将所有实现了 BeanPostProcessor 接口的类加载到 BeanFactory 中。
            // BeanPostProcessor 接口是 Spring 初始化 bean 时对外暴露的扩展点，
            // Spring IoC 容器允许 BeanPostProcessor 在容器初始化 bean 的前后，添加自己的逻辑处理。
            // 在这边只是注册到 BeanFactory 中，具体调用是在 bean 初始化的时候。
            // 具体的：在所有 bean 实例化时，
            // 执行初始化方法前会调用所有 BeanPostProcessor 的 postProcessBeforeInitialization 方法，
            // 执行初始化方法后会调用所有 BeanPostProcessor 的 postProcessAfterInitialization 方法。
            registerBeanPostProcessors(beanFactory);

            // 初始化消息资源 MessageSource
            initMessageSource();

            // 初始化事件广播器 ApplicationEventMulticaster
            initApplicationEventMulticaster();

            // 模板方法，默认实现为空，提供给子类扩展（eg:SpringBoot）
            onRefresh();

            // 注册监听器
            registerListeners();

            // 实例化所有剩余的非懒加载单例 Bean，并调用 BeanPostProcessor
            // 除了一些内部的 Bean、实现了 BeanFactoryPostProcessor 接口的 Bean、实现了 BeanPostProcessor 接口的 bean，
            // 其他的非懒加载单例 bean 都会在这个方法中被实例化
            finishBeanFactoryInitialization(beanFactory);

            // 完成上下文的刷新，发布上下文刷新完毕事件（ContextRefreshedEvent ）到监听器
            finishRefresh();
        } catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                        "cancelling refresh attempt: " + ex);
            }
            destroyBeans();
            cancelRefresh(ex);
            throw ex;
        } finally {
            resetCommonCaches();
        }
    }
}
```