---
layout:     post
title:      "Executable类"
subtitle:   ""
date:       2020-05-06
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - Java反射
---

# Excutable是什么
Excutable是java.lang.reflect下的抽象类，也是Method和Constructor类的超类，可以看下jdk文档的注释
```
/**
 * A shared superclass for the common functionality of {@link Method}
 * and {@link Constructor}.
 **/
```

# 源码（精简版）
```
public abstract class Executable extends AccessibleObject
    implements Member, GenericDeclaration {
    
    // 获取调用对象的Class类
    public abstract Class<?> getDeclaringClass();

    // 获取调用对象的名称
    public abstract String getName();

    // 获取调用对象的修饰符数量
    public abstract int getModifiers();
    
    // 获取调用对象的参数类型数组
    public abstract Class<?>[] getParameterTypes();

    // 获取调用对象的参数个数
    public int getParameterCount() {
        throw new AbstractMethodError();
    }

    // 获取调用对象的参数
    public Parameter[] getParameters() {
        return privateGetParameters().clone();
    }

    // 返回指定类型的注解
    public <T extends Annotation> T getAnnotation(Class<T> annotationClass) {
        Objects.requireNonNull(annotationClass);
        return annotationClass.cast(declaredAnnotations().get(annotationClass));
    }
    
    // 返回指定类型的注解（会检查注解是否是可重复类型注解，是则返回一个或多个）
    public <T extends Annotation> T[] getAnnotationsByType(Class<T> annotationClass) {
        Objects.requireNonNull(annotationClass);

        return AnnotationSupport.getDirectlyAndIndirectlyPresent(declaredAnnotations(), annotationClass);
    }
    
    // 获取所有注解（不包含继承的）
    public Annotation[] getDeclaredAnnotations()  {
        return AnnotationParser.toArray(declaredAnnotations());
    }

    private transient Map<Class<? extends Annotation>, Annotation> declaredAnnotations;

    private synchronized  Map<Class<? extends Annotation>, Annotation> declaredAnnotations() {
        if (declaredAnnotations == null) {
            Executable root = getRoot();
            if (root != null) {
                declaredAnnotations = root.declaredAnnotations();
            } else {
                declaredAnnotations = AnnotationParser.parseAnnotations(
                    getAnnotationBytes(),
                    sun.misc.SharedSecrets.getJavaLangAccess().
                    getConstantPool(getDeclaringClass()),
                    getDeclaringClass());
            }
        }
        return declaredAnnotations;
    }

    // 获取调用对象的返回类型，可通过AnnotatedType.getType获取
   public abstract AnnotatedType getAnnotatedReturnType();

    // 获取调用对象的异常，可通过AnnotatedType.getType获取
    public AnnotatedType[] getAnnotatedExceptionTypes() {
        return TypeAnnotationParser.buildAnnotatedTypes(getTypeAnnotationBytes0(),
                sun.misc.SharedSecrets.getJavaLangAccess().
                        getConstantPool(getDeclaringClass()),
                this,
                getDeclaringClass(),
                getGenericExceptionTypes(),
                TypeAnnotation.TypeAnnotationTarget.THROWS);
    }
    
    // 获取调用对象的接收类型
    public AnnotatedType getAnnotatedReceiverType() {
        if (Modifier.isStatic(this.getModifiers()))
            return null;
        return TypeAnnotationParser.buildAnnotatedType(getTypeAnnotationBytes0(),
                sun.misc.SharedSecrets.getJavaLangAccess().
                        getConstantPool(getDeclaringClass()),
                this,
                getDeclaringClass(),
                getDeclaringClass(),
                TypeAnnotation.TypeAnnotationTarget.METHOD_RECEIVER);
    }
    
    // 获取调用对象的参数类型
    public AnnotatedType[] getAnnotatedParameterTypes() {
        return TypeAnnotationParser.buildAnnotatedTypes(getTypeAnnotationBytes0(),
                sun.misc.SharedSecrets.getJavaLangAccess().
                        getConstantPool(getDeclaringClass()),
                this,
                getDeclaringClass(),
                getAllGenericParameterTypes(),
                TypeAnnotation.TypeAnnotationTarget.METHOD_FORMAL_PARAMETER);
    }
}
```