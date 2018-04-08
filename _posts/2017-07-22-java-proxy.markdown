---
layout:     post
title:      "动态代理&静态代理"
subtitle:   "学习总结"
date:       2017-07-22 12:00:00
author:     "溜大虾"
header-img: "img/bg-proxy.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Java
    

---

代理，就是对类做一个包装，通过使用这个包装好的类来使用本来要用的类，这样可以额外做一些操作。
代理分为静态代理和动态代理。

### 静态代理

 	实现起来很简单，一个接口用来提供要实现的方法，一个被代理类和一个代理类都要实现这个接口。同时，被代理类实现接口的方法后要做自己要做的事，而代理类则要在内部创建一个被代理类的实例，然后在实现的接口的方法中调用被代理类的那个方法，同时做一些自己的修饰。

### 动态代理

不同于静态代理，动态代理的需求是当前有很多很多被代理类，要采用类似的代理方式。这个时候就要采用动态代理了。同样要定义接口及被代理类，区别在于代理类是通过反射动态生成的。具体：实现InvocationHandler这个接口，持有一个被代理类的对象object，然后实现：
/**

     * 返回代理对象
     * @return
     */
    public Object newProxyInstance() {
        return Proxy.newProxyInstance(proxied.getClass().getClassLoader(), proxied.getClass().getInterfaces(), this);
    }

这个方法是用来获得代理的对象。然后最重要的是要实现：
/**
     * 
     * @param proxy 代理对象
     * @param method 代理方法
     * @param args 方法参数
     * @return
     * @throws Throwable
     */
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable { 
        //将代理对象生成字节码到F盘上，方便反编译出java文件查看，实际动态代理是不需要自己生成的
        addClassToDisk(proxy.getClass().getName(), ProxyClassImpl.class,"F:/$Proxy0.class");
        System.out.println("method:"+method.getName());  
        System.out.println("args:"+args[0].getClass().getName());  
        System.out.println("Before invoke method...");  
        Object object=method.invoke(proxied, args);
        System.out.println("After invoke method...");  
        return object;  

这个方法是用来调用构造器的。具体使用时，我们不需要使用invoke方法，只是：
1.获得被代理类的一个对象，传给handler来获得代理类
2.调用handler的newInstance方法获取一个代理类对象
3.这个时候已经生成好了代理类，我们可以直接用这个代理对象