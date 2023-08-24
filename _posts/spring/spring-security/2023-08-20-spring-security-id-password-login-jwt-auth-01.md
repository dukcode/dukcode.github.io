---
layout: single
title: "[Spring Security ID/PW JWT 인증/인가] 01 - 회원 가입 기능 구현"
categories: spring
tag: [spring, spring-security, jwt, login, authentication]
toc: true
toc_sticky: true
author_profile: true
sidebar:
  nav: "docs"
header:
  teaser: "https://i.imgur.com/SxTRdtN.png"
# search: false
---

# 회원 가입 기능 구현

## 들어가기 전

![](https://i.imgur.com/SxTRdtN.png)

프로젝트를 진행하기 전 가장 먼저 구현해야겠다고 생각한 부분은 **인증/인가** 부분이었다. 모든 프로젝트에서 거의 필수적으로 구현하는 부분이기 때문이다. 그 중 **JWT를 활용한 인증/인가 방식**은 서버를 **stateless**하게 유지할 수 있는 장점이 있다. ID/PW 로그인 방식을 사용하면서 JWT 인증/인가를 적용해보자.

Spring Security를 공부하면서 Spring Security의 잘 구현되어 있는 **인증 흐름과 확장포인트를 최대한 활용**하려 JWT를 통한 인증 인가를 구현하고 싶었다. 따라서 JWT의 원리와 같은 부분은 최대한 넘어가고 Spring Security의 인증 흐름과 확장 포인트를 최대한 활용하는 방법에 집중해서 설명해 보겠다.

가장 먼저 프로젝트를 생성하고 회원 가입 기능을 구현해보겠다. 회원 가입 기능은 Spring Security를 이용하지는 않는다.

## 프로젝트 생성 및 설정

[start.spring.io](https://start.spring.io)에서 다음과 같이 설정해 프로젝트를 생성한다.

![](https://i.imgur.com/YwOeKA4.png)

`Spring Web`, `Lombok`, `Spring Data JPA`, `H2 Database`, `Validation`, `Spring Security`를 Dependencies로 추가해준다.

![](https://i.imgur.com/vbWbkSK.png){:.align-center width="500" }

프로젝트를 실행하면 위와 같은 구조의 프로젝트가 생성됨을 볼 수 있다. 이제 프로젝트를 시작하기 위한 설정을 해보자.

### H2 메모리 DB 설정

실제로 운영할 프로젝트가 아니기 때문에 H2 Database를 메모리 모드로 사용해 보겠다. 가장 먼저 프로젝트 루트의 `application.properties`를 `application.yml`로 변경하고 다음과 같이 작성한다.

```yaml
spring:  
  h2:  
    console:  
      enabled: true # /h2-console 설정  
  datasource:  
    url: jdbc:h2:mem:testdb # 메모리 H2 DB 경로 설정
    driver-class-name: org.h2.Driver  
    username: sa  
    password:  
  jpa:  
    properties:  
      hibernate:  
        show_sql: true # JPA가 생성하는 모든 쿼리를 로그에 출력  
        format_sql: true # 출력되는 쿼리를 포매팅해서 출력
```

H2의 디폴트 콘솔 엔드포인트인 `/h2-console`를 활성화 시켜준다. 그리고 메모리 H2 DB의 경로를 설정해준다. 또한 `JPA`가 생성하는 모든 쿼리를 포매팅해서 로그에 출력한다.

> datasource의 url을 굳이 지정해주지 않아도 되지만 지정하지 않으면 **임의의 이름**으로 생성된다. 따라서 h2-console에 접속 용이성을 위해 testdb로 이름을 고정해 두었다.

![](https://i.imgur.com/mDwamxR.png)

애플리케이션을 실행시키면 설정한 DB url로 잘 접속한 것을 확인할 수 있다. `http://localhost:8080/h2-console`에 진입해보면 Form Login 페이지가 뜨고 여기서 ID는 `user` PW는 Spring Security가 임의로 설정해 제공한 값을 입력하면 H2 Console 페이지로 넘어갈 수 있다.

그대로 접속해보면 Spring Security의 설정으로 인해 에러 페이지로 연결된다. Spring Security를 설정해 H2 Console에 연결할 수 있도록 해보자.

### Spring Security 설정

![](https://i.imgur.com/iuKiWPf.png){:.align-center width="500" }

Spring Security와 H2 웹콘솔을 함께 사용하기 위해서는 추가적인 설정이 필요하다. 아래와 같이 설정하면 된다. 설정의 이유는 [Spring Security에서 H2 Consosle 사용하기](https://dukcode.github.io/spring/h2-console-with-spring-security/)에서 확인할 수 있다.

```java
package com.dukcode.securityboilerplate.global.security.configuration;  

@EnableWebSecurity(debug = true)  
@Configuration  
public class SecurityConfiguration {  
  
  @Bean  
  @ConditionalOnProperty(name = "spring.h2.console.enabled", havingValue = "true")  
  public WebSecurityCustomizer configureH2ConsoleEnable() {  
    return web -> web.ignoring()  
        .requestMatchers(PathRequest.toH2Console());  
  }  
  
  @Bean  
  public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {  
    http.authorizeHttpRequests(  
            request -> request
                .requestMatchers(
                    new AntPathRequestMatcher("/member", HttpMethod.POST.name())).permitAll()
            .anyRequest().authenticated())  
        .csrf(AbstractHttpConfigurer::disable)  
        .requestCache(RequestCacheConfigurer::disable)  
        .sessionManagement(AbstractHttpConfigurer::disable);  
  
    return http.build();  
  }  
}
```

H2 Console을 사용하기 위한 설정 말고도 추가적으로 다른 설정을 적용했다.

- **회원가입 요청 인증 불필요** : 회원 가입 요청은 인증이 불필요하게 설정한다. **나머지 요청은 모두 인증이 필수**가 되도록 설정한다.
- **Security debug 활성화** : 디버그 옵션을 활성화하면 요청정보와 Security Filter Chain 목록을 로그로 확인할 수 있다.
- **CSRF 비활성화** : 우리는 JWT를 활용한 stateless REST API를 만들 예정이므로 요청 시 쿠키와 세션을 사용하지 않는다. 따라서 CSRF 보안을 disable해준다. CSRF 필터가 필터 체인 목록에서 사라지게 된다.
- **RequestCache 비활성화**:   disable로 설정하면 `RequestCache`를 `NullRequestCache`로 설정함과 동시에 필터를 제거할 수 있다.
- **세션 비활성화** : 세션 비활성화를 다음과 같이 설정할 수도 있다.

```java
http.sessionManagement(session ->
            session.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
```

> **[CSRF 보안을 사용하지 않는 이유?]**<br>
> CSRF 공격은 클라이언트에 저장되어 있는 **쿠키**를 이용해 사용자가 **의도치 않는 요청**을 서버에 보내는 것을 의미한다. 우리의 서버는 **Token을 통한 인증**을 구현할 예정이다. 따라서 CSRF가 발생할 가능성이 거의 없다고 보는 것이다. 또한 CSRF Filter는 내부적으로 CSRF 토큰을 관리할 때 **세션을 이용**하므로 stateless 서버에서 적용하기 맞지 않다.

> **[RequestCache란?]**<br>
> Spring Security의 기본 설정은 다음과 같다. 인증이 필요한 요청에서 인증에 실패했을 때, 클라이언트에게 로그인 페이지로의 리디렉션 응답을 보냄과 동시에 기존 요청을 `RequestCache`를 통해 **저장**해 놓는다. `RequestCache`의 구현체인 `HttpSessionRequestCache`는 **세션을 통해 기존 요청을 저장**한다. 추후 클라이언트가 인증에 성공하면 Spring Security는 `RequestCache`에서 기존 요청을 꺼내 처리하고 클라이언트에게 응답을 보내게 된다.
>
> 이때 `RequestCache`를 `disable`하면 기존 요청을 꺼내와 현재 요청과 교체해주는 `RequestCacheAwareFilter`를 필터 체인에서 제거하고, `RequestCache`를 `NullRequestCache`로 설정해 **기존 요청이 저장되지 않게** 한다. 즉, **세션**을 통한 기존 요청의 **저장과 불러오기**를 모두 **중단**시키는 것이다.

> **[세션 비활성화 설정이 두가지 인 이유]**<br>
> 필터 체인에서 세션 관련 필터를 지움과 동시에 세션을 아얘 사용하고 싶지 않다면 싶다면 `disable`을 이용하고, 인증 관련해서 세션을 사용하지 않고 다른 부분에 사용할 예정이라면 위의 방법을 사용하면된다.
>
> [Spring Security 공식 문서](https://docs.spring.io/spring-security/reference/servlet/authentication/session-management.html#stateless-authentication)에서는 Statless Authentication 사용하려면 `STATELESS`를 사용하라고 나와있다. `STATELESS` 설정은 **인증에 관련해서 세션 기능을 사용하지 않도록** 하는 설정이다. 하지만 개발자가 **인증 외에 세션을 사용하기를 원할 수 있기 때문에** 인증 외에 세션에 관련된 필터는 계속 **필터 체인에 존재**한다. 예를 들어 `SessionManagementFilter`, `DisableEncodeUrlFilter` 등이 있다.
>
> **`disable`로 설정하면 세션관련 설정을 아얘 하지 않는다.** 따라서 `disable`로 설정하면 세션 관련 필터도 설정되지 않는다. 예를 들면, 응답 URL에 세션ID가 포함되는 것을 방지하는 `DisableEncodeUrlFilter`가 필터 체인에서 없어지는 것을 확인할 수 있다.
>
> `STATELESS`설정은 `RequestCache`를 `NullRequestCache`로 설정한다. 따라서 `RequestCache`를 비활성화 할 필요는 없다. 하지만 `disable`로 설정하는 경우에는 아무 설정도 하지 않기 때문에 세션 관련 기능을 비활성화 하려면 `requestCache`도 `disable`하거나 `NullRequestCache`로 설정해야 **세션 관련 기능을 완벽히 제거**할 수 있다.

## Member 구현

### Member

```java
package com.dukcode.securityboilerplate.domain.member;  
  
@Getter  
@NoArgsConstructor(access = AccessLevel.PROTECTED)  
@Entity  
public class Member {  
  
  @Id  
  @GeneratedValue(strategy = GenerationType.IDENTITY)  
  private Long id;  
  
  @Column(unique = true, nullable = false, updatable = false)  
  private String email;  
  
  @AttributeOverride(name = "rawPassword", column = @Column(name = "password", nullable = false))  
  @Embedded  
  private Password password;  
  
  @Column(nullable = false)  
  private String name;  
  
  @Column(unique = true, nullable = false)  
  private String nickname;  
  
  private String profileImageUrl;  
  
  @Column(nullable = false)  
  @Enumerated(EnumType.STRING)  
  private Role role;  
  
  @Builder  
  public Member(String email, Password password, String name, String nickname,  
      String profileImageUrl,  
      Role role) {  
    this.email = email;  
    this.password = password;  
    this.name = name;  
    this.nickname = nickname;  
    this.profileImageUrl = profileImageUrl;  
    this.role = role;  
  }  
}
```

`Member` 정보에 관련된 필드는 최대한 간단하게 구성했다. 이메일, 패스워드, 이름, 닉네임, 프로필URL을 가진다. 패스워드는 `@Embedded`를 통해 패스워드의 책임을 `Password` 클래스로 분리했다. `Password` 클래스를 분리해 `rawPassword`를 암호화할 책임을 `Password`로 분리해 응집도를 높였다.

롬복은 다음과 같이 이용했다.

- **`@Getter`** : 롬복을 통해 getter들을 추가한다. 좋은 객체지향 프로그래밍은 getter와 setter 모두 지양하지만, DTO로 변환해야 하기 때문에 getter를 추가했다.
- **`@NoArgsContructor`** : JPA 엔티티는 프록시를 이용하므로 **기본 생성자를 필요**로 하기 때문에 롬복을 통해 추가했다. 하지만 어디서나 접근 가능하면 객체 생성의 안전성을 떨어트리므로 **JPA에서 허용하는 최소범위**인 `PROTECTED`로 설정해 외부에서 기본생성을 막았다.
- **`@Builder`** : 애노테이션을 **클래스에 추가하지 않고 원하는 매개변수만 초기화 할 수 있게 하기 위해 생성자에 추가**했다. 클래스에 애노테이션을 추가하면 모든 멤버 변수에 대한 빌더를 생성하기 때문에 객체 생성 시 받아야 하지 않아야 하는 데이터가 객체에 들어올 수 있기 때문이다.

### Password

```java
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Embeddable
public class Password {

  private static final PasswordEncoder passwordEncoder = PasswordEncoderFactories.createDelegatingPasswordEncoder();

  private String encodedPassword;

  public Password(final String rawPassword) {
    this.encodedPassword = encodePassword(rawPassword);
  }

  private String encodePassword(final String rawPassword) {
    return passwordEncoder.encode(rawPassword);
  }

  public void changePassword(final String oldRawPassword, final String newRawPassword) {
    if (isMatches(oldRawPassword)) {
      this.encodedPassword = encodePassword(newRawPassword);
    }
  }

  public boolean isMatches(String rawPassword) {
    return passwordEncoder.matches(rawPassword, encodedPassword);
  }
}

```

패스워드를 암호화할 **책임**을 클래스로 **분리**했다. `PasswordEncoder`를 **캐싱**해 새로운 클래스를 만들어도 같은 `PasswordEncoder`를 새로 생성할 필요가 없게 했다. 

### Role

```java
public enum Role {
  USER, ADMIN;
}
```

`USER`와 `ADMIN`를 구분했다. 일반 회원가입을 통해 가입한 회원은 `USER`로 설정한다.

### DTO

컨트롤러에서 응답과 요청을 처리하고 반환하기위한 DTO를 만들어보자. 응답에서는 회원의 이메일과 닉네임을 반환한다. **Validation을 이용해 허용되지 않은 값이 들어왔을 때 처리**할 수 있도록 만들었다. 적절한 값이 들어오지 않았을 때는 [Spring Custom Exception과 예외 처리 전략에 관한 고민](https://dukcode.github.io/spring/spring-custom-exception-and-exception-strategy/)에서와 같은 방식으로 `@ControllerAdivice`를 이용해 **일관되게 처리**될 수 있도록 했다.

```java
@Getter
@NoArgsConstructor(access = AccessLevel.PRIVATE)
@Builder
@AllArgsConstructor
public class MemberSignupRequest {

  @Email
  private String email;

  @NotBlank
  private String nickname;

  @NotBlank
  private String password;

  @URL
  private String profileImageUrl;

}

@Getter
@Builder
@AllArgsConstructor
public class MemberSignupResponse {

  private String email;
  private String nickname;

  public static MemberSignupResponse from(Member member) {
    return MemberSignupResponse.builder()
        .email(member.getEmail())
        .nickname(member.getNickname())
        .build();
  }
}
```

lombok 관련 애노테이션은 DTO의 **캡슐화를 최대한 보장**하는 방법으로 작성하였다. (창고 : [Spring DTO는 어떻게 작성하고 변환해야 할까?](https://dukcode.github.io/spring/spring-dto/))

## 리포지토리 구현

**Data JPA**를 이용해서 간단하게 구현한다. 이메일과 닉네임은 중복되지 않기 때문에 해당 값이 존재하는지 여부를 체크하기 위해 메서드를 추가했다.

```java
public interface MemberRepository extends JpaRepository<Member, Long> {  
  
  boolean existsByEmail(String email);  
  boolean existsByNickname(String nickname);  
  
}
```

## 서비스 구현

```java
package com.dukcode.securityboilerplate.domain.member.application;

@RequiredArgsConstructor
@Transactional(readOnly = true)
@Service
public class MemberSignupService {

  private final MemberRepository memberRepository;

  @Transactional
  public MemberSignupResponse signup(MemberSignupRequest request) {
    if (memberRepository.existsByEmail(request.getEmail())) {
      throw new EmailDuplicateException(request.getEmail());
    }

    if (memberRepository.existsByNickname(request.getNickname())) {
      throw new NicknameDuplicateException(request.getNickname());
    }

    Member member = Member.builder()
        .email(request.getEmail())
        .nickname(request.getNickname())
        .password(new Password(request.getPassword()))
        .profileImageUrl(request.getProfileImageUrl())
        .role(Role.USER)
        .build();

    memberRepository.save(member);

    return MemberSignupResponse.from(member);
  }
}

```

가입할 회원의 이메일과 닉네임 중복을 검사하고 통과하면 리포지토리에 저장하는 간단한 로직이다. **예외의 구현과 처리**에 관한 부분은 [Spring Custom Exception과 예외 처리 전략에 관한 고민](https://dukcode.github.io/spring/spring-custom-exception-and-exception-strategy/)과 같은 방식으로 처리했다.

## 컨트롤러 구현

```java
package com.dukcode.securityboilerplate.domain.member.controller;

@Slf4j
@RequiredArgsConstructor
@RestController
public class MemberSignupController {

  private final MemberSignupService memberSignupService;

  @PostMapping("/member")
  public ResponseEntity<MemberSignupResponse> signup(
      @RequestBody @Valid MemberSignupRequest request) {
    MemberSignupResponse response = memberSignupService.signup(request);
    return ResponseEntity.status(HttpStatus.CREATED).body(response);
  }
}
```

`@Valid`를 이용해 DTO에 적절한 값이 있는지 검사하도록 했다.


## 실행

이로써 간단하게 회원가입 API를 구현해 보았다. PostMan을 통해 회원가입 API를 간단하게 테스트 해보자.

![](https://i.imgur.com/8hqTKfZ.png)

PostMan으로 위와 같이 회원가입 요청을 보내보면 정상적인 응답이 온다.

![](https://i.imgur.com/d3qyspp.png)

만약 중복된 이메일로 가입하게 되면 위와같이 ErrorResponse가 오는 것을 확인할 수 있다.

![](https://i.imgur.com/7duG9G8.png)

중복된 닉네임으로의 가입도 ErrorResponse가 전달된다.

![](https://i.imgur.com/xiqfeo6.png)

h2-console에서 확인해 보면 Member 테이블에 데이터가 저장된 것을 확인할 수 있다. 이번 포스트에서는 Spring MVC를 이용해 회원 가입 기능을 구현했다. 이제 로그인 기능을 구현해볼 차례이다. 다음 포스트에서 로그인 기능을 구현해보자.
