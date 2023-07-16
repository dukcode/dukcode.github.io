---
layout: single
title: "Spring Security에서 H2 Console 사용하기"
categories: spring
tag: [spring, spring-security, H2, h2-console, csrf, X-Frame-Options]
toc: true
toc_sticky: true
author_profile: true
sidebar:
  nav: "docs"
header:
  teaser: "https://i.imgur.com/iuKiWPf.png"
# search: false
---

# Spring Security에서 H2 Console 사용하기

![](https://i.imgur.com/iuKiWPf.png){:.align-center width="500" }

로컬 환경에서 자주 사용되는 H2 DB는 웹 클라이언트를 제공하고 있고, 이를 Spring Boot에서도 지원한다.

가장 먼저 `application.yml`을 다음과 같이 설정해 콘솔을 활성화 시킨다.

```yml
spring:  
  h2:  
    console:  
      enabled: true # /h2-console 설정
  datasource:  
    url: jdbc:h2:mem:testdb # 메모리 H2 DB 경로 설정
```

애플리케이션을 실행하고 `http://localhost:8080/h2-console`로 접속하게 되면 H2가 제공하는 웹 콘솔에 접근할 수 있다. (`url`설정을 하지 않으면 Spring Boot가 임의로 이름을 설정하기 때문에 접속이 번거로워 추가했다.)

하지만 Spring Security와 이를 함께 사용하려면 접근이 불가능하다. 

- `permitAll()` 설정
- CSRF 설정
- X-Frame-Options 설정

위와 같은 Spring Security 설정이 필요하다. 그 이유와 해결 방법을 알아보자.

## `permitAll()` 설정

매번 콘솔에 접근할 때마다 인증을 요구하면 번거로울 것이다. `/h2-console` 관련 URL들에 대해 인증을 면제해주어야 쉽게 접근할 수 있다. 다음과 같은 설정으로 `/h2-console` 관련 URL에 대해 인증을 면제할 수 있다.

```java

@EnableWebSecurity
@Configuration
public class SpringConfig {

  @Bean
  public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http.authorizeHttpRequests(
            request -> request.requestMatchers(PathRequest.toH2Console()).permitAll()
                .anyRequest().authenticated());

    return http.build();
  }
}

```

그리고 나머지 URL에 대해 모두 인증이 필요하도록 설정한다.

> `PathRequest.toH2Console()`을 사용하면 H2 Console의 URL을 `application.yml`을 통해 디폴트가 아닌 다른 값으로 변경하더라도 해당 클래스를 고칠 필요가 없다.

## CSRF 설정

H2 웹콘솔은 CSRF 토큰을 받아도 다음 요청에 활용하지 않는다. 즉, H2 웹콘솔은 CSRF에 대응되지 않는 페이지이다.

따라서 `/h2-console`관련한 URL에 대해 CSRF 설정을 꺼주어야 할 필요가 있다. 

Spring Security는 `CsrfFilter`를 기본적으로 Default Filter로 설정하고 있다. `CsrfFilter`가 H2 웹콘솔 관련 경로를 무시하도록 설정하자.

```java

@EnableWebSecurity
@Configuration
public class SpringConfig {

  @Bean
  public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http.authorizeHttpRequests(
            request -> request.requestMatchers(PathRequest.toH2Console()).permitAll()
                .anyRequest().authenticated())
        .csrf(csrf -> csrf.ignoringRequestMatchers(PathRequest.toH2Console()));
        
    return http.build();
  }
}

```

`/h2-console`에 관련한 URL만 CSRF를 적용하지 않도록 설정하고 있다. **만약 REST API를 만든다면 CSRF 보안 기능을 끄는 편이므로 관련 코드를 `csrf(AbstractHttpConfigurer::disable)`로 교체해 CSRF 보안 기능을 Spring Security에서 비활성화 할 수 있다.**

## X-Frame-Options 설정

H2 웹콘솔은 iframe을 통해 화면을 구성한다. 브라우저에서 iframe에 대한 모든 요청을 허용하면 **디도스 공격**, **클릭재킹** 공격에 취약해 질 수 있다. 따라서 **브라우저**는 요청 응답에 있는 `X-Frame-Options` 헤더의 내용에 따라 iframe에서의 요청을 허용할 것인지 허용하지 않을 것인지 판단한다.

Spring Security는 기본적으로 `HeaderWriterFilter`를 활성화 시킨다. `HeaderWriterFilter`는 `XFrameOptionsHeaderWriter`를 이용해 설정에 따라 `X-Frame-Options`헤더에 `DENY`, `SAMEORIGIN` 등의 값을 설정한다.

기본 설정은 iframe을 사용하지 못하도록 `DENY`로 설정하고 있다. 따라서 브라우저는 모든 iframe에서의 요청을 막는다.

H2 웹콘솔의 iframe이 정상적으로 작동하려면 이를 같은 Origin에 대해 허용하도록 설정해야 한다.

```java

@EnableWebSecurity(debug = true)
@Configuration
public class SpringConfig {

  @Bean
  public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http.authorizeHttpRequests(
            request -> request.requestMatchers(PathRequest.toH2Console()).permitAll()
                .anyRequest().authenticated())
        .csrf(csrf -> csrf.ignoringRequestMatchers(PathRequest.toH2Console()))
        .headers(headers -> headers.frameOptions(FrameOptionsConfig::sameOrigin));

    return http.build();
  }
}

```

위와 같이 설정하면 `X-Frame-Options` 헤더의 값이 `SAMEORIGIN`으로 변경된다. 브라우저는 iframe에서 일어난 요청에 대해 Origin을 비교하고 같으면 요청을 허용하게 된다. 즉, `http://localhost:8080/**`의 iframe에서의 요청이 `http://localhost:8080/**`으로 향하는 요청인지 확인한다.

위의 3가지 설정을 마치면 H2 웹콘솔을 Spring Security와 함께 이용할 수 있다.
