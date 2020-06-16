---
layout:     post
title:      "设计模式(一) -- 普通工厂模式"
subtitle:   ""
date:       2020-04-27
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - 设计模式
---

# 项目背景
项目需要输出UGC内容，包括了文章，视频，语音等内容，输出的操作都包含了查询数据，获取数据操作，所以可以使用工厂模式来控制到底操作的是文章，视频还是其他

# 什么是普通工厂模式
工厂模式是java中常见的设计模式，在工厂模式中，客户端不需要显式的创建对象，而可以通过调用接口传参创建或执行方法，对象的具体实现就可以对客户端透明，客户端只需要关心接口的调用即可。

# 如何使用
当前疫情期间学校采用了网上教学的方式，此时学生和老师登陆教学系统时会产生各式不同的操作。所以我们就简单用工厂模式实现一下
```
public interface SchoolPerson {
    /**
     * 登陆系统上课
     */
    void attendClass();
}

public class Student implements SchoolPerson {

    public void attendClass() {
        System.out.println("打开书本");
        System.out.println("开始学习");
        System.out.println("------");
    }
}

public class Teacher implements SchoolPerson {

    public void attendClass() {
        System.out.println("打开备案");
        System.out.println("开始教学");
        System.out.println("------");
    }
}

public class SchoolPersonFactory {
    
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
}

public class SchoolPersonTest {
    public static void main(String[] args) {
        SchoolPersonFactory schoolPersonFactory = new SchoolPersonFactory();      

        SchoolPerson studentOne = schoolPersonFactory .getSchoolPerson("student");
        studentOne.attendClass();

        SchoolPerson teacherOne = schoolPersonFactory .getSchoolPerson("teacher");
        teacherOne.attendClass();
    }
}
```
上面就是我们的简单代码，我们可以看下执行结果
```
Connected to the target VM, address: '127.0.0.1:55743', transport: 'socket'
打开书本
开始学习
------
打开备案
开始教学
------
Disconnected from the target VM, address: '127.0.0.1:55743', transport: 'socket'

Process finished with exit code 0
```
所以可以看到，对于学生和老师并不需要显示的创建自己的对象然后上课，而是可以直接通过传参调用接口的getSchoolPerson方法获取，这可以让调用方不用关心服务端的代码，但是工厂方法会增加类之间的耦合，需要依据实际情况使用。