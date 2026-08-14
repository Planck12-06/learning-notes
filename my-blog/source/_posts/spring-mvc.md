---
title: spring mvc
date: 2026-07-31 17:33:18
categories:
tags:
---

在 WebMvcConfiguration 中扩展Spring MVC的消息转换器，统一对日期类型进行格式化处理

```java
protected void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
    log.info("开始扩展消息转换器...");
    //创建一个消息转化器对象    
    MappingJackson2HttpMessageConverter converter = new MappingJackson2HttpMessageConverter();    
    //设置对象转换器，可以将Java对象转为json字符串    
    converter.setObjectMapper(new JacksonObjectMapper());
    //将我们自己的转换器放入spring MVC框架的容器中  
    converters.add(0,converter);
}

```

`BeanUtils`类中的copyProperties方法，可以将一个对象的属性值拷贝到另一个对象中，但是如果两个对象的属性名称相同，则会进行覆盖。

`BaseContext`类中声明ThreadLocal变量，用于存储当前线程的上下文信息，如用户信息、请求信息等。并且提供了get方法用于获取当前线程的上下文信息和set方法用于设置当前线程的上下文信息。