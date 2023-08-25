---
layout: single
title: "[Spring Security ID/PW JWT 인증/인가] 02 - ID/PW 로그인 구현"
categories: spring
tag: [spring, spring-security, jwt, login, authentication]
toc: true
toc_sticky: true
author_profile: true
sidebar:
  nav: "docs"
header:
  teaser: "https://i.imgur.com/UGOeVxp.png"
# search: false
---

# ID/PW 로그인 구현

![](https://i.imgur.com/UGOeVxp.png)

Spring Security에서 구현해 놓은 **인증 흐름**에 맞게 각 **엔드포인트를 확장**해 JSON을 통한 로그인을 구현해보자. 우리가 구현할 것은 HTTP body에 JSON으로 ID와 PW를 받아서 이를 처리하는 것이다. 가장 먼저 **Spring Security의 인증 흐름**을 알아보자. 

## 인증 흐름


![](https://i.imgur.com/JPgYxun.png)

위의 그림처럼 인증 흐름의 시작은 `AbstractAuthenticationProcessingFilter`에서 시작된다. Spring Security는 form login 인증 방식을 기본적으로 지원한다. **form login**을 enable시키면 Spring Security 필터 체인에 `AbstractAuthenticationProcessingFilter`를 상속한 `UsernamePasswordAuthenticationFilter`를 추가한다.

`AbstractAuthenticationProcessingFilter`는 `HttpServletRequest`로 부터 `Authentication`객체를 생성해 `AuthenticationManager`에 넘긴다. `AuthenticationManager`는 넘겨받은 인증객체로 부터 인증을 시도한다. 인증 성공 시  `SecurityContext`에 인증 객체를 설정하고, 실패 시 `SecurityContext`를 clear한다.

## ID/PW 로그인 구현

![](https://i.imgur.com/eQKv1RG.png)

위에서 말햇듯이 Spring Security의 form login 처리는`AbstractAuthenticationProcessingFilter`를 상속한 `UsernamePasswordAuthenticationFilter`에서 시작된다.

`UsernamePasswordAuthenticationFilter`는 `AbstractAuthenticationProcessingFilter`의 추상메서드인 `attemtpAuthentication()`메서드를 구현하는 **템플릿 메서드 패턴**으로 작성되어 있다. 해당 메서드의 역할은 요청 정보에서 ID와 PW를 **파싱**한 후 `AuthenticationManager`에게 **인증을 위임**한다.

우리는 `AbstractAuthenticationProcessingFilter`를 상속하는 클래스를 작성하고 `AbstractAuthenticationProcessingFilter`가 시작하는 응답 흐름에 올라타면 된다. JSON 형식으로 전달된 ID와 PW를 파싱해 `AuthenticationManager`로 인증을 위임하는 필터를 작성해 보자.

### JsonLoginProcessingFilter

우리가 필수로 구현해야 하는 유일한 메서드는 `attemptAuthentication()` 메서드이다. 이 메서드의 책임은 요청에서 사용자의 **인증 정보를 가져오고** 이를 `AuthenticationManager`에 **인증을 위임**하는 것이다. 해당 메서드만 구현하면 상속하는 클래스인 `AbstractAuthenticationProcessingFilter`가 우리가 구현한 메서드를 호출하고 성공 또는 실패 여부에 따라 인증 흐름을 진행시켜 줄 것이다.

```java
package com.dukcode.securityboilerplate.global.security.login.filter;  
  
public class JsonLoginProcessingFilter extends AbstractAuthenticationProcessingFilter {  
  
  
  public static final String DEFAULT_USERNAME_KEY = "email";  
  public static final String DEFAULT_PASSWORD_KEY = "password";  
  
  private static final AntPathRequestMatcher DEFAULT_ANT_PATH_REQUEST_MATCHER =  
      new AntPathRequestMatcher("/login", "POST");  
  private final ObjectMapper objectMapper;  
  private String usernameParameter = DEFAULT_USERNAME_KEY;  
  private String passwordParameter = DEFAULT_PASSWORD_KEY;  
  private boolean postOnly = true;  
  
  public JsonLoginProcessingFilter(ObjectMapper objectMapper) {  
    super(DEFAULT_ANT_PATH_REQUEST_MATCHER);  
    this.objectMapper = objectMapper;  
  }  
  
  @Override  
  public Authentication attemptAuthentication(HttpServletRequest request,  
      HttpServletResponse response) throws AuthenticationException, IOException {  
    if (this.postOnly && !request.getMethod().equals("POST")) {  
      throw new AuthenticationServiceException(  
          "Authentication method not supported: " + request.getMethod());  
    }  
  
    if (!request.getContentType().equals(MediaType.APPLICATION_JSON_VALUE)) {  
      throw new AuthenticationServiceException(  
          "Authentication content-type not supported: " + request.getContentType());  
    }  
  
    ServletInputStream inputStream = request.getInputStream();  
    Map<String, String> usernamePasswordMap = objectMapper.readValue(inputStream, Map.class);  
  
    String username = obtainParameter(usernameParameter, usernamePasswordMap);  
    String password = obtainParameter(passwordParameter, usernamePasswordMap);  
  
    UsernamePasswordAuthenticationToken authRequest =  
        UsernamePasswordAuthenticationToken.unauthenticated(username, password);  
  
    return getAuthenticationManager().authenticate(authRequest);  
  }  
  
  public void setUsernameParameter(String usernameParameter) {  
    Assert.hasText(usernameParameter, "Username parameter must not be empty or null");  
    this.usernameParameter = usernameParameter;  
  }  
  
  public void setPasswordParameter(String passwordParameter) {  
    Assert.hasText(passwordParameter, "Password parameter must not be empty or null");  
    this.passwordParameter = passwordParameter;  
  }  
  
  public void setPostOnly(boolean postOnly) {  
    this.postOnly = postOnly;  
  }  
  
  private String obtainParameter(String parameter, Map<String, String> usernamePasswordMap) {  
    String value = usernamePasswordMap.get(parameter);  
    if (Objects.isNull(value)) {  
      return "";  
    }  
  
    return value;  
  }  
  
}
```

핵심은 `attemptAuthentication()` 메서드이다. `ObjectMapper`를 생성자를 통해 받아와 JSON 요청의 `username`과 `password`를 `Map`으로 파싱한다. 이를 `UsernamePasswordAuthenticationToken`을 만들어 `AuthenticationManager`에 전달해 인증을 위임한다.

나머지 메서드들은 설정을 바꾸기위한 간단한 메서드들이다.

### UserDetailsServiceImpl

인증 정보가 `AuthenticationManager`로 넘어왔다. `AuthenticationManager`의 실제 구현체는 `ProviderManager`이다. `ProviderManager`는 여러 `AuthenticationProvider`를 가지고 있고 상황에 맞게 `AuthenticationProvider`의 구현체를 골라 인증을 위임한다. `ProviderManager`의 실제 코드를 살펴보자.

![](https://i.imgur.com/fEgKs0G.png)

`ProviderManager`의 `authenticate()`메서드를 살펴보면 `for`문을 돌며 `AuthenticationProvider`가 해당 인증 요청을 `supports`하는지 확인하고 인증을 위임하는 것을 확인할 수 있다.

여기서 우리의 인증정보는 `DaoAuthenticationProvider`에 의해 `supports`된다.

`JsonLoginProcessingFilter`에서 만들어서 넘긴 `Authentication`객체가 `UsernamePasswordAuthenticationToken`이기 때문이다. `DaoAuthenticationProvider`의 부모 클래스인 `AbstrctUserDetailsAuthenticationProvider`의 `supports()`메서드를 살펴보면 다음과 같다.

![](https://i.imgur.com/6tEKXRv.png)

`Authentication`객체가 `UsernamePasswordAuthenticationToken`이면 `true`를 반환한다. 우리는 `JsonLoginProcessingFilter`에서 `UsernamePasswordAuthenticationToken`을 생성해서 전달했기 때문에 해당 `DaoAuthenticationProvider`가 선택된다.

`DaoAuthenticationProvider`은 DAO에서 계정 정보를 꺼내오고 패스워드 일치 여부를 검사하고 인증 여부를 넘겨준다. `DaoAuthenticationProvider`가 유저 정보를 가져오는 메서드인 `retrieveUser()`메서드를 확인해 보자.

![](https://i.imgur.com/Fva4Kxo.png)

`UserDetailsService`에 `loadUserByUsername()` 메서드에 `username`을 전달해 유저 정보를 가져오는 것을 확인할 수 있다. 따라서 우리는 우리의 DB에서 유저의 정보를 가져오도록 클래스를 구현해야 할 것이다. `UserDetailsService`를 구현해 DB에서 `username`을 가진 유저의 정보를 가져와 보자.

```java
package com.dukcode.securityboilerplate.global.security.login.service;

@RequiredArgsConstructor
@Service
public class UserDetailsServiceImpl implements UserDetailsService {

  private final MemberRepository memberRepository;

  @Override
  public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
    Member member = memberRepository.findByEmail(username)
        .orElseThrow(() -> new UsernameNotFoundException("존재하지 않는 사용자 입니다."));
    return User.builder()
        .username(member.getEmail())
        .password(member.getPassword().getEncodedPassword())
        .roles(member.getRole().name())
        .build();
  }
}

```

위와 같이 `loadUserByUsername()`메서드를 구현해 `MemberRepository`를 통해 멤버를 가져와 `User` 객체로 리턴하도록 구현했다.

### LoginSucessHandler & LoginFailureHandler

`AbstractAuthenticationProcessingFilter`는 `AuthenticationSuccessHandler`와 `AuthenticationFailureHandler`를 멤버 변수로 가지고 있다. `AbstractAuthenticationProcessingFilter`의 코드를 살펴보자.

![](https://i.imgur.com/BqDI94x.png)

`AbstractAuthenticatiionProcessingFilter`는 우리가 구현한 템플릿 메서드인 `attemptAuthentication()`메서드를 호출 하고 그 결과에 따라 `AuthenticationSuccessHandler`의 `onAuthenticationSuccess()` 메서드를 호출하거나 `AuthenticationFailureHandler`의 `onAuthenticationFailure()`메서드를 호출한다.

![](https://i.imgur.com/VhJ9E8i.png)

실제로 메서드 내부를 확인해보면 인증 실패 성공 여부에 따라 핸들러의 메서드를 호출하는 것을 확인할 수 있다.

따라서 우리는 `AuthenticationFailureHandler`를 구현해 인증 실패 시 응답을 구현할 수 있다. 또한 `AuthenticationSuccessHandler`를 구현해 인증 성공시 JWT 토큰을 발급해 응답에 포함시킬 수 있다.

아직은 JWT에 대해 소개하기 전이므로 `AuthenticationSuccessHandler`에서는 간단하게 로그를 찍고 200(OK)응답만 주는 것으로 구현해 보겠다.

```java
package com.dukcode.securityboilerplate.global.security.login.handler;

@Slf4j
public class LoginSuccessHandler implements AuthenticationSuccessHandler {

  @Override
  public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response,
      Authentication authentication) throws IOException, ServletException {

    log.info("login success");
    response.setStatus(HttpStatus.OK.value());
  }
}

```

간단하게 로그를 찍고 200 응답을 보낸다. 추후 JWT 관련 클래스들의 구현이 마무리 되면 `LoginSuccessHandler`에서 JWT 토큰을 발급해 응답 포함시킬 예정이다.

실패 응답은 `ErrorResponse`를 이용해 보낸다. ([Spring Custom Exception과 예외 처리 전략에 관한 고민](https://dukcode.github.io/spring/spring-custom-exception-and-exception-strategy/)의 `ErrorResponse` 참조)

```java
package com.dukcode.securityboilerplate.global.security.login.handler;

@RequiredArgsConstructor
public class LoginFailureHandler implements AuthenticationFailureHandler {

  private final ObjectMapper objectMapper;

  @Override
  public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response,
      AuthenticationException exception) throws IOException {

    int status = HttpStatus.UNAUTHORIZED.value();
    ErrorResponse errorResponse = ErrorResponse.of(status, ErrorCode.INVALID_USERNAME_OR_PASSWORD);

    String body = objectMapper.writeValueAsString(errorResponse);
    response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
    response.setContentType(MediaType.APPLICATION_JSON_VALUE);
    response.setCharacterEncoding(StandardCharsets.UTF_8.name());
    response.getWriter().write(body);
  }

}

```

로그인 요청이 실패 했으므로 UNAUTHORIZED(401)응답을 `ErrorResponse`에 담아서 응답한다.

### `SecurityConfiguration`

이제 이 클래스들을 **조합**하게 되면 인증 흐름에 맞게 로그인 과정이 진행된다. 시큐리티 설정을 통해 클래스들을 조립해보자.

```java
package com.dukcode.securityboilerplate.global.security.configuration;  
  
@RequiredArgsConstructor  
@EnableWebSecurity(debug = true)  
@Configuration  
public class SecurityConfiguration {  
  
  private final ObjectMapper objectMapper;  
  private final ObjectPostProcessor<Object> objectPostProcessor;  
  private final MemberRepository memberRepository;  
  
  @Bean  
  @ConditionalOnProperty(name = "spring.h2.console.enabled", havingValue = "true")  
  public WebSecurityCustomizer configureH2ConsoleEnable() {  
    return web -> web.ignoring()  
        .requestMatchers(PathRequest.toH2Console());  
  }  
  
  @Bean  
  public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {  
    // ...

    // Filter 추가
    http.addFilterAt(loginProcessingFilter(authenticationManager()),  
        UsernamePasswordAuthenticationFilter.class);  
  
    return http.build();  
  }  
  
  
  @Bean  
  public AbstractAuthenticationProcessingFilter loginProcessingFilter(  
      AuthenticationManager authenticationManager) {  
    JsonLoginProcessingFilter jsonLoginProcessingFilter = new JsonLoginProcessingFilter(  
        objectMapper);  

    // AuthenticationManager 설정
    jsonLoginProcessingFilter.setAuthenticationManager(authenticationManager);  

    // Handler 설정
    jsonLoginProcessingFilter.setAuthenticationSuccessHandler(authenticationSuccessHandler());
    jsonLoginProcessingFilter.setAuthenticationFailureHandler(authenticationFailureHandler());

    return jsonLoginProcessingFilter;  
  }  
  
  @Bean  
  public AuthenticationManager authenticationManager() throws Exception {  
    AuthenticationManagerBuilder builder = new AuthenticationManagerBuilder(objectPostProcessor);  

    // UserDetailsService, PasswordEncoder 설정
    builder.userDetailsService(userDetailsService()).passwordEncoder(passwordEncoder());  
    return builder.build();  
  }  
  
  @Bean  
  public PasswordEncoder passwordEncoder() {  
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();  
  }  
  
  @Bean  
  public UserDetailsService userDetailsService() {  
    return new UserDetailsServiceImpl(memberRepository);  
  }  
  
}
```

`UserDetailsServiceImpl`과 `DelegatingPasswordEncoder`를 `AuthenticationManager`에 설정하고 이를 `JsonLoginProcessingFilter`를 생성하면서 설정해준다. 또한 Handler 빈도 설정해준다.

마지막으로 빈으로 설정된 `LoginProcessingFilter`를 form로그인을 담당했던 필터인 `UsernamePasswordAuthenticationFilter`의 자리에 추가해준다. 그러면 시큐리티 필터 체인에 우리가 구현한 로그인 필터가 끼워지게 된다.

### 실행

![](https://i.imgur.com/6Y8I5GB.png)

로그인 URL로 요청을 전송하면 200 OK 응답이 오는 것을 확인할 수 있다. 물론 미리 가입요청을 보내야 한다. 로그도 찍힌다.

![](https://i.imgur.com/A4hgZAt.png)

다른 비밀번호나 없는 계정으로 요청하면 401 Unauthorized 응답과 함께 ErrorResponse를 확인할 수 있다.

![](https://i.imgur.com/wewn8Sp.png)

Spring Security의 기본적인 인증 흐름에서의 확장포인트들을 구현해 우리가 원하는 방식으로 확장할 수 있었다. 다음 포스트에서는 로그인 시 JWT 토큰을 발급하고 이를 다른 요청에서 인증할 수 있도록 구현해 보자.