---
layout: post
title: 设计模式 -- 工厂模式
date: 2020-07-02
categories: 技术
tags: Java设计模式 
---

### 什么是工厂模式 ？

工厂模式（Factory Pattern）的意义就跟它的名字一样，在面向对象程序设计中，工厂通常是一个用来创建其他对象的对象。工厂模式根据不同的参数来实现不同的分配方案和创建对象。

在工厂模式中，我们在创建对象时不会对客户端暴露创建逻辑，并且是通过使用一个`共同的接口`来指向新创建的对象。例如用工厂来创建 `人` 这个对象，如果我们需要一个男人对象，工厂就会为我们创建一个男人；如果我们需要一个女人，工厂就会为我们生产一个女人。

工厂模式通常分为：

- 普通工厂模式
- 多个工厂方法模式
- 静态工厂方法模式

### 普通工厂模式

刚刚我们说到，用工厂模式来创建人。先创建一个男人，他每天都“吃饭、睡觉、打豆豆”，然后我们再创建一个女人，她每天也“吃饭、睡觉、打豆豆”。

我们以普通工厂模式为例，新建一个`FactoryTest.java`。

![wm.png](https://i.loli.net/2020/07/02/o9batuUTspv4QNG.png)

代码如下：

~~~java
// 二者共同的接口
interface Human{
    public void eat();
    public void sleep();
    public void beat();
}

// 创建实现类 Male
class Male implements Human{
    public void eat(){
        System.out.println("Male can eat.");
    }
    public void sleep(){
        System.out.println("Male can sleep.");
    }
    public void beat(){
        System.out.println("Male can beat.");
    }
}
//创建实现类 Female
class Female implements Human{
    public void eat(){
        System.out.println("Female can eat.");
    }
    public void sleep(){
        System.out.println("Female can sleep.");
    }
    public void beat(){
        System.out.println("Female can beat.");
    }
}

// 创建普通工厂类
class HumanFactory{
    public Human createHuman(String gender){
        if( gender.equals("male") ){
           return new Male();
        }else if( gender.equals("female")){
           return new Female();
        }else {
            System.out.println("请输入正确的类型！");
            return null;
        }
    }
}

// 工厂测试类
public class FactoryTest {
    public static void main(String[] args){
        HumanFactory factory = new HumanFactory();
        Human male = factory.createHuman("male");
        male.eat();
        male.sleep();
        male.beat();
    }
}
~~~

### 多个工厂方法模式

普通工厂模式就是上面那样子了，那么多个工厂方法模式又有什么不同呢？在普通工厂方法模式中，如果传递的字符串出错，则不能正确创建对象。多个工厂方法模式是提供多个工厂方法，分别创建对象。

部分示例代码，其他与上面普通工厂模式示例代码一样：

~~~java
// 多个工厂方法
public class HumanFactory{
    public Male createMale() {
        return new Male();
    }
    public Female createFemale() {
        return new Female();
    }
}

// 工厂测试类
public class FactoryTest {
    public static void main(String[] args){
        HumanFactory factory = new HumanFactory();
        Human male = factory.createMale();
        male.eat();
        male.sleep();
        male.beat();
    }
}
~~~

### 静态工厂方法模式

将上面的多个工厂方法模式里的方法置为静态的，不需要创建实例，直接调用即可。

部分示例代码：

~~~java
// 多个工厂方法
public class HumanFactory{
    public static Male createMale() {
        return new Male();
    }
    public static Female createFemale() {
        return new Female();
    }
}

// 工厂测试类
public class FactoryTest {
    public static void main(String[] args){
        Human male = HumanFactory.createMale();
        male.eat();
        male.sleep();
        male.beat();
    }
}
~~~

>  总结：凡是出现了大量的产品需要创建，并且具有共同的接口时，可以通过工厂方法模式进行创建。在以上的三种模式中，第一种如果传入的字符串有误，不能正确创建对象，第三种相对于第二种，不需要实例化工厂类，所以，大多数情况下，我们会选用第三种——静态工厂方法模式。