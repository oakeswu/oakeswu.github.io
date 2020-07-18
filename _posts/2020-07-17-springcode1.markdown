---
layout:     post
title:      "Spring源码(一) -- 一个测试用例"
subtitle:   ""
date:       2020-07-17
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - Spring源码
---

**本文及后续学习主要参考Spring5.0源码，《Spring源码深度解析》**
# 背景
Spring在Java开发中是大家都会使用的开源框架，以我的使用经验来说Spring提供了一个容器管理，注解，常用组件的集成，事务等，但是今天有人问了我一个问题BeanFactory和FactoryBean的区别是什么，只能简单的回答出BeanFactory的一些使用上的理解，并不能很好的回答，所以也反思了下应该很早就学习Spring源码和ES源码，不过任何时候出发都不晚，所以开始啃Spring源码。
# 准备工作
我们可以从github上面拉取到Spring源码，然后本地执行gradle进行项目的编译，之前在公司一直编译失败，但是在家中就能编译成功，所以编译过程中出错也可能是网络原因,我就先按照源码解析一书中提到的开场一样先建立了一个xml文件(beanFactoryTest.xml)，一个测试类(MyTestBean)，以及一个测试方法类(BeanFactoryTest)，如图所示：
![](/img/doc/springcode/spring1one.png)xml配置bean文件是Spring最原始的方式，当然我们现在使用的SpringBoot已经通过注解简化了xml这种方式，不过SpringBoot也是基于Spring发展而来，所以我们还是先从最原始的学习。
# 测试用例
上面说到了我们创建的测试用例，我们这次就来分别看下这三个文件
- MyTestBean
```
public class MyTestBean {
    private String testStr = "testStr";

    public String getTestStr() {
        return testStr;
    }

    public void setTestStr(String testStr) {
        this.testStr = testStr;
    }
}
```
- beanFactoryTest.xml
![](/img/doc/springcode/spring1two.png)
因为xml在jeklly不能正常显示，所以我们这边用图片展示，其实重要的就是中间的配置一行

- BeanFactoryTest
```
public class BeanFactoryTest {
    @Test
    public void testSimpleLoad(){
        BeanFactory beanFactory = new XmlBeanFactory(new ClassPathResource("beanFactoryTest.xml"));
        MyTestBean bean = (MyTestBean)beanFactory.getBean("myTestBean");
        Assert.assertEquals("testStr", bean.getTestStr());
    }
}
```
然后我们执行以下测试方法，看下执行的结果是啥？
![](/img/doc/springcode/spring1three.png)
我们可以看到测试代码成功根据配置文件中id获取到了对应的MyTestBean对象。这其实就是一个我们常说的简单的Ioc案例。以前我们获取MyTestBean对象都是通过new一个对象出来，但是案例中Spring通过xml配置获取到了MyTestBean对象，对象的控制权由用户反转给了Spring。下面就是我们需要去理解这个测试代码的过程从而进入Spring的源码世界。
测试代码执行中可能会报**找不到org.apache.commons.logging.Log错误** 解决办法就是**在build.gradle文件configure(allprojects)节点的dependencies中compile("org.slf4j:jcl-over-slf4j:${log4jVersion}")就可以解决了**