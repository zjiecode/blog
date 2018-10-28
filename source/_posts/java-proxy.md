---
layout: drafts
title: Java的静态代理和动态代理
tags:
  - Java
  - 代理
categories:
  - Java
date: 2018-10-26 12:56:50
---


# 什么是代理
什么是代理呢，其实很好理解，就是不直接访问目标，而是通过一个中间层来访问，就好像下面这样：
{% asset_img proxy.png 代理 %}

# Java的静态代理
举个例子，如果我们一些水果，比如：香蕉、苹果等，写成Java代码，大概是下面这个样子：
```
//Fruit.java
/**
 * 水果的接口
 */
public interface Fruit {
    /**
     * 获取水果的名字
     */
    public String getName();
}

//Apple.java
public class Apple implements Fruit {
    @Override
    public String getName() {
        return "苹果";
    }
}

//Banana.java
public class Banana implements Fruit {
    @Override
    public String getName() {
        return "香蕉";
    }
}

```
但是吃水果，你要削皮吧，你不能每个水果都写一个子类，类处理削皮这个事情吧。因此，我们可以做一个代理 ，吃苹果之前，先把它削皮。
就像下面这样，把原来的水果包一层：
```

//PeelFruitProxy.java
/**
 * 代理，让每个水果去皮
 */
public class PeelFruitProxy implements Fruit {
    private Fruit mFruit;

    public PeelFruit(Fruit fruit) {
        this.mFruit = fruit;
    }
    @Override
    public String getName() {
        System.out.println("proxt:" + proxy.getClass().getName());
        return "去皮的" + mFruit.getName();
    }
}
```
添加了测试类，测试类如下：
```
//Main.java
public class Main {
    public static void main(String[] args) {
        Apple apple=new Apple();//原始的苹果
        Banana banana=new Banana();//原始的香蕉

        PeelFruitProxy peelApple=new PeelFruitProxy(apple);//代理,添加去皮功能
        PeelFruitProxy peelBanana=new PeelFruitProxy(banana);//代理,添加去皮功能
        System.out.println(peelApple.getName());
        System.out.println(peelBanana.getName());
    }
}
```
{% asset_img proxy_1_run.png 代理 %}

以上就是Java的静态代理，简单点说，就是把原来的目标对象包一层，加入新东西再去调用目标本身。
但是如果只是这样的静态代理，一个接口，就需要一个代理，实现起来是不是很繁琐。

# Java的动态代理

在Java中，有一个干这个事情的类，叫做`Proxy`，可以直接使用反射方式，代理拦截。
先简单的介绍一下这个类，其实最常用的只有一个静态方法`Proxt.newProxyInstance()`,是这样的：
```
 public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
```

首先我们要实现InvocationHandler，实现其中的invoke方法，在调用目标对象的时候，会先调用到invoke方法，需要实现者在这个方法中，在主动调用被调用者方法。
```
//FruitInvocationHandler.java
/**
 * 调用方法拦截器
 */
public class FruitInvocationHandler implements InvocationHandler {
    private Fruit mFruit;

    public FruitInvocationHandler(Fruit fruit) {
        this.mFruit = fruit;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String result = (String) method.invoke(mFruit, args);//需要在这个方法里面，主动调用被代理的对象。
        return "去皮的" + result;
    }
}
```
运行一下：
```
//Main.Java
public class Main {
    public static void main(String[] args) {
        Apple apple = new Apple();
        Fruit proxyApple = (Fruit) Proxy.newProxyInstance(Fruit.class.getClassLoader(), new Class[]{Fruit.class}, new FruitInvocationHandler(apple));
        System.out.println(proxyApple.getClass().getName());
        System.out.println(proxyApple.getName());

        Banana banana = new Banana();
        Fruit proxyBanana = (Fruit) Proxy.newProxyInstance(Fruit.class.getClassLoader(), new Class[]{Fruit.class}, new FruitInvocationHandler(banana));
        System.out.println(proxyApple.getClass().getName());
        System.out.println(proxyBanana.getName());
    }
}
```

{% asset_img proxy_2_run.png 代理 %}

这个方法，就是生成一个上文中的`PeelFruitProxy`（当然，我们看到的他名字叫：com.sun.proxy.$Proxy0），动态的生成，避免每次都需要写，这个也是叫他动态代理的原因，因为我们可以在运行时代理任意类。
很多程序中的AOP就是这样实现的，但是我们发现一些特点，newProxyInstance()的第二个参数，是一个interfaces的列表，为啥要有这个这个列表呢？

因为我们动态生成的代理类，也需要实现接口，这样才方便向下转型，使用其中的方法，不然，生成的类，类名就是com.sun.proxy.$Proxy0这种，并且是在内存中，无法调用生成的方法。
** 所以，这种动态代理的方法，有一个致命的缺点，那就是被代理的类，必须要实现接口。**

# CGLib代理
> cglib is a powerful, high performance and quality Code Generation Library, It is used to extend JAVA classes and implements interfaces at runtime. 

另一个大名鼎鼎的Java代理实现，就是[CGLib(Code Generation Library)](https://github.com/cglib/cglib),一个基于ASM的代码生成框架，可以用他来动态生成类，然后实现对方法的拦截，就可以避开JDK的动态代理， 必须要目标类实现接口的问题了。
也就是说，可以用CGLib来生成上文中的`PeelFruitProxy`。

简单介绍一下怎么用，首先这个CGLib是一个三方的库，我们要把它依赖进来：
```
compile 'cglib:cglib:3.2.8'
```
最新版本可以在这里看(新版本)[https://github.com/cglib/cglib/releases]
然后我们来试一试，我们来实现一下上面的代理
```
//FruitMethodInterceptor.java
/**
 * CGLib代理的方法拦截器
 */
public class FruitMethodInterceptor implements MethodInterceptor{
    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        String result = (String) proxy.invokeSuper(obj, args);//主要，这里调用的是夫类，也就是说， 生成的类和原始类是继承关系
        return "去皮的"+result;
    }
}

//Main.java
public class Main {
    public static void main(String[] args) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(Apple.class);
        enhancer.setCallback(new FruitMethodInterceptor());
        Apple apple = (Apple) enhancer.create();
        System.out.println(apple.getClass().getName());
        System.out.println(apple.getName());
    }
}
```
运行效果如下：

{% asset_img proxy_3_run.png CGLib代理 %}
我们看到，实现了同样的功能，但是，Apple已经不是原来的Apple类了，变成了`com.zjiecode.learn.java.proxy.Apple$$EnhancerByCGLIB$$44ade224`，没错，我们正真使用的是这个类，而不是原来的`Apple`了，这个类继承自Apple，最后实现了对Apple类的代理。
这种方式，因为使用的是继承，所以，无需被代理的类实现接口。当然，他也可以通过接口来实现代理。

# 总结
- 第一种代理，就不说了，只适合单一的一个接口的代理，在编译时就决定好了。
- 第二、三种代理，都是动态时代理 ，但是我们看到也有差别：
    - JDK的动态代理 ，只能实现接口代理，并且是包装的被代理对象(类的实例),也就是说，在代理的过程中，有2个对象，一个代理对象，一个目标对象，目标对象被包装在代理对象里面。
    - CGLib的代理，是继承目标对象，生成了一个新的类，然后来实现代理，这样，在内存中就是有代理对象，没有目标对象了，使用的是直接集成的方式
- 生成代理类是在运行时，有别于`javapoet`在编译时生成类。

# 学习源码
纸上谈来终觉浅，绝知此事要躬行。
文中提到的源代码，你都可以参考：
[https://github.com/zjiecode/learn-java/tree/feature/java-proxy](https://github.com/zjiecode/learn-java/tree/feature/java-proxy)

# 参考资料
> https://blog.csdn.net/danchu/article/details/70238002
> https://blog.csdn.net/lovejj1994/article/details/78080124
> https://www.jianshu.com/p/9a61af393e41?from=timeline&isappinstalled=0
> https://github.com/cglib/cglib/wiki




