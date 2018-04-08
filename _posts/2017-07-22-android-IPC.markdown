---
layout:     post
title:      "Android中的IPC机制"
subtitle:   "学习总结"
date:       2017-07-19 12:00:00
author:     "溜大虾"
header-img: "img/bg-ipc.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Android
    

---

> Android IPC机制，就是Andorid中的进程间通信机制，基于Linux，用了自己独特的Binder  

---





 



## 一些基本概念

### 什么是IPC?
IPC即Inter Process Communication  ，进程间通信机制的缩写。每一个app默认就是一个进程（除非手动指定多开进程），多个进程之间进行通信就是进程间通信

### 什么情况下要用到IPC ?
当我们的app有多个进程，或者多个app进程之间需要通信时，就要用到

### 多进程会有什么问题
常见的问题：静态成员、单例模式失效，线程同步机制失效，Shared{reference可靠性下降，Application多次创建

### 有哪些进程间通信方式？

操作系统常用的进程间通信方式有很多，如信号量、共享文件等。在Android中有AIDL  Socket  ContentProvider等，而这些机制的底层都是用了BIinder机制。所以所，Android的IPC机制，就是Binder机制。  



## Binder机制

### 为什么要用Binder机制？

[可以参考这个回答](https://www.zhihu.com/question/39440766)



Android系统基于Linux，Binder机制是相较于Linux的管道、消息队列、共享内存、套接字、信号量、信号等方式之下，相对稳定、高性能、安全、架构清晰的方式。



### Binder的结构

Binder结构是一个C/S架构，两个要通信的进程，分别作为客户端和服务器端。其中客户端不能直接对服务器端做任何操作，需要服务器提供一个代理对象，客户端对服务器的操作要先传给这个代理对象，再由代理对象传给服务器，返回结果之后也是先返回给代理，然后由代理对象返回给客户度端。在这期间，需要有BinderDriver来做枢纽，他作为一个桥梁、中介，一方面对接服务器的代理对象，一方面对接实际的服务端。

## AIDL

AIDL是一种常见的IPC方式，可以自己写个小Demo学习一下。我在学习的过程中遇到的坑不少，比如需要辅以.aidl文件做通信，要注意在客户端和服务器端aidl文件的包名要相同。
