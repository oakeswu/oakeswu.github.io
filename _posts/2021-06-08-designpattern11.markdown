---
layout:     post
title:      "设计模式之禅(四) -- 迪米特法则"
subtitle:   ""
date:       2021-06-08
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - 设计模式
---

# 定义
迪米特法则也叫最少知道原则，即一个对象应该对其他对象有最少的了解，一个类对需要耦合的类知道的最少，也即高内聚低耦合。

# 四层含义
- 只和朋友交流（类只和必须关联的类去耦合）
- 朋友之间也是有距离的（耦合类之间减少）
- 是自己的就是自己的
- 谨慎使用Serializable接口

# 只和朋友交流
![](http://upload-images.jianshu.io/upload_images/9082703-050ec4bdf79e0dab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
public class Girl {
}

public class GroupLeader {

    public void countGirls(List<Girl> listGirls) {
        System.out.println("女生数量是：" + listGirls.size());
    }
}

public class Teacher {

    public void command(GroupLeader groupLeader) {
        List listGirls = new ArrayList();
        //初始化女生
        for (int i = 0; i < 20; i++) {
            listGirls.add(new Girl());
        }
        //告诉体育委员开始执行清查任务
        groupLeader.countGirls(listGirls);
    }
}

public class Client {

    public static void main(String[] args) {
        Teacher teacher = new Teacher();
        //老师发布命令
        teacher.command(new GroupLeader());
    }
}
```
我们看上面这个例子里面Teacher既跟GroupLeader交互，也跟Gril直接交互。这明显违背了只跟朋友交互的问题。
```
public class Girl {
}

public class GroupLeader {

    private List<Girl> listGirls;

    //传递全班的女生进来
    public GroupLeader(List<Girl> listGirls) {
        this.listGirls = listGirls;
    }

    //清查女生数量
    public void countGirls() {
        System.out.println("女生数量是：" + listGirls.size());
    }
}

public class Teacher {

    //老师对学生发布命令，清一下女生
    public void command(GroupLeader groupLeader) {
        //告诉体育委员开始执行清洁任务
        groupLeader.countGirls();
    }
}

public class Client {

    public static void main(String[] args) {
        //产生一个女生群体
        List<Girl> listGirls = new ArrayList<>();
        //初始化女生
        for (int i = 0; i < 20; i++) {
            listGirls.add(new Girl());
        }
        Teacher teacher = new Teacher();

        //老师发布命令
        teacher.command(new GroupLeader(listGirls));
    }
}
```
修改后的类图是
![](http://upload-images.jianshu.io/upload_images/9082703-bcdcba47a7db75cc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们发现代码更改后Teacher类只跟GroupLeader沟通，不再需要跟Girl类耦合。

# 朋友圈也是有距离的
![](http://upload-images.jianshu.io/upload_images/9082703-8c12b917d6eba843.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```
public class Wizard {
    private Random random = new Random(System.currentTimeMillis());
    //第一步
    public int first() {
        System.out.println("执行第一个方法...");
        return random.nextInt(100);
    }

    //第二步
    public int second() {
        System.out.println("执行第二个方法...");
        return random.nextInt(100);
    }

    //第三步
    public int third() {
        System.out.println("执行第三个方法...");
        return random.nextInt(100);
    }
}

public class InstallSoftware {

    public void installWizard(Wizard wizard) {
        int first = wizard.first();
        //根据first返回的结果，看是否执行second
        if (first > 50) {
            int second = wizard.second();
            //根据second返回的结果，看是否执行third
            if (second > 50) {
                int third = wizard.third();
                //根据third返回的结果，看是否执行first
                if (third > 50) {
                    wizard.first();
                }
            }
        }
    }
}

public class Client {

    public static void main(String[] args) {
       InstallSoftware invoker = new InstallSoftware();
       invoker.installWizard(new Wizard());
    }
}
```
我们看上面这一段逻辑就是很简单的一个安装软件的过程，似乎一眼看起来并没有问题，但其实wizard自身的安装逻辑方法很多都暴露给了调用方，如果任意一个安装方法返回类型变化必然会引起调用方变化，实际安装的变化应该隐藏在wizard内部，所以我们修改一下。
```
public class Wizard {
    private Random random = new Random(System.currentTimeMillis());
    //第一步
    private int first() {
        System.out.println("执行第一个方法...");
        return random.nextInt(100);
    }

    //第二步
    private int second() {
        System.out.println("执行第二个方法...");
        return random.nextInt(100);
    }

    //第三步
    private int third() {
        System.out.println("执行第三个方法...");
        return random.nextInt(100);
    }

    public void installWizard( ) {
        int first = first();
        //根据first返回的结果，看是否执行second
        if (first > 50) {
            int second = second();
            //根据second返回的结果，看是否执行third
            if (second > 50) {
                int third = third();
                //根据third返回的结果，看是否执行first
                if (third > 50) {
                    first();
                }
            }
        }
    }
}

public class InstallSoftware {

    public void installWizard(Wizard wizard) {
        wizard.installWizard();
    }
}

public class Client {

    public static void main(String[] args) {
       InstallSoftware invoker = new InstallSoftware();
       invoker.installWizard(new Wizard());
    }
}
```
我们将wizard的安装步骤方法改成private，并将安装步骤移到wizard中，迪米特法则要求类尽量不要对外公布太多的public方法和非静态的public变量，尽量内敛。

# 是自己的就是自己的
在实际场景中我们会遇到一个方法放在本类中也可以，放在其他类中也没有错的情况，书中的建议是**如果一个方法放在本类中，即不增加类间关系，也对本类不产生负面影响，那就放置在本类中**，不过我们组在工作中是通过**判读这个方法能力应该由谁来提供，在UML建模时就可以探讨出来放在那里合适**。

# 谨慎使用Serializable接口
Serializable接口主要用于远程方法调用中序列化一个对象进行网络传输，如果客户端的对象修改了对象属性的权限，譬如从public修改成private，服务器上没有做相应变化时会序列化失败。

# 思考与总结
我将迪米特法则简单描述成**是你的就是你的，不是你的你不要，就算是你的你也得珍惜**，这三句话分别对应了前三个原则，我在实际工作中CR或者编程没有想着说我要以六大原则去审视一下自己的代码，在工作中通过UML模型图以及控制好方法的权限没啥问题就通过了，所以等开闭原则学完之后以六大原则审视一下自己负责的项目，看是否有优化的地方，CR或者编写时是否可以找到一套方法论去更好的校验或评判。