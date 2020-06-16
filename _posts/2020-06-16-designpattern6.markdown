---
layout:     post
title:      "设计模式(六) -- 代理模式"
subtitle:   ""
date:       2020-06-16
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - 设计模式
---

# 什么是代理模式
代理模式可以理解成一个类代表另一个类的功能并可加自己的代理功能。简单来说可以理解成汽车代理商代理汽车工厂卖汽车一样，可以增加一系列增值服务。代理模式分为静态代理和动态代理，其中动态代理在Spring AOP是个典型应用。

# 静态代理
```
public interface CarSale {
    /**
     * 卖车
     */
    void sellCar();
}

public class CarFactory implements CarSale{

    @Override
    public void sellCar() {
        System.out.println("造车交付");
    }
}

public class CarProxy implements CarSale{
    private CarFactory carFactory;

    public CarProxy() {
        carFactory = new CarFactory();
    }

    @Override
    public void sellCar() {
        System.out.println("增值服务");
        carFactory.sellCar();
    }
}
```
我们可以看到我们先定义了一个接口，然后实现了汽车工厂和汽车经销商，然后我们看到汽车代理商就是一个代理类，通过代理方法可以增加一系列增值服务。我们看下测试代码并运行
```
public class CarSaleTest {

    public static void main(String[] args) {
        CarFactory carFactory = new CarFactory();
        //厂家自营卖车
        carFactory.sellCar();
        System.out.println("----------");
        //经销商卖车
        CarProxy carProxy = new CarProxy();
        carProxy.sellCar();
    }
}
```
```
Connected to the target VM, address: '127.0.0.1:51829', transport: 'socket'
Disconnected from the target VM, address: '127.0.0.1:51829', transport: 'socket'
造车交付
----------
增值服务
造车交付

Process finished with exit code 0
```
可以发现静态代理中代理对象在代码运行前就已经确定好了，而动态代理的区别就在于代理对象在代码运行时才会动态的创建。我们现在来看下动态代理
# 动态代理
- JDK动态代理

```
public class DynamicProxyHandler implements InvocationHandler {
    private Object object;

    public DynamicProxyHandler(Object object) {
        this.object = object;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (proxy instanceof CarSale && "sellCar".equalsIgnoreCase(method.getName())) {
            System.out.println("增值服务");
        }
        Object result = method.invoke(object, args);
        return result;
    }
}

public class DynamicProxyTest {

    public static void main(String[] args) {
        CarSale carSale = new CarFactory();

        CarSale carProxy = (CarSale) Proxy.newProxyInstance(CarSale.class.getClassLoader(), new Class[]{CarSale.class}, new DynamicProxyHandler(carSale));
        carProxy.sellCar();
    }
}
```
我们可以看到动态代理其实用到了Java反射，不显式的定义代理类而是通过编写一个动态代理Handler去动态创建任何满足条件的动态对象。我们上文也提到动态代理是Spring AOP的典型应用，AOP中使用了两种动态代理，一种就是上面的JDK动态代理，另一种是cglib字节码技术。如果目标对象实现了接口如同上面的CarFactory实现了CarSale接口，就会优先选择JDK动态代理技术，如果目标没有实现接口，就会选择使用cglib字节码技术。现在我们再来看看cglib如何实现动态代理。
- cglib动态代理
 
```
public class NewCarFactory {
    public void sellCar() {
        System.out.println("造车交付");
    }
}

public class CglibProxy implements MethodInterceptor {
    private Object object;

    public Object newInstance(Object object) {
        this.object = object;
        Enhancer enhancer = new Enhancer();
        //设置被代理的对象
        enhancer.setSuperclass(this.object.getClass());
        //将被代理对象的方法都转发到intercept方法上拦截
        enhancer.setCallback(this);
        //返回被代理对象
        return enhancer.create();
    }

    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        String methodName = method.getName();
        if ("sellCar".equalsIgnoreCase(methodName)) {
            System.out.println("增值服务");
        }
        return proxy.invokeSuper(obj, args);
    }
}

public class CglibProxyTest {
    public static void main(String[] args) {
        NewCarFactory newCarFactory = (NewCarFactory)new CglibProxy().newInstance(new NewCarFactory());
        newCarFactory.sellCar();
    }
}
```
可以通过运行上面两个动态代理的测试代码验证下，我们可以发现代理都能给原始类增加代理格外功能，具有高拓展性，隔离等特性，但是也会增加工作量和间接访问的耗时。