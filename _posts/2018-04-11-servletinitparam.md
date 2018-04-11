---
layout:     post
title:      "web.xml中Servlet参数初始化"
subtitle:   "学习总结"
date:       2018-04-10 12:00:00
author:     "溜大虾"
header-img: "img/20180411/top.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Servlet
    

---

- context参数初始化

```
<context-param>
<param-name>param1</param-name>
<param-value>value1</param-value>
</context-param>
<context-param>
<param-name>param2</param-name>
<param-value>value2</param-value>
</context-param>
```

- servlet参数初始化

```
<servlet>
 <!--name可以是任意的，但一般是类名-->
<servlet-name>MyServlet</servlet>
 <!--class用于指定你的servlet存放的路径-->
<servlet-class>com.web.MyServlet</servlet-class>
<!--设置各自servlet的初始化参数-->
 <!--参数1-->
<init-param>
<param-name>driver</param-name>
<param-value>com.mysql.jdbc.Driver</param-value>
</init-param>
 <!--参数2-->
<init-param>
<param-name>url</param-name>
<param-value>jdbc:mysql://localhost:3306/mysql</param-value>
</init-param>

</servlet>

```