---
layout:     post
title:      hrms-feignclient-mock 框架
subtitle:   对基于spring-cloud-starter-openfeign框架实现远程调用的接口支持数据响应模拟
date:       2021-10-10
author:     zj
header-img: img/post-bg-js-version.jpg
catalog: true
tags:
    - 工具整理

---

   

#### 框架介绍

- 解决了单元测试中对远程调用服务的依赖以及生成模拟数据的繁琐，但不仅仅对单元测试的数据的模拟，也支持远程调用服务没有开发完毕或者不能提供服务，需要验证本服务业务流程场景时的测试或者开发环境
- 针对spring 容器中使用spring-cloud-starter-openfeign框架实现远程调用的接口
- 自动生成模拟对象数据以json格式存放至固定文件中且支持响应对象自定义修改
- 对业务代码无侵入性即可实现接口响应数据模拟
- 相比openfeign 契约测试框架更简单易用，但是功能也更加单一，不能模拟真实http请求，专用于本服务内部逻辑测试数据模拟
- 模拟对象不支持枚举和map，这两种数据结构出现在接口交互中为大忌



#### 实现原理

- 使用框架自身模拟对象工厂类动态替换spring容器中openfeign 原对象工厂类
- 生成模拟文件名称格式【远程调用类名称+.mk】,该文件维护该远程接口类中以方法签名为key,模拟对象为value 的map 数据结构
- 防止同一远程接口类中方法重载现象，方法签名格式为【方法简称+(参数类型简称列表)+md5(原java方法签名字符串)】



#### 框架使用

- maven 依赖

  ```xml
  <dependency>
      <groupId>com.hrms.frame</groupId>
      <artifactId>hrms-feignclient-mock-lib</artifactId>
      <version>1.0-SNAPSHOT</version>
      <scope>test</scope>
  </dependency>
  ```

- 使用示例

  ![image-20211015143421280](https://gitee.com/zjhy/PicGo/raw/master/img/20211015143428.png)

- 生成模拟文件解释

  ![image-20211015144031077](https://gitee.com/zjhy/PicGo/raw/master/img/20211015144031.png)

- <font color=red>模拟文件中方法签名字段请勿随意修改否则会导致模拟数据无法正常加载</font>，修改模拟响应内容即可

