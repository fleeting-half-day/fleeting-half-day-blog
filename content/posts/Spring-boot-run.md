---
title: "SpringBoot 源码解析 （一）：启动流程"
date: 2024-07-01
draft: true
metaAlignment: center
categories:
- SpringBoot
tags:
- SpringBoot 源码
- SpringBoot 启动流程
---

<!--more-->

### SpringApplication 初始化
* 设置主类属性
* 设置应用类型
* 设置初始化器
* 设置监听器
### SpringApplication.run() 执行
* 获取并启动监听器
  * 把监听器放到广播器中
  * 发布启动事件
* 构建环境
  * 发布准备事件，ConfigFileApplicationListener 加载项目配置
* 创建容器
  * AnnotationConfigServletWebApplicationContext
* 容器前置处理
  * 容器初始化类执行
  * 发布事件：容器已准备好
  * 主类加载到容器
  * 发布事件：容器加载完成
* 刷新容器
  * 参考 [Spring 源码解析 （一）：容器刷新](/2024/05/spring-源码解析-一容器刷新/)
* 容器后置处理
  * 空实现，留给子类扩展
* 发布 ApplicationStartedEvent 事件
* 执行 Runners
  * 执行 ApplicationRunner
  * 执行 CommandLineRunner

new SpringApplication().run(args);
```java
@SpringBootApplication
public class HelloWorldMainApplication {
    public static void main(String[] args) {
        SpringApplication.run(MainApplication.class, args);
    }
}
public static ConfigurableApplicationContext run(Class<?> primarySource,String... args) {
    return run(new Class<?>[] { primarySource }, args);
}
public static ConfigurableApplicationContext run(Object[] sources, String[] args) {
    return (new SpringApplication(sources)).run(args);
}

public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    // 把 MainApplication.class 设置为属性存储起来
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    // 设置应用类型是Standard还是Web
    this.webApplicationType = deduceWebApplicationType();
    // 设置初始化器(Initializer)，最后会调用这些初始化器
    setInitializers((Collection) getSpringFactoriesInstances( ApplicationContextInitializer.class));
    // 设置监听器(Listener)
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    this.mainApplicationClass = deduceMainApplicationClass();
}

/**
 * 根据能加载到的类判断应用类型
 * 相关常量
 * REACTIVE_WEB_ENVIRONMENT_CLASS = "org.springframework.web.reactive.DispatcherHandler";
 * MVC_WEB_ENVIRONMENT_CLASS = "org.springframework.web.servlet.DispatcherServlet";
 * WEB_ENVIRONMENT_CLASSES = {  "javax.servlet.Servlet", 
 *                              "org.springframework.web.context.ConfigurableWebApplicationContext" };
 * @return
 */
private WebApplicationType deduceWebApplicationType() {
    // 
    if (ClassUtils.isPresent(REACTIVE_WEB_ENVIRONMENT_CLASS, null)
            && !ClassUtils.isPresent(MVC_WEB_ENVIRONMENT_CLASS, null)) {
        return WebApplicationType.REACTIVE;
    }
    // 非 web 应用
    for (String className : WEB_ENVIRONMENT_CLASSES) {
        if (!ClassUtils.isPresent(className, null)) {
            return WebApplicationType.NONE;
        }
    }
    // web 应用
    return WebApplicationType.SERVLET;
}
```

### run 方法源码

```java
public ConfigurableApplicationContext run(String... args) {
    long startTime = System.nanoTime();
    DefaultBootstrapContext bootstrapContext = createBootstrapContext();
    ConfigurableApplicationContext context = null;
    // 配置运行程序的系统环境，以确保可正确的运行
    configureHeadlessProperty();
    // 获取在 SpringApplication 上的所有监听器
    SpringApplicationRunListeners listeners = getRunListeners(args);
    // 通知所有监听器，启动应用程序
    listeners.starting(bootstrapContext, this.mainApplicationClass);
    try {
        // 封装应用程序的主程序参数
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        // 准备应用环境，生成环境变量
        ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);
        configureIgnoreBeanInfo(environment);
        // 打印 banner
        Banner printedBanner = printBanner(environment);
        // 创建容器
        context = createApplicationContext();
        context.setApplicationStartup(this.applicationStartup);
        // 容器前刷新置处理
        prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
        // 刷新容器，调用父类 AbstractApplicationContext 的 refresh 方法
        refreshContext(context);
        // 容器刷新后置处理
        afterRefresh(context, applicationArguments);
        Duration timeTakenToStartup = Duration.ofNanos(System.nanoTime() - startTime);
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), timeTakenToStartup);
        }
        // 通知所有监听器，应用程序已经启动
        listeners.started(context, timeTakenToStartup);
        // 运行所有已经注册的 runner
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, listeners);
        throw new IllegalStateException(ex);
    }
    try {
        Duration timeTakenToReady = Duration.ofNanos(System.nanoTime() - startTime);
        // 发布应用上下文就绪事件
        listeners.ready(context, timeTakenToReady);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, null);
        throw new IllegalStateException(ex);
    }
    return context;
}
```