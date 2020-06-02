---
layout:     post
title:      "Class类"
subtitle:   ""
date:       2020-05-06
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - Java反射
---

# 什么是Class类
Class类是java.lang包中的类，是java反射中的一个特别重要的类，因为Class类可以获取到类中的方法，字段等信息，等会看下Class类的具体源码

![类关系.png](/img/doc/reflect/reflect1one.png)



# Class类源码（精简）
```
public final class Class<T> implements java.io.Serializable, GenericDeclaration, Type, AnnotatedElement {
    private volatile transient Constructor<T> cachedConstructor;
    private volatile transient Class<?>       newInstanceCallerCache;

    //class的构造函数 
    private Class(ClassLoader loader) {
        // Initialize final field for classLoader.  The initialization value of non-null
        // prevents future JIT optimizations from assuming this final field is null.
        classLoader = loader;
    }
  
    @CallerSensitive
    public static Class<?> forName(String className)  throws ClassNotFoundException {
        Class<?> caller = Reflection.getCallerClass();
        return forName0(className, true, ClassLoader.getClassLoader(caller), caller);
    }

    @CallerSensitive
    public static Class<?> forName(String name, boolean initialize,  ClassLoader loader)
        throws ClassNotFoundException
    {
        Class<?> caller = null;
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            // Reflective call to get caller class is only needed if a security manager
            // is present.  Avoid the overhead of making this call otherwise.
            caller = Reflection.getCallerClass();
            if (sun.misc.VM.isSystemDomainLoader(loader)) {
                ClassLoader ccl = ClassLoader.getClassLoader(caller);
                if (!sun.misc.VM.isSystemDomainLoader(ccl)) {
                    sm.checkPermission(
                        SecurityConstants.GET_CLASSLOADER_PERMISSION);
                }
            }
        }
        return forName0(name, initialize, loader, caller);
    }

    /** Called after security check for system loader access checks have been made. */
    private static native Class<?> forName0(String name, boolean initialize, ClassLoader loader, Class<?> caller) throws ClassNotFoundException;

    @CallerSensitive
    public T newInstance()
        throws InstantiationException, IllegalAccessException
    {
        if (System.getSecurityManager() != null) {
            checkMemberAccess(Member.PUBLIC, Reflection.getCallerClass(), false);
        }

        // NOTE: the following code may not be strictly correct under
        // the current Java memory model.

        // Constructor lookup
        if (cachedConstructor == null) {
            if (this == Class.class) {
                throw new IllegalAccessException(
                    "Can not call newInstance() on the Class for java.lang.Class"
                );
            }
            try {
                Class<?>[] empty = {};
                final Constructor<T> c = getConstructor0(empty, Member.DECLARED);
                // Disable accessibility checks on the constructor
                // since we have to do the security check here anyway
                // (the stack depth is wrong for the Constructor's
                // security check to work)
                java.security.AccessController.doPrivileged(
                    new java.security.PrivilegedAction<Void>() {
                        public Void run() {
                                c.setAccessible(true);
                                return null;
                            }
                        });
                cachedConstructor = c;
            } catch (NoSuchMethodException e) {
                throw (InstantiationException)
                    new InstantiationException(getName()).initCause(e);
            }
        }
        Constructor<T> tmpConstructor = cachedConstructor;
        // Security check (same as in java.lang.reflect.Constructor)
        int modifiers = tmpConstructor.getModifiers();
        if (!Reflection.quickCheckMemberAccess(this, modifiers)) {
            Class<?> caller = Reflection.getCallerClass();
            if (newInstanceCallerCache != caller) {
                Reflection.ensureMemberAccess(caller, this, null, modifiers);
                newInstanceCallerCache = caller;
            }
        }
        // Run constructor
        try {
            return tmpConstructor.newInstance((Object[])null);
        } catch (InvocationTargetException e) {
            Unsafe.getUnsafe().throwException(e.getTargetException());
            // Not reached
            return null;
        }
    }
    
    // 入参是否是当前类的对象
    public native boolean isInstance(Object obj);
    // 入参是否是当前类的超类或相同类
    public native boolean isAssignableFrom(Class<?> cls);
    //当前类是否是接口
    public native boolean isInterface();
    //当前类是否是数组
    public native boolean isArray();
    //当前类是否是基本数据类型
    public native boolean isPrimitive();
    //当前类是否是注解
    public boolean isAnnotation() {
        return (getModifiers() & ANNOTATION) != 0;
    }
    //当前类是否jvm自己生成类
    public boolean isSynthetic() {
        return (getModifiers() & SYNTHETIC) != 0;
    }
    //获取当前类的类加载器
    @CallerSensitive
    public ClassLoader getClassLoader() {
        ClassLoader cl = getClassLoader0();
        if (cl == null)
            return null;
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            ClassLoader.checkClassLoaderPermission(cl, Reflection.getCallerClass());
        }
        return cl;
    }
    // Package-private to allow ClassLoader access
    ClassLoader getClassLoader0() { return classLoader; }
    // Initialized in JVM not by private constructo
    private final ClassLoader classLoader;

    //返回泛型类中的泛型参数数组
    public TypeVariable<Class<T>>[] getTypeParameters() {
        ClassRepository info = getGenericInfo();
        if (info != null)
            return (TypeVariable<Class<T>>[])info.getTypeParameters();
        else
            return (TypeVariable<Class<T>>[])new TypeVariable<?>[0];
    }

    //获取当前类的超类
    public native Class<? super T> getSuperclass();
    //获取当前类的直接超类
    public Type getGenericSuperclass() {
        ClassRepository info = getGenericInfo();
        if (info == null) {
            return getSuperclass();
        }

        // Historical irregularity:
        // Generic signature marks interfaces with superclass = Object
        // but this API returns null for interfaces
        if (isInterface()) {
            return null;
        }

        return info.getSuperclass();
    }

    //获取当前类实现的接口数组
    public Class<?>[] getInterfaces() {
        ReflectionData<T> rd = reflectionData();
        if (rd == null) {
            // no cloning required
            return getInterfaces0();
        } else {
            Class<?>[] interfaces = rd.interfaces;
            if (interfaces == null) {
                interfaces = getInterfaces0();
                rd.interfaces = interfaces;
            }
            // defensively copy before handing over to user code
            return interfaces.clone();
        }
    }

    private native Class<?>[] getInterfaces0();
    //获取当前类直接实现的接口数组
    public Type[] getGenericInterfaces() {
        ClassRepository info = getGenericInfo();
        return (info == null) ?  getInterfaces() : info.getSuperInterfaces();
    }

    //获取当前数组的元素类型
    public native Class<?> getComponentType();
    //获取当前类的修饰符
    public native int getModifiers();
    //获取当前类的签名
    public native Object[] getSigners();
    
    //获取当前类的类名
    public String getSimpleName() {
        if (isArray())
            return getComponentType().getSimpleName()+"[]";

        String simpleName = getSimpleBinaryName();
        if (simpleName == null) { // top level class
            simpleName = getName();
            return simpleName.substring(simpleName.lastIndexOf(".")+1); // strip the package name
        }
        int length = simpleName.length();
        if (length < 1 || simpleName.charAt(0) != '$')
            throw new InternalError("Malformed class name");
        int index = 1;
        while (index < length && isAsciiDigit(simpleName.charAt(index)))
            index++;
        return simpleName.substring(index);
    }

    //获取当前public修饰的内部类和接口
    public Class<?>[] getClasses() {
        checkMemberAccess(Member.PUBLIC, Reflection.getCallerClass(), false);
        return java.security.AccessController.doPrivileged(
            new java.security.PrivilegedAction<Class<?>[]>() {
                public Class<?>[] run() {
                    List<Class<?>> list = new ArrayList<>();
                    Class<?> currentClass = Class.this;
                    while (currentClass != null) {
                        Class<?>[] members = currentClass.getDeclaredClasses();
                        for (int i = 0; i < members.length; i++) {
                            if (Modifier.isPublic(members[i].getModifiers())) {
                                list.add(members[i]);
                            }
                        }
                        currentClass = currentClass.getSuperclass();
                    }
                    return list.toArray(new Class<?>[0]);
                }
            });
    }
    //获取当前类所有内部类和接口，不过排除直接继承
    public Class<?>[] getDeclaredClasses() throws SecurityException {
        checkMemberAccess(Member.DECLARED, Reflection.getCallerClass(), false);
        return getDeclaredClasses0();
    }

    //获取当前类中public修饰的字段
    public Field[] getFields() throws SecurityException {
        checkMemberAccess(Member.PUBLIC, Reflection.getCallerClass(), true);
        return copyFields(privateGetPublicFields(null));
    }
    //获取指定名称的方法
    public Field getField(String name)
        throws NoSuchFieldException, SecurityException {
        checkMemberAccess(Member.PUBLIC, Reflection.getCallerClass(), true);
        Field field = getField0(name);
        if (field == null) {
            throw new NoSuchFieldException(name);
        }
        return field;
    }
    //获取所有声明的字段
    public Field[] getDeclaredFields() throws SecurityException {
        checkMemberAccess(Member.DECLARED, Reflection.getCallerClass(), true);
        return copyFields(privateGetDeclaredFields(false));
    }

    //获取public修饰的方法
    public Method[] getMethods() throws SecurityException {
        checkMemberAccess(Member.PUBLIC, Reflection.getCallerClass(), true);
        return copyMethods(privateGetPublicMethods());
    }
    //获取指定传参类型和名称的方法
    public Method getMethod(String name, Class<?>... parameterTypes)
        throws NoSuchMethodException, SecurityException {
        checkMemberAccess(Member.PUBLIC, Reflection.getCallerClass(), true);
        Method method = getMethod0(name, parameterTypes, true);
        if (method == null) {
            throw new NoSuchMethodException(getName() + "." + name + argumentTypesToString(parameterTypes));
        }
        return method;
    }
    //获取所有声明的类
    public Class<?>[] getDeclaredClasses() throws SecurityException {
        checkMemberAccess(Member.DECLARED, Reflection.getCallerClass(), false);
        return getDeclaredClasses0();
    }
  
    //获取public修饰的构造器
    public Constructor<?>[] getConstructors() throws SecurityException {
        checkMemberAccess(Member.PUBLIC, Reflection.getCallerClass(), true);
        return copyConstructors(privateGetDeclaredConstructors(true));
    }
    //获取传参的构造器
    public Constructor<T> getConstructor(Class<?>... parameterTypes)
        throws NoSuchMethodException, SecurityException {
        checkMemberAccess(Member.PUBLIC, Reflection.getCallerClass(), true);
        return getConstructor0(parameterTypes, Member.PUBLIC);
    }
    //获取所有声明的构造器
    public Constructor<?>[] getDeclaredConstructors() throws SecurityException {
        checkMemberAccess(Member.DECLARED, Reflection.getCallerClass(), true);
        return copyConstructors(privateGetDeclaredConstructors(false));
    }
}
```
# 例子
```
package reflect;

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
```
```
package reflect;

import java.lang.reflect.Field;
import java.lang.reflect.Method;

public class EmployeeDtoReflect {

    public void test(Class<?> clazz) throws Exception{
        if (clazz.isInstance(new EmployeeDto())) {
            EmployeeDto employee = (EmployeeDto) clazz.newInstance();
            System.out.println("EmployeeDeo:" + employee);
            Method[] methods = clazz.getDeclaredMethods();
            for(Method method : methods) {
                System.out.println("Method:" + method.getName());
            }
            Field[] fileds = clazz.getDeclaredFields();
            for (Field field : fileds) {
                System.out.println("Fields:" + field.getName());
            }
        }
    }

    public static void main(String[] args) {
        try {
            new EmployeeDtoReflect().test(EmployeeDto.class);
        } catch (Exception e) {
            e.printStackTrace();
        }

    }
}
```
我们可以看下执行结果
```
Connected to the target VM, address: '127.0.0.1:54290', transport: 'socket'
Disconnected from the target VM, address: '127.0.0.1:54290', transport: 'socket'
EmployeeDeo:age:0,name:default
Method:toString
Method:getName
Method:setName
Method:setAge
Method:isOlder
Method:getAge
Fields:age
Fields:name

Process finished with exit code 0
```
# 总结
- 上面测试代码只测试了newInstance()方法，getDeclaredMethods()和getDeclaredFields()方法. 测试代码通过类.class获取Class类，我们也阔以通过Class.forName("类路径")获取，这个在获取具体数据库驱动时常用到。
- Class类是Java反射中基础的一个类，因为它能获取到一个类中的变量，方法，构造函数等，像spring等开源框架中大量使用了反射，但是反射的效率一般，并且有可能破坏java中访问修饰符的访问权限控制