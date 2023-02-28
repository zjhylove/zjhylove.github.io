---
layout:     post
title:      ShardingJdbc
subtitle:   shardingJdbc与springboot
date:       2023-02-27
author:     zj
header-img: img/post-bg-keybord.jpg
catalog: true
tags:
    - 实战经验

---



#### 使用场景

- 读写分离

  1、系统写入数据不多但是存在大量的读取数据功能。

  2、读写分离并不取决于数据量还是取决于并发量，访问用户多才需要类似的功能。

  3、读写分离其实是个比较低端的处理读取并发量的操作，因为还是有对数据库的访问操作的，但是读写分离相对于其它处理方式而言的好处在于时效性比较高和对系统要求比较低。

  4、读写分离在效率上是低于页面静态化和缓存服务的，但是好处是不用改动系统代码，因为都是连接数据库。

  5、数据量大的情况下使用的技术不是读写分离，是分表和分库，或者使用分布式存储引擎，读写分离不能解决数据量大的问题。

  6、系统写入操作并发量大不适合使用读写分离，至于需要什么技术看具体业务需求，而且大量写入操作本身就是个难以处理的大数据问题，但是读写分离从一定程度上减轻写入操作的负担。

- 分库分表

  随着业务数据的增加，原有的数据库性能瓶颈凸显，以此就需要对数据库进行分库分表操作

  **IO瓶颈**

  - 第一种：磁盘读IO瓶颈，热点数据太多，数据库缓存放不下，每次查询时会产生大量的IO，降低查询速度。这种情况适合采用分库和垂直分表。
  - 第二种：网络IO瓶颈，请求的数据太多，网络带宽不够。这种情况适合采用分库。

  **CPU瓶颈**

  - 第一种：SQL问题，如SQL中包含join，group by，order by，非索引字段条件查询等，增加CPU运算的操作。这种情况适合采用SQL优化，建立合适的索引，或者把一些SQL操作移到在业务层中台代码中去做业务计算。
  - 第二种：单表数据量太大，查询时扫描的行太多，SQL效率低，CPU率先出现瓶颈这种情况适合采用水平分表

  **水平分库/表与垂直分库/表**

  - 水平分表是指，以字段为依据，按照一定策略（hash、range等），将一个表中的数据拆分到多个表中，水平分表适用的场景是，系统绝对并发量并没有上来，只是单表的数据量太多，影响了SQL效率，加重了CPU负担，以至于成为瓶颈。
  - 垂直分表是指，以字段为依据，按照字段的活跃性，将表中字段拆到不同的表（主表和扩展表）中，垂直分表适用的场景是，系统绝对并发量并没有上来，表的记录并不多，但是字段多，并且热点数据和非热点数据在一起，单行数据所需的存储空间较大。以至于数据库缓存的数据行减少，查询时会去读磁盘数据产生大量的随机读IO，产生IO瓶颈。
  - 水平分库是指，以字段为依据，按照一定策略（hash、range等），将一个库中的数据拆分到多个库中，水平分库适用的场景是，系统绝对并发量上来了，分表难以根本上解决问题，并且还没有明显的业务归属来垂直分库
  - 垂直分库是指，以表为依据，按照业务归属不同，将不同的表拆分到不同的库中，垂直分库适用的场景是，系统绝对并发量上来了，并且可以抽象出单独的业务模块。

#### POM依赖

- 主要集成框架为springboot、mybatis、druid数据源

```xml
        <!-- springboot-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <!-- 日志依赖-->
        <dependency>
            <groupId>net.logstash.logback</groupId>
            <artifactId>logstash-logback-encoder</artifactId>
        </dependency>
        <!-- 持久化框架-->
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>${spring.mybatis.version}</version>
        </dependency>
        <!-- 数据源连接池-->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid</artifactId>
        </dependency>
        <!-- mysql 驱动 -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
        <!-- 分表分库框架-->
        <dependency>
            <groupId>org.apache.shardingsphere</groupId>
            <artifactId>shardingsphere-jdbc-core-spring-boot-starter</artifactId>
            <version>${shardingsphere.version}</version>
            <exclusions>
                <exclusion>
                    <artifactId>bcprov-jdk15on</artifactId>
                    <groupId>org.bouncycastle</groupId>
                </exclusion>
                <exclusion>
                    <artifactId>commons-lang</artifactId>
                    <groupId>commons-lang</groupId>
                </exclusion>
                <exclusion>
                    <artifactId>commons-logging</artifactId>
                    <groupId>commons-logging</groupId>
                </exclusion>
                <exclusion>
                    <artifactId>error_prone_annotations</artifactId>
                    <groupId>com.google.errorprone</groupId>
                </exclusion>
                <exclusion>
                    <artifactId>protobuf-java</artifactId>
                    <groupId>com.google.protobuf</groupId>
                </exclusion>
                <exclusion>
                    <artifactId>quatz</artifactId>
                    <groupId>com.google.protobuf</groupId>
                </exclusion>
            </exclusions>
        </dependency>
        <!-- 单元测试-->
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
```



#### 读写分离

- 条件

  3306端口的读库，3307端口写库,（3306和3307进行mysql底层支持的数据同步）在读库中新建student表，并且添加对student表的新增和查询方法

- 配置

  ```yaml
  # 日志打印级别
  logging:
    level:
      root: info
  # mybatis
  mybatis:
    mapper-locations: classpath:mapper/*Mapper.xml
  spring:
    shardingsphere:
      # 运行模式类型。可选配置：Memory、Standalone、Cluster
      mode:
        type: Memory
      database:
        name: hrms-test
      props:
        sql-show: true
      datasource:
        names: hrms-test-read,hrms-test-write
        # 读库
        hrms-test-read:
          url: jdbc:mysql://127.0.0.1:3306/hrms_test?useUnicode=true&serverTimezone=Asia/Shanghai&characterEncoding=UTF-8&useSSL=false&connectTimeout=30000&socketTimeout=60000
          username: dml
          password: T76yP8jpJGLW9BrBEJd6fYERO9Zrx6gPb534DWAMhLnivphIDsgLOg5bmep6SaWYNLngnDy154bLKLlzv+NM4A==
          driver-class-name: com.mysql.cj.jdbc.Driver
          type: com.alibaba.druid.pool.DruidDataSource
          initialSize: 5
          minIdle: 10
          maxActive: 50
          maxWait: 60000
          timeBetweenEvictionRunsMillis: 60000
          minEvictableIdleTimeMillis: 300000
          validationQuery: SELECT 'x'
          testWhileIdle: true
          testOnBorrow: false
          testOnReturn: false
          poolPreparedStatements: false
          maxPoolPreparedStatementPerConnectionSize: 20
          removeAbandoned: true
          removeAbandonedTimeout: 1800
          logAbandoned: true
          filters: config,stat,wall
          publickey: MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBAJlMoFlQNur/osTA89hkHiw1c8rIES/Q+u0PbAZGZWCAS9Im5RdqXDd2YiIQROOmryiEXwlYclZp6McDrVjGLhsCAwEAAQ==
          connectionProperties: config.decrypt=true;config.decrypt.key=${spring.shardingsphere.datasource.hrms-test-read.publickey};druid.stat.mergeSql=true;druid.stat.slowSqlMillis=10000
        # 写库
        hrms-test-write:
          url: jdbc:mysql://127.0.0.1:3307/hrms_test?useUnicode=true&serverTimezone=Asia/Shanghai&characterEncoding=UTF-8&useSSL=false&connectTimeout=30000&socketTimeout=60000
          username: dml
          password: T76yP8jpJGLW9BrBEJd6fYERO9Zrx6gPb534DWAMhLnivphIDsgLOg5bmep6SaWYNLngnDy154bLKLlzv+NM4A==
          driver-class-name: com.mysql.cj.jdbc.Driver
          type: com.alibaba.druid.pool.DruidDataSource
          initialSize: 5
          minIdle: 10
          maxActive: 50
          maxWait: 60000
          timeBetweenEvictionRunsMillis: 60000
          minEvictableIdleTimeMillis: 300000
          validationQuery: SELECT 'x'
          testWhileIdle: true
          testOnBorrow: false
          testOnReturn: false
          poolPreparedStatements: false
          maxPoolPreparedStatementPerConnectionSize: 20
          removeAbandoned: true
          removeAbandonedTimeout: 1800
          logAbandoned: true
          filters: config,stat,wall
          publickey: MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBAJlMoFlQNur/osTA89hkHiw1c8rIES/Q+u0PbAZGZWCAS9Im5RdqXDd2YiIQROOmryiEXwlYclZp6McDrVjGLhsCAwEAAQ==
          connectionProperties: config.decrypt=true;config.decrypt.key=${spring.shardingsphere.datasource.hrms-test-write.publickey};druid.stat.mergeSql=true;druid.stat.slowSqlMillis=10000
      rules:
        # 读写分离配置  org.apache.shardingsphere.readwritesplitting.spring.boot.ReadwriteSplittingRuleSpringbootConfiguration
        readwrite-splitting:
          dataSources:
            #读写分离逻辑数据源名称
            hrms-test:
              #读写分离类型（静态读写分离）动态读写分离参考 org.apache.shardingsphere.readwritesplitting.strategy.ReadwriteSplittingStrategyFactory
              type: Static
              props:
                # 读数据源名称 只支持一个写库
                write-data-source-name: hrms-test-write
                # 写数据源名称 多个以逗号分隔
                read-data-source-names: hrms-test-read
              # 负载均衡名称
              load-balancer-name: alg_round
          load-balancers:
            # 负载均衡算法名称 org.apache.shardingsphere.readwritesplitting.spi.ReadQueryLoadBalanceAlgorithm
            alg_round:
              type: RANDOM
              props:
                # 根据算分所需参数进行添加属性
                hrms-test-read: 1.0
  ```

  

- 测试sql

  ```mysql
  CREATE TABLE `student` (
    `id` varchar(64) NOT NULL,
    `name` varchar(32) NOT NULL COMMENT '姓名',
    `age` int(3) NOT NULL COMMENT '年龄',
    `gender` int(1) NOT NULL COMMENT '性别 1：男  0：女',
    `school_name` varchar(32) NOT NULL COMMENT '学校名称',
    `class_name` varchar(32) NOT NULL COMMENT '班级名称',
    PRIMARY KEY (`id`)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
  ```

- 测试方法

  ```java
  import com.hrms.demo.po.StudentPO;
  import org.apache.ibatis.annotations.Mapper;
  import org.apache.ibatis.annotations.Param;
  
  /**
   * Student持久化服务
   *
   * @author zhengjun
   */
  @Mapper
  public interface StudentMapper {
  
      /**
       * 新增
       *
       * @param po 学生信息
       */
      void insert(StudentPO po);
  
  
      /**
       * 通过id 查询学生信息
       *
       * @param id 唯一id
       * @return 人信息
       */
      StudentPO selectById(@Param("id") String id);
  }
  ```

- 测试结果

  ![image-20230227163850384](https://gitee.com/zjhy/PicGo/raw/master/img/image-20230227163850384.png)

  ![image-20230227164024525](https://gitee.com/zjhy/PicGo/raw/master/img/image-20230227164024525.png)

- 由于读写库存在一定延迟，当插入数据库后立刻需要进行读取场景时就需要从写库中直接读取，不通过读库读取

  - 方法添加@Transactional注解

    ![image-20230227171242831](https://gitee.com/zjhy/PicGo/raw/master/img/image-20230227171242831.png)

  - 使用api  HintManager

    ![image-20230227171820522](https://gitee.com/zjhy/PicGo/raw/master/img/image-20230227171820522.png)

- 负载均衡算法可选

  | 算法类型                  | 算法说明     |
  | ------------------------- | ------------ |
  | ROUND_ROBIN               | 轮询         |
  | RANDOM                    | 随机         |
  | WEIGHT                    | 权重         |
  | FIXED_PRIMARY             | 固定主库     |
  | FIXED_REPLICA_RANDOM      | 固定从库随机 |
  | FIXED_REPLICA_ROUND_ROBIN | 固定从库轮询 |
  | FIXED_REPLICA_WEIGHT      | 固定从库权重 |
  | TRANSACTION_RANDOM        | 事务随机     |
  | TRANSACTION_ROUND_ROBIN   | 事务轮询     |
  | TRANSACTION_WEIGHT        | 事务权重     |
  
  除去上述均衡算法外，可以自己实现`org.apache.shardingsphere.readwritesplitting.spi.ReadQueryLoadBalanceAlgorithm`接口自定义适合自身业务的算法，自定义算法需要通过spi 加载

#### 分库分表

- 条件

  两个数据库ds_0、ds_1,两张结构相同的表 person_0、person_1，并新增逻辑表的新增和查询方法

- 配置

  ```yaml
  # 日志打印级别
  logging:
    level:
      root: info
  # mybatis
  mybatis:
    mapper-locations: classpath:mapper/*Mapper.xml
  spring:
    shardingsphere:
      # 运行模式类型。可选配置：Memory、Standalone、Cluster
      mode:
        type: Memory
      database:
        name: hrms-test
      props:
        sql-show: true
      datasource:
        names: ds_0,ds_1
        # 数据库0
        ds_0:
          url: jdbc:mysql://127.0.0.1:3306/hrms_test?useUnicode=true&serverTimezone=Asia/Shanghai&characterEncoding=UTF-8&useSSL=false&connectTimeout=30000&socketTimeout=60000
          username: dml
          password: T76yP8jpJGLW9BrBEJd6fYERO9Zrx6gPb534DWAMhLnivphIDsgLOg5bmep6SaWYNLngnDy154bLKLlzv+NM4A==
          driver-class-name: com.mysql.cj.jdbc.Driver
          type: com.alibaba.druid.pool.DruidDataSource
          initialSize: 5
          minIdle: 10
          maxActive: 50
          maxWait: 60000
          timeBetweenEvictionRunsMillis: 60000
          minEvictableIdleTimeMillis: 300000
          validationQuery: SELECT 'x'
          testWhileIdle: true
          testOnBorrow: false
          testOnReturn: false
          poolPreparedStatements: false
          maxPoolPreparedStatementPerConnectionSize: 20
          removeAbandoned: true
          removeAbandonedTimeout: 1800
          logAbandoned: true
          filters: config,stat,wall
          publickey: MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBAJlMoFlQNur/osTA89hkHiw1c8rIES/Q+u0PbAZGZWCAS9Im5RdqXDd2YiIQROOmryiEXwlYclZp6McDrVjGLhsCAwEAAQ==
          connectionProperties: config.decrypt=true;config.decrypt.key=${spring.shardingsphere.datasource.ds_0.publickey};druid.stat.mergeSql=true;druid.stat.slowSqlMillis=10000
        # 数据库1
        ds_1:
          url: jdbc:mysql://127.0.0.1:3306/hrms_test?useUnicode=true&serverTimezone=Asia/Shanghai&characterEncoding=UTF-8&useSSL=false&connectTimeout=30000&socketTimeout=60000
          username: dml
          password: T76yP8jpJGLW9BrBEJd6fYERO9Zrx6gPb534DWAMhLnivphIDsgLOg5bmep6SaWYNLngnDy154bLKLlzv+NM4A==
          driver-class-name: com.mysql.cj.jdbc.Driver
          type: com.alibaba.druid.pool.DruidDataSource
          initialSize: 5
          minIdle: 10
          maxActive: 50
          maxWait: 60000
          timeBetweenEvictionRunsMillis: 60000
          minEvictableIdleTimeMillis: 300000
          validationQuery: SELECT 'x'
          testWhileIdle: true
          testOnBorrow: false
          testOnReturn: false
          poolPreparedStatements: false
          maxPoolPreparedStatementPerConnectionSize: 20
          removeAbandoned: true
          removeAbandonedTimeout: 1800
          logAbandoned: true
          filters: config,stat,wall
          publickey: MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBAJlMoFlQNur/osTA89hkHiw1c8rIES/Q+u0PbAZGZWCAS9Im5RdqXDd2YiIQROOmryiEXwlYclZp6McDrVjGLhsCAwEAAQ==
          connectionProperties: config.decrypt=true;config.decrypt.key=${spring.shardingsphere.datasource.ds_1.publickey};druid.stat.mergeSql=true;druid.stat.slowSqlMillis=10000
      rules:
        sharding:
          tables:
            person:
              actual-data-nodes: ds_$->{0..1}.person_$->{0..1}
              # 分库策略可以使用内置算法或自定义
              database-strategy:
                standard:
                  sharding-column: id
                  sharding-algorithm-name: database-hashmod
              # 分表策略可以使用内置算法或自定义
              table-strategy:
                standard:
                  sharding-column: id
                  sharding-algorithm-name: table-hashmod
          # 分表
          sharding-algorithms:
            table-hashmod:
              type: HASH_MOD
              props:
                sharding-count: 2
            database-hashmod:
              type: HASH_MOD
              props:
                sharding-count: 2
  ```

- 测试sql

  ```mysql
  CREATE TABLE `person_0` (
    `id` varchar(64) NOT NULL,
    `name` varchar(32) NOT NULL COMMENT '姓名',
    `age` int(3) DEFAULT NULL COMMENT '年龄',
    `gender` int(1) NOT NULL COMMENT '性别 1：男  0：女',
    PRIMARY KEY (`id`)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
  CREATE TABLE `person_1` (
    `id` varchar(64) NOT NULL,
    `name` varchar(32) NOT NULL COMMENT '姓名',
    `age` int(3) DEFAULT NULL COMMENT '年龄',
    `gender` int(1) NOT NULL COMMENT '性别 1：男  0：女',
    PRIMARY KEY (`id`)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
  ```

- 测试方法

  ```java
  package com.hrms.demo.mapper;
  
  import com.hrms.demo.po.PersonPO;
  import org.apache.ibatis.annotations.Mapper;
  import org.apache.ibatis.annotations.Param;
  
  /**
   * Person持久化服务
   *
   * @author zhengjun
   */
  @Mapper
  public interface PersonMapper {
  
      /**
       * 新增
       *
       * @param po 人信息
       */
      void insert(PersonPO po);
  
  
      /**
       * 通过id 查询人信息
       *
       * @param id 唯一id
       * @return 人信息
       */
      PersonPO selectById(@Param("id") String id);
  }
  
  ```

- 测试结果

  ![image-20230228094820135](https://gitee.com/zjhy/PicGo/raw/master/img/image-20230228094820135.png)

对于新增操作由于分库与分表采用相同的路由策略`id=29e35a28-2215-448d-9d88-37c040d13766` 的用户被新增至数据库ds_0,表person_0 中

![image-20230228095142393](https://gitee.com/zjhy/PicGo/raw/master/img/image-20230228095142393.png)

对于查询操作由于分库与分表采用相同的路由策略，使用分表键进行查询后，准确定位到了数据库与表，并没有进行全表扫描查询



- 常见分片算法及使用（分库与分表算法相同以下只列举分表使用方式）

  - **取模**

    通过数据取模来分片,只需要指定分片算法类型和分片的数量，就会自动根据**分片键的数据** % **分片的数量** 完成分片

    ```yaml
    spring:
      shardingsphere:
        rules:
          sharding:
            tables:
              person:
                actual-data-nodes: hrms-test.person_$->{0..1}
                # 分表策略可以使用内置算法或自定义
                table-strategy:
                  standard:
                    # 分表键需为数字类型
                    sharding-column: id
                    sharding-algorithm-name: table-mod
            # 分表
            sharding-algorithms:
              table-mod:
                type: MOD
                props:
                  sharding-count: 2
    ```

  - **哈希取模**

    取模算法的基础上加了一层 hash运算 然后再取模,主要的特点是 可以针对**非数值类型字段**作为**分片键**

    ```yaml
    spring:
      shardingsphere:
        rules:
          sharding:
            tables:
              person:
                actual-data-nodes: hrms-test.person_$->{0..1}
                # 分表策略可以使用内置算法或自定义
                table-strategy:
                  standard:
                    #分表键为非数字类型
                    sharding-column: id
                    sharding-algorithm-name: table-hashmod
            # 分表
            sharding-algorithms:
              table-hashmod:
                type: HASH_MOD
                props:
                  sharding-count: 2
    ```

  - **基于分片容量的范围分片**

    根据数据的容量进行拆分;比如一个需求一个表中最多只让存2条数据，就可以使用这个分片算法，用来严格的控制每个表的容量;分配键需为数字类型

    ```yaml
    spring:
      shardingsphere:
        rules:
          sharding:
            tables:
              person:
                actual-data-nodes: hrms-test.person_$->{0..5}
                # 分表策略可以使用内置算法或自定义
                table-strategy:
                  standard:
                    #分表键需为数字类型
                    sharding-column: id
                    sharding-algorithm-name: volume-range
            # 分表
            sharding-algorithms:
              volume-range:
                type: VOLUME_RANGE
                # 分表后数据区间(-∞,1),[1,3),[3,5),[5,7),[7,8),[8,+∞) 分表对应person_0、person_1、person_2、person_3、person_4、person_5
                props:
                  # 表示每张表中最大两条数据
                  sharding-volume: 2
                  # 下限为1
                  range-lower: 1
                  # 上限为8
                  range-upper: 8
    ```

  - **基于分片边界的范围分片**

    和基于容量的分界的算法类似，都是为了能够切分成几个区间

    ```yaml
    spring:
      shardingsphere:
        rules:
          sharding:
            tables:
              person:
                actual-data-nodes: hrms-test.person_$->{0..2}
                # 分表策略可以使用内置算法或自定义
                table-strategy:
                  standard:
                    #分表键需为数字类型
                    sharding-column: id
                    sharding-algorithm-name: boundary-range
            # 分表
            sharding-algorithms:
              boundary-range:
                type: BOUNDARY_RANGE
                # 分表后数据区间(-∞,10),[10,40),[40,+∞) 分表对应person_0、person_1、person_2
                props:
                  # 多个数字逗号分割，相当自定义区间
                  sharding-ranges: 10,40
    ```

  - **自动时间段分配**

    此类型针对时间字段类型作为分片键进行查询;可根据固定的时间段，比如天，月，年进行分表

    ```yaml
    spring:
      shardingsphere:
        rules:
          sharding:
            tables:
              person:
                actual-data-nodes: hrms-test.person_$->{0..2}
                # 分表策略可以使用内置算法或自定义
                table-strategy:
                  standard:
                    #分表键需为时间类型
                    sharding-column: create_time
                    sharding-algorithm-name: auto-interval
            # 分表
            sharding-algorithms:
              auto-interval:
                type: AUTO_INTERVAL
                # 分表后数据区间(-∞,2023-02-28 00:08:00),[2023-02-28 00:08:00,2023-03-01 00:08:00),(2023-03-01 00:08:00,+∞) 分表对应person_0、person_1、person_2,
                # 精确值8 分取值结合算法 org.apache.shardingsphere.sharding.algorithm.sharding.datetime.AutoIntervalShardingAlgorithm 参考
                props:
                  # 开始最小时间
                  datetime-lower: '2023-02-28 00:00:00'
                  # 开始最大时间
                  datetime-upper: '2023-03-01 00:00:00'
                  # 每个分区的秒数 86400 为一天秒数
                  sharding-seconds: 86400
    ```

  - **行表达式分片**

    简单的单分片键的，基于goovy 表达式的inline 配置语句,inline 不支持范围查询，如果需要范围查询需要配置 allow-range-query-with-inline-sharding:true,走全表扫描的范围查询

    ```yaml
    spring:
      shardingsphere:
        rules:
          sharding:
            tables:
              person:
                actual-data-nodes: hrms-test.person_$->{0..2}
                # 分表策略可以使用内置算法或自定义
                table-strategy:
                  standard:
                    #分表键需为数字类型
                    sharding-column: id
                    sharding-algorithm-name: inline
            # 分表
            sharding-algorithms:
              inline:
                type: INLINE
                props:
                  # 行表达式(id 对3 取模)
                  algorithm-expression: person_$->{id % 3}
                  # 是否支持范围查询
                  allow-range-query-with-inline-sharding: true
    ```

  - **时间范围分片**

    一种时间范围的分片，跟自动时间段不同的是，分片的后缀是可以有意义的，比如t_person_2023_01  t_person_2023_02  可以是以时间为后缀的

    ```yaml
    spring:
      shardingsphere:
        rules:
          sharding:
            tables:
              person:
                actual-data-nodes: hrms-test.person_2023_$->{['01','02','03','04','05','06','07','08','09','10','11','12']}
                # 分表策略可以使用内置算法或自定义
                table-strategy:
                  standard:
                    #分表键需为日期
                    sharding-column: create_time
                    sharding-algorithm-name: interval
            # 分表
            sharding-algorithms:
              # 具体算法参考 org.apache.shardingsphere.sharding.algorithm.sharding.datetime.IntervalShardingAlgorithm
              interval:
                type: INTERVAL
                props:
                  datetime-pattern: 'yyyy-MM-dd HH:mm:ss'
                  datetime-lower: '2023-01-01 00:00:00'
                  datetime-upper: '2024-01-01 00:00:00'
                  sharding-suffix-pattern: 'yyyy-MM-dd'
                  datetime-interval-amount: 1
                  datetime-interval-unit: MONTHS
    ```

  - **复合行表达式分片**

    支持多个分片键并且使用行表达式来分片

    ```yaml
    spring:
      shardingsphere:
        rules:
          sharding:
            tables:
              person:
                actual-data-nodes: hrms-test.person_$->{0..2}
                # 分表策略可以使用内置算法或自定义
                table-strategy:
                  complex:
                    #多分表键均为数字类型
                    sharding-columns: id,age
                    sharding-algorithm-name: complex-inline
            # 分表
            sharding-algorithms:
              complex-inline:
                type: COMPLEX_INLINE
                props:
                  algorithm-expression: person_$->{(id + age) % 3}
                  sharding-columns: id,age
                  allow-range-query-with-inline-sharding: true
    ```

  - **Hint 行表达式分片**

    通过inline 表达式和 Api 来实现一个分片算法,在此配置中，策略为 hint ,不用指定分片键，不从数据中解析分片信息，而是通过HintManager

    ```yaml
    spring:
      shardingsphere:
        rules:
          sharding:
            tables:
              person:
                actual-data-nodes: hrms-test.person_$->{0..2}
                # 分表策略可以使用内置算法或自定义
                table-strategy:
                  hint:
                    sharding-algorithm-name: hint-inline
            # 分表
            sharding-algorithms:
              hint-inline:
                type: HINT_INLINE
                # value 实际值通过 HintManager 对应的api 来赋值
                props:
                  algorithm-expression: person_$->{value}
    ```

    ![image-20230228150809516](https://gitee.com/zjhy/PicGo/raw/master/img/image-20230228150809516.png)

- **自定义分片**

  | 分片类型 | 分片描述 | 分片实现类                                                   |
  | -------- | -------- | ------------------------------------------------------------ |
  | STANDARD | 标准分片 | org.apache.shardingsphere.sharding.api.sharding.standard.StandardShardingAlgorithm |
  | COMPLEX  | 复合分片 | org.apache.shardingsphere.sharding.api.sharding.complex.ComplexKeysShardingAlgorithm |
  | HINT     | 精准分片 | org.apache.shardingsphere.sharding.api.sharding.hint.HintShardingAlgorithm |

  自定义实现及配置，以自定义标准算法为示例，其他类型自定义类似

  ```java
  package com.hrms.demo.sharding;
  
  import org.apache.shardingsphere.infra.datanode.DataNodeInfo;
  import org.apache.shardingsphere.sharding.api.sharding.standard.PreciseShardingValue;
  import org.apache.shardingsphere.sharding.api.sharding.standard.RangeShardingValue;
  import org.apache.shardingsphere.sharding.api.sharding.standard.StandardShardingAlgorithm;
  
  import java.util.Collection;
  import java.util.Properties;
  
  /**
   * 自定义标准分片算法
   *
   * @author zhengjun
   */
  public class CustomStandardShardingAlgorithm implements StandardShardingAlgorithm<String> {
  
      /**
       * 可以定义算法所需要的配置
       */
      private Properties props;
  
      /**
       * 自定义属性
       */
      private static final String SHARDING_COUNT_KEY = "sharding-count";
  
      @Override
      public String doSharding(Collection<String> availableTargetNames, PreciseShardingValue<String> shardingValue) {
          int shardingCount = Integer.parseInt(props.getProperty(SHARDING_COUNT_KEY));
          String suffix = String.valueOf(Math.abs(shardingValue.getValue().hashCode()) % shardingCount);
          return findMatchedTargetName(availableTargetNames, suffix, shardingValue.getDataNodeInfo()).orElse(null);
      }
  
      @Override
      public Collection<String> doSharding(Collection<String> availableTargetNames, RangeShardingValue<String> shardingValue) {
          return availableTargetNames;
      }
  
      @Override
      public Properties getProps() {
          return props;
      }
  
      @Override
      public void init(Properties properties) {
          this.props = properties;
      }
  
      public void setProps(Properties props) {
          this.props = props;
      }
  }
  
  ```

  ```yaml
  spring:
    shardingsphere:
      rules:
        sharding:
          tables:
            person:
              actual-data-nodes: hrms-test.person_$->{0..1}
              # 自定义算法配置入口 org.apache.shardingsphere.sharding.route.strategy.ShardingStrategyFactory
              table-strategy:
                standard:
                  sharding-column: id
                  sharding-algorithm-name: standard-custom
          # 分表
          sharding-algorithms:
            # 自定义算法程序入口 org.apache.shardingsphere.sharding.algorithm.sharding.classbased.ClassBasedShardingAlgorithm
            standard-custom:
              type: CLASS_BASED
              props:
                # 策略类型根据自定义类型可选 STANDARD、COMPLEX、HINT
                strategy: STANDARD
                # 自定义算法类名称为固定属性
                algorithmClassName: com.hrms.demo.sharding.CustomStandardShardingAlgorithm
                #自定义属性
                sharding-count: 2
  ```

  

#### 示例代码下线

[示例参考](https://github.com/zjhylove/sharding-jdbc-demo.git
