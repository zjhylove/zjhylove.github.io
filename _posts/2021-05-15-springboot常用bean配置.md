---
layout:     post
title:      springboot常用bean配置
subtitle:   常用bean配置
date:       2021-05-15
author:     zj
header-img: img/post-bg-coffee.jpg
catalog: true
tags:
    - 实战经验
---

   

- Jackson 序列化

  ```java
  @Bean
      @Primary
      @ConditionalOnMissingBean(ObjectMapper.class)
      public ObjectMapper customObjectMapper(Jackson2ObjectMapperBuilder builder) {
          Map<Class<?>, JsonSerializer<?>> serializers = new HashMap<>(2);
          serializers.put(Long.class, new ToStringSerializer());
          DateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
          serializers.put(Date.class, new DateSerializer(true, dateFormat));
          builder.serializersByType(serializers);
  
          Map<Class<?>, JsonDeserializer<?>> deserializers = new HashMap<>(1);
          DateDeserializers.DateDeserializer base = DateDeserializers.DateDeserializer.instance;
          DateDeserializers.DateDeserializer dateDeserializer = new DateDeserializers.DateDeserializer(base, dateFormat, null);
          deserializers.put(Date.class, dateDeserializer);
          builder.deserializersByType(deserializers);
  
          ObjectMapper objectMapper = new ObjectMapper();
          objectMapper.setDateFormat(dateFormat);
          //忽略属性不匹配反序列话报错
          objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
          builder.configure(objectMapper);
          return builder.build();
      }
  ```

- openfeign 拦截器，比如添加授权信息，加解密等等

  ```java
  @Component
  public class FeignInterceptor implements RequestInterceptor {
  
      public static final String AUTH_HEADER = "Authorization";
  
      @Value("${ins.user.center.authKey}")
      private String authKey;
  
      @Value("${ins.user.center.url}")
      private String feignServerUrl;
  
      @Override
      public void apply(RequestTemplate requestTemplate) {
          Target<?> target = requestTemplate.feignTarget();
          if (!UriUtils.isAbsolute(feignServerUrl)) {
              feignServerUrl = "http://" + StringUtils.trim(feignServerUrl);
          }
          if (Objects.equals(target.url(), feignServerUrl)) {
              requestTemplate.header(AUTH_HEADER, authKey);
          }
      }
  }
  ```

- SpringBeanUtils 配置

  ```java
  @Component
  public final class SpringUtils implements BeanFactoryPostProcessor {
  
      private static ConfigurableListableBeanFactory beanFactory;
  
      @Override
      public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
          SpringUtils.beanFactory = beanFactory;
      }
  
      /**
       * Access to the object
       *
       * @param name bean id名称
       * @return Object An instance of a bean registered with the given name
       */
      @SuppressWarnings("unchecked")
      public static <T> T getBean(String name) {
          return (T) beanFactory.getBean(name);
      }
  
      /**
       * Gets an object of Type required Type
       *
       * @param clz
       * @return
       */
      public static <T> T getBean(Class<T> clz) {
          @SuppressWarnings("unchecked")
          T result = (T) beanFactory.getBean(clz);
          return result;
      }
  
      /**
       * Returns true if the Bean Factory contains a Bean definition that matches the given name
       *
       * @param name
       * @return boolean
       */
      public static boolean containsBean(String name) {
          return beanFactory.containsBean(name);
      }
  
      /**
       * Determines whether the bean definition registered with the given name is a singleton or a prototype.
       * If the bean definition corresponding to the given name is not found,
       * an exception is thrown（NoSuchBeanDefinitionException）
       *
       * @param name
       * @return boolean
       */
      public static boolean isSingleton(String name) {
          return beanFactory.isSingleton(name);
      }
  
      /**
       * @param name
       * @return Class The type of the registered object
       * @throws org.springframework.beans.factory.NoSuchBeanDefinitionException
       */
      public static Class<?> getType(String name) {
          return beanFactory.getType(name);
      }
  
      /**
       * If the given bean name has aliases in the bean definition, these aliases are returned
       *
       * @param name
       * @return
       */
      public static String[] getAliases(String name) {
          return beanFactory.getAliases(name);
      }
  
      /**
       * Gets the specified annotation bean
       *
       * @param clz
       * @return
       */
      public static Map<String, Object> getBeansWithAnnotation(Class<? extends Annotation> clz) {
          return beanFactory.getBeansWithAnnotation(clz);
      }
  
  
      /**
       * 获取某个类下的实现类的bean对象
       *
       * @param clz 超类类型
       * @param <T> ioc容器子类对象
       * @return 子类对象
       */
      public static <T> Map<String, T> getBeansOfType(Class<T> clz) {
          return beanFactory.getBeansOfType(clz);
      }
  
  }
  ```

- RestTemplate

  ```java
      @Bean
      public RestTemplate restTemplate(ObjectMapper objectMapper) {
          RestTemplate restTemplate = new RestTemplate();
  
          HttpComponentsClientHttpRequestFactory factory = new HttpComponentsClientHttpRequestFactory();
          factory.setConnectTimeout(20000);
          factory.setReadTimeout(50000);
          restTemplate.setRequestFactory(factory);
  
          MediaType[] mediaTypes = new MediaType[]{
                  MediaType.APPLICATION_JSON,
                  MediaType.APPLICATION_OCTET_STREAM,
  
                  MediaType.TEXT_HTML,
                  MediaType.TEXT_PLAIN,
                  MediaType.TEXT_XML,
                  MediaType.APPLICATION_STREAM_JSON,
                  MediaType.APPLICATION_ATOM_XML,
                  MediaType.APPLICATION_FORM_URLENCODED,
                  MediaType.APPLICATION_JSON_UTF8,
                  MediaType.APPLICATION_PDF,
          };
          MappingJackson2HttpMessageConverter converter = new MappingJackson2HttpMessageConverter();
          converter.setObjectMapper(objectMapper);
          converter.setSupportedMediaTypes(Arrays.asList(mediaTypes));
          restTemplate.getMessageConverters().add(converter);
          restTemplate.getInterceptors().add(new RestTemplateLoggerAspect());
          return restTemplate;
      }
  ```

- rocketmq 生产者注册

  ```
  @Bean
      public MQProducer registeredProducer() {
          try {
              DefaultMQProducer producer = new DefaultMQProducer(producerProperty.getGroupName());
              producer.setNamesrvAddr(producerProperty.getNamesrvAddr());
              producer.setRetryTimesWhenSendFailed(producerProperty.getRetryTimes());
              LinkedHashSet<String> ipv4s = NetUtil.localIpv4s();
              if (ipv4s.size() == 0) {
                  LOGGER.error("没有可用的ip地址进行注册mq生产者");
                  throw ErrorConstant.newServiceException(ErrorConstant.SYS_ERR, "注册mq生产者没有可用的ip", null);
              }
              producer.setInstanceName(ipv4s.iterator().next());
              producer.start();
              return producer;
          } catch (MQClientException e) {
              LOGGER.error("启动mq 生产者失败mq 错误码={}，错误信息={}", new Object[]{e.getResponseCode(), e.getErrorMessage()}, e);
          } catch (Exception e) {
              LOGGER.error("注册mq 生产者发生不可预期异常，请检查环境配置！", e);
          }
          return new DefaultMQProducer("un created success mq");
      }
  ```

  
