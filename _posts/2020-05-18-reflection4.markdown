---
layout:     post
title:      "Constructor类"
subtitle:   ""
date:       2020-05-18
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - Java反射
---

# 类图
之前我们用反射创建一个类的对象可以通过该Class.newInstance()方法，但是可以发现这个方法只能构造包含无参构造函数的类，如果不包含则会报NoSuchMethodException异常，那么这个时候就需要用到Constructor类了。我们先看看Constructor类的类关系图
![类图](/img/doc/reflect/reflect4one.png)

# 源码（精简版）
```
public final class Constructor<T> extends Executable {
    //用来存储ConstructorAccessor
    private Constructor<T>      root;
    //通过传入参数构造相对应的实例
    public T newInstance(Object ... initargs)
        throws InstantiationException, IllegalAccessException,
               IllegalArgumentException, InvocationTargetException
    {
        if (!override) {
            if (!Reflection.quickCheckMemberAccess(clazz, modifiers)) {
                Class<?> caller = Reflection.getCallerClass();
                checkAccess(caller, clazz, null, modifiers);
            }
        }
        if ((clazz.getModifiers() & Modifier.ENUM) != 0)
            throw new IllegalArgumentException("Cannot reflectively create enum objects");
        ConstructorAccessor ca = constructorAccessor;   // read volatile
        if (ca == null) {
            ca = acquireConstructorAccessor();
        }
        @SuppressWarnings("unchecked")
        T inst = (T) ca.newInstance(initargs);
        return inst;
    }
    //获取ConstructorAccessor 
    private ConstructorAccessor acquireConstructorAccessor() {
        // First check to see if one has been created yet, and take it
        // if so.
        ConstructorAccessor tmp = null;
        if (root != null) tmp = root.getConstructorAccessor();
        if (tmp != null) {
            constructorAccessor = tmp;
        } else {
            // Otherwise fabricate one and propagate it up to the root
            tmp = reflectionFactory.newConstructorAccessor(this);
            setConstructorAccessor(tmp);
        }

        return tmp;
    }

    // 直接返回root里的ConstructorAccessor
    ConstructorAccessor getConstructorAccessor() {
        return constructorAccessor;
    }

    // 给root赋值
    void setConstructorAccessor(ConstructorAccessor accessor) {
        constructorAccessor = accessor;
        // Propagate up
        if (root != null) {
            root.setConstructorAccessor(accessor);
        }
    }
}
```
由于其他方法基本都是复写的父类等，所以这边就只看了newInstance方法，可以通过reflectionFactory.newConstructorAccessor(this);方法获取不同的ConstructorAccessor并调用newInstance创建一个实例

# 实例
```
public class EmployeeDto {
    private int age;
    private String name;

    EmployeeDto() {
        this.age = 0;
        this.name = "default";
    }

    EmployeeDto(int age, String name) {
        this.age = age;
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public boolean isOlder(int tmpAge) {
        return age > tmpAge;
    }

    @Override
    public String toString() {
        return "age:" + age + ",name:" + name;
    }
}

public class EmployeeConstructorTest {

    public static void main(String[] args) {
        try {
            Class clazz = Class.forName("reflect.EmployeeDto");
            Constructor[] constructors = clazz.getDeclaredConstructors();
            for (Constructor constructor : constructors) {
                if (constructor.getParameterCount() == 2) {
                    constructor.setAccessible(true);
                    EmployeeDto employeeDto = (EmployeeDto) constructor.newInstance(1,"test");
                    System.out.println(employeeDto);
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
EmployeeDto类有一个含参构造函数，然后main方法获取到该构造函数并newInstance创建一个实例并打印构建的对象，我们看看执行结果
```
Connected to the target VM, address: '127.0.0.1:65499', transport: 'socket'
age:1,name:test
Disconnected from the target VM, address: '127.0.0.1:65499', transport: 'socket'
```
执行没有问题