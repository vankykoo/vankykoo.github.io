---
title: 使用Guava 工具实现驼峰命名相互转换
date: 2024-04-25
draft: "false"
tags:
  - 开发规范
  - 开发工具
---

## 1.引入依赖

```xml
<dependency>
  <groupId>com.google.guava</groupId>
  <artifactId>guava</artifactId>
  <version>33.1.0-jre</version>
</dependency>
```



## 2.下划线 -> 驼峰

```java
String columnName = "student_name";
String resultStr = CaseFormat.LOWER_UNDERSCORE.to(CaseFormat.LOWER_CAMEL, columnName);

// resultStr输出 ---> studentName 
```



## 3.驼峰 -> 下划线

```java
String resultStr = CaseFormat.LOWER_CAMEL.to(CaseFormat.LOWER_UNDERSCORE, "studentName");

// resultStr输出 ---> student_name
```

