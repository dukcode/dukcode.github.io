---
layout: single
title: "[Spring Security ID/PW JWT 인증/인가] 03 - JWT 인증 구현"
categories: spring
tag: [spring, spring-security, jwt, login, authentication]
toc: true
toc_sticky: true
author_profile: true
sidebar:
  nav: "docs"
header:
  teaser: "https://i.imgur.com/SRrIfJs.png"
# search: false
---

# JWT 인증 구현

![](https://i.imgur.com/SRrIfJs.png){:.align-center width="500" }


우리는 회원 가입과 로그인 프로세스는 구현을 마쳤다. 여기에 **JWT 토큰을 이용한 인증**을 추가하면된다. 우리가 해야할 일은 다음과 같다.

- 로그인 시 JWT 토큰 **발급**
- 인증이 필요한 URL 요청 시, 요청에 포함된 JWT 토큰을 검증

이번 포스트에서 위 3가지를 순서대로 진행할 것이다. 먼저 JWT를 사용하기 위한 디펜던시를 추가해보자.

## JWT 라이브러리 추가 및 설정

가장 먼저 JWT 관련 라이브러리를 추가해주어야 한다. 대표적인 자바의 JWT 라이브러리로는 `jjwt`와 `java-jwt`가 존재한다. 어떤 것을 사용해도 상관 없지만 이번 프로젝트에서는 `java-jwt`를 사용해보려고 한다.

`build.gradle`에 다음과 같이 디펜던시를 추가한다.

```groovy
dependencies {
    // jwt
    implementation 'com.auth0:java-jwt:4.4.0'
}
```

이제 JWT 관련 설정을 추가해 보자. `application.yml`에 다음과 같이 추가한다.

```yml
jwt:  
  properties:  
    secret: 123  
    access-token-expiration-seconds: 900 # 900 15분  
    refresh-token-expiration-seconds: 1209600 # 1209600 14일  
    token-prefix: Bearer  
  response:  
    access-token-header-name: Authorization
    refresh-token-header-name: Refresh-Token
```

refresh token은 추후에 구현할 예정이므로 미리 설정 정보에 포함시켰다.

> 실제 환경에서는 secret같은 **민감한 정보**는 노출되면 안된다. 실제 개발 시에는 spring-cloud-config또는 aws-secret-manager를 이용하는 방법이 있다.

Spring은 `@Value` 애노테이션을 통해 yml 파일의 값을 가져올 수 있는 기능을 제공한다.. 이 정보들은 `@Value` 애노테이션을 이용해 `JwtService`에서 사용할 것이다.

## 로그인 시 JWT 토큰 발급

우리는 로그인을 구현하면서 `AuthenticationSuccessHandler`를 구현했다. 이전 포스트에서는 간단히 200(OK)를 보내고 로그를 찍도록 구현했지만, 이제는 로그인에 성공 시, JWT 토큰을 발급해주어야 한다.

### JwtService

먼저 JWT 토큰를 관리하는 `JwtService` 클래스를 만들고 `JwtLoginSuccessHandler`를 구현해보자.

```java
package com.dukcode.securityboilerplate.global.security.jwt.service;  
  
@RequiredArgsConstructor  
public class JwtService {  
  
  private static final String ACCESS_TOKEN_SUBJECT = "access-token";  
  private static final String REFRESH_TOKEN_SUBJECT = "refresh-token";  
  private static final String USERNAME_CLAIM = "email";  
  private static final String ROLES_CLAIM = "roles";  

  @Value("${jwt.properties.secret}")  
  private String secret;  

  @Value("${jwt.properties.access-token-expiration-seconds}")  
  private Long accessTokenExpirationSeconds;  

  @Value("${jwt.properties.refresh-token-expiration-seconds}")  
  private Long refreshTokenExpirationSeconds;  

  @Value("${jwt.properties.token-prefix}")  
  private String tokenPrefix;  

  private Algorithm algorithm;  
  private JWTVerifier verifier;  
    
  @PostConstruct  
  public void init() {  
    algorithm = Algorithm.HMAC256(secret);  
    verifier = JWT.require(algorithm).build();  
  }
  
  public String createAccessTokenWithPrefix(Authentication authentication) {  
    UserDetails userDetails = (UserDetails) authentication.getPrincipal();  
    String username = userDetails.getUsername();  
    List<String> roles = userDetails.getAuthorities().stream()  
        .map(GrantedAuthority::toString)  
        .collect(Collectors.toList());  
  
    String accessToken = createAccessToken(username, roles);  
  
    return withTokenPrefix(accessToken);  
  }  
  
  private String createAccessToken(String username, List<String> roles) {  
    return JWT.create().withSubject(ACCESS_TOKEN_SUBJECT)  
        .withIssuedAt(new Date())  
        .withExpiresAt(new Date(System.currentTimeMillis() + accessTokenExpirationSeconds * 1000L))  
        .withClaim(USERNAME_CLAIM, username)  
        .withClaim(ROLES_CLAIM, roles)  
        .sign(algorithm);  
  }  
  
  
  private String withTokenPrefix(String token) {  
    return tokenPrefix + " " + token;  
  }  
  
}
```

이제 외부에서 `createAccessTokenWithPrefix()`메서드를 호출하면 JWT 토큰을 생성할 수 있게 된다. `Bearer aaaaa.bbbbb.ccccc` 형태로 반환된 것이다.

### JwtLoginSuccessHandler

`AuthetncationSuccessHandler`를 상속하는 `JwtLoginSuccessHandler`를 구현해보자. Spring Security는 자격 증명에 성공 시 `onAuthenticationSuccess()`메서드를 호출하기 때문에 이 메서드에서 JWT 토큰을 생성해 응답 헤더에 추가한다.

```java
package com.dukcode.securityboilerplate.global.security.login.handler;  

@Slf4j  
@RequiredArgsConstructor  
public class JwtLoginSuccessHandler implements AuthenticationSuccessHandler {  
  
  private final JwtService jwtService;  

  @Value("${jwt.response.access-token-header-name}")  
  private String accessTokenHeaderName;  
  @Value("${jwt.response.refresh-token-header-name}")
  private String refreshTokenHeaderName;
  
  @Override  
  public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response,  
      Authentication authentication) throws IOException, ServletException {  
  
    String accessTokenWithPrefix = jwtService.createAccessTokenWithPrefix(authentication);  
  
    response.setHeader(accessTokenHeaderName, accessTokenWithPrefix);  
    response.setStatus(HttpStatus.OK.value());  
  }  
  
}
```

로그인이 성공했을 때, `JwtService`를 통해 JWT토큰을 생성하고 `Authorization`헤더에 토큰을 실어보낼 수 있게 한다.

### SecurityConfiguration

`JwtService`와 `JwtLoginSuccessHandler`를 빈으로 등록해보자. `SecurityConfiguration`에서 다음과 같은 메서드를 추가하면 된다.

```java
  @Bean
  public JwtService jwtService() {
    return new JwtService();
  }

  @Bean
  public AuthenticationSuccessHandler authenticationSuccessHandler() {
    return new JwtLoginSuccessHandler(jwtService());
  }

```

### 실행

PostMan으로 실제 로그인 성공 시, JWT 토큰이 실려 오는지 확인해 보자.

![](https://i.imgur.com/UeiK9zo.png)

응답 헤더에 우리가 설정한대로 토큰이 실려오는 것을 확인할 수 있다.

## JWT 토큰 검증

이제 클라이언트는 받은 토큰을 요청이 실어보내면 서버에서 해당 토큰이 맞는지 검증하면 된다. `JwtAuthenticationProcessingFilter`를 작성하고 Spring Security 필터 체인에 끼워 넣어 요청의 JWT 토큰이 정상적인 토큰인지, 유효기간 안의 토큰인지 검사할 수 있도록 해보자.

### JwtService

가장 먼저 `JwtService`에 메서드를 추가해 토큰이 정상 토큰인지 검사할 수 있도록 해보자.

```java
package com.dukcode.securityboilerplate.global.security.jwt.service;  
  
@RequiredArgsConstructor  
public class JwtService {  

  // ...
  
  public Authentication resolveAccessTokenWithPrefix(String accessTokenWithPrefix) {  
    String token = removePrefix(accessTokenWithPrefix);  
    DecodedJWT decodeToken = verifier.verify(token);  
  
    String username = decodeToken.getClaim(USERNAME_CLAIM).asString();  
    List<SimpleGrantedAuthority> roles = decodeToken.getClaim(ROLES_CLAIM)  
        .asList(String.class)  
        .stream()  
        .map(SimpleGrantedAuthority::new)  
        .toList();  
  
    return UsernamePasswordAuthenticationToken.authenticated(username, null, roles);  
  }  
  
  private String removePrefix(String token) {  
    if (!token.startsWith(tokenPrefix + " ")) {  
      throw new JWTVerificationException("Invalid Token Format");  
    }  
    return token.substring(tokenPrefix.length() + 1);  
  }  
  
}
```

클라이언트 객체는 prefix가 포함된 access token을 메서드로 전달하면 된다. `JwtService`는 토큰을 검증한다. 잘못된 토큰이거나 유효기간이 지난 토큰이라면 `JWTVerificationException` 및 하위 예외를 발생시킨다.

### JwtAuthenticationProcessingFilter

이제 JWT 토큰을 검증하는 필터를 만들어 Spring Security 필터 체인에 끼워 넣어보자. 해당 필터의 역할은 인증이 필요한 요청에서 `HttpServletRequest`의 헤더에 실려온 JWT토큰을 검사해 유효한 토큰인지 검사하고 이를 `SecurityContext`에 저장하는 것이다.

```java
package com.dukcode.securityboilerplate.global.security.authentication.filter;  
  
@Slf4j  
@RequiredArgsConstructor  
public class JwtAuthenticationProcessingFilter extends OncePerRequestFilter {  
  
  private final JwtService jwtService;  
  
  @Value("${jwt.response.access-token-header-name}")  
  private String accessTokenHeaderName;  
  
  @Override  
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,  
      FilterChain filterChain) throws ServletException, IOException {  
  
    String accessTokenWithPrefix = extractAccessTokenWithPrefix(request);  
  
    // access-token 없으면 pass  
    if (!StringUtils.hasText(accessTokenWithPrefix)) {  
      filterChain.doFilter(request, response);  
      return;  
    }  
  
    try {  
      Authentication authentication  
          = jwtService.resolveAccessTokenWithPrefix(accessTokenWithPrefix);  
      // SecurityContext에 인증 정보를 저장
      SecurityContextHolder.getContext().setAuthentication(authentication);  
    } catch (JWTVerificationException e) {  
      // 인증이 실패했을 경우 request에 예외 저장  
      request.setAttribute("exception", e);  
    }  
  
    filterChain.doFilter(request, response);  
  }  
  
  
  private String extractAccessTokenWithPrefix(HttpServletRequest request) {  
    return request.getHeader(accessTokenHeaderName);  
  }  
}
```

헤더에 JWT 토큰이 존재하지 않으면 다음 필터로 지나가도록 한다. 토큰이 존재하면 `JwtService`를 통해 검증한다. 검증이 성공하면 `SecurityContext`에 인증된 `Authentication`객체를 저장하고 다음 필터로 진행한다.

검증이 어떤 이유로 **실패**할 수 있다. 이 때 `JWTVerifier`는 원인에 따라 `JWTVerificationException` 밑 하위 예외를 뱉는다. 예를 들어 토큰 기간이 만료되었다면 `JWTVerificationException`의 자식 예외인 `TokenExpiredException`을 뱉는다.

이 **예외를 처리**하는 방법은 여러가지로 고려해 볼 수 있다.

- 검증 예외 발생 시, 바로 해당 필터에서 예외 상황에 대한 `ErrorResponse`를 전송하는 방법
- 검증 예외 발생 시, 해당 필터 앞에 예외에 대한 응답을 하는 `JwtExceptionTranslationFilter`를 추가하고 해당 필터가 예외를 던지면 앞의 필터가 받아서 처리
- 검증 예외 발생 시, 예외를 request의 **attribute**에 저장해 놓고 추후 Spring Security가 인증되지 않는 요청을 처리할 때 사용

첫번째 방법과 두번째 방법은 직관적이지만 **단점**이 존재한다. 만약 해당 요청이 **인증이 필요하지 않은 요청**이었다고 가정하자. 하지만 요청에는 유효시간이 지났거나 검증에 실패하는 토큰이 포함되어 있었다고 하자.

첫번째 방법과 두번째 방법은 해당 요청이 **인증이 필요한 요청인지 아닌지** 판단하지 않고 예외가 발생하면 **바로 응답을 낸다**.

하지만 세번째 방법은 **Spring Security가 해당 요청이 인증이 필요한 요청이라고 판단이 내려졌을 때** JWT 토큰과 관련된 예외를 처리한다. 따라서 **인증이 필요하지 않은 요청**이라면 JWT 토큰에 문제가 있어도 **정상적인 응답**을 낸다.

위와 같은 장점을 가져가기 위해서 request의 attribute를 활용했다.

### JwtAuthententicationEntryPoint

Spring Security는 최종적인 인증/인가 요청을 `AuthorizationFilter`에서 판단한다. 여기서 인증이 필요한 요청인데 인증이 되어 있지 않다거나, 필요한 권한이 존재하지 않는 요청을 `AuthorizationDecision`이라는 클래스로 판단한다. 이때 요건이 충족되지 않았다고 판단이 되면 **예외를 던지고** 이는 `AuthorizationFilter`의 바로 앞 필터인 `ExceptionTranslationFilter`에서 처리하게 된다.

이 때, 인증이 필요한 요청에 인증이 되지 않은 요청이라면 `ExceptionTranslactionFilter`는 `AuthenticationEntryPoint`의 메서드를 호출한다.

`AuthenticationEntryPoint`의 역할은 다음과 같다. 인증이 필요한 자원에 접근할 때 인증이 되지 않으면 자격 증명을 요청하는 HTTP 응답을 보내는 데 사용된다. 즉, `AuthenticationEntryPoint`에 도달했다는 의미는 인증이 필요한 곳에 인증이 되지 않았다는 것을 의미한다.

이제 `AuthenticationEntryPoint`를 구현해보자. `AuthenticationEntryPoint`에 도달 했다는 의미는 인증이 필요한 요청에 인증이 이루어 지지 않았기 때문이다. 인증이 이루어 지지 않은 이유는 크게 두 가지로 생각해 볼 수 있다.

- **토큰이 존재하지 않음** : request의 attribute에 exception 존재 X
- **유효한 토큰이 아님** : request의 attribute에 exception 존재 O
    - 토큰이 만료됨
    - 그 외

유효한 토큰이 아닌 경우에는 request의 attribute에 `JWTVerificationException`이 들어가 있을 것이다. 이 때 실제 예외가 어떤 클래스의 인스턴스인지 파악해 상황별로 다른 응답을 보낼 수 있다.

request에 attribute가 존재하지 않는 경우에는 토큰이 존재하지 않는 경우이므로 Unauthroized 응답을 보낼 수있다. 코드를 살펴보자.

```java
package com.dukcode.securityboilerplate.global.security.jwt.entrypoint;  

@RequiredArgsConstructor  
public class JwtAuthenticationEntryPoint implements AuthenticationEntryPoint {  
  
  private final ObjectMapper objectMapper;  
  
  
  @Override  
  public void commence(HttpServletRequest request, HttpServletResponse response,  
      AuthenticationException authException) throws IOException, ServletException {  
    // attribute에서 exception을 꺼낸다.
    JWTVerificationException jwtVerificationException =  
        (JWTVerificationException) request.getAttribute("exception");  

    // 토큰 만료의 경우 다른 응답
    if (jwtVerificationException instanceof TokenExpiredException) {  
      sendErrorResponse(response, ErrorCode.TOKEN_EXPIRED, 499);  
      return;  
    }  

    // 유효한 토큰이 아닌 경우 다른 응답
    if (jwtVerificationException != null) {  
      sendErrorResponse(response, ErrorCode.INVALID_TOKEN, HttpServletResponse.SC_UNAUTHORIZED);  
      return;  
    }  

    // 토큰이 존재 하지 않는 경우 다른 응답
    sendErrorResponse(response, ErrorCode.UNAUTHORIZED, HttpServletResponse.SC_UNAUTHORIZED);  
  }  
  
  private void sendErrorResponse(HttpServletResponse response, ErrorCode errorCode, int status)  
      throws IOException {  
    ErrorResponse errorResponse = ErrorResponse.of(status, errorCode);  
  
    String body = objectMapper.writeValueAsString(errorResponse);  
    response.setStatus(status);  
    response.setContentType(MediaType.APPLICATION_JSON_VALUE);  
    response.setCharacterEncoding(StandardCharsets.UTF_8.name());  
    response.getWriter().write(body);  
  }  
  
}
```

위와 같이 구성하면 인증 실패 원인을 클라이언트에게 명확하게 전송할 수 있다. 따라서 클라이언트는 상황에 맞게 대처할 수 있게 된다.

### SecurityConfiguration

이제 우리가 구성한 클래스들을 Spring Security 필터 체인에 추가해 보자. `SecurityConfiguration`에 다음과 같이 설정한다.

```java
package com.dukcode.securityboilerplate.global.security.configuration;  
  
@RequiredArgsConstructor  
@EnableWebSecurity(debug = true)  
@Configuration  
public class SecurityConfiguration {  

  // ...
  
  @Bean  
  public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {  
    http.authorizeHttpRequests(  
            request -> request
                .requestMatchers(new AntPathRequestMatcher("/error")).permitAll()
                .requestMatchers(
                    new AntPathRequestMatcher("/member", HttpMethod.POST.name())).permitAll()
                .anyRequest().authenticated()

        )  
        .logout(AbstractHttpConfigurer::disable)  
        .csrf(AbstractHttpConfigurer::disable)  
        .requestCache(RequestCacheConfigurer::disable)  
        .sessionManagement(AbstractHttpConfigurer::disable);  
  
    http.addFilterAt(loginProcessingFilter(authenticationManager()),  
        UsernamePasswordAuthenticationFilter.class);  
    // 로그인 필터 앞에 JWT 인증 필터 추가
    http.addFilterAfter(jwtAuthenticationProcessingFilter(), JsonLoginProcessingFilter.class);  

    // authenticationEntryPoint 등록
    http.exceptionHandling(ex -> ex.authenticationEntryPoint(authenticationEntryPoint()));

  
    return http.build();  
  }  

  // ...

  // JWT 인증 필터 빈 등록
  @Bean  
  public JwtAuthenticationProcessingFilter jwtAuthenticationProcessingFilter() {  
    return new JwtAuthenticationProcessingFilter(jwtService());  
  }  

  // AuthenticationEntryPoint 등록
  @Bean
  public AuthenticationEntryPoint authenticationEntryPoint() {
    return new JwtAuthenticationEntryPoint(objectMapper);
  }

}
```

다음 포스트에서는 Refresh Token을 도입해 보안을 강화해 볼 것이다.