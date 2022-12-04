---
title: 数据库连接参数allowMultiQueries的作用
date: 2022-12-04 14:38
categories: 
- [日常踩坑]
tags:
- Mysql
- 数据库
---
如上，今天有个同事写了一条sql，先不管sql的业务逻辑和可用性，在本地执行没有问题，部署到QA环境报错，原因是本地数据库连接时设置了allowMultiQueries=true，但是QA环境没有设置，那默认是false。记录一下 allowMultiQueries 参数的作用
<!--more-->
~~~ mysql
 <delete id="delete">
        <foreach collection="list" item="item" index="index" separator=";" >
            delete from xxxx_table
            where gid = #{item.gId} and pid = #{item.pId}
        </foreach>
    </delete>
~~~

如上，今天有个同事写了一条sql，先不管sql的业务逻辑和可用性，在本地执行没有问题，部署到QA环境报错，大概意思就是sql在where gid = #{item.gId} 执行语法有问题。

##### 问题原因：

本地数据库连接时设置了allowMultiQueries=true，但是QA环境没有设置，那默认是false。

##### allowMultiQueries=true的作用

* 可以在sql语句后携带分号，实现多语句执行。

* 可以执行批处理，同时发出多个SQL语句。

将QA环境的数据库连接加上这个属性就可以了。
