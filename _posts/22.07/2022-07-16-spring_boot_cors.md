---
title: Spring Boot Cors 정책 적용
author: woodsection
date: 2022-07-16 11:30:00 +0800
categories: [Web, Java, Spring Boot]
tags: [Web]
published: true
---

# Spring Boot Cors

# 현재 WebMvcConfigurerAdapter는 Deprecated

> `WebMvcConfigurer` 는 자동구성된 스프링 MVC 구성에 `Formatter`, `MessageConverter` 등을 추가등록할 수 있다.
 `WebMvcRegistrations`는 `RequestMappingHandlerMapping`, `RequestMappingHandlerAdapter`와 `ExceptionHandlerExceptionResolver`를 재정의할 때 사용한다.

`WebMvcConfigurr`와 `WebMvcRegistrations` 소스를 살펴보면 메서드들을 `default` 메서드로 선언했다. 필요한 메서드만 구현해서 사용하라는 의미다. 

Java7을 지원하는 스프링 부트 1.5 에서는 `[default` 메서드(https://goo.gl/A7CL31)](https://docs.oracle.com/javase/tutorial/java/IandI/defaultmethods.html)가 없었기에 `WebMvcConfigurer`를 사용하려면 인터페이스에 선언된 메서드를 모두 구현해야 했다. 그런 불편함을 해소하고자 `WebMvcConfigurerAdapter` 추상 클래스를 제공했다. 이 추상 클래스를 상속받아 필요한 메서드만 오버라이드했다.

스프링 부트 2.0부터 Java8과 스프링 5.0을 사용하면서 `WebMvcConfigurer` 메서드에 `default`를 선언했다. 그 덕분에 `WebMvcConfigurer`를 구현하는 클래스에서 모든 메서드를 구현해야하는 강제력이 사라졌다. 쓰임새가 사라진 `WebMvcConfigurerAdapter` 클래스는 스프링 부트 2.0에서 제외(Deprecated)되었다.
> 

# Spring Boot Cors 정책 허용

- WebSecurityConfigurerAdapter를 상속받고, Spring Security가 적용된 `WebSecirityConfig.java`
- CorsConfigurationSource를 스프링 Bean 등록

![Untitled](/assets/img/2022-07-16-spring_boot_cors/Untitled.png)

- addAllowedOrigin() CORS 정책 적용

![Untitled](/assets/img/2022-07-16-spring_boot_cors/Untitled%201.png)

- registerCorsConfiguration(), 모든 path에 대해 CORS 정책 적용

![Untitled](/assets/img/2022-07-16-spring_boot_cors/Untitled%202.png)

# Exception 발생 시 CORS Header 적용여부

## TestController.java

```java
package com.skt.scale.paas.server.controller;

import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

@Configuration
@RestController
@RequestMapping("/api")
public class TestController {

  @RequestMapping(value = "/test/null", method = RequestMethod.GET)
  public void resNull(String test) {
    throw new NullPointerException();
  }

  @ExceptionHandler(NullPointerException.class)
  public ResponseEntity<Object> handleAll(Exception ex) {
    ex.printStackTrace();
    return new ResponseEntity<>(ex.toString(), new HttpHeaders(), HttpStatus.INTERNAL_SERVER_ERROR);
  }
}
```

## request 결과

![Untitled](/assets/img/2022-07-16-spring_boot_cors/Untitled%203.png)

- NullPointerException 발생 및 500 에러 response
- origin, referer 헤더에 따른 CORS 헤더는 정상적으로 반환