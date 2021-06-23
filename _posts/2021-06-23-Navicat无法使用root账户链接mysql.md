---
layout:     post
title:      Navicat 无法使用root 用户连接mysql
subtitle:   使用root账户链接mysql 错误码 ‘1045’
date:       2021-06-23
author:     zj
header-img: img/post-bg-debug.png
catalog: true
tags:
    - 实战经验
---

   

- mysql 执行命令

  ```sql
  GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'root-password' WITH GRANT OPTION;
  
  FLUSH PRIVILEGES;
  ```

​     第一条命令是指【授权所有全局权限给 root 用户，root 用户密码为 root-password，主机为所有 ip 地址，进行授权】。这样一来，就可以在任何地方以 root 用户连接数据库。第二条命令是刷新权限，让刚刚修改的即时生效。