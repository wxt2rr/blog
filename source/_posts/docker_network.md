---
title: 相同宿主机下的docker容器之间相互访问
date: 2022-11-04 14:38
categories: 
- [开发技巧]
tags:
  - 日常开发
---
因为我目前只有一台公网环境的ECS，所以一些公网访问的服务都以docker容器方式部署在该ECS上，目前通过nginx代理3个服务（博客、bitwarden、导航页），那么就需要多个docker容器之间进行网络通信，整理了3种实现方式。
<!--more-->
1.通过宿主机，宿主机相当于一个交换机，docker容器之间通过宿主机转接进行网络通信，不需要知道每个容器的内网ip是多少，只要知道启动容器时暴漏的端口就可访问【推荐】

~~~sh
ip addr show docker0
docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:c2:a2:58:71 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:c2ff:fea2:5871/64 scope link 
       valid_lft forever preferred_lft forever
       
#172.17.0.1 为宿主机ip
#假如a容器的端口为1000:80，b容器的端口为1001:80，则a容器内可以通过172.17.0.1:1001访问到b容器
~~~

2.通过docker容器的内网ip，每次创建新的docker容器都需要查看容器的内网ip，非常繁琐【不推荐】

~~~sh
#进入docker容器，查看当前容器内网ip
ifconfig
#假如a容器的端口为1000:80，ip为172.17.0.2，b容器的端口为1001:80，ip为ip为172.17.0.3，则a容器内可以通过172.17.0.3:80访问到b容器
~~~

3.通过bridge网络

~~~sh
#创建bridge网络
docker network create testnet
#看到网桥下的容器ip信息
docker network inspect dyl_testnet
#启动容器时链接到网桥
docker run --name xxx --netwrok testnet --network-aloas nginx
#a容器启动时为a，1000:80;b容器启动时为b,1001:80，则a容器可以通过ping b:8081访问
~~~

