---
layout:     post
title:      MyBatis与ShardingJdbc集成ShardingJdbc某些语句无法执行的解决方案
subtitle:   sql执行时替换shardingJdbc执行sqlSession，并指定当前sql执行对应的dataSource即可
date:       2021-05-10
author:     zj
header-img: img/post-bg-debug.png
catalog: true
tags:
    - 实战经验

---

###### 核心代码

- ```java
  package com.jdaz.user.member.api;
  
  import com.jdaz.user.member.repository.dao.ActivityEventMapper;
  import com.jdaz.user.member.support.utils.SpringUtils;
  import org.apache.ibatis.binding.MapperRegistry;
  import org.apache.ibatis.mapping.Environment;
  import org.apache.ibatis.mapping.MappedStatement;
  import org.apache.ibatis.mapping.ResultMap;
  import org.apache.ibatis.session.Configuration;
  import org.apache.ibatis.session.SqlSession;
  import org.apache.ibatis.session.SqlSessionFactory;
  import org.apache.ibatis.session.defaults.DefaultSqlSessionFactory;
  import org.apache.shardingsphere.shardingjdbc.jdbc.core.datasource.ShardingDataSource;
  import org.springframework.util.ReflectionUtils;
  
  import javax.sql.DataSource;
  import java.lang.reflect.Field;
  import java.util.Map;
  
  /**
   * 屏蔽sharding-jdbc执行器
   *
   * @author zj
   * @date 2020/12/23 20:08
   */
  public class ShieldShardingJdbcMapperExecutor {
  
      private ShieldShardingJdbcMapperExecutor() {
      }
  
      /**
       * 屏蔽sharding-jdbc 执行sql
       *
       * @param mapper         mybatis mapper类
       * @param dataSourceId   数据源id
       * @param mapperExecutor mapper 执行器
       * @param <T>            mapper 接口
       */
      public static <T> void executeMapper(Class<T> mapper, String dataSourceId, MapperExecutor<T> mapperExecutor) {
  
          Configuration configuration = getCurrentExecuteConfiguration();
  
          Configuration copyConfiguration = copyCurrentConfiguration(dataSourceId, configuration);
  
          DefaultSqlSessionFactory copySqlSessionFactory = new DefaultSqlSessionFactory(copyConfiguration);
  
          SqlSession session = copySqlSessionFactory.openSession();
  
          try {
              mapperExecutor.executor(session.getMapper(mapper));
          } catch (Exception e) {
              session.rollback();
          } finally {
              session.close();
          }
      }
  
      /**
       * 通过数据源id 从sharding-jdbc 列表中获取指定数据源
       *
       * @param dataSourceId 数据源id
       * @return 数据源
       */
      private static DataSource getDataSourceFromShardingDataSourceMap(String dataSourceId) {
          ShardingDataSource shardingDataSource = SpringUtils.getBean(ShardingDataSource.class);
          Map<String, DataSource> dataSourceMap = shardingDataSource.getConnection().getDataSourceMap();
          return dataSourceMap.get(dataSourceId);
      }
  
      /**
       * 获取当前sql 配置信息
       *
       * @return 执行环境
       */
      private static Configuration getCurrentExecuteConfiguration() {
          SqlSessionFactory sqlSessionFactory = SpringUtils.getBean(SqlSessionFactory.class);
          return sqlSessionFactory.getConfiguration();
      }
  
      /**
       * 复制sql 执行环境
       *
       * @param dataSourceId  数据源id
       * @param configuration 当前执行环境
       * @return 所需执行环境
       */
      private static Configuration copyCurrentConfiguration(String dataSourceId, Configuration configuration) {
  
          Environment environment = configuration.getEnvironment();
          Environment copyEnv = new Environment.Builder(dataSourceId)
                  .dataSource(getDataSourceFromShardingDataSourceMap(dataSourceId)).transactionFactory(environment.getTransactionFactory()).build();
          Configuration copyConfiguration = new Configuration(copyEnv);
  
          Field mapperRegistry = ReflectionUtils.findField(Configuration.class, "mapperRegistry");
          ReflectionUtils.makeAccessible(mapperRegistry);
          MapperRegistry mapperRegistryValue = (MapperRegistry) ReflectionUtils.getField(mapperRegistry, configuration);
          ReflectionUtils.setField(mapperRegistry, copyConfiguration, mapperRegistryValue);
  
  
          Field mappedStatements = ReflectionUtils.findField(Configuration.class, "mappedStatements");
          ReflectionUtils.makeAccessible(mappedStatements);
          Map<String, MappedStatement> mappedStatementsValue = (Map<String, MappedStatement>) ReflectionUtils.getField(mappedStatements, configuration);
          ReflectionUtils.setField(mappedStatements, copyConfiguration, mappedStatementsValue);
  
          Field resultMaps = ReflectionUtils.findField(Configuration.class, "resultMaps");
          ReflectionUtils.makeAccessible(resultMaps);
          Map<String, ResultMap> resultMapsValue = (Map<String, ResultMap>) ReflectionUtils.getField(resultMaps, configuration);
          ReflectionUtils.setField(resultMaps, copyConfiguration, resultMapsValue);
          return copyConfiguration;
      }
  
      /**
       * 使用方式
       */
      public static void main(String[] args) {
          ShieldShardingJdbcMapperExecutor.executeMapper(ActivityEventMapper.class, "ds1", ActivityEventMapper::selectList);
      }
  }
  
  ```

  

- ```java
  /**
   * mapper 执行器
   *
   * @author zj
   * @date 2020/12/23 20:08
   */
  public interface MapperExecutor<M> {
  
      void executor(M mapper);
  }
  ```

  
