---
layout:     post
title:      "设计模式之禅(三) -- 接口隔离原则"
subtitle:   ""
date:       2021-05-25
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - 设计模式
---

# 定义
程序间不依赖不需要的接口，且依赖最小的接口

# 两种接口
- 实例接口
实例接口书中的意思是java类中new关键字生成的实例，此java类就是实例类的接口（我个人理解是跟依赖倒置或者说面向接口编程一起考虑的，那样我们依赖的就是抽象接口，不用考虑实例类，但是不是所有都适用依赖倒置原则，所以实体类也可以借鉴接口隔离原则，思想是相通的）
- 类接口
类接口就是指Java中interface关键字定义的接口

# 两层含义
- 客户端不应该依赖它不需要的接口
这个很好理解，就是类不需要引用它不需要的接口
- 类间的依赖关系应该建立在最小的接口上
这个要求的是接口尽量细化，保证单一原则就好，接口纯洁

# 实例
书中是以一个星探发现漂亮女生为例子，介绍了两种PettyGril判断方式，应该做接口分离，首先是接口不分离的代码
```
public interface IPettyGirl {
    //好的面容
    void goodLooking();

    //好的身材
    void niceFigure();

    //好的气质
    void greatTemperament();
}

public class PettyGirl implements IPettyGirl {
    //美女名字
    private String name;

    public PettyGirl(String name) {
        this.name = name;
    }

    @Override
    public void goodLooking() {
        System.out.println(this.name + "_面容姣好");
    }

    @Override
    public void niceFigure() {
        System.out.println(this.name + "_身材好");
    }

    @Override
    public void greatTemperament() {
        System.out.println(this.name + "_气质好");
    }
}

public interface ISearcher {

    //搜索美女显示信息
    void show();
}

public abstract class AbstractSearcher implements ISearcher{

    protected IPettyGirl pettyGirl;

    public AbstractSearcher(IPettyGirl pettyGirl) {
        this.pettyGirl = pettyGirl;
    }

    public abstract void show();
}

public class Searcher extends AbstractSearcher {
    public Searcher(IPettyGirl pettyGirl) {
        super(pettyGirl);
    }

    @Override
    public void show() {
        //展示面容
        super.pettyGirl.goodLooking();
        //展示身材
        super.pettyGirl.niceFigure();
        //展示气质
        super.pettyGirl.greatTemperament();
    }

    public static void main(String[] args) {
        new Searcher(new PettyGirl("lisa")).show();
    }
}
```
因为现实中存在面容一般，身材一般但是气质非常好的女子，所以上面一个IPettyGirl需要拆分成两个接口IGoodBodyGirl和，拆分之后的代码如下
```
public interface IGoodBodyGirl {

    //好的面容
    void goodLooking();

    //好的身材
    void niceFigure();
}

public interface IGreatTemperamentGirl {
    //好的气质
    void greatTemperament();
}

public class PettyGirl implements IGoodBodyGirl, IGreatTemperamentGirl {
    //美女名字
    private String name;

    public PettyGirl(String name) {
        this.name = name;
    }

    @Override
    public void goodLooking() {
        System.out.println(this.name + "_面容姣好");
    }

    @Override
    public void niceFigure() {
        System.out.println(this.name + "_身材好");
    }

    @Override
    public void greatTemperament() {
        System.out.println(this.name + "_气质好");
    }
}

public abstract class AbstractSearcher implements ISearcher{

    protected IGoodBodyGirl goodBodyGirl;

    protected IGreatTemperamentGirl greatTemperamentGirl;

    public AbstractSearcher(IGoodBodyGirl goodBodyGirl) {
        this.goodBodyGirl = goodBodyGirl;
    }

    public AbstractSearcher(IGreatTemperamentGirl greatTemperamentGirl) {
        this.greatTemperamentGirl = greatTemperamentGirl;
    }
    
    public AbstractSearcher(IGoodBodyGirl goodBodyGirl, IGreatTemperamentGirl greatTemperamentGirl) {
        this.goodBodyGirl = goodBodyGirl;
        this.greatTemperamentGirl = greatTemperamentGirl;
    }

    public abstract void show();
}

public class Searcher extends AbstractSearcher {
    public Searcher(IGoodBodyGirl goodBodyGirl) {
        super(goodBodyGirl);
    }

    public Searcher(IGreatTemperamentGirl greatTemperamentGirl) {
        super(greatTemperamentGirl);
    }

    public Searcher(IGoodBodyGirl goodBodyGirl, IGreatTemperamentGirl greatTemperamentGirl) {
        super(goodBodyGirl, greatTemperamentGirl);
    }

    @Override
    public void show() {
        //展示面容
        super.goodBodyGirl.goodLooking();
        //展示身材
        super.goodBodyGirl.niceFigure();
        //展示气质
        super.greatTemperamentGirl.greatTemperament();
    }

    public static void main(String[] args) {
        new Searcher(new PettyGirl("lisa"), new PettyGirl("lucy")).show();
    }
}
```

# 接口规范约束
- 接口要尽量小，但是不能违反单一职责原则
- 接口要高内聚，就是提高自身的处理能力，减少对外的交互
- 定制服务，为每个个体提供优良的服务，考虑好性能等问题
- 接口设计是有限度的，设计粒度越小系统越灵活，但是会增加开发难度，降低可维护性

# 最佳实践
- 一个接口只服务于一个子模块或业务逻辑
- 通过业务逻辑压缩接口中的public方法，接口时长去回顾
- 已经被污染的接口尽量去修改
- 拒绝盲从，根据自身业务逻辑去设计

# 总结与思考
在实际的项目开发中写接口很容易为了方便写得臃肿，原因就是边界并不容易思考得清晰，导致的一个常见问题是两个接口可能有两个功能本质上是重复的，所以在编写接口时尽量小且不违反单一职责原则去做。当然也可以在一定项目阶段之后做一些项目的回顾并做好CodeReview工作可以尽量减少此类问题。