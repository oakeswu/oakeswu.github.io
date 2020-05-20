---
layout:     post
title:      "AccessibleObject类"
subtitle:   ""
date:       2020-05-06
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - Java反射
---

# AccessibleObject是什么
```
The AccessibleObject class is the base class for Field, Method and
Constructor objects.  It provides the ability to flag a reflected
object as suppressing default Java language access control check
when it is used. 
```
我们可以看下jdk自带的注释，说AccessibleObject 类是Field，Method，Constructors对象的基本类，它可以破解java对反射对象的访问控制检查。

# 源码（精简版）
```
public class AccessibleObject implements AnnotatedElement {
    /**
    * The Permission object that is used to check whether a client
    * has sufficient privilege to defeat Java language access
    * control checks.
    **/
    static final private java.security.Permission ACCESS_PERMISSION =
    new ReflectPermission("suppressAccessChecks");
    
    // Indicates whether language-level access checks are overridden
    // by this object. Initializes to "false". This field is used by
    // Field, Method, and Constructor.
    boolean override;
    
    //Get the value of the {@code accessible} flag for this object.
    public boolean isAccessible() {
        return override;
    }
    
    public static void setAccessible(AccessibleObject[] array, boolean flag)
            throws SecurityException {
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) sm.checkPermission(ACCESS_PERMISSION);
        for (int i = 0; i < array.length; i++) {
            setAccessible0(array[i], flag);
        }
    }
    
    public void setAccessible(boolean flag) throws SecurityException {
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) sm.checkPermission(ACCESS_PERMISSION);
        setAccessible0(this, flag);
    }
    
    private static void setAccessible0(AccessibleObject obj, boolean flag)
            throws SecurityException
    {
        if (obj instanceof Constructor && flag == true) {
            Constructor<?> c = (Constructor<?>)obj;
            if (c.getDeclaringClass() == Class.class) {
                throw new SecurityException("Cannot make a java.lang.Class" +
                        " constructor accessible");
            }
        }
        obj.override = flag;
    }
}
```
里面最主要方法就是setAccessible(boolean flag)，这个方法可以改变Field，Method，Constructor的访问控制，在反射中运用较多