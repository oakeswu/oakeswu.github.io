---
layout:     post
title:      "观察者模式"
subtitle:   ""
date:       2020-04-29
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - 设计模式
---

# 背景
在许多公共工具中我们可能听说过一个函数叫回调函数，回调函数就是被传入然后再调用的函数，观察者模式传入一个观察者，然后有变化的时候会再调用一下观察者执行任务，那我们就来写一个消费者观察厂商是否出新品，厂商有新品的时候消费者就会购买的代码实例

#代码实例
```
//定义一个消费者接口
public interface Consumer {
    /**
     * 购买行为
     */
    void buy();
}

//定义厂家
public class Producter {
    private List<Consumer> consumerList = new ArrayList<>();
    //增加消费者作为观察者
    public void addConsumer(Consumer consumer) {
        consumerList.add(consumer);
    }
    
    public void startSelling() {
        for (Consumer consumer : consumerList) {
            consumer.buy();
        }
    }
}

//定义一个普通消费者
public class NormalConsumer implements Consumer {
    @Override
    public void buy() {
        System.out.println("普通消费者买一个！");
    }
}

//定义一个特殊消费者
public class SpecialConsumer implements Consumer {
    @Override
    public void buy() {
        System.out.println("特殊消费者买好多个！");
    }
}
```
上面就会发现当厂家自己开始售卖的时候，作为观察者的消费者就会开始执行购买操作。我们开下执行结果
```
public class ConsumerTest {
    public static void main(String[] args) {
        Producter producter = new Producter();
        //添加两个观察者
        producter.addConsumer(new NormalConsumer());
        producter.addConsumer(new SpecialConsumer());

        producter.startSelling();
    }
}
```
执行结果如图所示
![](https://upload-images.jianshu.io/upload_images/9082703-39ef754b1346e43d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
上面就是一个完整的观察者模式案例。本质就是回调函数的使用。