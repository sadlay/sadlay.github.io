---
layout: post
title: Spring中Bean的生命周期流程
categories: Spring
description: 使用Spring框架，我们需要了解Bean的创建加载过程，需要熟悉Bean是如何获取和使用的。
keywords: Spring, Bean, Lifecycle 
---

# Spring中的Bean的生命周期流程

## 一、生命周期

Bean的完整生命周期图如下

![fbe424a3-c67a-356d-bfec-be8c030ec0a6](https://tva2.sinaimg.cn/large/007DFXDhly1g5y0trlim3j30g90dxgn6.jpg)

### 1.1 容器启动时

BeanFactoryPostProcessor->**postProcessBeanFactory**() ;

### 1.2 实例化之后调用

 InstantiationAwareBeanPostProcessor ->**postProcessPropertyValues()**

### 1.3 Bean初始化时

1. 属性注入（setter）
2. BeanNameAware ->**setBeanName**() 
3. BeanFactoryAware->**setBeanFactory**() 
4. ApplicationContextAware->**setApplicationContext**()
5. BeanPostProcessor ->**postProcessBeforeInitialization()**
6. InitializingBean->**afterPropertiesSet**()
7. **init-method**属性
8. BeanPostProcessor->**postProcessAfterInitialization()**
9. DiposibleBean->**destory**() 
10. **destroy-method**属性

## 二、分类

对于BeanFactoryAware和BeanNameAware接口，第一个接口让bean感知容器（即BeanFactory实例，从而以此获取该容器配置的其他bean对象），而后者让bean获得配置文件中对应的配置名称。在一般情况下用户不需要关心这两个接口。如果bean希望获得容器中的其他bean，可以通过属性注入的方式引用这些bean。如果bean希望在运行期获知在配置文件中的Bean名称，可以简单的将名称作为属性注入

综上所述，我们认为除非编写一个基于spring之上的扩展框架插件或者子项目之类的东西，否则用户完全可以抛开以上4个bean生命周期的接口类

但BeanPostProcessor接口却不一样，它不要求bean去继承它，它可以完全像插件一样注册到spring容器中，为容器提供额外的功能。spring充分利用了BeanPostProcessor对bean进行加工处理（SpringAOP以此为基础)

Bean的完整生命周期经历了各种方法调用，这些方法可以划分为以下几类： 

### 2.1 Bean自身的方法

　　这个包括了Bean本身调用的方法和通过配置文件中<bean>的init-method和destroy-method指定的方法

### 2.2 Bean级生命周期接口方法

　　这个包括了BeanNameAware、BeanFactoryAware、InitializingBean和DiposableBean这些接口的方法

### 2.3 容器级生命周期接口方法

　　这个包括了InstantiationAwareBeanPostProcessor 和 BeanPostProcessor 这两个接口实现，一般称它们的实现类为“后处理器”。

### 2.4 工厂后处理器接口方法

　　这个包括了AspectJWeavingEnabler, ConfigurationClassPostProcessor, CustomAutowireConfigurer等等非常有用的工厂后处理器　　接口的方法。工厂后处理器也是容器级的。在应用上下文装配配置文件之后立即调用。

## 三、示例

我们用一个简单的Spring Bean来演示一下Spring Bean的生命周期。 

### 3.1 定义一个Bean

首先是一个简单的Spring Bean，调用Bean自身的方法和Bean级生命周期接口方法，

为了方便演示，它**实现了BeanNameAware、BeanFactoryAware、InitializingBean和DiposableBean这4个接口，**

同时有2个方法，对应配置文件中<bean>的init-method和destroy-method。如下

```java
package com.github.sadlay.spring.core.model;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.*;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;

/**
 * Knight骑士的model
 *
 * @Author: lay
 * @Date: Created in 9:42 2019/8/13
 * @Modified By:IntelliJ IDEA
 */

public class Knight  implements BeanNameAware, BeanFactoryAware, ApplicationContextAware, InitializingBean, DisposableBean {
    private String name;
    private String sex;
    private String level;

    public Knight(String name, String sex) {
        System.out.println("【构造器】调用Knight的构造器实例化");
        this.name = name;
        this.sex = sex;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        System.out.println("【注入属性】注入属性name");
        this.name = name;
    }

    public String getSex() {
        return sex;
    }

    public void setSex(String sex) {
        System.out.println("【注入属性】注入属性sex");
        this.sex = sex;
    }

    public String getLevel() {
        return level;
    }

    public void setLevel(String level) {
        this.level = level;
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        System.out.println("【BeanFactoryAware接口】调用BeanFactoryAware.setBeanFactory()");
    }

    @Override
    public void setBeanName(String s) {
        System.out.println("【BeanNameAware接口】调用BeanNameAware.setBeanName()");
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("【DiposibleBean接口】调用DiposibleBean.destory()");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("【InitializingBean接口】调用InitializingBean.afterPropertiesSet()");
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        System.out.println("【ApplicationContextAware】ApplicationContextAware.setApplicationContext()");
    }

    public void initMethod(){
        System.out.println("【init-method】调用<bean>的init-method属性指定的初始化方法");
    }

    public void destroyMethod(){
        System.out.println("【destroy-method】调用<bean>的destroy-method属性指定的初始化方法");
    }

    @Override
    public String toString() {
        return "Knight{" +
                "name='" + name + '\'' +
                ", sex='" + sex + '\'' +
                ", level='" + level + '\'' +
                '}';
    }
}

```

### 3.2 BeanPostProcessor接口

```java
package com.github.sadlay.spring.core.cycle;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanPostProcessor;

/**
 * BeanPostProcessor
 *
 * BeanPostProcessor接口包括2个方法postProcessAfterInitialization和postProcessBeforeInitialization，这两个方法的第一个参数都是要处理的Bean对象，第二个参数都是Bean的name。返回值也都是要处理的Bean对象。这里要注意。
 *
 * @Author: lay
 * @Date: Created in 10:43 2019/8/13
 * @Modified By:IntelliJ IDEA
 */
public class MyBeanPostProcessor implements BeanPostProcessor {

    public MyBeanPostProcessor() {
        super();
        System.out.println("这是【BeanPostProcessor】实现类构造器！！");
    }

    @Override
    public Object postProcessAfterInitialization(Object arg0, String arg1) throws BeansException {
        System.out.println("【BeanPostProcessor】接口方法postProcessAfterInitialization对属性进行更改！");
        return arg0;
    }

    @Override
    public Object postProcessBeforeInitialization(Object arg0, String arg1) throws BeansException {
        System.out.println("【BeanPostProcessor】接口方法postProcessBeforeInitialization对属性进行更改！");
        return arg0;
    }
}

```

 如上，BeanPostProcessor接口包括2个方法postProcessAfterInitialization和postProcessBeforeInitialization，这两个方法的第一个参数都是要处理的Bean对象，第二个参数都是Bean的name。返回值也都是要处理的Bean对象。这里要注意。

### 3.3 InstantiationAwareBeanPostProcessor

InstantiationAwareBeanPostProcessor 接口本质是BeanPostProcessor的子接口，一般我们继承Spring为其提供的适配器类InstantiationAwareBeanPostProcessorAdapter来使用它，如下：

````java
package springBeanTest;  
   
 import java.beans.PropertyDescriptor;    
 import org.springframework.beans.BeansException;  
 import org.springframework.beans.PropertyValues;  
 import org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessorAdapter;  
   
 public class MyInstantiationAwareBeanPostProcessor extends  
        InstantiationAwareBeanPostProcessorAdapter {  
  
    public MyInstantiationAwareBeanPostProcessor() {  
        super();  
        System.out.println("这是InstantiationAwareBeanPostProcessorAdapter实现类构造器！！");  
    }  
  
    // 接口方法、实例化Bean之前调用  
    @Override  
    public Object postProcessBeforeInstantiation(Class beanClass,  
            String beanName) throws BeansException {  
       
        System.out.println("InstantiationAwareBeanPostProcessor调用postProcessBeforeInstantiation方法");  
        return null;  
    }  
  
    // 接口方法、实例化Bean之后调用  
    @Override  
    public Object postProcessAfterInitialization(Object bean, String beanName)  
            throws BeansException {  
        System.out.println("InstantiationAwareBeanPostProcessor调用postProcessAfterInitialization方法");  
        return bean;  
    }  
  
    // 接口方法、设置某个属性时调用  
    @Override  
    public PropertyValues postProcessPropertyValues(PropertyValues pvs,  
            PropertyDescriptor[] pds, Object bean, String beanName)  
            throws BeansException {  
        System.out.println("InstantiationAwareBeanPostProcessor调用postProcessPropertyValues方法");  
        return pvs;  
    }  
}  
````

这个有3个方法，其中第二个方法postProcessAfterInitialization就是重写了BeanPostProcessor的方法。**第三个方法postProcessPropertyValues用来操作属性**，返回值也应该是PropertyValues对象。

### 3.4 BeanFactoryPostProcessor 

演示工厂后处理器接口方法，如下：

```java
package springBeanTest;  
   
 import org.springframework.beans.BeansException;  
 import org.springframework.beans.factory.config.BeanDefinition;  
 import org.springframework.beans.factory.config.BeanFactoryPostProcessor;  
 import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;  
   
 public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {  
   
    public MyBeanFactoryPostProcessor() {  
        super();  
        System.out.println("这是BeanFactoryPostProcessor实现类构造器！！");  
    }  
  
    @Override  
    public void postProcessBeanFactory(ConfigurableListableBeanFactory arg)  
            throws BeansException {  
        System.out  
                .println("BeanFactoryPostProcessor调用postProcessBeanFactory方法");  
        BeanDefinition bd = arg.getBeanDefinition("person");  
        bd.getPropertyValues().addPropertyValue("phone", "");  
    }   
}  
```

### 3.5 配置文件

使用javaConfig的形式注册bean

```java
package com.github.sadlay.spring.core.config;

import com.github.sadlay.spring.core.cycle.MyBeanFactoryPostProcessor;
import com.github.sadlay.spring.core.cycle.MyBeanPostProcessor;
import com.github.sadlay.spring.core.cycle.MyInstantiationAwareBeanPostProcessor;
import com.github.sadlay.spring.core.model.Knight;
import org.springframework.beans.factory.config.BeanFactoryPostProcessor;
import org.springframework.beans.factory.config.BeanPostProcessor;
import org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessorAdapter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * Bean配置
 *
 * @Author: lay
 * @Date: Created in 9:42 2019/8/13
 * @Modified By:IntelliJ IDEA
 */
@Configuration
public class BeanConfig {

    @Bean(initMethod = "initMethod", destroyMethod = "destroyMethod")
    public Knight knight() {
        return new Knight("lay", "男");
    }

    @Bean
    public BeanPostProcessor myBeanPostProcessor() {
        return new MyBeanPostProcessor();
    }

    @Bean
    public InstantiationAwareBeanPostProcessorAdapter myInstantiationAwareBeanPostProcessor() {
        return new MyInstantiationAwareBeanPostProcessor();
    }

    @Bean
    public BeanFactoryPostProcessor myBeanFactoryPostProcessor() {
        return new MyBeanFactoryPostProcessor();
    }
}

```

### 3.6 测试

```jva
package com.github.sadlay.spring.core;

import com.github.sadlay.spring.core.model.Knight;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

/**
 * 主启动类
 *
 * @Author: lay
 * @Date: Created in 9:38 2019/8/13
 * @Modified By:IntelliJ IDEA
 */
@Configuration
@ComponentScan("com.github.sadlay")
public class SpringCoreApplication {

    public static void main(String[] args) {
        System.out.println("现在开始初始化容器");
        ApplicationContext context = new AnnotationConfigApplicationContext(SpringCoreApplication.class);
        System.out.println("容器初始化成功");
        Knight bean = context.getBean(Knight.class);
        System.out.println(bean.toString());
        System.out.println("现在开始关闭容器！");
        ((AnnotationConfigApplicationContext) context).close();
    }
}

```







