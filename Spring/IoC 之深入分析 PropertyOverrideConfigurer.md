# IoC 之深入分析 PropertyOverrideConfigurer

﻿在文章 [《【死磕 Spring】—— IoC 之深入分析 BeanFactoryPostProcessor》](http://svip.iocoder.cn/Spring/IoC-BeanFactoryPostProcessor) 中提到，BeanFactoryPostProcessor 作用与 BeanDefinition 完成加载之后与 Bean 实例化之前，是 Spring 提供的一种强大的扩展机制。它有两个重要的子类，一个是 PropertyPlaceholderConfigurer，另一个是 PropertyOverrideConfigurer ，其中 PropertyPlaceholderConfigurer 允许我们通过配置 Properties 的方式来取代 Bean 中定义的占位符，而 **PropertyOverrideConfigurer** 呢？正是我们这篇博客介绍的。

```properties
PropertyOverrideConfigurer 允许我们对 Spring 容器中配置的任何我们想处理的 bean 定义的 property 信息进行覆盖替换。
```

这个定义听起来有点儿玄乎，通俗点说，就是我们可以通过 PropertyOverrideConfigurer 来覆盖任何 bean 中的任何属性，只要我们想。

# 1. 使用