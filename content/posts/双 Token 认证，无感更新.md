---
title: 双 Token 认证，无感更新
date: 2024-08-28
draft: "false"
tags:
  - Token
  - 认证
---


![](https://raw.githubusercontent.com/vankykoo/image/main/cut/008.png)

# 一、流程

用户首次登录后，服务端给客户端发两个 token ，分别为 accessToken 和 refreshToken。

- accessToken 的生存时间较短，用于后续客户访问接口时使用。
- refreshToken 则用于更新 accessToken 并返回给客户端， 更新后 refreshToken 也需要刷新。

更新或生成accessToken时，需要更新用户所有的 refreshToken 的过期时间。

# 二、token 说明

1. 过期时间【测试】

- accessToken ：2天
- refreshToken ：15天

1. 存储信息：

- accessToken：


- - 用户基本信息：用户Id，用户的权限


- refreshToken：


- - 过期时间
  - 用户id

1. redis 缓存内容：

使用 hash 数据类型保存用户的token信息，因为一个用户可能会多地登录。

- Key：【user_token : [userId]】
- Value【hash】：


- - key : accessToken
  - value：


- - - refreshToken
    - expireTime（refreshToken的过期时间）

# 三、社交登录

社交登录简单基本流程：【以gitee为例】

1. 在第三方应用【gitee】中注册应用，获取 client_id 和 client_secret
2. 引导用户到第三方【gitee】认证页面，此时请求带上 client_id
3. 用户授权后，带着第三方【gitee】生成的 code 到回调地址。
4. 服务端带着 code，client_id，client_secret，redirect_uri 信息，请求第三方【gitee】获取access_token
5. 服务端带着 access_token 可以获取用户的信息
6. 可以使用用户信息进行 注册/登录

# 四、问题及解决

## 1、token 过期后无法解析

使用以下 jwt 工具进行生成jwt和解析jwt。

```xml
<dependency>
  <groupId>io.jsonwebtoken</groupId>
  <artifactId>jjwt</artifactId>
  <version>0.9.1</version>
</dependency>
```

如果 解析过期的 token 会报错，抛出【io.jsonwebtoken.ExpiredJwtException】异常。

io.jsonwebtoken.ExpiredJwtException: JWT expired at 2024-05-15T22:00:07Z. 

Current time: 2024-05-15T22:00:56Z, a difference of 49868 milliseconds.  

Allowed clock skew: 0 milliseconds.

所以过期后的 token 就没什么用了，我们无法从中获取到对应的refreshToken信息。

### 解决：

在redis 中存储该 accessToken 对应的 refreshToken，如果 accessToken 过期，就到redis 中获取 refreshToken ，解析 refreshToken，获取用户信息，并更新accessToken。

如果 refreshToken 也过期了，就提醒用户重新登录。

最后删除掉过期的 accessToken 对应的信息。

redis accessToken 和 refreshToken 映射信息：

key：【 access_refresh_mapping : accessToken】

value：refreshToken

## 2、若accessToken过期时，同时返回新的accessToken和响应数据

前提：这个accessToken对应的refreshToken没过期。

### 解决：

在过滤器中检测到accessToken过期但refreshToken没过期，生成新的accessToken，并把新的accessToken放到响应头中。

```java
Claims claims = Jwts.parser()
        .setSigningKey(REFRESH_SECRET_KEY)
        .parseClaimsJws(refreshToken)
        .getBody();

String newAccessToken = Jwts.builder()
            .setSubject(claims.getSubject())
            .setExpiration(new Date(System.currentTimeMillis() + EXPIRATION_TIME))
            .signWith(SignatureAlgorithm.HS512, SECRET_KEY)
            .compact();

response.setHeader("x-access-token", newAccessToken);
request.setAttribute("claims", claims);
```

## 3、feign 请求不用认证

微服务内部的请求不用请求了，那么如何辨别哪些请求是 feign 请求呢？

### 解决：

当发送 feign 请求前，可以在请求头加上 feign 请求相关标志。