---
title: Maven引入外部jar包的方式
date: 2022-12-04 14:38
categories: 
- [开发技巧]
tags:
  - 日常开发
---
当我们遇到没有上传到maven仓库中但又需要导入本地项目使用的jar包时，可以参考maven引入外部jar包的方式，在本地开发环境直接引入相关的jar包进行使用，不过部署到生产项目时，不要忘记同时推到私仓里。
<!-- more -->
##### 一、dependency 本地jar包

```html
<dependency>
    <groupId>com.dataozi</groupId>  <!--自定义-->
    <artifactId>sdk</artifactId>    <!--自定义-->
    <version>1.0.0</version> <!--自定义-->
    <scope>system</scope> <!--system，类似provided，需要显式提供依赖的jar以后，Maven就不会在Repository中查找它-->
    <systemPath>${libpath}/lib/sdk-1.0.0.jar</systemPath> <!--项目根目录下的lib文件夹下-->
</dependency> 
```

##### 二、编译阶段指定外部lib

```html
<plugin>
     <artifactId>maven-compiler-plugin</artifactId>
     <version>2.3.2</version>
     <configuration>
     <source>1.8</source>
     <target>1.8</target>
     <encoding>UTF-8</encoding>
     <compilerArguments>
     <extdirs>lib</extdirs><!--指定外部lib-->
     </compilerArguments>
     </configuration>
</plugin>
```

##### 三、将外部jar打入本地maven仓库(推荐)

使用maven命令在指定maven仓库路径打好jar包

> ```sh
> mvn install:install-file -Dfile=${jar包路径} -DgroupId=${groupId} -DartifactId=${artifactId} -Dversion=${指定版本} -Dpackaging=${指定打包方式}
> ```

> ```sh
> mvn install:install-file -Dfile=test.jar -DgroupId=com.dataozi -DartifactId=sdk -Dversion=1.0.0 -Dpackaging=jar
> ```

在项目中直接引入jar包

```html
<dependency>
     <groupId>com.dataozi</groupId>
     <artifactId>sdk</artifactId>
     <version>1.0.0</version>
</dependency>
```
