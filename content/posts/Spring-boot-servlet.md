---
title: "SpringBoot 源码解析 （三）：Servlet容器"
date: 2024-07-03
draft: true
metaAlignment: center
categories:
- SpringBoot
tags:
- SpringBoot 源码
- 嵌入式容器
---

<!--more-->

SpringBoot 嵌入式 Servlet 容器的加载是通过自动配置类 ServletWebServerFactoryAutoConfiguration 完成的，看过上篇 [SpringBoot 源码解析 （二）：自动配置](/2024/07/springboot-源码解析-二自动配置/) ，我们知道 SpringBoot 是从 META-INF/spring.factories （或者较新版本 META-INF/spring/%s.imports ）中加载自动配置类。

### ServletWebServerFactoryConfiguration 源码
```java
@Configuration(proxyBeanMethods = false)
class ServletWebServerFactoryConfiguration {

    @Configuration(proxyBeanMethods = false)
    // Tomcat、Servlet 和 UpgradeProtocol 类必须在 classloader 中
    @ConditionalOnClass({ Servlet.class, Tomcat.class, UpgradeProtocol.class })
    // 当前 Spring 容器中不存在 ServletWebServerFactory 的实例
    @ConditionalOnMissingBean(value = ServletWebServerFactory.class, search = SearchStrategy.CURRENT)
    static class EmbeddedTomcat {
        // 上述注解条件都成立，构造 TomcatServletWebServerFactory
        @Bean
        TomcatServletWebServerFactory tomcatServletWebServerFactory(ObjectProvider<TomcatConnectorCustomizer> connectorCustomizers, ObjectProvider<TomcatContextCustomizer> contextCustomizers, ObjectProvider<TomcatProtocolHandlerCustomizer<?>> protocolHandlerCustomizers) {
            TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
            // ...
            return factory;
        }
    }

    @Configuration(proxyBeanMethods = false)
    @ConditionalOnClass({ Servlet.class, Server.class, Loader.class, WebAppContext.class })
    @ConditionalOnMissingBean(value = ServletWebServerFactory.class, search = SearchStrategy.CURRENT)
    static class EmbeddedJetty {
        @Bean
        JettyServletWebServerFactory jettyServletWebServerFactory(
                ObjectProvider<JettyServerCustomizer> serverCustomizers) {
            JettyServletWebServerFactory factory = new JettyServletWebServerFactory();
            factory.getServerCustomizers().addAll(serverCustomizers.orderedStream().collect(Collectors.toList()));
            return factory;
        }
    }

    @Configuration(proxyBeanMethods = false)
    @ConditionalOnClass({ Servlet.class, Undertow.class, SslClientAuthMode.class })
    @ConditionalOnMissingBean(value = ServletWebServerFactory.class, search = SearchStrategy.CURRENT)
    static class EmbeddedUndertow {
        @Bean
        UndertowServletWebServerFactory undertowServletWebServerFactory(ObjectProvider<UndertowDeploymentInfoCustomizer> deploymentInfoCustomizers, ObjectProvider<UndertowBuilderCustomizer> builderCustomizers) {
            UndertowServletWebServerFactory factory = new UndertowServletWebServerFactory();
            // ...
            return factory;
        }
        @Bean
        UndertowServletWebServerFactoryCustomizer undertowServletWebServerFactoryCustomizer(ServerProperties serverProperties) {
            return new UndertowServletWebServerFactoryCustomizer(serverProperties);
        }
    }
}
```
默认条件下会加载 **TomcatServletWebServerFactory**。

回顾 [SpringBoot 源码解析（一）：启动流程](/2024/07/springboot-源码解析-一-----启动流程/) 中 **创建容器** 和 **刷新容器** 步骤：

> **创建容器**：会根据 WebApplicationType 创建 Spring 容器，如果时 web 应用，则创建 AnnotationConfigServletWebServerApplicationContext 容器。
> 
> **刷新容器**：refreshContext(context) 实则调用抽象父类 AbstractApplicationContext 的 refresh 方法刷新容器，该方法会调用一个 onRefresh 模版方法（参考 [Spring 源码解析（一）：容器刷新](/2024/05/spring-源码解析-一-----容器刷新/) ），供子类扩展，这里的子类既是 AnnotationConfigServletWebServerApplicationContext，但 AnnotationConfigServletWebServerApplicationContext 并没有实现 onRefresh 方法，而是由父类 ServletWebServerApplicationContext 实现，源码如下：
```java
/**
 * ServletWebServerApplicationContext#onRefresh
 */
protected void onRefresh() {
    super.onRefresh();
    try {
        createWebServer();
    } catch (Throwable ex) {
        throw new ApplicationContextException("Unable to start web server", ex);
    }
}

/**
 * 创建 WebServer
 */
private void createWebServer() {
    WebServer webServer = this.webServer;
    ServletContext servletContext = getServletContext();
    if (webServer == null && servletContext == null) {
        StartupStep createWebServer = this.getApplicationStartup().start("spring.boot.webserver.create");
        // 获取 WebServer 工厂，上面知道加载了 TomcatServletWebServerFactory
        ServletWebServerFactory factory = getWebServerFactory();
        createWebServer.tag("factory", factory.getClass().toString());
        // 获取 WebServer
        this.webServer = factory.getWebServer(getSelfInitializer());
        createWebServer.end();
        getBeanFactory().registerSingleton("webServerGracefulShutdown",
                new WebServerGracefulShutdownLifecycle(this.webServer));
        getBeanFactory().registerSingleton("webServerStartStop",
                new WebServerStartStopLifecycle(this, this.webServer));
    }
    else if (servletContext != null) {
        try {
            getSelfInitializer().onStartup(servletContext);
        }
        catch (ServletException ex) {
            throw new ApplicationContextException("Cannot initialize servlet context", ex);
        }
    }
    initPropertySources();
}

/**
 * 获取 WebServer，TomcatServletWebServerFactory#getWebServer
 * @param initializers
 * @return
 */
public WebServer getWebServer(ServletContextInitializer... initializers) {
    if (this.disableMBeanRegistry) {
        Registry.disableRegistry();
    }
    // 创建 Tomcat
    Tomcat tomcat = new Tomcat();
    File baseDir = (this.baseDirectory != null) ? this.baseDirectory : createTempDir("tomcat");
    tomcat.setBaseDir(baseDir.getAbsolutePath());
    for (LifecycleListener listener : this.serverLifecycleListeners) {
        tomcat.getServer().addLifecycleListener(listener);
    }
    Connector connector = new Connector(this.protocol);
    connector.setThrowOnFailure(true);
    tomcat.getService().addConnector(connector);
    customizeConnector(connector);
    tomcat.setConnector(connector);
    tomcat.getHost().setAutoDeploy(false);
    configureEngine(tomcat.getEngine());
    for (Connector additionalConnector : this.additionalTomcatConnectors) {
        tomcat.getService().addConnector(additionalConnector);
    }
    prepareContext(tomcat.getHost(), initializers);
    // 获取 TomcatWebServer
    return getTomcatWebServer(tomcat);
}

/**
 * TomcatServletWebServerFactory#getTomcatWebServer
 * @param tomcat
 * @return
 */
protected TomcatWebServer getTomcatWebServer(Tomcat tomcat) {
    return new TomcatWebServer(tomcat, getPort() >= 0, getShutdown());
}

public TomcatWebServer(Tomcat tomcat, boolean autoStart, Shutdown shutdown) {
    Assert.notNull(tomcat, "Tomcat Server must not be null");
    this.tomcat = tomcat;
    this.autoStart = autoStart;
    this.gracefulShutdown = (shutdown == Shutdown.GRACEFUL) ? new GracefulShutdown(tomcat) : null;
    // 初始化
    initialize();
}

/**
 * 初始化，TomcatWebServer#initialize
 * @throws WebServerException
 */
private void initialize() throws WebServerException {
    logger.info("Tomcat initialized with port(s): " + getPortsDescription(false));
    synchronized (this.monitor) {
        try {
            // 这里只保留我们关注的代码，启动 tomcat
            this.tomcat.start();
        } catch (Exception ex) {}
    }
}
```
