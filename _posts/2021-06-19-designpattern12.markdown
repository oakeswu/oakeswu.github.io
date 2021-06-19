---
layout:     post
title:      "设计模式之禅(五) -- 开闭原则"
subtitle:   ""
date:       2021-06-19
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - 设计模式
---

# 定义
开闭原则的定义就是**一个软件实体如类、模块和函数应该对扩展开放，对修改关闭**

# 含义
- 在设计时尽量适应变化，已提高项目的稳定性和灵活性
- 尽量通过扩展软件实体行为实现变化，而不是通过修改已有的代码实现

# 实例
![](/img/doc/designPattern/12/12-1.png)
```
public interface IBook {

    String getName();

    int getPrice();

    String getAuthor();
}

public class NovelBook implements IBook {
    private String name;

    private int price;

    private String author;

    public NovelBook(String name, int price, String author) {
        this.name = name;
        this.price = price;
        this.author = author;
    }

    @Override
    public String getName() {
        return this.name;
    }

    @Override
    public int getPrice() {
        return this.price;
    }

    @Override
    public String getAuthor() {
        return this.author;
    }
}

public class BookStore {
    private final static ArrayList<IBook> bookList = new ArrayList<>();

    static {
        bookList.add(new NovelBook("天龙八部", 3200, "金庸"));
        bookList.add(new NovelBook("巴黎圣母院", 5600, "雨果"));
        bookList.add(new NovelBook("悲惨世界", 3500, "雨果"));
    }

    public static void main(String[] args) {
        NumberFormat format = NumberFormat.getCurrencyInstance();
        format.setMaximumFractionDigits(2);
        System.out.println("书店卖出去的书籍记录如下");
        for (IBook book : bookList) {
            System.out.println("书籍名称:" + book.getName() + "\t书籍作者:" +
                    book.getAuthor() + "\t书籍价格:" + book.getPrice() / 100 + "元");
        }
    }
}
```
我们以书店销售书为例，书籍目前有三个方法，现在书籍需要增加一个折扣方法，专门进行打折处理，如果直接在NovelBook的getPrice方法上修改，就会影响到老逻辑，另外如果折扣价是对一部分人开放，这种方式就不行。我们换另一种方式去做，如下所示：
![](/img/doc/designPattern/12/12-2.png)
```
public class OffNovelBook extends NovelBook {
    
    public OffNovelBook(String name, int price, String author) {
        super(name, price, author);
    }
    
    @Override
    public int getPrice() {
        int selfPrice = super.getPrice();
        int offPrice = 0;
        if (selfPrice > 4000) {
            offPrice = selfPrice * 90 / 100;
        } else {
            offPrice = selfPrice * 80 / 100;
        }
        return offPrice;
    }
}

public class BookStore {
    private final static ArrayList<IBook> bookList = new ArrayList<>();

    static {
        bookList.add(new OffNovelBook("天龙八部", 3200, "金庸"));
        bookList.add(new OffNovelBook("巴黎圣母院", 5600, "雨果"));
        bookList.add(new OffNovelBook("悲惨世界", 3500, "雨果"));
    }

    public static void main(String[] args) {
        NumberFormat format = NumberFormat.getCurrencyInstance();
        format.setMaximumFractionDigits(2);
        System.out.println("书店卖出去的书籍记录如下");
        for (IBook book : bookList) {
            System.out.println("书籍名称:" + book.getName() + "\t书籍作者:" +
                    book.getAuthor() + "\t书籍价格:" + book.getPrice() / 100 + "元");
        }
    }
}
```
我们通过增加一个子类完成了扩展，就就是对扩展开放，对修改关闭的一种体现。

# 使用必要性
- 开闭原则对测试的影响
- 开闭原则可以提高复用性
- 开闭原则可以提高可维护性
- 面向对象开发的要求

# 总结与思考
设计模式有六大原则（单一职责，里氏替换原则，依赖倒置，接口隔离，迪米特法则，开闭原则），我们在编程过程中怎么去判断所写代码是否符合六大原则呢？我想有一个简单的准则就是写的过程当中用开闭原则去思考就好了，开闭原则可以更多的理解为是思想，而另五大原则是实践这种思想的方法论。我们在编程或CR过程中用开闭原则去审视是否可以用五大原则优化代码，久而久之就会形成良好的代码风格。