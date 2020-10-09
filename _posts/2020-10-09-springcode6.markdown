---
layout:     post
title:      "Spring源码(六) -- Bean的获取"
subtitle:   ""
date:       2020-10-09
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - Spring源码
---

# 背景
我们上一节学习Bean加载流程中发现bean的获取最终都走到了getObjectForBeanInstance方法，这一节我们就来仔细学习下Bean是如何获取到的。

# FactoryBean
```
public interface FactoryBean<T> {
    //获取泛型对应对象实例
    @Nullable
    T getObject() throws Exception;
    
    //获取泛型类型
    @Nullable
    Class<?> getObjectType();
     
    //是否单例
    default boolean isSingleton() {
	return true;
    }
}
```
我们这里首先介绍下FactoryBean接口，用户可以实现FactoryBean接口定制实例化bean的逻辑，并且通过AbstractFactoryBean抽象类进行了bean生命周期，避免单例循环依赖等管理，getObjectForBeanInstance方法就可能根据FactoryBean的getObject()方法获取bean实例。

# getObjectForBeanInstance
```
protected Object getObjectForBeanInstance(
        Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

    //如果name是&符号开头的Factory类型引用，但是beanInstance不是FactroyBean返回报错
    if (BeanFactoryUtils.isFactoryDereference(name)) {
        if (beanInstance instanceof NullBean) {
            return beanInstance;
        }
        if (!(beanInstance instanceof FactoryBean)) {
            throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
        }
    }

    // 如果当前beanInstance是正常的bean不是FactoryBean实例，则直接返回
    if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
        return beanInstance;
    }
  
    //加载FactoryBean
    Object object = null;
    if (mbd == null) {
        // 尝试从缓存中获取bean
        object = getCachedObjectForFactoryBean(beanName);
    }
    if (object == null) {
        FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
        if (mbd == null && containsBeanDefinition(beanName)) {
            mbd = getMergedLocalBeanDefinition(beanName);
        }
        boolean synthetic = (mbd != null && mbd.isSynthetic());
        object = getObjectFromFactoryBean(factory, beanName, !synthetic);
    }
    return object;
}
```
我们看到上面是对FactoryBean进行一些判断和验证，最后走到了getObjectFromFactoryBean方法
```
protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
    //如果是单例直接从缓存中获取并且加锁保证全局唯一，如果非单例则直接获取
    if (factory.isSingleton() && containsSingleton(beanName)) {
        synchronized (getSingletonMutex()) {
            Object object = this.factoryBeanObjectCache.get(beanName);
            if (object == null) {
                object = doGetObjectFromFactoryBean(factory, beanName);
       
                Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
                if (alreadyThere != null) {
                    object = alreadyThere;
                }
                else {
                    if (shouldPostProcess) {
                        if (isSingletonCurrentlyInCreation(beanName)) {
                            return object;
                        }
                        beforeSingletonCreation(beanName);
                        try {
                            object = postProcessObjectFromFactoryBean(object, beanName);
                        }
                        catch (Throwable ex) {
                            throw new BeanCreationException(beanName,
                                    "Post-processing of FactoryBean's singleton object failed", ex);
                        }
                        finally {
                            afterSingletonCreation(beanName);
                        }
                    }
                    if (containsSingleton(beanName)) {
                        this.factoryBeanObjectCache.put(beanName, object);
                    }
                }
            }
            return object;
        }
    }
    else {
        Object object = doGetObjectFromFactoryBean(factory, beanName);
        if (shouldPostProcess) {
            try {
                object = postProcessObjectFromFactoryBean(object, beanName);
            }
            catch (Throwable ex) {
                throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
            }
        }
        return object;
    }
}

private Object doGetObjectFromFactoryBean(final FactoryBean<?> factory, final String beanName)
        throws BeanCreationException {

    Object object;
    try {
        if (System.getSecurityManager() != null) {
            AccessControlContext acc = getAccessControlContext();
            try {
                object = AccessController.doPrivileged((PrivilegedExceptionAction<Object>) factory::getObject, acc);
            }
            catch (PrivilegedActionException pae) {
                throw pae.getException();
            }
        }
        else {
            object = factory.getObject();
        }
    }
    catch (FactoryBeanNotInitializedException ex) {
        throw new BeanCurrentlyInCreationException(beanName, ex.toString());
    }
    catch (Throwable ex) {
        throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
    }

    if (object == null) {
        if (isSingletonCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(
                    beanName, "FactoryBean which is currently in creation returned null from getObject");
        }
        object = new NullBean();
    }
    return object;
}
```
我们可以看到最后在doGetObjectFromFactoryBean方法中执行factory.getObject()获取到bean实例。