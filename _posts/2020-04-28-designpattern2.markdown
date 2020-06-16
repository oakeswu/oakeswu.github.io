---
layout:     post
title:      "设计模式(二) -- 抽象工厂模式"
subtitle:   ""
date:       2020-04-28
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - 设计模式
---

# 什么是抽象工厂模式
前面有说过普通工厂模式，是一网上教学系统来举例的，当时创建了SchoolPerson接口和SchoolPersonFactory工厂类，而抽象工厂模式就在SchoolPersonFactory工厂类上再进行一次封装

# 如何使用
```
public interface CompanyPerson {
    /**
     * 开始上班
     */
    void startWork();
}

public class Boss implements CompanyPerson{
    @Override
    public void startWork() {
        System.out.println("开始会议");
        System.out.println("聆听汇报");
        System.out.println("------");
    }
}

public class Employee implements CompanyPerson {
    @Override
    public void startWork() {
        System.out.println("开始办公");
        System.out.println("准备材料");
        System.out.println("------");
    }
}

public class CompanyPersonFactory extends Person {
    @Override
    public SchoolPerson getSchoolPerson(String name) {
        return null;
    }

    @Override
    public CompanyPerson getCompanyPerson(String name) {
        if (StringUtils.isEmpty(name)) {
            return null;
        }
        if ("boss".equalsIgnoreCase(name)) {
            return new Boss();
        } else if ("employee".equalsIgnoreCase(name)) {
            return new Employee();
        }
        return null;
    }
}
```
我们模仿普通工厂方法的SchoolPerson创造了一个CompanyPerson，我们可以看下执行结果
```
Connected to the target VM, address: '127.0.0.1:55625', transport: 'socket'
开始会议
聆听汇报
------
Disconnected from the target VM, address: '127.0.0.1:55625', transport: 'socket'
开始办公
准备材料
------

Process finished with exit code 0
```
但是大家可以看一下CompanyPersonFactory继承了一个Person类
```
public abstract class Person {
    /**
     * 获取学校人员
     */
    public abstract SchoolPerson getSchoolPerson(String name);

    /**
     * 获取公司人员
     */
    public abstract CompanyPerson getCompanyPerson(String name);
}

public class SchoolPersonFactory extends Person {

    @Override
    public SchoolPerson getSchoolPerson(String name) {
        if (StringUtils.isEmpty(name)) {
            return null;
        }
        if ("student".equalsIgnoreCase(name)) {
            return new Student();
        } else if ("teacher".equalsIgnoreCase(name)) {
            return new Teacher();
        }
        return null;
    }

    @Override
    public CompanyPerson getCompanyPerson(String name) {
        return null;
    }
}
```
我们可以看到Person类里面有两个抽象方法getSchoolPerson和getCompanyPerson，然后SchoolPersonFactory和CompanyPersonFactory都是Person类的子类，方便等会工厂类获取具体工厂
```
public class PersonFactory {
    public Person getPersonFactory(String name) {
        if (StringUtils.isEmpty(name)) {
            return null;
        }
        if ("company".equalsIgnoreCase(name)) {
            return new CompanyPersonFactory();
        } else if ("school".equalsIgnoreCase(name)) {
            return new SchoolPersonFactory();
        }
        return null;
    }
}

public class PersonFactoryTest {

    public static void main(String[] args) {
        PersonFactory personFactory = new PersonFactory();
        Person companyPersonFactory = personFactory.getPersonFactory("company");
        companyPersonFactory.getCompanyPerson("boss").startWork();
        companyPersonFactory.getCompanyPerson("employee").startWork();

        Person schoolPersonFactory = personFactory.getPersonFactory("school");
        schoolPersonFactory.getSchoolPerson("teacher").attendClass();
        schoolPersonFactory.getSchoolPerson("student").attendClass();
    }
}
```
上面就是一个PersonFactory工厂类，我们可以看下执行结果
```
Connected to the target VM, address: '127.0.0.1:56179', transport: 'socket'
开始会议
聆听汇报
------
开始办公
准备材料
------
打开备案
开始教学
------
打开书本
开始学习
------
Disconnected from the target VM, address: '127.0.0.1:56179', transport: 'socket'

Process finished with exit code 0
```
所以我们可以看到代码执行成功，之前有两个分别的工厂类SchoolPersonFactory和CompanyPersonFactory通过一个PersonFactory工厂获得，所以抽象工厂模式就是对普通工厂模式的进一步工厂




