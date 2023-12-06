---
title: "Spring Bean 生命周期"
date: 2023-07-18
metaAlignment: center
categories:
- Spring
tags:
- Bean生命周期
---

<!--more-->

这里从源码中摘取 Bean 创建的步骤，仅保留关键的方法调用。

Spring 项目启动的时候会调用 AbstractApplicationContext#refresh 方法，刷新上下文。refresh() 方法会调用当前类的 finishBeanFactoryInitialization() 方法，这里会触发 Bean 的创建和初始化。跟源码会发现 Spring 最终是调用 AbstractAutowireCapableBeanFactory#doCreateBean 方法创建并初始化 Bean。

AbstractAutowireCapableBeanFactory#doCreateBean
![doCreateBean](/images/spring/doCreateBean.png)

doCreateBean() 方法会调用当前类的 initializeBean() 方法初始化 Bean。

AbstractAutowireCapableBeanFactory#initializeBean
![initializeBean](/images/spring/initializeBean.png)

这里只列出 Bean 创建过程中的重要步骤，给出了注释。其中还有一些重要步骤仅在注释中指出了触发的时机和实现的类，有兴趣的可以再深入跟一下源码。