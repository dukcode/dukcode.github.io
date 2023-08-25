---
layout: single
title: "[Spring Security ID/PW JWT 인증/인가] 04 - Refresh Token 도입"
categories: spring
tag: [spring, spring-security, jwt, login, authentication]
toc: true
toc_sticky: true
author_profile: true
sidebar:
  nav: "docs"
header:
  teaser: "https://i.imgur.com/5QlGQnW.png"
# search: false
---

# Refresh Token 도입

![](https://i.imgur.com/5QlGQnW.png)

우리가 이전 포스트에서 구현한 토큰을 가지고 충분히 인증 프로세스는 잘 돌아간다. 보안 위협이 없다면 말이다.

JWT 토큰의 탈취가 일어난다면 탈취된 토큰은 유효시간 만큼 정상적으로 작동하고 서버에서 이를 막을 수 있는 방법이 존재하지 않는다. 우리의 서버는 무상태(stateless)이기 때문이다.

위의 이유로 우리는 기존 토큰의 유효기간을 짧게 설정해 토큰이 탈취되어도 공격자의 유의미한 행동을 최소화할 수 있는 방법을 생각할 수 있다. 유효기간이 짧은 토큰은 유효기간이 끝날 때 마다 다시 인증을 받아야 한다. 이는 사용자의 입장에서 매우 귀찮은 일일 것이다. 따라서 토큰 재발급을 위한 토큰인 Refresh Token을 발급해 단점을 상쇄할 수 있다.

- 토큰(Access Token)의 유효기간을 짧게 설정한다.
- 유효기간이 긴, 토큰 재발급을 위한 Refresh Token을 도입한다.

우리는 토큰 탈취의 위협 때문에 유효 기간을 짧게 설정하고 Refresh Token을 도입했다. 하지만 Refresh Token이 탈취된다면?

Refresh Token이 탈취된다면 Refresh Token의 도입 이유가 없어진다. 이를 극복하기 위해서 Refresh Token Refresh 방법을 사용할 것이다. 즉, 한번 사용한 Refresh Token은 다시 사용할 수 없다는 원칙이다. 즉, 재발급 요청 시마다 새로운 Refresh Token을 발급한다. 시나리오는 다음과 같다.

- 공격자는 사용자가 토큰 재발급을 받을 때 Refresh Token을 탈취한다.
- 공격자의 Access Token 만료로 다시 토큰을 재발급 받는다. Refresh Token은 새로 발급되고 서버에도 함께 저장된다.
- 사용자가 토큰을 재발급 받는다. 서버에 저장되어 있던 Refresh Token과 사용자가 가져온 Refresh Token을 비교한다.
- 두 토큰이 다르다면 해당 토큰은 다른 곳에서 사용되었다는 의미이므로 서버에 저장된 Refresh Token을 삭제한다.
- 공격자는 다시 재발급 할 수 없고 이용자는 다시 인증해서 서비스를 이용한다.

이제 위의 시나리오대로 구현해보자.

## RefreshTokenManager 구현

가장 먼저 refresh token을 관리하는 `RefreshTokenManager` 클래스를 만들자. 인터페이스로 선언해서 여러 방법을 구현을 할 수 있게 할 예정이다.  가장 먼저 JPA를 이용해 refresh token을 관리하는`JpaRefreshTokenManager`을 구현할 생각이다. 만약 redis를 사용해 refresh token을 관리할 것이라면 `RedisRefreshTokenManager`를 구현해서 DI하면 될 것이다. 추후에 redis를 이용해보고 어느정도 성능 개선이 이루어 지는지 측정해보자.

```java
public interface RefreshTokenManager {

  void update(String username, String token);

  Optional<String> find(String username);

  void delete(String username);

}

```

- **update** 메서드: username을 키로 가진 refresh token을 새로운 토큰으로 업데이트하거나 생성한다.
- **find** 메서드: username을 키로 가진 refresh token을 반환한다.
- **delete** 메서드: username을 키로 가진 refresh token을 삭제한다.

이제 실제로 `JpaRefreshTokenManager`를 구현하기 전에 `RefreshToken` 엔티티부터 구현해보자.

```java
package com.dukcode.securityboilerplate.global.security.jwt;  
  
@Getter  
@RequiredArgsConstructor(access = AccessLevel.PROTECTED)  
@Entity  
public class RefreshToken {  
  
  @Id  
  @GeneratedValue  private Long id;  
  
  private String token;  
  
  @Column(nullable = false, unique = true)  
  private String email;  
  
  public RefreshToken(String token, String email) {  
    this.token = token;  
    this.email = email;  
  }  
  
  public void updateToken(String token) {  
    this.token = token;  
  }  

  public void deleteToken() {
    this.token = null;
  }
}
```

토큰 값과 이메일 값을 가지고 있는 간단한 엔티티이다.

```java
package com.dukcode.securityboilerplate.global.security.jwt.manager;  
  
@RequiredArgsConstructor  
public class JpaRefreshTokenManager implements RefreshTokenManager {  
  
  private final EntityManager em;  
  
  @Override  
  public void update(String username, String token) {  
    List<RefreshToken> refreshTokens = em.createQuery(  
            "SELECT rt FROM RefreshToken rt WHERE rt.email = :username",  
            RefreshToken.class)  
        .setParameter("username", username)  
        .getResultList();  
  
    if (refreshTokens.isEmpty()) {  
      em.persist(new RefreshToken(token, username));  
      return;  
    }  
  
    RefreshToken refreshToken = refreshTokens.get(0);  
    refreshToken.updateToken(token);  
  }  
  
  @Override  
  public Optional<String> find(String username) {  
    List<RefreshToken> refreshTokens = em.createQuery(  
            "SELECT rt FROM RefreshToken rt WHERE rt.email = :username",  
            RefreshToken.class)  
        .setParameter("username", username)  
        .getResultList();  
  
    if (refreshTokens.isEmpty()) {  
      return Optional.empty();  
    }  
  
    RefreshToken refreshToken = refreshTokens.get(0);  
  
    return Optional.of(refreshToken.getToken());  
  }  

  @Override
  public void delete(String username) {
    List<RefreshToken> refreshTokens = em.createQuery(
            "SELECT rt FROM RefreshToken rt WHERE rt.email = :username",
            RefreshToken.class)
        .setParameter("username", username)
        .getResultList();

    if (refreshTokens.isEmpty()) {
      return;
    }

    RefreshToken refreshToken = refreshTokens.get(0);

    refreshToken.deleteToken();
  }

}
```

- `update()`메서드는 가장 먼저 `username`을 가지는 refresh token이 있는지 찾고 없다면 새로운 토큰을 추가하고, 존재한다면 기존의 토큰 값을 업데이트 시킨다.
- `find()`메서드는 `username`을 가지는 refresh token이 있는지 조회하고 토큰 값을 반환한다.
- `delete()`메서드는 `username`을 가지는 refresh token을 삭제한다.

## JwtService에 적용

이제 `JpaRefreshTokenManager`를 `JwtService`에 적용하고 refresh token 생성 메서드를 구현해보자.

```java
package com.dukcode.securityboilerplate.global.security.jwt.service;  
  
@RequiredArgsConstructor  
public class JwtService {  

  private final RefreshTokenManager refreshTokenManager;

  // ...

  @Transactional
  public String createRefreshTokenWithPrefix(Authentication authentication) {  
    UserDetails userDetails = (UserDetails) authentication.getPrincipal();  
    String username = userDetails.getUsername();  
  
    String refreshToken = createRefreshTokenWithPrefix(username);  
  
    refreshTokenManager.update(username, refreshToken);  
  
    return withTokenPrefix(refreshToken);  
  }  
  
  private String createRefreshTokenWithPrefix(String username) {  
    return JWT.create().withSubject(REFRESH_TOKEN_SUBJECT)  
        .withIssuedAt(new Date())  
        .withExpiresAt(new Date(System.currentTimeMillis() + refreshTokenExpirationSeconds * 1000L))  
        .withClaim(USERNAME_CLAIM, username)  
        .sign(algorithm);  

  }  

  // ...
  
}
```

refresh token을 생성하고 이 값을 `RefrefhTokenManager`를 통해 저장한다. refresh token은 만료 시점을 정보에 포함하고 있고 `Role`은 포함하지 않도록 설계했다.

## 로그인 시 Refresh Token 발급

이제 로그인 시 refresh token을 함께 발급할 수 있도록 `JwtLoginSuccessHandler`에 추가구현해보자.

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

    // access token 생성
    String accessTokenWithPrefix = jwtService.createAccessTokenWithPrefix(authentication);  
    // refresh token 생성 및 저장
    String refreshTokenWithPrefix = jwtService.createRefreshTokenWithPrefix(authentication);  

    // 응답에 토큰 포함
    response.setHeader(accessTokenHeaderName, accessTokenWithPrefix);  
    response.setHeader(refreshTokenHeaderName, refreshTokenWithPrefix);  
    response.setStatus(HttpStatus.OK.value());  
  }  
  
}
```

`JwtService`를 통해서 간단하게 access token과 refresh token을 발급하고 저장할 수 있다.

`RefreshTokenManager`를 빈으로 등록하고 `JwtService`에 DI하도록 설정한다.

```java
package com.dukcode.securityboilerplate.global.security.configuration;  
  
@RequiredArgsConstructor  
@EnableWebSecurity(debug = true)  
@Configuration  
public class SecurityConfiguration {  

  // ...
  
  private final EntityManager entityManager;  
  
  // ...

  @Bean  
  public RefreshTokenManager refreshTokenManager() {  
    return new JpaRefreshTokenManager(entityManager);  
  }  
  
  @Bean  
  public JwtService jwtService() {  
    return new JwtService(refreshTokenManager());  
  }  

  // ...
  
}
```

위와 같이 추가 설정하게 되면 로그인 시에 access token과 refresh token이 함께 발급된다.

![](https://i.imgur.com/fI8daUJ.png)

위와 같이 access token과 refresh token이 로그인 응답에 실려 오는 것을 확인할 수 있다.

## 재발급 로직

다시 reissue 과정을 살펴보자.

1. 클라이언트가 만료된 access token으로 요청을 보낸다.
2. 서버는 access token이 만료되었다고 응답을 보낸다.
3. 클라이언트는 access token이 만료되었다는 응답을 받으면 로그인 시에 받아놓은 refresh token을 요청에 담아 전송한다.
4. refresh token이 유효한 토큰인지 검증한다. 유효기간이 만료된 토큰이라면
5. 서버는 클라이언트가 전송한 refresh token과 서버에 저장되어 있던 refresh token을 비교한다.
6. 요청의 refresh token과 서버의 refresh token가 같다면 새로운 access token과 refresh token을 발급한다. 이때 새로 발급한 refresh token은 서버에 업데이트 시킨다.
7. 요청의 refresh token과 서버의 refresh token가 다르다면 두가지 가능성이 있다. 클라이언트가 reissue를 받기 전 refresh token이 이미 탈취되었고 악의적인 사용자가 먼저 reissue를 요청한 것이다. 다른 가능성은 사용자가 기존 기기에서 로그인을 진행하고 다른 기기에서 로그인한 후에 다시 원래 기기로 돌아와서 reissue를 요청하는 경우이다. 첫번째 경우엔 서버에 저장된 refresh token을 삭제해 악의적인 사용자가 더 이상 접속하지 못하도록 하는 것이 맞을 것이다. 두 번째 경우에는 중복 로그인 허용할 수 있도록 refresh token을 여러개 저장하는 방법이 있지만 현재 프로젝트에서는 중복 로그인을 방지하는 차원에서 하나의 refresh token을 관리하는 방법을 사용하겠다. 이 경우에도 서버의 refresh token을 삭제해 다른 기기에서의 로그인을 종료시킨다.

### JwtService에 적용

이제 `JwtService`의 `reissue()`메서드를 위의 로직을 따라 구현해보자.

```java
package com.dukcode.securityboilerplate.global.security.jwt.service;

@RequiredArgsConstructor
public class JwtService {

  @Transactional
  public TokenDto reissue(String refreshTokenWithPrefix) {
    String refreshToken = withTokenPrefix(refreshTokenWithPrefix);
    // refresh token 검증
    // 토큰 검증 오류 발생 시 JWTVerificationException 발생
    DecodedJWT decodedToken = verifier.verify(refreshToken);

    String username = decodedToken.getClaim(USERNAME_CLAIM).asString();
    List<String> roles = decodedToken.getClaim(ROLES_CLAIM).asList(String.class);

    Optional<String> savedRefreshToken = refreshTokenManager.find(username);

    // 저장되어 있던 username의 refresh token이 존재하지 않거나
    // 요청 시의 토큰과 일치 하지 않으면 
    if (savedRefreshToken.isEmpty() || !refreshToken.equals(savedRefreshToken.get())) {
      refreshTokenManager.delete(username);
      throw new UnexpectedRefreshTokenException();
    }

    String newRefreshToken = createRefreshToken(username, roles);
    String newAccessToken = createAccessToken(username, roles);

    return new TokenDto(withTokenPrefix(newAccessToken), withTokenPrefix(newRefreshToken));
  }

}
```

클라이언트에게 refresh token을 받아서 검증한다. 검증 시, 토큰에 문제(기간 만료, 유효하지 않은 토큰 등)가 있다면 `JWTVerificationException`이 발생된다.

토큰으로부터 `username`을 얻고 이를 통해 서버에 저장되어 있는 refresh token을 가져온다. 만약 저장된 refresh token이 null이거나 클라이언트로부터 온 refresh token과 다르면 `JWTVerificationException`을 상속한 `UnexpectedRefreshTokenException`을 발생시킨다.

서버에 저장된 refresh token과 클라이언트에서 요청한 refresh token이 같다면 새로운 access token과 refresh token을 발급해 리턴한다.

### JwtAuthenticationProcessingFilter에 적용

이제 `JwtAuthenticationProcessingFilter`에서 reissue 로직을 추가해 보자.

```java
package com.dukcode.securityboilerplate.global.security.authentication.filter;  
  
@Slf4j  
@RequiredArgsConstructor  
public class JwtAuthenticationProcessingFilter extends OncePerRequestFilter {  
  
  private final JwtService jwtService;  
  
  @Value("${jwt.response.access-token-header-name}")  
  private String accessTokenHeaderName;  
  @Value("${jwt.response.refresh-token-header-name}")  
  private String refreshTokenHeaderName;  
  
  @Override  
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,  
      FilterChain filterChain) throws ServletException, IOException {  
    String refreshTokenWithPrefix = extractRefreshTokenWithPrefix(request);  
  
    String accessTokenWithPrefix = extractAccessTokenWithPrefix(request);  
  
    // refresh token 포함 == 재발급의 경우  
    if (StringUtils.hasText(refreshTokenWithPrefix)) {  
      try {  
        TokenDto tokenDto = jwtService.reissue(refreshTokenWithPrefix);  
        sendReissueSuccessResponse(response, tokenDto);  
        return;  
      } catch (JWTVerificationException e) {  
        request.setAttribute("exception", e);  
        filterChain.doFilter(request, response);  
        return;  
      }  
    }  
  
    // access-token 없으면 pass  
    if (!StringUtils.hasText(accessTokenWithPrefix)) {  
      filterChain.doFilter(request, response);  
      return;  
    }  
  
    try {  
      Authentication authentication  
          = jwtService.resolveAccessTokenWithPrefix(accessTokenWithPrefix);  
      SecurityContextHolder.getContext().setAuthentication(authentication);  
    } catch (JWTVerificationException e) {  
      // 인증이 실패했을 경우 request에 예외 저장  
      request.setAttribute("exception", e);  
    }  
  
    filterChain.doFilter(request, response);  
  }  
  
  private void sendReissueSuccessResponse(HttpServletResponse response, TokenDto tokenDto) {  
    response.setHeader(accessTokenHeaderName, tokenDto.getAccessTokenWithPrefix());  
    response.setHeader(refreshTokenHeaderName, tokenDto.getRefreshTokenWithPrefix());  
    response.setStatus(HttpServletResponse.SC_RESET_CONTENT);  
  }  

  // ...
}
```

요청에 refresh token이 포함되어 있다면 reissue 요청이므로 재발급을 진행한다. `JwtService`의 `reissue()`메서드를 호출해 재발급을 진행한다. 재발급 도중 오류가 있으면 오류를 `request`의 `attribute`에 저장한다. 이는 추후 `AuthenticationEntryPoint`에서 예외 별로 다른 응답을 구성하기 위해 저장하고 다음 필터로 넘어간다.

재발급 과정이 정상적으로 이루어 졌다면 `sendReissueSuccessResponse()`메서드를 호출해 RESET_CONTENT(205)응답을 구성하고 `return`한다.