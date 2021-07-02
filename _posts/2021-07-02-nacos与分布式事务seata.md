---
layout:     post
title:      nacos与分布式事务seata
subtitle:   最新版seata1.4.2 分布式事务框架与nacos整合的资料非常少，seata官网资料也不太健全，故写此文章
date:       2021-07-02
author:     zj
header-img: img/post-bg-e2e-ux.jpg
catalog: true
tags:
    - 实战经验
---

   

#### 一  前言

​    微服务普及的情况下，存在一个令人头疼的问题，那必然有分布式事务管理的一亩三分地，好在阿里巴巴提供分布式事务框架seata,可以很好的解决这个问题。当然没有分布式事务也可以采用其他的方式解决数据一致性的问题，例如常使用的方法有调用方重试请求（要求接口必须支持幂等）、或者被调用方额外提供结果查询接口以及回滚接口等，但是随着微服务越来越多补偿操作就会越多，喧宾夺主情况非常常见，所以拯救一下程序员吧。



#### 二  技术版本

| 框架         | 版本          | 说明           |
| ------------ | ------------- | -------------- |
| Spring Cloud | 2.2.5.RELEASE | 微服务架构     |
| Nacos        | 1.4.0         | 注册、配置中心 |
| Seata        | 1.4.2         | 分布式事务     |



#### 三  Nacos 安装

​     下载地址 https://github.com/alibaba/nacos/releases/tag/1.4.2 ，下载zip 包后解压到安装目录即可

![image-20210702161055785](https://github.com/zjhylove/zjhylove.github.io/tree/master/img/image-20210702161055785.png)

​    然后在mysql 中建立相关的表，建表sql 在安装目录 conf 目录中的【nacos-mysql.sql】和【schema.sql】文件，nacos 表结构建立完成以后需要修改conf 目录下的数据库链接

![image-20210702161542094](img/image-20210702161542094.png)

上图红色标记的为关键修改点注意填写正确，完成以后访问 http://locahost:8848 登录nacos 为seata 建立一个namespace，下面seata 的配置需要用到



#### 四  Seata安装

​    1）下载地址 https://github.com/seata/seata/releases/download/v1.4.2/seata-server-1.4.2.zip 

下载完成以后解压至安装目录即可。

![image-20210702161916388](https://github.com/zjhylove/zjhylove.github.io/tree/master/img/image-20210702161916388.png)

​    2）下载nacos 配置中心配置文件和推送脚本，https://github.com/seata/seata/tree/develop/script/config-center 

![image-20210702162506702](https://github.com/zjhylove/zjhylove.github.io/tree/master/img/image-20210702162506702.png)

上图中红色箭头根据自身要求修改，本人使用db 配置，修改完成以后执行脚本命令

```shell
sh ${SEATAPATH}/script/config-center/nacos/nacos-config.sh -h localhost -p 8848 -g SEATA_GROUP -t 5a3c7d6c-f497-4d68-a71a-2e5e3340b3ca -u username -w password

```

​     将config.txt 中的配置信息同步至nacos 中【SEATA_GROUP】为nacos 组 【5a3c7d6c-f497-4d68-a71a-2e5e3340b3ca】为namespace 其它参数不做过多解读，执行完脚本以后打开nacos管理的可以查看到同步的seata配置如下

![image-20210702163407806](https://github.com/zjhylove/zjhylove.github.io/tree/master/img/image-20210702163407806.png)

3）修改seata 注册配置文件【registry.conf】

![image-20210702162952711](C:\Users\zhengjun\AppData\Roaming\Typora\typora-user-images\image-20210702162952711.png)

4)  修改file.conf 文件

![image-20210702163232361](https://github.com/zjhylove/zjhylove.github.io/tree/master/img/image-20210702163232361.png)

4）建Seata 所需要的表sql 地址 https://github.com/seata/seata/blob/1.4.1/script/server/db/mysql.sql

5） 经过以上一系列骚操作以后启动seata 服务，默认启动端口为 8091，如果想要本地启动多个实例可以使用以下命令启动

```sh
sh seata-server.sh -p 18091
```

```bash
seata-server.bat -p 18091
```





#### 五  seata 客户端配置

1） seata默认使用AT 模式管理事务，那么需要在客户端数据库中建立 undo_log 表

​       脚本地址 https://github.com/seata/seata/blob/1.4.1/script/client/at/db/mysql.sql

2） maven 依赖添加

```xml
<dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>io.seata</groupId>
                    <artifactId>seata-all</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>io.seata</groupId>
                    <artifactId>seata-spring-boot-starter</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>io.seata</groupId>
            <artifactId>seata-all</artifactId>
            <version>1.4.2</version>
        </dependency>
        <dependency>
            <groupId>io.seata</groupId>
            <artifactId>seata-spring-boot-starter</artifactId>
            <version>1.4.2</version>
        </dependency>
```



3) yml 配置

   ![image-20210702164743706](https://github.com/zjhylove/zjhylove.github.io/tree/master/img/image-20210702164743706.png)

 例如我这里配置的 hrms_tx_group  那么nacos 配置的就是**service.vgroupMapping.hrms_tx_group=default**



4) 启动服务测试大功告成
