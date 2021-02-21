---
layout:     post
title:      "设计模式之禅(一) -- 里氏替换原则"
subtitle:   ""
date:       2021-02-22
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - 设计模式
---

**本文及后续设计模式之禅主要参考《设计模式之禅》**
# 定义
所有引用基类的地方必须能透明地使用其子类的对象，通俗点讲就是只要父类出现的地方替换成子类不产生任何异常或错误

# 四层含义
- 子类必须完全实现父类的方法
该含义是为了保证子类和父类类型一致，书中举了一个cs游戏的例子，有一个包含shoot方法的抽象类abstrctGun，如果是子类是一个玩具枪继承了abstractGun但是又无法实现shoot方法，那么在cs中使用玩具枪就无法干掉对方。

- 子类可以有自己的个性
里氏替换原则可以正着用，无法反过来用，子类因为有自己的个性所以无法用父类去替换。

- 覆盖或实现父类的方法时输入参数可以被放大
```
public class Father {

    public Collection doSomething(HashMap map) {
        System.out.println("father");
        return map.values();
    }

    public Collection doSomething2(Map map) {
        System.out.println("father");
        return map.values();
    }
}

public class Son extends Father {

    public Collection doSomething(Map map) {
        System.out.println("son");
        return map.values();
    }

    public Collection doSomething2(HashMap map) {
        System.out.println("son");
        return map.values();
    }
}

public class Client {

    public static void main(String[] args) {
        Father father = new Father();
        Son son = new Son();

        HashMap map = new HashMap();

        father.doSomething(map);
        son.doSomething(map);

        father.doSomething2(map);
        son.doSomething2(map);
    }
}
```
![](http://upload-images.jianshu.io/upload_images/9082703-420d3330ec7d25fe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们看下输出结果中子类输入参数比父类大的时候满足里氏替换原则，子类输入参数比父类小的时候输出不一致了。

- 覆写或实现父类方法时输出结果可以被缩小
这是因为覆写方法本身的要求就是子类的相同方法返回值必须小于等于父类相同方法的返回值。

# 思考
目前个人只理解到里氏替换原则是跟面向接口原则配合使用的一个原则，也是多态思想的具体体现，在代码迁移时依赖的父类或抽象可以很方便的特定指向完成一些简单的工作，网友如果有其他的经验希望能一起分享讨论。
