---
title: "SpringBoot 源码解析 （二）：自动配置"
date: 2024-07-02
draft: true
metaAlignment: center
categories:
- SpringBoot
tags:
- SpringBoot 源码
- 自动配置
---

<!--more-->

### SpringBoot 启动类加载

在上篇 [SpringBoot 源码解析 （一）：启动流程](/2024/07/springboot-源码解析-一启动流程/) 中已经介绍，在 prepareContext 时会调用 SpringApplication#load 方法加载 SpringBoot 启动类，该方法最终会调用 BeanDefinitionLoader#load 方法将 SpringBoot 启动类注入到 spring 容器 beanDefinitionMap 中。

### @SpringBootApplication
```java
// 当前类是配置类
@SpringBootConfiguration
// 开启自动配置
@EnableAutoConfiguration
// 配置扫描包
@ComponentScan
public @interface SpringBootApplication {}

  @Configuration  // 当前类是配置类
  public @interface SpringBootConfiguration {}
    @Component  // 表示当前类是 SpringBean
    public @interface Configuration {}


  @AutoConfigurationPackage   // 自动配置包
  @Import(AutoConfigurationImportSelector.class)  // 通过 import 导入满足条件的 bean，并加载到 spring 的 ioc 容器里面
  public @interface EnableAutoConfiguration {}

    @Import(AutoConfigurationPackages.Registrar.class)  // 把 Registrar 导入到 spring 容器里面，Registrar 的作用是扫描包，默认是把主类所在的包和子包里面全部类扫描进容器里面
    public @interface AutoConfigurationPackage {}
```
#### AutoConfigurationImportSelector 调用过程：

* AutoConfigurationImportSelector#selectImports
  * AutoConfigurationImportSelector#getAutoConfigurationEntry
    * AutoConfigurationImportSelector#getCandidateConfigurations
      * 调用 SpringFactoriesLoader#loadFactoryNames 方法，从 META-INF/spring.factories 中加载所有默认的自动配置类；
      * **（新版本）** 调用 ImportCandidates#load 方法，根据条件加载 META-INF/spring/%s.imports 中的配置类；
```java
/**
 * 版本：2.2.x
 * @param metadata
 * @param attributes
 * @return
 */
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
  List<String> configurations = SpringFactoriesLoader.loadFactoryNames(this.getSpringFactoriesLoaderFactoryClass(), this.getBeanClassLoader());
  Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you are using a custom packaging, make sure that file is correct.");
  return configurations;
}
/**
 * 版本：2.7.x
 * @param metadata
 * @param attributes
 * @return
 */
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    List<String> configurations = new ArrayList<>(
              // 加载 META-INF/spring.factories 中的配置类
              SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader()));
    // 根据条件加载 META-INF/spring/%s.imports 中的配置类
    ImportCandidates.load(AutoConfiguration.class, getBeanClassLoader()).forEach(configurations::add);
    Assert.notEmpty(configurations,
            "No auto configuration classes found in META-INF/spring.factories nor in META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports. If you "
                    + "are using a custom packaging, make sure that file is correct.");
    return configurations;
}
```
* @ComponentScan 包扫描
