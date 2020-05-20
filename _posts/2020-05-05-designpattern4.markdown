---
layout:     post
title:      "Builder模式"
subtitle:   ""
date:       2020-05-05
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - 设计模式
---

# 背景
因为部门中大多都需要使用到ES创建数据索引，ES可以使用Query DSL语言或者自带的java client，因为需要用到一些公司日志并且方便自定义功能，所以我们基于Query DSL开发了一个公共包，由于查询的传参众多，譬如should,must查询，sort排序，pageno等，所以会使用到builder模式

# 例子
```
public class People {
    private Long cardId ;
    private String name;
    private Integer age;
    private String address;
    private Boolean isMale;

    People(Long cardId, String name, Integer age, String address, Boolean isMale) {
        this.cardId = cardId;
        this.name = name;
        this.age = age;
        this.address = address;
        this.isMale = isMale;
    }

    People(Builder builder) {
        this.cardId = builder.cardId;
        this.name = builder.name;
        this.age = builder.age;
        this.address = builder.address;
        this.isMale = builder.isMale;
    }

    public static class Builder {
        private Long cardId ;
        private String name;
        private Integer age;
        private String address;
        private Boolean isMale;

        Builder() {}

        Builder(Long cardId) {
            this.cardId = cardId;
        }

        public Builder setCardId(Long cardId) {
             this.cardId = cardId;
             return this;
        }

        public Builder setName(String name) {
            this.name = name;
            return this;
        }

        public Builder setAge(Integer age) {
            this.age = age;
            return this;
        }


        public Builder setAddress(String address) {
            this.address = address;
            return this;
        }

        public Builder IsMale(Boolean isMale) {
            this.isMale = isMale;
            return this;
        }

        public People build() {
            return new People(this);
        }
    }

    @Override
    public String toString() {
        return "cardId:" + cardId +
                ", name:" + name +
                ",age:" + age +
                ",address:" + address +
                ",isMale:" +isMale;
    }
}
```
```
public class PeopleTest {

    public static void main(String[] args) {
        People people  = new People.Builder(1L).setName("test").setAge(18).setAddress("shanghai").build();
        System.out.println(people);
    }
}
```
我们可以看一下运行结果
```
Connected to the target VM, address: '127.0.0.1:53518', transport: 'socket'
cardId:1, name:test,age:18,address:shanghai,isMale:null
Disconnected from the target VM, address: '127.0.0.1:53518', transport: 'socket'

Process finished with exit code 0
```
# 总结
我们可以看到People里面目前有五个参数，当然项目很可能有更多参数，譬如工作，电话号码等，那样可以看到People类的构造函数会十分庞大，此时我们可以看到使用builder模式，可以通过拆分的方法让创建一个People对象变得简洁