---
layout:     post
title:      "设计模式之禅(二) -- 依赖倒置原则"
subtitle:   ""
date:       2021-02-25
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - 设计模式
---

# 两层含义
- 高层模块不应该依赖底层模块，两者都应该依赖于抽象
低层模块：不可分割的原子逻辑实现就是低层模块
高层模块：由原子逻辑实现再组装就是高层模块

- 抽象不应该依赖细节，细节应该依赖抽象
抽象：接口或抽象类，不能被直接实例化
细节：具体实现类，可以直接被实例化

# 优劣分析
-  优点：
1：依赖倒置可以减少类之间的耦合，提高系统稳定性
```
public class Driver {
    public void drive(Benz benz) {
        benz.run();
    }
}

public class Benz {
    public void run() {
        System.out.println("奔驰汽车开始运行");
    }
}

public class Client {
    public static void main(String[] args) {
        Driver zhangsan = new Driver();
        Benz benz = new Benz();
        zhangsan.drive(benz);
    }
}

public class BMW {
    public void run() {
        System.out.println("宝马汽车开始运行");
    }
}
```
一开始Driver买了一辆奔驰车，然后按照逻辑Driver类就有了奔驰车的驾驶能力，但是zhangsan如果新买了一辆宝马车，按照现实来说用户能开奔驰车的情况下也能开宝马车，而此时问题就在于Driver类并没有提供宝马车，如果Driver类提供的是Car抽象类而不是Benz具体类就可以了。
```
public interface IDriver {
    //司机就能驾驶汽车
    public void drive(ICar car);
}

public class Driver implements IDriver {
    public void drive(ICar car) {
        car.run();
    }
}

public interface ICar {
    void run();
}

public class Benz implements ICar{
    @Override
    public void run() {
        System.out.println("奔驰汽车开始运行");
    }
}

public class BMW implements ICar {
    @Override
    public void run() {
        System.out.println("宝马汽车开始运行");
    }
}

public class Client {
    public static void main(String[] args) {
        Driver zhangsan = new Driver();
        Benz benz = new Benz();
        zhangsan.drive(benz);

        BMW bmw = new BMW();
        zhangsan.drive(bmw);
    }
}
```
我们只需要对Driver和Car进行一下抽象就能很好的解决上面的问题，并且zhangsan可以继续买各种车进行驾驶了。

2：两个类之间有依赖关系，只要制定出两者之间的接口就可以独立开发，并且项目之间的单元测试可以独立开发，典型应用就是TDD(测试驱动开发)

- 缺点：
1：抽象需要新增接口或者抽象类，并且抽象在代码中需要找到具体实现类看实际执行的逻辑

- 对比：
上面我们列举了优点和缺点，在实际工作中我们就会发现依赖倒置原则也更符合面向接口开发，对拓展开放对修改关闭等原则，但这不是绝对的，因为当系统非常简单且不需要后续拓展情况下直接依赖细节就可以了。所以任何工作需要以实际出发考虑。
