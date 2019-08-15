---
layout: post
title: Spring中Bean的生命周期流程（源码分析）
categories: Spring
description: 使用Spring框架，我们需要了解Bean的创建加载过程，需要熟悉Bean是如何获取和使用的。本片由BeanPostProcessor进行切入点，对Srring源码进行分析。
keywords: Spring, Bean, Lifecycle, Source-Code-Analysis
---

# Spring的Bean的生命周期（源码分析）

## 一、 BeanPostProcessor

接口定义

```java
package org.springframework.beans.factory.config;  
public interface BeanPostProcessor {  
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;  
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;  
}  
```

BeanPostProcessor是Spring容器的一个扩展点，可以进行自定义的实例化、初始化、依赖装配、依赖检查等流程，即可以覆盖默认的实例化，也可以增强初始化、依赖注入、依赖检查等流程，其javadoc有如下描述：

​    e.g. checking for marker interfaces or wrapping them with proxies.

​    大体意思是可以检查相应的标识接口完成一些自定义功能实现，如包装目标对象到代理对象。

我们可以看到BeanPostProcessor一共有两个回调方法postProcessBeforeInitialization和postProcessAfterInitialization，那这两个方法会在什么Spring执行流程中的哪个步骤执行呢？还有目前Spring提供哪些相应的实现呢？

Spring还提供了BeanPostProcessor一些其他接口实现，来完成除实例化外的其他功能，后续详细介绍。

## 二、 通过源代码看看创建一个Bean实例的具体执行流程

 ![img](http://dl.iteye.com/upload/attachment/0066/8083/a7988c9e-c90f-32f6-8d17-ec5e206f48fb.jpg)

AbstractApplicationContext内部使用DefaultListableBeanFactory，且DefaultListableBeanFactory继承AbstractAutowireCapableBeanFactory，因此我们此处分析AbstractAutowireCapableBeanFactory即可。

### 2.1 createBean

AbstractAutowireCapableBeanFactory的createBean方法代码如下：

```java
protected Object createBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) throws BeanCreationException {  
    resolveBeanClass(mbd, beanName); /1解析Bean的class  
    mbd.prepareMethodOverrides(); //2 方法注入准备  
    Object bean = resolveBeforeInstantiation(beanName, mbd); //3 第一个BeanPostProcessor扩展点  
    if (bean != null) { //4 如果3处的扩展点返回的bean不为空，直接返回该bean，后续流程不需要执行  
        return bean;  
    }   
    Object beanInstance = doCreateBean(beanName, mbd, args); //5 执行spring的创建bean实例的流程啦  
    return beanInstance;  
}  
```

### 2.2 resolveBeforeInstantiation

AbstractAutowireCapableBeanFactory的resolveBeforeInstantiation方法代码如下：

```java
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {  
        Object bean = null;  
        if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {  
            // Make sure bean class is actually resolved at this point.  
            if (mbd.hasBeanClass() && !mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {  
                //3.1、执行InstantiationAwareBeanPostProcessor的postProcessBeforeInstantiation回调方法  
                bean = applyBeanPostProcessorsBeforeInstantiation(mbd.getBeanClass(), beanName);  
                if (bean != null) {  
                    //3.2、执行InstantiationAwareBeanPostProcessor的postProcessAfterInitialization回调方法  
                    bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);  
                }  
            }  
            mbd.beforeInstantiationResolved = (bean != null);  
        }  
        return bean;  
} 
```

### 2.3 doCreateBean

AbstractAutowireCapableBeanFactory的doCreateBean方法代码如下：

```java
        // 6、通过BeanWrapper实例化Bean   
        BeanWrapper instanceWrapper = null;  
        if (mbd.isSingleton()) {  
            instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);  
        }  
        if (instanceWrapper == null) {  
            instanceWrapper = createBeanInstance(beanName, mbd, args);  
        }  
        final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);  
        Class beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);  
  
        //7、执行MergedBeanDefinitionPostProcessor的postProcessMergedBeanDefinition流程  
        synchronized (mbd.postProcessingLock) {  
            if (!mbd.postProcessed) {  
                applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);  
                mbd.postProcessed = true;  
            }  
        }  
        // 8、及早暴露单例Bean引用，从而允许setter注入方式的循环引用  
        boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&  
                isSingletonCurrentlyInCreation(beanName));  
        if (earlySingletonExposure) {  
            //省略log  
            addSingletonFactory(beanName, new ObjectFactory() {  
                public Object getObject() throws BeansException {  
                    //8.1、调用SmartInstantiationAwareBeanPostProcessor的getEarlyBeanReference返回一个需要暴露的Bean（例如包装目标对象到代理对象）  
                    return getEarlyBeanReference(beanName, mbd, bean);  
                }  
            });  
        }  
          
        Object exposedObject = bean;  
        try {  
            populateBean(beanName, mbd, instanceWrapper); //9、组装-Bean依赖  
            if (exposedObject != null) {  
                exposedObject = initializeBean(beanName, exposedObject, mbd); //10、初始化Bean  
            }  
        }  
        catch (Throwable ex) {  
            //省略异常  
        }  
  
  
        //11如果是及早暴露单例bean，通过getSingleton触发3.1处的getEarlyBeanReference调用获取要及早暴露的单例Bean  
        if (earlySingletonExposure) {  
            Object earlySingletonReference = getSingleton(beanName, false);  
            if (earlySingletonReference != null) {  
                if (exposedObject == bean) {  
                    exposedObject = earlySingletonReference;  
                }  
                else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {  
                    String[] dependentBeans = getDependentBeans(beanName);  
                    Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);  
                    for (String dependentBean : dependentBeans) {  
                        if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {  
                            actualDependentBeans.add(dependentBean);  
                        }  
                    }  
                    if (!actualDependentBeans.isEmpty()) {  
                        throw new BeanCurrentlyInCreationException(beanName,  
                                "Bean with name '" + beanName + "' has been injected into other beans [" +  
                                StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +  
                                "] in its raw version as part of a circular reference, but has eventually been " +  
                                "wrapped. This means that said other beans do not use the final version of the " +  
                                "bean. This is often the result of over-eager type matching - consider using " +  
                                "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");  
                    }  
                }  
            }  
        }  
        //12、注册Bean的销毁回调  
        try {  
            registerDisposableBeanIfNecessary(beanName, bean, mbd);  
        }  
        catch (BeanDefinitionValidationException ex) {  
            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);  
        }  
  
        return exposedObject;  
}  
```

### 2.4 populateBean

AbstractAutowireCapableBeanFactory的populateBean方法代码如下：

```java
//9、组装-Bean  
protected void populateBean(String beanName, AbstractBeanDefinition mbd, BeanWrapper bw) {  
    PropertyValues pvs = mbd.getPropertyValues();  
    //省略部分代码  
    //9.1、通过InstantiationAwareBeanPostProcessor扩展点允许自定义装配流程（如@Autowired支持等）  
    //执行InstantiationAwareBeanPostProcessor的postProcessAfterInstantiation  
    boolean continueWithPropertyPopulation = true;  
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {  
        for (BeanPostProcessor bp : getBeanPostProcessors()) {  
            if (bp instanceof InstantiationAwareBeanPostProcessor) {  
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;  
                if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {  
                    continueWithPropertyPopulation = false;  
                    break;  
                }  
            }  
        }  
    }  
    if (!continueWithPropertyPopulation) {  
        return;  
    }  
    if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||  
            mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {  
        MutablePropertyValues newPvs = new MutablePropertyValues(pvs);  
        // 9. 2、自动装配（根据name/type）  
        if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {  
            autowireByName(beanName, mbd, bw, newPvs);  
        }  
        if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {  
            autowireByType(beanName, mbd, bw, newPvs);  
        }  
        pvs = newPvs;  
    }  
    boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();  
    boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);  
  
  
    //9. 3、执行InstantiationAwareBeanPostProcessor的postProcessPropertyValues  
    if (hasInstAwareBpps || needsDepCheck) {  
        PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw);  
        if (hasInstAwareBpps) {  
            for (BeanPostProcessor bp : getBeanPostProcessors()) {  
                if (bp instanceof InstantiationAwareBeanPostProcessor) {  
                    InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;  
                    pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);  
                    if (pvs == null) {  
                        return;  
                    }  
                }  
            }  
        }  
        //9. 4、执行依赖检查  
        if (needsDepCheck) {  
            checkDependencies(beanName, mbd, filteredPds, pvs);  
        }  
    }  
    //9. 5、应用依赖注入  
    applyPropertyValues(beanName, mbd, bw, pvs);  
}  
```

### 2.5 initializeBean

AbstractAutowireCapableBeanFactory的initializeBean方法代码如下：

```java
     //10、实例化Bean  
     protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {  
//10.1、调用Aware接口注入（BeanNameAware、BeanClassLoaderAware、BeanFactoryAware）  
invokeAwareMethods(beanName, bean);//此处省略部分代码  
//10.2、执行BeanPostProcessor扩展点的postProcessBeforeInitialization进行修改实例化Bean  
Object wrappedBean = bean;  
if (mbd == null || !mbd.isSynthetic()) {  
    wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);  
}  
//10.3、执行初始化回调（1、调用InitializingBean的afterPropertiesSet  2、调用自定义的init-method）  
try {  
    invokeInitMethods(beanName, wrappedBean, mbd);  
}  
catch (Throwable ex) {  
    //异常省略  
}  
//10.4、执行BeanPostProcessor扩展点的postProcessAfterInitialization进行修改实例化Bean  
if (mbd == null || !mbd.isSynthetic()) {  
    wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);  
}  
return wrappedBean;  
```

## 三、创建一个Bean实例的执行流程简化：

protected Object createBean(final String beanName, final RootBeanDefinition mbd, final Object[] args); 创建Bean

**（1****、resolveBeanClass(mbd, beanName);** 解析Bean class，若class配置错误将抛出CannotLoadBeanClassException；

 

**（2****、mbd.prepareMethodOverrides();** 准备和验证配置的方法注入，若验证失败抛出BeanDefinitionValidationException

有关方法注入知识请参考[【第三章】 DI 之 3.3 更多DI的知识 ——跟我学spring3](http://jinnianshilongnian.iteye.com/blog/1415461) 3.3.5 方法注入；

 

**（3****、Object bean = resolveBeforeInstantiation(beanName, mbd);** 第一个BeanPostProcessor扩展点，此处只执行InstantiationAwareBeanPostProcessor类型的BeanPostProcessor Bean；

（3.1、bean = applyBeanPostProcessorsBeforeInstantiation(mbd.getBeanClass(), beanName);执行InstantiationAwareBeanPostProcessor的实例化的预处理回调方法postProcessBeforeInstantiation（自定义的实例化，如创建代理）；

（3.2、bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);执行InstantiationAwareBeanPostProcessor的实例化的后处理回调方法postProcessAfterInitialization（如依赖注入），如果3.1处返回的Bean不为null才执行；

 

**（4****、如果3****处的扩展点返回的bean****不为空，直接返回该bean****，后续流程不需要执行；**

 

**（5****、Object beanInstance = doCreateBean(beanName, mbd, args);** 执行spring的创建bean实例的流程；



**（6****、createBeanInstance(beanName, mbd, args);** 实例化Bean

（6.1、instantiateUsingFactoryMethod 工厂方法实例化；请参考【http://jinnianshilongnian.iteye.com/blog/1413857】

（6.2、构造器实例化，请参考【http://jinnianshilongnian.iteye.com/blog/1413857】；

（6.2.1、如果之前已经解析过构造器

（6.2.1.1 autowireConstructor：有参调用autowireConstructor实例化

（6.2.1.2、instantiateBean：无参调用instantiateBean实例化；

（6.2.2、如果之前没有解析过构造器：

 

（6.2.2.1、通过SmartInstantiationAwareBeanPostProcessor的determineCandidateConstructors回调方法解析构造器，第二个BeanPostProcessor扩展点，返回第一个解析成功（返回值不为null）的构造器组，如AutowiredAnnotationBeanPostProcessor实现将自动扫描通过@Autowired/@Value注解的构造器从而可以完成构造器注入，请参考[【第十二章】零配置 之 12.2 注解实现Bean依赖注入 ——跟我学spring3](http://jinnianshilongnian.iteye.com/blog/1457224) ；

（6.2.2.2、autowireConstructor：如果（6.2.2.1返回的不为null，且是有参构造器，调用autowireConstructor实例化；

（6.2.2.3、instantiateBean： 否则调用无参构造器实例化；

 

**（7****、applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);**第三个BeanPostProcessor扩展点，执行Bean定义的合并；

（7.1、执行MergedBeanDefinitionPostProcessor的postProcessMergedBeanDefinition回调方法，进行bean定义的合并；

 

**（8****、addSingletonFactory(beanName, new ObjectFactory() {**

​                            **public Object getObject() throws BeansException {**

​                                   **return getEarlyBeanReference(beanName, mbd, bean);**

​                            **}**

​                     **});**  及早暴露单例Bean引用，从而允许setter注入方式的循环引用

（8.1、SmartInstantiationAwareBeanPostProcessor的getEarlyBeanReference；第四个BeanPostProcessor扩展点，当存在循环依赖时，通过该回调方法获取及早暴露的Bean实例；

 

**（9****、populateBean(beanName, mbd, instanceWrapper);**装配Bean依赖

（9.1、InstantiationAwareBeanPostProcessor的postProcessAfterInstantiation；第五个BeanPostProcessor扩展点，在实例化Bean之后，所有其他装配逻辑之前执行，如果false将阻止其他的InstantiationAwareBeanPostProcessor的postProcessAfterInstantiation的执行和从（9.2到（9.5的执行，通常返回true；

（9.2、autowireByName、autowireByType：根据名字和类型进行自动装配，自动装配的知识请参考[【第三章】 DI 之 3.3 更多DI的知识 ——跟我学spring3](http://jinnianshilongnian.iteye.com/blog/1415461)  3.3.3  自动装配；

（9.3、InstantiationAwareBeanPostProcessor的postProcessPropertyValues：第六个BeanPostProcessor扩展点，完成其他定制的一些依赖注入，如AutowiredAnnotationBeanPostProcessor执行@Autowired注解注入，CommonAnnotationBeanPostProcessor执行@Resource等注解的注入，PersistenceAnnotationBeanPostProcessor执行@ PersistenceContext等JPA注解的注入，RequiredAnnotationBeanPostProcessor执行@ Required注解的检查等等，请参考[【第十二章】零配置 之 12.2 注解实现Bean依赖注入 ——跟我学spring3](http://jinnianshilongnian.iteye.com/blog/1457224)；

（9.4、checkDependencies：依赖检查，请参考[【第三章】 DI 之 3.3 更多DI的知识 ——跟我学spring3](http://jinnianshilongnian.iteye.com/blog/1415461)  3.3.4  依赖检查；

（9.5、applyPropertyValues：应用明确的setter属性注入，请参考[【第三章】 DI 之 3.1 DI的配置使用 ——跟我学spring3 ](http://jinnianshilongnian.iteye.com/blog/1415277)；

 

**（10****、exposedObject = initializeBean(beanName, exposedObject, mbd);** 执行初始化Bean流程；

（10.1、invokeAwareMethods（BeanNameAware、BeanClassLoaderAware、BeanFactoryAware）：调用一些Aware标识接口注入如BeanName、BeanFactory；

（10.2、BeanPostProcessor的postProcessBeforeInitialization：第七个扩展点，在调用初始化之前完成一些定制的初始化任务，如BeanValidationPostProcessor完成JSR-303 @Valid注解Bean验证，InitDestroyAnnotationBeanPostProcessor完成@PostConstruct注解的初始化方法调用，ApplicationContextAwareProcessor完成一些Aware接口的注入（如EnvironmentAware、ResourceLoaderAware、ApplicationContextAware），其返回值将替代原始的Bean对象；

（10.3、invokeInitMethods ： 调用初始化方法；

（10.3.1、InitializingBean的afterPropertiesSet ：调用InitializingBean的afterPropertiesSet回调方法；

（10.3.2、通过xml指定的自定义init-method ：调用通过xml配置的自定义init-method

（10.3.3、BeanPostProcessor的postProcessAfterInitialization ：第八个扩展点，AspectJAwareAdvisorAutoProxyCreator（完成xml风格的AOP配置(<aop:config>)的目标对象包装到AOP代理对象）、AnnotationAwareAspectJAutoProxyCreator（完成@Aspectj注解风格（<aop:aspectj-autoproxy> @Aspect）的AOP配置的目标对象包装到AOP代理对象），其返回值将替代原始的Bean对象；

 

（11、if (earlySingletonExposure) {

​                     Object earlySingletonReference = getSingleton(beanName, false);

​            ……

​      } ：如果是earlySingleExposure，调用getSingle方法获取Bean实例；

earlySingleExposure =(mbd.isSingleton() && this.allowCircularReferences && isSingletonCurrentlyInCreation(beanName))

只要单例Bean且允许循环引用（默认true）且当前单例Bean正在创建中

（11.1、如果是earlySingletonExposure调用getSingleton将触发【8】处ObjectFactory.getObject()的调用，通过【8.1】处的getEarlyBeanReference获取相关Bean（如包装目标对象的代理Bean）；（在循环引用Bean时可能引起[Spring事务处理时自我调用的解决方案及一些实现方式的风险](https://www.iteye.com/topic/1122740)）；

 

**（12****、registerDisposableBeanIfNecessary(beanName, bean, mbd)** **：** 注册Bean的销毁方法（只有非原型Bean可注册）；

（12.1、单例Bean的销毁流程

（12.1.1、DestructionAwareBeanPostProcessor的postProcessBeforeDestruction ： 第九个扩展点，如InitDestroyAnnotationBeanPostProcessor完成@PreDestroy注解的销毁方法注册和调用；

（12.1.2、DisposableBean的destroy：注册/调用DisposableBean的destroy销毁方法；

（12.1.3、通过xml指定的自定义destroy-method ： 注册/调用通过XML指定的destroy-method销毁方法；

（12.1.2、Scope的registerDestructionCallback：注册自定义的Scope的销毁回调方法，如RequestScope、SessionScope等；其流程和【12.1 单例Bean的销毁流程一样】，关于自定义Scope请参考[【第三章】 DI 之 3.4 Bean的作用域 ——跟我学spring3](http://jinnianshilongnian.iteye.com/blog/1415463)

 

**（13****、到此Bean****实例化、依赖注入、初始化完毕可以返回创建好的bean****了。**

 

 从上面的流程我们可以看到BeanPostProcessor一个使用了九个扩展点，其实还一个扩展点（SmartInstantiationAwareBeanPostProcessor的predictBeanType在下一篇介绍），接下来我们看看BeanPostProcessor这些扩展点都主要完成什么功能及常见的BeanPostProcessor。

## 四、BeanPostProcessor接口及回调方法图 

![img](http://dl.iteye.com/upload/attachment/0066/9379/e2e1a684-ebeb-3e3f-8cd5-21471a239d16.png)

从图中我们可以看出一共五个接口，共十个回调方法，即十个扩展点，但我们之前的文章只分析了其中八个，另外两个稍候也会解析一下是干什么的。

## 五、五个接口十个扩展点解析

**1****、InstantiationAwareBeanPostProcessor**：实例化Bean后置处理器（继承BeanPostProcessor）

postProcessBeforeInstantiation ：在实例化目标对象之前执行，可以自定义实例化逻辑，如返回一个代理对象等，（3.1处执行；如果此处返回的Bean不为null将中断后续Spring创建Bean的流程，且只执行postProcessAfterInitialization回调方法，如当AbstractAutoProxyCreator的实现者注册了TargetSourceCreator（创建自定义的TargetSource）将改变执行流程，不注册TargetSourceCreator我们默认使用的是SingletonTargetSource（即AOP代理直接保证目标对象），此处我们还可以使用如ThreadLocalTargetSource（线程绑定的Bean）、CommonsPoolTargetSource（实例池的Bean）等等，大家可以去spring官方文档了解TargetSource详情；

postProcessAfterInitialization ： Bean实例化完毕后执行的后处理操作，所有初始化逻辑、装配逻辑之前执行，如果返回false将阻止其他的InstantiationAwareBeanPostProcessor的postProcessAfterInstantiation的执行，（3.2和（9.1处执行；在此处可以执行一些初始化逻辑或依赖装配逻辑；

postProcessPropertyValues ：完成其他定制的一些依赖注入和依赖检查等，如AutowiredAnnotationBeanPostProcessor执行@Autowired注解注入，CommonAnnotationBeanPostProcessor执行@Resource等注解的注入，PersistenceAnnotationBeanPostProcessor执行@ PersistenceContext等JPA注解的注入，RequiredAnnotationBeanPostProcessor执行@ Required注解的检查等等，（9.3处执行；

 

**2****、MergedBeanDefinitionPostProcessor**：合并Bean定义后置处理器         （继承BeanPostProcessor）

postProcessMergedBeanDefinition：执行Bean定义的合并，在（7.1处执行，且在实例化完Bean之后执行；

 

**3****、SmartInstantiationAwareBeanPostProcessor**：智能实例化Bean后置处理器（继承InstantiationAwareBeanPostProcessor）

predictBeanType：预测Bean的类型，返回第一个预测成功的Class类型，如果不能预测返回null；当你调用BeanFactory.getType(name)时当通过Bean定义无法得到Bean类型信息时就调用该回调方法来决定类型信息；BeanFactory.isTypeMatch(name, targetType)用于检测给定名字的Bean是否匹配目标类型（如在依赖注入时需要使用）；

determineCandidateConstructors：检测Bean的构造器，可以检测出多个候选构造器，再有相应的策略决定使用哪一个，如AutowiredAnnotationBeanPostProcessor实现将自动扫描通过@Autowired/@Value注解的构造器从而可以完成构造器注入，请参考[【第十二章】零配置 之 12.2 注解实现Bean依赖注入 ——跟我学spring3](http://jinnianshilongnian.iteye.com/blog/1457224) ，（6.2.2.1处执行；

getEarlyBeanReference：当正在创建A时，A依赖B，此时通过（8将A作为ObjectFactory放入单例工厂中进行early expose，此处B需要引用A，但A正在创建，从单例工厂拿到ObjectFactory（其通过getEarlyBeanReference获取及早暴露Bean），从而允许循环依赖，此时AspectJAwareAdvisorAutoProxyCreator（完成xml风格的AOP配置(<aop:config>)将目标对象（A）包装到AOP代理对象）或AnnotationAwareAspectJAutoProxyCreator（完成@Aspectj注解风格（<aop:aspectj-autoproxy> @Aspect）将目标对象（A）包装到AOP代理对象），其返回值将替代原始的Bean对象，即此时通过early reference能得到正确的代理对象，（8.1处实施；如果此处执行了，（10.3.3处的AspectJAwareAdvisorAutoProxyCreator或AnnotationAwareAspectJAutoProxyCreator的postProcessAfterInitialization将不执行，即这两个回调方法是二选一的；

 

**4****、BeanPostProcessor**：Bean后置处理器

postProcessBeforeInitialization：实例化、依赖注入完毕，在调用显示的初始化之前完成一些定制的初始化任务，如BeanValidationPostProcessor完成JSR-303 @Valid注解Bean验证，InitDestroyAnnotationBeanPostProcessor完成@PostConstruct注解的初始化方法调用，ApplicationContextAwareProcessor完成一些Aware接口的注入（如EnvironmentAware、ResourceLoaderAware、ApplicationContextAware），其返回值将替代原始的Bean对象；（10.2处执行；

postProcessAfterInitialization：实例化、依赖注入、初始化完毕时执行，如AspectJAwareAdvisorAutoProxyCreator（完成xml风格的AOP配置(<aop:config>)的目标对象包装到AOP代理对象）、AnnotationAwareAspectJAutoProxyCreator（完成@Aspectj注解风格（<aop:aspectj-autoproxy> @Aspect）的AOP配置的目标对象包装到AOP代理对象），其返回值将替代原始的Bean对象；（10.3.3处执行；此处需要参考getEarlyBeanReference；

 

**5****、DestructionAwareBeanPostProcessor**：销毁Bean后置处理器（继承BeanPostProcessor）

postProcessBeforeDestruction：销毁后处理回调方法，该回调只能应用到单例Bean，如InitDestroyAnnotationBeanPostProcessor完成@PreDestroy注解的销毁方法调用；（12.1.1处执行。

 六、内置的一些BeanPostProcessor

![img](http://dl.iteye.com/upload/attachment/0066/9383/1590f6b2-4ba4-3d81-b103-6d20a9a04012.png)

此图只有内置的一部分。

 

### 1、ApplicationContextAwareProcessor

容器启动时会自动注册。注入那些实现ApplicationContextAware、MessageSourceAware、ResourceLoaderAware、EnvironmentAware、

EmbeddedValueResolverAware、ApplicationEventPublisherAware标识接口的Bean需要的相应实例，在postProcessBeforeInitialization回调方法中进行实施，即（10.2处实施。

 

### 2、CommonAnnotationBeanPostProcessor

CommonAnnotationBeanPostProcessor继承InitDestroyAnnotationBeanPostProcessor，当在配置文件有<context:annotation-config>或<context:component-scan>会自动注册。

 

提供对JSR-250规范注解的支持@javax.annotation.Resource、@javax.annotation.PostConstruct和@javax.annotation.PreDestroy等的支持。

 

2.1、通过@Resource注解进行依赖注入：

​    postProcessPropertyValues：通过此回调进行@Resource注解的依赖注入；（9.3处实施；

2.2、用于执行@PostConstruct 和@PreDestroy 注解的初始化和销毁方法的扩展点：

​    postProcessBeforeInitialization()将会调用bean的@PostConstruct方法；（10.2处实施；

​    postProcessBeforeDestruction()将会调用单例 Bean的@PreDestroy方法（此回调方法会在容器销毁时调用），（12.1.1处实施。

 

详见[【第十二章】零配置 之 12.2 注解实现Bean依赖注入 ——跟我学spring3](http://jinnianshilongnian.iteye.com/blog/1457224)，JSR-250注解部分。

####  

### 3、AutowiredAnnotationBeanPostProcessor

当在配置文件有<context:annotation-config>或<context:component-scan>会自动注册。

 

提供对JSR-330规范注解的支持和Spring自带注解的支持。

 

3.1、Spring自带注解的依赖注入支持，@Autowired和@Value：

​    determineCandidateConstructors ：决定候选构造器；详见【12.2中的构造器注入】；（6.2.2.1处实施；

postProcessPropertyValues ：进行依赖注入；详见【12.2中的字段注入和方法参数注入】；（9.3处实施；

3.2、对JSR-330规范注解的依赖注入支持，@Inject：

​    同2.1类似只是查找使用的注解不一样；

 

详见[【第十二章】零配置 之 12.2 注解实现Bean依赖注入 ——跟我学spring3](http://jinnianshilongnian.iteye.com/blog/1457224)，Spring自带依赖注入注解和 JSR-330注解部分。

####  

### 4、RequiredAnnotationBeanPostProcessor

当在配置文件有<context:annotation-config>或<context:component-scan>会自动注册。

 

4.1、提供对@ Required注解的方法进行依赖检查支持：

​    postProcessPropertyValues：如果检测到没有进行依赖注入时抛出BeanInitializationException异常；（9.3处实施；

 

详见[【第十二章】零配置 之 12.2 注解实现Bean依赖注入 ——跟我学spring3](http://jinnianshilongnian.iteye.com/blog/1457224)，**@Required****：依赖检查**。

####  

### 5、PersistenceAnnotationBeanPostProcessor

当在配置文件有<context:annotation-config>或<context:component-scan>会自动注册。

 

5.1、通过对JPA @ javax.persistence.PersistenceUnit和@ javax.persistence.PersistenceContext注解进行依赖注入的支持；

​    postProcessPropertyValues ： 根据@PersistenceUnit/@PersistenceContext进行EntityManagerFactory和EntityManager的支持；

####  

### 6、AbstractAutoProxyCreator

AspectJAwareAdvisorAutoProxyCreator和AnnotationAwareAspectJAutoProxyCreator都是继承AbstractAutoProxyCreator，AspectJAwareAdvisorAutoProxyCreator提供对（<aop:config>）声明式AOP的支持，AnnotationAwareAspectJAutoProxyCreator提供对（<aop:aspectj-autoproxy>）注解式（@AspectJ）AOP的支持，因此只需要分析AbstractAutoProxyCreator即可。

 

当使用<aop:config>配置时自动注册AspectJAwareAdvisorAutoProxyCreator，而使用<aop:aspectj-autoproxy>时会自动注册AnnotationAwareAspectJAutoProxyCreator。

6.1、predictBeanType：预测Bean的类型，如果目标对象被AOP代理对象包装，此处将返回AOP代理对象的类型；

```java
public Class<?> predictBeanType(Class<?> beanClass, String beanName) {  
        Object cacheKey = getCacheKey(beanClass, beanName);  
        return this.proxyTypes.get(cacheKey); //获取代理对象类型，可能返回null  
}  
```

6.2、postProcessBeforeInstantiation：

```java
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {  
    //1、得到一个缓存的唯一key（根据beanClass和beanName生成唯一key）  
    Object cacheKey = getCacheKey(beanClass, beanName);  
    //2、如果当前targetSourcedBeans（通过自定义TargetSourceCreator创建的TargetSource）不包含cacheKey  
    if (!this.targetSourcedBeans.contains(cacheKey)) {  
        //2.1、advisedBeans（已经被增强的Bean，即AOP代理对象）中包含当前cacheKey或nonAdvisedBeans（不应该被增强的Bean）中包含当前cacheKey 返回null，即走Spring默认流程  
        if (this.advisedBeans.contains(cacheKey) || this.nonAdvisedBeans.contains(cacheKey)) {  
            return null;  
        }  
        //2.2、如果是基础设施类（如Advisor、Advice、AopInfrastructureBean的实现）不进行处理  
        //2.2、shouldSkip 默认false，可以生成子类覆盖，如AspectJAwareAdvisorAutoProxyCreator覆盖    （if (((AbstractAspectJAdvice) advisor.getAdvice()).getAspectName().equals(beanName)) return true;  即如果是自己就跳过）  
        if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {  
            this.nonAdvisedBeans.add(cacheKey);//在不能增强的Bean列表缓存当前cacheKey  
            return null;  
        }  
    }  
  
    //3、开始创建AOP代理对象  
    //3.1、配置自定义的TargetSourceCreator进行TargetSource创建  
    TargetSource targetSource = getCustomTargetSource(beanClass, beanName);  
    if (targetSource != null) {  
        //3.2、如果targetSource不为null 添加到targetSourcedBeans缓存，并创建AOP代理对象  
        this.targetSourcedBeans.add(beanName);  
        // specificInterceptors即增强（包括前置增强、后置增强等等）  
        Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);  
        //3.3、创建代理对象  
        Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);  
        //3.4、将代理类型放入proxyTypes从而允许后续的predictBeanType()调用获取  
        this.proxyTypes.put(cacheKey, proxy.getClass());  
        return proxy;  
    }  
    return null;  
}  
```

从如上代码可以看出，当我们配置TargetSourceCreator进行自定义TargetSource创建时，会创建代理对象并中断默认Spring创建流程。

6.3、getEarlyBeanReference

```java
//获取early Bean引用（只有单例Bean才能回调该方法）  
public Object getEarlyBeanReference(Object bean, String beanName) throws BeansException {  
    Object cacheKey = getCacheKey(bean.getClass(), beanName);  
    //1、将cacheKey添加到earlyProxyReferences缓存，从而避免多次重复创建  
    this.earlyProxyReferences.add(cacheKey);  
    //2、包装目标对象到AOP代理对象（如果需要）  
    return wrapIfNecessary(bean, beanName, cacheKey);  
}  
```

6.4、postProcessAfterInitialization

```java
public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {  
    if (bean != null) {  
        Object cacheKey = getCacheKey(bean.getClass(), beanName);  
        //1、如果之前调用过getEarlyBeanReference获取包装目标对象到AOP代理对象（如果需要），则不再执行  
        if (!this.earlyProxyReferences.contains(cacheKey)) {  
            //2、包装目标对象到AOP代理对象（如果需要）  
            return wrapIfNecessary(bean, beanName, cacheKey);  
        }  
    }  
    return bean;  
}  
```

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {  
    if (this.targetSourcedBeans.contains(beanName)) {//通过TargetSourceCreator进行自定义TargetSource不需要包装  
        return bean;  
    }  
    if (this.nonAdvisedBeans.contains(cacheKey)) {//不应该被增强对象不需要包装  
        return bean;  
    }  
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {//基础设施/应该skip的不需要保证  
        this.nonAdvisedBeans.add(cacheKey);  
        return bean;  
    }  
  
    // 如果有增强就执行包装目标对象到代理对象  
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);  
    if (specificInterceptors != DO_NOT_PROXY) {  
        this.advisedBeans.add(cacheKey);//将cacheKey添加到已经被增强列表，防止多次增强  
        Object proxy = createProxy(bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));//创建代理对象  
        this.proxyTypes.put(cacheKey, proxy.getClass());//缓存代理类型  
        return proxy;  
    }  
    this.nonAdvisedBeans.add(cacheKey);  
    return bean;  
}  
```

从如上流程可以看出 getEarlyBeanReference和postProcessAfterInitialization是二者选一的，而且单例Bean目标对象只能被增强一次，而原型Bean目标对象可能被包装多次。

### 7、BeanValidationPostProcessor

默认不自动注册，Spring3.0开始支持。

提供对JSR-303验证规范支持。

根据afterInitialization是false/true决定调用postProcessBeforeInitialization或postProcessAfterInitialization来通过JSR-303规范验证Bean，默认false。

### 8、MethodValidationPostProcessor

Spring3.1开始支持，且只支持Hibernate Validator 4.2及更高版本，从Spring 3.2起可能将采取自动检测Bean Validation 1.1兼容的提供商且自动注册（Bean Validation 1.1 (JSR-349)正处于草案阶段，它将提供方法级别的验证，提供对方法级别的验证），目前默认不自动注册。

Bean Validation 1.1草案请参考http://jcp.org/en/jsr/detail?id=349    http://beanvalidation.org/。

提供对方法参数/方法返回值的进行验证（即前置条件/后置条件的支持），通过JSR-303注解验证，使用方式如：

public @NotNull Object myValidMethod(@NotNull String arg1, @Max(10) int arg2)

默认只对@org.springframework.validation.annotation.Validated注解的Bean进行验证，我们可以修改validatedAnnotationType为其他注解类型来支持其他注解验证。而且目前只支持Hibernate Validator实现，在未来版本可能支持其他实现。

有了这东西之后我们就不需要在进行如Assert.assertNotNull（）这种前置条件/后置条件的判断了。

### 9、ScheduledAnnotationBeanPostProcessor

当配置文件中有<task:annotation-driven>自动注册或@EnableScheduling自动注册。

提供对注解@Scheduled任务调度的支持。

postProcessAfterInitialization：通过查找Bean对象类上的@Scheduled注解来创建ScheduledMethodRunnable对象并注册任务调度方法（仅返回值为void且方法是无形式参数的才可以）。

可参考Spring官方文档的任务调度章节学习@Scheduled注解任务调度。

### 10、AsyncAnnotationBeanPostProcessor

当配置文件中有<task:annotation-driven>自动注册或@EnableAsync自动注册。

提供对@ Async和EJB3.1的@javax.ejb.Asynchronous注解的异步调用支持。

postProcessAfterInitialization：通过ProxyFactory创建目标对象的代理对象，默认使用AsyncAnnotationAdvisor（内部使用AsyncExecutionInterceptor 通过AsyncTaskExecutor（继承TaskExecutor）通过submit提交异步任务）。

可参考Spring官方文档的异步调用章节学习@Async注解异步调用。

### 11、ServletContextAwareProcessor

在使用Web容器时自动注册。

类似于ApplicationContextAwareProcessor，当你的Bean 实现了ServletContextAware/ ServletConfigAware会自动调用回调方法注入ServletContext/ ServletConfig。

## 七、BeanPostProcessor如何注册

1、如ApplicationContextAwareProcessor会在ApplicationContext容器启动时自动注册，而CommonAnnotationBeanPostProcessor和AutowiredAnnotationBeanPostProcessor会在当你使用<context:annotation-config>或<context:component-scan>配置时自动注册。

2、只要将BeanPostProcessor注册到容器中，Spring会在启动时自动获取并注册。

## 八、BeanPostProcessor的执行顺序

1、如果使用BeanFactory实现，非ApplicationContext实现，BeanPostProcessor执行顺序就是添加顺序。

2、如果使用的是AbstractApplicationContext（实现了ApplicationContext）的实现，则通过如下规则指定顺序。

2.1、PriorityOrdered（继承了Ordered），实现了该接口的BeanPostProcessor会在第一个顺序注册，标识高优先级顺序，即比实现Ordered的具有更高的优先级；

2.2、Ordered，实现了该接口的BeanPostProcessor会第二个顺序注册；

int HIGHEST_PRECEDENCE = Integer.MIN_VALUE;//最高优先级

int LOWEST_PRECEDENCE = Integer.MAX_VALUE;//最低优先级

即数字越小优先级越高，数字越大优先级越低，如0（高优先级）——1000（低优先级）

2.3、无序的，没有实现Ordered/ PriorityOrdered的会在第三个顺序注册；

2.4、内部Bean后处理器，实现了MergedBeanDefinitionPostProcessor接口的是内部Bean PostProcessor，将在最后且无序注册。

3、接下来我们看看内置的BeanPostProcessor执行顺序

//1注册实现了PriorityOrdered接口的BeanPostProcessor

//2注册实现了Ordered接口的BeanPostProcessor

AbstractAutoProxyCreator              实现了Ordered，order = Ordered.LOWEST_PRECEDENCE

MethodValidationPostProcessor          实现了Ordered，LOWEST_PRECEDENCE

ScheduledAnnotationBeanPostProcessor   实现了Ordered，LOWEST_PRECEDENCE

AsyncAnnotationBeanPostProcessor      实现了Ordered，order = Ordered.LOWEST_PRECEDENCE

//3注册无实现任何接口的BeanPostProcessor

BeanValidationPostProcessor            无序

ApplicationContextAwareProcessor       无序

ServletContextAwareProcessor          无序

//3 注册实现了MergedBeanDefinitionPostProcessor接口的BeanPostProcessor，且按照实现了Ordered的顺序进行注册，没有实现Ordered的默认为Ordered.LOWEST_PRECEDENCE。

PersistenceAnnotationBeanPostProcessor  实现了PriorityOrdered，Ordered.LOWEST_PRECEDENCE - 4

AutowiredAnnotationBeanPostProcessor   实现了PriorityOrdered，order = Ordered.LOWEST_PRECEDENCE - 2

RequiredAnnotationBeanPostProcessor    实现了PriorityOrdered，order = Ordered.LOWEST_PRECEDENCE - 1

CommonAnnotationBeanPostProcessor    实现了PriorityOrdered，Ordered.LOWEST_PRECEDENCE

 

从上到下顺序执行，如果order相同则我们应该认为同序（谁先执行不确定，其执行顺序根据注册顺序决定）。

## 九、完成Spring事务处理时自我调用的解决方案及一些实现方式的风险分析

场景请先参考请参考[Spring事务处理时自我调用的解决方案及一些实现方式的风险](https://www.iteye.com/topic/1122740)中的3.3、通过BeanPostProcessor 在目标对象中注入代理对象。

**分析：**

![img](http://dl.iteye.com/upload/attachment/0066/9385/334849c0-7186-3cd0-b981-f1d3664aa03a.png)

**问题出现在**5和9处：

5、使用步骤1处注册的SingletonFactory（ObjectFactory.getObject() 使用AnnotationAwareAspectJAutoProxyCreator的getEarlyBeanReference获取循环引用Bean），因此此处将返回A目标对象的代理对象；

 

9、此处调用AnnotationAwareAspectJAutoProxyCreator的postProcessAfterInitialization，但发现之前调用过AnnotationAwareAspectJAutoProxyCreator的getEarlyBeanReference获取代理对象，此处不再创建代理对象，而是直接返回目标对象，因此使用InjectBeanSelfProcessor不能注入代理对象；但此时的Spring容器中的A已经是代理对象了，因此我使用了从上下文重新获取A代理对象的方式注入（context.getBean(beanName)）。

 

此处的getEarlyBeanReference和postProcessAfterInitialization为什么是二者选一的请参考之前介绍的AbstractAutoProxyCreator。

 

到此问题我们分析完毕，实际项目中的循环依赖应该尽量避免，这违反了“无环依赖原则”。